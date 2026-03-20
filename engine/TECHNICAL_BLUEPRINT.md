# Argos Engine — Technical Blueprint

**Version:** 1.3
**Date:** February 2026
**Framework:** FastAPI + LangChain

---

## Architecture

The engine follows a **pipeline architecture** with decoupled stages connected via buffers, and an in-memory registry as the central state hub.

```
Camera → PreProcessor → FrameBuffer (Registry) → VLM → Aggregator → Story → StoriesBuffer (Registry) → LLM → EventNormalizer → Publisher
```

Each camera runs this pipeline **independently and concurrently**. See the [Concurrency Model](#concurrency-model) section for details.

### Module Structure

```
apps/engine/src/engine/
├── registry/                # Hot state — singleton registry + per-camera buffers
│   ├── __init__.py
│   └── registry.py          # EngineStateRegistry (singleton), CameraBuffers
├── preprocessor/            # Camera capture and frame processing
│   ├── cameras_config.py    # CameraConfig, ConnectionConfig, FrameConfig,
│   │                        # CameraModelConfig, ObservationConfig, DecisionConfig
│   ├── camera_connector.py  # Base + RTSP + Local connectors (push frames → registry)
│   ├── connector_factory.py # Protocol-based factory
│   ├── frame_processor.py   # Resize + JPEG encode
│   └── manager.py           # Thread-per-camera orchestrator + dynamic add/remove/update
├── buffers/                 # Thread-safe data buffers
│   ├── frame_buffer.py      # FrameBuffer — FIFO queue, capped at frame_buffer_size, dequeues frame_batch_size per flush
│   ├── stories_buffer.py    # StoriesBuffer — FIFO queue, capped at stories_buffer_size, dequeues stories_batch_size per get_batch
│   └── policies.py          # BufferFetchPolicy protocol + FIFOPolicy
├── processing/              # AI processing stages
│   ├── vlm_processor.py     # VLMProcessor — frames → Description via VLM
│   ├── aggregator.py        # Aggregator — descriptions + frames → Story
│   ├── llm_detector.py      # LLMDetector — stories → Event via structured output
│   ├── event_normalizer.py  # EventNormalizer — runs ValidationCommands
│   ├── commands/            # Concrete ValidationCommand implementations
│   │   ├── confidence_threshold.py
│   │   ├── severity_normalization.py
│   │   ├── metadata_cleanup.py
│   │   └── frame_encoding.py
│   └── manager.py           # ProcessingManager — per-camera asyncio tasks
├── postprocessor/           # Event serialization + publishing
│   ├── event_serializer.py  # Serialize Event for Kafka
│   └── publisher.py         # Kafka publisher (stub)
├── models.py                # Frame, Description, Story, LLMDetection, DetectedEvent
├── pipeline.py              # Abstract pipeline interfaces (Buffer, EventPublisher)
├── model_factory.py         # create_model() — LangChain init_chat_model wrapper
├── config.py                # Global settings (Pydantic)
└── main.py                  # FastAPI app + lifespan + webhook endpoint
```

---

## Concurrency Model

Understanding where concurrency happens (and where it does not) is critical to reasoning about the engine's behavior.

### Two concurrency mechanisms coexist

| Layer | Mechanism | Reason |
|-------|-----------|--------|
| Frame capture (PreProcessor) | `threading.Thread` — one per camera | OpenCV's `VideoCapture.read()` is a blocking C call; threads allow cameras to capture in parallel |
| AI processing (ProcessingManager) | `asyncio.Task` — two per camera | VLM and LLM calls are async HTTP requests; asyncio multiplexes them on a single thread |

### Where parallelism IS real

**Capture threads** run in true OS-level parallel. OpenCV and RTSP I/O are implemented in C and release Python's GIL, so three capture threads genuinely run simultaneously.

**HTTP requests** to VLM/LLM providers are multiplexed by the asyncio event loop. When `cam-1` does `await vlm.process(frames)`, Python yields to the event loop, which immediately advances `cam-2`'s loop to send its own request. Both HTTP calls are in-flight at the OS level at the same time — even if they target the same provider endpoint.

```
Event loop (single thread):
  cam-1 vlm: ──send──[   waiting for HTTP response   ]──process──
  cam-2 vlm:       ──send──[   waiting for HTTP response   ]──process──
  cam-3 llm:            ──send──[  waiting for HTTP response  ]──process──
```

### Where parallelism is NOT real

Python bytecode execution is serialized by the **GIL (Global Interpreter Lock)**: only one thread runs Python code at a time. This means:

- Asyncio tasks do not run Python code in parallel — when `cam-1`'s task runs Python (aggregating stories, checking buffer state), the others wait.
- This is irrelevant in practice: those operations take microseconds. The seconds-long HTTP calls are the bottleneck, and those are concurrent.

### Why buffers still need `threading.Lock`

`FrameBuffer` and `StoriesBuffer` are written by OS capture threads and read by asyncio tasks. These are different threads — the GIL alone does not prevent race conditions between them (it can switch threads between any two bytecode instructions). The `threading.Lock` on each buffer guarantees atomic read/write.

Asyncio tasks share no mutable state between themselves (each task accesses only its own camera's buffers via the registry), so no additional locking is needed at the asyncio level.

### Per-camera task model

```
Startup:
  for each camera:
    asyncio.create_task(_vlm_loop(camera_id))
    asyncio.create_task(_llm_loop(camera_id))

_vlm_loop(cam_id):           _llm_loop(cam_id):
  loop forever:                 loop forever:
    if frame_buffer ready:        await event.wait()   ← sleeps until VLM signals (or backlog re-signal)
      flush frames                event.clear()
      await vlm.process()  ←      stories = stories_buffer.get_batch()  ← pops batch_size from front (FIFO)
        (HTTP, yields to           await llm.detect(stories)  ← (HTTP, yields)
         event loop)               normalizer.normalize(event)
      aggregate → stories         if stories_buffer ready:   ← backlog still present?
      stories_buffer.add()            event.set()  ◄──────── re-signal self to drain next batch
      if stories_buffer ready:
        event.set()  ──────────► wakes _llm_loop
```

Camera configs can share providers (e.g., cam-1 and cam-2 both use `openai:gpt-4o` with different prompts). Each camera has its own `VLMProcessor` / `LLMDetector` Python instance. When both send requests concurrently, the provider receives two independent HTTP requests and responds to each independently.

---

## Class Diagram

See [class-diagram-engine.mmd](../architecture/class-diagram-engine.mmd) for the full class diagram.

---

## Configuration

### Engine Settings (`config.py`)

Root settings loaded from environment variables with `ARGOS_` prefix. Model configuration has moved to per-camera `CameraConfig` — there are no global LLM/VLM settings.

| Variable | Description | Default |
|----------|-------------|---------|
| `ARGOS_HOST` | Bind address | `0.0.0.0` |
| `ARGOS_PORT` | Server port | `8000` |
| `ARGOS_LOG_LEVEL` | Logging level | `info` |
| `ARGOS_BFF_BASE_URL` | BFF service base URL | `http://localhost:3000` |
| `ARGOS_BUFFER_FRAME_BUFFER_SIZE` | Max frames held in memory; oldest evicted when full | `10` |
| `ARGOS_BUFFER_FRAME_BATCH_SIZE` | Frames sent to VLM per inference cycle; triggers `is_ready()` | `10` |
| `ARGOS_BUFFER_STORIES_BUFFER_SIZE` | Max stories held in memory; oldest evicted when full | `20` |
| `ARGOS_BUFFER_STORIES_BATCH_SIZE` | Stories sent to LLM per inference cycle; triggers `is_ready()` | `5` |

### Camera Configuration (`cameras_config.py`)

On startup the Engine loads config from `ARGOS_CAMERAS_CONFIG` (BFF pull-on-start is deferred until the BFF is implemented). At runtime the BFF sends config changes via webhook (`POST /config/cameras`).

For the full schema see [API_CONTRACT.md](./API_CONTRACT.md).

```json
[
  {
    "id": "cam-001",
    "name": "Main Entrance",
    "enabled": true,
    "connection": {
      "stream_url": "rtsp://192.168.1.100:554/stream",
      "protocol": "rtsp",
      "frame_interval": 1.0
    },
    "frame": {
      "target_width": 1280,
      "target_height": 720,
      "jpeg_quality": 85
    },
    "observation": {
      "model": "openai:gpt-4o",
      "kwargs": {},
      "system_prompt": "Describe all persons and vehicles observed in this frame."
    },
    "decision": {
      "model": "openai:gpt-4o",
      "kwargs": {},
      "system_prompt": "Detect unauthorized access or suspicious activity.",
      "output_schema": {}
    }
  }
]
```

---

## EngineStateRegistry

Singleton that serves as the central hot-storage hub:
- Stores `CameraConfig` per camera ID (including model config and prompts)
- Allocates and owns a `CameraBuffers` (FrameBuffer + StoriesBuffer) per camera
- Thread-safe via `threading.Lock()`
- `CameraConnector.run_capture_loop()` pushes frames directly into the registry's per-camera FrameBuffer

---

## PreProcessor

Each camera gets its own capture thread managed by `PreprocessorManager`.

### Factory Pattern

`CameraConnectorFactory` dispatches the correct connector by protocol:
- `CameraProtocol.RTSP` → `RtspCameraConnector`
- `CameraProtocol.LOCAL` → `LocalCameraConnector`

New protocols are added via `CameraConnectorFactory.register()`.

### Dynamic Operations

`PreprocessorManager` supports hot-swap without restart:
- `add_camera(config)` — start a new capture thread
- `remove_camera(camera_id)` — stop and join the thread
- `update_camera(config)` — remove then re-add (restart with new config)

---

## Processing Pipeline

### FrameBuffer
- FIFO queue per camera in the registry, capped at `frame_buffer_size`. Oldest frame is evicted when full.
- `flush()` dequeues exactly `frame_batch_size` frames from the front via the configured `BufferFetchPolicy`, leaving any remainder for the next VLM cycle.
- `is_ready()` triggers when `len >= frame_batch_size`.

### VLM Processor
- Receives the frame batch produced by `FrameBuffer.flush()` when `is_ready()` triggers
- Uses the camera's `ObservationConfig` (model + prompt) via `ProcessingManager`
- Outputs `Description` objects

### Aggregator
- Receives `Description` list + source `Frame` list
- Outputs `Story` objects (description bundled with source frames)

### StoriesBuffer
- FIFO queue of `Story` objects per camera in the registry. Stories accumulate freely while the LLM is busy.
- `get_batch()` dequeues exactly `stories_buffer_size` stories from the front via the configured `BufferFetchPolicy`, leaving any remainder for the next LLM cycle.
- The default policy is `FIFOPolicy`; future policies (e.g. Round Robin) can be injected at construction time without changing the buffer.
- When threshold is reached, VLM loop signals the LLM loop via `asyncio.Event`.

### LLM Detector
- Receives stories batch from `StoriesBuffer`
- Uses the camera's `DecisionConfig` (model + prompt + `output_schema`)
- `output_schema` is passed to `with_structured_output()` — defaults to built-in `LLMDetection` schema
- If `detected=true` and `event_type` present, emits a `DetectedEvent`

### ProcessingManager
- Maintains `_vlm_cache` and `_llm_cache`: `dict[camera_id, processor]`
- Builds processors from `CameraConfig` via `model_factory.create_model()`
- On `start()`, launches two `asyncio.Task` per camera: `_vlm_loop` and `_llm_loop`
- `add_camera()` — caches processors and immediately starts tasks if already running
- `remove_camera()` — cancels tasks and evicts cache (async)
- `update_camera()` — remove then re-add (async)

### EventNormalizer (Command Pattern)
- Holds a list of `ValidationCommand` objects, executed sequentially
- If any command returns `None`, the event is dropped
- Commands: `ConfidenceThresholdCheck`, `SeverityNormalization`, `MetadataCleanup`, `FrameEncoding`

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | Health check |
| GET | `/info` | Pipeline component status |
| GET | `/cameras` | List registered cameras + thread status |
| POST | `/config/cameras` | BFF webhook — apply camera config changes |

For full request/response schemas see [API_CONTRACT.md](./API_CONTRACT.md).

---

## Running

```bash
# Local
cd apps/engine
uv venv && source .venv/bin/activate
uv sync --all-extras
cp ../../.env.example ../../.env  # edit as needed
source ../../.env && uv run uvicorn engine.main:app --host $ARGOS_HOST --port $ARGOS_PORT

# Docker
docker compose up --build
```
