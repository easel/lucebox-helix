---
ddx:
  id: FEAT-005
  status: draft
  links:
    - id: prd
      rel: informs
    - id: FEAT-002
      rel: depends-on
---

# Feature Specification: FEAT-005 — Users

**Feature ID**: FEAT-005
**Status**: Draft
**Priority**: P0 (v1 harness adapters); P1 (v2 web UI)
**Owner**: Product Team
**Covered PRD Subsystem(s)**: Agentic Harness Wrappers
**Covered PRD Requirements**: FR-14, FR-15, FR-16, FR-17
**Cross-Subsystem Rationale**: None — single subsystem.

## Overview

This feature defines the user-facing interface layer through which developers
interact with the Lucebox inference server using their existing tools. In v1,
it delivers six harness adapters — one `lucebox` CLI subcommand per supported
agentic coding tool — that configure and exec the real client binary pointed
at the local endpoint, satisfying the PRD's FR-14 through FR-17 requirements
for no-cloud-key operation and stateless invocation. In v2 (deferred), it
extends to a browser-based management UI at `lucebox.local`.

## Ideal Future State

A developer already running claude-code, codex, opencode, hermes-agent, pi, or
openclaw against cloud endpoints runs a single `lucebox <tool>` command and
their existing tool starts working against the local Lucebox inference server —
no API key changes, no configuration file edits, no understanding of the
inference server internals required. The tool behaves identically to its
cloud-connected form: completions arrive, context is preserved, and the
developer's existing workflow is unchanged except that inference is local and
free of cloud API charges.

When a requested underlying tool is not installed, Lucebox fails clearly with
a message naming the missing binary and where to get it — not silently,
not with a cryptic error from inside the underlying tool.

In v2, the same developer can open `lucebox.local` in a browser to inspect
model status, view inference metrics, manage installed models, and configure
harness defaults — without any additional server process to manage.

## Problem Statement

- **Current situation**: Agentic coding tools (claude-code, codex, opencode,
  hermes-agent, pi, openclaw) are built to call cloud inference endpoints.
  Pointing them at a local server requires reading each tool's documentation,
  setting environment variables or editing config files, finding the right API
  key format the local server accepts, and identifying the correct model
  identifier string — different for every tool.
- **Pain points**: The per-tool setup is repetitive, error-prone, and
  non-obvious. Developers who have already invested in a specific tool's
  workflow (keyboard shortcuts, context management, project rules) do not want
  to learn a new tool to get local inference. Any cloud-first default that
  leaks through — a hardcoded endpoint, an enforced key format, a model ID
  that differs from the server's registered name — silently falls back to
  cloud usage and defeats the privacy and cost goals.
- **Desired outcome**: Zero cloud API calls when a developer invokes
  `lucebox <tool>`. No adapter requires a cloud API key. A developer new to
  Lucebox can switch any supported tool to local inference in under 60 seconds
  after the server is running.

## Functional Areas

| Area | User question or job | Feature responsibility |
|------|----------------------|------------------------|
| Harness Adapters | How do I run my existing tool against the local server? | Locate the binary, configure the environment, exec the real process |
| Shared Adapter Interface | What is the common behavior all adapters guarantee? | `find_bin()` / `exec_client()` contract; default credentials; statelessness |
| Claude Code Adapter (Anthropic compat) | Does claude-code work without code changes on my side? | Route through the Anthropic Messages API-compatible endpoint |
| v2 Web UI (deferred) | How do I manage models and monitor the server without a CLI? | Browser SPA at `lucebox.local`; deferred to v2 |

## Requirements

### Functional Requirements by Area

#### Harness Adapters

ADAPT-01. `lucebox claude-code` locates the `claude` binary and execs it
configured to use the local Lucebox inference endpoint.

ADAPT-02. `lucebox codex` locates the `codex` binary and execs it configured
to use the local Lucebox inference endpoint.

ADAPT-03. `lucebox opencode` locates the `opencode` binary and execs it
configured to use the local Lucebox inference endpoint.

