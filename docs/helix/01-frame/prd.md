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
tuned inference server with model management and thin wrapper scripts that
route popular agentic coding tools (Claude, Codex, OpenCode, Charm) to the
local endpoint. This eliminates cloud API dependency and removes setup churn
for the technical users who already run local LLMs but can't get reliable
performance from generic stacks.

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
2. Switching an existing agentic coding tool (Claude, Codex, OpenCode, Charm)
   to local inference requires one command after the server is running.
3. Non-Lucebox hardware users can self-qualify via a compatibility check and
   install the same software stack.

### Success Metrics

| Metric | Target | Measurement Method |
|--------|--------|--------------------|
| Inference throughput (Qwen3.6-27B Q4_K_M, single-request, launch HW) | ≥120 tok/s sustained | `luce bench` output on reference machine |
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
4. `luce check` reports hardware compatibility and specific deficiencies.
5. Harness wrappers for Claude, Codex, OpenCode, and Charm route to local endpoint.
6. Software installs on any compatible Linux hardware (not locked to Lucebox boxes).

### Should Have (P1)

1. Anthropic Messages API-compatible endpoint for Claude harness compatibility.
2. 128K context window support on Qwen3.6-27B.
3. Active model switching via CLI without server restart.
4. Hardware status and inference metrics via CLI (tokens/sec, VRAM usage, queue depth).
5. Server runs as a systemd service and starts on boot.

### Nice to Have (P2)

1. DFlash speculative decoding for higher peak throughput.
2. Multiple concurrent models (memory-permitting).
3. `luce bench` command for reproducible throughput benchmarks.
4. Shell completions for `luce` CLI.

## Functional Requirements

### Subsystem: Inference Server

- **FR-1** — Server starts and listens on a configurable local port via `luce start`; exits cleanly on SIGTERM.
- **FR-2** — Inference throughput on Qwen3.6-27B Q4_K_M on the Strix Halo + RTX 3090 profile reaches ≥120 tok/s sustained on single-request load.
- **FR-3** — Server exposes `/v1/completions`, `/v1/chat/completions`, and `/v1/models` endpoints conforming to OpenAI API schema v1.
- **FR-4** — Server handles context windows up to 128K tokens for Qwen3.6-27B without OOM on the launch hardware profile.
- **FR-5** — Excess inference requests are queued (not dropped); queue depth is visible via CLI.

### Subsystem: Model Management

- **FR-6** — `luce pull <model>` downloads the named model from the Lucebox-maintained registry.
- **FR-7** — At pull time, the server selects the highest-fidelity quantization that fits in available VRAM + RAM without user input.
- **FR-8** — `luce list` shows installed models, their quantization, and disk usage.
- **FR-9** — `luce rm <model>` removes a model and frees its storage.
- **FR-10** — `luce use <model>` switches the active model; inference continues uninterrupted for in-flight requests.

### Subsystem: Hardware Compatibility

- **FR-11** — `luce check` runs before installation and reports pass/fail for each hardware requirement (GPU VRAM, system RAM, OS, driver version, PCIe bandwidth where relevant).
- **FR-12** — Each failed check names the specific deficiency and links to a documented fix (driver update, BIOS setting, or a statement that the hardware is not supported).
- **FR-13** — On the Strix Halo + RTX 3090 reference profile, all checks pass and throughput estimates are printed.

### Subsystem: Agentic Harness Wrappers

- **FR-14** — `luce install-harness <name>` installs the wrapper for the named tool (claude, codex, opencode, charm); the tool subsequently routes inference to the local server.
- **FR-15** — Installed wrappers require no cloud API key for the Lucebox inference path.
- **FR-16** — The Claude wrapper exposes an Anthropic Messages API-compatible endpoint so `claude` CLI routes without code changes.
- **FR-17** — Wrapper install is idempotent; re-running it on an already-configured tool does not break the existing setup.

### Subsystem: Software Distribution

- **FR-18** — Lucebox software installs on Linux x86_64 systems that pass `luce check`; it is not hardware-locked to Lucebox-branded machines.
- **FR-19** — Installation from a clean OS completes in ≤3 commands (e.g., `brew install lucebox` or equivalent package manager).
- **FR-20** — The server registers as a systemd service and starts automatically on boot after the first `luce start`.

## Acceptance Test Sketches

| Requirement | Scenario | Input | Expected Output |
|-------------|----------|-------|-----------------|
| FR-2 | Throughput on reference hardware | `luce bench qwen3.6-27b` on Strix Halo + RTX 3090 | ≥120 tok/s reported |
| FR-3 | OpenAI API surface | `curl localhost:<port>/v1/models` after `luce start` | JSON array listing active model |
| FR-7 | Auto-quantization | `luce pull qwen3.6-27b` on machine with 24GB VRAM + 128GB RAM | Q4_K_M selected automatically; confirmed in `luce list` |
| FR-11 | Compatibility check failure | `luce check` on machine with RTX 2060 8GB | "GPU VRAM insufficient: 8 GB found, 24 GB required" |
| FR-14 | Harness wrapper install | `luce install-harness claude` then `claude "explain this code"` | Completion served from local Lucebox server, no API key prompt |
| FR-16 | Anthropic endpoint compat | Anthropic Messages API POST to local endpoint | Valid response in Anthropic response schema |

