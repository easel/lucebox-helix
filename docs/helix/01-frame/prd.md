---
ddx:
  id: prd
  kind: product
  status: draft
  links:
    - id: product-vision
      rel: informs
    - id: concerns
      rel: informs
---

# Product Requirements Document

## Summary

Lucebox is a local AI inference product shipping in two integrated parts: a
build-to-order desktop box (Strix Halo + RTX 3090) and a software stack that
installs on any compatible Linux hardware. The software is organized into five
features: an inference engine (C++17, GPU-optimized kernels), an inference
server (OpenAI-compatible HTTP API), installation and tuning (host check,
Docker setup, autotune), operations (model management, benchmarking, metrics),
and users (harness adapters routing claude-code, codex, opencode, hermes-agent,
pi, and openclaw to the local endpoint). This eliminates cloud API dependency
and removes setup churn for technical users who can't get reliable performance
from generic stacks.

**Top 3 success metrics:** ≥120 tok/s sustained on Qwen3.6-27B Q4_K_M on the
launch hardware profile; ≤15 minutes from hardware boot to first harness-routed
inference; NPS ≥50 from active users at 90 days.

## Problem and Goals

### Problem

Developers who run local LLMs spend 2–8 hours per driver or runtime update
debugging quantization mismatches, VRAM overflow, and API incompatibilities
across generic stacks (Ollama, llama.cpp, LM Studio). When it works, throughput
is unpredictable because no stack owns the hardware layer. When it breaks, there
is no supported fix path. Developers who want to stop paying $50–$300/month in
cloud API subscriptions for coding agents have no reliable on-premise
alternative — the existing tools were built for flexibility across hardware,
not performance on specific hardware.

### Goals

1. Developers on the Strix Halo + RTX 3090 profile get ≥120 tok/s on
   Qwen3.6-27B without manual tuning.
2. Switching an existing agentic coding tool (claude-code, codex, opencode,
   hermes-agent, pi, openclaw) to local inference requires one command after
   the server is running.
3. Non-Lucebox hardware users can self-qualify via a compatibility check and
   install the same software stack.

### Success Metrics

| Metric | Target | Measurement Method |
|--------|--------|--------------------|
| Inference throughput (Qwen3.6-27B Q4_K_M, single-request, launch HW) | ≥120 tok/s sustained | `lucebox profile` output on reference machine |
| Time to first harness-routed inference (fresh Linux install, CLI path) | ≤15 minutes | Timed setup walkthrough on reference machine |
| Compatibility check accuracy on unsupported hardware | ≥80% diagnose specific deficiency | Manual test matrix across known-incompatible configs |
| NPS (active users, 90-day survey) | ≥50 | Monthly survey |

### Non-Goals

- Custom agentic harness / fork — deferred.
- Subscription model and curated model library — post hardware launch.
- Multi-box clustering or distributed inference.
- Windows and macOS support at launch.
- Cloud-hosted or hybrid inference modes.
- Fine-tuning pipelines — inference only at launch.

Deferred items tracked in `docs/helix/01-frame/parking-lot.md`.

## Users and Scope

### Primary Persona: Morgan — The Overcharged Developer

**Role**: Software developer using AI coding assistance daily (Claude, Copilot,
or similar). Pays $100–$300/month in cloud API subscriptions.

**Goals**: Replace cloud API spend with local inference; get fast completions
without changing how existing coding tools work; keep employer code on-premises.

**Pain Points**: Cloud API bills are unpredictable and rising; rate limits hit
at peak hours; IT is blocking cloud API access for sensitive projects; has tried
Ollama but it's too slow on their gaming PC for real use.

**Purchase trigger**: Monthly API bill exceeds a threshold, or a compliance
requirement blocks cloud access for a specific project.

### Secondary Persona: Sam — The ML Tinkerer

**Role**: ML engineer or researcher who runs model experiments locally, tracks
new model releases, and benchmarks quantization variants.

