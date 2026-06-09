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
**Covered PRD Requirements**: FEAT-004 (Operations)
**Cross-Subsystem Rationale**: Model management and server lifecycle are distinct PRD subsystems, but both are runtime operator actions issued through the same CLI surface. The runtime management workflow — start server, pull model, activate model, check metrics — is the capability; splitting it would produce two features neither of which represents a complete operator workflow.

## Overview

FEAT-004 defines the day-to-day runtime management surface of a running Lucebox system. It covers everything an operator does after the system is installed (FEAT-003) and the inference engine is available (FEAT-002): starting and stopping the server, pulling and activating models, inspecting hardware and inference metrics, running verification and benchmarks, managing configuration, and keeping software current. These operations are the primary interface between an operator and a live Lucebox deployment and are therefore P0.

## Ideal Future State

An operator managing a Lucebox deployment performs any routine task without consulting documentation. Starting the server, switching models, checking throughput, and updating software are each single-action operations with clear, machine-readable output. Model selection is automatic — the system chooses the best-fit quantization without prompting. Metrics are visible at any time from the CLI. Configuration changes are explicit and discoverable through a consistent dotted-key interface. The operator has confidence that the system is healthy because smoke tests and benchmarks are reproducible and exit with a clear pass or fail.

## Problem Statement

- **Current situation**: Lucebox ships a working inference server and model management tooling, but the operational surface — how an operator starts/stops the server, manages models across sessions, checks system health, and updates software — is not governed by a single coherent feature spec. Individual operations exist but their contracts, error behaviors, and interactions are not formally specified.
- **Pain points**: Without a governed operational surface, operators encounter undocumented failure modes (e.g., model switch behavior under load, VRAM overcommit during download), inconsistent CLI output formats across operations, and no authoritative reference for what the update path looks like.
- **Desired outcome**: Every operator action listed in this spec has a documented operation, a defined success output, and a defined error output. Operators on the reference hardware can complete any routine operational task — including a model switch under load — without ambiguity.

## Functional Areas

| Area | User question or job | Feature responsibility |
|------|----------------------|------------------------|
| Server Lifecycle | Is the server running? How do I start or stop it? | Start, stop, and restart the inference server container; surface server status |
| Model Management | Which models do I have? How do I add, remove, or switch the active model? | Download, list, remove, and activate GGUF models; auto-select quantization at download time |
| Observability | How is the system performing right now? | Expose tokens/sec, VRAM usage, and queue depth via CLI from the inference server's `/props` endpoint |
| Benchmarking and Verification | Is the system behaving correctly? What throughput am I getting? | Smoke-test the API endpoint (`lucebox smoke`); run a luce-bench throughput profile snapshot (`lucebox profile`) |
| Configuration Management | How do I read or change a config value? | Get, set, and unset values in config.toml via dotted-key notation; surface the active config layer |
| Updates | How do I get the latest server image or CLI version? | Pull a new Docker image; update the lucebox Python package |

## Requirements

### Functional Requirements by Area

#### Server Lifecycle

- **SRV-01**. The inference server container is started by the server lifecycle operation and the server is reachable on the configured local port within 30 seconds on reference hardware.
- **SRV-02**. Starting the server is idempotent: if the container is already running, the operation reports the running state and exits without error.
- **SRV-03**. The server container stops cleanly on SIGTERM; in-flight requests receive a response or a well-formed error before the process exits.
- **SRV-04**. The server start operation reports a structured error with a specific cause when the container fails to start (port conflict, Docker daemon not reachable, missing model).

#### Model Management

- **MDL-01**. Models are downloaded from the Lucebox-maintained registry on demand and placed in the configured models directory.
- **MDL-02**. At download time, the system selects the highest-fidelity quantization that fits within the available VRAM plus system RAM without user input; the selected quantization is displayed in the download confirmation and in the model list.
- **MDL-03**. Model download reports a structured error when available VRAM plus RAM is insufficient for any supported quantization of the requested model.
- **MDL-04**. The model list operation lists every installed model with its name, selected quantization, and disk usage; output is human-readable.
- **MDL-05**. Model removal deletes the named model and all associated files; disk space is freed and the model no longer appears in the model list.
- **MDL-06**. Model removal reports a structured error when the named model is currently active and removal is not forced.
- **MDL-07**. Model activation switches the inference server to serve the named model; the switch completes without restarting the server process.
- **MDL-08**. During an active model switch, in-flight requests on the prior model receive a complete response before the switch takes effect.
- **MDL-09**. Model activation reports a structured error when the named model is not installed or when the server is not running.