## Technical Context

- **Language/Runtime**: Inference core: C++/CUDA (custom kernels); CLI and tooling: TypeScript/Bun (per concerns.md `language-runtime` slot); harness wrappers: shell scripts or thin Bun scripts.
- **Key Libraries**: Custom CUDA kernels (DFlash, PFlash, megakernel for hybrid LLMs); llama.cpp or equivalent GGUF loader; Bun for CLI tooling.
- **APIs**: OpenAI Chat Completions API v1; Anthropic Messages API (for Claude harness wrapper).
- **Platform Targets**: Linux x86_64 at launch; kernel ≥5.15; CUDA 12.x for RTX 3090; AMD ROCm or AMDGPU driver for Strix Halo iGPU.
- **Hardware Profiles**: Strix Halo (AMD Ryzen AI MAX+ 395, 128GB LPDDR5X) + RTX 3090 (24GB GDDR6X) — single launch profile.
- **Model Format**: GGUF. Any GGUF-compatible model installs via `luce pull` or manual placement.

Stack selection rationale belongs in ADRs; none exist yet — see Open Questions.

## Constraints, Assumptions, Dependencies

### Constraints

- **Technical**: Linux x86_64 only at launch; CUDA 12.x required for RTX 3090 path; hybrid VRAM + unified memory routing is complex and requires validated driver combinations.
- **Business**: First hardware batch ships July 2026; software must be stable before units leave the warehouse.
- **Legal/Compliance**: Bundled open-weight models must have licenses permitting commercial distribution; no PII stored by the inference server.

### Assumptions

- Strix Halo + RTX 3090 hardware profile delivers ≥120 tok/s on Qwen3.6-27B Q4_K_M (based on ~130 tok/s sustained claim on lucebox.com for Qwen3.5-27B; Qwen3.6-27B assumed comparable).
- Claude, Codex, OpenCode, and Charm all support configurable inference endpoints at launch.
- Qwen3.6-27B remains a competitive default model at the time of hardware shipment.
- `luce` CLI tooling already substantially exists per current development state.

### Dependencies

- NVIDIA CUDA 12.x driver stability on Linux for the RTX 3090.
- AMD AMDGPU/ROCm driver stability for Strix Halo unified memory path.
- Upstream tool CLIs (claude, codex, opencode, charm) maintaining configurable endpoint support.

## Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Driver regression breaks hybrid GPU routing | Med | High | Pin tested driver versions per release; publish tested driver matrix; `luce check` validates driver version |
| Claude/Codex/OpenCode change their endpoint config interface | Med | Med | Wrapper scripts versioned and pinned; maintain compatibility shims; alert users via release notes |
| Qwen3.6-27B superseded before hardware ships, weakening default model story | Low | Med | Model registry is updatable post-install; ship at least two validated models so default can be swapped |
| Compatible-hardware install path creates unsupported-config support burden | Med | Med | `luce check` gates install; publish a supported hardware list; community tier users self-support |
| July batch sells out before software is stable | Low | High | Software gates hardware fulfillment; no units ship without passing the benchmark suite on reference hardware |

## Open Questions

- [ ] What is the minimum compatible hardware spec for non-Lucebox installs (minimum VRAM, RAM, PCIe gen)? — blocks FR-11, FR-18 — ask Erik/hardware team.
- [ ] Which Linux distributions are officially supported at launch (Ubuntu 22.04/24.04, Fedora, Arch)? — blocks FR-18, FR-19.
- [ ] Is the Anthropic Messages API wrapper officially supported or community-maintained? — blocks FR-16 support posture.
- [ ] What is the software license (Apache 2.0, BSL, proprietary)? — blocks distribution and community-tier decisions.
- [ ] Which specific Charm tool is targeted by the harness wrapper (mods, gum, other)? — blocks FR-14 for Charm.
- [ ] Will `luce` be distributed via Homebrew, apt, or a custom install script at launch? — blocks FR-19.

## Success Criteria

- A developer on a fresh Linux install with Strix Halo + RTX 3090 hardware runs `luce check` (all checks pass), installs Lucebox, pulls Qwen3.6-27B, and routes at least one agentic coding harness to local inference — all within 15 minutes.
- `luce bench` on the reference hardware reports ≥120 tok/s sustained on Qwen3.6-27B Q4_K_M before the first hardware batch ships.
- `luce check` on a machine with known-incompatible hardware (e.g., 8GB VRAM GPU) names the specific failing check with a link to a fix.
- Zero harness wrappers require a cloud API key to function against the local Lucebox endpoint.
