# ADR-004: Concurrent Per-Camera Processing with Independent Asyncio Tasks

**Status:** Accepted
**Date:** February 2026

## Context

The original `ProcessingManager` ran a single `run()` loop that iterated all cameras **sequentially**:

```python
while True:
    for camera_id, frame_buffer in registry.get_all_frame_buffers().items():
        await _process_camera(camera_id, frame_buffer)  # blocks everything else
```

`_process_camera()` ran the full pipeline in a single call: flush frames → VLM inference → aggregate → LLM inference → normalize. Because every stage used `await`, the loop could not advance to the next camera until the current one finished both AI calls.

This caused two concrete problems:

1. **Cross-camera blocking:** With 3 cameras, if cam-1's VLM inference takes 3 seconds, cam-2 and cam-3 cannot send their requests to the provider during that time — even if their frame buffers are already full and the provider is idle.

2. **Intra-camera blocking:** Within a single camera, the VLM and LLM stages were coupled in the same call. While the LLM was doing inference, the VLM could not process a new frame batch, even though those are completely independent operations.

The system also did not contemplate that multiple cameras can share the same provider. Each camera has its own `VLMProcessor` / `LLMDetector` Python instance (with its own prompt and config), but they may all point to the same API endpoint. With the sequential loop, concurrent requests to the same provider were never possible.

## Options

### Option A: `asyncio.gather()` in the main loop (Rejected)

Replace the `for` loop with `asyncio.gather()` to run all cameras concurrently per cycle, but keep VLM and LLM coupled inside `_process_camera()`.

```python
await asyncio.gather(*[_process_camera(id, buf) for id, buf in buffers.items()])
```

**Pros:** Simple change, cross-camera blocking is eliminated.

**Cons:** VLM and LLM remain coupled — LLM inference still blocks the next VLM batch for the same camera. Also, the gather waits for the slowest camera before re-polling, adding unnecessary latency.

### Option B: Per-camera OS threads for processing (Rejected)

Run a full processing thread per camera (mirroring the capture thread model).

**Pros:** True OS-level parallelism, familiar model.

**Cons:** Python's GIL limits CPU parallelism. Processing is I/O-bound (HTTP calls), where asyncio already provides equivalent concurrency with lower overhead and simpler synchronization. Adding more threads increases complexity without a clear benefit.

### Option C: Per-camera asyncio tasks with decoupled VLM and LLM loops (Chosen)

Each camera gets **two independent `asyncio.Task`** objects launched when the camera is added:

- **`_vlm_loop`**: polls `FrameBuffer`, flushes when ready, calls VLMProcessor, aggregates stories into `StoriesBuffer`, signals the LLM task via `asyncio.Event`.
- **`_llm_loop`**: sleeps until signaled by the VLM task, reads `StoriesBuffer`, calls LLMDetector, runs EventNormalizer.

```
cam-1: [vlm_loop task]──event──►[llm_loop task]
cam-2: [vlm_loop task]──event──►[llm_loop task]   all tasks run concurrently
cam-3: [vlm_loop task]──event──►[llm_loop task]
```

## Decision

**Option C.** Per-camera asyncio tasks with an `asyncio.Event` as the VLM → LLM signal.

### Why asyncio.Event instead of polling

The `_llm_loop` could busy-poll `stories_buffer.is_ready()` every second. Instead, it calls `await event.wait()`, which suspends the task with zero CPU cost until the VLM loop calls `event.set()`. This avoids repeated LLM inference on unchanged story batches and eliminates the "re-process same stories" problem that polling would cause once `StoriesBuffer` is saturated.

### Concurrency model

`asyncio` is single-threaded and cooperative. Tasks do not run in true parallel — only one executes Python bytecode at a time. However, `await` yields control to the event loop, which immediately resumes another ready task. Because VLM and LLM calls are HTTP requests (network I/O), the actual waiting time is in the network stack, not in Python. All in-flight requests proceed simultaneously at the OS level.

When cameras share a provider (e.g., cam-1 and cam-2 both use `openai:gpt-4o`), their requests fly concurrently because both `await vlm.process()` calls are in-flight at the same time — the event loop interleaves them at each `await`.

## Consequences

- `ProcessingManager.run()` and `_process_camera()` are removed. The main processing unit is now `_vlm_loop()` / `_llm_loop()` per camera.
- `_camera_tasks: dict[str, tuple[Task, Task]]` and `_llm_events: dict[str, asyncio.Event]` are added to `ProcessingManager`.
- `remove_camera()` and `update_camera()` become `async` (they must cancel and await the running tasks).
- `main.py` webhook handler calls `await _processing_manager.remove_camera(...)` and `await _processing_manager.update_camera(...)`.
- No `threading.Lock` is needed inside asyncio tasks (single thread, cooperative scheduling guarantees no interleaving within a `await`-free code block).
- `FrameBuffer` and `StoriesBuffer` **retain** their `threading.Lock` because they are shared between OS capture threads (writers) and the asyncio event loop (readers).
