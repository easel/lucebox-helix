---
ddx:
  id: TD-005
  status: draft
  links:
    - id: FEAT-003
      rel: informs
    - id: FEAT-004
      rel: informs
    - id: FEAT-005
      rel: informs
    - id: ADR-002
      rel: informed-by
    - id: ADR-003
      rel: informed-by
---

# Technical Design: TD-005 — CLI Surface

**Source**: PR #335 (`feat(lucebox): hub CLI + autotune/sweep/profile + harness adapters + shell wrapper`)  
**Head SHA**: `a56b51b7c5c2ef050792f4fd66c47740a558a896`  
**Status**: Draft — reflects implemented code; open questions noted inline.

---

## Overview

The `lucebox` CLI is a two-layer system. `lucebox.sh` is the shell entry point (~1,275 LOC): it handles systemd lifecycle, container management, and `serve`. Everything else is dispatched either directly to shell commands or routed into the `lucebox` Python package (Typer app, `lucebox/src/lucebox/cli.py`). This document is the authoritative command-surface contract for both layers.

**Package structure**

| Package | Role |
|---------|------|
| `lucebox/` (Python, uv) | Typer CLI — host-check, autotune, config, models, smoke, profile, harness launchers |
| `harness/` (Python, uv) | Per-tool adapter clients — `claude_code`, `codex`, `opencode`, `hermes`, `pi`, `openclaw` |
| `lucebox.sh` (POSIX shell) | Entry point; systemd, serve, pull, update, logs, completion; catch-all routes to container |
| `install.sh` (POSIX shell) | Bootstrap — installs shell wrapper and Python package |

---

## Dispatch Architecture

```
lucebox <cmd> [args]
     │
     ▼
lucebox.sh  main()
     │
     ├── install / uninstall          → cmd_systemd_install / cmd_systemd_uninstall
     ├── start / stop / restart       → cmd_systemctl_passthrough → systemctl --user
     ├── enable / disable / status    → cmd_systemctl_passthrough → systemctl --user
     ├── logs                         → cmd_logs → docker logs
     ├── serve                        → cmd_serve → docker run (managed start)
     ├── pull                         → cmd_pull → docker pull
     ├── update                       → cmd_update → docker pull + optional restart
     ├── check                        → cmd_check → python -m lucebox check
     ├── completion                   → cmd_completion
     ├── help / --help / -h           → usage
     ├── version / --version          → print VERSION
     └── * (catch-all)                → cmd_route_to_container
              │
              ├── container running   → docker exec lucebox python -m lucebox <cmd>
              └── container absent    → docker run --rm ... python -m lucebox <cmd>
```

Commands routed through the catch-all: `autotune`, `config`, `models`, `profile`, `smoke`, `print-run`, `print-serve-argv`, and all harness launchers (`claude`, `codex`, `opencode`, `hermes`, `pi`, `openclaw`).

---

## Command Reference

### Shell-layer commands (`lucebox.sh`)

These commands never reach the Python CLI.

---

#### `lucebox serve`

Start the inference server container.

```
lucebox serve
```

No flags. Behavior by current container state:

| Container state | Action |
|----------------|--------|
| `absent` | Start a new container with the active config |
| `running` or `restarting` | Error — redirect operator to `docker logs` or `lucebox stop` |
| `exited`, `created`, `paused`, `dead` | `docker rm -f` stale container, then start fresh |
| anything else | Warn, `docker rm -f`, then start fresh |

Exit: 0 on successful start; non-zero with a diagnostic on any failure.

---

#### `lucebox start` / `lucebox stop` / `lucebox restart`

```
lucebox start
lucebox stop
lucebox restart
```

Passed directly to `systemctl --user <action> lucebox.service`. On `start` and `restart`, polls `is-active` for up to 10 s; dumps `status` and journal on failure.

---

#### `lucebox enable` / `lucebox disable`

```
lucebox enable
lucebox disable
```