#### Observability

- **OBS-01**. The status operation reports current tokens/sec, VRAM usage, and queue depth sourced from the inference server's `/props` endpoint.
- **OBS-02**. The status operation reports a structured error when the server is not reachable, distinguishing "server not started" from "server unreachable".
- **OBS-03**. Status output includes the name and quantization of the currently active model.
- **OBS-04**. Status output is human-readable. Machine-parseable output is not implemented in the current CLI; see TD-005 open questions.

#### Benchmarking and Verification

- **BNV-01**. The smoke test operation sends a minimal completion request to the configured local endpoint and exits 0 on a valid response or exits non-zero with a structured error on any failure.
- **BNV-02**. The profile capture operation records a benchmark snapshot via luce-bench integration and writes the result to a timestamped output file in the configured output directory.
- **BNV-03**. The profile operation (`lucebox profile`) runs reproducibly against the active model on the running server via luce-bench and reports sustained tokens/sec.
- **BNV-04**. On the Strix Halo + RTX 3090 reference hardware with Qwen3.6-27B Q4_K_M active, `lucebox profile` reports results satisfying all three throughput targets: ≥120 tok/s short-context (T1), ≥60 tok/s at 128K multi-turn with prefix caching (T2), and ≥60 tok/s at 128K with PFlash compression (T3).
- **BNV-05**. Profile output includes the model name, quantization, hardware profile identifier, and timestamp so results are comparable across runs.

#### Configuration Management

- **CFG-01**. Reading a configuration key returns its effective value, reflecting the active layer: environment variable override, then config.toml, then dataclass default.
- **CFG-02**. Setting a configuration key writes the value to config.toml; the change takes effect on the next server start without requiring a full reinstall.
- **CFG-03**. Unsetting a configuration key removes it from config.toml, reverting to the dataclass default on the next server start.
- **CFG-04**. Reading a configuration key reports which config layer supplied the value (env var, config.toml, or default).
- **CFG-05**. The following environment variables are recognized and override their config.toml equivalents: `LUCEBOX_IMAGE`, `LUCEBOX_VARIANT`, `LUCEBOX_PORT`, `LUCEBOX_CONTAINER`, `LUCEBOX_MODELS`.
- **CFG-06**. Reading an unrecognized configuration key reports a structured error.

#### Updates

- **UPD-01**. The container image update operation pulls the latest Docker image for the configured `LUCEBOX_IMAGE` tag from the registry and reports the digest of the newly pulled image.
- **UPD-02**. Pulling a new image does not restart a running server; the operator must start the server again after pulling to apply the update.
- **UPD-03**. The container image update operation reports the current and new image digests so the operator can confirm a new version was fetched.
- **UPD-04**. The lucebox Python package is updatable via the standard package manager (uv or pip) without reinstalling the Docker image.

### Non-Functional Requirements

- **Performance — server start**: The server start operation reaches a healthy state (responds to `/v1/models`) within 30 seconds on the reference hardware profile.
- **Performance — model switch**: Model activation completes the active model switch (new model serving requests) within 60 seconds on the reference hardware profile.
- **Performance — observability latency**: The status operation returns within 2 seconds under normal server load.
- **Performance — smoke test**: The smoke test operation completes within 10 seconds on a healthy server.
- **Reliability — idempotency**: Server start, model download, and container image pull are idempotent; re-invoking them in the same state produces the same result and exits 0.
- **Reliability — exit codes**: All operations exit 0 on success and exit non-zero on any error; the exit code is distinct between "resource not found" (exit 1) and "server not reachable" (exit 2).
- **Observability — output format**: Status and model list output is human-readable. Machine-parseable (`--json`) output is not yet implemented.
- **Security**: The configuration write operation writes only to config.toml in the lucebox data directory; it does not write to system files, environment files, or files outside the lucebox data directory.

## User Stories

- [TODO: create story] — As Morgan, I want to start the inference server with a single action so that I can begin using my coding agent immediately after booting my machine.
- [TODO: create story] — As Morgan, I want to download a new model without choosing a quantization manually so that I can get a working setup without understanding VRAM math.
- [TODO: create story] — As Sam, I want to switch the active model without restarting the server so that I can compare models mid-session without losing my request queue.
- [TODO: create story] — As Sam, I want to see current tokens/sec and VRAM usage so that I can confirm the server is performing within expected bounds.
- [TODO: create story] — As Sam, I want to run a reproducible throughput benchmark so that I can compare performance across software updates and model changes.
- [TODO: create story] — As Jordan, I want to read and write configuration values through the CLI so that I do not need to manually edit config files or know their locations.
- [TODO: create story] — As Jordan, I want to pull a new Docker image without the server restarting automatically so that I control when the update takes effect.
- [TODO: create story] — As Sam, I want to run a smoke test against the local endpoint so that I can confirm the server is accepting requests before pointing a harness at it.

