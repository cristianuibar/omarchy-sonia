# Architecture

**Analysis Date:** 2026-09-12

<!-- refreshed: 2026-09-12 -->

## Pattern Overview

**Overall:** Local Python CLI daemon that routes speech (or typed text) through OpenAI, then executes Omarchy/Hyprland tools behind a policy gate.

**Key Characteristics:**
- Single Python package (`omarchy_voice`) with argparse subcommands
- Two model paths share the same tools and policy: Realtime websocket (speech) and Chat Completions (typed `say`)
- No local ASR; mute = kill `pw-record`, not capture-and-discard
- File + Unix-socket state for bar widgets and keybindings
- Capability manifest is read from the live Hyprland/Omarchy install, not hardcoded APIs

## Layers

**CLI / wrappers:**
- Purpose: Parse args, load env/config, route to handlers
- Contains: `cli.py`, `bin/omarchy-voice`, `omarchy/bin/omarchy-voice-*`
- Depends on: config, session, planner, realtime, capabilities
- Used by: user, systemd unit, Hyprland bindings, Omarchy `omarchy voice …` routes

**Session control:**
- Purpose: Talk to a running daemon without going through the model
- Contains: `session.py` (`ControlServer`, `send_control`, confirm/cancel phrase matching)
- Depends on: config paths (`SOCKET_PATH`, `RUNTIME_DIR` mode 700)
- Used by: `listen *` CLI, keybindings, realtime handler

**Intelligence (model adapters):**
- Purpose: Turn user intent into function calls + spoken/text reply
- Contains: `realtime.py` (websocket, PipeWire I/O, barge-in), `planner.py` (HTTPS Chat Completions)
- Depends on: persona, capabilities, tools, feedback, session
- Used by: `run` and `say`

**Policy + hands:**
- Purpose: The only code that may mutate the desktop
- Contains: `tools.py` (`Policy`, `Executor`, `TOOL_SCHEMAS`), `keys.py`
- Depends on: capabilities (live queries), config deny/confirm patterns
- Used by: realtime and planner

**World model:**
- Purpose: Tell the model *this* Hyprland and *this* Omarchy
- Contains: `capabilities.py` (stub parse, `omarchy commands --json`, hyprctl, desktop files, cache)
- Depends on: host files under `/usr/share/hypr`, `/usr/share/omarchy`
- Used by: both model adapters and `manifest`

**Feedback / UI:**
- Purpose: Status for humans (notifications, bar, optional local TTS)
- Contains: `feedback.py`; QML plugins under `plugin/`
- Depends on: runtime state files
- Used by: realtime session; bar widgets poll JSON

**Install / packaging:**
- Purpose: Drop binaries, service, bindings, env file
- Contains: `install.sh`, `uninstall.sh`, `share/*`, `omarchy/upstream/`
- Depends on: none of the Python layers at runtime

## Data Flow

**Realtime listen (daemon):**

1. systemd/`omarchy-voice run` → `cli.cmd_run` → `realtime.run`
2. Daemon starts muted; `ControlServer` binds `control.sock`
3. `SUPER + SHIFT + V` / `listen toggle` starts `pw-record` → PCM frames over Realtime websocket
4. Model emits audio (`pw-cat`) and `function_call` items
5. `Executor` runs tools through `Policy` (deny / hold / execute)
6. Held actions wait for local `listen confirm` / `cancel` (not a transcript)
7. `Feedback` writes `state.json` + `level` for the bar; session log under XDG state

**Typed `say`:**

1. `omarchy-voice say "…"` → `Planner.think` (Chat Completions + same `TOOL_SCHEMAS`)
2. Same `Executor` / confirm prompt on TTY
3. No websocket, no microphone

**State Management:**
- Config: `~/.config/omarchy-voice/config.toml` + mode-600 `env`
- Runtime: `$XDG_RUNTIME_DIR/omarchy-voice/` (`control.sock`, `state.json`, `level`)
- Cache: capability manifest under XDG cache, keyed on host versions
- In-process: pending confirm, websocket, PipeWire child processes

## Key Abstractions

**Executor + Policy:**
- Purpose: Gate every tool call; hold destructive actions
- Examples: `Denied`, `NeedsConfirmation`, `Executor.run_pending`
- Pattern: Regex deny/confirm lists on action descriptions

**Capability manifest:**
- Purpose: Version-correct Hypr Lua + Omarchy CLI surface
- Examples: `capabilities.manifest()`, `live_state()`, `VOICE_GROUPS` allow-list
- Pattern: Cached document injected into system prompt

**Control socket:**
- Purpose: Unauthenticated-but-owner-only IPC for toggle/confirm
- Examples: `ControlServer`, `send_control`
- Pattern: Unix SOCK_STREAM, chmod 600, refuse if runtime dir not 700

**General tools, not one-per-verb:**
- Purpose: Survive Omarchy/Hyprland releases
- Examples: `hypr_query`, dispatch via `hl.dsp.*`, `omarchy` argv, `wtype`
- Pattern: Small schema list + live manifest for syntax

**RealtimeSession:**
- Purpose: Mic/speaker/websocket lifecycle, echo tail, reconnect
- Examples: `realtime.run`, mute = process lifetime
- Pattern: asyncio + thread pool for blocking tools

## Entry Points

**Dev launcher:**
- Location: `bin/omarchy-voice`
- Triggers: repo checkout
- Responsibilities: `sys.path` insert, `cli.main`

**Module entry (installed):**
- Location: `src/omarchy_voice/__main__.py`
- Triggers: `python3 -m omarchy_voice` from Omarchy wrappers
- Responsibilities: avoid recursive `omarchy-voice` PATH lookup

**Omarchy CLI routes:**
- Location: `omarchy/bin/omarchy-voice-*`
- Triggers: `omarchy voice …`
- Responsibilities: set `PYTHONPATH`, exec module with a subcommand

**User service:**
- Location: `share/omarchy-voice.service`
- Triggers: graphical session
- Responsibilities: `ExecStart=…/omarchy-voice run`, `EnvironmentFile=…/env`

**Bar plugins:**
- Location: `plugin/voice.indicator/`, `plugin/voice.orb/`
- Triggers: Omarchy shell
- Responsibilities: poll status; click toggle/confirm

## Error Handling

**Strategy:** Fail a turn, not the daemon. Policy exceptions become spoken/text replies. Control-handler exceptions are stringified back over the socket.

**Patterns:**
- Planner: `PlannerUnavailable` vs generic catch on one turn
- Realtime: reconnect loop, benign OpenAI error codes, rate-limit retries
- systemd: `Restart=on-failure` with start-limit (missing key/websockets is not transient)
- `doctor` enumerates missing pieces instead of crashing later

## Cross-Cutting Concerns

**Logging:**
- Session log via `feedback` / `omarchy-voice log`
- systemd journal for the daemon

**Validation:**
- Policy regexes; Hypr dispatch regex; desktop-id regex; key/mod normalisation
- Confirm phrases must be the whole utterance (fillers only), not substrings

**Authentication / secrets:**
- `OPENAI_API_KEY` from env file (CLI merge + systemd EnvironmentFile)
- Per-install `safety-id` for Realtime safety identifier
- No credentials in repo maps or examples beyond the env *path*

**Safety:**
- Allow-listed Omarchy command groups in the manifest
- Shell dispatchers (`exec*`) extra-gated
- `--dry-run` still runs read-only tools so the planner can see the desktop

---

*Architecture analysis: 2026-09-12*
*Update when major patterns change*
