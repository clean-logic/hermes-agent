# Phase 2 — Interactive prompt methods

> Read `00-BACKGROUND.md` first. Depends on Phase 1 (callback server, `_make_button`,
> `_register_action`/`_pop_action`, `_handle_callback` dispatch, and the
> `_resolve_*` stubs).

## Goal

Implement the four `send_*` methods on `MattermostAdapter` and fill in the four
`_resolve_*` methods stubbed in Phase 1. Each method:

- Returns `SendResult(success=False)` immediately when `not self._buttons_enabled()`
  (so the gateway uses its text fallback).
- Posts a Mattermost interactive message with the right buttons (built via
  `_make_button`, ids bare-alphanumeric).
- Registers pending state via `_register_action({...})` AFTER the post is confirmed
  live (so we never leave dangling state if the post fails).
- On click, `_handle_callback` pops the state and calls the matching `_resolve_*`,
  which invokes the correct core resolver and returns the post-update label.

All four mirror the **Discord** plugin's methods/views
(`plugins/platforms/discord/adapter.py`). The signatures below MUST match what the
gateway calls (taken from Discord, which the gateway already drives).

## Method signatures (match exactly)

```python
async def send_exec_approval(
    self, chat_id: str, command: str, session_key: str,
    description: str = "dangerous command", metadata: Optional[dict] = None,
) -> SendResult: ...

async def send_slash_confirm(
    self, chat_id: str, title: str, message: str, session_key: str,
    confirm_id: str, metadata: Optional[dict] = None,
) -> SendResult: ...

async def send_clarify(
    self, chat_id: str, question: str, choices: Optional[list], clarify_id: str,
    session_key: str, metadata: Optional[Dict[str, Any]] = None,
) -> SendResult: ...

async def send_update_prompt(
    self, chat_id: str, prompt: str, default: str = "",
    session_key: str = "", metadata: Optional[Dict[str, Any]] = None,
) -> SendResult: ...
```

## Choice strings & resolvers (authoritative)

| Surface | Button labels → choice | Resolver call |
|---|---|---|
| approval | Allow Once→`once`, Allow Session→`session`, Always Allow→`always`, Deny→`deny` | `resolve_gateway_approval(session_key, choice, resolve_all=False)` |
| slash | Approve Once→`once`, Always Approve→`always`, Cancel→`cancel` | `await tools.slash_confirm.resolve(session_key, confirm_id, choice)` (returns optional follow-up text) |
| update | Yes→`y`, No→`n` | write answer to `get_hermes_home()/".update_response"` (atomic tmp+replace) |
| clarify | one button per choice → that choice's text; "Other" → text-capture | numeric: `resolve_gateway_clarify(clarify_id, choice_text)`; other: `mark_awaiting_text(clarify_id)` |

## Reference: the fork's previous exec-approval implementation (the format to follow)

Retrieve with `git show 85b08f405:gateway/platforms/mattermost.py`. Its
`send_exec_approval`, `_approval_button`, and `_handle_approval_action` are the gold
reference. Key excerpts (adapt to the Phase-1 generic infra — note the previous code
used `MATTERMOST_CALLBACK_URL` directly and a webhook route; you instead use
`_buttons_enabled()`, `_make_button`, `_register_action`, and the `/hermes-callback`
route):

```python
# Posting the approval card (previous code):
attachment = {
    "fallback": "Command approval required",
    "callback_id": "hermes_approval",
    "text": f"```\n{cmd_preview}\n```\nReason: {description}",
    "actions": [ ...four buttons... ],
}
payload = {
    "channel_id": chat_id,
    "message": "\u26a0\ufe0f Command approval required",
    "props": {"attachments": [attachment]},
}
resp = await self._api_post("posts", payload)
post_id = resp.get("id", "")
# treat missing post_id as failure -> SendResult(success=False) (text fallback)

# Click handler returned a post-update + ephemeral toast:
return web.json_response({
    "update": {"message": label_map[choice], "props": {}},
    "ephemeral_text": "Approval recorded.",
})
```

