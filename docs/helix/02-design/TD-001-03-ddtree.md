---
ddx:
  id: TD-001-03-ddtree
  status: stub
  links:
    - id: FEAT-001
      rel: implements
    - id: TD-001-01-dflash
      rel: extends
---

# Technical Design: DDTree — Tree Verification for Speculative Decoding

**TD ID**: TD-001-03-ddtree
**Status**: Stub
**Owner**: Core Engine Team
**Implements**: FEAT-001 ENG-11
**Extends**: TD-001-01-dflash

## Purpose

This document owns the design and acceptance criteria for DDTree tree
verification — the tree-based mode of the DFlash speculative decoding system.
DDTree builds a tree of candidate continuations from the draft model and
verifies multiple branches in a single target-model forward pass, improving
verification efficiency over the default chain mode.

## Context

DFlash's default verification mode is chain: the draft model produces one
continuation, the target verifies it token by token. DDTree replaces this with
a tree: the draft model generates a branching tree of continuations, and the
target model verifies all branches simultaneously in one forward pass.

The tree structure allows the target model to accept more draft tokens per
forward pass on average, improving throughput. The benefit scales with tree
budget and model acceptance rate; a budget of 22 is the tested sweet spot on
RTX 3090.

DDTree is enabled at server launch; it cannot be toggled per-request.

## Design Decisions

### Tree budget

`--ddtree-budget N` controls the tree size — the number of candidate tokens the
draft generates per step. A larger budget increases the expected tokens accepted
per target forward pass but also increases draft compute cost and VRAM for the
tree state.

| Hardware | Tested budget | Notes |
|----------|:------------:|-------|
| RTX 3090 (reference) | 22 | Default; balances acceptance rate vs draft cost |
| RTX 5090 | 40 | Higher VRAM headroom allows larger tree |
| DGX Spark / GB10 | re-sweep | Not yet characterized |

### Relationship to chain verify

Chain verify is the DFlash default when `--ddtree` is not set. The
`VERIFY_MODE=ddtree` env var is the harness-side equivalent of `--ddtree`.
Both control the same engine code path.

All other DFlash behaviors (KV cache, draft residency, FA window, multi-GPU
placement) apply identically in DDTree mode; this TD covers only the
tree-verification logic layered on top of TD-001-01-dflash.

## Server Interface

| Flag / env | Default | Description |
|-----------|---------|-------------|
| `--ddtree` | off | Enable tree verification (vs chain) |
| `--ddtree-budget N` | `22` | Tree size; 22 on RTX 3090, 40 on RTX 5090 |
| `VERIFY_MODE=ddtree` | — | Harness-side equivalent of `--ddtree` |

All other flags are inherited from TD-001-01-dflash.

## Performance Contract

Derived from FEAT-001 non-functional requirements:

| Target | Condition | Floor |
|--------|-----------|-------|
| DDTree speedup | Qwen3.6-27B Q4_K_M, RTX 3090 24GB, `--ddtree-budget 22`, vs non-speculative baseline | ≥4.84× decoding speedup |

Observed baselines (from lucebox-hub README):

| Model + config | Result |
|---------------|--------|
| Qwen3.6-27B + DDTree, budget 22, RTX 3090 | 4.84× vs llama.cpp |
| Qwen3.5-27B + DDTree, RTX 3090 | 3.43× vs llama.cpp |
| Qwen3.6-27B + DDTree, budget 40, RTX 5090 | 4.84× vs llama.cpp |

## Open Questions

- [ ] What is the tree structure exactly — branching factor, depth, or a flat token budget? (determines how budget translates to draft compute)
- [ ] How does acceptance rate vary with model temperature and prompt type? (needed to set realistic throughput expectations under agentic workloads)
- [ ] Does DDTree interact with the prefix cache path, and if so, is the tree state preserved across cached prefix hits?
- [ ] What budget produces the peak throughput on RTX 5090 — is 40 optimal or just tested?

## References

- FEAT-001 ENG-11: requirement declaration
- TD-001-01-dflash: base DFlash decode path (this TD extends it)
- lucebox-hub README: benchmark figures and flag reference
- [DFlash benchmarks](../../lucebox-hub/server/RESULTS.md) (external)
