# Architecture Decision Records

This folder documents key architectural decisions made for Argos.

| ADR | Decision | Status |
|-----|----------|--------|
| [001](001-monolith-vs-separation.md) | Separated services vs monolith | ✅ Accepted |
| [002](002-pipeline.md) | Two-stage VLM + LLM pipeline with buffers | ✅ Accepted |
| [003](003-config-propagation-webhook.md) | HTTP webhook for config propagation (BFF → Engine) | ✅ Accepted |
| [004](004-concurrent-per-camera-processing.md) | Concurrent per-camera processing with independent asyncio tasks | ✅ Accepted |
| [005](005-frame-storage-in-stories-buffer.md) | Frame storage strategy for StoriesBuffer | ✅ Accepted (Phase 1) |
| [006](006-buffer-fetch-policy.md) | Buffer fetch policy — FIFO queue with Strategy pattern | ✅ Accepted |
| [007](007-broker.md) | NATS Core as the message broker for Engine → BFF events | 📝 Proposed |
| [008](008-model.md) | Unified model serving (Qwen3.5-4B) and GPU/MIG strategy | 📝 Proposed |
| [009](009-event-schema.md) | Canonical event schema and NATS subject hierarchy | 📝 Proposed |