**Goals**: Stable, reproducible inference environment; fast model switching;
throughput numbers that match advertised benchmarks without manual kernel
tuning.

**Pain Points**: Spent two days recovering from a driver regression that broke
llama.cpp after a model update; can't run 27B+ models at useful speeds on a
gaming GPU with 12GB VRAM; no supported upgrade path when hardware configs
change.

**Purchase trigger**: Hit a VRAM ceiling on a model they wanted to evaluate,
or wasted a sprint recovering from a driver regression.

### Tertiary Persona: Jordan — The Privacy-First Engineer

**Role**: Security engineer, compliance-sensitive backend developer, or
independent consultant working with confidential code or data.

**Goals**: Capable AI assistance without code leaving the network; a setup
auditors can sign off on without per-vendor contracts.

**Pain Points**: IT rejected the cloud API request; OpenRouter doesn't have a
clean data-processing agreement; building a DIY air-gapped setup requires
expertise they don't have; existing local stacks don't have a supported
configuration they can document for an audit.

**Purchase trigger**: Client contract requires no code egress; or an IT
rejection forces a search for an on-premise alternative.

## Requirements

### Must Have (P0)

1. Inference engine with DFlash, DDTree, PFlash, and Megakernel optimizations running on all validated GPU profiles. *(FEAT-001)*
2. ≥120 tok/s sustained on Qwen3.6-27B Q4_K_M on the Strix Halo + RTX 3090 reference hardware. *(FEAT-001)*
3. Multi-GPU layer split across Strix Halo iGPU and RTX 3090 — required for full performance on reference hardware. *(FEAT-001)*
4. OpenAI-compatible API exposed on localhost (completions, chat, models endpoints). *(FEAT-002)*
5. `lucebox check` reports host compatibility and specific deficiencies before install. *(FEAT-003)*
6. Software installs on any compatible Linux hardware via a single bootstrap command. *(FEAT-003)*
7. Model download, list, remove, and activate via CLI with automatic quantization selection. *(FEAT-004)*
8. `lucebox profile` produces reproducible throughput benchmarks on reference hardware. *(FEAT-004)*
9. Anthropic Messages API-compatible endpoint for claude-code harness compatibility. *(FEAT-002, FEAT-005)*
10. Harness adapters for claude-code, codex, opencode, hermes-agent, pi, and openclaw route to the local endpoint. *(FEAT-005)*

### Should Have (P1)

1. Automated tuning selects optimal quantization and draft configuration for the installed hardware. *(FEAT-003)*
2. Inference server container starts on boot via a `lucebox.service` systemd unit. *(FEAT-003)*
3. Active model switching without server restart. *(FEAT-004)*
4. Inference metrics accessible in real time (tokens/sec, VRAM usage, queue depth). *(FEAT-004)*
5. 128K+ context window on Qwen3.6-27B on the reference hardware profile. *(FEAT-001)*

### Nice to Have (P2)

1. Multiple concurrent models loaded simultaneously (memory-permitting). *(FEAT-002)*
2. Shell completions for the `lucebox` CLI. *(FEAT-004)*

## Capabilities

Detailed requirements for each capability area live in the corresponding feature specification. The PRD owns scope, priority, and success criteria; the feature specs own behavior and boundaries.

### Inference Engine — [FEAT-001](features/FEAT-001-inference-engine.md)

The inference engine is a C++17 GPU runtime purpose-built for the Lucebox hardware profile. It runs speculative decoding on the decode path, speculative compression on the prefill path, and fused kernel dispatch for hybrid models. It splits computation across the Strix Halo iGPU and RTX 3090 discrete GPU, achieves ≥120 tok/s sustained on Qwen3.6-27B Q4_K_M on reference hardware, and supports context windows up to 128K tokens. Validated GPU profiles span RTX 3090 (reference), RTX 5090, RTX 4090, RTX 2080 Ti, Strix Halo HIP, and RX 7900 XTX.

