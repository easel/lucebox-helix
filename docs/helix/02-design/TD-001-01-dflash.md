---
ddx:
  id: TD-001-01-dflash
  status: stub
  links:
    - id: FEAT-001
      rel: implements
---

# Technical Design: DFlash — Speculative Decoding

**TD ID**: TD-001-01-dflash
**Status**: Stub
**Owner**: Core Engine Team
**Implements**: FEAT-001 ENG-09

## Purpose

This document owns the design and acceptance criteria for DFlash speculative
decoding — the primary decode-path optimization in the Lucebox inference engine.
DFlash generates candidate tokens with a small draft model and verifies multiple
tokens per target-model forward pass, achieving near-draft-speed decoding on the
target.

## Context

DFlash is the default decode path for all supported dense transformer models.
The server binary (`dflash_server`) links against `Luce-Org/llama.cpp@luce-dflash`
(no PyTorch required at the server layer). The draft model ships separately as
a GGUF and attaches via `--draft <path>`.

Chain verification is the default mode. Tree verification (DDTree) is a separate
optimization mode that builds on DFlash; see TD-001-03-ddtree.

## Design Decisions

### Draft model lifecycle

The draft model can be held resident across requests (`persistent`), evicted
after each request's draft work (`request-scoped`), or managed automatically
(`auto`). `auto` is the default; it honors the low-VRAM heuristic and the
`--lazy-draft` flag.

| Mode | VRAM cost | Latency |
|------|-----------|---------|
| `persistent` | always allocated | lowest inter-request latency |
| `request-scoped` | freed after each request | max headroom for target model |
| `auto` | heuristic-driven | default; adapts to VRAM pressure |

`--lazy-draft` is a legacy alias for `request-scoped`.

### KV cache quantization

TQ3_0 (3.5 bits per value) is the default KV cache format, enabled via
`DFLASH27B_KV_TQ3=1`. This fits a 256K KV cache within 24 GB VRAM on the RTX
3090. Q4_0 (4.5 bpv, ~128K ceiling) is the legacy alternative.

### Sliding attention window

`--fa-window N` (default 2048) limits the flash-attention window during decode.
Setting `--fa-window 0` enables full attention. The window trades throughput for
effective context length; the default 2048 is the tested sweet spot on RTX 3090.

### Multi-GPU placement

Target and draft models can run on separate GPUs (`--target-gpu` / `--draft-gpu`).
Mixed CUDA/HIP configurations (e.g. RTX 3090 target + Strix Halo draft) require
the out-of-process draft binary (`--draft-ipc-bin`).

On the Lucebox reference hardware (RTX 3090 + Strix Halo), target weights run on
RTX 3090 CUDA. The layer-split allocation for Strix Halo unified memory is
governed by ENG-08a; this TD covers only the decode path behavior.

### Prefix cache

In-memory prefix cache slots are configured via `--prefix-cache-slots N`.
On-disk persistence uses `--kv-cache-dir` with an optional `--kv-cache-budget`
cap. Prefix cache is the enabling mechanism for the T2 throughput target
(≥60 tok/s at 128K multi-turn context).

## Server Interface

### Key flags

| Flag | Default | Description |
|------|---------|-------------|
| `--draft <path>` | required | Draft model GGUF path |
| `--fa-window N` | `2048` | Sliding FA window; `0` = full attention |
| `--draft-residency {auto,persistent,request-scoped}` | `auto` | Draft weight eviction policy |
| `--lazy-draft` | off | Legacy alias for `request-scoped` |
| `--prefix-cache-slots N` | — | In-memory prefix cache slot count |
| `--kv-cache-dir <path>` | — | On-disk prefix cache directory |
| `--kv-cache-budget N` | — | On-disk cache size cap |
| `--target-gpu N` | `0` | Target GPU index |
| `--draft-gpu N` | same as target | Draft GPU index |
| `--draft-ipc-bin <path>` | — | Out-of-process draft binary (mixed CUDA/HIP) |

### Key env vars

| Variable | Default | Description |
|----------|---------|-------------|
| `DFLASH27B_KV_TQ3=1` | default | TQ3_0 K+V cache (3.5 bpv) |
| `DFLASH27B_KV_Q4=1` | off | Q4_0 K+V cache (legacy, ~128K ceiling) |
| `DFLASH_TARGET_GPU=N` | `0` | Target GPU index (env equivalent) |
| `DFLASH_DRAFT_GPU=N` | same as target | Draft GPU index (env equivalent) |

## Performance Contract

Derived from FEAT-001 non-functional requirements:

| Target | Condition | Floor |
|--------|-----------|-------|
| T1 — decode throughput | Qwen3.6-27B Q4_K_M, RTX 3090 24GB, ~1024-token prompt, single-request | ≥120 tok/s sustained |
| T2 — decode throughput | Qwen3.6-27B Q4_K_M, 128K cached prefix context, prefix cache enabled | ≥60 tok/s sustained |

Observed baselines (from lucebox-hub README; not contractual floors for non-reference hardware):

| Hardware | Result |
|----------|--------|
| RTX 3090 (reference) | ~130 tok/s |
| RTX 5090 | 205 tok/s, 4.84× vs llama.cpp |
| RTX 2080 Ti | 53 tok/s |
| Strix Halo HIP | 37 tok/s |
| RX 7900 XTX | 50 tok/s |

## Open Questions

- [ ] What is the exact TQ3_0 encoding and how does it interact with the FA window at 128K context? (needed before T2 acceptance criteria can be written)
- [ ] What is the minimum draft model VRAM allocation in `request-scoped` mode on RTX 3090? (affects autotune guidance in FEAT-003)
- [ ] What does `--prefix-cache-slots` map to in bytes on RTX 3090 with TQ3_0 KV? (needed to size the T2 test setup)

## References

- FEAT-001 ENG-09: requirement declaration
- FEAT-001 ENG-08a: layer-split allocation (Strix Halo + RTX 3090)
- TD-001-03-ddtree: tree verification mode (builds on this TD)
- lucebox-hub README: benchmark figures and flag reference
- [DFlash benchmarks](../../lucebox-hub/server/RESULTS.md) (external)
- [DFlash blog post](https://lucebox.com/blog/dflash27b) (external)
