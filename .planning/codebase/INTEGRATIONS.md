# External Integrations

**Analysis Date:** 2026-09-12
<!-- refreshed: 2026-09-12 -->

## APIs & External Services

**OpenAI Realtime API**
- Speech-in / speech-out daemon (`src/omarchy_voice/realtime.py`)
- Transport: WebSocket `wss://api.openai.com/v1/realtime` (`python-websockets`)
- Auth: `Authorization: Bearer $OPENAI_API_KEY` plus OpenAI Realtime protocol headers
- Models (config `[realtime]`): default `gpt-realtime-2.1`, voice `marin`, `semantic_vad`, input transcription `gpt-4o-mini-transcribe`
- Audio: 24 kHz PCM16 frames from `pw-record` to the socket; assistant audio to `pw-cat`
- Behavior: reconnect (6 attempts, exponential backoff), rate-limit retries on TPM failures
- Privacy: while listening, room audio streams continuously; mute stops `pw-record` rather than discard-after-capture

**OpenAI Chat Completions**
- Typed path `omarchy-voice say` (`src/omarchy_voice/planner.py`)
- Transport: HTTPS `https://api.openai.com/v1/chat/completions` via `urllib.request` (60s timeout)
- Auth: Bearer from `api_key_env` (default `OPENAI_API_KEY`)
- Model: `[openai] planner_model` default `gpt-4.1`; same tool schemas and policy gate as Realtime
- Rate limits: documented in README against platform.openai.com (org-tier; not encoded in client)

**No other cloud APIs.** No Stripe, email, Sentry, analytics SDKs.

## Data Storage

**Databases:**
- None

**File Storage (local only):**
- Config: `~/.config/omarchy-voice/config.toml`, `env` (keys), `safety-id`
- Cache: `~/.cache/omarchy-voice` (capability manifest hashing)
- State: `~/.local/state/omarchy-voice/session.log`
- Runtime: `$XDG_RUNTIME_DIR/omarchy-voice/` — Unix control socket, `state.json`, `level` (orb meter)

**Caching:**
- In-process / disk hash of desktop capability snapshot so Realtime prefix cache stays stable; no Redis

## Authentication & Identity

**Auth Provider:**
- OpenAI API key only (no OAuth, no user accounts)
- Location: `~/.config/omarchy-voice/env` (chmod 600; installer creates empty) and/or process environment
- systemd user unit: `EnvironmentFile=-%h/.config/omarchy-voice/env` — a shell-exported key does **not** reach the daemon
- Control socket: unauthenticated local IPC; directory must not be world-writable (`dir_is_private`)

**OAuth Integrations:**
- None

## Local desktop integrations (not SaaS)

These are subprocess CLIs the tools/policy layer drives:

| Binary / API | Role |
|---|---|
| `hyprctl` / Lua `hl.dsp.*` | windows, workspaces, monitors, binds; stubs from `/usr/share/hypr/stubs/hl.meta.lua` |
| `omarchy` | `omarchy commands --json`, themes, pkg, etc.; wrappers as `omarchy voice …` |
| `.desktop` files | installed apps for launch/compose |
| `wtype` | keystrokes |
| `uwsm-app` | launch apps |
| PipeWire `pw-record` / `pw-cat` | mic / speakers; optional `share/echo-cancel.conf` |
| `notify-send` | `[mouth] notify` |
| `piper` / `espeak-ng` | optional TTS when `[mouth] speak` |
| `tmux` | list/read/watch terminal panes |
| Omarchy shell QML plugins | `voice.orb`, `voice.indicator` poll runtime state files |

Policy (`tools.Policy`): deny/confirm regexes before any mutating call; `allow_shell` default false; `--dry-run` still runs read-only `hypr_query`.

## Monitoring & Observability

**Error Tracking:** none (no Sentry)

**Analytics:** none

**Logs:**
- File: `~/.local/state/omarchy-voice/session.log` (`omarchy-voice log -f`)
- systemd journal for the user unit
- `omarchy-voice doctor` / `status --json` for local health (websockets, key, PipeWire)

## CI/CD & Deployment

**Hosting:**
- Local user install only; not a hosted service
- Target: Omarchy 4.x laptop session

**CI Pipeline:**
- No `.github/workflows` in this tree (as mapped)
- Tests: local `unittest`

## Environment Configuration

**Development:**
- Required: `OPENAI_API_KEY` in env file or environment
- Optional: `config.toml` (all keys have defaults)
- Mocks: tests patch `subprocess` / skip Realtime without `websockets`
- `--dry-run` narrates mutating tools

**Staging:**
- None

**Production:**
- Same machine as development (user session)
- Secrets: `~/.config/omarchy-voice/env` only — never commit keys
- systemd: `Restart=on-failure` with start-rate limit so missing key/websockets does not loop forever

## Webhooks & Callbacks

**Incoming:** none (no HTTP server; Unix control socket for toggle/confirm/cancel/quit)

**Outgoing:**
- OpenAI Realtime WS + Chat Completions HTTPS only
- No third-party webhooks

---

*Integration audit: 2026-09-12*
*Update when adding/removing external services*
