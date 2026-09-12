# Codebase Structure

**Analysis Date:** 2026-09-12

<!-- refreshed: 2026-09-12 -->

## Directory Layout

```
omarchy-voice/
├── bin/                    # Checkout launcher
│   └── omarchy-voice
├── src/omarchy_voice/      # Python package (all runtime logic)
├── omarchy/bin/            # `omarchy voice …` route wrappers
├── omarchy/upstream/       # Installer snippet for Omarchy packaging
├── plugin/                 # Omarchy shell QML bar widgets
│   ├── voice.indicator/
│   └── voice.orb/
├── share/                  # Unit, example config, echo-cancel, bindings snippet
├── tests/                  # unittest modules (policy, realtime, keys, …)
├── tools/                  # bench_realtime.py
├── docs/                   # omarchy-voice.html
├── install.sh              # Local install
├── uninstall.sh
├── README.md
└── HANDOFF.md              # Maintainer notes
```

## Directory Purposes

**src/omarchy_voice/**
- Purpose: Entire application
- Contains: Python modules
- Key files: `cli.py`, `realtime.py`, `tools.py`, `planner.py`, `capabilities.py`, `session.py`, `config.py`, `feedback.py`, `persona.py`, `keys.py`, `__main__.py`
- Subdirectories: none

**omarchy/bin/**
- Purpose: One executable per Omarchy CLI route (comment metadata for `omarchy commands`)
- Contains: bash wrappers that `exec python3 -m omarchy_voice <subcommand>`
- Key files: `omarchy-voice` (status), `-toggle`, `-start`, `-stop`, `-confirm`, `-cancel`, `-say`, `-doctor`, `-log`, `-manifest`

**plugin/**
- Purpose: Bar widgets
- Contains: `manifest.json` + QML
- Key files: `VoiceIndicator.qml`, `VoiceOrb.qml`

**share/**
- Purpose: Files the installer copies
- Contains: `omarchy-voice.service`, `config.example.toml`, `echo-cancel.conf`, `bindings.lua.snippet`

**tests/**
- Purpose: Offline unit tests
- Contains: `test_*.py` (compose, config, keys, policy, reach, realtime, realtime_wire, terminal, web)

**bin/**
- Purpose: Run from git without install (`PYTHONPATH` via `sys.path`)

## Key File Locations

**Entry Points:**
- `bin/omarchy-voice` — repo CLI
- `src/omarchy_voice/__main__.py` — installed/wrapper CLI (`python3 -m omarchy_voice`)
- `src/omarchy_voice/cli.py` — argparse + command handlers
- `share/omarchy-voice.service` — user unit (`…/omarchy-voice run`)

**Configuration:**
- `share/config.example.toml` — shipped defaults template
- Runtime: `~/.config/omarchy-voice/config.toml`, `env`, `safety-id` (not in repo)

**Core Logic:**
- `realtime.py` — speech daemon
- `planner.py` — typed `say`
- `tools.py` — Executor / Policy / tool schemas
- `capabilities.py` — live manifest
- `session.py` — Unix control socket
- `feedback.py` — state.json / notifications
- `keys.py` — keysym/mod normalisation

**Testing:**
- `tests/test_*.py`
- `tools/bench_realtime.py`

**Documentation:**
- `README.md`, `HANDOFF.md`, `docs/omarchy-voice.html`

## Naming Conventions

**Files:**
- Python: snake_case modules (`realtime.py`)
- Tests: `test_<area>.py`
- Omarchy wrappers: `omarchy-voice[-action]`
- QML plugins: `voice.<id>/` + PascalCase `.qml`

**Directories:**
- kebab-case top-level (`omarchy-voice` package dir is snake_case for Python)

**Special Patterns:**
- `__main__.py` required so wrappers never `exec omarchy-voice` (wrapper name collision)
- `# omarchy:summary=` / `group=` / `examples=` comments on Omarchy bin scripts

## Where to Add New Code

**New CLI subcommand:**
- Parser + `cmd_*` in `src/omarchy_voice/cli.py`
- Optional wrapper in `omarchy/bin/`
- Tests under `tests/`

**New model tool:**
- Schema + handler in `tools.py`; policy description strings there
- Tests in `test_policy.py` / domain tests (`test_web.py`, `test_terminal.py`)

**New capability source:**
- `capabilities.py` + cache key inputs; `omarchy-voice manifest --refresh`

**New bar UI:**
- `plugin/voice.<id>/` with `manifest.json` + QML; read `state.json` / `level`

**Utilities:**
- Keep helpers next to the domain module; no `utils/` package today

## Special Directories

**Runtime (not in repo):**
- `$XDG_RUNTIME_DIR/omarchy-voice/` — socket, state, level
- `~/.local/state/omarchy-voice/` — session.log
- `~/.cache/omarchy-voice/` — manifest cache
- Installed copy: `~/.local/share/omarchy-voice/` (`OMARCHY_VOICE_SRC`)

**.planning/codebase/**
- Purpose: BUM maps
- Source: codebase-mapper
- Committed: as the project chooses

---

*Structure analysis: 2026-09-12*
*Update when directory structure changes*
