---
ddx:
  id: FEAT-002
  status: draft
  links:
    - id: prd
      rel: informs
    - id: FEAT-001
      rel: depends-on
---

# Feature Specification: FEAT-002 — Inference Server

**Feature ID**: FEAT-002
**Status**: Draft
**Priority**: P0
**Owner**: Server Team
**Covered PRD Subsystem(s)**: Inference Server
**Covered PRD Requirements**: FEAT-002 (Inference Server)
**Cross-Subsystem Rationale**: None — single subsystem.

## Overview

The inference server is the HTTP API layer that wraps the inference engine
(FEAT-001) and exposes it to all clients. It owns the server process lifecycle,
the protocol-compatible API surface, the request queue, and the configuration
contract. Every agentic harness adapter (FEAT-005) and every direct API client
reaches the inference engine exclusively through this server.

The server exists because clients — OpenAI-SDK-based tools, the Claude Code CLI,
and future integrations — must be able to drop in a `base_url` and an API key
without code changes. Holding that contract is the full scope of this feature.

## Ideal Future State

A developer points any OpenAI-SDK-based tool at `http://localhost:<port>` with
API key `sk-lucebox`, sets `model=luce-dflash`, and gets responses
indistinguishable from a cloud endpoint — same schema, same streaming behavior,
same error shapes — at ≥120 tok/s sustained throughput on Qwen3.6-27B Q4_K_M.

The Claude Code CLI routes to the same box without any modification to the
claude binary: the server speaks the Anthropic Messages API on a dedicated
endpoint, and the claude-code harness adapter (FEAT-005) sets the right
environment variables.

The server starts on boot, survives container restarts, never drops a request
under normal load, and publishes its capabilities through a machine-readable
`/props` endpoint so clients and operators can negotiate schema versions and
feature availability without out-of-band documentation.

## Problem Statement

- **Current situation**: The inference engine (FEAT-001) provides raw inference
  capability but has no stable HTTP interface. Clients must use engine-internal
  APIs or proprietary bindings.
- **Pain points**: Every agentic tool (claude-code, codex, opencode, etc.)
  expects an OpenAI-compatible HTTP endpoint. Claude Code requires Anthropic
  Messages API semantics beyond the OpenAI surface. Without a protocol-compatible HTTP
  layer, no harness adapter can route to local inference without code changes in
  the upstream tool.
- **Desired outcome**: A single server process that passes OpenAI API
  conformance on `/v1/completions`, `/v1/chat/completions`, and `/v1/models`;
  passes Anthropic Messages API conformance on the Claude endpoint; and handles
  burst load without dropping requests — verified by automated conformance tests
  before each release.

## Functional Areas

| Area | User question or job | Feature responsibility |
|------|----------------------|------------------------|
| Server process | Is the server running and healthy? | Process lifecycle, port binding, SIGTERM handling, startup/shutdown |
| OpenAI API surface | Can I use my OpenAI SDK client unchanged? | `/v1/completions`, `/v1/chat/completions`, `/v1/models`, streaming, error schemas |
| Anthropic API surface | Can the Claude Code CLI route here without changes? | Anthropic Messages API endpoint, schema conformance, error translation |
| Capabilities endpoint | What does this server support? | `/props` schema versions 3 and 4, capability negotiation |
| Request queue | What happens when the server is busy? | Queue-based backpressure, observable queue depth, no dropped requests |
| Configuration | How do I change the port or model path? | `LUCEBOX_PORT`, `LUCEBOX_MODELS`, image/variant env vars; config.toml; dataclass defaults |
| Authentication | How do I authenticate? | API key validation; configurable key; default `sk-lucebox` |

## Requirements

### Functional Requirements by Area

#### Server Process

**SRV-01**. The server runs inside the `ghcr.io/luce-org/lucebox-hub:cuda12`
Docker container, started and stopped by the server lifecycle management
capability (FEAT-004).

**SRV-02**. The server binds to the port specified by `LUCEBOX_PORT`; when
unset, a documented default port is used.

**SRV-03**. The server exits cleanly on SIGTERM, completing in-flight requests
before shutdown; it does not abandon queued requests without a response.

**SRV-04**. The server exposes a health-check surface sufficient for the
container runtime to determine liveness.

#### OpenAI API Surface

**OAI-01**. `POST /v1/completions` accepts and responds with the OpenAI
Completions API v1 schema, including streaming (`stream: true`).

**OAI-02**. `POST /v1/chat/completions` accepts and responds with the OpenAI
Chat Completions API v1 schema, including streaming.

