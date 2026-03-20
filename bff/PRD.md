# Argos BFF - Product Requirements Document (PRD)

**Version**: 1.3
**Date**: February 2026
**Status**: Ready for Implementation
**Owner**: Alquimia Engineering Team

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Product Overview](#product-overview)
3. [User Personas](#user-personas)
4. [Functional Requirements](#functional-requirements)
5. [Non-Functional Requirements](#non-functional-requirements)
6. [API Requirements](#api-requirements)
7. [Success Metrics](#success-metrics)
8. [Out of Scope (v1)](#out-of-scope-v1)
9. [Dependencies](#dependencies)
10. [Assumptions & Constraints](#assumptions--constraints)

---

## Executive Summary

The **Argos Backend-For-Frontend (BFF)** is the control plane service that connects the Alquimia Argos frontend application with the Argos video processing engine. It acts as an intelligent intermediary responsible for authentication, configuration management, real-time event distribution, and video stream proxying.

**Why This Matters**:

- The frontend cannot communicate directly with the Argos engine (security, complexity)
- The BFF abstracts engine implementation details and provides a clean REST + SSE API
- Enables horizontal scaling, caching, and rate limiting at the gateway level
- Simplifies frontend development by providing a domain-specific API contract

**Key Capabilities**:

1. Bearer token authentication via static token from environment
2. RESTful API for camera CRUD with embedded prompts (observation and decisions)
3. Real-time event streaming via Server-Sent Events (SSE)
4. Video stream proxying from engine/camera to frontend
5. Persistent storage for events and camera configurations in document database
6. Kafka integration: subscribes to `argos.events` topic for event ingestion. Configuration changes are propagated to the Engine via HTTP webhook (see [ADR-003](../adr/003-config-propagation-webhook.md))

---

## Product Overview

### Context

**Alquimia Argos** is a real-time video surveillance dashboard that enables security operators to monitor multiple camera feeds and receive AI-powered event notifications. The system architecture consists of:

1. **Frontend** (Next.js): User interface for operators
2. **BFF** (This service): API gateway and control plane
3. **Argos Engine**: Video processing and AI detection engine
4. **Apache Kafka**: Event streaming (`argos.events` topic only)
5. **Document Database** (NoSQL): Configuration and event storage
6. **gRPC**: Video stream communication between BFF and engine

### The BFF's Role

The BFF is the **glue layer** that orchestrates all system components. It:

- **Receives** camera configuration updates from frontend (including prompts) --> validates --> saves to DB --> sends HTTP webhook to Engine (`POST /config/cameras`) --> Engine applies changes
- **Subscribes** to Kafka `argos.events` topic --> receives events from engine --> stores in DB --> fans out to frontend via SSE
- **Proxies** video streams from camera (using stored url/credentials) or from engine via gRPC --> frontend (with authentication)
- **Manages** user authentication (static bearer token validation) and authorization (camera access control)
- **Provides** a formal REST + SSE API matching the frontend contract

### Technology Foundation

- **Framework**: Fastify (TypeScript/Node.js)
- **Schema Validation**: Zod
- **Database**: Document database (NoSQL) + native driver
- **Message Broker**: Apache Kafka (`@kafkajs`)
- **Authentication**: Static bearer token from environment (`IAuthProvider` interface)
- **Video Communication**: gRPC (`@grpc/grpc-js`)
- **Architecture**: Hexagonal Architecture (Clean Architecture)

---

## User Personas

### 1. Frontend Developer

**Primary User of BFF API**

**Goals**:

- Consume a well-documented, type-safe API
- Receive real-time events via SSE without managing connections
- Fetch camera configurations for UI rendering
- Submit camera and prompt configuration updates with validation

**Needs**:

- OpenAPI 3.1 specification
- Clear error messages
- Consistent response formats
- TypeScript type definitions

---

### 2. Security Operator
**Indirect User (via Frontend)**

**Goals**:
- Monitor multiple camera feeds simultaneously
- Receive immediate notifications of anomalies
- Configure observation and decision prompts per camera
- Manually trigger notifications for critical events

**Needs from BFF**:
- Low-latency event delivery (<100ms)
- Reliable SSE connections with auto-reconnect
- Configuration changes reflected in engine within 30s
- Accurate camera status reporting

---

### 3. System Administrator
**Deployment & Operations**

**Goals**:
- Deploy BFF to production environment
- Monitor service health and performance
- Scale horizontally as camera count grows
- Troubleshoot issues quickly

**Needs**:
- Comprehensive deployment documentation
- Health check endpoints
- Structured logging with correlation IDs
- Metrics export (OpenTelemetry)
- Docker containerization

---

### 4. DevOps Engineer
**Infrastructure Management**

**Goals**:
- Provision and manage document database, Redis
- Configure environment variables securely
- Implement CI/CD pipelines
- Ensure high availability

**Needs**:
- Infrastructure as Code examples
- Environment variable documentation
- Docker Compose for local development
- Health/readiness probes

---

## Functional Requirements

### FR-001: Authentication & Authorization

**Priority**: P0 (Critical)

**Description**:
Validate a static bearer token (loaded from environment) on all protected endpoints. The auth module uses an `IAuthProvider` interface, allowing future migration to OAuth/OIDC without changing business logic.

**Acceptance Criteria**:

- All `/api/*` endpoints require valid `Authorization: Bearer <token>` header
- Token compared against `AUTH_BEARER_TOKEN` environment variable
- Invalid/missing tokens return `401 Unauthorized` with clear error message
- Camera access control: Users can only access cameras they're authorized for (v2 feature, but structure must support it)
- Auth middleware implemented via Fastify `preHandler` hook
- `IAuthProvider` interface abstracts the token validation strategy, enabling future swap to OAuth/OIDC (e.g., Keycloak) without modifying controllers or services

**Technical Details**:
- Implement `StaticTokenAuthProvider` that reads `AUTH_BEARER_TOKEN` from environment
- Implement Fastify `preHandler` hook for auth middleware
- All auth logic goes through `IAuthProvider` interface (`validateToken(token: string): Promise<AuthResult>`)
- Future providers (e.g., `KeycloakAuthProvider`) can be swapped in via dependency injection

**Example**:
```typescript
// Valid request
Authorization: Bearer my-static-secret-token

// Response: 200 OK with requested data

// Invalid token
Authorization: Bearer wrong_token

// Response: 401 Unauthorized
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid or expired authentication token",
    "timestamp": "2026-02-13T10:30:00Z"
  }
}
```

---

### FR-002: Configuration Management

**Priority**: P0 (Critical)

**Description**:
Manage camera configurations using the engine's `CameraConfig` nested schema (see [`docs/engine/API_CONTRACT.md`](../engine/API_CONTRACT.md)). Camera documents store the full engine config (`connection`, `frame`, `observation`, `decision` sub-objects) extended with BFF-only fields. When a camera is created or updated, validate the document, save to the document database, and send an HTTP webhook to the Engine (`POST /config/cameras`) so it can apply the change in real-time.

**Acceptance Criteria**:

- Camera documents use nested structure: `connection`, `frame`, `observation`, `decision` sub-objects
- On create/update: validate the camera document with Zod schema before saving
- Persist the camera document to the document database
- After saving, send HTTP webhook to Engine `POST /config/cameras` with the change payload (exclude BFF-only fields: `description`, `credentials`, `status`, `lastSeen`, `createdAt`, `updatedAt`)
- Argos Engine applies configuration changes and responds with updated status
- Return validation errors with field-level details on invalid input
- Configuration changes propagated to engine within 30 seconds

**Configuration Flow**:
```
Frontend --> BFF (validate with Zod) --> Save to Document DB --> POST /config/cameras (webhook) --> Engine applies changes
```

**Technical Details**:
- Camera document extends engine CameraConfig with BFF-only fields (`description`, `credentials`, `status`, `lastSeen`, `createdAt`, `updatedAt`)
- Webhook payload excludes BFF-only fields (same projection pattern used for frontend responses)
- Publish is fire-and-forget after successful DB save (async, non-blocking)
- Store `updatedAt` timestamp with each camera document
- Use Zod schemas to validate nested structure

**Example**:
```typescript
// PUT /api/cameras/cam-001
{
  "name": "Main Entrance",
  "description": "Front door camera covering the lobby",
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
    "system_prompt": "Monitor for unauthorized persons entering the building after hours."
  },
  "decision": {
    "model": "openai:gpt-4o",
    "kwargs": {},
    "system_prompt": "Alert if any person is detected between 22:00 and 06:00.",
    "json_schema": {}
  },
  "credentials": {
    "username": "admin",
    "password": "secret"
  }
}

// Response: 200 OK
{
  "data": {
    "id": "cam-001",
    "name": "Main Entrance",
    "description": "Front door camera covering the lobby",
    "enabled": true,
    "connection": {
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
      "system_prompt": "Monitor for unauthorized persons entering the building after hours."
    },
    "decision": {
      "model": "openai:gpt-4o",
      "kwargs": {},
      "system_prompt": "Alert if any person is detected between 22:00 and 06:00.",
      "json_schema": {}
    },
    "status": "online",
    "lastSeen": "2026-02-13T10:40:00Z",
    "updatedAt": "2026-02-13T10:40:00Z"
  },
  "configPublished": true
}
```

---

### FR-003: Event Processing & Storage

**Priority**: P0 (Critical)

**Description**:
Subscribe to Kafka `argos.events` topic, receive events from Argos Engine, validate event schema, persist to database, and fan out to connected frontend clients via SSE.

**Acceptance Criteria**:
- Subscribe to Kafka `argos.events` topic on startup
- Validate incoming events against `DetectedEvent` Zod schema
- Persist valid events to document database `events` collection
- Fan out events to authorized frontend clients via SSE
- Filter events per user based on authorized camera IDs (v2, but structure must support)
- Handle malformed events gracefully (log error, do not crash)
- Implement at-least-once delivery semantics via Kafka consumer offset management

**Technical Details**:
- Use Apache Kafka with `@kafkajs`
- Index events by `cameraId`, `timestamp`, `type`
- Implement event retention policy (30 days in v1)

**Event Flow**:
```
Argos Engine --> Kafka (argos.events) --> BFF KafkaConsumer --> Event Processor
   |
   v
   Document DB (events collection)
   |
   v
   SSE Manager --> Fan out to Frontend Clients
```

Disclaimer: the examples are assumptions, SLA TBD
**Example Event**:
```json
{
  "id": "evt-001",
  "cameraId": "cam-001",
  "timestamp": "2026-02-13T10:35:12Z",
  "type": "anomaly",
  "severity": "high",
  "description": "Unauthorized person detected in restricted area",
  "confidence": 0.92,
  "notified": false,
  "metadata": {
    "zone": "server_room",
    "detectionBox": { "x": 120, "y": 45, "width": 80, "height": 150 }
  }
}
```

---

### FR-004: SSE Streaming (Real-Time Events)

**Priority**: P0 (Critical)

**Description**:
Provide Server-Sent Events (SSE) endpoint for real-time event notifications to frontend. Manage client connections, implement heartbeats, and handle reconnections gracefully.

**Acceptance Criteria**:
- `GET /api/events/stream` - SSE endpoint (requires authentication)
- Send `event: message` with event payload when new event detected
- Send `event: heartbeat` every 30 seconds to keep connection alive
- Support multiple concurrent connections (different users/tabs)
- Filter events per connection based on user's authorized cameras (v2)
- Close idle connections after 5 minutes of no client activity
- Support graceful shutdown (drain connections before exit)
- Multi-instance support: Each BFF instance uses its own Kafka consumer group to receive all events

**Technical Details**:
- Use Fastify `reply.raw` for SSE responses
- Maintain in-memory map of active connections: `Map<userId, Set<Response>>`
- Each BFF instance in its own Kafka consumer group receives all events and fans out to its local SSE connections
- Set HTTP headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`
- Use newline-separated format per SSE spec

**Example SSE Stream**:
```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: message
data: {"id":"evt-001","cameraId":"cam-001","timestamp":"2026-02-13T10:35:12Z","type":"anomaly","severity":"high","description":"Unauthorized access","confidence":0.92,"notified":false}

event: heartbeat
data: {"timestamp":"2026-02-13T10:36:00Z"}

event: message
data: {"id":"evt-002","cameraId":"cam-003","timestamp":"2026-02-13T10:36:45Z","type":"normal","severity":"low","description":"Employee entering","confidence":0.88,"notified":false}
```

---

### FR-005: Camera Management

**Priority**: P0 (Critical)

**Description**:
Provide full CRUD operations for cameras. Each camera document uses the engine's nested `CameraConfig` structure extended with BFF-only fields (`description`, `credentials`, `status`, `lastSeen`, `createdAt`, `updatedAt`). The `connection.stream_url` and `credentials` are never exposed to the frontend.

**Acceptance Criteria**:

**Create Camera**:
- `POST /api/cameras` - Create a new camera
- Accept: name, description, enabled, connection (stream_url, protocol, frame_interval), frame (target_width, target_height, jpeg_quality), observation (model, kwargs, system_prompt), decision (model, kwargs, system_prompt, output_schema), credentials (optional)
- Validate input with Zod schema
- Persist to document database
- Send configuration change to Engine via webhook (`POST /config/cameras`)
- Return created camera (without `connection.stream_url` / `credentials`)

**List Cameras**:
- `GET /api/cameras` - Return all cameras with status
- Include: all fields except `connection.stream_url` and `credentials`
- Cache camera list for 30 seconds (SWR on frontend will poll)

**Get Camera Details**:
- `GET /api/cameras/:id` - Return single camera details
- Exclude: `connection.stream_url`, `credentials` (never sent to frontend)
- Return `404 Not Found` if camera doesn't exist

**Update Camera**:
- `PUT /api/cameras/:id` - Update camera details
- Accept any combination of fields (partial updates of nested objects supported)
- Validate input with Zod schema
- Persist to document database
- Send configuration change to Engine via webhook (`POST /config/cameras`)
- Return updated camera (without `connection.stream_url` / `credentials`)

**Delete Camera**:
- `DELETE /api/cameras/:id` - Delete a camera
- Remove from document database
- Send configuration change to Engine via webhook (`POST /config/cameras`)
- Return `404 Not Found` if camera doesn't exist

**Camera Status**:
- Fetch camera status from Argos Engine (online/offline)
- Update `lastSeen` timestamp when engine sends heartbeat
- Mark camera as offline if no heartbeat for 60 seconds

Disclaimer: the examples are assumptions, SLA TBD
**Camera Document Model**:
```typescript
{
  id: string                     // Unique identifier
  name: string                   // Display name
  description: string            // Human-readable description (BFF-only)
  enabled: boolean               // Whether the camera is active
  connection: {
    stream_url: string           // RTSP URL or local device index (NEVER exposed to frontend)
    protocol: "rtsp" | "local"   // Connection protocol
    frame_interval: number       // Seconds between frame captures
  }
  frame: {
    target_width: number         // Frame resize width (pixels)
    target_height: number        // Frame resize height (pixels)
    jpeg_quality: number         // JPEG encode quality (1-100)
  }
  observation: {
    model: string                // VLM model in provider:model format
    kwargs: object               // Extra kwargs for model init
    system_prompt: string        // Instruction sent to VLM alongside each frame
  }
  decision: {
    model: string                // LLM model in provider:model format
    kwargs: object               // Extra kwargs for model init
    system_prompt: string        // Instruction sent to LLM alongside stories
    json_schema: object        // JSON Schema for structured output
  }
  credentials?: {                // Optional auth credentials (BFF-only, NEVER exposed to frontend)
    username: string
    password: string
  }
  status: "online" | "offline"   // Current connection status (BFF-only)
  lastSeen: string               // ISO timestamp of last engine heartbeat (BFF-only)
  createdAt: string              // ISO timestamp (BFF-only)
  updatedAt: string              // ISO timestamp (BFF-only)
}
```

**Technical Details**:
- Store cameras in document database `cameras` collection
- Implement health check mechanism with engine
- Cache camera data in Redis with 30-second TTL
- Exclude `connection.stream_url` and `credentials` fields from all responses to frontend
- Exclude BFF-only fields (`description`, `credentials`, `status`, `lastSeen`, `createdAt`, `updatedAt`) from engine webhook payloads and `GET /api/cameras/engine` responses

Disclaimer: the examples are assumptions, SLA TBD
**Example**:
```typescript
// GET /api/cameras
{
  "cameras": [
    {
      "id": "cam-001",
      "name": "Main Entrance",
      "description": "Front door camera covering the lobby",
      "enabled": true,
      "connection": {
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
        "system_prompt": "Monitor for unauthorized persons entering the building after hours."
      },
      "decision": {
        "model": "openai:gpt-4o",
        "kwargs": {},
        "system_prompt": "Alert if any person is detected between 22:00 and 06:00.",
        "json_schema": {}
      },
      "status": "online",
      "lastSeen": "2026-02-13T10:30:00Z"
    }
  ]
}
```

---

### FR-006: Stream Proxying

**Priority**: P0 (Critical)

**Description**:
Proxy video streams from cameras to the frontend. The BFF acts as the sole intermediary: the frontend never has direct access to camera URLs or credentials. The frontend calls `/api/cameras/:id/stream`, and the BFF connects to the camera using the stored url/credentials, or receives the video stream from the engine via gRPC, then streams the video to the frontend.

**Acceptance Criteria**:
- `GET /api/cameras/:id/stream` - Proxy video stream to frontend
- Verify bearer token before serving stream
- BFF looks up camera url and credentials from the database
- BFF connects to the camera directly or receives the stream from the engine via gRPC
- Stream video data to the frontend client
- Frontend never sees or touches camera URLs or credentials
- Return `404 Not Found` if camera doesn't exist
- Return `503 Service Unavailable` if camera is offline

**Technical Details**:
- BFF retrieves camera connection details (url, credentials) from document database
- Option A: BFF connects directly to the camera stream and proxies to frontend
- Option B: BFF receives video from engine via gRPC and proxies to frontend
- Both options may be used depending on deployment scenario
- gRPC communication with engine uses `@grpc/grpc-js`

**Example**:
```typescript
// GET /api/cameras/cam-001/stream
// Response: proxied video stream (binary data)
// Content-Type depends on stream format (e.g., multipart/x-mixed-replace, video/mp4, application/x-mpegURL)
```

---

### FR-007: Health & Readiness Endpoints

**Priority**: P1 (High)

**Description**:
Provide health check endpoints for monitoring and load balancers.

**Acceptance Criteria**:
- `GET /health` - Basic health check (always returns 200 if service is up)
- `GET /health/ready` - Readiness check (checks DB, Redis, Engine connectivity)
- Return 200 OK if all dependencies healthy
- Return 503 Service Unavailable if any dependency unreachable

**Example**:
```typescript
// GET /health/ready
{
  "status": "healthy",
  "timestamp": "2026-02-13T10:40:00Z",
  "checks": {
    "database": "ok",
    "redis": "ok",
    "messageBroker": "ok",
    "engine": "ok"
  }
}
```

---

### FR-008: Engine Internal Endpoint

**Priority**: P0 (Critical)

**Description**:
Expose `GET /api/cameras/engine` as an internal endpoint (no authentication) that returns all cameras in engine `CameraConfig` format. Called by the Engine on startup to load its initial camera configuration (pull-on-start pattern, see [ADR-003](../adr/003-config-propagation-webhook.md)).

**Acceptance Criteria**:
- `GET /api/cameras/engine` returns all cameras in engine `CameraConfig` format
- No authentication required (internal network communication)
- Response includes `connection.stream_url` (needed by engine)
- Response excludes BFF-only fields: `description`, `credentials`, `status`, `lastSeen`, `createdAt`, `updatedAt`
- Returns `200 OK` with `{ "cameras": [CameraConfig, ...] }`
- See [`docs/engine/API_CONTRACT.md`](../engine/API_CONTRACT.md) for the canonical `CameraConfig` schema

**Technical Details**:
- Registered on the InternalController
- BFF excludes BFF-only fields from the response (same projection pattern used elsewhere)
- Engine falls back to `ARGOS_CAMERAS_CONFIG` env var if this endpoint is unreachable

---

### FR-009: Testing Strategy

**Priority**: P1 (High)

**Description**:
Comprehensive testing strategy covering unit, integration, contract, and E2E tests.

**Acceptance Criteria**:

**Unit Tests** (Vitest):
- Services (CameraService, EventService, ConfigService, AuthService)
- Zod schemas (validation for all request/response types)
- Auth provider (StaticTokenValidator)
- Domain errors
- Target: 100% coverage on business logic (services)

**Integration Tests** (Vitest + Testcontainers):
- All API endpoints including `GET /api/cameras/engine`
- Kafka consumer event processing
- SSE broadcast
- Webhook sending (mocked engine)
- Target: all endpoints covered

**Contract Tests**:
- Validate `GET /api/cameras/engine` output conforms to engine `CameraConfig` schema
- Validate webhook payloads (`POST /config/cameras`) conform to engine `CameraConfig` schema
- Use engine API contract as the source of truth

**E2E Tests**:
- Camera CRUD → webhook delivery
- Kafka event → SSE delivery
- Auth flows (valid/invalid tokens)

**Coverage Target**: >80% overall, 100% for services

---

## Non-Functional Requirements

### NFR-001: Performance (all this section is TBD, this is just an estimate)

**Requirement**: The BFF must handle high-throughput event processing and low-latency SSE delivery.

**Acceptance Criteria**:
- API endpoint response time: p99 < 200ms (excluding stream proxying)
- SSE event latency: < 100ms from engine event to frontend client
- Support 100 concurrent SSE connections per BFF instance
- Process 1000 events/second from message broker
- Database query optimization: All queries < 50ms

**Implementation**:
- Use connection pooling for document database
- Index all frequently queried fields
- Kafka consumer processes events directly (no intermediate queue)
- Implement Redis caching for camera list

---

### NFR-002: Scalability

**Requirement**: The BFF must scale horizontally to handle increasing load.

**Acceptance Criteria**:
- Stateless design: No in-memory session storage (except SSE connections)
- Support multiple BFF instances behind load balancer
- Use Kafka consumer groups for cross-instance event distribution (each instance in its own consumer group receives all events)
- Database connection pooling to prevent connection exhaustion
- No single point of failure

**Implementation**:
- Deploy multiple BFF instances (Docker containers)
- Use Kafka for event distribution across instances
- Use document database for persistent storage
- Implement graceful shutdown (drain connections)

---

### NFR-003: Reliability

**Requirement**: The BFF must be highly available and resilient to failures.

**Acceptance Criteria**:
- Target uptime: 99.9% (8.76 hours downtime/year)
- Graceful degradation: If engine is down, BFF continues serving cached data
- Automatic retry via Kafka consumer offset management (don't commit until processing succeeds)
- Database connection retry on transient failures
- SSE auto-reconnect support (client implements exponential backoff)

**Implementation**:
- Kafka consumer provides at-least-once delivery via offset management
- Implement circuit breaker for engine API calls
- Log all errors with correlation IDs
- Monitor error rates with alerts

---

### NFR-004: Security

**Requirement**: The BFF must enforce authentication, authorization, and prevent common vulnerabilities.

**Acceptance Criteria**:
- All API endpoints protected with static bearer token validation
- NoSQL injection prevention (input validation and sanitization)
- Rate limiting: 100 requests/minute per user
- CORS configuration: Only allow frontend domain
- Input validation: All inputs validated with Zod schemas
- Secure headers: Helmet middleware (CSP, X-Frame-Options, etc.)
- Secrets management: Environment variables, never hardcoded
- Camera URLs and credentials never exposed to frontend

**Implementation**:
- Use `@fastify/helmet` for security headers
- Use `@fastify/rate-limit` for rate limiting
- Use `@fastify/cors` with strict origin whitelist
- Validate all request bodies with Zod schemas
- Sanitize all inputs before database operations

---

### NFR-005: Observability

**Requirement**: The BFF must provide comprehensive logging, tracing, and metrics for monitoring.

**Acceptance Criteria**:
- Structured JSON logging (Pino)
- Log levels: DEBUG, INFO, WARN, ERROR
- Correlation IDs: Propagate request ID through all layers
- Distributed tracing: OpenTelemetry instrumentation
- Metrics export: Request latency, error rates, event throughput
- Log sensitive data redaction (no tokens, passwords in logs)

**Implementation**:
- Use Pino for logging (Fastify default)
- Use OpenTelemetry for traces and metrics
- Export metrics to Prometheus-compatible endpoint
- Implement request ID middleware (X-Request-ID header)

**Example Log Entry**:
```json
{
  "level": "info",
  "time": 1612345678000,
  "msg": "Event processed successfully",
  "requestId": "req-123",
  "userId": "user-456",
  "eventId": "evt-001",
  "duration": 45
}
```

---

### NFR-006: Maintainability

**Requirement**: The codebase must be easy to understand, extend, and maintain by outsourced teams.

**Acceptance Criteria**:
- Hexagonal architecture with clear layer separation
- TypeScript strict mode enabled
- 100% test coverage for business logic (services, repositories)
- Comprehensive inline documentation (JSDoc)
- Consistent code style (ESLint + Prettier)
- OpenAPI 3.1 spec auto-generated from Zod schemas

**Implementation**:
- Follow Hexagonal Architecture pattern
- Use Dependency Injection for testability
- Write unit tests for all services
- Write integration tests for API endpoints
- Generate API docs with @fastify/swagger

---

## API Requirements

The BFF must implement the **exact API contract** defined in:
`apps/web/docs/BFF_API_CONTRACT.md`

### Key Endpoints Summary

| Method | Endpoint | Description | Priority |
|--------|----------|-------------|----------|
| GET | /api/cameras | List all cameras | P0 |
| GET | /api/cameras/:id | Get camera details | P0 |
| POST | /api/cameras | Create a new camera | P0 |
| PUT | /api/cameras/:id | Update a camera | P0 |
| DELETE | /api/cameras/:id | Delete a camera | P0 |
| GET | /api/cameras/:id/stream | Proxy video stream | P0 |
| GET | /api/cameras/engine | Internal: engine startup config pull (no auth) | P0 |
| GET | /api/events/stream | SSE event stream | P0 |
| GET | /api/events/history | Event history | P1 |
| GET | /health | Health check | P1 |
| GET | /health/ready | Readiness check | P1 |

### Response Format Standards

**Success Response**:
```json
{
  "data": { /* response payload */ },
  "timestamp": "2026-02-13T10:40:00Z"
}
```

**Error Response**:
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": { /* field-level validation errors */ },
    "timestamp": "2026-02-13T10:40:00Z"
  }
}
```

**Rate Limit Headers**:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1612345678
```

---

## Success Metrics

The BFF will be considered successful when:

### Technical Metrics
- API endpoint p99 latency < 200ms
- SSE event delivery latency < 100ms
- Event processing throughput > 1000 events/sec
- 99.9% uptime over 30 days
- Zero NoSQL injection vulnerabilities
- Zero authentication bypass vulnerabilities
- Test coverage > 80%

### User Experience Metrics
- Frontend can fetch camera list in < 500ms
- Configuration updates reflected in engine within 30 seconds
- SSE reconnects successfully after network interruption
- Error messages are actionable and clear

### Developer Experience Metrics
- Outsourcing team starts coding within 1 day of handoff
- API documentation is comprehensive and accurate
- Zero ambiguity in requirements
- New developer onboarded in < 4 hours

---

## Out of Scope (v1)

The following features are **NOT** required for v1 but should be considered for v2:

### Future Features (v2)
- User management API (defer to future identity provider)
- WebSocket support (SSE is sufficient for v1)
- Multi-tenancy (single organization in v1)
- Fine-grained camera permissions per user (all users see all cameras in v1)
- Advanced analytics (event aggregation, trends)
- Email/Slack/SMS notification integration (log only in v1)
- Configuration versioning and rollback
- Audit logging (who changed what when)
- GraphQL API (REST + SSE only in v1)
- OAuth/OIDC authentication (static token in v1, `IAuthProvider` interface supports future migration)

---

## Dependencies

### External Systems

**Argos Engine** (Video Processing):
- Provides camera status and video streams (via gRPC)
- Publishes detected events to Kafka (`argos.events`)
- Receives configuration changes via HTTP webhook (`POST /config/cameras`)
- Pulls initial camera config from BFF on startup (`GET /api/cameras/engine`)
- **BFF Dependency**: gRPC client (`@grpc/grpc-js`), Kafka consumer (`argos.events`), HTTP client for Engine webhook
- **Contract**: [`docs/engine/API_CONTRACT.md`](../engine/API_CONTRACT.md) — canonical CameraConfig schema and interface spec

**Document Database (NoSQL)** (Database):
- Stores camera documents (with embedded prompts)
- Stores detected events and metadata
- Stores notification records
- **BFF Dependency**: Native database driver

**Apache Kafka** (Message Broker):
- Event streaming for event distribution (`argos.events` topic only)
- Persistent message log with replay capability
- **BFF Dependency**: `@kafkajs`

**Redis** (Optional Cache):
- Cache for camera list and configs (optional, not required for core functionality)
- **BFF Dependency**: Redis client (ioredis), if caching is enabled

**Frontend** (Consumer):
- Alquimia Argos Next.js application
- Consumes BFF REST + SSE API
- **BFF Dependency**: API contract adherence

---

## Assumptions & Constraints

### Assumptions
1. A static bearer token is configured via the `AUTH_BEARER_TOKEN` environment variable
2. Argos Engine publishes events to Kafka `argos.events` topic
3. Argos Engine exposes `POST /config/cameras` endpoint for receiving configuration changes via HTTP webhook
4. Frontend developers have access to OpenAPI spec for type generation
5. Document database is provisioned and accessible before BFF deployment
6. Kafka and document database are highly available (managed services or clustered)
7. Camera count in v1: < 50 cameras
8. Concurrent users in v1: < 100 users
9. Event rate in v1: < 1000 events/minute

### Constraints
1. **Technology**: Must use TypeScript/Node.js (team skillset)
2. **Budget**: No premium services (use open-source tools)
3. **Security**: Must validate bearer token on all protected endpoints
4. **Deployment**: Must run in Docker containers
5. **Database**: Document database (NoSQL)
6. **Video Communication**: gRPC for engine video streaming

---

## Acceptance Criteria

The BFF is ready for production deployment when:

- All P0 functional requirements implemented
- All P0 + P1 NFRs validated
- OpenAPI 3.1 spec available
- All API endpoints match frontend contract exactly
- Unit tests pass with > 80% coverage
- Integration tests pass
- E2E tests pass (with frontend)
- Load test: 100 concurrent SSE connections + 1000 events/sec
- Security audit: No critical vulnerabilities
- Deployment documentation complete
- Operator runbooks created

---

## Appendix

### Related Documents
- [Technical Blueprint](./TECHNICAL_BLUEPRINT.md) - Architecture and design patterns
- [API Specification](./API_SPEC.md) - OpenAPI 3.1 spec
- [Database Schema](./DATABASE_SCHEMA.md) - Document database schema design
- [Engine Integration](./ENGINE_INTEGRATION.md) - How to integrate with Argos Engine
- [Engine API Contract](../engine/API_CONTRACT.md) - Canonical BFF↔Engine interface spec (CameraConfig schema, startup endpoint, webhook)
- [Setup Guide](./SETUP_GUIDE.md) - Developer environment setup
- [Frontend API Contract](../../web/docs/BFF_API_CONTRACT.md) - Frontend expectations
- [ADR-003: Config Propagation](../adr/003-config-propagation-webhook.md) - Pull-on-start, push-on-change pattern

### Glossary
- **BFF**: Backend-For-Frontend - API gateway tailored to frontend needs
- **SSE**: Server-Sent Events - HTTP-based server push protocol
- **HLS**: HTTP Live Streaming - Video streaming protocol
- **gRPC**: High-performance RPC framework for video stream communication
- **JWT**: JSON Web Token - Stateless authentication token
- **Fastify**: High-performance web framework for Node.js
- **Zod**: TypeScript schema validation library
- **Kafka**: Distributed event streaming platform for high-throughput messaging
- **@kafkajs**: Confluent's official Kafka client for Node.js
- **IAuthProvider**: Authentication interface that abstracts token validation strategy

---

**Document Version**: 1.3
**Last Updated**: February 2026
**Next Review**: After v1 deployment