### Inference Server — [FEAT-002](features/FEAT-002-inference-server.md)

The inference server wraps the engine in an HTTP API. It exposes an OpenAI-compatible endpoint for broad tool compatibility, an Anthropic Messages API-compatible endpoint for Claude Code, and a capabilities endpoint for configuration and real-time metrics. Requests are queued rather than dropped under load.

### Installation and Tuning — [FEAT-003](features/FEAT-003-installation-and-tuning.md)

Covers the full journey from bare hardware to a tuned, running server. Includes host compatibility checking with specific deficiency reporting and remediation links, a single-command bootstrap installer, Docker image management, a boot service, and an automated tuning pass that selects optimal quantization, draft model, and context window configuration for the installed hardware.

### Operations — [FEAT-004](features/FEAT-004-operations.md)

Day-to-day management of a running Lucebox system. Operators download models from a curated registry with automatic quantization selection, list and remove installed models, switch the active model without restarting the server, and run reproducible throughput benchmarks.

### Users — [FEAT-005](features/FEAT-005-users.md)

Harness adapters route claude-code, codex, opencode, hermes-agent, pi, and openclaw to the local inference endpoint without requiring a cloud API key. The claude-code adapter is compatible with the Anthropic Messages API so Claude Code routes without reconfiguration. Adapters are stateless. A browser-based management UI at `lucebox.local` is also in scope, exposing model management, metrics, and server controls via a SPA served directly from the inference server.

## Acceptance Test Sketches

| Capability | Scenario | Condition | Expected Outcome |
|------------|----------|-----------|-----------------|
| Inference Engine (FEAT-001) | Throughput on reference hardware | Benchmark run on Strix Halo + RTX 3090 with Qwen3.6-27B Q4_K_M | ≥120 tok/s sustained reported |
| Inference Server (FEAT-002) | OpenAI API surface | Model listing request to running server | Valid JSON array of available models returned |
| Inference Server (FEAT-002) | Anthropic API compat | Anthropic Messages API request to running server | Valid response conforming to Anthropic response schema |
| Installation and Tuning (FEAT-003) | Compatibility check — passing | Host check run on reference hardware | All checks pass; throughput estimate printed |
| Installation and Tuning (FEAT-003) | Compatibility check — failing | Host check run on host with 8GB VRAM GPU | VRAM check fails with specific deficiency message and remediation link |
| Operations (FEAT-004) | Auto-quantization on download | Model downloaded on host with 24GB VRAM and 128GB RAM | Q4_K_M selected automatically without user input |
| Users (FEAT-005) | Harness adapter — no API key | Coding tool launched via harness adapter | Completion served from local server; no cloud API key prompted |

## Technical Context

- **Language/Runtime**: Inference engine: C++17 with CUDA 12 / HIP 7 (custom kernels — DFlash, PFlash, megakernel); CLI and harness adapters: Python/uv (Typer-based `lucebox` package + `harness/` package); system orchestration: POSIX shell (`lucebox.sh`). See ADR-002.
- **Key Libraries**: Custom CUDA/HIP kernels; GGUF loader; uv / Typer for CLI; pytest for tests.
- **APIs**: OpenAI Chat Completions API v1; Anthropic Messages API (for `claude` harness adapter).
- **Deployment**: Inference server runs in a Docker container (`ghcr.io/luce-org/lucebox-hub:cuda12`); `lucebox.sh` manages container lifecycle; systemd manages the Docker daemon and `lucebox.service`. See ADR-003.
- **Platform Targets**: Linux x86_64 at launch; kernel ≥5.15; CUDA 12.x for RTX 3090; AMD ROCm / AMDGPU for Strix Halo iGPU.
- **Hardware Profiles**: Strix Halo (AMD Ryzen AI MAX+ 395, 128GB LPDDR5X) + RTX 3090 (24GB GDDR6X) — single launch profile.
- **Model Format**: GGUF. Any GGUF-compatible model installs via `lucebox models download` or manual placement.

