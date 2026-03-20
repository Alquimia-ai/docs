# ADR-009: Canonical Event Schema and NATS Subject Hierarchy

**Status:** Proposed  
**Date:** March 2026

## Context

ADR-007 adopted **NATS Core** as the message broker for Engine → BFF event delivery. The Engine publishes **detection events** as JSON metadata through NATS, and the BFF subscribes to store evidence and stream updates via SSE.

Currently, the `DetectedEvent` model is tightly coupled to the Engine's internal pipeline. It mixes **internal processing state** (e.g., `reasoning`) with the **event payload**, and there is no defined NATS subject hierarchy, making subscriber routing and filtering impossible.

The event must carry the **full detection context** — including the stories and frames that triggered the detection — so that consumers (BFF, dashboard, alerting) have the complete evidence without needing a secondary lookup.

We need to define:
1. A **canonical event envelope** — the wire-format schema published to NATS
2. A **NATS subject hierarchy** — enabling subscribers to filter by camera and event type

## Decision Drivers

- **Self-contained events**: each event must carry all evidence (frames + stories) so consumers can process without secondary queries
- **Loose coupling**: the wire schema must be independent of Engine internals so BFF and future consumers are not coupled to the pipeline model
- **Subscriber flexibility**: subjects must allow granular filtering (per camera, per event type, or wildcard)
- **Extensibility**: the envelope must support future metadata without breaking existing consumers

## Proposed Event Envelope

The canonical event published to NATS is a JSON object with this structure:

```json
{
  "event_type": "intrusion",
  "camera_id": "cam-lobby-01",
  "timestamp": "2026-03-05T14:00:00Z",
  "description": "A person entered the restricted area through the side door",
  "confidence": 0.87,
  "severity": "high",
  "stories": [
    {
      "description": "A person wearing dark clothing approaches the side door",
      "timestamp": "2026-03-05T13:59:55Z",
      "frames": ["<base64-encoded JPEG>"]
    }
  ],
  "metadata": {}
}
```

### Field Definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `event_type` | `string` | ✅ | Classification of the detected event |
| `camera_id` | `string` | ✅ | Identifier of the source camera |
| `timestamp` | `string` (ISO 8601) | ✅ | Detection timestamp |
| `description` | `string` | ✅ | Human-readable event description |
| `confidence` | `float` [0.0, 1.0] | ✅ | Detection confidence score |
| `severity` | `enum` (low, medium, high, critical) | ✅ | Event severity level |
| `stories` | `array` | ✅ | Stories used for detection, each with description + base64 frames |
| `metadata` | `object` | ❌ | Extensible key-value metadata for future use |

### Stories Structure

Each story in the `stories` array contains:

| Field | Type | Description |
|---|---|---|
| `description` | `string` | Natural language description of the observation |
| `timestamp` | `string` (ISO 8601) | Observation timestamp |
| `frames` | `array[string]` | Base64-encoded JPEG frames that produced the observation |

### What is excluded from the wire schema

| Excluded field | Reason |
|---|---|
| `reasoning` | Internal LLM artifact, not relevant for consumers |


## NATS Subject Hierarchy

### Structure

```
argos.events.<camera_id>.<event_type>
```

### Examples

| Subject | Description |
|---|---|
| `argos.events.cam-lobby-01.intrusion` | Intrusion event from lobby camera |
| `argos.events.cam-parking.fire` | Fire event from parking camera |
| `argos.events.cam-entrance.anomaly` | Anomaly from entrance camera |

### Subscriber Patterns (NATS Wildcards)

| Pattern | Matches | Use Case |
|---|---|---|
| `argos.events.>` | All events from all cameras | BFF main subscriber, SSE |
| `argos.events.cam-lobby-01.>` | All events from a specific camera | Per-camera dashboard |
| `argos.events.*.intrusion` | Intrusion events from any camera | Alert-specific subscribers |

## Decision

Adopt the canonical event envelope and subject hierarchy described above.

- The **`EventSerializer`** in the Engine is responsible for transforming a `DetectedEvent` (internal model) into the canonical wire-format `bytes` (JSON-encoded envelope), stripping internal fields
- **`NatsPublisher`** receives serialized bytes and publishes to the computed subject (`argos.events.<camera_id>.<event_type>`)

## Data Flow

```
Engine                       NATS                        BFF                        Web Client
  │                           │                           │                           │
  │  EventSerializer:         │                           │                           │
  │  DetectedEvent → JSON     │                           │                           │
  │  (frames as base64)       │                           │                           │
  │                           │                           │                           │
  ├──publish(subject, bytes)─►│                           │                           │
  │                           ├──deliver(event)──────────►│                           │
  │                           │                           │                           │
  │                           │                           │  Persist to EVDB:         │
  │                           │                           │  - frames (decoded)       │
  │                           │                           │  - stories                │
  │                           │                           │  - event metadata          │
  │                           │                           │                           │
  │                           │                           ├──SSE: event + refs───────►│
  │                           │                           │  (frame URLs, not base64) │
  │                           │                           │                           │
  │                           │                           │◄──GET /evidence/frame─────┤
  │                           │                           ├──frame binary────────────►│
```

1. **Engine** serializes the `DetectedEvent` into the canonical envelope (frames as base64, excluding `reasoning`) and publishes via NATS
2. **BFF** receives the full event, decodes the frames and persists everything in **EVDB** (frames, stories, metadata)
3. **BFF** notifies the **Web Client** via SSE with event metadata and **references** to the already-persisted frames (URLs/IDs)
4. **Web Client** fetches frames via REST when it needs to display them

## Consequences

### Positive
- Clean separation between internal pipeline model and wire schema
- Self-contained events: consumers have full evidence (frames + stories) without secondary queries
- Flexible subscriber routing via subject hierarchy and NATS wildcards
- BFF and future consumers depend only on the canonical envelope, not Engine internals
- SSE stays lightweight: event metadata + frame references instead of inline base64

### Negative / Trade-offs
- `EventSerializer` must be updated to strip internal fields and encode to bytes
- NATS messages are larger due to base64-encoded frames; acceptable for appliance (single-node, low-latency bus)
- Adding new fields to the envelope requires coordinating Engine + BFF schema updates

## Follow-ups

- Implement `NatsPublisher`
- Update `EventSerializer` to produce canonical envelope as `bytes`
- Define validation rules for `event_type` values (free-form string vs. enum)
- Add integration tests: Engine → NATS → BFF subscriber
