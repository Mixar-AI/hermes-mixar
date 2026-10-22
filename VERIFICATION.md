# Verification

## Executed

Run on 2026-10-22 on macOS (Apple Silicon) against Mixar 4.3.5 (`/Applications/Mixar.app`), with plugin commit `9901137` and Hermes Agent built from source at `6e8adc7a46` (`uv sync --frozen --extra mcp` into an isolated virtual environment). Every Hermes command used a separate, empty `HERMES_HOME`; no normal Hermes profile was read or changed.

| Check | Result |
|---|---|
| `hermes plugins validate <repo> --install-deps` | Passed: portable manifest, manifest fields, config schema, server availability (`mixar` available, version 4.3.5, `/Applications/Mixar.app`), security scan `safe`, isolation "portable plugin (no Python)" |
| `hermes plugins doctor <repo>` | Passed: discovery, manifest parsing, import and registration |
| `scripts/validate_plugin_catalog.py` on `docs/catalog-entry.yaml` (sha set to `9901137`) | `OK: 1 file(s) valid` |
| `scripts/scan_plugin_payloads.py <repo>` | Passed (no binaries, bytecode or archives) |
| `hermes plugins install file://<repo> --ref <sha> --no-enable`, then `hermes plugins enable mixar` | Installed pinned at `9901137`; `bin/mixar-mcp` kept mode 755 |
| Hermes's MCP config translation | `command` = the installed `bin/mixar-mcp`, `cwd` = plugin root, env `HOME`, `MIXAR_MCP_LAUNCHER`, `MIXAR_MCP_DISCOVERY_DIR` plus `PLUGIN_ROOT`/`PLUGIN_DATA`, `timeout` 610 s |
| Hermes's MCP discovery (`discover_mcp_tools`) | Connected over stdio in 1.8 s; 77 tools registered as `mcp__mixar__*` |
| Availability gate (`check_fn` of a Mixar tool) | Offered (Mixar.app present, 4.3.5 ≥ 4.2.1) |
| `mcp__mixar__mixar_status` through Hermes's tool registry | Returned the connected desktop's instance, scene session and capabilities |
| `skills_list` | Lists `agent-plugin-mixar-f619df08:mixar` |
| Direct stdio smoke through `bin/mixar-mcp` with only `PATH` and `HOME` set | `initialize` (protocol 2025-06-18, server instructions 2,046 characters), `tools/list` 73 tools, `mixar_status` succeeded |

Observed during the direct smoke: Mixar was closed when the first `mixar_status` call ran, and the launcher started the installed app, as the README discloses. That call answered "Mixar is starting or unavailable"; the same call succeeded once the app had signed in and connected. Listing tools did not start Mixar.

## Not executed

- A model-driven Hermes chat session. The isolated profile had no model provider, so the skill's instructions were not exercised by a model.
- A scene mutation (create, read back and delete a scratch object in a new scene tab). Only read-only tools were called, against the user's real Mixar session.
- Windows and Linux. The package declares macOS only; see the README for the Windows workaround.
- The catalog's clean pinned-source admission job, which needs the repository on GitHub at the pinned commit.
- Hermes releases other than the source commit above. `requires_hermes` is set to 0.22.1, the first release that honours a portable server's `timeout` (commit `fcc170b798` first ships in v0.22.1), from the Hermes commit history rather than a run on that release.