`exec systemctl --user <action> lucebox.service`. No polling.

---

#### `lucebox status`

```
lucebox status
```

`exec systemctl --user status lucebox.service --no-pager`. Shows unit state, not server metrics. For inference metrics, use `lucebox smoke` or `lucebox autotune --json`.

---

#### `lucebox logs`

```
lucebox logs [args...]
```

Forwards all arguments to `docker logs lucebox`. Supports the same flags as `docker logs` (e.g. `--follow`, `--tail N`).

---

#### `lucebox pull`

```
lucebox pull
```

Shell layer intercepts this and calls `docker pull` for the configured image tag directly. Also available as a Python CLI command (see below) when running inside the container context.

---

#### `lucebox update`

```
lucebox update
```

Pulls the latest Docker image for the active `LUCEBOX_IMAGE:LUCEBOX_VARIANT` tag. Does not restart a running server; the operator must restart separately to apply the update. Reports current and new image digests.

---

#### `lucebox install` / `lucebox uninstall`

```
lucebox install
lucebox uninstall
```

Register or remove the `lucebox.service` systemd unit. `install` also enables the service.

---

#### `lucebox completion`

```
lucebox completion
```

Emit shell completion script. Shell detected from `$SHELL`. Output is suitable for `eval` or sourcing. (P2 — basic implementation present.)

---

### Python CLI commands (`lucebox` Typer app)

Invoked via the catch-all dispatch or directly as `python -m lucebox <cmd>` inside the container.

---

#### `lucebox check`

Report host readiness against all six prerequisites.

```
lucebox check
```

No flags. Evaluates and prints pass/fail for:

| Check | Pass condition |
|-------|---------------|
| Docker daemon | `has_docker = True` |
| NVIDIA driver | `driver_major ≥ 525` |
| NVIDIA Container Toolkit | `ctk` in `{"runtime", "cdi"}` |
| GPU VRAM | `vram_gb ≥ 24` |
| System RAM | `ram_gb ≥ 32` (floor TBD — see open questions) |
| systemd | `has_systemd = True` |

On the Strix Halo + RTX 3090 reference profile all checks pass and a throughput estimate is printed.

**Exit codes**: 0 = all checks pass; 1 = one or more checks fail.

**Output**: Human-readable by default. Each failing check names the deficiency and references a fix path.

---

#### `lucebox autotune`

Print or persist optimal DFlash runtime configuration for the detected hardware.

