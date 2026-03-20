# ADR-005: Frame Storage Strategy for StoriesBuffer

**Status:** Accepted (Phase 1) / Planned (Phase 2)
**Date:** February 2026

## Context

When a VLM batch is processed, the `Aggregator` creates a `Story` object that bundles the text description with the source frames:

```python
Story(
    camera_id=...,
    description="Person walking near entrance",
    frames=[frame1, frame2, ..., frame10]   # full batch stored here
)
```

These `Story` objects live in the `StoriesBuffer`, a rolling window that the `LLMDetector` reads to detect events. The frames inside each story serve as **visual evidence** — when an event is detected, the `DetectedEvent` includes the source stories with their frames so operators can see what triggered the alert.

The problem is that by the time the LLM detects an event, the `FrameBuffer` has already been flushed multiple times. The frames inside `Story` objects are the **only copy** of that visual evidence. Removing them would mean losing the ability to show what was happening when the event was triggered.

### Why this matters at scale

The RAM consumed by `StoriesBuffer` grows with the number of cameras and buffer configuration:

```
RAM_per_camera = stories_buffer_size × frame_buffer_size × avg_frame_bytes
RAM_total      = N_cameras × RAM_per_camera
```

With default configuration (`stories_buffer_size=20`, `frame_buffer_size=10`) and typical JPEG frames at 1280×720 quality 85 (~300 KB each):

| Cameras | RAM_per_camera | RAM_total |
|---------|---------------|-----------|
| 3       | 20 × 10 × 300 KB = **60 MB**  | ~180 MB |
| 10      | 60 MB                         | ~600 MB |
| 20      | 60 MB                         | **~1.2 GB** |

This is RAM consumed by the Python process itself, subject to Python's memory fragmentation — freed objects may not return to the OS promptly, so peak usage is higher in practice.

### Minimum TTL required for a frame cache

For any solution that stores frames outside the `Story` and fetches them at event detection time, the storage must guarantee frames are available for the entire pipeline latency window. The worst-case time from frame capture to event detection is:

```
TTL_min = (frame_interval × frame_buffer_size + avg_vlm_latency) × stories_buffer_size + avg_llm_latency
```

With default values and conservative latency estimates (VLM ~5s, LLM ~10s):

```
TTL_min = (1s × 10 + 5s) × 20 + 10s = 15s × 20 + 10s = 310 seconds ≈ 6 minutes
```

A safe TTL is **15 minutes** to account for provider slowdowns or queue buildup.

---

## Options

### Option A: Keep all frames in every Story (current)

Each `Story` stores all `frame_buffer_size` frames from the VLM batch.

**Pros:**
- Simple — no additional infrastructure
- Evidence always available at event detection time

**Cons:**
- RAM grows linearly with cameras × buffer sizes (formula above)
- Python memory fragmentation amplifies peak usage
- At 20 cameras with default config: ~1.2 GB just for StoriesBuffers

---

### Option B: Store one representative frame per Story (Phase 1 — chosen)

Each `Story` stores only the **last frame** of the VLM batch instead of all frames.

```
RAM_per_camera = stories_buffer_size × 1 × avg_frame_bytes
               = 20 × 300 KB = 6 MB

RAM_total (20 cameras) = 20 × 6 MB = 120 MB   (10× reduction)
```

The `Aggregator` selects the last frame of the batch as the representative frame, since it is the most temporally recent and therefore most relevant to the description the VLM generated.

**Pros:**
- 10× RAM reduction with no infrastructure changes
- Visual evidence still present at event detection time
- Single frame is sufficient to identify what was happening at the moment

**Cons:**
- Loss of the full frame sequence — operators see one frame per story instead of ten
- Does not scale indefinitely; at very high camera counts RAM still grows

---

### Option C: Frames stored in Redis with TTL (Phase 2 — planned)

`Story` objects store only frame IDs (UUIDs). Frame bytes live in Redis with a TTL of at least `TTL_min` (see formula above, recommended 15 minutes). When an event is detected, the engine fetches the relevant frame bytes by ID.

```
Python process RAM ≈ stories_buffer_size × avg_story_text_size
                   ≈ 20 × 1 KB = 20 KB per camera   (negligible)

Redis RAM = bounded by TTL, not by camera count
```

**Pros:**
- Python process RAM for stories becomes negligible regardless of camera count
- Engine becomes stateless with respect to frame history — can restart without losing evidence window
- Enables multi-instance deployments (multiple engine processes share the same Redis)

**Cons:**
- Requires Redis as an additional infrastructure dependency
- Frame fetch latency added at event detection time (acceptable — event detection is not on the hot path)
- If Redis is unavailable, events are detected but lack visual evidence
- TTL must be configured correctly; if set too short, frames may expire before the event is detected

---

## Decision

**Phase 1 (now): Option B** — store one representative frame per Story.

Reduces RAM by 10× with no infrastructure changes and no loss of evidence. Justified while the system operates with fewer than ~10 cameras on a single instance.

**Phase 2 (when scaling beyond ~10 cameras or moving to multi-instance): Option C** — Redis frame store with TTL.

The trigger for Phase 2 is any of:
- Camera count exceeds 10
- Engine needs to run as multiple instances
- Observed RAM pressure on the host exceeds 50% of available memory

---

## Consequences

- `Aggregator.aggregate()` changes: instead of passing all `frames` to `Story`, it passes only `frames[-1:]` (last frame of the batch).
- `Story.frames` remains `list[Frame]` — the schema does not change, only the number of elements stored.
- The `DetectedEvent` will contain one frame of evidence per story instead of the full batch sequence. This is sufficient for operator review.
- Tests for `Aggregator` must verify that exactly one frame is stored per story.
- Phase 2 introduces a `FrameStore` abstraction (in-memory dict → Redis adapter) so the swap is transparent to `Aggregator` and `LLMDetector`.
