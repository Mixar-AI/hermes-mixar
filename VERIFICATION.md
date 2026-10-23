# Verification

Run on 2026-10-22 on macOS (Apple Silicon) with Mixar 4.3.5 (`/Applications/Mixar.app`) signed in with 0 credits, and Hermes Agent v0.22.4 installed by the official installer (checkout `4d04303818`). Lifecycle, gate, state and timeout tests used separate throwaway `HERMES_HOME` folders; live tests used the default profile and the running Mixar. Runtime tests call Hermes's own MCP code (`discover_mcp_tools`, the tool registry) without a model. Tests ran from plugin commit `4af0f50` to `9879d8c`; the commits in between changed only the skill and documentation.

## Results

### Packaging

| Check | Result |
|---|---|
| `hermes plugins validate <repo> --install-deps` | Passed: manifest, config schema, server availability (`available, version 4.3.5`), security scan `safe`, "portable plugin (no Python)" |
| `hermes plugins doctor <repo>` | Passed |
| `scripts/validate_plugin_catalog.py` on `docs/catalog-entry.yaml` | `OK: 1 file(s) valid` |
| `scripts/scan_plugin_payloads.py <repo>` | Passed |
| Catalog CI on Linux | `validate` reports availability as informational, so the macOS-only declaration does not fail the Linux runner |

### Lifecycle (real Hermes CLI, throwaway home)

| Check | Result |
|---|---|
| Install `--no-enable`; the disabled package exposes no server | Pass |
| Enable: server present, `timeout` 610 s | Pass |
| Disable removes it; enable restores it | Pass |
| Second install without `--force` | Refused ("already exists") |
| `--force` reinstall keeps `bin/mixar-mcp` executable | Pass |
| A same-name `mcp_servers.mixar` in `config.yaml` replaces the package entry | Pass |
| A pinned install refuses `plugins update`; `install --force --ref <new sha>` moves it (0.1.0 → 0.1.1) | Pass |
| `plugins remove` deletes the plugin and its server | Pass |
| Dashboard (`hermes dashboard`) lists `mixar` with `runtime_status: enabled` | Pass |

### Install gates

| Check | Result |
|---|---|
| Mixar.app missing | Install refused: `unavailable: missing_app` |
| Mixar older than `min_version` | Install refused: `unavailable: version_too_old` |
| Mixar 4.3.5 present | Installs |

### Launcher shim

| Check | Result |
|---|---|
| Launcher missing | Exit 1, nothing on stdout, stderr points to Connect MCP |
| `HOME` unset | Exit 1, "HOME is not set" |
| `HOME` without Mixar | Names the expected launcher path |
| `HOME` with spaces; arguments | Resolved; passed through |
| Process | `exec` replaces the shim (same PID) |

### Launcher states through Hermes (fake connector folders; no Mixar was started)

| State | Hermes result |
|---|---|
| MCP turned off | 80 tools listed from the saved list; calls answer "MCP is turned off in Mixar…" |
| Mixar closed and not startable | "Mixar is not open, and no installed Mixar could be started" |
| Never connected (no saved list) | Only 15 local tools; `mixar_status` is not listed |
| Update in progress | "Mixar is starting or unavailable" |
| Launcher missing | 0 tools, "Connection closed"; the shim's message is in `logs/mcp-stderr.log` |

### Live Mixar through Hermes

| Check | Result |
|---|---|
| Discovery | 77 tools in about 2 s; names ≤ 64 characters, no duplicates, every schema an object |
| Every read-only tool that needs no arguments (26) | All routed; no transport failure |
| New scene tab, build a crate, read back, render, reopen the render by image ref | Pass; Hermes saved the render under `~/.hermes/cache/images` |
| Export a Godot `.glb` to Downloads | Valid glTF, `verification_passed` reported (test file deleted) |
| Syntax error, blocked `import os`, unknown object, wrongly typed argument | Clear tool errors, no crash |
| `mixar_call_status` for the build call | Outcome recovered |
| Close a tab holding new objects | Refused without `contents: "discard"`; with it the tab closes and the connection re-pins to the original tab, which is unchanged |
| Price quotes | `create_layered_material` 1 credit; `enqueue_generation` `existing_job_queue`; scripts 0 |
| `world_labs` without `user_requested` | Refused before queueing |
| `image_gen` at 0 credits | `dispatched: true`, then the job fails with "This generation uses credits — get a subscription to continue"; balance stays 0 |
| Reload (shutdown and rediscover) | Tools and live calls restored |
| Mixar quits, then is reopened by hand | `mixar_ui_context(instance)` with the instance named in the error re-pins the same Hermes session |
| Tool-call timeout | A 330-second call answered through the package (610 s); the same call through a server on Hermes's default timed out at 300 s |
| Tool names in `SKILL.md`, `references/setup.md`, `README.md` | All exist in the live list |

### Windows and Linux

| Check | Result |
|---|---|
| Hermes parses the `darwin`, `win32` and `linux` app blocks (`hermes_platform.declaration`) | Pass; `validate` unchanged |
| The Linux block evaluated by Hermes's resolver on this machine | `mixar` not on PATH, launcher path found: available |
| `.github/workflows/launcher-shim.yml` script run locally on macOS with the MCP SDK Hermes pins (2.3.0) | Pass: initialize, tools/list and tools/call through `bin/mixar-mcp` to a stub launcher in a home folder with a space; missing launcher exits 1 with an empty stdout |
| The same workflow on `macos-latest`, `windows-latest` and `ubuntu-latest` runners ([run 39890262818](https://github.com/Mixar-AI/hermes-mixar/actions/runs/39890262818), commit `a01c4cf`) | Pass on all three; on Windows the SDK resolved `./bin/mixar-mcp` to `bin\mixar-mcp.CMD` |
| Windows with a real Mixar, installed from GitHub (checked by the maintainer on 2026-10-22) | Reported working |

## Found and fixed during testing

- Before Mixar has ever connected, `mixar_status` is not listed: the skill falls back to `mixar_ui_context`.
- A launcher that cannot start shows only "Connection closed" in Hermes: the skill and setup point to `logs/mcp-stderr.log`.
- A queued job at 0 credits reports `dispatched: true` and fails later: the skill reports it instead of resubmitting.
- Hermes pauses a server for 60 s after three failed calls in a row (hard-coded; any tool; script errors count), which blocked even the recovery call: the skill never retries more than twice in a row.
- A Mixar that quits mid-session is not reopened by the pinned connection: the skill asks the user to open it and re-pins with `mixar_ui_context`.

## Observations outside this plugin

- Mixar launcher: after the Mixar a connection was pinned to quits, calls answer "Mixar is starting or unavailable" although nothing is starting; once Mixar is reopened the message is accurate.
- Hermes: the per-server breaker cannot be tuned from a plugin.

## Not executed

- Model-driven Hermes sessions (`hermes -z` with the skill): no model provider was signed in on this machine.
- A successful paid generation: the account had 0 credits.
- Signed out, and several signed-in Mixar apps open at once.
- Linux with a real Mixar: only the shim against a stub launcher runs there (CI); a live Mixar session through Hermes on Linux is untested.
- The catalog's clean pinned-source admission job, which needs the repository on GitHub at the pinned commit.