```
lucebox autotune [--apply] [--json] [--force] [--sweep] [--yes|-y]
                 [--profile <name>] [--list-profiles]
```

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--apply` | bool | False | Write the selected config to `config.toml`; without this, only prints |
| `--json` | bool | False | Emit config as JSON instead of human-readable table |
| `--force` | bool | False | Overwrite existing `config.toml` values without prompting |
| `--sweep` | bool | False | Run empirical VRAM-tier bracket sweep instead of heuristic profile selection |
| `--yes` / `-y` | bool | False | Skip interactive confirmation before writing |
| `--profile` | str | `"heuristic"` | Named profile to apply; `--list-profiles` shows available names |
| `--list-profiles` | bool | False | Print available profile names and exit |

Without `--sweep`: selects a named profile based on detected `vram_gb`. With `--sweep`: runs live inference at each quantization tier bracket and selects the highest-fidelity tier that sustains throughput above the target threshold.

**DFlash keys written by `--apply`** (the 11-field snapshot-allowlist — these map to `dflash.*` in `config.toml`):

| Key | Type | Default |
|-----|------|---------|
| `dflash.budget` | int | 22 |
| `dflash.max_ctx` | int | 16384 |
| `dflash.lazy` | bool | False |
| `dflash.prefix_cache_slots` | int | 0 |
| `dflash.prefill_cache_slots` | int | 0 |
| `dflash.cache_type_k` | str | `""` |
| `dflash.cache_type_v` | str | `""` |
| `dflash.prefill_mode` | `"off"\|"auto"\|"always"` | `"off"` |
| `dflash.prefill_keep_ratio` | float | 0.05 |
| `dflash.prefill_threshold` | int | 32000 |
| `dflash.prefill_drafter` | str | `""` |

**Exit codes**: 0 = success; 1 = hardware detection failed or no profile matches.

---

#### `lucebox config get [key]`

```
lucebox config get [key]
```

| Argument | Default | Description |
|----------|---------|-------------|
| `key` | `""` (empty) | Dotted key to read; if empty, prints all known keys and their effective values |

Reports the effective value and which layer supplied it: `env`, `config.toml`, or `default`.

**Exit codes**: 0 = success; 1 = unrecognized key.

---

#### `lucebox config set key=value`

```
lucebox config set <key=value>
```

`key=value` is a single positional argument with the `=` separator. Writes the value to `config.toml`. Does not affect a running server — change takes effect on next `lucebox serve`.

Emits a warning if a running server is detected and the key is not hot-reloadable.

**Exit codes**: 0 = written; 1 = unrecognized key or type error.

---

#### `lucebox config unset key`

```
lucebox config unset <key>
```

Removes the key from `config.toml`, reverting to the dataclass default on next start.

**Exit codes**: 0 = removed; 1 = key not present in `config.toml` or unrecognized.

---

#### `lucebox models` / `lucebox models list`

```
lucebox models
lucebox models list
```

List all installed model presets with name, quantization level, and disk usage. `lucebox models` (no subcommand) invokes `list` by default.

Output columns: `PRESET`, `TARGET_FILE`, `DRAFT_FILE`. JSON output: flag TBD (see open questions).

**Exit codes**: 0 = success; 1 = models directory not found or not readable.

---

#### `lucebox models download [preset] [--activate]`

```
lucebox models download [preset] [--activate]
```

| Argument / Flag | Default | Description |
|-----------------|---------|-------------|
| `preset` | `""` (empty) | Named preset from the Lucebox registry; if empty, lists available presets |
| `--activate` | False | Set the downloaded model as the active model after download completes |

Selects the highest-fidelity quantization that fits within available `vram_gb + ram_gb`. Reports selected quantization before downloading. Reports a structured error if no quantization fits.

**Note**: This is the model management command; `lucebox pull` (shell layer) is Docker image management, not model download. These are distinct operations.

**Exit codes**: 0 = downloaded (and activated if `--activate`); 1 = preset not found; 2 = server not reachable for activation; 3 = insufficient VRAM/RAM for any quantization.

---

#### `lucebox profile [--level] [--url]`

Run a luce-bench snapshot against the running server.

```
lucebox profile [--level <level>] [--url <url>]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--level` | `"level1"` | Benchmark level passed to luce-bench (`level1`, `level2`, etc.) |
| `--url` | `""` (uses configured port) | Override the server URL for the benchmark |

Writes results to a timestamped file in the configured output directory. Requires the `lucebench` package (`#337`) to be importable.

**Note on naming**: The PRD references `lucebox bench` for the throughput benchmark. The implemented Python command is `lucebox profile`. The shell wrapper's catch-all may alias `bench` to route to this command — this needs confirmation. See open questions.

**Exit codes**: 0 = benchmark completed and written; 1 = server not reachable; 2 = luce-bench not installed.

---

#### `lucebox smoke [--tools/--no-tools] [--timeout]`

Smoke-test the running server.

```
lucebox smoke [--tools] [--no-tools] [--timeout <seconds>]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--tools` / `--no-tools` | `--tools` | Whether to exercise tool-call routes in addition to basic completion |
| `--timeout` | 60.0 | Seconds before the test is considered failed |

Hits `/props` and `POST /v1/chat/completions` on the configured local endpoint. Exits 0 on a valid response from both; non-zero with a structured error on any failure.