`cmd_preview = command[:3800] + "..." if len(command) > 3800 else command`. Use real
`\n` newlines in the `text` (not the literal two-character `\\n`).

## Implementation pattern (apply to all four)

```python
async def send_exec_approval(self, chat_id, command, session_key,
                             description="dangerous command", metadata=None) -> SendResult:
    if not self._buttons_enabled():
        return SendResult(success=False, error="Mattermost callback server not running")
    cmd_preview = command[:3800] + "..." if len(command) > 3800 else command
    # Pre-generate the opaque id so all buttons share it.
    action_id = secrets.token_urlsafe(16)
    actions = [
        self._make_button("approveonce", "Allow Once", "once",
                          {"action_id": action_id, "choice": "once"}, style="primary"),
        self._make_button("approvesession", "Allow Session", "session",
                          {"action_id": action_id, "choice": "session"}),
        self._make_button("approvealways", "Always Allow", "always",
                          {"action_id": action_id, "choice": "always"}),
        self._make_button("deny", "Deny", "deny",
                          {"action_id": action_id, "choice": "deny"}, style="danger"),
    ]
    payload = {
        "channel_id": chat_id,
        "message": "\u26a0\ufe0f Command approval required",
        "props": {"attachments": [{
            "fallback": "Command approval required",
            "callback_id": "hermes_callback",
            "text": f"```\n{cmd_preview}\n```\nReason: {description}",
            "actions": actions,
        }]},
    }
    try:
        resp = await self._api_post("posts", payload)
        post_id = resp.get("id", "") if isinstance(resp, dict) else ""
        if not post_id:
            return SendResult(success=False, error="Mattermost API returned no post_id", raw_response=resp)
        # Register only after the card is confirmed live.
        self._pending_actions[action_id] = {"kind": "approval", "session_key": session_key}
        return SendResult(success=True, message_id=post_id, raw_response=resp)
    except Exception as exc:
        logger.error("[Mattermost] send_exec_approval failed: %s", exc, exc_info=True)
        return SendResult(success=False, error=str(exc))
```

Note: above pre-generates `action_id` and stores state directly (instead of calling
`_register_action`) so the id is shared across all four buttons. Either approach is
fine as long as the four buttons embed the SAME `action_id` and state is registered
only after `post_id` is confirmed.

### `_resolve_approval` (fill in the Phase-1 stub)

```python
async def _resolve_approval(self, payload, context, user_id) -> str:
    choice = context.get("choice", "")
    if choice not in {"once", "session", "always", "deny"}:
        raise ValueError(f"invalid approval choice: {choice!r}")
    from tools.approval import resolve_gateway_approval
    resolve_gateway_approval(payload["session_key"], choice, resolve_all=False)
    user = await self._lookup_username(user_id) if hasattr(self, "_lookup_username") else user_id
    return {
        "once": f"\u2705 Approved once by {user}",
        "session": f"\u2705 Approved for session by {user}",
        "always": f"\u2705 Approved permanently by {user}",
        "deny": f"\u274c Denied by {user}",
    }[choice]
```

(If `_lookup_username` doesn't exist on the current adapter, either add a small helper
that GETs `users/{id}` via `_api_get` and returns the username, or just use `user_id`.)

### slash confirm

`send_slash_confirm`: 3 buttons (`approveonce`/Approve Once/`once`,
`approvealways`/Always Approve/`always`, `cancel`/Cancel/`cancel`); message body =
`message`, title shown in attachment. State payload:
`{"kind": "slash", "session_key": ..., "confirm_id": confirm_id}`.

