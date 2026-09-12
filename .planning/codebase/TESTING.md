# Testing Patterns

**Analysis Date:** 2026-09-12

<!-- refreshed: 2026-09-12 -->

## Test Framework

**Runner:**
- Python stdlib `unittest` (`unittest.TestCase`, `unittest.IsolatedAsyncioTestCase`)
- No pytest, no pyproject test extra, no coverage.ini

**Assertion Library:**
- `unittest` assertions: `assertEqual`, `assertIn`, `assertNotIn`, `assertTrue`/`False`, `assertRaises`, `assertIsNone`
- `subTest` for parameterized cases (policy actions, tools, layouts)

**Run Commands:**
```bash
python3 -m unittest discover -s tests          # all tests (README gate before push)
python3 -m unittest tests.test_policy          # one module
python3 tests/test_config.py                   # file as script (`if __name__ == "__main__"`)
```

No watch mode and no coverage script in-repo.

## Test File Organization

**Location:**
- Separate tree: `tests/` at repo root
- Source under `src/omarchy_voice/` (not on default path)

**Naming:**
- `test_<domain>.py`: `test_policy`, `test_config`, `test_keys`, `test_web`, `test_terminal`, `test_compose`, `test_reach`, `test_realtime`, `test_realtime_wire`

**Structure:**
```
src/omarchy_voice/     # package under test
tests/
  test_policy.py       # Policy, confirmation matching, Executor gates
  test_config.py       # TOML load, retired keys, pattern union
  test_realtime.py     # tool schema conversion, confirmation gate
  test_realtime_wire.py
  test_web.py / test_terminal.py / test_keys.py / test_compose.py / test_reach.py
tools/bench_realtime.py  # not a unittest; live sweep helper
```

Every test file starts with:
```python
sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "src"))
from omarchy_voice ...
```

## Test Structure

**Suite Organization:**
```python
class PolicyTests(unittest.TestCase):
    def setUp(self):
        self.policy = Policy(Config())

    def test_destructive_actions_are_denied(self):
        for action in ["rm -rf ~/Documents", "dd if=/dev/zero of=/dev/sda"]:
            with self.subTest(action=action):
                with self.assertRaises(Denied):
                    self.policy.check(action)
```

**Patterns:**
- Class names: `*Tests` (`ConfigLoadTests`, `SpokenKeyNameTests`)
- Method names: `test_<behavior_in_plain_english>` (`test_unknown_keys_are_kept_for_doctor`)
- `setUp` for policy/executor; `TemporaryDirectory` + `addCleanup` for files
- Async realtime: `IsolatedAsyncioTestCase` + `unittest.mock.patch.object` on `feedback` path constants
- Comments explain *why* a case exists (retired `ears.mode = "push"`)

## Mocking

**Framework:**
- `unittest.mock` (`mock.patch`, `mock.patch.object`)
- In-test fakes: `FakeSocket` (records JSON), stub `Executor` with `launched`/`closed` lists

**What to Mock:**
- Filesystem paths for config, logs, sockets (`LOG_FILE`, `STATE_FILE`, `RUNTIME_DIR`)
- Subprocess / `omarchy launch` / Hyprland (`hyprctl`) — never hit a live desktop
- WebSocket send path (`FakeSocket.sent`)
- Time / audio / OpenAI network — tests must not require a mic or API key

**What NOT to Mock:**
- `Policy.check` regexes (real `Config` defaults)
- TOML load and pattern union
- Key normalisation, URL encoding, layout plans (pure functions)

## Fixtures and Factories

**Test Data:**
- Inline TOML via helper `write(self, text) -> Path` using `tempfile.TemporaryDirectory`
- Default `Config()` for policy unless the case is about custom patterns
- Fake window ids (`0xfeed`, `0xbad`) and pane targets (`Work:1.1`)

**Location:**
- Helpers live in the test class, not `tests/fixtures/`
- Regression corpus: comments pointing at session-log mistakes in `test_compose.py`

## Coverage

**Requirements:**
- No numeric coverage gate
- Safety-critical paths are required: deny/confirm, confirm-phrase matching (no substring `"don't confirm"`), realtime tool schema shape, spoken confirmation not bypassable by the model
- README: unittest discover is the pre-push gate

**Configuration:**
- None (no coverage.py config)

## Test Types

**Unit Tests:**
- Policy, config, keys, URL/query encoding, layout plans
- Fast, no network

**Integration-style (still unittest):**
- `Executor` with mocked launch/close; terminal pane resolution; web search/open_page
- Realtime session object with patched files + fake socket

**E2E / live:**
- Not in `tests/`; `tools/bench_realtime.py` is a manual/live sweep
- No Playwright; QML widgets untested by unittest

## Common Patterns

**Error testing:**
```python
with self.assertRaises(Denied):
    self.policy.check("rm -rf ~/Documents")
with self.assertRaises(NeedsConfirmation):
    self.policy.check("omarchy update")
```

**Must-not-raise:**
```python
self.policy.check(action)  # ordinary actions pass
json.dumps(self.tools)     # must not raise
```

**Deny beats confirm:**
- If both lists match, assert `Denied`, not hold-for-confirm

**Async:**
```python
class RealtimeSessionTests(unittest.IsolatedAsyncioTestCase):
    async def test_...(self):
        ...
```

**Snapshot testing:**
- Not used; assert exact lists/strings

---

*Testing analysis: 2026-09-12*
*Update when test patterns change*