**Exit codes**: 0 = all checks pass; 1 = server not reachable; 2 = invalid response schema.

---

#### `lucebox claude [--prompt] [--url] [--model]`

Launch Claude Code pointed at the local server.

```
lucebox claude [--prompt|-p <text>] [--url <url>] [--model <id>]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--prompt` / `-p` | None | Non-interactive prompt; if omitted, launches interactive TUI |
| `--url` | Configured port | Override server base URL |
| `--model` | `luce-dflash` | Model ID to pass to the client |

**Binary lookup order**: `$CLAUDE_BIN` env → `PATH` (shutil.which `claude`) → `$CLIENT_WORK_DIR/clients/claude_code/npm/bin/claude`. Fails with a clear error naming the missing binary if none found.

**Environment injected into child process**:

| Variable | Value |
|----------|-------|
| `ANTHROPIC_API_KEY` | `sk-lucebox` (or `--api-key` override) |
| `ANTHROPIC_BASE_URL` | server base URL |
| `CLAUDE_CODE_API_BASE_URL` | server base URL |
| `CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` | `1` |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | `1` |
| `CLAUDE_CODE_DISABLE_TELEMETRY` | `1` |

**Interactive mode** (no `--prompt`): `exec`s the claude binary, replacing the lucebox process. No lucebox wrapper remains running.

**Non-interactive mode** (`--prompt` supplied): Adds `--print --output-format json --model <id> --permission-mode dontAsk --no-session-persistence` to the child invocation. Returns exit code 124 on timeout (matches GNU `timeout` convention).

**Exit codes**: 0 = child exited 0; non-zero = child exit code; 124 = timeout; 127 = binary not found.

---

#### `lucebox codex [--prompt] [--url] [--model]`

```
lucebox codex [--prompt|-p <text>] [--url <url>] [--model <id>]
```

Same shape as `lucebox claude`. Routes to the OpenAI-compatible endpoint (not the Anthropic endpoint). Binary: `codex`.

---

#### `lucebox opencode [--prompt] [--url] [--model]`

```
lucebox opencode [--prompt|-p <text>] [--url <url>] [--model <id>]
```

Same shape. Binary: `opencode`.

---

#### `lucebox hermes [--prompt] [--url] [--model]`

```
lucebox hermes [--prompt|-p <text>] [--url <url>] [--model <id>]
```

Same shape. Binary: `hermes-agent`. Note: the Python module is `hermes.py` but the binary name is `hermes-agent`.

---

#### `lucebox pi [--prompt] [--url] [--model]`

```
lucebox pi [--prompt|-p <text>] [--url <url>] [--model <id>]
```

Same shape. Binary: `pi`.

---

#### `lucebox openclaw [--prompt] [--url] [--model]`

```
lucebox openclaw [--prompt|-p <text>] [--url <url>] [--model <id>]
```

Same shape. Binary: `openclaw`.

---

#### `lucebox version`

```
lucebox version
```

Print the installed `lucebox` package version string. Shell layer also handles `lucebox --version` (prints `$VERSION` from the wrapper without invoking Python).

---

### Internal commands (not for direct operator use)

| Command | Consumer |
|---------|----------|
| `lucebox print-run` | `lucebox.sh cmd_serve` — emits the full `docker run` argv |
| `lucebox print-serve-argv` | `lucebox.sh cmd_serve` — emits raw argv lines for container start |

---

## Environment Variable Registry

All variables are read by `_load_or_build()` at CLI startup (env overrides `config.toml` overrides dataclass defaults).

| Variable | Type | Default | Applies to |
|----------|------|---------|------------|
| `LUCEBOX_VARIANT` | str | `"cuda12"` | Docker image variant tag |
| `LUCEBOX_IMAGE` | str | `"ghcr.io/luce-org/lucebox-hub"` | Docker image name |
| `LUCEBOX_CONTAINER` | str | `"lucebox"` | Docker container name |
| `LUCEBOX_PORT` | int | `8080` | Inference server listening port |
| `LUCEBOX_MODELS` | path | `~/.local/share/lucebox/models` | Model storage directory |
| `CLAUDE_BIN` | path | — | Override claude binary location |
| `CLIENT_WORK_DIR` | path | — | Fallback binary search root for all harness clients |

