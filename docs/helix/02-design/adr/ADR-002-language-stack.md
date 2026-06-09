---
ddx:
  id: ADR-002
  status: accepted
  links:
    - id: prd
      rel: informs
    - id: concerns
      rel: informs
---

# ADR-002: Language and Runtime Stack

| Date | Status | Deciders | Related | Confidence |
|------|--------|----------|---------|------------|
| 2026-06-08 | Accepted | Erik LaBianca | FEAT-001, FEAT-002, FEAT-006 | High |

## Context

| Aspect | Description |
|--------|-------------|
| Problem | Multiple layers of the stack need language choices: the inference engine, the CLI tooling, the harness adapters, and (future) the web UI. The HELIX `language-runtime` slot defaulted to `typescript-bun` from project bootstrap boilerplate — that default does not reflect the actual implementation or the product's needs |
| Current State | Inference engine: C++17 with CUDA 12 / HIP 7 (custom kernels, existing codebase). CLI and harness adapters: Python/uv (PR #334, #335 — 10,500+ LOC across `lucebox/` and `harness/` packages). Shell wrapper: `lucebox.sh` (~1,275 LOC, top-level orchestrator). No TypeScript/Bun code exists or is planned for v1 |
| Requirements | Inference engine must support CUDA/HIP kernel development; CLI must integrate with Python harness clients; distribution must work on Linux without requiring users to build from source |
| Decision Drivers | GPU kernel development is C++/CUDA — no realistic alternative. Python is the natural language for the harness adapter layer (AI tool CLIs are Python-native ecosystems). `lucebox.sh` shell wrapper already exists and handles systemd/Docker orchestration; shell is the right porcelain for that role |

## Decision

We will use C++17 as the inference engine language, Python/uv as the CLI and harness adapter language, and POSIX shell as the system-level wrapper (`lucebox.sh`). Go is the designated escalation path if shell complexity outgrows what is maintainable. Web UI framework selection is deferred to ADR-004.

**Key Points**: Three-layer stack (C++ → Python → Shell) maps cleanly to three scopes (engine, tooling, orchestration) | Python/uv was chosen in PR #334/#335 and is already evidenced by 10,500+ LOC | Shell/Go porcelain boundary is explicit — Go is a named future option, not a speculation

## Alternatives

| Option | Pros | Cons | Evaluation |
|--------|------|------|------------|
| TypeScript/Bun (HELIX default) | Unified language across CLI and web UI; fast runtime | No CUDA/HIP FFI story; AI tool ecosystem is Python-native; no existing code; would require rewriting PRs | Rejected: boilerplate default, no actual fit for the inference and harness layers |
| Rust for CLI | Memory-safe; fast binary distribution; no runtime dependency | No existing code; significant ramp-up; Python ecosystem integration for AI tools is friction | Rejected: high switching cost with unclear benefit over Python for this layer |
| Go for CLI (now) | Fast binary; easy distribution; single-binary no-runtime deploy | Existing Python codebase would be discarded; `lucebox.sh` already handles orchestration | Deferred: named as the escalation path if shell complexity grows beyond maintainability |
| Python only (no shell wrapper) | Single language for tooling | Systemd and Docker lifecycle management is easier in shell; `lucebox.sh` is already 1,275 LOC and tested | Rejected: shell wrapper handles OS-level concerns that are cleaner in shell than Python |
| **C++ + Python/uv + Shell (+ Go escalation)** | Matches existing implementation; each language owns the scope it's best at; Go named for shell escalation | Three languages to maintain; Python runtime must be present on host for CLI | **Selected**: evidenced by existing codebase; clean scope boundaries; Go gives a credible exit from shell complexity |

## Consequences

| Type | Impact |
|------|--------|
| Positive | Inference engine can use CUDA/HIP intrinsics without a FFI layer; Python CLI integrates naturally with AI tool ecosystems; shell wrapper handles systemd/Docker without language overhead |
| Negative | Users need Python on the host (mitigated by `install.sh` bootstrap); three languages increase contributor ramp-up; shell code has weak type safety |
| Neutral | A SPA framework (see ADR-004 when v2 begins) is the anticipated choice for the web UI; this ADR does not prejudice that decision |

## Risks

| Risk | Prob | Impact | Mitigation |
|------|------|--------|------------|
| `lucebox.sh` shell complexity grows unmaintainable | Med | Med | Go is the named escalation path; evaluate when `lucebox.sh` exceeds ~2,000 LOC or requires significant branching logic |
| Python version drift causes host compatibility issues | Low | Med | `install.sh` pins Python version; uv manages venv isolation |
| Contributor confusion about which layer owns what | Low | Low | Layer ownership documented in architecture overview; each package has a clear README scope statement |

## Validation

| Success Metric | Review Trigger |
|----------------|----------------|
| CLI tests pass on clean Linux installs without manual Python setup | If >20% of support issues relate to Python environment problems, evaluate Go migration for the CLI |
| `lucebox.sh` change velocity remains high | If `lucebox.sh` exceeds ~2,000 LOC or test coverage becomes impractical, initiate Go porcelain ADR |

## Supersession

- **Supersedes**: None
- **Superseded by**: None

## Concern Impact

- **Concern selection**: The `language-runtime` slot is `python-uv` (v1 CLI/harness) + C++17 (inference engine). See `concerns.md` slot decisions.
- **Practice note**: Web UI language/framework is out of scope for this ADR; see ADR-004 when v2 begins.

## References

- PR #334 (docker-stack): Python workspace root, uv.lock, pyproject.toml
- PR #335 (lucebox-cli): `lucebox/` and `harness/` Python packages, 10,500+ LOC
- PRD §Technical Context
- `concerns.md` — slot decisions
