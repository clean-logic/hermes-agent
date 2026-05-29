# Phase 4 — Tests & validation

> Read `00-BACKGROUND.md` first. Depends on Phases 1-3.

## Goal

Replace the stale test file with comprehensive, hermetic tests covering all four
interactive surfaces, the callback router, authorization, the double-click guard, and
the text-fallback path. Then get the Mattermost test subset (ideally the whole suite)
green.

## Files

- EDIT/REWRITE: `tests/gateway/test_mattermost_approval_buttons.py` (it currently
  imports the deleted `gateway.platforms.mattermost` and fails).
- Optionally add `tests/gateway/test_mattermost_interactive.py` for the new surfaces
  if you prefer to keep the file names focused — either is fine; keep it under
  `tests/gateway/`.

## The one import fix that's mandatory

The old test starts with:

```python
from gateway.platforms.mattermost import MattermostAdapter   # BROKEN now
```

Change to the plugin path (the same pattern as
`tests/gateway/test_google_chat.py`, which imports
`from plugins.platforms.google_chat.adapter import ...`):

```python
from plugins.platforms.mattermost.adapter import MattermostAdapter
from gateway.config import Platform, PlatformConfig
```

Retrieve the old test for salvageable assertions:
`git show 85b08f405:tests/gateway/test_mattermost_approval_buttons.py`. Its
`_make_adapter()` / `_make_request()` helpers and the exec-approval card assertions
are reusable once repointed at the plugin and the new
`_make_button`/`_handle_callback`/`_pending_actions` API.

## Test conventions (AGENTS.md)

- stdlib + `pytest` + `unittest.mock` only. **No network.** Mock `_api_post` /
  `_api_get`.
- Use `pytest.mark.asyncio` or `asyncio.run(...)` for the async methods (match how
  other gateway tests do it — check `tests/gateway/test_google_chat.py`).
- Do NOT write change-detector tests (no asserting full button-list snapshots that
  would break on a label tweak). Assert behavior and relationships.
- Don't write to `~/.hermes`; the autouse `_isolate_hermes_home` fixture in
  `tests/conftest.py` redirects `HERMES_HOME` to a temp dir — rely on it for the
  update-prompt `.update_response` test.

## Coverage checklist

Build an adapter via a helper like:

```python
def _make_adapter(buttons_enabled=True):
    cfg = PlatformConfig(enabled=True, token="test-token")
    a = MattermostAdapter(cfg)
    a._base_url = "http://mm.local"
    a._bot_user_id = "bot"
    a._session = MagicMock()
    a._runner = object() if buttons_enabled else None   # _buttons_enabled() -> bool
    return a
```

1. **Button format:** `_make_button("approveonce","Allow Once","once",{"action_id":"x"})`
   has `name`, `integration.url`, `integration.context`, bare-alphanumeric `id`, and
   NO Slack-style `text`/`value` keys.
2. **Fallback:** with `buttons_enabled=False`, each of the four `send_*` returns
   `SendResult(success=False)` and posts nothing.
3. **Send + register:** with `_api_post` mocked to return `{"id": "p1"}`, each `send_*`
   returns `success=True, message_id="p1"` and registers exactly one entry in
   `_pending_actions` with the right `kind`.
4. **Post failure:** `_api_post` returns `{}` (no id) → `SendResult(success=False)`
   and `_pending_actions` stays empty.
5. **Approval resolution:** patch `tools.approval.resolve_gateway_approval`; feed a
   callback body `{"user_id":"u","context":{"action_id":<id>,"choice":"once"}}` into
   `_handle_callback`; assert the resolver was called once with
   `(session_key,"once",resolve_all=False)` and the response JSON has an `update`.
6. **Double-click guard:** a second identical `_handle_callback` call returns the
   "already resolved" ephemeral and does NOT call the resolver again.
7. **Auth:** set `MATTERMOST_ALLOWED_USERS="alice"`; a callback from `user_id="bob"`
   returns HTTP 403 and leaves `_pending_actions` intact (token not consumed).
8. **Slash confirm:** patch `tools.slash_confirm.resolve` (AsyncMock); assert it's
   awaited with `(session_key, confirm_id, "once"/"always"/"cancel")`.
9. **Update prompt:** click "Yes"/"No"; assert `get_hermes_home()/".update_response"`
   contains `"y"`/`"n"`.
10. **Clarify:** multi-choice click index 1 → `resolve_gateway_clarify(clarify_id,
    choices[1])`; "Other" (index -1) → `mark_awaiting_text(clarify_id)`; empty choices
    → text post, `success=True`, no buttons.

## Acceptance

```powershell
cd "c:\Users\Anton\github\clean-logic\hermes-agent"
# bash wrapper if available (preferred, CI-parity):
bash scripts/run_tests.sh tests/gateway/ -k mattermost -q
# Windows fallback:
python -m pytest tests/gateway/ -k mattermost -q -n 4
```

All Mattermost tests green. Before handing off for review, run the broader gateway
suite to catch regressions:

```powershell
python -m pytest tests/gateway/ -q -n 4
```

## Output to report

- Test file(s) touched and the count of test cases.
- The green test run output (tail).
- Any coverage item from the checklist you could not implement and why.
