---
ddx:
  id: FEAT-003
  status: draft
  links:
    - id: prd
      rel: informs
---

# Feature Specification: FEAT-003 — Installation and Tuning

**Feature ID**: FEAT-003
**Status**: Draft
**Priority**: P0
**Owner**: Platform Team
**Covered PRD Subsystem(s)**: Hardware Compatibility; Software Distribution
**Covered PRD Requirements**: FEAT-003 (Installation and Tuning)
**Cross-Subsystem Rationale**: Installation and hardware compatibility are the same user journey — a user who fails host compatibility checking cannot install, and a user who installs without checking will hit the same failures later. Splitting these would produce two features neither of which can ship independently.

## Overview

This feature covers the complete journey from bare hardware to a tuned, running inference server: validating host prerequisites, installing the Lucebox software stack, pulling the inference container image, registering the boot service, and running autotune to write an optimized configuration. The journey is intentionally unified — install and tune are sequential steps of one capability, not two.

## Ideal Future State

A developer on compatible Linux hardware opens a terminal and runs a single curl command. The installer validates prerequisites, installs the CLI and Python package, and exits with a clear next-step prompt. The developer then pulls the inference container and runs autotune, which probes the hardware, selects the highest-fidelity quantization the VRAM can sustain, and writes a configuration file. From that point forward, the server starts on every boot without manual intervention.

When this feature is working well:

- Any developer who passes host compatibility checking can complete the entire install-and-tune journey in under 15 minutes without reading documentation beyond the inline prompts.
- Host compatibility is checked first in every install workflow — users on incompatible hardware receive a clear diagnosis with a named fix path, not a silent failure later during inference.
- Autotune eliminates manual quantization and context-window decisions. The developer does not need to understand GGUF quantization levels to get optimal performance.
- The inference container starts automatically on reboot; the developer never needs to manually start the inference server.
- Configuration is transparent: the written `config.toml` is human-readable, and env vars can override any value without touching the file.

## Problem Statement

- **Current situation**: Installing a performant local LLM stack on Linux requires manual driver validation, container toolkit setup, quantization selection, context-window sizing, and service registration — typically across multiple tools with conflicting documentation.
- **Pain points**: Developers spend 2–8 hours per install or driver-update cycle debugging quantization mismatches, VRAM overflow, and silent API incompatibilities. There is no guided path from bare hardware to a tuned, running server. Wrong quantization choices silently degrade throughput; VRAM overflow causes OOM crashes with no actionable error. There is no hardware gate preventing installation on unsupported configurations.
- **Desired outcome**: A developer on compatible hardware completes the install-and-tune journey — from host compatibility verification to a tuned, boot-persistent inference server — in ≤15 minutes with zero manual quantization decisions. Incompatible hardware is diagnosed before any install step proceeds.

## Functional Areas

| Area | User question or job | Feature responsibility |
|------|----------------------|------------------------|
| Host Check | Does this hardware support Lucebox? What specifically is wrong if not? | Report pass/fail for all 6 prerequisites; name each deficiency with a fix path |
| Bootstrap Install | How do I get the Lucebox CLI on this machine? | Single curl command installs the shell wrapper and the `lucebox` Python package |
| Docker Setup | How do I get the inference container? | The container image pull fetches the `ghcr.io/luce-org/lucebox-hub:cuda12` image |
| Boot Service | How do I make the server start automatically? | A systemd unit starts the container on boot; the shell wrapper manages lifecycle |
| Autotune | What is the right configuration for this hardware? | Empirical VRAM-tier bracket sweep writes optimal quantization, context window, draft model, and sampling defaults to `config.toml` |

## Requirements

### Functional Requirements by Area

#### Host Check

**CHK-01**. Host compatibility is checked before installation, evaluating all six prerequisites and reporting a pass/fail result for each: Docker daemon reachable, NVIDIA driver version ≥ r525, NVIDIA Container Toolkit (CTK) registered, GPU VRAM adequate for the target model (minimum 24 GB for Qwen3.6-27B Q4_K_M), system RAM adequate, and systemd present.

**CHK-02**. Each failed check names the specific deficiency in human-readable form (e.g., "NVIDIA driver r520 found; r525 or later required") and references a documented fix path.

**CHK-03**. On the Strix Halo + RTX 3090 reference profile, all six checks pass and a throughput estimate for the detected hardware configuration is printed.

**CHK-04**. The host compatibility check is idempotent and non-destructive; it makes no changes to the host.

