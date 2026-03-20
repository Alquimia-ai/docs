# Argos BFF - Technical Blueprint

**Version**: 2.2
**Date**: February 2026
**Architecture**: Hexagonal Architecture
**Target Audience**: Development Team

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Technology Stack](#technology-stack)
3. [Folder Structure](#folder-structure)
4. [Core Patterns](#core-patterns)
5. [Authentication Flow](#authentication-flow)
6. [Event Processing Pipeline](#event-processing-pipeline)
7. [SSE Implementation](#sse-implementation)
8. [Database Strategy](#database-strategy)
9. [API Design](#api-design)
10. [Message Broker Integration](#message-broker-integration)
11. [gRPC Video Proxy](#grpc-video-proxy)
12. [Performance Optimization](#performance-optimization)
13. [Security Best Practices](#security-best-practices)
14. [Testing Strategy](#testing-strategy)
15. [Deployment](#deployment)

---

## Architecture Overview

### Hexagonal Architecture (Ports & Adapters)

The BFF follows **Hexagonal Architecture** principles to achieve:

- Clear separation of concerns
- Easy testability (mock adapters)
- Flexibility to swap implementations (e.g., swap message broker without touching business logic)

**Core Principle**: Dependencies point inward. Business logic has no knowledge of external systems.

```
┌──────────────────────────────────────────────────────────────┐
│                        Presentation Layer                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ Controllers │  │ Middleware  │  │     DTO     │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                       Application Layer                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  Services   │  │    Ports    │  │  Use Cases  │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                         Domain Layer                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Models    │  │   Schemas   │  │    Errors   │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                     Infrastructure Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Database │  │ Messaging│  │   Auth   │  │   HTTP   │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
│  ┌──────────┐  ┌──────────┐                                  │
│  │   gRPC   │  │   SSE    │                                  │
│  └──────────┘  └──────────┘                                  │
└──────────────────────────────────────────────────────────────┘
```

**Layers**:

1. **Domain Layer** (Pure Business Logic):
   - Domain entities (Camera, Evidence)
   - Zod schemas (validation + types)
   - Domain errors (CameraNotFoundError, ConfigValidationError)
   - No external dependencies

2. **Application Layer** (Orchestration):
   - Services (CameraService, EventService, ConfigService)
   - Ports (interfaces): IEventRepository, IMessageBroker, IConfigPublisher
   - Use cases (coordinate services)

3. **Infrastructure Layer** (External Adapters):
   - Database (document DB, repositories)
   - Message Broker (Apache Kafka for events)
   - HTTP Client (webhook to Engine for config changes)
   - Auth (static token validation)
   - HTTP (Fastify server)
   - SSE (connection manager)
   - gRPC (video stream proxy)

4. **Presentation Layer** (HTTP Interface):
   - Controllers (REST endpoint handlers)
   - Middleware (auth, validation, error handling)
   - DTOs (request/response shapes)

---

## Technology Stack

### Framework: Fastify

**Performance**:

- Uses `fast-json-stringify` for response serialization
- Schema-based routing for performance optimization
- Built for high-throughput applications

**TypeScript Support**: First-class TypeScript integration

- Type inference from schemas
- Type-safe route handlers
- Excellent IDE autocomplete

**Plugin Ecosystem**: Rich and well-maintained

- `@fastify/cors`, `@fastify/helmet`, `@fastify/rate-limit`
- `@fastify/swagger` for OpenAPI generation
- `fastify-type-provider-zod` for Zod integration

**Async/Await Native**: Built for modern Node.js

- Promise-based from the ground up

**Validation**: Built-in schema validation

- JSON Schema validation out of the box
- Integrates seamlessly with Zod via type providers

---

### Schema Validation: Zod

**TypeScript-First**: Best-in-class type inference

```typescript
const CameraSchema = z.object({
  id: z.string(),
  name: z.string(),
  status: z.enum(['online', 'offline', 'error'])
})

type Camera = z.infer<typeof CameraSchema> // Automatic type inference
```

**Runtime + Static Validation**: Single source of truth

- Define schema once
- Get TypeScript types automatically
- Validate at runtime

**Excellent Error Messages**: User-friendly validation errors

```typescript
{
  "cameras[0].name": "Required"
}
```

**Composable**: Easy to build complex schemas

```typescript
const BaseCameraSchema = z.object({ id: z.string(), name: z.string() })
const CameraCreateSchema = BaseCameraSchema.extend({
  url: z.string().url(),
  observation_prompt: z.string(),
  decisions_prompt: z.string()
})
```

**Fastify Integration**: `fastify-type-provider-zod`

- Automatic request validation
- Automatic response validation
- OpenAPI generation from schemas

---

### Database: Document Database (NoSQL) + native driver/ODM

**Why Document Database?**

- **Flexible Schema**: Camera configurations and evidence are naturally document-shaped
- **Nested Data**: Prompts, credentials, and metadata fit well as embedded documents
- **Horizontal Scaling**: Easy to shard by camera ID or date range
- **JSON-Native**: No impedance mismatch between application objects and storage
- **Performance**: Excellent for read-heavy workloads with document-level operations

**Repository Pattern with native driver**:
```typescript
const camera = await db.collection('cameras').findOne({ id: 'cam-001' })
// camera: Camera | null (mapped through domain model)
```

**No ORM overhead**: Direct driver access through repository abstraction provides type safety at the application boundary while avoiding ORM complexity.

---

### Message Broker: Apache Kafka via `@kafkajs`

Kafka is used **exclusively for event streaming** (`argos.events` topic). Configuration changes are propagated to the Engine via HTTP webhook (see [ADR-003](../adr/003-config-propagation-webhook.md)).

**Why Kafka (for events)?**

- **Persistent messages with replay**: Kafka retains messages on disk; consumers can replay from any offset
- **Consumer group offset management**: Replaces BullMQ retry semantics -- don't commit offset until processing succeeds, and the message will be re-delivered on failure
- **High throughput**: Designed for millions of messages per second with low latency
- **Ordered delivery per partition**: Events for a given camera can be routed to the same partition for strict ordering

**Why `@kafkajs`?**

- Confluent's official Node.js client
- Built on `librdkafka` (C library) for maximum performance
- KafkaJS-compatible API surface -- familiar to Node.js developers
- Actively maintained with commercial support available

**Abstraction Strategy**:

```typescript
// application/ports/IMessageBroker.ts
export interface IMessageBroker {
  subscribe(channel: string, handler: (message: any) => Promise<void>): Promise<void>
  disconnect(): Promise<void>
}

// infrastructure/messaging/KafkaConsumer.ts
export class KafkaConsumer implements IMessageBroker { }
```

> **Note on retry**: Event processing retry is handled by Kafka consumer offset management. The consumer does not commit the offset until processing succeeds. If processing fails, the message is re-delivered on the next poll. This replaces the need for a separate job queue (BullMQ).

---

### Authentication: Simple token validation middleware (static bearer token from env)

**Why Static Token?**

- **Simplicity**: No external auth service dependency for v1
- **Low Latency**: No network call for token verification
- **Easy Setup**: Single environment variable configuration
- **Migration Path**: `IAuthProvider` interface preserved for future OAuth/OIDC migration

**Example**:
```typescript
import crypto from 'node:crypto'

export class StaticTokenValidator implements IAuthProvider {
  constructor(private validTokens: string[]) {}

  async validate(token: string): Promise<AuthResult> {
    if (!this.validTokens.includes(token)) {
      throw new UnauthorizedError('Invalid bearer token')
    }
    return { authenticated: true, correlationId: crypto.randomUUID() }
  }
}
```

---

### gRPC: `@grpc/grpc-js` for video stream communication with engine

**Why gRPC?**

- **Efficient Streaming**: HTTP/2 based bidirectional streaming
- **Binary Protocol**: Protobuf serialization for video frame data
- **Low Latency**: Persistent connections, no per-request overhead
- **Type Safety**: Proto definitions generate TypeScript types

---

### Logging: Pino

**Why Pino?**

- **Extremely Fast**: Fastest Node.js logger
- **Structured JSON**: Machine-parsable logs
- **Fastify Default**: Built-in integration
- **Low Overhead**: Minimal performance impact
- **Child Loggers**: Contextual logging

**Example**:
```typescript
logger.info({
  requestId: 'req-123',
  cameraId: 'cam-001',
  eventId: 'evt-001',
  duration: 45
}, 'Event processed successfully')
```

---

## Folder Structure

```
apps/bff/
├── src/
│   ├── domain/                   # Pure business logic (no external deps)
│   │   ├── models/              # Domain entities
│   │   │   ├── Camera.ts
│   │   │   └── Evidence.ts
│   │   ├── errors/              # Domain-specific errors
│   │   │   ├── CameraNotFoundError.ts
│   │   │   ├── ConfigValidationError.ts
│   │   │   └── EvidenceNotFoundError.ts
│   │   ├── schemas/             # Zod schemas (validation + types)
│   │   │   ├── camera.schema.ts
│   │   │   ├── evidence.schema.ts
│   │   │   └── config.schema.ts
│   │   └── types/               # TypeScript types/interfaces
│   │       └── index.ts
│   │
│   ├── application/             # Use cases / orchestration
│   │   ├── services/            # Business logic services
│   │   │   ├── CameraService.ts
│   │   │   ├── EventService.ts
│   │   │   ├── ConfigService.ts
│   │   │   └── AuthService.ts
│   │   └── ports/               # Interfaces (dependency inversion)
│   │       ├── ICameraRepository.ts
│   │       ├── IEvidenceRepository.ts
│   │       ├── IMessageBroker.ts
│   │       ├── IConfigPublisher.ts
│   │       ├── IAuthProvider.ts
│   │       └── IStreamProvider.ts
│   │
│   ├── infrastructure/          # External adapters (implements ports)
│   │   ├── db/                  # Database implementations
│   │   │   ├── client.ts        # Database client connection
│   │   │   ├── seed.ts          # Seed data for development
│   │   │   └── repositories/    # Repository implementations
│   │   │       ├── CameraRepository.ts
│   │   │       └── EvidenceRepository.ts
│   │   │
│   │   ├── messaging/           # Message broker adapters
│   │   │   └── KafkaConsumer.ts  # Kafka consumer (argos.events)
│   │   │
│   │   ├── engine/              # Engine communication
│   │   │   └── EngineConfigClient.ts # HTTP webhook client for config changes
│   │   │
│   │   ├── auth/                # Auth adapter
│   │   │   ├── TokenValidator.ts # Static token validation
│   │   │   └── middleware.ts    # Auth middleware
│   │   │
│   │   ├── grpc/                # gRPC client for engine communication
│   │   │   ├── client.ts        # gRPC client setup
│   │   │   └── protos/          # Protocol buffer definitions
│   │   │       └── video.proto
│   │   │
│   │   ├── http/                # HTTP server (Fastify)
│   │   │   ├── server.ts        # Fastify instance
│   │   │   ├── plugins/         # Fastify plugins
│   │   │   │   ├── cors.ts
│   │   │   │   ├── helmet.ts
│   │   │   │   ├── rateLimit.ts
│   │   │   │   └── swagger.ts
│   │   │   └── routes/          # Route registrations
│   │   │       ├── cameras.ts
│   │   │       ├── events.ts
│   │   │       ├── config.ts
│   │   │       └── health.ts
│   │   │
│   │   └── sse/                 # SSE implementation
│   │       └── SSEManager.ts    # Client connection manager
│   │
│   ├── presentation/            # HTTP interface layer
│   │   ├── controllers/         # REST controllers
│   │   │   ├── CameraController.ts
│   │   │   ├── EventController.ts
│   │   │   ├── ConfigController.ts
│   │   │   └── SSEController.ts
│   │   ├── middleware/          # HTTP middleware
│   │   │   ├── auth.middleware.ts
│   │   │   ├── validation.middleware.ts
│   │   │   ├── error.middleware.ts
│   │   │   └── requestId.middleware.ts
│   │   └── dto/                 # Data Transfer Objects
│   │       ├── CameraDTO.ts
│   │       ├── EventDTO.ts
│   │       └── ConfigDTO.ts
│   │
│   ├── config/                  # Configuration management
│   │   ├── env.ts               # Environment variables (Zod validation)
│   │   ├── database.ts          # DB connection config
│   │   ├── kafka.ts             # Kafka broker config
│   │   └── logger.ts            # Pino logger config
│   │
│   └── main.ts                  # Application entry point
│
├── tests/
│   ├── unit/                    # Unit tests
│   │   ├── services/
│   │   └── utils/
│   ├── integration/             # Integration tests
│   │   ├── api/
│   │   └── db/
│   └── e2e/                     # End-to-end tests
│       └── scenarios/
│
├── docs/                        # Documentation
├── package.json
├── tsconfig.json
├── .eslintrc.json
├── .prettierrc
├── docker-compose.yml
└── Dockerfile
```

---

## Core Patterns

### 1. Dependency Injection

**Why**: Testability, flexibility, loose coupling

**Implementation**:
```typescript
// application/services/EventService.ts
export class EventService {
  constructor(
    private eventRepo: IEvidenceRepository,
    private messageBroker: IMessageBroker,
    private logger: Logger
  ) {}

  async handleNewEvent(event: DetectedEvent): Promise<void> {
    this.logger.info({ eventId: event.id }, 'Processing event')

    await this.eventRepo.save(event)
    await this.messageBroker.publish('events:processed', event)

    this.logger.info({ eventId: event.id }, 'Event processed successfully')
  }
}
```

**Injection in main.ts**:
```typescript
// main.ts
const dbClient = new DatabaseClient(process.env.DB_CONNECTION_STRING)
const eventRepo = new EvidenceRepository(dbClient)
const cameraRepo = new CameraRepository(dbClient)
const logger = pino()

const eventService = new EventService(eventRepo, logger)
const sseManager = new SSEManager()
const configService = new ConfigService(cameraRepo, configPublisher, logger)

// Kafka consumer for events + HTTP client for config webhook
const kafkaConsumer = new KafkaEventConsumer(kafka, eventService, sseManager, logger)
const engineConfigClient = new EngineConfigClient(process.env.ENGINE_URL, logger)
```

**Benefits**:
- Easy to mock dependencies in tests
- Swap implementations without changing service code
- Clear dependency graph

---

### 2. Repository Pattern

**Why**: Abstract database access, enable testing, support multiple storage backends

**Port (Interface)**:
```typescript
// application/ports/IEvidenceRepository.ts
export interface IEvidenceRepository {
  save(event: DetectedEvent): Promise<void>
  findById(id: string): Promise<DetectedEvent | null>
  findByCamera(cameraId: string, limit: number): Promise<DetectedEvent[]>
  findRecent(limit: number): Promise<DetectedEvent[]>
  markAsNotified(id: string): Promise<void>
}
```

**Adapter (Implementation)**:
```typescript
// infrastructure/db/repositories/EvidenceRepository.ts
export class EvidenceRepository implements IEvidenceRepository {
  constructor(private db: DatabaseClient) {}

  async save(event: DetectedEvent): Promise<void> {
    await this.db.collection('evidence').insertOne({
      id: event.id,
      cameraId: event.cameraId,
      timestamp: new Date(event.timestamp),
      type: event.type,
      severity: event.severity,
      description: event.description,
      confidence: event.confidence,
      notified: event.notified,
      metadata: event.metadata,
      createdAt: new Date()
    })
  }

  async findById(id: string): Promise<DetectedEvent | null> {
    const doc = await this.db.collection('evidence').findOne({ id })
    return doc ? this.toDomain(doc) : null
  }

  async findByCamera(cameraId: string, limit: number): Promise<DetectedEvent[]> {
    const docs = await this.db.collection('evidence')
      .find({ cameraId })
      .sort({ timestamp: -1 })
      .limit(limit)
      .toArray()
    return docs.map(this.toDomain)
  }

  // ... other methods

  private toDomain(doc: any): DetectedEvent {
    return {
      id: doc.id,
      cameraId: doc.cameraId,
      timestamp: doc.timestamp.toISOString(),
      type: doc.type,
      severity: doc.severity,
      description: doc.description,
      confidence: doc.confidence,
      notified: doc.notified,
      metadata: doc.metadata
    }
  }
}
```

**Benefits**:

- Business logic doesn't know about the database driver
- Easy to write in-memory repository for tests
- Can migrate to different database without changing services

---

### 3. Request Context Pattern

**Why**: Propagate correlation IDs and request metadata through all layers

**Implementation**:

```typescript
// domain/types/RequestContext.ts
export interface RequestContext {
  authenticated: boolean
  correlationId: string
  timestamp: Date
}
```

**Middleware**:

```typescript
// presentation/middleware/auth.middleware.ts
export async function authMiddleware(request: FastifyRequest, reply: FastifyReply) {
  const token = extractToken(request.headers.authorization)

  if (!token) {
    return reply.code(401).send({ error: { code: 'UNAUTHORIZED', message: 'Missing token' } })
  }

  try {
    const result = await authProvider.validate(token)

    request.context = {
      authenticated: result.authenticated,
      correlationId: result.correlationId,
      timestamp: new Date()
    }
  } catch (error) {
    return reply.code(401).send({ error: { code: 'UNAUTHORIZED', message: 'Invalid token' } })
  }
}
```

**Usage in Controllers**:

```typescript
// presentation/controllers/EventController.ts
export class EventController {
  async notifyEvent(request: FastifyRequest, reply: FastifyReply) {
    const { id } = request.params
    const context = request.context // RequestContext

    await this.eventService.notifyEvent(id, context)

    reply.send({ success: true, notifiedAt: new Date().toISOString() })
  }
}
```

---

### 4. Error Handling Strategy

**Domain Errors**:

```typescript
// domain/errors/CameraNotFoundError.ts
export class CameraNotFoundError extends Error {
  constructor(public cameraId: string) {
    super(`Camera ${cameraId} not found`)
    this.name = 'CameraNotFoundError'
  }
}

// domain/errors/ConfigValidationError.ts
export class ConfigValidationError extends Error {
  constructor(public details: Record<string, string>) {
    super('Configuration validation failed')
    this.name = 'ConfigValidationError'
  }
}

// domain/errors/UnauthorizedError.ts
export class UnauthorizedError extends Error {
  constructor(message: string = 'Unauthorized') {
    super(message)
    this.name = 'UnauthorizedError'
  }
}
```

**Error Middleware**:

```typescript
// presentation/middleware/error.middleware.ts
export function errorHandler(error: Error, request: FastifyRequest, reply: FastifyReply) {
  const logger = request.log

  if (error instanceof CameraNotFoundError) {
    return reply.code(404).send({
      error: {
        code: 'NOT_FOUND',
        message: error.message,
        timestamp: new Date().toISOString()
      }
    })
  }

  if (error instanceof ConfigValidationError) {
    return reply.code(422).send({
      error: {
        code: 'VALIDATION_ERROR',
        message: error.message,
        details: error.details,
        timestamp: new Date().toISOString()
      }
    })
  }

  if (error instanceof UnauthorizedError) {
    return reply.code(401).send({
      error: {
        code: 'UNAUTHORIZED',
        message: error.message,
        timestamp: new Date().toISOString()
      }
    })
  }

  // Zod validation errors
  if (error instanceof ZodError) {
    const details = error.errors.reduce((acc, err) => {
      acc[err.path.join('.')] = err.message
      return acc
    }, {})

    return reply.code(400).send({
      error: {
        code: 'BAD_REQUEST',
        message: 'Invalid request data',
        details,
        timestamp: new Date().toISOString()
      }
    })
  }

  // Unexpected errors
  logger.error({ err: error, requestId: request.id }, 'Unhandled error')
  return reply.code(500).send({
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: 'An unexpected error occurred',
      timestamp: new Date().toISOString()
    }
  })
}
```

---

## Authentication Flow

**Static Bearer Token Validation**

```
Frontend
    │
    │ GET /api/cameras
    │ Authorization: Bearer <token>
    ▼
BFF Auth Middleware
    │
    │ Extract token from Authorization header
    │ Compare against AUTH_TOKEN env var
    │
    ├── Token matches ──► Create RequestContext (correlationId, timestamp)
    │                     │
    │                     ▼
    │                  Controller processes request
    │                     │
    │                     ▼
    │                  200 OK + Response data
    │
    └── Token invalid ──► 401 Unauthorized
```

**Implementation**:

```typescript
// infrastructure/auth/TokenValidator.ts
import crypto from 'node:crypto'

export interface AuthResult {
  authenticated: boolean
  correlationId: string
}

export class StaticTokenValidator implements IAuthProvider {
  private validTokens: string[]

  constructor(authTokenEnv: string) {
    // Support multiple tokens separated by comma (for rotation)
    this.validTokens = authTokenEnv.split(',').map(t => t.trim()).filter(Boolean)
  }

  async validate(token: string): Promise<AuthResult> {
    if (!this.validTokens.includes(token)) {
      throw new UnauthorizedError('Invalid bearer token')
    }
    return { authenticated: true, correlationId: crypto.randomUUID() }
  }
}
```

**Auth Middleware**:

```typescript
// presentation/middleware/auth.middleware.ts
function extractToken(header: string | undefined): string | null {
  if (!header) return null
  const [scheme, token] = header.split(' ')
  if (scheme !== 'Bearer' || !token) return null
  return token
}

export function createAuthMiddleware(authProvider: IAuthProvider) {
  return async function authMiddleware(request: FastifyRequest, reply: FastifyReply) {
    const token = extractToken(request.headers.authorization)

    if (!token) {
      return reply.code(401).send({
        error: { code: 'UNAUTHORIZED', message: 'Missing bearer token' }
      })
    }

    try {
      const result = await authProvider.validate(token)
      request.context = {
        authenticated: result.authenticated,
        correlationId: result.correlationId,
        timestamp: new Date()
      }
    } catch (error) {
      return reply.code(401).send({
        error: { code: 'UNAUTHORIZED', message: 'Invalid bearer token' }
      })
    }
  }
}
```

**IAuthProvider Interface** (preserved for future OAuth migration):

```typescript
// application/ports/IAuthProvider.ts
export interface IAuthProvider {
  validate(token: string): Promise<AuthResult>
}
```

When the project migrates to OAuth/OIDC in the future, a new adapter (e.g., `OIDCTokenValidator`) can implement `IAuthProvider` without changing any service or middleware code.

---

## Event Processing Pipeline

**Architecture**:

```
Argos Engine
    │
    │ Produces to Kafka
    ▼
Kafka (argos.events topic)
    │
    │ Consume (eachMessage)
    ▼
BFF KafkaConsumer
    ├── Validate event schema (Zod)
    ├── Save to evidence collection (document DB)
    ├── SSE Manager broadcast to connected clients
    └── Commit offset on success
    │
    ▼
Frontend Clients (EventSource)
```

**Why This Architecture?**

- **Persistent messages**: Kafka retains messages on disk; if the BFF restarts, it resumes from the last committed offset
- **Offset-based retry**: If processing fails, the offset is not committed and the message is re-delivered on the next poll -- no separate retry queue needed
- **Ordered delivery per partition**: Events for a given camera can be keyed to the same partition for strict ordering
- **No extra infrastructure**: No BullMQ or Redis needed for event processing -- Kafka handles persistence, retry, and delivery
- **Simplicity**: Single consumer processes events end-to-end (validate, save, broadcast) in one step

**Implementation**:

**Kafka Event Consumer**:

```typescript
// infrastructure/messaging/KafkaConsumer.ts
import { Kafka, Consumer, EachMessagePayload } from '@kafkajs'

export class KafkaEventConsumer {
  private consumer: Consumer

  constructor(
    private kafka: Kafka,
    private eventService: EventService,
    private sseManager: SSEManager,
    private logger: Logger
  ) {
    this.consumer = kafka.consumer({ groupId: 'bff-events' })
  }

  async start(): Promise<void> {
    await this.consumer.connect()
    await this.consumer.subscribe({ topic: 'argos.events', fromBeginning: false })

    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }: EachMessagePayload) => {
        try {
          const data = JSON.parse(message.value!.toString())
          const event = DetectedEventSchema.parse(data)

          await this.eventService.saveEvent(event)
          this.sseManager.broadcast(event)

          this.logger.info({ eventId: event.id, partition }, 'Event processed')
        } catch (error) {
          this.logger.error({ error, topic, partition, offset: message.offset }, 'Failed to process event')
          // Do not commit offset -- message will be re-delivered
          throw error
        }
      }
    })
  }

  async stop(): Promise<void> {
    await this.consumer.disconnect()
  }
}
```

---

## SSE Implementation

**Architecture**:

```
Frontend Client (EventSource)
    │
    │ GET /api/events/stream
    ▼
BFF Instance
    ├── Auth Middleware (verify token)
    ├── SSE Controller (establish connection)
    └── SSEManager.addClient(correlationId, response)
    │
    ▼
SSEManager (In-Memory Connection Map)
    │
    │ KafkaConsumer calls sseManager.broadcast(event)
    │ directly after processing each message
    ▼
SSEManager.broadcast(event)
    │
    │ For each connected client
    ▼
Send SSE event to Frontend
```

**Multi-Instance Support**:

For v1 (single BFF instance), the Kafka consumer processes events and calls `sseManager.broadcast()` directly -- no cross-instance fanout is needed.

For multi-instance deployments, each BFF instance runs its own Kafka consumer in the same consumer group (e.g., `bff-events`). Since the topic `argos.events` would need to be received by all instances, each instance can use a **separate consumer group** (e.g., `bff-events-{instanceId}`) so that all instances receive all events and broadcast locally to their connected SSE clients. No Redis or additional pub/sub layer is required.

**Implementation**:

**SSE Manager**:

```typescript
// infrastructure/sse/SSEManager.ts
export class SSEManager {
  private clients = new Map<string, Set<FastifyReply>>()

  addClient(correlationId: string, reply: FastifyReply) {
    if (!this.clients.has(correlationId)) {
      this.clients.set(correlationId, new Set())
    }

    this.clients.get(correlationId)!.add(reply)

    reply.raw.on('close', () => {
      this.removeClient(correlationId, reply)
    })

    // Send initial heartbeat
    this.sendEvent(reply, 'heartbeat', { timestamp: new Date().toISOString() })

    // Start heartbeat interval
    const interval = setInterval(() => {
      if (reply.raw.destroyed) {
        clearInterval(interval)
        return
      }
      this.sendEvent(reply, 'heartbeat', { timestamp: new Date().toISOString() })
    }, 30000)
  }

  removeClient(correlationId: string, reply: FastifyReply) {
    const clients = this.clients.get(correlationId)
    if (clients) {
      clients.delete(reply)
      if (clients.size === 0) {
        this.clients.delete(correlationId)
      }
    }
  }

  broadcast(event: DetectedEvent) {
    // Fan out directly to all locally connected clients
    for (const [_id, clients] of this.clients.entries()) {
      for (const reply of clients) {
        this.sendEvent(reply, 'message', event)
      }
    }
  }

  private sendEvent(reply: FastifyReply, type: string, data: any) {
    if (reply.raw.destroyed) return

    try {
      reply.raw.write(`event: ${type}\n`)
      reply.raw.write(`data: ${JSON.stringify(data)}\n\n`)
    } catch (error) {
      logger.error({ error }, 'Failed to send SSE event')
    }
  }

  async gracefulShutdown() {
    for (const [_id, clients] of this.clients.entries()) {
      for (const reply of clients) {
        reply.raw.end()
      }
    }
    this.clients.clear()
  }
}
```

**SSE Controller**:

```typescript
// presentation/controllers/SSEController.ts
export class SSEController {
  constructor(private sseManager: SSEManager) {}

  async streamEvents(request: FastifyRequest, reply: FastifyReply) {
    const context = request.context // RequestContext (from auth middleware)

    reply.raw.setHeader('Content-Type', 'text/event-stream')
    reply.raw.setHeader('Cache-Control', 'no-cache')
    reply.raw.setHeader('Connection', 'keep-alive')
    reply.raw.setHeader('X-Accel-Buffering', 'no') // Disable nginx buffering

    this.sseManager.addClient(context.correlationId, reply)

    // Keep connection open
    await new Promise(() => {}) // Never resolves
  }
}
```

---

## Database Strategy

### Document Model

**Two Collections** (plus notifications):

1. **`cameras`**: Camera configurations and connection details
2. **`evidence`**: Detected events from the engine

**Benefits**:

- Separate concerns (config vs events)
- Different backup strategies (cameras = frequent, evidence = archival)
- Easier to scale (shard evidence collection by date if needed)

### Camera Document

The camera document extends the engine's `CameraConfig` schema (see [`docs/engine/API_CONTRACT.md`](../engine/API_CONTRACT.md)) with BFF-only fields.

```typescript
{
  // --- Engine-shared fields (sent to engine via webhook / GET /api/cameras/engine) ---
  id: string,                    // Unique identifier
  name: string,                  // Human-readable name
  enabled: boolean,              // Whether camera is active
  connection: {
    stream_url: string,          // RTSP URL or local device index
    protocol: 'rtsp' | 'local', // Connection protocol
    frame_interval: number       // Seconds between frame captures
  },
  frame: {
    target_width: number,        // Frame resize width (pixels)
    target_height: number,       // Frame resize height (pixels)
    jpeg_quality: number         // JPEG encode quality (1-100)
  },
  observation: {
    model: string,               // VLM model in provider:model format
    kwargs: object,              // Extra kwargs for model init
    system_prompt: string        // Instruction sent to VLM alongside each frame
  },
  decision: {
    model: string,               // LLM model in provider:model format
    kwargs: object,              // Extra kwargs for model init
    system_prompt: string,       // Instruction sent to LLM alongside stories
    json_schema: object        // JSON Schema for structured output
  },

  // --- BFF-only fields (never sent to engine) ---
  description: string,           // Camera description/location info (for UI)
  credentials?: {                // Optional authentication (never in responses)
    username: string,
    password: string
  },
  status: 'online' | 'offline' | 'error',
  lastSeen: Date,
  createdAt: Date,
  updatedAt: Date
}
```

**Example camera document as stored in DB**:
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
  "frame": { "target_width": 1280, "target_height": 720, "jpeg_quality": 85 },
  "observation": {
    "model": "openai:gpt-4o",
    "kwargs": {},
    "system_prompt": "Monitor the entrance and describe all persons."
  },
  "decision": {
    "model": "openai:gpt-4o",
    "kwargs": {},
    "system_prompt": "Alert if unauthorized person detected after hours.",
    "json_schema": {}
  },
  "description": "Front door camera covering the lobby",
  "credentials": { "username": "admin", "password": "secret" },
  "status": "online",
  "lastSeen": "2026-02-13T10:30:00.000Z",
  "createdAt": "2026-02-01T00:00:00.000Z",
  "updatedAt": "2026-02-13T10:30:00.000Z"
}
```

### Evidence Document

```typescript
{
  id: string,                    // Unique identifier
  cameraId: string,              // Reference to camera
  timestamp: Date,               // When the event occurred
  type: string,                  // Event type (normal, anomaly, etc.)
  severity?: string,             // low, medium, high, critical
  description: string,           // Human-readable description
  confidence: number,            // 0-1 confidence score
  notified: boolean,             // Whether notification was sent
  metadata?: Record<string, any>,// Additional event data
  createdAt: Date
}
```

### Repository Implementation

```typescript
// infrastructure/db/repositories/CameraRepository.ts
export class CameraRepository implements ICameraRepository {
  constructor(private db: DatabaseClient) {}

  async findAll(): Promise<Camera[]> {
    return this.db.collection('cameras').find().toArray()
  }

  async findById(id: string): Promise<Camera | null> {
    return this.db.collection('cameras').findOne({ id })
  }

  async create(input: CameraCreateInput): Promise<Camera> {
    const doc = {
      ...input,
      status: 'offline',
      lastSeen: new Date(),
      createdAt: new Date(),
      updatedAt: new Date()
    }
    await this.db.collection('cameras').insertOne(doc)
    return doc
  }

  async update(id: string, data: CameraUpdateInput): Promise<Camera> {
    const result = await this.db.collection('cameras').findOneAndUpdate(
      { id },
      { $set: { ...data, updatedAt: new Date() } },
      { returnDocument: 'after' }
    )
    if (!result) throw new CameraNotFoundError(id)
    return result
  }

  async delete(id: string): Promise<void> {
    const result = await this.db.collection('cameras').deleteOne({ id })
    if (result.deletedCount === 0) throw new CameraNotFoundError(id)
  }

  async updateStatus(id: string, status: string): Promise<void> {
    await this.db.collection('cameras').updateOne(
      { id },
      { $set: { status, lastSeen: new Date(), updatedAt: new Date() } }
    )
  }
}
```

### Indexing Strategy

**Critical Indexes**:

- `evidence(cameraId, timestamp)`: Camera event history queries
- `evidence(type, timestamp)`: Filter anomalies by time
- `evidence(notified, timestamp)`: Find un-notified events
- `cameras(status)`: Filter online cameras
- `cameras(id)`: Unique lookups (primary key)

### Database Client

```typescript
// infrastructure/db/client.ts
export function createDatabaseClient(connectionString: string): DatabaseClient {
  const client = new MongoClient(connectionString, {
    maxPoolSize: 20,
    minPoolSize: 5,
    maxIdleTimeMS: 600000,
    connectTimeoutMS: 10000
  })

  return client
}

// Connection pool config via DB_CONNECTION_STRING
// mongodb://user:pass@host:27017/argos?maxPoolSize=20&retryWrites=true
```

---

## API Design

### REST Conventions

**Resource-Based URLs**:

- `/api/cameras` (collection)
- `/api/cameras/:id` (resource)
- `/api/cameras/:id/stream` (video stream proxy)

**HTTP Methods**:

- `GET`: Retrieve resource(s)
- `POST`: Create resource or trigger action
- `PUT`: Replace/update entire resource
- `DELETE`: Remove resource

**Status Codes**:
- `200 OK`: Successful GET, PUT, POST (non-creation)
- `201 Created`: Resource created
- `204 No Content`: Successful DELETE
- `400 Bad Request`: Invalid input
- `401 Unauthorized`: Missing/invalid token
- `404 Not Found`: Resource doesn't exist
- `409 Conflict`: State conflict (e.g., already notified)
- `422 Unprocessable Entity`: Validation failed
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Unexpected error
- `503 Service Unavailable`: Dependency down

### Endpoint Summary

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/api/cameras` | Yes | List all cameras |
| `GET` | `/api/cameras/:id` | Yes | Get camera by ID |
| `POST` | `/api/cameras` | Yes | Create a new camera |
| `PUT` | `/api/cameras/:id` | Yes | Update camera configuration |
| `DELETE` | `/api/cameras/:id` | Yes | Delete a camera |
| `GET` | `/api/cameras/:id/stream` | Yes | Proxy video stream from engine via gRPC |
| `GET` | `/api/cameras/engine` | Yes | Internal: return cameras in engine CameraConfig format |
| `GET` | `/api/events/stream` | Yes | SSE stream of real-time events |
| `GET` | `/api/events/history` | Yes | Paginated event history |
| `GET` | `/api/events/:id` | Get event by ID |
| `GET` | `/health` | Basic health check |
| `GET` | `/health/ready` | Readiness check (DB, Kafka, Engine) |

### Request/Response Patterns

**Request Validation** (Zod + Fastify):
```typescript
// domain/schemas/camera.schema.ts
const ConnectionSchema = z.object({
  stream_url: z.string(),
  protocol: z.enum(['rtsp', 'local']).default('rtsp'),
  frame_interval: z.number().positive().default(1.0)
})

const FrameSchema = z.object({
  target_width: z.number().int().positive().default(1280),
  target_height: z.number().int().positive().default(720),
  jpeg_quality: z.number().int().min(1).max(100).default(85)
})

const ObservationSchema = z.object({
  model: z.string().default('openai:gpt-4o'),
  kwargs: z.record(z.unknown()).default({}),
  system_prompt: z.string().min(1)
})

const DecisionSchema = z.object({
  model: z.string().default('openai:gpt-4o'),
  kwargs: z.record(z.unknown()).default({}),
  system_prompt: z.string().min(1),
  json_schema: z.record(z.unknown()).default({})
})

const CredentialsSchema = z.object({
  username: z.string(),
  password: z.string()
})

const CreateCameraSchema = z.object({
  name: z.string().min(1),
  description: z.string().optional().default(''),
  enabled: z.boolean().default(true),
  connection: ConnectionSchema,
  frame: FrameSchema.optional().default({}),
  observation: ObservationSchema,
  decision: DecisionSchema,
  credentials: CredentialsSchema.optional()
})

fastify.post('/api/cameras', {
  schema: {
    body: CreateCameraSchema
  }
}, async (request, reply) => {
  const camera = await configService.createCamera(request.body)
  reply.code(201).send({ data: camera, timestamp: new Date().toISOString() })
})
```

**Update Camera**:
```typescript
const UpdateCameraSchema = z.object({
  name: z.string().min(1).optional(),
  description: z.string().optional(),
  enabled: z.boolean().optional(),
  connection: ConnectionSchema.partial().optional(),
  frame: FrameSchema.partial().optional(),
  observation: ObservationSchema.partial().optional(),
  decision: DecisionSchema.partial().optional(),
  credentials: CredentialsSchema.optional()
})

fastify.put('/api/cameras/:id', {
  schema: {
    body: UpdateCameraSchema,
    params: z.object({ id: z.string() })
  }
}, async (request, reply) => {
  const camera = await configService.updateCamera(request.params.id, request.body)
  reply.send({ data: camera, timestamp: new Date().toISOString() })
})
```

**Response Format**:
```typescript
// Success
{
  "data": { /* payload */ },
  "timestamp": "2026-02-18T10:40:00Z"
}

// Error
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": { /* field-level errors */ },
    "timestamp": "2026-02-18T10:40:00Z"
  }
}
```

### Pagination (v2)

```typescript
// GET /api/events/history?page=1&limit=50
{
  "events": [ /* ... */ ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 230,
    "totalPages": 5,
    "hasNext": true,
    "hasPrev": false
  }
}
```

---

## Message Broker Integration

### Communication Architecture

The BFF uses two communication channels for Engine integration (see [ADR-003](../adr/003-config-propagation-webhook.md)):

1. **Kafka** (`argos.events` topic) -- receive detected events from the Engine (high-throughput, unidirectional)
2. **HTTP webhook** -- send configuration changes to the Engine (low-frequency, request/response)

```
                    ┌─────────────┐
                    │   Apache    │
                    │    Kafka    │
                    └─────────────┘
                          │
                          │  argos.events
                          │  (consume)
                          ▼
                    ┌─────────────┐
                    │     BFF     │
                    └─────────────┘
                     │           ▲
  POST /config/      │           │  Kafka produce
  cameras (webhook)  │           │  (argos.events)
                     ▼           │
                    ┌─────────────┐
                    │   Engine    │
                    └─────────────┘
```

### Config Webhook Client

When camera configuration is updated through the API, the BFF sends an HTTP POST to the Engine so it can apply the change:

```typescript
// infrastructure/engine/EngineConfigClient.ts
export interface ConfigChange {
  type: 'camera:created' | 'camera:updated' | 'camera:deleted'
  cameraId: string
  config?: Camera
  timestamp: string
}

export class EngineConfigClient implements IConfigPublisher {
  constructor(private engineUrl: string, private logger: Logger) {}

  async publishConfigChange(change: ConfigChange): Promise<void> {
    const response = await fetch(`${this.engineUrl}/config/cameras`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(change)
    })

    if (!response.ok) {
      this.logger.error(
        { type: change.type, cameraId: change.cameraId, status: response.status },
        'Failed to send config webhook to Engine'
      )
      return
    }

    this.logger.info(
      { type: change.type, cameraId: change.cameraId },
      'Config change sent to Engine via webhook'
    )
  }
}
```

### Config Propagation in Services

```typescript
// application/services/ConfigService.ts
export class ConfigService {
  constructor(
    private cameraRepo: ICameraRepository,
    private configPublisher: IConfigPublisher,
    private logger: Logger
  ) {}

  async createCamera(input: CameraCreateInput): Promise<Camera> {
    const camera = await this.cameraRepo.create(input)

    await this.configPublisher.publishConfigChange({
      type: 'camera:created',
      cameraId: camera.id,
      config: camera,
      timestamp: new Date().toISOString()
    })

    this.logger.info({ cameraId: camera.id }, 'Camera created and config webhook sent')
    return camera
  }

  async updateCamera(id: string, data: CameraUpdateInput): Promise<Camera> {
    const camera = await this.cameraRepo.update(id, data)

    await this.configPublisher.publishConfigChange({
      type: 'camera:updated',
      cameraId: camera.id,
      config: camera,
      timestamp: new Date().toISOString()
    })

    this.logger.info({ cameraId: camera.id }, 'Camera updated and config webhook sent')
    return camera
  }

  async deleteCamera(id: string): Promise<void> {
    await this.cameraRepo.delete(id)

    await this.configPublisher.publishConfigChange({
      type: 'camera:deleted',
      cameraId: id,
      timestamp: new Date().toISOString()
    })

    this.logger.info({ cameraId: id }, 'Camera deleted and config webhook sent')
  }
}
```

### IConfigPublisher Port

The port interface remains the same regardless of transport (Kafka or HTTP). This is the benefit of hexagonal architecture — if the transport changes in the future, only the adapter needs to change.

```typescript
// application/ports/IConfigPublisher.ts
export interface IConfigPublisher {
  publishConfigChange(change: ConfigChange): Promise<void>
}
```

---

## Engine Integration

The BFF integrates with the Engine via two mechanisms (see [`docs/engine/API_CONTRACT.md`](../engine/API_CONTRACT.md) and [ADR-003](../adr/003-config-propagation-webhook.md)):

1. **Startup pull**: Engine calls `GET /api/cameras/engine` on the BFF to load initial camera configbut 
2. **Runtime push**: BFF sends `POST /config/cameras` webhook on camera CRUD operations

### Internal Endpoint: GET /api/cameras/engine

This endpoint is registered on the `InternalController` — **no auth middleware** is applied. It returns all cameras excluding BFF-only fields (`description`, `credentials`, `status`, `lastSeen`, `createdAt`, `updatedAt`).

```typescript
// presentation/controllers/InternalController.ts
export class InternalController {
  constructor(private cameraService: CameraService) {}

  async getEngineConfig(request: FastifyRequest, reply: FastifyReply) {
    const cameras = await this.cameraService.getAllForEngine()
    reply.send({ cameras })
  }
}
```

The same field-projection pattern used to exclude `connection.stream_url` and `credentials` from frontend responses is used here to exclude BFF-only fields from engine responses.

---

## gRPC Video Proxy

The BFF uses `@grpc/grpc-js` to connect to the engine's gRPC video service, receive video frames, and proxy them as an HTTP stream to the frontend.

### Architecture

```
Frontend
    │
    │ GET /api/cameras/:id/stream
    │ (HTTP chunked transfer / MJPEG)
    ▼
BFF (Fastify)
    │
    │ gRPC server streaming call
    │ (VideoService.StreamFrames)
    ▼
Engine (gRPC Server)
    │
    │ Reads from camera source
    │ Encodes frames
    ▼
Camera RTSP/HTTP Source
```

### Proto Definition (this is an assumption, TBD)

```protobuf
// infrastructure/grpc/protos/video.proto
syntax = "proto3";

package argos.video;

service VideoService {
  rpc StreamFrames (StreamRequest) returns (stream FrameResponse);
}

message StreamRequest {
  string camera_id = 1;
  string format = 2;       // "mjpeg" or "raw"
  int32 quality = 3;        // 1-100
  int32 max_fps = 4;        // Frame rate limit
}

message FrameResponse {
  bytes data = 1;            // JPEG frame data
  string camera_id = 2;
  int64 timestamp = 3;       // Unix timestamp ms
  int32 width = 4;
  int32 height = 5;
}
```

### gRPC Client (this is an assumption, TBD)

```typescript
// infrastructure/grpc/client.ts
import * as grpc from '@grpc/grpc-js'
import * as protoLoader from '@grpc/proto-loader'

export function createVideoClient(engineUrl: string) {
  const packageDef = protoLoader.loadSync(
    path.join(__dirname, 'protos/video.proto'),
    { keepCase: true, longs: String, enums: String, defaults: true }
  )
  const proto = grpc.loadPackageDefinition(packageDef) as any

  return new proto.argos.video.VideoService(
    engineUrl,
    grpc.credentials.createInsecure()
  )
}
```

### Stream Proxy Controller (this is an assumption, TBD)

```typescript
// presentation/controllers/CameraController.ts (stream handler)
async streamVideo(request: FastifyRequest, reply: FastifyReply) {
  const { id } = request.params as { id: string }

  // Verify camera exists
  const camera = await this.cameraService.getCamera(id)
  if (!camera) throw new CameraNotFoundError(id)

  reply.raw.setHeader('Content-Type', 'multipart/x-mixed-replace; boundary=frame')
  reply.raw.setHeader('Cache-Control', 'no-cache')
  reply.raw.setHeader('Connection', 'keep-alive')

  const stream = this.videoClient.StreamFrames({ camera_id: id, format: 'mjpeg', quality: 80, max_fps: 15 })

  stream.on('data', (frame: FrameResponse) => {
    reply.raw.write(`--frame\r\nContent-Type: image/jpeg\r\nContent-Length: ${frame.data.length}\r\n\r\n`)
    reply.raw.write(frame.data)
    reply.raw.write('\r\n')
  })

  stream.on('error', (err: Error) => {
    logger.error({ err, cameraId: id }, 'gRPC stream error')
    reply.raw.end()
  })

  stream.on('end', () => {
    reply.raw.end()
  })

  request.raw.on('close', () => {
    stream.cancel()
  })
}
```

---

## Performance Optimization

### Database Optimization (this is a recommendation)

**Indexes**:
- Index frequently queried fields
- Compound indexes for multi-field queries
- TTL indexes for automatic evidence cleanup (optional)

**Connection Pooling**:
- Pool size: 10-20 connections per instance
- Timeout: 30 seconds
- Idle timeout: 10 minutes

**Query Optimization**:
- Use projections to fetch only needed fields
- Use sort with indexes to avoid in-memory sorts
- Batch operations where possible

**Example**:
```typescript
// Fetch only fields needed for camera list (exclude credentials)
const cameras = await db.collection('cameras')
  .find({}, { projection: { credentials: 0 } })
  .toArray()

// Efficient paginated query with index
const events = await db.collection('evidence')
  .find({ cameraId: 'cam-001' })
  .sort({ timestamp: -1 })
  .skip(page * limit)
  .limit(limit)
  .toArray()
```

### Redis Caching (Optional)

> Redis can optionally be used for caching camera lists with 30s TTL. Not required for v1 single-instance deployment.

**Cache Strategy** (if enabled):
- Camera list: TTL 30 seconds
- Configuration: TTL 5 minutes

**Example**:
```typescript
async function getCameras(): Promise<Camera[]> {
  if (redis) {
    const cached = await redis.get('cameras:list')
    if (cached) return JSON.parse(cached)
  }

  const cameras = await cameraRepo.findAll()

  if (redis) {
    await redis.set('cameras:list', JSON.stringify(cameras), 'EX', 30)
  }

  return cameras
}
```

### Kafka Consumer Optimization (this is a recommendation)

**Tuning Parameters**:
- `max.poll.records`: Control how many records are fetched per poll (default 500, tune based on processing time)
- `fetch.min.bytes`: Minimum data to fetch per request (increase for higher throughput, decrease for lower latency)
- `session.timeout.ms`: Consumer session timeout (default 30s, increase if processing is slow)

**Batch Processing**:
- Use `eachBatch` handler instead of `eachMessage` for higher throughput when individual ordering is not critical
- Commit offsets in batches to reduce Kafka coordinator overhead

**Partition Assignment**:
- For v1 single instance: all partitions are assigned to the single consumer
- For multi-instance: Kafka automatically rebalances partitions across consumer group members

**Example**:
```typescript
const consumer = kafka.consumer({
  groupId: 'bff-events',
  sessionTimeout: 30000,
  maxPollRecords: 100,
  fetchMinBytes: 1024
})
```

---

## Security Best Practices

### 1. Token Validation

- **Extract Bearer token** from Authorization header on all protected routes
- **Compare against AUTH_TOKEN** environment variable
- **Reject requests** with missing or invalid tokens immediately
- **IAuthProvider interface** allows future migration to OAuth/OIDC without code changes

### 2. Input Validation

- **Validate all inputs** with Zod schemas
- **Sanitize user input** (prevent XSS)
- **Reject unexpected fields** (strict schemas)

### 3. NoSQL Injection Prevention

- **Validate all input with Zod** before passing to database queries
- **Never pass raw user input** directly to database query operators
- **Use typed query builders** that prevent operator injection
- **Whitelist allowed fields** for update and filter operations

```typescript
// Always validate before querying
const params = CameraIdSchema.parse(request.params)
const camera = await db.collection('cameras').findOne({ id: params.id })
```

### 4. Credential Protection

- **Camera credentials** (URL, username, password) are never exposed in list API responses
- **Use projections** to exclude sensitive fields from query results
- **Separate DTO** for public camera data vs internal camera data

```typescript
// CameraDTO strips credentials from response
function toCameraDTO(camera: Camera): CameraDTO {
  const { credentials, ...publicFields } = camera
  return publicFields
}
```

### 5. Rate Limiting

```typescript
// @fastify/rate-limit
fastify.register(rateLimit, {
  max: 100,           // 100 requests
  timeWindow: 60000,  // per minute
  cache: 10000,       // Cache 10k clients
  keyGenerator: (request) => request.ip
})
```

### 6. CORS

```typescript
// @fastify/cors
fastify.register(cors, {
  origin: process.env.FRONTEND_URL, // Strict origin
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS']
})
```

### 7. Security Headers

```typescript
// @fastify/helmet
fastify.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", 'data:', 'https:']
    }
  }
})
```

### 8. Secrets Management

- **Use environment variables** for all secrets
- **Never commit secrets** to Git
- **Use `.env.example`** with dummy values
- **Rotate AUTH_TOKEN** regularly (supports comma-separated tokens for rotation)

---

## Testing Strategy

### Unit Tests (Vitest)

**Test pure functions and services**:
```typescript
// tests/unit/services/EventService.test.ts
import { describe, it, expect, vi } from 'vitest'
import { EventService } from '@/application/services/EventService'

describe('EventService', () => {
  it('should save event and publish to broker', async () => {
    const eventRepo = {
      save: vi.fn()
    }
    const messageBroker = {
      publish: vi.fn()
    }
    const logger = { info: vi.fn() }

    const service = new EventService(eventRepo, messageBroker, logger)
    const event = { id: '1', cameraId: 'cam-001', /* ... */ }

    await service.handleNewEvent(event)

    expect(eventRepo.save).toHaveBeenCalledWith(event)
    expect(messageBroker.publish).toHaveBeenCalledWith('events:processed', event)
  })
})
```

**Test auth provider**:
```typescript
// tests/unit/auth/TokenValidator.test.ts
describe('StaticTokenValidator', () => {
  it('should accept valid token', async () => {
    const validator = new StaticTokenValidator('my-secret-token')
    const result = await validator.validate('my-secret-token')
    expect(result.authenticated).toBe(true)
    expect(result.correlationId).toBeDefined()
  })

  it('should reject invalid token', async () => {
    const validator = new StaticTokenValidator('my-secret-token')
    await expect(validator.validate('wrong-token')).rejects.toThrow(UnauthorizedError)
  })
})
```

### Integration Tests (Vitest + Testcontainers)

**Test with real database and Kafka**:
```typescript
// tests/integration/api/cameras.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest'
import { MongoDBContainer } from '@testcontainers/mongodb'
import { createTestServer } from '@/tests/utils/testServer'

describe('GET /api/cameras', () => {
  let container: MongoDBContainer
  let server: FastifyInstance

  beforeAll(async () => {
    container = await new MongoDBContainer().start()
    server = await createTestServer(container.getConnectionString())
  })

  afterAll(async () => {
    await server.close()
    await container.stop()
  })

  it('should return camera list', async () => {
    const response = await server.inject({
      method: 'GET',
      url: '/api/cameras',
      headers: {
        authorization: 'Bearer valid-token'
      }
    })

    expect(response.statusCode).toBe(200)
    expect(response.json()).toEqual({
      data: expect.arrayContaining([
        expect.objectContaining({
          id: expect.any(String),
          name: expect.any(String)
        })
      ]),
      timestamp: expect.any(String)
    })
  })
})
```

### Contract Tests (Vitest)

**Validate engine CameraConfig conformance**:
```typescript
// tests/contract/engine-config.test.ts
import { describe, it, expect } from 'vitest'
import { EngineCameraConfigSchema } from '@/domain/schemas/engine-camera.schema'

describe('GET /api/cameras/engine contract', () => {
  it('should return cameras conforming to engine CameraConfig schema', async () => {
    const response = await server.inject({
      method: 'GET',
      url: '/api/cameras/engine'
    })

    expect(response.statusCode).toBe(200)
    const { cameras } = response.json()

    for (const camera of cameras) {
      // Must include connection.stream_url
      expect(camera.connection.stream_url).toBeDefined()
      // Must not include BFF-only fields
      expect(camera.description).toBeUndefined()
      expect(camera.credentials).toBeUndefined()
      expect(camera.status).toBeUndefined()
      expect(camera.lastSeen).toBeUndefined()
      expect(camera.createdAt).toBeUndefined()
      expect(camera.updatedAt).toBeUndefined()
      // Must parse against engine schema
      expect(() => EngineCameraConfigSchema.parse(camera)).not.toThrow()
    }
  })
})
```

**Validate webhook payloads**:
```typescript
// tests/contract/webhook-payload.test.ts
describe('Webhook payload contract', () => {
  it('should send engine CameraConfig on camera:created', async () => {
    const webhookPayloads: any[] = []
    // Mock engine endpoint
    mockEngine.post('/config/cameras', (req, res) => {
      webhookPayloads.push(req.body)
      res.json({ status: 'applied', cameras: [] })
    })

    await server.inject({
      method: 'POST',
      url: '/api/cameras',
      headers: { authorization: 'Bearer valid-token' },
      payload: { /* CameraCreateRequest */ }
    })

    expect(webhookPayloads).toHaveLength(1)
    const payload = webhookPayloads[0]
    expect(payload.type).toBe('camera:created')
    expect(payload.config.connection.stream_url).toBeDefined()
    expect(payload.config.description).toBeUndefined()
    expect(payload.config.credentials).toBeUndefined()
  })
})
```

### Integration Tests: GET /api/cameras/engine

```typescript
// tests/integration/api/internal.test.ts
describe('GET /api/cameras/engine', () => {
  it('should return cameras without authentication', async () => {
    const response = await server.inject({
      method: 'GET',
      url: '/api/cameras/engine'
      // No authorization header
    })

    expect(response.statusCode).toBe(200)
    expect(response.json().cameras).toBeInstanceOf(Array)
  })

  it('should include stream_url and exclude BFF-only fields', async () => {
    // Seed a camera first
    await seedCamera({ id: 'cam-test', connection: { stream_url: 'rtsp://test' } })

    const response = await server.inject({
      method: 'GET',
      url: '/api/cameras/engine'
    })

    const camera = response.json().cameras[0]
    expect(camera.connection.stream_url).toBe('rtsp://test')
    expect(camera.credentials).toBeUndefined()
    expect(camera.description).toBeUndefined()
  })
})
```

### E2E Tests

**Test full flow with frontend**:
```typescript
// tests/e2e/event-flow.test.ts
describe('Event Flow E2E', () => {
  it('should receive event via SSE after engine publishes', async () => {
    // 1. Frontend establishes SSE connection
    const eventSource = new EventSource('http://localhost:8000/api/events/stream')

    // 2. Engine produces event to Kafka
    await producer.send({
      topic: 'argos.events',
      messages: [{ key: 'cam-001', value: JSON.stringify({ id: 'evt-1', /* ... */ }) }]
    })

    // 3. Frontend should receive event via SSE
    const receivedEvent = await new Promise((resolve) => {
      eventSource.onmessage = (event) => {
        resolve(JSON.parse(event.data))
      }
    })

    expect(receivedEvent).toEqual({ id: 'evt-1', /* ... */ })
  })
})
```

---

## Deployment

### Docker

**Dockerfile**:
```dockerfile
FROM node:20-alpine AS base
RUN npm install -g pnpm

FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile --prod

FROM base AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM base AS runner
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./

ENV NODE_ENV=production
EXPOSE 8000

CMD ["node", "dist/main.js"]
```

**docker-compose.yml** (Local Dev):
```yaml
services:
  bff:
    build: ./apps/bff
    ports:
      - "8000:8000"
    environment:
      - DB_CONNECTION_STRING=mongodb://db:27017/argos
      - KAFKA_BROKERS=kafka:9092
      - AUTH_TOKEN=${AUTH_TOKEN}
      - ENGINE_URL=http://engine:9000
      # - REDIS_URL=redis://redis:6379  # Optional: only if caching enabled
    depends_on:
      - db
      - kafka

  db:
    image: mongo:7  # or other document DB
    ports:
      - "27017:27017"
    volumes:
      - db-data:/data/db

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,HOST://localhost:9092
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:29092,CONTROLLER://0.0.0.0:9093,HOST://0.0.0.0:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: "argos-local-dev-cluster-001"
    volumes:
      - kafka-data:/var/lib/kafka/data

  # redis:  # Optional: uncomment if caching is enabled
  #   image: redis:7-alpine
  #   ports:
  #     - "6379:6379"

volumes:
  db-data:
  kafka-data:
```

### Environment Variables

```bash
NODE_ENV=development
PORT=8000
DB_CONNECTION_STRING="mongodb://localhost:27017/argos"
KAFKA_BROKERS="localhost:9092"
AUTH_TOKEN="your-secret-token-here"
ENGINE_URL="http://localhost:9000"  # Used for config webhooks (POST /config/cameras)
FRONTEND_URL="http://localhost:3000"
# REDIS_URL="redis://localhost:6379"  # Optional: only if caching enabled
```

---

**Document Version**: 2.2
**Last Updated**: February 2026
