---
ddx:
  id: ADR-003
  status: accepted
  links:
    - id: prd
      rel: informs
    - id: concerns
      rel: informs
---

# ADR-003: Docker as Server Deployment Mechanism

| Date | Status | Deciders | Related | Confidence |
|------|--------|----------|---------|------------|
| 2026-06-08 | Accepted | Erik LaBianca | FEAT-001, FEAT-002 | High |

## Context

| Aspect | Description |
|--------|-------------|
| Problem | The dflash_server inference engine is a C++17 binary with a hard dependency on a specific CUDA runtime, custom kernel `.so` files, and model weights that must co-locate with the binary. Distributing and keeping this stack consistent across heterogeneous Linux hosts without requiring users to build from source is a hard packaging problem |
| Current State | PR #334 ships a complete Docker stack: `Dockerfile`, `docker-bake.hcl`, `scripts/build_image.sh`, `server/scripts/entrypoint.sh`, and a GitHub Actions `docker.yml` CI pipeline. The pre-built image (`ghcr.io/luce-org/lucebox-hub:cuda12`, ~14 GB) bundles the CUDA runtime, custom kernels, and server binary. `lucebox docker_run` (Python) manages container lifecycle; `lucebox.sh` wraps it for the operator |
| Requirements | Users must not need to compile C++ or install CUDA toolkit to use the inference server; updates must be testable and rollback-capable; the CLI must be able to start, stop, and inspect the server |
| Decision Drivers | CUDA runtime distribution is notoriously fragile; Docker + NVIDIA Container Toolkit (CTK) is the standard Linux solution; the pre-built image is already built and published in CI |

## Decision

We will deploy the dflash_server inference engine as a Docker container managed by the `lucebox` CLI and `lucebox.sh` wrapper. The Python `lucebox docker_run` module owns container lifecycle (pull, run, stop, inspect). The host requires Docker daemon + NVIDIA CTK; `lucebox check` validates both before any server operation.

**Key Points**: Container image bundles CUDA runtime — users need only Docker + CTK, not the full CUDA toolkit | Image is published to `ghcr.io/luce-org/lucebox-hub` via CI on every tagged release | `lucebox.sh` is the operator-facing entrypoint; it shells out to `lucebox docker_run` for container management

## Alternatives

| Option | Pros | Cons | Evaluation |
|--------|------|------|------------|
| Bare binary + LD_LIBRARY_PATH | No Docker dependency; lower overhead | CUDA runtime version must exactly match host; .so hell; no isolation; hard to version and rollback | Rejected: CUDA runtime distribution without containers is the primary source of "driver hell" the product promises to eliminate |
| conda / venv environment | Familiar to Python/ML users | conda is not a C++ binary packager; still requires CUDA toolkit on host; large environment size without isolation benefits | Rejected: solves the Python layer, not the C++ runtime layer |
| AppImage / Flatpak | Single-file distribution; no Docker dependency | Poor CUDA/GPU device passthrough story; no official NVIDIA support path; community AppImage for GPU workloads is fragile | Rejected: NVIDIA's supported Linux GPU compute path is Docker + CTK |
| Nix | Reproducible; GPU support improving | Steep operator learning curve; not standard in ML/inference community; limited harness tool compatibility | Rejected: too high a barrier for the beachhead user (Morgan, Sam) |
| **Docker + NVIDIA CTK** | NVIDIA-supported GPU passthrough; image bundles CUDA runtime; versioned and rollback-capable; CI-built and published | ~14 GB image; Docker daemon required; CTK must be installed | **Selected**: solves CUDA distribution cleanly; standard in inference infrastructure; evidenced by PR #334 |

## Consequences

| Type | Impact |
|------|--------|
| Positive | CUDA runtime is bundled — no host-level CUDA toolkit required; image versions are pinned and rollback-capable; CI builds and publishes images automatically |
| Negative | Docker daemon + NVIDIA CTK are hard prerequisites (`lucebox check` must fail loudly if absent); ~14 GB image pull on first install; Docker adds ~5–10% latency overhead vs. bare metal for GPU workloads |
| Neutral | `deploy-target:local-appliance` concern practice updates: inference daemon runs in Docker (not bare systemd service); systemd manages the Docker daemon and `lucebox.sh` service unit |

## Risks

| Risk | Prob | Impact | Mitigation |
|------|------|--------|------------|
| Docker daemon not available in enterprise/locked-down environments | Low | High | `lucebox check` fails loudly with a clear install path; bare-binary path documented as a DIY option for advanced users |
| NVIDIA CTK version mismatch with Docker | Med | Med | `lucebox check` validates CTK registration; pin CTK version in install.sh |
| ~14 GB image pull is prohibitive on slow connections | Med | Low | Image layers are cached; incremental updates on version bumps are small; models are not in the image |

## Validation

| Success Metric | Review Trigger |
|----------------|----------------|
| `lucebox check` on a reference machine with Docker + CTK installed reports all-pass | If >15% of support issues relate to Docker/CTK setup, evaluate a bare-binary distribution path for advanced users |
| GPU passthrough latency overhead ≤10% vs. bare metal on reference hardware | If overhead exceeds 10% on the reference hardware benchmark, evaluate bare-binary path for performance-critical users |

## Supersession

- **Supersedes**: None
- **Superseded by**: None

## Concern Impact

- **Practice override**: The `deploy-target:local-appliance` concern currently states "systemd services for inference daemon." This ADR overrides that practice: the inference daemon runs in a Docker container; systemd manages the Docker daemon and a `lucebox.service` unit that calls `lucebox.sh`. Update `concerns.md` Project Overrides to reference ADR-003.

## References

- PR #334 (docker-stack): `Dockerfile`, `docker-bake.hcl`, `server/scripts/entrypoint.sh`, `.github/workflows/docker.yml`
- PRD §Technical Context, §Constraints
- PRD §Requirements — FR-18 (installs on compatible Linux hardware), P1 (systemd service)
