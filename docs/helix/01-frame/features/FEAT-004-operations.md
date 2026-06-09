---
ddx:
  id: FEAT-004
  status: draft
  links:
    - id: prd
      rel: informs
    - id: FEAT-002
      rel: depends-on
    - id: FEAT-003
      rel: depends-on
---

# Feature Specification: FEAT-004 — Operations

**Feature ID**: FEAT-004
**Status**: Draft
**Priority**: P0
**Owner**: Platform Team
**Covered PRD Subsystem(s)**: Model Management; Inference Server (operational surface)
**Covered PRD Requirements**: FR-1, FR-6, FR-7, FR-8, FR-9, FR-10, P1-3, P1-4, P2-3, P2-4
**Cross-Subsystem Rationale**: Model management and server lifecycle are distinct PRD subsystems, but both are runtime operator actions issued through the same CLI surface. The runtime management workflow — start server, pull model, activate model, check metrics — is the capability; splitting it would produce two features neither of which represents a complete operator workflow.

## Overview

FEAT-004 defines the day-to-day runtime management surface of a running Lucebox system. It covers everything an operator does after the system is installed (FEAT-003) and the inference engine is available (FEAT-002): starting and stopping the server, pulling and activating models, inspecting hardware and inference metrics, running verification and benchmarks, managing configuration, and keeping software current. These operations are the primary interface between an operator and a live Lucebox deployment and are therefore P0.

## Ideal Future State

An operator managing a Lucebox deployment issues a small set of `lucebox` subcommands to perform any routine task without consulting documentation. Starting the server, switching models, checking throughput, and updating software are each single-command actions with clear, machine-readable output. Model selection is automatic — the system chooses the best-fit quantization without prompting. Metrics are visible at any time from the CLI. Configuration changes are explicit and discoverable through a consistent dotted-key interface. The operator has confidence that the system is healthy because smoke tests and benchmarks are reproducible and exit with a clear pass or fail.

## Problem Statement

- **Current situation**: Lucebox ships a working inference server and model management tooling, but the operational surface — how an operator starts/stops the server, manages models across sessions, checks system health, and updates software — is not governed by a single coherent feature spec. Individual commands exist but their contracts, error behaviors, and interactions are not formally specified.
- **Pain points**: Without a governed operational surface, operators encounter undocumented failure modes (e.g., model switch behavior under load, VRAM overcommit during download), inconsistent CLI output formats across subcommands, and no authoritative reference for what the update path looks like.
- **Desired outcome**: Every operator action listed in this spec has a documented command, a defined success output, and a defined error output. Operators on the reference hardware can complete any routine operational task — including a model switch under load — without ambiguity.

## Functional Areas

| Area | User question or job | Feature responsibility |
|------|----------------------|------------------------|
| Server Lifecycle | Is the server running? How do I start or stop it? | Start, stop, and restart the inference server container; surface server status |
| Model Management | Which models do I have? How do I add, remove, or switch the active model? | Download, list, remove, and activate GGUF models; auto-select quantization at download time |
| Observability | How is the system performing right now? | Expose tokens/sec, VRAM usage, and queue depth via CLI from the inference server's `/props` endpoint |
| Benchmarking and Verification | Is the system behaving correctly? What throughput am I getting? | Smoke-test the API endpoint; run a throughput benchmark; capture a performance profile snapshot |
| Configuration Management | How do I read or change a config value? | Get, set, and unset values in config.toml via dotted-key notation; surface the active config layer |
| Updates | How do I get the latest server image or CLI version? | Pull a new Docker image; update the lucebox Python package |

## Requirements

### Functional Requirements by Area

#### Server Lifecycle

- **SRV-01**. `lucebox serve` starts the inference server container and the server is reachable on the configured local port within 30 seconds on reference hardware.
- **SRV-02**. `lucebox serve` is idempotent: if the container is already running, the command reports the running state and exits without error.
- **SRV-03**. The server container stops cleanly on SIGTERM; in-flight requests receive a response or a well-formed error before the process exits.
- **SRV-04**. `lucebox serve` reports a structured error with a specific cause when the container fails to start (port conflict, Docker daemon not reachable, missing model).

#### Model Management

- **MDL-01**. `lucebox model download <model>` downloads the named GGUF model from the Lucebox-maintained registry and places it in the configured models directory.
- **MDL-02**. At download time, the system selects the highest-fidelity quantization that fits within the available VRAM plus system RAM without user input; the selected quantization is displayed in the download confirmation and in `lucebox model list`.
- **MDL-03**. `lucebox model download` reports a structured error when available VRAM plus RAM is insufficient for any supported quantization of the requested model.
- **MDL-04**. `lucebox model list` lists every installed model with its name, selected quantization, and disk usage; output is human-readable and machine-parseable (JSON flag supported).
- **MDL-05**. `lucebox model remove <model>` removes the named model and all associated files; disk space is freed and the model no longer appears in `lucebox model list`.
- **MDL-06**. `lucebox model remove` reports a structured error when the named model is currently active and the `--force` flag is not passed.
- **MDL-07**. `lucebox model activate <model>` switches the inference server to serve the named model; the switch completes without restarting the server process.
- **MDL-08**. During an active model switch, in-flight requests on the prior model receive a complete response before the switch takes effect.
- **MDL-09**. `lucebox model activate` reports a structured error when the named model is not installed or when the server is not running.

