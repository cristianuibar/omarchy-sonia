# Coding Conventions

**Analysis Date:** 2026-09-12

<!-- refreshed: 2026-09-12 -->

## Naming Patterns

**Files:**
- Python package: `snake_case` modules under `src/omarchy_voice/` (`realtime.py`, `tools.py`)
- Tests: `tests/test_<area>.py` (not collocated)
- CLI wrappers: kebab-case `omarchy-voice-*` under `omarchy/bin/` and `bin/`
- QML plugins: PascalCase widgets (`VoiceOrb.qml`, `VoiceIndicator.qml`) plus `manifest.json`

**Functions:**
- `snake_case` everywhere
- CLI handlers: `cmd_<subcommand>` (`cmd_say`, `cmd_listen`, `cmd_run`)
- Private helpers: leading `_` (`_runtime_dir`, `_matches`, `_normalize`)
- No `async_` prefix; async is `IsolatedAsyncioTestCase` / `async def` on the realtime path

**Variables:**
- `snake_case` locals
- Module constants: `UPPER_SNAKE_CASE` (`SOCKET_PATH`, `DEFAULT_DENY`, `READ_ONLY_TOOLS`)
- Config/env paths derived from XDG (`CONFIG_HOME`, `RUNTIME_DIR`)

**Types:**
- `@dataclass` for records (`Config`, `Policy`, `Result`)
- Named exceptions: `Denied`, `NeedsConfirmation` (not `*Error` suffix)
- No TypeScript; Python 3.x with `from __future__ import annotations`

## Code Style

**Formatting:**
- No Black/Ruff/Prettier config in-repo; match surrounding files
- Double quotes for strings (docstrings triple-double)
- 4-space indent
- Type hints on public functions (`-> int`, `Path`, `list[str]`)
- Section banners: `# --- commands ---------------------------------------------------------------`

**Linting:**
- No project ESLint/ruff config
- Occasional `# noqa: BLE001` on bench tools that must not abort a sweep
- Gate before push: `python3 -m unittest discover -s tests` (README)

## Import Organization

**Order:**
1. `from __future__ import annotations`
2. stdlib (`argparse`, `json`, `os`, `pathlib`, …)
3. blank line
4. relative package imports (`.config`, `.tools`, `.session`)

**Grouping:**
- Tests: stdlib, then `sys.path.insert` to `src/`, then `from omarchy_voice ...`
- No path aliases; package is `omarchy_voice`

## Error Handling

**Patterns:**
- Policy: raise `Denied` / `NeedsConfirmation`; callers convert to `Result(ok=False)` or hold pending work
- CLI: catch `(ConnectionError, OSError)`, print `error: ...` to stderr, return `1`
- Control socket: `PermissionError` if runtime dir is not mode 700
- Swallow only when documented (`except OSError: pass` on chmod; env file missing returns empty warnings)

**Error Types:**
- Throw on invariant / safety (untrusted mic → deny/confirm)
- Return `Result` / CLI exit codes for expected user-facing failures
- `cmd_say` / `cmd_listen` return `0`/`1`; no traceback for missing daemon

## Logging

**Framework:**
- Session log file (`STATE_DIR / "session.log"`), not a logging library
- User-facing: stdout with ANSI when TTY (`_bold`, `_tick`); stderr for errors
- Feedback: notifications, `state.json` for bar widgets, optional TTS (`piper` / `espeak-ng`)

**Patterns:**
- Atomic writes: write `.tmp` then `replace` for state files
- Level file separate from status so bar watchers are not woken every audio frame

## Comments

**When to Comment:**
- Module docstring states role (“the hands”, “the mouth”, Unix socket purpose)
- Safety rationale next to deny/confirm lists and runtime-dir 700 check
- Why not `exec omarchy-voice` in bash wrappers (PATH recursion)

**TODO:**
- Rare; prefer tests that encode past session-log mistakes (`test_compose.py`)

## Function Design

**Size:**
- CLI commands stay short; desktop surface lives in `tools.py` (large module, still grouped by concern)
- Guard clauses and early returns (`if not args.words: return 1`)

**Parameters:**
- `config: Config` passed explicitly; avoid globals except path constants
- CLI: argparse `args` plus loaded `config`

**Return Values:**
- Commands return `int` exit codes
- Tools return `Result(ok, output)`
- Policy `check` returns `None` or raises

## Module Design

**Exports:**
- Named exports; `__init__.py` only `__version__`
- Entry: `python3 -m omarchy_voice` via `__main__.py` → `cli`
- Bash wrappers set `PYTHONPATH` to installed `src` then `exec python3 -m omarchy_voice ...`
- Omarchy metadata in comments: `# omarchy:summary=`, `# omarchy:group=voice`

**Shell:**
- `set -euo pipefail` on wrappers
- Never `exec omarchy-voice` from a binary also named `omarchy-voice`

---

*Convention analysis: 2026-09-12*
*Update when patterns change*