Harness-specific binary overrides follow the pattern `<TOOL>_BIN` (e.g. `CODEX_BIN`, `OPENCODE_BIN`). Exact names are implementation-defined per client module — `CLAUDE_BIN` is confirmed; others need verification.

---

## Configuration Key Registry

Managed via `lucebox config get/set/unset`. Keys are stored as TOML in `~/.lucebox/config.toml` (or `$XDG_CONFIG_HOME/lucebox/config.toml` — exact path is implementation-defined; see open questions).

**Top-level keys**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `variant` | str | `"cuda12"` | Docker image variant |
| `image` | str | `"ghcr.io/luce-org/lucebox-hub"` | Docker image name |
| `container_name` | str | `"lucebox"` | Container name |
| `port` | int | `8080` | Server port |
| `models_dir` | path | `~/.local/share/lucebox/models` | Model directory |

**`model.*` keys** (active model selection)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `model.preset` | str | `""` | Active model preset name |
| `model.target_file` | str | `""` | Path to primary GGUF file |
| `model.draft_file` | str | `""` | Path to draft GGUF file for speculative decoding |

**`dflash.*` keys** (snapshot-allowlist — written by `autotune --apply`)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dflash.budget` | int | 22 | KV-cache budget (GB) |
| `dflash.max_ctx` | int | 16384 | Maximum context window tokens |
| `dflash.lazy` | bool | False | Lazy KV-cache allocation |
| `dflash.prefix_cache_slots` | int | 0 | Prefix cache slot count |
| `dflash.prefill_cache_slots` | int | 0 | Prefill cache slot count |
| `dflash.cache_type_k` | str | `""` | K-cache quantization type |
| `dflash.cache_type_v` | str | `""` | V-cache quantization type |
| `dflash.prefill_mode` | `"off"\|"auto"\|"always"` | `"off"` | PFlash prefill mode |
| `dflash.prefill_keep_ratio` | float | 0.05 | PFlash keep ratio |
| `dflash.prefill_threshold` | int | 32000 | PFlash activation threshold (tokens) |
| `dflash.prefill_drafter` | str | `""` | Draft model path for PFlash |

**`dflash.*` keys** (outside snapshot allowlist — not written by autotune, manually configurable)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dflash.think_max` | int | 15488 | Maximum thinking token budget |
| `dflash.fa_window` | int | 0 | Flash attention window size |
| `dflash.think_soft_close_min_ratio` | float | 0.0 | Thinking soft-close minimum ratio |
| `dflash.debug_thinking_logits` | bool | False | Enable thinking logit debug output |

---

## Exit Code Contract

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Resource not found / check failed / config key unrecognized |
| 2 | Server not reachable (connection refused or timeout at network layer) |
| 3 | Insufficient resources (VRAM/RAM) — `models download` only |
| 124 | Timeout — harness non-interactive mode (matches GNU `timeout`) |
| 127 | Binary not found — harness commands only |

Non-zero exits always emit a structured error to stderr. Raw Python tracebacks are not permitted in the operator-facing output.

---

## Data Types

### `HostFacts`

Populated by `host_facts.from_env()` and probed at `check` time.

| Field | Type | Default |
|-------|------|---------|
| `nproc` | int | 0 |
| `ram_gb` | int | 0 |
| `gpu_vendor` | `"nvidia"\|"amd"\|"none"` | `"none"` |
| `gpu_name` | str | `""` |
| `gpu_count` | int | 0 |
| `vram_gb` | int | 0 |
| `gpu_sm` | str | `""` |
| `driver_version` | str | `""` |
| `driver_major` | int | 0 |
| `has_systemd` | bool | False |
| `is_wsl` | bool | False |
| `has_docker` | bool | False |
| `docker_version` | str | `""` |
| `ctk` | `"runtime"\|"cdi"\|"installed-unwired"\|"none"` | `"none"` |