#### Observability

- **OBS-01**. `lucebox status` reports current tokens/sec, VRAM usage, and queue depth sourced from the inference server's `/props` endpoint.
- **OBS-02**. `lucebox status` reports a structured error when the server is not reachable, distinguishing "server not started" from "server unreachable".
- **OBS-03**. `lucebox status` output includes the name and quantization of the currently active model.
- **OBS-04**. All observability output is machine-parseable (JSON flag supported).

#### Benchmarking and Verification

- **BNV-01**. `lucebox smoke` sends a minimal completion request to the configured local endpoint and exits 0 on a valid response or exits non-zero with a structured error on any failure.
- **BNV-02**. `lucebox profile` captures a benchmark snapshot via luce-bench integration and writes the result to a timestamped output file in the configured output directory.
- **BNV-03**. `lucebox bench` runs a reproducible throughput benchmark against the active model on the running server and reports sustained tokens/sec.
- **BNV-04**. On the Strix Halo + RTX 3090 reference hardware with Qwen3.6-27B Q4_K_M active, `lucebox bench` reports ≥120 tok/s sustained.
- **BNV-05**. `lucebox bench` output includes the model name, quantization, hardware profile identifier, and timestamp so results are comparable across runs.

#### Configuration Management

- **CFG-01**. `lucebox config get <key>` returns the effective value for a dotted configuration key, reflecting the active layer: environment variable override, then config.toml, then dataclass default.
- **CFG-02**. `lucebox config set <key> <value>` writes the value to config.toml; the change takes effect on the next server start without requiring a full reinstall.
- **CFG-03**. `lucebox config unset <key>` removes the key from config.toml, reverting to the dataclass default on the next server start.
- **CFG-04**. `lucebox config get` reports which config layer supplied the value (env var, config.toml, or default).
- **CFG-05**. The following environment variables are recognized and override their config.toml equivalents: `LUCEBOX_IMAGE`, `LUCEBOX_VARIANT`, `LUCEBOX_PORT`, `LUCEBOX_CONTAINER`, `LUCEBOX_MODELS`.
- **CFG-06**. `lucebox config get <key>` reports a structured error for unrecognized keys.

#### Updates

- **UPD-01**. `lucebox pull` pulls the latest Docker image for the configured `LUCEBOX_IMAGE` tag from the registry and reports the digest of the newly pulled image.
- **UPD-02**. `lucebox pull` does not restart a running server; the operator must run `lucebox serve` after pulling to apply the update.
- **UPD-03**. `lucebox pull` reports the current and new image digests so the operator can confirm a new version was fetched.
- **UPD-04**. The lucebox Python package is updatable via the standard package manager (uv or pip) without reinstalling the Docker image.

### Non-Functional Requirements

- **Performance — server start**: `lucebox serve` reaches a healthy state (responds to `/v1/models`) within 30 seconds on the reference hardware profile.
- **Performance — model switch**: `lucebox model activate` completes the active model switch (new model serving requests) within 60 seconds on the reference hardware profile.
- **Performance — observability latency**: `lucebox status` returns within 2 seconds under normal server load.
- **Performance — smoke test**: `lucebox smoke` completes within 10 seconds on a healthy server.
- **Reliability — idempotency**: `lucebox serve`, `lucebox model download`, and `lucebox pull` are idempotent; re-invoking them in the same state produces the same result and exits 0.
- **Reliability — exit codes**: All `lucebox` subcommands exit 0 on success and exit non-zero on any error; the exit code is distinct between "resource not found" (exit 1) and "server not reachable" (exit 2).
- **Observability — machine-readable output**: `lucebox status`, `lucebox model list`, and `lucebox bench` support a `--json` flag; JSON output is stable across patch releases.
- **Security**: `lucebox config set` writes only to config.toml in the lucebox data directory; it does not write to system files, environment files, or files outside the lucebox data directory.

## User Stories

