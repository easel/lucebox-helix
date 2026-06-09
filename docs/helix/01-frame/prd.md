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
installs on any compatible Linux hardware. The v1 software is CLI-first — a
tuned inference server with model management and harness adapters that route
popular agentic coding tools (claude-code, codex, opencode, hermes-agent,
pi, openclaw) to the local endpoint. This eliminates cloud API dependency and
removes setup churn for the technical users who already run local LLMs but
can't get reliable performance from generic stacks.

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
| Inference throughput (Qwen3.6-27B Q4_K_M, single-request, launch HW) | ≥120 tok/s sustained | `lucebox bench` output on reference machine |
| Time to first harness-routed inference (fresh install) | ≤15 minutes | Timed setup walkthrough on reference machine |
| Compatibility check accuracy on unsupported hardware | ≥80% diagnose specific deficiency | Manual test matrix across known-incompatible configs |
| NPS (active users, 90-day survey) | ≥50 | Monthly survey |

### Non-Goals

- Web management UI at `lucebox.local` — deferred to v2.
- Custom agentic harness / fork — deferred to v3.
- Subscription model and curated model library — post hardware launch.
- Multi-box clustering or distributed inference.
- Windows and macOS support at launch.
- Cloud-hosted or hybrid inference modes.
- Fine-tuning pipelines (inference only in v1).

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

1. Inference server runs Qwen3.6-27B at ≥120 tok/s sustained on Strix Halo + RTX 3090.
2. OpenAI-compatible API exposed on localhost (completions, chat, models endpoints).
3. Model pull, list, and delete via CLI with automatic quantization selection.
4. `lucebox check` reports hardware compatibility and specific deficiencies.
5. Harness adapters for claude-code, codex, opencode, hermes-agent, pi, and openclaw route to the local endpoint.
6. Software installs on any compatible Linux hardware (not locked to Lucebox boxes).

### Should Have (P1)

1. Anthropic Messages API-compatible endpoint for Claude harness compatibility.
2. 128K context window support on Qwen3.6-27B.
3. Active model switching via CLI without server restart.
4. Hardware status and inference metrics via CLI (tokens/sec, VRAM usage, queue depth).
5. Inference server container starts on boot via a `lucebox.service` systemd unit managed by `lucebox.sh`.

### Nice to Have (P2)

1. DFlash speculative decoding for higher peak throughput.
2. Multiple concurrent models (memory-permitting).
3. `lucebox bench` command for reproducible throughput benchmarks.
4. Shell completions for the `lucebox` CLI.

## Functional Requirements

### Subsystem: Inference Server

- **FR-1** — `lucebox serve` starts the inference server container and listens on a configurable local port; exits cleanly on SIGTERM.
- **FR-2** — Inference throughput on Qwen3.6-27B Q4_K_M on the Strix Halo + RTX 3090 profile reaches ≥120 tok/s sustained on single-request load.
- **FR-3** — Server exposes `/v1/completions`, `/v1/chat/completions`, and `/v1/models` endpoints conforming to OpenAI API schema v1.
- **FR-4** — Server handles context windows up to 128K tokens for Qwen3.6-27B without OOM on the launch hardware profile.
- **FR-5** — Excess inference requests are queued (not dropped); queue depth is visible via CLI.

### Subsystem: Model Management

- **FR-6** — `lucebox model download <model>` downloads the named model from the Lucebox-maintained registry.
- **FR-7** — At download time, the server selects the highest-fidelity quantization that fits in available VRAM + RAM without user input.
- **FR-8** — `lucebox model list` lists installed models, their quantization, and disk usage.
- **FR-9** — `lucebox model remove <model>` removes a model and frees its storage.
- **FR-10** — `lucebox model activate <model>` switches the active model; in-flight requests complete on the prior model without interruption.

### Subsystem: Hardware Compatibility

- **FR-11** — `lucebox check` runs before installation and reports pass/fail for each requirement: Docker daemon reachable, NVIDIA driver ≥r525, NVIDIA Container Toolkit registered, GPU VRAM adequate, system RAM adequate, systemd present.
- **FR-12** — Each failed check names the specific deficiency and links to a documented fix (driver install, CTK registration, or a statement that the hardware is not supported).
- **FR-13** — On the Strix Halo + RTX 3090 reference profile, all checks pass and throughput estimates are printed.

### Subsystem: Agentic Harness Wrappers

- **FR-14** — Each harness adapter is a `lucebox` subcommand (`lucebox claude-code`, `lucebox codex`, `lucebox opencode`, `lucebox hermes-agent`, `lucebox pi`, `lucebox openclaw`) that configures and execs the real client binary pointed at the local inference endpoint.
- **FR-15** — Harness adapters require no cloud API key; the local endpoint accepts the default key `sk-lucebox`.
- **FR-16** — The `claude-code` adapter exposes an Anthropic Messages API-compatible endpoint so the Claude Code CLI routes without code changes.
- **FR-17** — Harness adapters are stateless; invoking them multiple times does not corrupt the underlying client tool's configuration.

