---
ddx:
  id: TD-001-02-pflash
  status: stub
  links:
    - id: FEAT-001
      rel: implements
---

# Technical Design: PFlash — Speculative Prefill Compression

**TD ID**: TD-001-02-pflash
**Status**: Stub
**Owner**: Core Engine Team
**Implements**: FEAT-001 ENG-10

## Purpose

This document owns the design and acceptance criteria for PFlash speculative
prefill compression. PFlash scores the input prompt using a small drafter model
(Qwen3-0.6B BF16), identifies which token blocks carry the most information,
and compresses the prompt to only those blocks before running the target model's
prefill pass. This enables the T3 throughput target: ≥60 tok/s on Qwen3.6-27B
at 128K context.

## Context

Prefill is the bottleneck at long context. At 128K tokens, a full prefill pass
through Qwen3.6-27B is prohibitively slow for interactive use. PFlash reduces
effective prefill cost by 5–10× depending on keep ratio, with the drafter paying
a small upfront cost to score and select blocks.

PFlash is orthogonal to DFlash (decode speculative decoding). Both can run in
the same server session; prefill compression reduces TTFT, DFlash reduces
per-token decode latency.

The headline 10.4× speedup requires BSA (block sparse attention) on sm_80+,
enabled via `DFLASH_FP_USE_BSA=1`. The 5.6× headline for Qwen3.6-27B is the
production target without BSA.

## Design Decisions

### Block selection and keep ratio

PFlash divides the prompt into blocks and scores each block via the drafter. The
`--prefill-keep-ratio` controls what fraction of source tokens survive compression.
The ratio is context-length-dependent: at 128K, 2% of tokens are kept; at 32K,
10% are kept. A piecewise curve can override the flat ratio.

| Context length | Default keep ratio |
|---------------|-------------------|
| ~10K tokens | ~50% |
| ~32K tokens | ~10% |
| ~40K tokens | ~20% |
| ~100K+ tokens | ~10% |

The `--prefill-curve T:R [T:R ...]` flag accepts breakpoints (token count : ratio)
and linear-interpolates between them, overriding `--prefill-keep-ratio`. A
per-session bandit override can further adjust the ratio at runtime.

### Activation threshold

In `auto` mode, PFlash activates only when the prompt exceeds
`--prefill-threshold` tokens (default 32,000). Below the threshold, the drafter
cost is not worth the compression gain and the engine falls back to full prefill.
`always` forces PFlash regardless of prompt length.

### Drafter residency

The drafter (Qwen3-0.6B BF16 GGUF) loads at request time by default and parks
between requests. `--prefill-skip-park` holds it resident across requests, trading
VRAM for lower inter-request latency.

### Block sparse attention (BSA)

`DFLASH_FP_USE_BSA=1` dispatches the compressed prefill through BSA rather than
dense attention. Requires sm_80+ (Ampere and newer). This is required to reach
the 10.4× headline figure. The production 5.6× target for Qwen3.6-27B does not
require BSA.

## Server Interface

### Key flags

| Flag | Default | Description |
|------|---------|-------------|
| `--prefill-compression {off,auto,always}` | `off` | Activation mode |
| `--prefill-threshold N` | `32000` | Token count threshold for `auto` mode |
| `--prefill-keep-ratio F` | `0.05` | Fraction of source tokens kept (flat) |
| `--prefill-curve T:R [T:R ...]` | off | Piecewise keep-ratio curve; overrides flat ratio |
| `--prefill-drafter <gguf>` | required if on | Drafter weights (Qwen3-0.6B BF16 GGUF) |
| `--prefill-skip-park` | off | Hold drafter resident across requests |

### Key env vars

| Variable | Default | Description |
|----------|---------|-------------|
| `DFLASH_FP_USE_BSA=1` | `0` | Dispatch sparse FA through BSA (sm_80+); required for 10.4× |
| `DFLASH_FP_ALPHA=0.85` | `0.12` | Block-selection threshold; higher = stricter = fewer K-blocks |
| `DFLASH_FP_PROFILE=1` | `0` | Per-stage timing log |

## Performance Contract

Derived from FEAT-001 non-functional requirements:

| Target | Condition | Floor |
|--------|-----------|-------|
| T3 — decode throughput | Qwen3.6-27B Q4_K_M, 128K context, PFlash compression enabled | ≥60 tok/s sustained |
| PFlash speedup | Qwen3.6-27B Q4_K_M, RTX 3090 24GB, vs non-speculative baseline | ≥5.6× prefill speedup |

Observed baselines (from lucebox-hub README):

| Model | Result |
|-------|--------|
| Qwen3.6-27B + PFlash, RTX 3090 | ~5.6× vs llama.cpp |
| Laguna-XS.2 33B + PFlash @128K | 5.4× vs llama.cpp |

## Open Questions

- [ ] What is the exact block size used by the scorer? (determines compression granularity)
- [ ] How does `--prefill-curve` interact with the per-session bandit override? Which wins?
- [ ] At what context length does BSA produce measurably better results than dense sparse dispatch on sm_86 (RTX 3090)? (determines whether BSA should be on by default in the Lucebox reference config)
- [ ] What is the VRAM cost of the Qwen3-0.6B BF16 drafter at runtime? (needed for autotune VRAM math in FEAT-003)

## References

- FEAT-001 ENG-10: requirement declaration
- TD-001-01-dflash: decode path (runs in same session as PFlash)
- lucebox-hub README: benchmark figures and flag reference
- [PFlash benchmarks](../../lucebox-hub/optimizations/pflash/README.md) (external)
- [PFlash blog post](https://lucebox.com/blog/pflash) (external)
