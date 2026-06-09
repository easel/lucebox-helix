---
ddx:
  id: FEAT-001
  status: draft
  links:
    - id: prd
      rel: informs
---

# Feature Specification: FEAT-001 — Inference Engine

**Feature ID**: FEAT-001
**Status**: Draft
**Priority**: P0
**Owner**: Core Engine Team
**Covered PRD Subsystem(s)**: Inference Engine
**Covered PRD Requirements**: FEAT-001 (Inference Engine)
**Cross-Subsystem Rationale**: None — single subsystem.

## Overview

The Inference Engine is the C++17 core that loads, manages, and executes LLM
inference on validated GPU hardware. It is the primary subject of FEAT-001 and
the sole source of inference capability for the product. Not user-facing, the
engine is wrapped by the Inference Server (FEAT-002), which exposes the
OpenAI-compatible API. All performance optimization techniques (DFlash,
PFlash, DDTree, Megakernel) are capabilities of this engine, each with a
dedicated technical design doc.

## Ideal Future State

A developer on any validated hardware profile loads a supported GGUF model,
submits a prompt, and receives a completion at the full token throughput the
GPU delivers — with no manual tuning, no VRAM overflow, and no
quantization guesswork. The engine selects the optimization path appropriate
to the model architecture and hardware automatically. Throughput and context
window behavior are predictable and reproducible: the same model + hardware +
quantization tier always produces the same throughput characteristics. Gemma
MoE hybrid architectures run on the same engine as dense transformer models
without special-case configuration. Hardware profiles — NVIDIA and AMD — are
interchangeable from the model's perspective; the engine surfaces any
hardware-specific constraint as a clear, actionable diagnostic rather than a
silent failure.

## Problem Statement

- **Current situation**: Generic inference stacks (Ollama, llama.cpp) target
  flexibility across hardware rather than performance on specific hardware
  profiles. Throughput is unpredictable because no stack owns the hardware
  kernel layer or tunes for specific GPU memory bandwidth and VRAM envelopes.
- **Pain points**: Developers on 24GB VRAM hardware cannot sustain useful
  token throughput on 27B-class models without quantization that degrades
  output quality; context windows beyond 32K cause OOM on generic stacks;
  hybrid MoE architectures (Gemma-4) require separate configuration paths
  that generic stacks do not support; driver regressions in generic stacks
  have no supported fix path.
- **Desired outcome**: On the reference RTX 3090 24GB profile, Qwen3.6-27B
  Q4_K_M sustains ≥120 tok/s on single-request load. Qwen3.6-27B with PFlash
  speculative prefill reaches 5.6× speedup over the non-speculative baseline.
  Qwen3.6-27B with DDTree speculative decoding reaches 4.84× speedup over
  baseline. All six validated hardware profiles produce documented throughput
  figures. The engine never silently discards tokens or corrupts output
  under any supported model + hardware + quantization combination.

## Functional Areas

The engine capability spans four subordinate areas: model loading and format
handling, hardware dispatch and memory management, optimization techniques,
and diagnostics. Each area is a part of the single inference capability and
cannot ship or be metrically evaluated independently.

| Area | User question or job | Feature responsibility |
|------|----------------------|------------------------|
| Model Loading | Does the engine load this GGUF model on this hardware? | Accept GGUF format; load any supported model + quantization tier onto the active hardware profile; report load failure with a specific cause |
| Hardware Dispatch | Does the engine use the GPU correctly for this hardware profile? | Target validated NVIDIA (CUDA 12) and AMD (HIP 7) hardware profiles; route memory allocation and kernel dispatch per the active profile; surface unsupported hardware as a diagnostic, not a crash |
| Optimization Techniques | Is the engine fast enough for this model and workload? | Execute DFlash speculative decoding, PFlash speculative prefill, DDTree speculative decoding, and Megakernel (hybrid MoE) paths when the model and hardware support them; fall back to the baseline path when they do not |
| Diagnostics | Why is the engine slow / failing? | Emit structured throughput, memory, and error data that FEAT-002 (Inference Server) surfaces via API and CLI |

## Requirements

### Functional Requirements by Area

#### Model Loading

ENG-01. The engine loads any GGUF file containing a supported model architecture
(dense transformer or Gemma MoE hybrid) without format-specific configuration
from the caller.