**CHK-05**. The host compatibility check exits with a non-zero status code when any check fails, enabling use in scripted install pipelines.

#### Bootstrap Install

**INS-01**. The bootstrap installer completes the full bootstrap on a host that passes all host compatibility checks, triggered by a single curl command.

**INS-02**. The installer places the shell wrapper script at `$HOME/.local/bin/lucebox` and ensures it is executable.

**INS-03**. The installer installs the `lucebox` Python package using uv as the package manager.

**INS-04**. The installer sets `LUCEBOX_INSTALL_URL` to the install script URL so future self-update operations can locate the canonical distribution point.

**INS-05**. The installer reports each step as it runs and exits with a non-zero status and a human-readable error message on failure.

**INS-06**. On a host that passes all compatibility checks, the full install-to-running-server journey completes in ≤3 steps.

#### Docker Setup

**DCK-01**. The inference container image `ghcr.io/luce-org/lucebox-hub:cuda12` is fetched from the registry as a distinct installation step.

**DCK-02**. NVIDIA Container Toolkit provides GPU passthrough from the host to the container; the image does not bundle drivers.

**DCK-03**. The container image pull reports download progress and exits with a non-zero status if the pull fails.

**DCK-04**. Pulling the container image when it is already present and current is a no-op; it does not re-download the image unnecessarily.

#### Boot Service

**SVC-01**. A systemd unit is registered during install and configured to start the inference container on system boot.

**SVC-02**. The shell wrapper script manages the Docker container lifecycle (start, stop, restart) for the inference server; the systemd unit delegates to it.

**SVC-03**. The systemd unit declares the correct dependency ordering so it starts after the Docker daemon is running.

**SVC-04**. The boot service uses the active configuration from `config.toml` (subject to env var overrides) at start time.

**SVC-05**. The inference server started by the boot service is indistinguishable to callers from one started manually.

#### Autotune

**TUN-01**. Autotune runs an empirical VRAM-tier bracket sweep across supported quantization levels (Q4_K_M, Q5_K_M, Q8_0, F16) and identifies the highest-fidelity quantization that sustains throughput at or above the target threshold on the detected hardware.

**TUN-02**. Autotune selects a draft model for speculative decoding based on available VRAM after the primary model is allocated.

**TUN-03**. Autotune determines the maximum context window size the hardware can support up to 128K tokens.

**TUN-04**. Autotune writes all tuned values — quantization selection, draft model, context window size, and sampling parameter defaults (temperature, top_p, and equivalent parameters) — to `config.toml`.

**TUN-05**. Configuration precedence is: environment variables override `config.toml` values, which override dataclass defaults. All three layers are always consulted in this order.

**TUN-06**. The following configuration keys are recognized and documented: `LUCEBOX_IMAGE`, `LUCEBOX_VARIANT`, `LUCEBOX_PORT`, `LUCEBOX_CONTAINER`, `LUCEBOX_MODELS`.

**TUN-07**. Autotune can be re-run after a hardware change (e.g., VRAM upgrade) and overwrites the prior `config.toml` with updated values.

**TUN-08**. Autotune reports the selected configuration values and the rationale for each selection (e.g., "Q5_K_M selected: fits in 24 GB VRAM with 2 GB headroom") before writing.

### Non-Functional Requirements

- **Performance**: The host compatibility check completes in ≤10 seconds on any host. The autotune bracket sweep completes in ≤5 minutes on the reference hardware profile.
- **Reliability**: The bootstrap installer is idempotent — running it a second time on an already-installed host produces a valid, consistent install state without errors.
- **Compatibility**: The install path supports Linux x86_64 with kernel ≥5.15. No Windows or macOS support at launch.
- **Observability**: Every install, pull, and autotune step logs its action and outcome to stdout. Silent failures are not permitted.
- **Security**: The installer does not request or store cloud API credentials. `LUCEBOX_INSTALL_URL` is the only network-sourced configuration written to the host by the installer.

## User Stories

- [TODO: create story] US-XXX — Developer checks host compatibility on a compatible machine and all checks pass
- [TODO: create story] US-XXX — Developer checks host compatibility on an incompatible machine and receives a named diagnosis
- [TODO: create story] US-XXX — Developer bootstraps Lucebox on a fresh Linux install using the curl command
- [TODO: create story] US-XXX — Developer fetches the inference container image
- [TODO: create story] US-XXX — Developer reboots the machine and the inference server starts automatically via the boot service
- [TODO: create story] US-XXX — Developer runs autotune and receives a written `config.toml` with selected quantization and context window
- [TODO: create story] US-XXX — Developer re-runs autotune after upgrading GPU and gets an updated configuration

