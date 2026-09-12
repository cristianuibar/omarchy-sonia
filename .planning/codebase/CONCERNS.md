# Codebase Concerns

**Analysis Date:** 2026-09-12
<!-- refreshed: 2026-09-12 -->

Desktop voice daemon (`omarchy-voice`): OpenAI Realtime over WebSocket, Hyprland/Omarchy tools, systemd user service. An open microphone is an untrusted input channel.

## Tech Debt

**Prompt / token budget comments vs live API:**
- Issue: Comments and tests still talk about ~8–9k tokens per turn, cached-prefix TPM, and a 40k legacy bucket (`src/omarchy_voice/realtime.py`, `capabilities.py`, `tools.py` OCR_LIMIT, `tests/test_realtime.py`).
- Why: Real constraint when `gpt-4o-realtime` aliased a 40k TPM bucket.
- Impact: Over-trimming OCR (`OCR_LIMIT` 6000, `OUTPUT_LIMIT` 4000) and capability snapshots; HANDOFF says the 800k bucket is the live ceiling.
- Fix approach: Revisit limits as product choices, not TPM survival; keep logging `rate_limits.updated`.

**Chrome profile-error dialogs in compose:**
- Issue: `compose_windows` launches a new Chrome process per web pane; lock contention raises “Profile error occurred”. Code dismisses the dialog after each web pane (`tools.py` `_dismiss_browser_error_dialogs`).
- Why: Omarchy `omarchy launch webapp` starts a new process that should hand off; racing SQLite on one profile.
- Impact: Dialog steals focus and breaks layout preselect; workaround is fragile (match empty class + title).
- Fix approach: Serialise web launches or reuse one browser; upstream Omarchy launcher, not a local race.

**`ydotool` / uinput host setup:**
- Issue: Clicking needs `uinput` loaded, udev, and user `ydotool` service; `mousemove --absolute` is 2× on some displays so code uses `hl.dsp.cursor.move` then ydotool click only.
- Why: Arch ships user `ydotool`, not root `ydotoold`.
- Impact: Clicks fail or land wrong until host is primed; doctor/uinput hints in `tools.py`.
- Fix approach: Keep host docs; do not switch absolute mouse path without measuring.

**`rm` deny vs assistant-created files:**
- Issue: DEFAULT_DENY `\brm\s+-[a-zA-Z]*[rf]` (and related) blocks deleting a file the assistant just wrote.
- Why: Voice mishears must not recursive-delete.
- Impact: Recovery becomes rename/move; extra turns.
- Fix approach: Narrow deny (e.g. require `-r`/`/`/`~`) or allow `rm` of paths created in-session — high caution.

## Known Bugs

**Acoustic echo / self-barge-in:**
- Symptoms: TTS transcribed as user speech; phantom commands (`HANDOFF.md` Korean/Cyrillic fragments, `press CTRL+r`).
- Trigger: Speakers + same interface as mic, `barge_in = true`, no PipeWire AEC.
- Workaround: Default `barge_in = false` holds mic while `Speaker.write` duration + 350ms; `share/echo-cancel.conf`; `doctor` warns same-device barge-in (`realtime.py` ~1295).
- Root cause: Server VAD cannot tell TTS from user.
- Trade-off: Holding the mic disables mid-sentence interrupt on speakers.

**OCR click false positives:**
- Symptoms: Clicks prose that shares words with the target (“changed files” vs “Files changed”).
- Trigger: Text-heavy pages, `click_text`.
- Workaround: Require every query word (one slack on long phrases); `--psm 3` not 6; prefer `send_shortcut` / unique wording.
- Root cause: OCR match is lexical, not link vs body.

**`run_in_terminal` newlines:**
- Symptoms: Model retries heredocs; thirteen commands for a two-line file.
- Trigger: Newlines refused (would become Enter).
- Workaround: Refusal now names `printf '...\n...' > f`.
- Residual: Policy still blocks `rm` of the failed file.

**tmux watch idle race (fixed, regression-sensitive):**
- Symptoms: Job reported done in ~0.4s with echoed command as output.
- Trigger: `pane_current_command` still `bash` immediately after `send-keys`.
- Root cause: Must observe busy before idle; 0.6s grace. Infinite-busy watches used to skip age check.
- Tests: `tests/test_terminal.py`. Do not drop the busy-first rule.

## Security Considerations

**Voice as untrusted input:**
- Risk: Misheard speech runs Hyprland, terminal, or shell tools.
- Mitigation: `Policy` deny/confirm regex (`config.py` DEFAULT_DENY/CONFIRM); `allow_shell` default false; Hypr `exec*` dispatchers gated; tmux run only if client attached **and** terminal on a drawn workspace; confirm is local CLI/keybind, not model-trusted (`realtime.py` confirm path).
- Recommendations: Treat regex policy as last line, not a parser; keep `deny_patterns_replace` off unless intentional; never enable `allow_shell` without reviewing confirm list.

**Control socket:**
- Risk: Unauthenticated local remote-control of the desktop.
- Mitigation: Socket under `XDG_RUNTIME_DIR` (mode 700); never `/tmp` (`config.py` `_runtime_dir`).
- Recommendations: Do not relocate socket; keep filesystem perms.

**Secrets / env:**
- Risk: `OPENAI_API_KEY` in `~/.config/omarchy-voice/env`; world-readable file.
- Mitigation: `load_env_file` warns if group/other bits set; systemd `EnvironmentFile`.
- Recommendations: `chmod 600` on env and `safety-id`. Safety id is random, chmod 600, hashed for OpenAI grouping — not username@host.