**OAI-03**. `GET /v1/models` returns a list of available and active models in
the OpenAI models list schema; the default model ID `luce-dflash` is always
present when a model is loaded.

**OAI-04**. Error responses on `/v1/*` endpoints use the OpenAI error schema
(`error.type`, `error.message`, `error.code`).

**OAI-05**. The model alias `luce-dflash` resolves to the currently active
model with DFlash optimizations enabled; requests specifying `luce-dflash` must
not fail with model-not-found errors when a model is loaded.

#### Anthropic API Surface

**ANT-01**. The server exposes an Anthropic Messages API-compatible endpoint
that accepts and responds with the Anthropic Messages API schema.

**ANT-02**. The Anthropic endpoint supports streaming responses equivalent to
the streaming behavior of the Anthropic Messages API.

**ANT-03**. Authentication on the Anthropic endpoint accepts the same
configurable API key used by the OpenAI surface.

#### Capabilities Endpoint

**CAP-01**. `GET /props` returns a structured document describing server
properties and capabilities; the document conforms to either schema version 3
or schema version 4 depending on client negotiation.

**CAP-02**. `/props` includes the current queue depth so clients and operators
can observe backpressure without a separate management call.

**CAP-03**. `/props` is the authoritative source for capability negotiation
between server and client; clients must be able to determine supported API
surfaces and schema versions from this endpoint alone.

#### Request Queue

**QUE-01**. Inference requests that arrive while the engine is busy are placed
in a queue; they are not rejected or dropped.

**QUE-02**. Queue depth is observable via `/props` (or a dedicated status
endpoint) and via the CLI status command (FEAT-004).

**QUE-03**. The queue is bounded; when the queue is full, the server returns a
documented error response rather than accepting the request silently and
losing it.

#### Configuration

**CFG-01**. Configuration is resolved in priority order: environment variables
override `config.toml` values, which override dataclass defaults.

**CFG-02**. `LUCEBOX_PORT` sets the listening port.

**CFG-03**. `LUCEBOX_MODELS` sets the model search path used by the server to
discover installed models.

**CFG-04**. `LUCEBOX_IMAGE` and `LUCEBOX_VARIANT` select the Docker image and
variant; these are managed by the installation subsystem (FEAT-003) and are
read-only from the server's perspective.

#### Authentication

**AUTH-01**. The server validates API keys on every request to protected
endpoints.

**AUTH-02**. The default API key is `sk-lucebox`; it is accepted without any
configuration change on a fresh install.

**AUTH-03**. The API key is configurable via `config.toml` or an environment
variable; the server rejects requests carrying an unrecognized key with HTTP
401.

### Non-Functional Requirements

- **Throughput**: ≥120 tok/s sustained on Qwen3.6-27B Q4_K_M under single-request
  load on the reference hardware profile (RTX 3090 24GB + 128GB system RAM).
- **Context window**: Handles inference requests up to 128K tokens for
  Qwen3.6-27B without OOM on the reference hardware profile.
- **Latency**: Time-to-first-token on a 1K-token prompt must not exceed 2 s
  under unloaded conditions on reference hardware (target; to be validated
  against engine benchmarks).
- **Reliability**: The server must not drop or lose a queued request during
  normal operation; request loss under graceful shutdown is zero.
- **Startup time**: Server reaches the health-check passing state within 60 s
  of container start on reference hardware (excludes model load time, which is
  governed by FEAT-001).
- **Security**: All endpoints require API key authentication; unauthenticated
  requests are rejected with HTTP 401 before any inference work is performed.
- **Protocol conformance**: OpenAI API v1 conformance is verified by automated
  tests before each release; Anthropic Messages API conformance is verified by
  automated tests before each release.
- **Observability**: Queue depth and active request count are always readable
  from `/props` without authentication so operators can diagnose backpressure
  from a monitoring script.

## User Stories

- [TODO: create story] — Developer routes an existing OpenAI-SDK-based tool to Lucebox using only `base_url` and `api_key` changes.
- [TODO: create story] — Developer routes Claude Code CLI to Lucebox via the Anthropic Messages API endpoint without modifying the claude binary.
- [TODO: create story] — Developer observes queue depth during a long-running batch job to confirm no requests are being dropped.
- [TODO: create story] — Operator changes the listening port via `LUCEBOX_PORT` without editing `config.toml`.
- [TODO: create story] — Operator queries `/props` to confirm supported schema version before upgrading a client integration.

## Edge Cases and Error Handling

- **Request arrives during model load**: Request is held in queue until the
  model is ready; the client does not receive an error unless the queue fills
  before load completes.