## Edge Cases and Error Handling

- **Check fails mid-install**: If a compatibility check fails during an install script, the installer exits immediately with the failing check named. Partial install state is not left on the host.
- **Docker daemon not running at check time**: The host compatibility check reports "Docker daemon not reachable" as a named failure, not a generic connection error. It does not attempt to start the daemon.
- **VRAM exactly at the minimum threshold (24 GB)**: The host compatibility check reports a pass with a warning that no headroom exists for a draft model; autotune skips draft model selection in this case.
- **Container image pull interrupted mid-download**: A subsequent pull resumes or restarts the download cleanly; a partial image layer does not leave the local registry in an unusable state.
- **`config.toml` missing when boot service starts**: The boot service falls back to dataclass defaults and logs a warning naming each defaulted value; it does not refuse to start.
- **Autotune run before the container image is present**: Autotune reports that the container image is not present and instructs the user to pull it first; it does not proceed with the sweep.
- **Env var conflicts with `config.toml`**: The active env var value takes precedence and a warning is emitted naming the overridden key and its `config.toml` value.
- **systemd not present**: The host compatibility check reports "systemd not found" as a named failure; the boot service area is unavailable on non-systemd hosts.

## Success Metrics

- Time from running the bootstrap curl command to a tuned, boot-persistent inference server on the reference hardware profile: ≤15 minutes.
- Host compatibility checking correctly identifies all six prerequisites on a known-incompatible host (e.g., driver below r525, VRAM below 24 GB): ≥80% of known-incompatible configurations name the specific failing check.
- Autotune selects the correct highest-fidelity quantization on the reference hardware profile (Q5_K_M or better on 24 GB VRAM) in 100% of runs on reference hardware.
- The install path is idempotent: re-running the bootstrap on an already-installed host exits without error in 100% of test runs.
- The boot service starts the inference server within 60 seconds of system boot on the reference hardware profile.

## Constraints and Assumptions

- Linux x86_64 with kernel ≥5.15 is the only supported platform at launch. Windows and macOS are out of scope.
- Docker daemon and NVIDIA Container Toolkit must be present on the host before compatibility checking can pass; the installer does not install them.
- The minimum GPU VRAM required to pass compatibility checking for Qwen3.6-27B Q4_K_M is 24 GB. The minimum system RAM floor is TBD pending FR-11 resolution.
- The inference container image `ghcr.io/luce-org/lucebox-hub:cuda12` is approximately 14 GB; the install path assumes sufficient disk space is available.
- Autotune writes to `config.toml` in a well-known location; the exact path is an implementation detail resolved in the technical design.
- `LUCEBOX_INSTALL_URL` is the canonical mechanism for self-updates; the exact update workflow is not part of this feature.
- The autotune throughput threshold used to select quantization is the PRD target of ≥120 tok/s on the reference hardware profile.

## Dependencies

- **Other features**: FEAT-004 (Model Management) — day-to-day model pull, list, and delete are out of scope here; the container image pull in this feature fetches the runtime image, not a model. FEAT-005 (Harness Setup) — harness adapter configuration begins after the server is running, which this feature delivers.
- **External systems**: GitHub Container Registry (`ghcr.io/luce-org/lucebox-hub`) for the container image; GitHub raw content (`raw.githubusercontent.com/Luce-Org/lucebox-hub/main/install.sh`) for the bootstrap script.
- **Host prerequisites**: Docker daemon, NVIDIA driver ≥ r525, NVIDIA Container Toolkit, systemd — all must be present on the host before install proceeds; none are installed by this feature.
- **PRD requirements**: FEAT-003 (Installation and Tuning) — host compatibility checking, software distribution, Docker setup, boot service, and autotune.

## Out of Scope

- Day-to-day model management (download, list, remove, activate) — covered by FEAT-004.
- Agentic harness adapter configuration and per-tool endpoint routing — covered by FEAT-005.
- Installing or upgrading the Docker daemon, NVIDIA driver, or NVIDIA Container Toolkit. This feature gates on their presence; it does not provision them.
- Windows and macOS install paths.
- Multi-box or distributed install configurations.
- Web UI or GUI for configuration or status — covered by FEAT-005.
- Fine-tuning pipelines or training workflows.
- Reproducible benchmarking (P2 nice-to-have, not part of the install-and-tune journey).
- Uninstall or removal workflow.
- The self-update workflow beyond storing `LUCEBOX_INSTALL_URL`.