**Screen OCR vs terminal capture:**
- Risk: `read_screen` sends a screenshot-derived text blob to OpenAI; may include password UI.
- Mitigation: Comments in `tools.py` note password-box OCR; prefer `read_terminal` (local tmux, no picture).
- Recommendations: Keep screen reads bounded; do not log full OCR to world-readable files.

**ydotool / uinput:**
- Risk: Injected input is full HID; group access is powerful.
- Mitigation: Distro udev + user service; project does not install a custom udev rule.
- Recommendations: Do not broaden uinput access in install scripts.

## Performance Bottlenecks

**Realtime turn size:**
- Problem: Persona + capability manifest + snapshot on each response create.
- Measurement: HANDOFF ~9,700 tokens/turn historically; cache prefix work reduced unchanged resend (~2,270 routes used to be pasted every turn).
- Cause: Desktop manifest is large by design.
- Improvement path: Snapshot on `speech_started` only; keep capability as cached prefix.

**Window compose:**
- Problem: Sequential pane wait up to `PANE_TIMEOUT` (web 12s) and `COMPOSE_BUDGET` 32s (`tools.py`).
- Cause: Must wait for map to place; Chrome cold start.
- Improvement path: Do not raise MAX_PANES (6) without budget; remaining panes launch without wait past budget.

**OCR / grim:**
- Problem: Full-screen grim + tesseract on the hot path for click/read.
- Cause: No accessibility tree for arbitrary apps.
- Improvement path: Prefer tmux for terminals; keep OCR_LIMIT.

## Fragile Areas

**Hyprland Lua dispatch / keysyms:**
- Files: `src/omarchy_voice/keys.py`, `tools.py` `_DISPATCH_RE`.
- Why: `Enter` is not a keysym (`Return`); Hyprland reports success; model trusts “ok”.
- Safe modification: Always `normalise_key`; refuse unresolved names.
- Tests: `tests/test_keys.py`.

**Policy regex:**
- File: `src/omarchy_voice/config.py`, `Policy.check`.
- Why: Description-string matching, not AST; `rm -rf` vs `rm file`; `sudo` substring.
- Safe modification: Add tests in `tests/test_policy.py` / `test_reach.py` for every new pattern.

**Audio subprocesses:**
- File: `realtime.py` mic/speaker, `feedback.py` piper/aplay/espeak.
- Why: pactl defaults, PipeWire device names, echo cancel module.
- Common failures: Same-box loopback; missing echo-cancel; hung `pw-cat`.
- Tests: Mostly mocked; live audio is doctor + log `mic held N frames`.

**QML indicators:**
- Files: `plugin/voice.indicator/`, `plugin/voice.orb/` — watch `state.json` / `level`.
- Why: Ten-times-a-second level file is split from status so watchers do not wake every frame.

## Scaling Limits

**Single-user desktop:**
- Current capacity: One systemd user daemon, one Realtime session.
- Limit: OpenAI TPM/org; one mic stream.
- Symptoms: `rate_limit_exceeded`, `response failed`.
- Scaling path: Not multi-user; do not share the control socket across users.

**Compose / tools:**
- MAX_PANES 6, COMPOSE_BUDGET 32s, OUTPUT_LIMIT 4000, OCR_LIMIT 6000.

## Dependencies at Risk

**OpenAI Realtime API / `gpt-realtime-2.1`:**
- Risk: Event names, header names, model aliases, TPM buckets change.
- Impact: Daemon is the product.
- Migration: `tests/test_realtime_wire.py`; websockets 14 `additional_headers` vs `extra_headers` (`_open_socket`).

**Omarchy / Hyprland / ydotool / tmux / grim / tesseract:**
- Risk: Upstream `omarchy launch`, portal Secret (PRs #9319/#9320), ydotool packaging.
- Impact: Tools assume these binaries and Hypr Lua `hl.dsp.*`.
- Migration: `capabilities.py` live manifest, not a frozen verb list.

**python-websockets on Arch:**
- Risk: Connect kwarg rename.
- Mitigation: Dual import path in `_open_socket`.

## Missing Critical Features

**True barge-in on speakers:**
- Problem: Interrupt while speaking without echo-as-command.
- Workaround: Headphones, AEC module, or hold-mic.
- Blocks: Natural conversation on laptop speakers.
- Complexity: High (AEC quality + VAD).

**Accessibility / hit-testing for clicks:**
- Problem: Clicks cannot distinguish link vs prose.
- Workaround: Unique phrases, shortcuts.
- Complexity: High (AT-SPI per toolkit).

## Test Coverage Gaps

**Live audio / PipeWire / ydotool:**
- What's not tested: Real echo, real click coordinates, udev.
- Risk: Host-only failures (uinput not loaded).
- Priority: Medium — doctor + HANDOFF.
- Difficulty: Needs hardware.

**Chrome compose race:**
- What's not tested: Profile-error dialog in CI.
- Priority: Low if dismisser stays covered by compose tests (`tests/test_compose.py`).

**Policy completeness:**
- What's not tested: Every deny against paraphrases the model might emit.
- Priority: High when adding tools.
- Coverage exists: `tests/test_policy.py`, `test_reach.py`, `test_web.py`, `test_terminal.py`, `test_realtime.py`.

---

*Concerns audit: 2026-09-12*
*Update as issues are fixed or new ones discovered*
