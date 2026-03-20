# ADR-006 — Buffer Fetch Policy: FIFO Queue with Strategy Pattern

**Status:** Accepted
**Date:** February 2026

---

## Context

The engine's two inference stages run at different speeds:

- **VLM** describes frames relatively quickly.
- **LLM** uses structured output (JSON schema enforcement), which is slower.

When the LLM is busy, frames and stories continue to accumulate in their respective buffers. The original buffer design had two failure modes under this load:

1. **`FrameBuffer.add()` drops frames when full.** Once the buffer reaches `max_size`, new frames arriving from the capture thread are silently discarded, permanently losing visual data.

2. **`StoriesBuffer.get_batch()` returns everything without consuming items.** The LLM receives the full rolling window on each call, even if there is a large backlog of stories that haven't been processed yet. There is no clean way to drain a backlog incrementally.

The desired behavior is: buffers should accumulate freely while inference is running, and each inference cycle should dequeue exactly `buffer_size` items from the front, leaving the remainder for the next cycle. This is a classic FIFO queue.

Additionally, there is an interest in making the fetch strategy pluggable. Future use cases (e.g., Round Robin across cameras, priority-based sampling) should be injectable without modifying the buffer classes themselves.

---

## Options

### Option A — Keep drop guard, add separate `batch_size` config

Keep `FrameBuffer`'s capacity guard (drop on full) but add a separate `batch_size` parameter that controls how many frames `flush()` returns. `StoriesBuffer` gains an analogous param.

**Rejected because:**
- Dropping frames on full is still the failure mode — it just triggers less often.
- Does not address the inability to drain a backlog cleanly.
- Adds a second size parameter that is confusing alongside `max_size`.

### Option B — FIFO queue with Strategy pattern (chosen)

Remove the drop guard entirely. Both buffers become unbounded FIFO queues. A `BufferFetchPolicy` protocol defines a single `fetch(items, batch_size)` method. `FIFOPolicy` is the default concrete implementation: it pops the front `batch_size` items from the deque, leaving the rest.

Buffers delegate `flush()` / `get_batch()` to the injected policy. The `_llm_loop` re-signals its `asyncio.Event` after `get_batch()` if the buffer is still `is_ready()`, draining the backlog batch by batch without waiting for the VLM.

**Selected because:**
- No data loss: frames and stories accumulate freely.
- Clean backlog draining: each inference cycle processes exactly `batch_size` items.
- Extensible: new fetch strategies (Round Robin, priority) are injected at construction time without touching buffer or loop code.

---

## Decision

Adopt **Option B**.

### New file: `buffers/policies.py`

```
BufferFetchPolicy   ← protocol (single method: fetch)
FIFOPolicy          ← concrete, dequeues front N items
```

### `FrameBuffer` changes

- `_frames` changes from `list[Frame]` to `deque[Frame]` (unbounded).
- `add()` no longer has a capacity guard — appends unconditionally.
- `flush()` delegates to `_policy.fetch(self._frames, self.max_size)`, popping the front `max_size` frames and leaving the remainder.
- `is_ready()` still returns `True` when `len >= max_size`.
- Constructor accepts an optional `policy: BufferFetchPolicy = FIFOPolicy()`.

### `StoriesBuffer` changes

- `_stories` deque loses `maxlen=` (becomes unbounded).
- `get_batch()` delegates to `_policy.fetch(self._stories, self.batch_size)`, popping front `batch_size` stories.
- `flush()` continues to return and clear everything (used for explicit drains).
- `is_ready()` still returns `True` when `len >= batch_size`.
- Constructor accepts an optional `policy: BufferFetchPolicy = FIFOPolicy()`.

### `_llm_loop` change

After `get_batch()` consumes a batch, if `stories_buffer.is_ready()` is still `True` (backlog remains), the loop re-sets the `asyncio.Event` immediately. This drains the backlog batch by batch on the asyncio event loop without spinning in a tight loop and without waiting for the VLM to produce new stories.

```
_llm_loop(cam_id):
  loop forever:
    await event.wait()         ← sleeps until VLM signals (or backlog re-signal)
    event.clear()
    stories = stories_buffer.get_batch()   ← pops batch_size from front (FIFO)
    await llm.detect(stories)
    normalizer.normalize(event)
    if stories_buffer.is_ready():          ← backlog still present?
        event.set()            ← re-signal self to drain next batch
```

---

## Consequences

- **Positive:** No frame or story loss under inference backlog.
- **Positive:** LLM backlog drains incrementally; the event loop remains responsive.
- **Positive:** Fetch strategy is a constructor dependency — easy to test and swap.
- **Negative:** Buffers are now unbounded; a persistently overloaded system will grow memory indefinitely. A separate circuit-breaker or back-pressure mechanism may be needed in the future if this becomes a concern.
- **Neutral:** `StoriesBuffer.get_batch()` is no longer a read-only peek — it consumes items. Callers that relied on the previous non-destructive semantics must be updated.