- **Request exceeds 128K-token context limit**: Server returns a documented
  error response (HTTP 400, OpenAI error schema) before forwarding to the
  engine; no OOM is triggered.
- **Unknown model ID in request**: Server returns HTTP 404 with an OpenAI
  error schema response; `luce-dflash` is never treated as unknown when a model
  is loaded.
- **Queue full**: Server returns HTTP 429 (Too Many Requests) with a
  `Retry-After` header; the request is not accepted into the queue.
- **Invalid or missing API key**: Server returns HTTP 401 before performing any
  inference work.
- **SIGTERM during active requests**: In-flight requests complete; queued
  requests receive a 503 with a documented shutdown error code; no request is
  silently abandoned.
- **`/props` schema version not supported by client**: Server returns the
  highest supported schema version; clients must handle downgrade gracefully.
- **Container restart mid-request**: Requests in-flight at container crash are
  not completed; clients receive a connection reset and must retry; the server
  does not guarantee at-most-once delivery beyond the TCP layer.

## Success Metrics

- OpenAI API conformance tests pass on 100% of tested endpoints before each
  release.
- Anthropic Messages API conformance tests pass on the Claude endpoint before
  each release.
- Zero requests dropped under sustained load at queue-not-full depth in
  automated load tests.
- `luce-dflash` model alias resolves successfully on 100% of requests when a
  model is loaded, in automated smoke tests.
- `/props` returns a valid schema-version-3 or schema-version-4 document on
  100% of requests in automated smoke tests.
- Server reaches health-check passing state within 60 s of container start in
  automated startup timing tests on reference hardware.

## Constraints and Assumptions

- The server is not responsible for model loading, quantization selection, or
  VRAM management — those are FEAT-001 concerns.
- The server does not manage the Docker container lifecycle — that is owned by
  the server lifecycle management capability (FEAT-004) and the shell wrapper
  script.
- `LUCEBOX_IMAGE` and `LUCEBOX_VARIANT` are consumed by the server as read-only
  values set by the installation subsystem (FEAT-003); the server does not pull
  or switch Docker images.
- Anthropic Messages API endpoint is P0, required for the claude-code harness adapter (FEAT-005). The OpenAI surface is also P0 and ships regardless.
- The default API key `sk-lucebox` is suitable for local loopback use; it is
  not suitable for network-exposed deployments. Operators exposing the server
  beyond localhost are responsible for setting a strong key.
- Queue bound is implementation-defined for v1; the exact limit is published in
  the `config.toml` schema documentation, not in this spec.
- The `/props` schema versions 3 and 4 are pre-existing contracts from the
  inference engine team; this spec does not define their schema — it specifies
  that the server must serve them.

## Dependencies

- **FEAT-001 — Inference Engine**: The server wraps FEAT-001; it cannot serve
  inference requests without a running engine instance.
- **FEAT-003 — Installation**: `LUCEBOX_IMAGE` and `LUCEBOX_VARIANT` values are
  set by the installation subsystem; the server reads but does not manage them.
- **FEAT-004 — Operations**: The server lifecycle management capability starts
  the server process. Queue depth observability via CLI is a FEAT-004
  responsibility that depends on the `/props` contract defined here.
- **Docker runtime**: `ghcr.io/luce-org/lucebox-hub:cuda12` must be present
  on the host; the server runs exclusively inside this container.
- **OpenAI API schema v1**: The server's primary protocol contract. Changes to
  the OpenAI schema that are not backward-compatible require a versioned
  migration plan.
- **Anthropic Messages API schema**: The server's secondary protocol contract
  (P1). Subject to upstream Anthropic schema changes.

## Out of Scope

- **Inference engine internals** (kernel dispatch, quantization, VRAM
  management, GGUF loading) — FEAT-001.
- **Runtime operations** (server lifecycle, status, model management) —
  FEAT-004.
- **Installation and image management** (shell wrapper, systemd unit, Docker
  image pull) — FEAT-003.
- **Agentic harness adapters** (environment variable injection for claude-code,
  codex, opencode, etc.) — FEAT-005.
- **Web management UI** — covered by FEAT-005; the server must serve the static SPA assets but does not own the UI itself.
- **Multi-box clustering or distributed inference** — not in scope per PRD non-goals.
- **Cloud-hosted or hybrid inference** — not in scope per PRD non-goals.
- **Fine-tuning endpoints** — inference only at launch.
- **Rate limiting beyond queue depth** (per-key quotas, token-budget enforcement) — not in scope at launch.
- **TLS termination** — the server listens on plaintext localhost; TLS is an operator concern for network-exposed setups.