```python
async def _resolve_slash(self, payload, context, user_id) -> str:
    choice = context.get("choice", "")
    if choice not in {"once", "always", "cancel"}:
        raise ValueError(f"invalid slash choice: {choice!r}")
    from tools import slash_confirm as _sc
    follow_up = await _sc.resolve(payload["session_key"], payload["confirm_id"], choice)
    if follow_up:
        try:
            await self._api_post("posts", {"channel_id": context.get("channel_id") or payload.get("channel_id"), "message": follow_up})
        except Exception:
            pass
    return {"once": "\u2705 Approved once", "always": "\u2705 Always approved", "cancel": "\u274c Cancelled"}[choice]
```

Store `channel_id` in the payload at send time so the follow-up can be posted.

### update prompt

`send_update_prompt`: 2 buttons (`updateyes`/Yes/`y`, `updateno`/No/`n`); body =
`prompt` (+ ` (default: {default})` if default). State payload `{"kind": "update"}`.

```python
async def _resolve_update(self, payload, context, user_id) -> str:
    answer = context.get("choice", "")
    if answer not in {"y", "n"}:
        raise ValueError(f"invalid update answer: {answer!r}")
    from hermes_constants import get_hermes_home
    path = get_hermes_home() / ".update_response"
    tmp = path.with_suffix(".tmp")
    tmp.write_text(answer)
    tmp.replace(path)
    return "\u2705 Yes" if answer == "y" else "\u274c No"
```

### clarify

`send_clarify`: when `choices` is non-empty, render one button per choice (cap ~20;
button ids must be bare alphanumeric, e.g. `clarify0`, `clarify1`, ...) plus an
`clarifyother` "Other (type answer)" button. When `choices` is empty/None, post the
question as plain text (no buttons) and return `SendResult(success=True)` (the
gateway's text-intercept captures the next message). State payload
`{"kind": "clarify", "clarify_id": clarify_id, "choices": [...]}`. Embed the choice
index in each button's context (`{"action_id": action_id, "choice_index": i}`), and
`{"action_id": action_id, "choice_index": -1}` for Other.

```python
async def _resolve_clarify(self, payload, context, user_id) -> str:
    idx = context.get("choice_index", None)
    clarify_id = payload["clarify_id"]
    if idx == -1:
        from tools.clarify_gateway import mark_awaiting_text
        mark_awaiting_text(clarify_id)
        return "\u270f\ufe0f Awaiting your typed answer..."
    choices = payload.get("choices") or []
    if not isinstance(idx, int) or not (0 <= idx < len(choices)):
        raise ValueError(f"invalid clarify index: {idx!r}")
    choice_text = choices[idx]
    from tools.clarify_gateway import resolve_gateway_clarify
    resolve_gateway_clarify(clarify_id, choice_text)
    return f"\u2705 Answered: {choice_text}"
```

## Important rules

- Real `\n` newlines (NOT `\\n`) in all attachment `text`.
- All button `id`s bare alphanumeric (no `_`/`-`).
- Register pending state only AFTER the post is confirmed (has a `post_id`).
- Each `send_*` returns `SendResult(success=False)` when `not self._buttons_enabled()`.
- Update the Phase-1 `_handle_callback` only if needed (it already dispatches by
  `kind` and pops state); the four `_resolve_*` are where the real work goes.

## Acceptance

- `python -c "import plugins.platforms.mattermost.adapter"` clean.
- Simulated unit behavior (covered fully in Phase 4, but sanity-check here):
  - Each `send_*` with buttons disabled → `SendResult(success=False)`.
  - With a mocked `_api_post` returning `{"id": "p1"}`, each `send_*` posts the right
    button set and registers exactly one pending `action_id`.
  - Feeding a fake callback body (`{user_id, context:{action_id, choice/choice_index}}`)
    into `_handle_callback` calls the correct resolver once and pops the state;
    a second identical call is a no-op ("already resolved").
  - Unauthorized `user_id` (with `MATTERMOST_ALLOWED_USERS` set) → 403, state intact.

## Output to report

- The four methods + four resolvers added.
- Any signature deviation from the table above (should be none).