### Subsystem: Software Distribution

- **FR-18** — Lucebox software installs on Linux x86_64 systems that pass `lucebox check`; it is not hardware-locked to Lucebox-branded machines.
- **FR-19** — Installation from a clean OS completes in ≤3 commands: `curl -fsSL https://raw.githubusercontent.com/Luce-Org/lucebox-hub/main/install.sh | bash` bootstraps `lucebox.sh` and the `lucebox` Python package.
- **FR-20** — A `lucebox.service` systemd unit starts the inference container on boot; `lucebox.sh` manages the Docker container lifecycle.

## Acceptance Test Sketches

| Requirement | Scenario | Input | Expected Output |
|-------------|----------|-------|-----------------|
| FR-2 | Throughput on reference hardware | `lucebox bench qwen3.6-27b` on Strix Halo + RTX 3090 | ≥120 tok/s reported |
| FR-3 | OpenAI API surface | `curl localhost:<port>/v1/models` after `lucebox serve` | JSON array listing active model |
| FR-7 | Auto-quantization | `lucebox model download qwen3.6-27b` on machine with 24GB VRAM + 128GB RAM | Q4_K_M selected automatically; confirmed in `lucebox model list` |
| FR-11 | Compatibility check failure | `lucebox check` on machine with RTX 2060 8GB | "GPU VRAM insufficient: 8 GB found, 24 GB required" |
| FR-14 | Harness adapter invocation | `lucebox claude-code "explain this code"` | Completion served from local Lucebox server, no API key prompt |
| FR-16 | Anthropic endpoint compat | Anthropic Messages API POST to local endpoint | Valid response in Anthropic response schema |

## Technical Context

- **Language/Runtime**: Inference engine: C++17 with CUDA 12 / HIP 7 (custom kernels — DFlash, PFlash, megakernel); CLI and harness adapters: Python/uv (Typer-based `lucebox` package + `harness/` package); system orchestration: POSIX shell (`lucebox.sh`). See ADR-002.
- **Key Libraries**: Custom CUDA/HIP kernels; GGUF loader; uv / Typer for CLI; pytest for tests.
- **APIs**: OpenAI Chat Completions API v1; Anthropic Messages API (for `claude` harness adapter).
- **Deployment**: Inference server runs in a Docker container (`ghcr.io/luce-org/lucebox-hub:cuda12`); `lucebox.sh` manages container lifecycle; systemd manages the Docker daemon and `lucebox.service`. See ADR-003.
- **Platform Targets**: Linux x86_64 at launch; kernel ≥5.15; CUDA 12.x for RTX 3090; AMD ROCm / AMDGPU for Strix Halo iGPU.
- **Hardware Profiles**: Strix Halo (AMD Ryzen AI MAX+ 395, 128GB LPDDR5X) + RTX 3090 (24GB GDDR6X) — single launch profile.
- **Model Format**: GGUF. Any GGUF-compatible model installs via `lucebox model download` or manual placement.

Stack selection rationale: ADR-001 (license), ADR-002 (language stack), ADR-003 (Docker deployment).

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

- [ ] What is the minimum compatible hardware spec for non-Lucebox installs (minimum VRAM, RAM, PCIe gen)? — defer to `lucebox check` for compatibility determination; exact non-Lucebox floor TBD — blocks FR-11, FR-18.
- [ ] Which Linux distributions are officially supported at launch (Ubuntu 22.04/24.04, Fedora, Arch)? — blocks FR-18, FR-19.
- [ ] Is the Anthropic Messages API wrapper officially supported or community-maintained? — blocks FR-16 support posture.
- [x] Software license: Apache 2.0. See ADR-001.
- [x] Distribution mechanism: `curl -fsSL https://raw.githubusercontent.com/Luce-Org/lucebox-hub/main/install.sh | bash`. See FR-19.

## Success Criteria

- A developer on a fresh Linux install with Strix Halo + RTX 3090 hardware runs `lucebox check` (all checks pass), installs Lucebox, downloads Qwen3.6-27B, and routes at least one agentic coding harness to local inference — all within 15 minutes.
- `lucebox bench` on the reference hardware reports ≥120 tok/s sustained on Qwen3.6-27B Q4_K_M before the first hardware batch ships.
- `lucebox check` on a machine with known-incompatible hardware (e.g., 8GB VRAM GPU) names the specific failing check with a link to a fix.
- Zero harness adapters require a cloud API key to function against the local Lucebox endpoint.