## Edge Cases and Error Handling

- **Model activate while server is stopped**: Model activation reports that the server is not running and exits non-zero; it does not silently update a config value without a running server to validate against.
- **VRAM insufficient at download time**: Model download reports the minimum VRAM required for the lowest available quantization, the VRAM detected, and the shortfall; it does not download a partial file.
- **Port conflict on serve**: The server start operation detects that the configured port is already bound, reports the conflicting process (if identifiable), and exits non-zero rather than silently using an alternate port.
- **Active model removed forcibly**: Forcibly removing the active model stops the server, removes the model, and reports that the server must be restarted with a different active model; it does not leave the server in a partially degraded state.
- **Config key set while server is running**: Writing a configuration key writes the value to config.toml and emits a warning that the running server will not reflect the change until the next server start.
- **Pull when no new image is available**: The container image pull exits 0 and reports "already at latest" with the current image digest; it does not treat an up-to-date state as an error.
- **Status when server is starting**: The status operation distinguishes "server is starting" (container running, `/props` not yet responsive) from "server is not running" (container absent or exited).
- **Model download interrupted**: A partially downloaded model does not appear in the model list; the partial data is cleaned up automatically.

## Success Metrics

- The server start operation reaches healthy state within 30 seconds on reference hardware across 10 consecutive cold starts (zero failures).
- Model activation completes an active model switch on a server with 10 in-flight requests without dropping any request (all requests return a valid response or well-formed error).
- `lucebox profile` on reference hardware with Qwen3.6-27B Q4_K_M reports all three throughput targets — T1 ≥120 tok/s (short-context), T2 ≥60 tok/s (128K + prefix caching), T3 ≥60 tok/s (128K + PFlash) — in 5 consecutive runs with ≤5% variance between runs per scenario.
- The smoke test exits 0 on a healthy server 100% of the time across the reference hardware test matrix.
- All operations exit non-zero and print a structured error message (not a raw Python traceback) for every documented error condition.

## Constraints and Assumptions

- All operations assume the inference server and Docker runtime are available as specified in FEAT-002 and FEAT-003.
- Configuration management reads and writes only to config.toml in the lucebox data directory; it does not manage environment variables in the shell or in systemd unit files.
- Model auto-quantization selection is based on VRAM and RAM reported at download time; VRAM changes after download (e.g., another process consuming VRAM) are not re-evaluated.
- The container image pull requires network access to the Docker registry; air-gapped operation is not addressed in this feature.
- The `/props` endpoint contract is defined in FEAT-002; this feature consumes it but does not define it.
- Model switching without server restart requires inference server support for hot-swap; this capability is a FEAT-002 contract requirement, not implemented here.
- Throughput benchmark results are reproducible only under controlled conditions (no concurrent inference load, fixed hardware power state); the spec does not guarantee reproducibility across hardware profiles.

## Dependencies

- **FEAT-002**: Inference server must expose the `/props` endpoint (OBS-01, OBS-03) and support active model switching without restart (MDL-07, MDL-08).
- **FEAT-003**: Installation and autotune must have completed successfully before any operation in this spec is valid.
- **Docker daemon**: Server start, container image pull, and server lifecycle operations require the Docker daemon to be running and the NVIDIA Container Toolkit to be registered.
- **Lucebox model registry**: Model download depends on the Lucebox-maintained GGUF registry being reachable (or a local mirror for air-gapped setups).
- **luce-bench**: Profile capture depends on the luce-bench integration being installed and callable.

## Out of Scope

- Initial software installation and hardware autotune — covered by FEAT-003.
- Inference server internals, kernel configuration, and VRAM allocation strategy — covered by FEAT-002.
- Harness adapter configuration and agentic tool routing — covered by FEAT-005.
- Web management UI — covered by FEAT-005.
- Multi-model concurrent serving — addressed as a P2 PRD item; not a requirement of the operational surface in this spec.
- Air-gapped or offline model download — not in scope at launch.
- Windows and macOS support — not in scope per PRD Constraints.
- Fine-tuning, training, or model conversion operations — inference only at launch.
- Automated update scheduling or unattended upgrades — operators initiate all updates explicitly.
