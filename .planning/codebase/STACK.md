# Technology Stack

**Analysis Date:** 2026-09-12
<!-- refreshed: 2026-09-12 -->

## Languages

**Primary:**
- Python 3 (stdlib `tomllib`, `asyncio`, dataclasses) — daemon, CLI, policy gate, planner, realtime client (`src/omarchy_voice/`)

**Secondary:**
- Bash — `install.sh`, `uninstall.sh`, `omarchy/bin/omarchy-voice*` wrappers, `omarchy/upstream/omarchy-install-service-voice`
- QML — Omarchy shell plugins (`plugin/voice.orb/`, `plugin/voice.indicator/`)
- Lua snippet — Hyprland keybinding fragment (`share/bindings.lua.snippet`)
- TOML — user config (`share/config.example.toml` → `~/.config/omarchy-voice/config.toml`)
- systemd unit — `share/omarchy-voice.service`

## Runtime

**Environment:**
- CPython 3 via `#!/usr/bin/env python3` (`bin/omarchy-voice`)
- No venv in repo; install copies sources to `~/.local/share/omarchy-voice` and PATH-links `~/.local/bin/omarchy-voice`
- Graphical session: Hyprland 0.56+ / Omarchy 4.x (Lua dispatch API)
- Audio: PipeWire (`pw-record` capture, `pw-cat` playback), 24 kHz PCM16

**Package Manager:**
- Arch `pacman` (`python-websockets` from `extra`); no `pyproject.toml` / pip lockfile
- Optional: `omarchy pkg` / `pacman` for host tools (`wtype`, notify, piper/espeak)

## Frameworks

**Core:**
- None (stdlib CLI + asyncio websocket client)
- Qt/QML via Omarchy shell plugin loader (not a standalone Qt app)

**Testing:**
- `unittest` in `tests/` (`python -m unittest`); subprocess/hyprctl mocked
- Wire tests skip if `python-websockets` missing (`tests/test_realtime_wire.py`)

**Build/Dev:**
- No compiler/bundler; install is copy + chmod + systemd user enable
- `tools/bench_realtime.py` — Realtime API latency bench

## Key Dependencies

**Critical:**
- `websockets` (Arch `python-websockets`) — OpenAI Realtime `wss://api.openai.com/v1/realtime` (`websockets.asyncio.client`; fallback `legacy` extra_headers)
- Python stdlib `urllib.request` — Chat Completions HTTPS for `omarchy-voice say`
- Python stdlib `tomllib` — config

**Infrastructure (host binaries, not PyPI):**
- `hyprctl` — Hyprland query/dispatch (`hl.dsp.*`)
- `omarchy` CLI — command catalog + `omarchy voice` wrappers
- `pw-record` / `pw-cat` — mic/speakers
- `wtype` — keyboard injection
- `uwsm-app` — app launch
- `notify-send` — desktop notifications
- Optional: `piper` / `espeak-ng` — extra TTS (`[mouth] speak`)
- Optional: `tmux` — terminal watch/read tools

## Configuration

**Environment:**
- `~/.config/omarchy-voice/env` (mode 600) — `OPENAI_API_KEY` for systemd `EnvironmentFile` and CLI (`config.load_env_file`); process env wins
- `OPENAI_API_KEY` (name overridable via `api_key_env`)
- XDG: `XDG_CONFIG_HOME`, `XDG_CACHE_HOME`, `XDG_STATE_HOME`, `XDG_RUNTIME_DIR` (control socket; never world-writable `/tmp`)

**Files:**
- `~/.config/omarchy-voice/config.toml` — `[openai]`, `[ears]`, `[realtime]`, `[hands]`, `[mouth]`
- Runtime: `$XDG_RUNTIME_DIR/omarchy-voice/{control.sock,state.json,level}`
- Logs: `~/.local/state/omarchy-voice/session.log`
- Hyprland: installer patches `~/.config/hypr/bindings.lua`

**Build:**
- No `pyproject.toml` / `requirements.txt`

## Platform Requirements

**Development:**
- Linux (Omarchy/Arch + Hyprland); not portable to macOS/Windows as a product
- `python-websockets`, `hyprctl`, PipeWire tools for full daemon tests

**Production:**
- User-local install (`~/.local/share/omarchy-voice`) + systemd `--user` unit `WantedBy=graphical-session.target`
- Omarchy 4.x, Hyprland 0.56+, microphone PipeWire can see
- Optional bar plugins under `~/.config/omarchy/plugins`

---

*Stack analysis: 2026-09-12*
*Update after major dependency changes*