ENG-02. The engine loads GGUF files quantized at Q4_K_M, Q5_K_M, Q8_0, and F16
tiers. Any tier that exceeds the available VRAM on the active hardware profile
is rejected at load time with a specific error naming the VRAM shortfall.

ENG-03. For each supported model, the engine validates that the loaded weights
are internally consistent before accepting inference requests.

ENG-04. The engine supports all five curated model targets: Qwen3.6-27B,
Qwen3.5-27B, Gemma-4-26B-A4B, Gemma-4-31B, and Laguna-XS.2 33B. Each
model+hardware+quantization combination listed in the validated hardware
profiles table is a tested and supported configuration.

#### Hardware Dispatch

ENG-05. The engine executes on all six validated hardware profiles: RTX 3090
24GB (CUDA 12), RTX 5090 (CUDA 12), RTX 2080 Ti (CUDA 12), RTX 4090 (CUDA 12),
AMD Strix Halo HIP integrated GPU (HIP 7), and AMD RX 7900 XTX (HIP 7).

ENG-06. On hardware profiles that do not appear in the validated list, the
engine refuses to start and emits a diagnostic identifying the specific
unsupported characteristic.

ENG-07. The engine manages GPU memory within the VRAM envelope of the active
hardware profile. For 24GB profiles running Qwen3.6-27B Q4_K_M, context windows
up to 128K tokens do not cause OOM.

ENG-08. Kernel dispatch selects custom GPU kernels appropriate to the active
hardware profile. See hardware profile technical design docs (to be written,
one per validated profile).

ENG-08a. On the Strix Halo + RTX 3090 reference configuration, the engine
distributes model layer weights across the RTX 3090 VRAM (24 GB) and the
Strix Halo unified system memory (128 GB LPDDR5X), using the combined memory
envelope to support larger models and longer context windows than RTX 3090
VRAM alone permits. Compute executes on the RTX 3090 CUDA path; the Strix Halo
unified memory functions as a layer-weight offload pool. The exact allocation
policy is specified in the hardware profile technical design doc for the
reference configuration. [Note: the precise boundary between what runs on-VRAM
vs. what offloads is an open architectural detail pending TD authoring — this
requirement captures the PRD P0 intent.]

#### Optimization Techniques

ENG-09. The engine supports DFlash speculative decoding on compatible
model+hardware combinations. Behavior and performance contract are specified in
TD-001-dflash (to be written).

ENG-10. The engine supports PFlash speculative prefill on compatible
model+hardware combinations. On Qwen3.6-27B, PFlash achieves ≥5.6× prefill
speedup over the non-speculative baseline on the RTX 3090 reference hardware.
Behavior and performance contract are specified in TD-002-pflash (to be
written).

ENG-11. The engine supports DDTree speculative decoding on compatible
model+hardware combinations. On Qwen3.6-27B, DDTree achieves ≥4.84× decoding
speedup over the non-speculative baseline on the RTX 3090 reference hardware.
Behavior and performance contract are specified in TD-003-ddtree (to be
written).

ENG-12. The engine supports a Megakernel execution path for hybrid MoE
architectures (including Gemma-4-26B-A4B and Gemma-4-31B). The Megakernel path
is automatically selected when the loaded model architecture is a Gemma-style
MoE hybrid. Behavior and design are specified in TD-004-megakernel (to be
written).

ENG-13. When an optimization technique is not applicable to the active
model+hardware combination, the engine falls back to the baseline execution path
without returning an error to the caller.

#### Diagnostics

ENG-14. The engine emits a structured throughput sample (tokens per second,
measured over a sliding window) at least once per second during active
inference. FEAT-002 (Inference Server) consumes this data to serve the
`/metrics` endpoint.

ENG-15. The engine emits current VRAM allocation and peak VRAM usage after each
inference request completes.

ENG-16. Load errors, OOM events, and kernel dispatch failures produce structured
error records with a machine-readable error code and a human-readable cause
string. These are not stack traces; they are diagnostic outputs consumable by
FEAT-002.

### Non-Functional Requirements

- **Throughput — T1 (short context)**: ≥120 tok/s sustained on Qwen3.6-27B
  Q4_K_M, single-request, ~1024-token prompt, RTX 3090 24GB reference hardware.
  Sustained means the throughput average across the full 1024-token generation
  does not fall below 120 tok/s.
