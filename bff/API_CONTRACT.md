# BFF API Contract
## Backend-For-Frontend API Specification

**Version**: 3.0
**Date**: February 2026
**Protocol**: REST + Server-Sent Events (SSE)

> **Engine Interface**: The canonical BFF↔Engine interface contract (CameraConfig schema, startup endpoint, webhook spec) is defined in [`docs/engine/API_CONTRACT.md`](../engine/API_CONTRACT.md). This document covers the BFF's public API (frontend-facing) and internal API (engine-facing).

---

## Table of Contents
1. [Base Configuration](#base-configuration)
2. [Authentication](#authentication)
3. [Camera Endpoints](#camera-endpoints)
4. [Internal Endpoints (Engine-Facing)](#internal-endpoints-engine-facing)
5. [Webhook Outbound (BFF → Engine)](#webhook-outbound-bff--engine)
6. [Event Endpoints](#event-endpoints)
7. [Error Responses](#error-responses)
8. [TypeScript Interfaces](#typescript-interfaces)
9. [Request/Response Examples](#requestresponse-examples)

---

## Base Configuration

### Base URL
```
Production:  https://api.alquimia.com
Development: http://localhost:8000
```

### API Version
All endpoints are prefixed with `/api`

### Content Type
```
Content-Type: application/json
Accept: application/json
```

### CORS
- **Allowed Origins**: Frontend domain (configured in BFF)
- **Credentials**: Included (`withCredentials: true`)
- **Methods**: GET, POST, PUT, DELETE, OPTIONS

---

## Authentication

### Authentication Method
**Static Bearer Token** - Token is configured via the `AUTH_TOKEN` environment variable on the BFF. The frontend must include this token in all requests.

### Authorization Header
All authenticated requests must include:
```
Authorization: Bearer <token>
```

### Token Validation
- Token is validated by authentication middleware on every request
- Compared using constant-time string comparison
- No token refresh logic required; the token is static

### Response on Invalid Token
```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid or missing authentication token",
    "timestamp": "2026-02-13T10:30:00Z"
  }
}
```

---

## Camera Endpoints

The BFF camera model extends the engine's `CameraConfig` schema (see [`docs/engine/API_CONTRACT.md`](../engine/API_CONTRACT.md)) with BFF-only fields (`description`, `credentials`, `status`, `lastSeen`, `createdAt`, `updatedAt`). The nested structure (`connection`, `frame`, `observation`, `decision`) matches the engine schema exactly.

### POST /api/cameras
Create a new camera configuration.

**Authentication**: Required

**Request**:
```
POST /api/cameras
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Main Entrance",
  "description": "Front door camera",
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
    "system_prompt": "Monitor for people entering and exiting the building"
  },
  "decision": {
    "model": "openai:gpt-4o",
    "kwargs": {},
    "system_prompt": "Classify as anomaly if unauthorized person detected outside business hours",
    "json_schema": {}
  },
  "credentials": {
    "username": "admin",
    "password": "secret"
  }
}
```

**Response**: `201 Created`
```json
{
  "camera": {
    "id": "cam-001",
    "name": "Main Entrance",
    "description": "Front door camera",
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
      "system_prompt": "Monitor for people entering and exiting the building"
    },
    "decision": {
      "model": "openai:gpt-4o",
      "kwargs": {},
      "system_prompt": "Classify as anomaly if unauthorized person detected outside business hours",
      "json_schema": {}
    },
    "status": "offline",
    "lastSeen": null,
    "createdAt": "2026-02-13T10:30:00Z",
    "updatedAt": "2026-02-13T10:30:00Z"
  },
  "message": "Camera created and configuration sent to engine"
}
```

**Note**: `connection.stream_url` and `credentials` are accepted in the request but are **never** included in API responses. The video stream is accessed via `/api/cameras/:id/stream`.

**Error Responses**:
- `400 Bad Request`: Invalid request body
- `401 Unauthorized`: Invalid or missing token
- `422 Unprocessable Entity`: Validation errors
- `500 Internal Server Error`: Server error

---

### GET /api/cameras
Retrieve list of all configured cameras.

**Authentication**: Required

**Request**:
```
GET /api/cameras
Authorization: Bearer <token>
```

**Response**: `200 OK`
```json
{
  "cameras": [
    {
      "id": "cam-001",
      "name": "Main Entrance",
      "description": "Front door camera",
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
        "system_prompt": "Monitor for people entering and exiting the building"
      },
      "decision": {
        "model": "openai:gpt-4o",
        "kwargs": {},
        "system_prompt": "Classify as anomaly if unauthorized person detected outside business hours",
        "json_schema": {}
      },
      "status": "online",
      "lastSeen": "2026-02-13T10:30:00Z",
      "createdAt": "2026-02-01T00:00:00Z",
      "updatedAt": "2026-02-13T10:30:00Z"
    },
    {
      "id": "cam-002",
      "name": "Parking Lot",
      "description": "South parking camera",
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
        "system_prompt": "Monitor for vehicles and persons in parking area"
      },
      "decision": {
        "model": "openai:gpt-4o",
        "kwargs": {},
        "system_prompt": "Classify as anomaly if vehicle enters after hours",
        "json_schema": {}
      },
      "status": "offline",
      "lastSeen": "2026-02-13T08:15:00Z",
      "createdAt": "2026-02-01T00:00:00Z",
      "updatedAt": "2026-02-13T08:15:00Z"
    }
  ]
}
```

**Note**: `connection.stream_url` and `credentials` are stored in the database but are **never** included in API responses. Use the `/api/cameras/:id/stream` endpoint to access the video stream.

**Error Responses**:
- `401 Unauthorized`: Invalid or missing token
- `500 Internal Server Error`: Server error

---

### GET /api/cameras/:id
Retrieve details for a specific camera.

**Authentication**: Required

**Path Parameters**:
- `id` (string): Camera ID

**Request**:
```
GET /api/cameras/cam-001
Authorization: Bearer <token>
```

**Response**: `200 OK`
```json
{
  "id": "cam-001",
  "name": "Main Entrance",
  "description": "Front door camera",
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
    "system_prompt": "Monitor for people entering and exiting the building"
  },
  "decision": {
    "model": "openai:gpt-4o",
    "kwargs": {},
    "system_prompt": "Classify as anomaly if unauthorized person detected outside business hours",
    "json_schema": {}
  },
  "status": "online",
  "lastSeen": "2026-02-13T10:30:00Z",
  "createdAt": "2026-02-01T00:00:00Z",
  "updatedAt": "2026-02-13T10:30:00Z"
}
```

**Note**: `connection.stream_url` and `credentials` are stored in the database but are **never** included in API responses.

**Error Responses**:
- `401 Unauthorized`: Invalid or missing token
- `404 Not Found`: Camera does not exist

---

### PUT /api/cameras/:id
Update an existing camera configuration. All fields are optional; only provided fields are updated.

**Authentication**: Required

**Path Parameters**:
- `id` (string): Camera ID

**Request**:
```
PUT /api/cameras/cam-001
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Main Entrance (Updated)",
  "observation": {
    "system_prompt": "Monitor for people and vehicles entering the building"
  }
}
```

**Response**: `200 OK`
```json
{
  "id": "cam-001",
  "name": "Main Entrance (Updated)",
  "description": "Front door camera",
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
    "system_prompt": "Monitor for people and vehicles entering the building"
  },
  "decision": {
    "model": "openai:gpt-4o",
    "kwargs": {},
    "system_prompt": "Classify as anomaly if unauthorized person detected outside business hours",
    "json_schema": {}
  },
  "status": "online",
  "lastSeen": "2026-02-13T10:30:00Z",
  "createdAt": "2026-02-01T00:00:00Z",
  "updatedAt": "2026-02-13T11:00:00Z"
}
```

**Error Responses**:
- `400 Bad Request`: Invalid request body
- `401 Unauthorized`: Invalid or missing token
- `404 Not Found`: Camera does not exist
- `422 Unprocessable Entity`: Validation errors
- `500 Internal Server Error`: Server error

---

### DELETE /api/cameras/:id
Remove a camera configuration.

**Authentication**: Required

**Path Parameters**:
- `id` (string): Camera ID

**Request**:
```
DELETE /api/cameras/cam-001
Authorization: Bearer <token>
```

**Response**: `204 No Content`

No response body.

**Error Responses**:
- `401 Unauthorized`: Invalid or missing token
- `404 Not Found`: Camera does not exist
- `500 Internal Server Error`: Server error

---

### GET /api/cameras/:id/stream
Proxy the video stream for a specific camera. The BFF reads the camera's stored URL from the database and streams the video content directly to the client. The frontend never sees the underlying camera URL.

**Authentication**: Required

**Path Parameters**:
- `id` (string): Camera ID

**Request**:
```
GET /api/cameras/cam-001/stream
Authorization: Bearer <token>
```

**Response**: `200 OK`

The response is the proxied video stream. Content-Type varies depending on the source format (e.g., `video/mp4`, `application/x-mpegURL`, `multipart/x-mixed-replace`).

```
Content-Type: <varies by stream format>
```

**Error Responses**:
- `401 Unauthorized`: Invalid or missing token
- `404 Not Found`: Camera does not exist
- `503 Service Unavailable`: Camera offline or stream unavailable

---

## Internal Endpoints (Engine-Facing)

These endpoints are **not** part of the public BFF API exposed to the frontend. They are used for internal communication between the BFF and the Engine on the same network.

### GET /api/cameras/engine
Return all cameras in engine `CameraConfig` format for the Engine to load on startup. See [`docs/engine/API_CONTRACT.md`](../engine/API_CONTRACT.md) for the full schema definition.

**Authentication**: None (internal network communication)

**Request**:
```
GET /api/cameras/engine
```

**Response**: `200 OK`
```json
{
  "cameras": [
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
        "system_prompt": "Monitor for people entering and exiting the building"
      },
      "decision": {
        "model": "openai:gpt-4o",
        "kwargs": {},
        "system_prompt": "Classify as anomaly if unauthorized person detected outside business hours",
        "json_schema": {}
      }
    }
  ]
}
```

**Fields included**: All engine `CameraConfig` fields (including `connection.stream_url`)

**Fields excluded** (BFF-only): `description`, `credentials`, `status`, `lastSeen`, `createdAt`, `updatedAt`

**Failure behavior**: If this endpoint is unreachable, the Engine falls back to the `ARGOS_CAMERAS_CONFIG` environment variable.

---

## Webhook Outbound (BFF → Engine)

When a camera is created, updated, or deleted, the BFF sends an HTTP POST to the Engine's webhook endpoint (`POST /config/cameras`) so the Engine can apply the change in real-time. The Engine URL is configured via the `ENGINE_URL` environment variable on the BFF.

### Webhook Payload Format

```json
{
  "type": "camera:created | camera:updated | camera:deleted",
  "cameraId": "cam-001",
  "config": { /* Engine CameraConfig — omitted for camera:deleted */ },
  "timestamp": "2026-02-25T10:00:00Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `"camera:created" \| "camera:updated" \| "camera:deleted"` | Yes | Change type |
| `cameraId` | `string` | Yes | Camera identifier |
| `config` | `CameraConfig` | Only for `created` / `updated` | Engine CameraConfig (excludes BFF-only fields) |
| `timestamp` | `ISO 8601 datetime` | Yes | When the change occurred |

The `config` object uses the engine `CameraConfig` format — it includes `connection.stream_url` but excludes BFF-only fields (`description`, `credentials`, `status`, `lastSeen`, `createdAt`, `updatedAt`).

### Event Types

| Event | Description |
|-------|-------------|
| `camera:created` | New camera added — Engine registers camera, allocates buffers, starts capture |
| `camera:updated` | Camera config changed — Engine restarts capture, rebuilds processors |
| `camera:deleted` | Camera removed — Engine stops capture, releases resources |

### Failure Handling

- If the Engine is unreachable, the BFF marks the change as **pending**
- On the next Engine startup, the full config is re-fetched via `GET /api/cameras/engine`, reconciling any missed changes
- See [ADR-003](../adr/003-config-propagation-webhook.md) for the pull-on-start, push-on-change pattern

---

## Event Endpoints

### GET /api/events/stream (SSE)
Server-Sent Events endpoint for real-time event notifications.

**Authentication**: Required

**Request**:
```
GET /api/events/stream
Authorization: Bearer <token>
Accept: text/event-stream
```

**Response**: `200 OK` (SSE Stream)
```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: message
data: {"id":"evt-001","cameraId":"cam-001","timestamp":"2026-02-13T10:35:12Z","type":"anomaly","severity":"high","description":"Unauthorized person detected in restricted area","confidence":0.92,"notified":false,"metadata":{"zone":"server_room"}}

event: message
data: {"id":"evt-002","cameraId":"cam-003","timestamp":"2026-02-13T10:35:45Z","type":"normal","severity":"low","description":"Employee entering main entrance","confidence":0.88,"notified":false,"metadata":{"person_id":"emp-1234"}}

event: heartbeat
data: {"timestamp":"2026-02-13T10:36:00Z"}
```

**Event Types**:
- `message`: New event detected
- `heartbeat`: Keep-alive signal (every 30s)

**Connection Management**:
- **Reconnection**: Client should implement exponential backoff (1s, 2s, 4s, 8s, max 30s)
- **Timeout**: Server closes idle connections after 5 minutes
- **Buffering**: BFF may buffer events during brief disconnections (optional)

**Error Responses**:
- `401 Unauthorized`: Invalid or missing token
- `500 Internal Server Error`: Stream initialization failed

---

### GET /api/events/history (Future v2)
Retrieve historical events with pagination and filtering.

**Authentication**: Required

**Query Parameters**:
- `page` (number, default: 1): Page number
- `limit` (number, default: 50, max: 100): Events per page
- `cameraId` (string, optional): Filter by camera
- `type` (string, optional): Filter by type (`normal`, `anomaly`)
- `startDate` (ISO 8601, optional): Start of time range
- `endDate` (ISO 8601, optional): End of time range

**Request**:
```
GET /api/events/history?page=1&limit=50&type=anomaly&cameraId=cam-001
Authorization: Bearer <token>
```

**Response**: `200 OK`
```json
{
  "events": [ /* DetectedEvent[] */ ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 230,
    "totalPages": 5
  }
}
```

---

## Error Responses

All error responses follow this structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "additional context"
    },
    "timestamp": "2026-02-13T10:50:00Z"
  }
}
```

### Error Codes

| HTTP Status | Code | Description |
|-------------|------|-------------|
| 400 | `BAD_REQUEST` | Invalid request format or parameters |
| 401 | `UNAUTHORIZED` | Missing or invalid authentication token |
| 403 | `FORBIDDEN` | Insufficient permissions for this resource |
| 404 | `NOT_FOUND` | Requested resource does not exist |
| 409 | `CONFLICT` | Request conflicts with current state |
| 422 | `UNPROCESSABLE_ENTITY` | Validation failed |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many requests |
| 500 | `INTERNAL_SERVER_ERROR` | Unexpected server error |
| 503 | `SERVICE_UNAVAILABLE` | Service temporarily unavailable |

### Example Error Response

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid camera configuration",
    "details": {
      "name": "Required field"
    },
    "timestamp": "2026-02-13T10:50:00Z"
  }
}
```

---

## TypeScript Interfaces

### Camera Types

```typescript
// src/lib/types/cameras.ts

export type CameraStatus = 'online' | 'offline' | 'error'
export type CameraProtocol = 'rtsp' | 'local'

export interface ConnectionConfig {
  stream_url: string
  protocol: CameraProtocol
  frame_interval: number
}

export interface FrameConfig {
  target_width: number
  target_height: number
  jpeg_quality: number
}

export interface ObservationConfig {
  model: string
  kwargs: Record<string, unknown>
  system_prompt: string
}

export interface DecisionConfig {
  model: string
  kwargs: Record<string, unknown>
  system_prompt: string
  json_schema: Record<string, unknown>
}

export interface CameraCredentials {
  username: string
  password: string
}

// Response type — connection.stream_url and credentials are never included
export interface Camera {
  id: string
  name: string
  description: string
  enabled: boolean
  connection: Omit<ConnectionConfig, 'stream_url'>
  frame: FrameConfig
  observation: ObservationConfig
  decision: DecisionConfig
  status: CameraStatus
  lastSeen: string | null // ISO 8601
  createdAt: string
  updatedAt: string
}

export interface CameraCreateRequest {
  name: string
  description?: string
  enabled?: boolean
  connection: ConnectionConfig
  frame?: Partial<FrameConfig>
  observation: ObservationConfig
  decision: DecisionConfig
  credentials?: CameraCredentials
}

export interface CameraUpdateRequest {
  name?: string
  description?: string
  enabled?: boolean
  connection?: Partial<ConnectionConfig>
  frame?: Partial<FrameConfig>
  observation?: Partial<ObservationConfig>
  decision?: Partial<DecisionConfig>
  credentials?: CameraCredentials
}

export interface CameraListResponse {
  cameras: Camera[]
}
```

### Event Types

```typescript
// src/lib/types/events.ts

export type EventType = 'normal' | 'anomaly'
export type EventSeverity = 'low' | 'medium' | 'high' | 'critical'

export interface DetectedEvent {
  id: string
  cameraId: string
  timestamp: string // ISO 8601
  type: EventType
  severity?: EventSeverity
  description: string
  confidence: number // 0.0 - 1.0
  notified: boolean
  metadata?: {
    zone?: string
    person_id?: string
    object_type?: string
    [key: string]: any
  }
}

export interface EventHistoryResponse {
  events: DetectedEvent[]
  pagination: {
    page: number
    limit: number
    total: number
    totalPages: number
  }
}
```

### API Response Types

```typescript
// src/lib/types/api.ts

export interface ApiError {
  error: {
    code: string
    message: string
    details?: Record<string, any>
    timestamp: string
  }
}

export interface ApiResponse<T> {
  data?: T
  error?: ApiError['error']
}
```

---

## Request/Response Examples

### Complete Event Flow Example

**1. Frontend establishes SSE connection**
```typescript
const eventSource = new EventSource(
  'http://localhost:8000/api/events/stream',
  { withCredentials: true }
)

eventSource.onmessage = (event) => {
  const detectedEvent: DetectedEvent = JSON.parse(event.data)
  console.log('New event:', detectedEvent)
}
```

**2. BFF sends event via SSE**
```
event: message
data: {"id":"evt-123","cameraId":"cam-001","timestamp":"2026-02-13T10:35:12Z","type":"anomaly","severity":"high","description":"Unauthorized access attempt","confidence":0.95,"notified":false}
```

---

### Camera CRUD Example

**1. Create a camera**
```typescript
const response = await fetch('http://localhost:8000/api/cameras', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    name: 'Main Entrance',
    description: 'Front door camera',
    enabled: true,
    connection: {
      stream_url: 'rtsp://192.168.1.100:554/stream1',
      protocol: 'rtsp',
      frame_interval: 1.0
    },
    frame: {
      target_width: 1280,
      target_height: 720,
      jpeg_quality: 85
    },
    observation: {
      model: 'openai:gpt-4o',
      kwargs: {},
      system_prompt: 'Monitor for people entering and exiting...'
    },
    decision: {
      model: 'openai:gpt-4o',
      kwargs: {},
      system_prompt: 'Classify as anomaly if unauthorized person...',
      json_schema: {}
    }
  })
})
const { camera } = await response.json()
```

**2. Update a camera**
```typescript
const response = await fetch('http://localhost:8000/api/cameras/cam-001', {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    name: 'Main Entrance (Updated)',
    observation: {
      system_prompt: 'Monitor for people and vehicles entering the building'
    }
  })
})
const updatedCamera: Camera = await response.json()
```

**3. Delete a camera**
```typescript
await fetch('http://localhost:8000/api/cameras/cam-001', {
  method: 'DELETE',
  headers: {
    'Authorization': `Bearer ${token}`
  }
})
// Response: 204 No Content
```

---

## Rate Limiting

The BFF implements rate limiting to prevent abuse:

- **Anonymous**: 10 requests/minute
- **Authenticated**: 100 requests/minute
- **SSE connections**: 1 connection per user

**Rate Limit Headers** (included in responses):
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1612345678
```

**Rate Limit Exceeded Response**: `429 Too Many Requests`
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please retry after 60 seconds.",
    "details": {
      "retryAfter": 60
    },
    "timestamp": "2026-02-13T10:55:00Z"
  }
}
```

---

## WebSocket Alternative (Future Consideration)

While v2 uses SSE for simplicity, the BFF may support WebSocket in future versions for bidirectional communication:

```
ws://localhost:8000/api/events/ws
```

**Benefits**:
- Bidirectional communication
- Lower overhead for high-frequency events
- Support for event acknowledgments

**Trade-offs**:
- More complex connection management
- Requires fallback for restricted networks

---

## BFF Implementation Notes

### Expected BFF Responsibilities
1. **Proxy video streams** from engine to frontend
2. **Transform engine events** to frontend-friendly format
3. **Validate bearer token authentication**
4. **Manage camera CRUD** and propagate config changes to the Engine via HTTP webhook (`POST /config/cameras`)
5. **Handle connection pooling** for SSE clients
6. **Implement rate limiting** per user
7. **Log API usage** for monitoring

### BFF Technology Stack
- **Framework**: Fastify + TypeScript
- **SSE**: Fastify StreamingResponse
- **Authentication**: Static bearer token validation
- **Validation**: Zod

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 3.0 | Feb 2026 | Align camera model with engine CameraConfig nested schema (`connection`, `frame`, `observation`, `decision` sub-objects). Add `GET /api/cameras/engine` internal endpoint for engine startup pull. Add webhook outbound documentation (BFF → Engine `POST /config/cameras`). Add `enabled` field. Replace flat `url`/`observation_prompt`/`decisions_prompt` with nested structure. Add `credentials` object (BFF-only, never in responses). Update all TypeScript interfaces and request/response examples. Add cross-reference to engine API contract. |
| 2.0 | Feb 2026 | Replace OAuth/Keycloak auth with static bearer token. Add camera CRUD endpoints (POST, PUT, DELETE). Remove configuration endpoints (`/api/config/*`). Update Camera schema with `description`, `observation_prompt`, `decisions_prompt` fields. Remove `location`, `subLocation`, `streamUrl`, `metadata` from Camera. Stream endpoint now proxies video directly. Remove obsolete types (`StreamResponse`, `ObservationConfig`, `DecisionsConfig`, etc.). |
| 1.0 | Feb 2026 | Initial API contract for v1 development |

---

**Document Owner**: Alquimia BFF Team
**Frontend Contact**: Frontend Lead
**Review Cycle**: Quarterly or on breaking changes