Stack selection rationale: ADR-001 (license), ADR-002 (language stack), ADR-003 (Docker deployment). Command surfaces and API contracts are specified in Technical Design documents under `docs/helix/02-design/`.

## Constraints, Assumptions, Dependencies

### Constraints

- **Technical**: Linux x86_64 only at launch; CUDA 12.x required for RTX 3090 path; Docker daemon + NVIDIA Container Toolkit required on host; hybrid VRAM + unified memory routing is complex and requires validated driver combinations.
- **Business**: First hardware batch ships July 2026; software must be stable before units leave the warehouse.
- **Legal/Compliance**: Bundled open-weight models must have licenses permitting commercial distribution (Apache 2.0, MIT, or equivalent); no PII stored by the inference server. See ADR-001.

### Assumptions

- Strix Halo + RTX 3090 hardware profile delivers ≥120 tok/s on Qwen3.6-27B Q4_K_M (based on ~130 tok/s sustained claim on lucebox.com for Qwen3.5-27B; Qwen3.6-27B assumed comparable).
- claude-code, codex, opencode, hermes-agent, pi, and openclaw all support configurable local inference endpoints at launch.
- Qwen3.6-27B remains a competitive default model at the time of hardware shipment.
- `lucebox` CLI tooling already substantially exists per PRs #334 and #335 in lucebox-hub.

### Dependencies

- NVIDIA CUDA 12.x driver stability on Linux for the RTX 3090.
- AMD AMDGPU/ROCm driver stability for Strix Halo unified memory path.
- Upstream tool CLIs (claude, codex, opencode, hermes-agent, pi, openclaw) maintaining configurable local endpoint support.

## Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Driver regression breaks hybrid GPU routing | Med | High | Pin tested driver versions per release; publish tested driver matrix; `lucebox check` validates driver version |
| Upstream tool CLI (claude, codex, opencode, hermes-agent, pi, openclaw) changes its endpoint config interface | Med | Med | Harness adapters are versioned; maintain per-tool compatibility shims; alert users via release notes |
| Qwen3.6-27B superseded before hardware ships, weakening default model story | Low | Med | Model registry is updatable post-install; ship at least two validated models so default can be swapped |
| Compatible-hardware install path creates unsupported-config support burden | Med | Med | `lucebox check` gates install; publish a supported hardware list; community tier users self-support |
| July batch sells out before software is stable | Low | High | Software gates hardware fulfillment; no units ship without passing the benchmark suite on reference hardware |

## Open Questions

- [ ] What is the minimum compatible hardware spec for non-Lucebox installs (minimum VRAM, RAM, PCIe gen)? — defer to `lucebox check` for compatibility determination; exact non-Lucebox floor TBD.
- [ ] Which Linux distributions are officially supported at launch (Ubuntu 22.04/24.04, Fedora, Arch)?
- [ ] Is the Anthropic Messages API wrapper officially supported or community-maintained?
- [x] Software license: Apache 2.0. See ADR-001.
- [x] Distribution mechanism: `curl -fsSL https://raw.githubusercontent.com/Luce-Org/lucebox-hub/main/install.sh | bash`. See FEAT-003.

## Success Criteria

- A developer on a fresh Linux install with Strix Halo + RTX 3090 hardware runs `lucebox check` (all checks pass), installs Lucebox, downloads Qwen3.6-27B, and routes at least one agentic coding harness to local inference — all within 15 minutes.
- `lucebox profile` on the reference hardware reports ≥120 tok/s sustained on Qwen3.6-27B Q4_K_M before the first hardware batch ships.
- `lucebox check` on a machine with known-incompatible hardware (e.g., 8GB VRAM GPU) names the specific failing check with a link to a fix.
- Zero harness adapters require a cloud API key to function against the local Lucebox endpoint.