ADAPT-04. `lucebox hermes-agent` locates the `hermes-agent` binary and execs
it configured to use the local Lucebox inference endpoint.

ADAPT-05. `lucebox pi` locates the `pi` binary and execs it configured to use
the local Lucebox inference endpoint.

ADAPT-06. `lucebox openclaw` locates the `openclaw` binary and execs it
configured to use the local Lucebox inference endpoint.

ADAPT-07. Each adapter replaces the current process via exec — no lucebox
wrapper process remains running after the underlying client binary is launched.

ADAPT-08. Each adapter passes through all positional arguments and flags
supplied by the user to the underlying client binary without modification.

#### Shared Adapter Interface

COMMON-01. `find_bin()` locates the client binary for any adapter; when the
binary is not found, the adapter exits with a clear error message naming the
missing binary and indicating that the underlying tool must be installed
separately.

COMMON-02. `exec_client()` sets the API base URL, API key, and model ID
environment variables for the target process before exec.

COMMON-03. All adapters default to API key `sk-lucebox` — no cloud API key is
required for any adapter to function against the local Lucebox endpoint.

COMMON-04. All adapters default to model ID `luce-dflash` when no explicit
model is specified by the user.

COMMON-05. All adapters are stateless: repeated invocations of the same
adapter do not modify, accumulate state in, or corrupt the underlying client
tool's configuration files, credential stores, or persistent settings.

#### Claude Code Adapter (Anthropic API Compatibility)

CLAUDE-01. `lucebox claude-code` routes requests through an Anthropic Messages
API-compatible endpoint on the inference server so that the `claude` CLI
operates without code changes to the CLI or to the user's project
configuration.

CLAUDE-02. The Anthropic-compatible endpoint accepts the `sk-lucebox` default
key without requiring a valid Anthropic API account.

#### v2 Web UI (Deferred)

The v2 web UI is explicitly out of scope for v1. Requirements in this area are
placeholders that anchor the v2 direction; full specification is deferred until
v2 begins.

WEB-01. (v2) A browser-based management UI is accessible at `lucebox.local`
with no additional server process required beyond the inference server.

WEB-02. (v2) The UI is a static SPA that consumes the C++ inference server API
directly; no Node.js or backend server process intermediates requests.

WEB-03. (v2) The UI surfaces model management (download, activate, remove),
server status, inference metrics, and harness configuration.

WEB-04. (v2) The frontend framework and static asset serving mechanism are
decided by ADR-004; SvelteKit with a static adapter or Svelte+Vite is the
leading candidate.

### Non-Functional Requirements

- **Latency**: Adapter startup overhead (binary lookup + exec handoff) adds
  ≤100ms to the first invocation after `lucebox <tool>` is entered.
- **Reliability**: Each adapter must not emit any output to stdout or stderr
  from the lucebox process after exec succeeds; all output belongs to the
  underlying tool.
- **Compatibility**: Adapters must not require any modification to the
  underlying client tool or its configuration; compatibility is maintained
  through environment variable injection only.
- **Isolation**: An adapter invocation must not leave modified environment
  variables, temporary files, or process-level side effects visible to
  processes other than the execed client binary.
- **Error clarity**: A "binary not found" error message must name the missing
  binary and be actionable within 10 seconds of reading — no stack trace, no
  ambiguous failure mode.

## User Stories

- [TODO: create story] US-XXX — Developer routes claude-code to local inference
  with a single command
- [TODO: create story] US-XXX — Developer routes codex to local inference with
  a single command
- [TODO: create story] US-XXX — Developer routes opencode to local inference
  with a single command
- [TODO: create story] US-XXX — Developer routes hermes-agent to local
  inference with a single command
- [TODO: create story] US-XXX — Developer routes pi to local inference with a
  single command
- [TODO: create story] US-XXX — Developer routes openclaw to local inference
  with a single command
- [TODO: create story] US-XXX — Developer sees a clear error when the
  underlying tool is not installed

