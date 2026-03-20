# ADR-007: Choosing the Message Broker for the Real-Time Event Bus

**Status:** Proposed  
**Date:** March 2026

## Context

Argos runs in both large clusters and edge appliances. In the appliance scenario, Argos performs **real-time camera analysis** using a VLM pipeline and emits detection events to the user-facing API and dashboard.

Architecture (simplified):

- **Engine** ingests camera streams (RTSP/HTTP), buffers frames and intermediate “stories”, runs:
  - **vLM Processor** (per-camera `ObservationConfig`)
  - **LLM Detector** (per-camera `DecisionConfig`)
- The Engine publishes **events (JSON metadata)** to a message broker.
- **BFF** subscribes to events, stores evidence, and streams updates to the Web client via **SSE**.
- Configuration is managed by the BFF and pushed to the Engine via **HTTP** (startup pull + runtime push).

In our larger clusters we previously used Kafka for event streaming. For the appliance, the key requirements differ:

- **Performance first:** very low latency and high throughput for real-time UX
- **Best-effort OK:** persistence/replay is not required for the event bus
- **Small footprint:** single-node appliance, minimal operational overhead
- **Fanout:** multiple subscribers are possible (SSE, storage, alerting)

We must choose the message broker for **Engine → BFF** event delivery.

## Decision Drivers

- **Low latency** delivery for real-time detections to the Web dashboard (SSE)
- **High throughput** with bursts (camera scenes can generate event spikes)
- **Simplicity** of deployment and operations on a single-node appliance
- **Loose coupling** between Engine and BFF (ability to add consumers without changing Engine)
- **Clear backpressure behavior:** when downstream lags, preserve “now” over “eventual delivery”

## Options Considered

### Option A: Kafka (Rejected)

Kafka for Engine → BFF events.

**Pros**
- Strong durability, replay, and ordering guarantees
- Mature ecosystem and tooling
- Works well in large clusters for multi-tenant streaming

**Cons**
- Operational overhead and storage tuning are high for an appliance
- Resource footprint is larger than needed for a best-effort real-time bus
- Durability features are unnecessary for this use case

### Option B: Redis Pub/Sub or Streams (Rejected)

**Pros**
- Simple deployment
- Often already used for caching/session state

**Cons**
- Pub/Sub has limited delivery semantics and observability
- Streams adds persistence/queue semantics we do not need
- Not purpose-built as a high-performance fanout messaging backbone

### Option C: Direct HTTP webhooks / gRPC streaming (Rejected)

**Pros**
- No broker component
- Straightforward in the smallest topology

**Cons**
- Tighter coupling between Engine and BFF
- Backpressure/fanout become application concerns
- Harder to add additional consumers (rules, alerting, integrations)

### Option D: NATS Core (Chosen)

Use **NATS (Core)** as the in-memory, low-latency event bus between Engine and BFF.

**Pros**
- Very low latency and high throughput
- Small CPU/RAM footprint; excellent fit for appliances
- Simple operations (single server, minimal config)
- Native pub/sub and fanout; easy to add new subscribers
- Strong client libraries for Python and TypeScript

**Cons**
- Best-effort delivery (no replay/durability by default)
- Requires explicit backpressure and drop policy in real-time pipelines

## Decision

Adopt **NATS Core** as the message broker for **Engine → BFF** events in the appliance real-time pipeline.

- The event bus is **best-effort** by design
- Events carry the **full detection payload**, including encoded frames and stories used as evidence
- When consumers lag, the system prioritizes **freshness**:
  - **drop/skip stale events** rather than increasing latency

## Consequences

### Positive
- Faster and more responsive real-time UX (SSE updates)
- Significantly lower operational overhead vs Kafka on appliances
- Decoupling preserved: Engine publishes once, BFF and other services subscribe as needed
- Easier local development and edge deployments

### Negative / Trade-offs
- No built-in replay of events; missed events are acceptable in this mode
- If later we require “at-least-once” or replay, we must:
  - selectively enable **NATS JetStream** for specific subjects, or
  - rely on evidence persistence in **EVDB** and treat NATS as the live bus


### Observability

- Expose NATS server metrics and subscriber throughput/lag
- Propagate trace/span IDs (headers) from Engine to BFF for end-to-end visibility

## Follow-ups

- Define canonical event schemas and validation rules
- Add load tests based on expected camera count and event rates
- Document JetStream activation criteria (only if durability becomes a requirement)

## Optional hardening (future)

If you want to make this ADR more “closed” later, add:

- **Quantitative targets** (e.g., “target p95 < 200ms Engine→Web”, “X events/s per camera”)
- A concrete drop policy (e.g., **latest-wins per `cameraId`**)
- A complementary ADR: “Enable JetStream only for audit/critical alerts”