- **Throughput — T2 (128K multi-turn, prefix caching)**: ≥60 tok/s sustained on
  Qwen3.6-27B Q4_K_M, single-request, 128K cached prefix context (agentic
  multi-turn), DFlash prefix cache enabled, RTX 3090 24GB reference hardware.
- **Throughput — T3 (128K long-context, PFlash)**: ≥60 tok/s sustained on
  Qwen3.6-27B Q4_K_M, single-request, 128K context processed via PFlash
  speculative prefill compression, RTX 3090 24GB reference hardware.
- **Context window**: 128K token context window on Qwen3.6-27B Q4_K_M on the
  RTX 3090 24GB profile without OOM. Required for the T2 and T3 throughput
  targets above.
- **Latency**: Time-to-first-token on Qwen3.6-27B Q4_K_M with a 512-token
  prompt is ≤2 seconds on the RTX 3090 24GB reference hardware.
- **Load time**: Engine reaches the ready-for-inference state within 30 seconds
  of receiving a load request for Qwen3.6-27B Q4_K_M on the RTX 3090 24GB
  profile (model already on local disk, warm filesystem cache).
- **Crash safety**: The engine does not corrupt GGUF model files stored on disk
  under any shutdown condition (clean shutdown, SIGKILL, or OOM-killer
  intervention).
- **Output correctness**: The engine produces bit-identical output for the same
  prompt, model weights, quantization tier, and random seed across successive
  runs on the same hardware profile. This applies to all six validated hardware
  profiles.

## User Stories

- [TODO: create story] — As Morgan (Overcharged Developer), I load Qwen3.6-27B Q4_K_M and receive completions at the target throughput without configuring GPU settings.
- [TODO: create story] — As Sam (ML Tinkerer), I switch quantization tiers for a loaded model and observe the throughput difference without restarting the server.
- [TODO: create story] — As Sam, I load Gemma-4-26B-A4B on the RTX 3090 and receive completions at predictable throughput without special MoE configuration.
- [TODO: create story] — As Sam, I submit a prompt against a 128K-token context on Qwen3.6-27B and receive a completion without an OOM error.
- [TODO: create story] — As Jordan (Privacy-First Engineer), the engine refuses to start on unsupported hardware and names the specific unsupported characteristic so I can evaluate alternatives.

## Edge Cases and Error Handling

- **VRAM overflow at load time**: If the requested model + quantization tier
  exceeds available VRAM, the engine rejects the load request with a specific
  error (ENG-02). It does not attempt a partial load or silently downgrade the
  quantization tier.
- **VRAM overflow at inference time** (context growth): If a request causes
  VRAM to exceed the available envelope mid-generation (e.g., at extreme
  context lengths beyond the validated 128K limit), the engine terminates the
  in-flight generation cleanly and returns a structured error. It does not
  crash the process or corrupt the KV-cache for subsequent requests.
- **Unsupported hardware profile at engine start**: The engine refuses to
  initialize and emits an ENG-06 diagnostic. It does not fall back to a
  degraded CPU path.
- **Model file corruption**: If the GGUF file fails the ENG-03 consistency
  check, the engine rejects the load and emits a structured error. It does
  not attempt inference on partially loaded weights.
- **Optimization technique not applicable**: When DFlash, PFlash, DDTree, or
  Megakernel cannot be activated for the active model+hardware combination,
  the engine silently falls back to the baseline path (ENG-13). The caller
  is not notified of the fallback; the diagnostics channel (ENG-14) reflects
  actual throughput.
- **Concurrent load requests**: The engine processes one model load at a time.
  A second load request arriving while a load is in progress is queued, not
  rejected.
- **Incomplete GGUF file** (disk full or interrupted download): Load fails at
  the ENG-03 consistency check with a structured error. The engine does not
  hold a partial file lock.

## Success Metrics

- T1 — Sustained throughput on Qwen3.6-27B Q4_K_M, RTX 3090 24GB, single-request,
  ~1024-token prompt: ≥120 tok/s across a 1024-token generation. Measured by
  the throughput benchmark.
- T2 — Sustained throughput on Qwen3.6-27B Q4_K_M, RTX 3090 24GB, 128K cached
  prefix context (agentic multi-turn), DFlash prefix cache enabled: ≥60 tok/s.
  Measured by the throughput benchmark with caching enabled.
