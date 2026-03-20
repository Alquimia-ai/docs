# Argos Engine — Product Requirements

**Version:** 1.0  
**Date:** February 2026  
**Status:** In Development

---

## Overview

The **Argos Engine** is the AI-powered video processing service that captures camera streams, analyzes frames through a VLM + LLM pipeline, and publishes detected events to a message broker.

## Core Responsibilities

1. **PreProcessor:** Connect to cameras (RTSP, local), capture frames, resize and encode
2. **Buffers:** Decouple pipeline stages with frame and description buffers
3. **Processing:** VLM describes frames, LLM analyzes temporal patterns and detects events
4. **PostProcessor:** Serialize events and publish to Kafka

## Key Requirements

### FR-001: Multi-Camera Capture
- Support RTSP streams and local camera devices
- Configurable capture rate per camera (`frame_interval`)
- Frame resizing to target resolution (default 720p)
- JPEG encoding with configurable quality

### FR-002: AI Processing Pipeline
- VLM generates natural language descriptions of frames
- LLM analyzes descriptions over time windows to detect events
- Pipeline stages decoupled via thread-safe buffers

### FR-003: Event Publishing
- Publish `DetectedEvent` objects to Kafka `argos.events` topic
- Events include: camera ID, timestamp, type, severity, description, confidence

### FR-004: Configuration
- On startup, fetch camera config from BFF (`GET /api/cameras`)
- Fallback to `ARGOS_CAMERAS_CONFIG` environment variable if BFF is unavailable
- Expose `POST /config/cameras` endpoint to receive config changes from BFF via webhook
- Apply config diff at runtime: start new cameras, stop removed cameras, update modified cameras
- Nested config: connection settings + processing settings per camera
- LLM/VLM providers configurable via LangChain class paths

## Technology Stack

| Technology | Purpose |
|-----------|---------|
| Python 3.11+ | Runtime |
| FastAPI | HTTP framework |
| OpenCV | Frame capture and processing |
| LangChain | LLM/VLM provider abstraction |
| Pydantic | Configuration validation |
| uv | Dependency management |

## Non-Functional Requirements

- Stateless design (no persistent storage in engine)
- Docker-ready with multi-stage build
- Configurable entirely via environment variables
- Graceful shutdown (drain camera threads)