## Edge Cases and Error Handling

- **Underlying binary not installed**: `find_bin()` fails before exec; the
  adapter exits immediately with a message naming the missing binary and
  indicating it must be installed separately. No partial environment setup
  occurs.
- **Inference server not running**: The adapter execs successfully; the error
  is surfaced by the underlying client tool when it attempts its first request.
  The adapter does not pre-validate server reachability.
- **User supplies an explicit `--model` or API key flag**: Adapter-injected
  defaults are overridden by explicit user-supplied flags at the underlying
  tool's argument level; the adapter does not suppress user overrides.
- **Multiple concurrent adapter invocations**: Each invocation is independent;
  because adapters are stateless and replace the current process, concurrent
  invocations cannot interfere with each other.
- **Adapter invoked inside an existing agentic tool session**: The behavior
  is governed entirely by the underlying tool's handling of a re-invocation;
  the adapter has no knowledge of the outer session.
- **`lucebox claude-code` with a non-Anthropic model ID active**: The
  Anthropic-compatible endpoint on the inference server is responsible for
  model routing; the adapter sets the model ID to `luce-dflash` by default
  and passes it through.

## Success Metrics

- 100% of supported adapters (claude-code, codex, opencode, hermes-agent, pi,
  openclaw) route zero requests to cloud endpoints when the local server is
  running.
- Time from `lucebox <tool>` invocation to the underlying tool's first prompt
  is ≤60 seconds on a clean install (excluding server cold-start).
- 0 cloud API key prompts across all adapter invocations in the acceptance
  test suite.
- "Binary not found" error messages are rated actionable (no follow-up
  question needed) in ≥90% of user-test observations.

## Constraints and Assumptions

- All six target CLIs (claude, codex, opencode, hermes-agent, pi, openclaw)
  support endpoint and API key configuration via environment variables at the
  time of v1 launch; if a tool does not, its adapter is blocked.
- The Anthropic Messages API-compatible endpoint on the inference server
  (FEAT-002) is available before FEAT-005 can ship the `claude-code` adapter.
- The v2 web UI static asset serving approach (C++ server vs. separate file
  server) is an open architectural decision tracked in ADR-004; this spec does
  not constrain that choice.
- Adapters are not responsible for informing the user whether the inference
  server is healthy; that responsibility belongs to `lucebox check` and the
  inference server itself.

## Dependencies

- **Other features**: FEAT-002 (Inference Server API) — adapters require a
  running inference server with an OpenAI-compatible endpoint; the
  `claude-code` adapter additionally requires the Anthropic Messages
  API-compatible endpoint defined in FEAT-002.
- **External binaries**: `claude` (Claude Code CLI), `codex` (OpenAI Codex
  CLI), `opencode` (OpenCode CLI), `hermes-agent` (Hermes Agent CLI), `pi`
  (Pi CLI), `openclaw` (OpenClaw CLI) — each must be installed independently
  by the user; Lucebox does not bundle or install them.
- **PRD requirements**: FR-14 (adapter subcommands), FR-15 (no cloud API key),
  FR-16 (Anthropic compat endpoint), FR-17 (stateless adapters).

## Out of Scope

- The inference server API surface itself (OpenAI-compatible and
  Anthropic-compatible endpoints) — covered by FEAT-002.
- Installation of the `lucebox` CLI package and the harness adapter scripts —
  covered by FEAT-003.
- Installation or management of the underlying tool CLIs (claude, codex,
  opencode, hermes-agent, pi, openclaw) — those are user-managed; Lucebox
  does not wrap their install paths.
- Custom agentic harness or fork of any underlying tool — deferred to v3 per
  the PRD non-goals.
- Windows or macOS adapter support at launch.
- Any network-level proxying, request logging, or content inspection of
  traffic between the adapter-launched client and the inference server.
- The v2 web SPA full specification — deferred; WEB-01 through WEB-04 are
  directional placeholders only.