- T3 — Sustained throughput on Qwen3.6-27B Q4_K_M, RTX 3090 24GB, 128K context
  with PFlash compression enabled: ≥60 tok/s. Measured by the throughput
  benchmark with PFlash enabled.
- RTX 5090 throughput on Qwen3.6-27B Q4_K_M: ≥205 tok/s. Measured by the
  throughput benchmark.
- PFlash speedup on Qwen3.6-27B, RTX 3090 24GB: ≥5.6× prefill speedup vs.
  baseline. Measured by the throughput benchmark with baseline comparison.
- DDTree speedup on Qwen3.6-27B, RTX 3090 24GB: ≥4.84× decoding speedup vs.
  baseline. Measured by the throughput benchmark with baseline comparison.
- 128K context window on Qwen3.6-27B Q4_K_M, RTX 3090 24GB: generation
  completes without OOM. Verified by a dedicated context-window stress test.
- Load time for Qwen3.6-27B Q4_K_M from warm disk: ≤30 seconds on RTX 3090
  24GB. Measured by engine load-timing instrumentation.
- Zero output corruption events (bit-non-identical output for identical
  prompt + seed + weights) across 100 consecutive same-seed runs per
  supported model+hardware profile.

## Constraints and Assumptions

- The engine targets custom GPU kernels only — no llama.cpp at the inference
  layer. Kernel design details are out of scope for this spec; see the TD
  docs referenced under Optimization Techniques.
- The engine is implemented in C++17 and compiled against CUDA 12 (for NVIDIA
  profiles) and HIP 7 (for AMD profiles). The same source tree targets both
  backends.
- GGUF is the only supported model file format. Non-GGUF weights are out of
  scope.
- Validated hardware profiles are the six listed in ENG-05. Throughput
  guarantees apply only to those profiles.
- The RTX 3090 24GB profile is the reference hardware for all performance
  targets in this spec. Other profiles have documented expected throughput
  but carry no contractual floor in this document.
- AMD Strix Halo HIP support covers the integrated GPU only. The Strix Halo
  also hosts an RTX 3090 in the Lucebox reference configuration; the inference
  path on that configuration uses the RTX 3090 CUDA path, not the Strix Halo
  HIP path.
- All throughput figures (130 tok/s RTX 3090 short-context, 205 tok/s RTX 5090,
  53 tok/s RTX 2080 Ti, 37 tok/s Strix Halo HIP, 50 tok/s RX 7900 XTX) are
  benchmarked on Qwen3.6-27B Q4_K_M, single-request, and are treated as
  observed baselines, not contractual minimums, except for the three RTX 3090
  reference targets (T1: ≥120 tok/s short-context, T2: ≥60 tok/s at 128K with
  prefix caching, T3: ≥60 tok/s at 128K with PFlash) which are P0 contractual
  targets.

## Dependencies

- **Other features**: FEAT-002 (Inference Server) is the sole consumer of the
  engine's public interface. The engine has no other callers.
- **Technical designs**: TD-001-dflash, TD-002-pflash, TD-003-ddtree,
  TD-004-megakernel (all to be written) govern the design and acceptance
  criteria of the four optimization techniques. Hardware profile TDs (one per
  validated profile, to be written) govern kernel dispatch per profile.
- **PRD requirements**: FEAT-001 (Inference Engine) — throughput target, 128K context window, multi-GPU layer split, all four optimization techniques.
- **External**: CUDA 12.x on NVIDIA hardware; HIP 7 on AMD hardware. The engine
  does not depend on any higher-level inference library at the kernel dispatch
  layer.

## Out of Scope

- HTTP API surface, request queuing, and model management CLI — these belong to
  FEAT-002 (Inference Server) and FEAT-003 (Model Management, to be written).
- Kernel design, memory allocation strategy, and CUDA/HIP implementation
  details — these belong in the TD docs referenced above.
- Non-GGUF model formats (safetensors, PyTorch checkpoint, ONNX).
- CPU-only inference fallback path.
- Symmetric multi-GPU compute parallelism (tensor/pipeline parallelism sharding one request across multiple identical GPUs for compute acceleration). The Strix Halo + RTX 3090 layer split (ENG-08a) is a memory allocation strategy, not compute parallelism, and is in scope.
- Fine-tuning, training, or weight adaptation.
- Windows and macOS support.
- Cloud-hosted or hybrid inference modes.
- Any hardware profile not listed in ENG-05.
- The management UI — covered by FEAT-005.