- [TODO: create story] — As Morgan, I want to start the inference server with a single command so that I can begin using my coding agent immediately after booting my machine.
- [TODO: create story] — As Morgan, I want to download a new model without choosing a quantization manually so that I can get a working setup without understanding VRAM math.
- [TODO: create story] — As Sam, I want to switch the active model without restarting the server so that I can compare models mid-session without losing my request queue.
- [TODO: create story] — As Sam, I want to see current tokens/sec and VRAM usage from the CLI so that I can confirm the server is performing within expected bounds.
- [TODO: create story] — As Sam, I want to run a reproducible throughput benchmark so that I can compare performance across software updates and model changes.
- [TODO: create story] — As Jordan, I want to read and write configuration values through the CLI so that I do not need to manually edit config files or know their locations.
- [TODO: create story] — As Jordan, I want to pull a new Docker image without the server restarting automatically so that I control when the update takes effect.
- [TODO: create story] — As Sam, I want to run a smoke test against the local endpoint so that I can confirm the server is accepting requests before pointing a harness at it.

## Edge Cases and Error Handling

- **Model activate while server is stopped**: `lucebox model activate` reports that the server is not running and exits non-zero; it does not silently update a config value without a running server to validate against.
- **VRAM insufficient at download time**: `lucebox model download` reports the minimum VRAM required for the lowest available quantization, the VRAM detected, and the shortfall; it does not download a partial file.
- **Port conflict on serve**: `lucebox serve` detects that the configured port is already bound, reports the conflicting process (if identifiable), and exits non-zero rather than silently using an alternate port.
- **Active model removed with --force**: `lucebox model remove --force` on the active model stops the server, removes the model, and reports that the server must be restarted with a different active model; it does not leave the server in a partially degraded state.
- **Config key set while server is running**: `lucebox config set` writes the value to config.toml and emits a warning that the running server will not reflect the change until the next `lucebox serve`.
- **Pull when no new image is available**: `lucebox pull` exits 0 and reports "already at latest" with the current image digest; it does not treat an up-to-date state as an error.
- **`lucebox status` when server is starting**: `lucebox status` distinguishes "server is starting" (container running, `/props` not yet responsive) from "server is not running" (container absent or exited).
- **Model download interrupted**: A partially downloaded model does not appear in `lucebox model list`; the partial data is cleaned up automatically.

## Success Metrics

- `lucebox serve` reaches healthy state within 30 seconds on reference hardware across 10 consecutive cold starts (zero failures).
- `lucebox model activate` completes an active model switch on a server with 10 in-flight requests without dropping any request (all requests return a valid response or well-formed error).
- `lucebox bench` on reference hardware with Qwen3.6-27B Q4_K_M reports ≥120 tok/s sustained in 5 consecutive runs with ≤5% variance between runs.
- `lucebox smoke` exits 0 on a healthy server 100% of the time across the reference hardware test matrix.
- All `lucebox` subcommands exit non-zero and print a structured error message (not a raw Python traceback) for every documented error condition.

## Constraints and Assumptions

- All operations assume the inference server and Docker runtime are available as specified in FEAT-002 and FEAT-003.
- `lucebox config` reads and writes only to config.toml in the lucebox data directory; it does not manage environment variables in the shell or in systemd unit files.
- Model auto-quantization selection is based on VRAM and RAM reported at download time; VRAM changes after download (e.g., another process consuming VRAM) are not re-evaluated by the CLI.
- `lucebox pull` requires network access to the Docker registry; air-gapped operation is not addressed in this feature.
- The `/props` endpoint contract is defined in FEAT-002; this feature consumes it but does not define it.
- Model switching without server restart requires inference server support for hot-swap; this capability is a FEAT-002 contract requirement, not implemented here.
- `lucebox bench` results are reproducible only under controlled conditions (no concurrent inference load, fixed hardware power state); the spec does not guarantee reproducibility across hardware profiles.

## Dependencies

- **FEAT-002**: Inference server must expose the `/props` endpoint (OBS-01, OBS-03) and support active model switching without restart (MDL-07, MDL-08).
- **FEAT-003**: Installation and autotune must have completed successfully before any operation in this spec is valid.
- **Docker daemon**: `lucebox serve`, `lucebox pull`, and server lifecycle operations require the Docker daemon to be running and the NVIDIA Container Toolkit to be registered.
- **Lucebox model registry**: `lucebox model download` depends on the Lucebox-maintained GGUF registry being reachable (or a local mirror for air-gapped setups).
- **luce-bench**: `lucebox profile` depends on the luce-bench integration being installed and callable.

## Out of Scope

- Initial software installation and hardware autotune — covered by FEAT-003.
- Inference server internals, kernel configuration, and VRAM allocation strategy — covered by FEAT-002.
- Harness adapter configuration and agentic tool routing — covered by FEAT-005.
- Web management UI — deferred to v2 per PRD Non-Goals.
- Multi-model concurrent serving — addressed as a P2 PRD item; not a requirement of the operational CLI surface in this spec.
- Air-gapped or offline model download — out of scope for v1.
- Windows and macOS support — out of scope per PRD Constraints.
- Fine-tuning, training, or model conversion operations — inference only in v1.
- Automated update scheduling or unattended upgrades — operators initiate all updates explicitly.
