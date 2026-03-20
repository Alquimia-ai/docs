# Argos Engine — BFF ↔ Engine API Contract

**Version:** 1.0
**Date:** February 2026
**Audience:** BFF developers

This document specifies the full interface contract between the BFF and the Engine. It covers:
- The camera config schema the BFF must provide
- The startup endpoint the BFF must expose
- The webhook the Engine exposes for runtime config changes

> **Not covered here:** events published by the Engine to the message broker (`argos.events` topic). That contract is TBD.

---

## Communication Model

```
Startup:  Engine ──GET /api/cameras/engine──► BFF
Runtime:  BFF ──POST /config/cameras────────► Engine
Events:   Engine ──Kafka argos.events topic──► BFF  (TBD)
```

The pattern is **pull-on-start, push-on-change** (see [ADR-003](../adr/003-config-propagation-webhook.md)):
1. On startup the Engine pulls the full camera config from the BFF
2. At runtime the BFF pushes individual changes via webhook
3. If the Engine is down during a webhook, the next startup reconciles state

---

## 1. CameraConfig Schema

This is the canonical schema for a camera. Both the BFF startup response and the webhook payload use this object.

```json
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
    "system_prompt": "Monitor the entrance and describe all persons and vehicles you observe."
  },
  "decision": {
    "model": "openai:gpt-4o",
    "kwargs": {},
    "system_prompt": "Analyze the observation history. Raise an alert if an unauthorized person is detected after business hours.",
    "output_schema": {}
  }
}
```

### `connection` fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `stream_url` | `string` | **required** | RTSP URL or local device index (e.g. `"0"`) |
| `protocol` | `"rtsp" \| "local"` | `"rtsp"` | Connection protocol |
| `frame_interval` | `float` | `1.0` | Seconds between frame captures |

### `frame` fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `target_width` | `int` | `1280` | Frame resize width (pixels) |
| `target_height` | `int` | `720` | Frame resize height (pixels) |
| `jpeg_quality` | `int` | `85` | JPEG encode quality (1–100) |

### `observation` fields (VLM config)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `model` | `string` | `"openai:gpt-4o"` | Model in `provider:model` format — passed to LangChain's `init_chat_model` |
| `kwargs` | `object` | `{}` | Extra kwargs forwarded to `init_chat_model` (e.g. `temperature`, `api_key`, `base_url`) |
| `system_prompt` | `string` | see default | Instruction sent to the VLM alongside each frame |

### `decision` fields (LLM config)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `model` | `string` | `"openai:gpt-4o"` | Model in `provider:model` format — passed to LangChain's `init_chat_model` |
| `kwargs` | `object` | `{}` | Extra kwargs forwarded to `init_chat_model` |
| `system_prompt` | `string` | see default | Instruction sent to the LLM alongside the stories history |
| `output_schema` | `object` | `{}` | JSON Schema for structured output passed to `with_structured_output()`. When empty, the Engine uses the built-in `LLMDetection` schema (see below) |

#### Default `output_schema` (LLMDetection)

When `output_schema` is empty or omitted, the Engine uses this schema:

```json
{
  "type": "object",
  "properties": {
    "detected":    { "type": "boolean", "description": "Whether a notable event was detected" },
    "event_type":  { "type": "string",  "description": "Classification of the detected event" },
    "confidence":  { "type": "number",  "minimum": 0, "maximum": 1 },
    "severity":    { "type": "string",  "enum": ["low", "medium", "high", "critical"] },
    "description": { "type": "string",  "description": "Human-readable event description" },
    "metadata":    { "type": "object",  "description": "Additional event context" }
  },
  "required": ["detected"]
}
```

Custom schemas must include at minimum a `detected` boolean and an `event_type` string for the Engine to emit an event.

#### `model` format and `kwargs` examples

The `model` field uses LangChain's `provider:model` format:

| Provider | Example `model` value |
|----------|----------------------|
| OpenAI | `"openai:gpt-4o"`, `"openai:gpt-4o-mini"` |
| Anthropic | `"anthropic:claude-3-5-sonnet-20241022"` |
| Ollama (local) | `"ollama:llava"`, `"ollama:llama3.2"` |
| Azure OpenAI | `"azure_openai:gpt-4o"` |

Common `kwargs`:

```json
{
  "temperature": 0.0,
  "max_tokens": 1024,
  "api_key": "sk-...",
  "base_url": "https://custom-endpoint/v1"
}
```

---

## 2. BFF Startup Endpoint

The Engine calls this on startup to load its initial camera configuration.

### `GET /api/cameras/engine`

This is an **internal endpoint** — it is not part of the public BFF API exposed to the frontend. No authentication is required from the Engine side (internal network communication).

**Response:** `200 OK`

```json
{
  "cameras": [
    {
      "id": "cam-001",
      "name": "Main Entrance",
      "enabled": true,
      "connection": { "stream_url": "rtsp://...", "protocol": "rtsp", "frame_interval": 1.0 },
      "frame": { "target_width": 1280, "target_height": 720, "jpeg_quality": 85 },
      "observation": { "model": "openai:gpt-4o", "kwargs": {}, "system_prompt": "..." },
      "decision": { "model": "openai:gpt-4o", "kwargs": {}, "system_prompt": "...", "output_schema": {} }
    }
  ]
}
```

