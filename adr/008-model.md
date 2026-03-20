# ADR-008: Model Selection and GPU/MIG Serving Strategy for the Appliance (VLM Real-Time)

**Status:** Proposed   
**Date:** March 2026

## Context

Argos runs on an appliance to perform **real-time camera analysis** using a VLM pipeline (vision + text). We need to select the model(s) to deploy and define the **hardware and serving strategy**, including:

- GPU sizing and memory constraints (VRAM)
- Optional **MIG** partitioning (when available)
- Serving topology: **1 model ↔ 1 Pod ↔ 1 GPU (or 1 MIG slice)** to isolate performance and simplify scheduling
- Operational simplicity: single-node appliance, minimal moving parts
- Real-time: prioritize **latency and throughput**, tolerate best-effort behavior

We are choosing between:
1) **Two-model setup**
   - Text: `Qwen/Qwen3-1.7B` (Causal Language Model)   
   - Vision-Language: `Qwen/Qwen3-VL-4B-Instruct` (VLM) 
2) **Unified model setup**
   - `unsloth/Qwen3.5-4B` (Unified Vision-Language model; “Causal Language Model with Vision Encoder”) 
   - (Upstream card indicates “Unified Vision-Language Foundation … outperforms Qwen3-VL models…”) 

## Goals

- **Single deployment pattern** that works on constrained edge hardware
- **Stable latency** under bursty camera scenes
- **Simplified scheduling** and resource isolation using GPU / MIG
- Ability to run:
  - vision+text inference for camera observations
  - text-only inference for lightweight decisions (when applicable)

## Decision Drivers

1. **Latency (p95)**: keep end-to-end inference responsive for real-time UX.
2. **Throughput**: handle bursts without queue explosion.
3. **VRAM fit**: must fit weights + KV cache + vision encoder overhead with headroom.
4. **Operational simplicity**: fewer services/models to run and maintain on an appliance.
5. **Scheduling clarity**: deterministic resource assignment (1 Pod ↔ 1 GPU/MIG).
6. **Flexibility**: support both text-only and multimodal paths.

## Options Considered

### Option A: Two-model serving (Qwen3-1.7B + Qwen3-VL-4B-Instruct) — **Rejected**

- Text-only workloads go to `Qwen3-1.7B` (smaller/faster for pure text) 
- Vision-language workloads go to `Qwen3-VL-4B-Instruct` 

**Pros**
- Potentially cheaper/faster for **text-only** paths (1.7B is lighter than 4B)
- Independent scaling/tuning: text model can be optimized separately

**Cons**
- Two serving stacks to operate (two deployments, two health surfaces, two configs)
- More complex routing logic (Engine/BFF must choose model per request)
- GPU utilization becomes harder to optimize on a single-node appliance
- Higher probability of contention (two models competing for GPU time/memory)
- Harder to keep a consistent “capability envelope” (one model may lag features)

### Option B: Unified model serving (unsloth/Qwen3.5-4B) — **Chosen**

`Qwen3.5-4B` is presented as a unified vision-language model (“Causal Language Model with Vision Encoder”). :contentReference[oaicite:6]{index=6}  
Upstream highlights “Unified Vision-Language Foundation … outperforms Qwen3-VL models …”.

**Pros**
- One model covers **text** + **vision+text**, reducing operational complexity
- Simplifies routing: a single endpoint can serve both modalities
- Consistent behavior/capabilities across modes (same family, same tokenizer stack)
- Better fit for an appliance where “one robust path” beats “many moving parts”

**Cons / Trade-offs**
- Text-only may be less cost-efficient than a dedicated 1.7B model in some scenarios
- Requires careful VRAM management (KV cache + vision encoder)
- If the unified model fails/needs upgrade, it impacts both text and multimodal flows

## Decision

**Choose Option B: `unsloth/Qwen3.5-4B` as the single deployed model** for the appliance.

Additionally, adopt the serving strategy:

- **1 Pod ↔ 1 GPU** (or **1 Pod ↔ 1 MIG slice**) for strict isolation.
- Avoid multi-model concurrency on the same GPU slice unless we validate it with load tests.

## Hardware and Serving Considerations

### GPU / VRAM sizing
We must size VRAM for:
- model weights
- runtime overhead (framework, activations)
- **KV cache** based on `max_model_len`, concurrency, and batching
- vision encoder overhead (image tokens / visual embeddings)

**Guideline**
- Keep **headroom** (e.g., 15–25%) to avoid OOM under bursts.
- Treat `max_model_len` as a first-class knob: it strongly impacts KV cache.

### MIG strategy (when available)
If the GPU supports MIG:

- Partition GPU into **N slices** (e.g., 1g/2g/… depending on GPU type).
- Run **one k3s Pod per MIG slice**.
- Use a clear placement policy:
  - real-time cameras mapped to stable slices
  - avoid cross-camera jitter from shared GPU time

### Pod-to-GPU assignment (1:1)
We standardize on:
- **requests/limits** for `nvidia.com/gpu: 1` (or MIG-specific resource)
- one serving container per Pod (avoid sidecar-heavy designs on the appliance)
- bounded concurrency inside the server (to cap KV cache and latency)

## Consequences

### Positive
- Much simpler deployment/operations on the appliance (one model to run, monitor, upgrade)
- Easier GPU scheduling and more predictable performance with 1:1 assignment
- Fewer failure modes and fewer routing branches

### Negative / Trade-offs
- Text-only paths may be more expensive than a dedicated 1.7B model
- Unified model requires strict tuning for VRAM and concurrency
- If future workloads demand replayable “events”, that’s orthogonal (handled by the event bus ADR)

## Follow-ups

1. Define target SLOs (e.g., p95 inference < X ms for text-only, < Y ms for multimodal).
2. Establish “MIG sizing table” per GPU model present in appliances.
3. Run benchmark matrix:
   - `max_model_len` × concurrency × #cameras × image resolution
4. Document a fallback plan:
   - if unified model cannot meet text-only cost targets, reintroduce `Qwen3-1.7B` for text-only flows.