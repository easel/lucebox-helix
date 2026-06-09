---
ddx:
  id: concerns
  status: draft
  links:
    - id: product-vision
      rel: informs
---

# Project Concerns

Project Concerns declare active cross-cutting context for downstream work. They
are not principles, requirements, ADRs, test plans, or implementation tasks.

## Active Concerns

| Concern | Source | Areas | Why Active | Key Practices |
|---------|--------|-------|------------|---------------|
| `python-uv` | ADR-002 (language-runtime slot) | `area:cli` `area:inference` | CLI (`lucebox` Typer app) and harness adapters are Python/uv; C++ owns the inference engine; shell (`lucebox.sh`) owns systemd/Docker orchestration | Manage dependencies with uv; lint with Ruff; test with pytest; do not use Bun or Node for CLI/harness layers |
| `spa-framework` | ADR-004 pending (frontend-framework slot) | `area:ui` | The management UI is a SPA that consumes the C++ inference API directly — no Node.js server process; framework TBD (SvelteKit static adapter or Svelte+Vite is the leading candidate) | SPA talks directly to C++ `/v1` API; serve static assets from the C++ server or a simple file server; no SSR required; evaluate SvelteKit (adapter-static) vs. Svelte+Vite before UI work begins |
| `e2e-playwright` | library (e2e-framework slot) | `area:ui` | Operator-facing UI must have ≥1 whole-stack e2e covering the core model-pull-and-serve flow | Browser e2e via Playwright; at least the model-pull → inference-endpoint happy path must run green |
| `auth-local-sessions` | library (auth-provider slot) | `area:ui` `area:api` | Local appliance; single-user at beachhead but needs basic session protection for the management UI | Cookie-based sessions; no cloud identity dependency; password set on first boot |
| `admin-console` | library | `area:ui` | Management UI is operator-facing: users manage model library, server config, running inference tasks — mutable domain objects with lifecycle state | Core operator workflows (pull model, configure endpoint, monitor inference, pause/cancel job) must be exercised through the UI end-to-end |
| `testing` | library | `area:api` `area:ui` `area:cli` | Standard test discipline across server, UI, and CLI layers | Unit tests for inference configuration logic; integration tests for model management API; e2e via Playwright for the UI |
| `verification` | library | `area:api` `area:ui` | Running product must be verified with recorded evidence before marking work done | Inference throughput benchmarks run against the actual hardware; UI flows exercised in a running Lucebox environment |
| `o11y-otel` | library | `area:api` `area:infra` | Inference server needs latency, throughput, queue depth, and hardware-utilization metrics for both operator monitoring and performance tuning | OpenTelemetry spans on inference requests; hardware metrics (GPU/NPU utilization, memory bandwidth) exported to local collector |
| `security-owasp` | library | `area:ui` `area:api` | Management UI is a web app on the local network; inference API may be exposed to other machines on the LAN | Standard OWASP top-10 mitigations; CORS policy restricts API to trusted origins; CSP headers on the UI |
| `deploy-target:local-appliance` | project-local assumption (see ADR-003) | `area:infra` | Lucebox ships as a pre-built box, not a cloud service; the deploy target is the physical hardware running a Linux-based OS | Software updates delivered as tested bundles; inference daemon runs in Docker container managed by `lucebox.sh`; systemd manages the Docker daemon and a `lucebox.service` unit; no Kubernetes or cloud PaaS dependency at launch |

## Project Overrides

| Concern | Practice | Override | Authority |
|---------|----------|----------|-----------|
| `deploy-target:local-appliance` | (no shipped library concern — project-local) | Deploy to physical box via tested update bundle; inference daemon in Docker container, not bare systemd service | ADR-003 |
| `auth-local-sessions` | Multi-tenant auth flows | Single-user at beachhead; no tenant isolation or role hierarchy required at launch | Needs ADR |

## Slot Decisions

| Slot | Chosen Filler | Source |
|------|--------------|--------|
| `frontend-framework` | TBD — SPA (SvelteKit static or Svelte+Vite is the leading candidate) | ADR-004 pending; no Node.js server process |
| `language-runtime` | `python-uv` (CLI/harness); C++17 (inference engine) | ADR-002 — shipped default (`typescript-bun`) was boilerplate, not intentional |
| `e2e-framework` | `e2e-playwright` | shipped default |
| `auth-provider` | `auth-local-sessions` | shipped default |
| `deploy-target` | `local-appliance` (project-local) | assumption — Lucebox is a physical box, not a cloud service |
| `architecture-style` | TBD | undecided — decide before first ADR; `twelve-factor` or `classic-layered` are candidates |
| `datastore` | TBD | undecided — local SQLite likely for model metadata and conversation history; decide before data design |

## Area Labels

- `area:ui` — management UI (web SPA; framework TBD — SvelteKit or Svelte+Vite; served statically, no Node.js server)
- `area:api` — inference server API (OpenAI-compatible + Lucebox-native)
- `area:infra` — hardware management, system services, update delivery
- `area:cli` — command-line tools for server management and scripting
- `area:inference` — inference runtime optimization, model loading, quantization selection
- `area:hardware` — hardware-specific optimization layer, driver integration, thermal management

## Concern Conflicts

| Conflict | Resolution |
|----------|------------|
| `verification` (run on real hardware) vs. CI portability | Benchmark and hardware-utilization verification runs on a physical Lucebox; functional tests (unit + integration + Playwright) run in CI on Linux x86. Separate CI pipelines; hardware verification is a release gate, not a PR gate |
| SPA direct-API vs. real-time inference metrics | SPA talks directly to the C++ `/v1` API; use server-sent events or WebSocket from the inference API for live metrics (tokens/second, queue depth) — no SSR intermediary to route around |