### `ModelMeta`

| Field | Type | Default |
|-------|------|---------|
| `preset` | str | `""` |
| `target_file` | str | `""` |
| `draft_file` | str | `""` |

---

## Discrepancies with Feature Specs

These are implementation-to-spec gaps identified during authoring. Resolution belongs in the feature specs or in a follow-up PR.

| Spec reference | Spec says | Implementation has | Disposition |
|----------------|-----------|-------------------|-------------|
| PRD P0 item 8; FEAT-004 BNV-03 | `lucebox bench` | `lucebox profile` (Python CLI) | `bench` may be a `lucebox.sh` alias or `harness/bench.py` entrypoint — confirm dispatch and align spec |
| FEAT-005 ADAPT-04 | binary name `hermes-agent` | Python module `hermes.py`; actual binary name needs verification from client code | Verify binary name in `hermes.py` |
| FEAT-004 MDL-04 | model list with `--json` flag | `models list` has no `--json` flag documented in current CLI | Add `--json` to `models list` or update spec |
| FEAT-003 CHK-01, FEAT-004 MDL-02 | minimum system RAM floor | `ram_gb ≥ 32` used in `HostFacts` — exact floor not confirmed as a spec requirement | Confirm 32 GB floor and close open question in PRD |
| concerns.md `auth-local-sessions` | password set on first boot | No auth surface visible in CLI | Auth is web UI scope (FEAT-005 WEB area); confirm CLI has no auth commands |
| FEAT-004 CFG-05 | `LUCEBOX_IMAGE`, `LUCEBOX_VARIANT`, `LUCEBOX_PORT`, `LUCEBOX_CONTAINER`, `LUCEBOX_MODELS` | All five confirmed | Aligned |

---

## Open Questions

- [ ] **`config.toml` path**: exact location (`~/.lucebox/config.toml` vs. `$XDG_CONFIG_HOME/lucebox/config.toml`) — needed for documentation and for operators who manually edit.
- [ ] **`lucebox bench` dispatch**: does `lucebox.sh` catch-all route `bench` to `lucebox profile`, to `harness/bench.py`, or to `run_lucebench.sh`? Confirm and align PRD/FEAT-004 command name.
- [ ] **Harness binary env var names**: only `CLAUDE_BIN` confirmed. Need the pattern for `codex`, `opencode`, `hermes-agent`, `pi`, `openclaw` (e.g. `CODEX_BIN`?).
- [ ] **`models list --json`**: FEAT-004 OBS-04 and MDL-04 require machine-parseable output. Does `models list` support `--json`? If not, it needs adding before the feature ships.
- [ ] **System RAM floor for `lucebox check`**: `HostFacts.ram_gb` is detected; is 32 GB the documented minimum, or is it VRAM-only? PRD open question still unresolved.
- [ ] **ADR-004**: static asset serving for the web UI will require a new static-file serving surface in the server. This TD covers the CLI only; web UI API contract is a separate TD.

---

## Dependencies

- **PR #334 (docker-stack)**: `docker_run.py` shells out to `docker run` against the `lucebox-hub` image. CLI cannot exercise serve paths without this.
- **PR #336 (server-layer-split)**: `autotune` candidate configs assume layer-split build flags. Without #336, configs reduce to no-ops on Strix Halo + RTX 3090 reference hardware.
- **PR #337 (luce-bench)**: `lucebox profile` invokes `python -m lucebench.cli`. Without #337, profile/bench operations are unavailable.
- **FEAT-002 (Inference Server)**: `/props` and `/v1/chat/completions` must be reachable for `smoke` and `profile` to function.
