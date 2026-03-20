# Argos Documentation

## 📐 [Architecture](architecture/overview.md)

System overview, component diagrams, and data flow.

## 📋 [Architecture Decision Records](adr/README.md)

Key design decisions with context and rationale:
- [ADR-001: Separated Services vs Monolith](adr/001-monolith-vs-separation.md)
- [ADR-002: Two-Stage AI Pipeline](adr/002-pipeline.md)

---

## Components

### 🔧 [Engine](engine/) — AI Video Processing

| Document | Description |
|----------|-------------|
| [PRD](engine/PRD.md) | Product requirements |
| [Technical Blueprint](engine/TECHNICAL_BLUEPRINT.md) | Architecture, modules, config |

### 🌐 [BFF](bff/) — Backend for Frontend

| Document | Description |
|----------|-------------|
| [PRD](bff/PRD.md) | Product requirements |
| [Technical Blueprint](bff/TECHNICAL_BLUEPRINT.md) | Hexagonal architecture, patterns |
| [API Contract](bff/API_CONTRACT.md) | REST + SSE endpoint specification |

### 🖥️ [Web](web/) — Frontend Dashboard

| Document | Description |
|----------|-------------|
| [PRD](web/PRD.md) | Product requirements |
| [Technical Blueprint](web/TECHNICAL_BLUEPRINT.md) | Next.js architecture, components |