**Failure behavior:** If this endpoint is unreachable or returns an error, the Engine falls back to the `ARGOS_CAMERAS_CONFIG` environment variable. A warning is logged.

---

## 3. Engine Webhook Endpoint

The BFF calls this endpoint to notify the Engine of camera configuration changes at runtime.

### `POST /config/cameras`

**Base URL:** `http://<ENGINE_HOST>:<ENGINE_PORT>` (configured via `ENGINE_URL` env var in the BFF)

#### Request

```
POST /config/cameras
Content-Type: application/json
```

```json
{
  "type": "camera:created | camera:updated | camera:deleted",
  "cameraId": "cam-001",
  "config": { /* CameraConfig — omit for camera:deleted */ },
  "timestamp": "2026-02-24T10:00:00Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `"camera:created" \| "camera:updated" \| "camera:deleted"` | ✅ | Change type |
| `cameraId` | `string` | ✅ | Camera identifier |
| `config` | `CameraConfig` | Only for `created` and `updated` | Full camera config object |
| `timestamp` | `ISO 8601 datetime` | ✅ | When the change occurred |

#### Event type semantics

| Event | Engine behavior |
|-------|----------------|
| `camera:created` | Registers camera in registry, allocates buffers, starts capture thread, initializes VLM + LLM processors |
| `camera:updated` | Replaces config in registry, restarts capture thread with new connection settings, rebuilds VLM + LLM processors with new prompts/models |
| `camera:deleted` | Stops capture thread, evicts processor cache, releases buffers |

#### Response `200 OK`

```json
{
  "status": "applied",
  "cameras": [
    { "id": "cam-001", "name": "Main Entrance", "enabled": true },
    { "id": "cam-002", "name": "Parking Lot",   "enabled": true }
  ]
}
```

`cameras` reflects the Engine's current registered camera list after applying the change.

#### Error responses

| Status | When |
|--------|------|
| `422 Unprocessable Entity` | `config` is missing for `camera:created` or `camera:updated`, or schema validation fails |
| `500 Internal Server Error` | Unexpected engine error |

#### Failure handling

If the Engine is unreachable when the BFF sends a webhook:
- The BFF should mark the change as **pending**
- On the next Engine startup, the full config is re-fetched from `GET /api/cameras/engine`, reconciling any missed changes

---

## 4. Full Example — Create Camera

**BFF → Engine:**

```http
POST /config/cameras
Content-Type: application/json

{
  "type": "camera:created",
  "cameraId": "cam-003",
  "config": {
    "id": "cam-003",
    "name": "Server Room",
    "enabled": true,
    "connection": {
      "stream_url": "rtsp://10.0.0.50:554/live",
      "protocol": "rtsp",
      "frame_interval": 2.0
    },
    "frame": {
      "target_width": 1280,
      "target_height": 720,
      "jpeg_quality": 85
    },
    "observation": {
      "model": "openai:gpt-4o",
      "kwargs": { "temperature": 0.0 },
      "system_prompt": "Describe all persons and activities visible in the server room."
    },
    "decision": {
      "model": "openai:gpt-4o-mini",
      "kwargs": { "temperature": 0.0 },
      "system_prompt": "Alert if any unauthorized person is detected or any equipment is tampered with.",
      "output_schema": {}
    }
  },
  "timestamp": "2026-02-24T14:30:00Z"
}
```

**Engine response:**

```json
{
  "status": "applied",
  "cameras": [
    { "id": "cam-001", "name": "Main Entrance", "enabled": true },
    { "id": "cam-002", "name": "Parking Lot",   "enabled": true },
    { "id": "cam-003", "name": "Server Room",   "enabled": true }
  ]
}
```

---

## 5. Full Example — Update Prompts

Only the `observation.system_prompt` changes. The Engine rebuilds the VLM processor for this camera without restarting the capture thread.

```http
POST /config/cameras
Content-Type: application/json

{
  "type": "camera:updated",
  "cameraId": "cam-003",
  "config": {
    "id": "cam-003",
    "name": "Server Room",
    "enabled": true,
    "connection": {
      "stream_url": "rtsp://10.0.0.50:554/live",
      "protocol": "rtsp",
      "frame_interval": 2.0
    },
    "frame": { "target_width": 1280, "target_height": 720, "jpeg_quality": 85 },
    "observation": {
      "model": "openai:gpt-4o",
      "kwargs": { "temperature": 0.0 },
      "system_prompt": "Focus only on persons. Ignore equipment and cables."
    },
    "decision": {
      "model": "openai:gpt-4o-mini",
      "kwargs": { "temperature": 0.0 },
      "system_prompt": "Alert if any person is detected outside of business hours (09:00–18:00).",
      "output_schema": {}
    }
  },
  "timestamp": "2026-02-24T15:00:00Z"
}
```

---

## 6. Full Example — Delete Camera

```http
POST /config/cameras
Content-Type: application/json

{
  "type": "camera:deleted",
  "cameraId": "cam-003",
  "timestamp": "2026-02-24T16:00:00Z"
}
```

Note: `config` is omitted for `camera:deleted`.

---

## 7. Engine → BFF: Events (TBD)

The Engine publishes detected events to the Kafka `argos.events` topic. The BFF consumes this topic and delivers events to the frontend via SSE.

The event schema and Kafka topic configuration are not yet finalized. This section will be completed when the postprocessor layer is implemented.
