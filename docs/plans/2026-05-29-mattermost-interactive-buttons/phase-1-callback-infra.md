# Phase 1 — Plugin callback infrastructure

> Read `00-BACKGROUND.md` first. Depends on Phase 0 being complete.

## Goal

Add all the shared plumbing for Mattermost interactive buttons to
`plugins/platforms/mattermost/adapter.py`, with **no prompt methods yet** (those are
Phase 2). Deliverables:

1. Config resolution for the callback server (host/port/url).
2. A plugin-owned aiohttp server started in `connect()` and torn down in
   `disconnect()`, serving a single route `POST /hermes-callback`.
3. A pending-state registry (opaque id → payload) with an atomic-pop double-click
   guard.
4. The callback handler `_handle_callback` that authorizes the clicker, validates,
   pops state, and **dispatches by `context["kind"]`** to per-kind resolver hooks
   (the hooks themselves are stubbed here; Phase 2 fills them in).
5. Helpers: `_make_button(...)` (Mattermost action format) and `_buttons_enabled()`.

## Files

- EDIT: `plugins/platforms/mattermost/adapter.py` only.

## Current integration points (already in the file — inspect before editing)

`MattermostAdapter.__init__` (around line 74) sets `self._base_url`, `self._token`,
`self._session` (aiohttp ClientSession), `self._dedup`, etc. `connect()` (~line 195)
opens the session, authenticates (`users/me`), sets `self._bot_user_id`, and starts
the websocket loop. `disconnect()` (~line 229) cancels tasks and closes the session.
There is an `async def _api_post(self, path, payload)` HTTP helper and an
`async def _api_put(...)`; reuse `_api_post("posts", {...})` to create posts and the
REST API `posts/{id}/patch` (via `_api_put`) to edit them.

## 1. `__init__` additions

```python
import secrets  # ensure imported at top of file

# in __init__:
self._callback_host = (
    config.extra.get("callback_host")
    or os.getenv("MATTERMOST_CALLBACK_HOST")
    or "127.0.0.1"
)
try:
    self._callback_port = int(
        config.extra.get("callback_port")
        or os.getenv("MATTERMOST_CALLBACK_PORT")
        or 18065
    )
except (TypeError, ValueError):
    self._callback_port = 18065
# Explicit external URL wins (cross-host). Otherwise derive from host:port.
self._callback_url = (os.getenv("MATTERMOST_CALLBACK_URL", "") or "").strip()

self._runner = None          # aiohttp.web.AppRunner
# Pending interactive prompts: opaque id -> dict payload.
# payload schema: {"kind": "approval"|"slash"|"update"|"clarify", plus kind fields}
self._pending_actions: Dict[str, Dict[str, Any]] = {}
```

## 2. `_buttons_enabled()` + effective callback URL

```python
def _buttons_enabled(self) -> bool:
    """Interactive buttons require a reachable callback server."""
    return self._runner is not None

def _effective_callback_url(self) -> str:
    if self._callback_url:
        return self._callback_url
    return f"http://{self._callback_host}:{self._callback_port}/hermes-callback"
```

## 3. Server lifecycle — extend `connect()` / `disconnect()`

Model on the Teams plugin (`plugins/platforms/teams/adapter.py` connect/disconnect):

```python
# Teams reference (do not copy verbatim — adapt):
self._runner = web.AppRunner(aiohttp_app)
await self._runner.setup()
site = web.TCPSite(self._runner, "0.0.0.0", self._port)
await site.start()
# disconnect:
if self._runner:
    await self._runner.cleanup()
    self._runner = None
```

In Mattermost's `connect()`, AFTER the existing successful authentication and BEFORE
returning True, start the callback server (best-effort — a bind failure must NOT kill
the adapter; just log and leave buttons disabled so the text fallback applies):

```python
try:
    from aiohttp import web
    app = web.Application()
    app.router.add_post("/hermes-callback", self._handle_callback)
    app.router.add_get("/health", lambda _req: web.Response(text="ok"))
    self._runner = web.AppRunner(app)
    await self._runner.setup()
    site = web.TCPSite(self._runner, self._callback_host, self._callback_port)
    await site.start()
    logger.info(
        "Mattermost: interactive-button callback server on %s:%d (url=%s)",
        self._callback_host, self._callback_port, self._effective_callback_url(),
    )
except Exception as exc:
    logger.warning(
        "Mattermost: could not start callback server (%s); "
        "interactive buttons disabled, falling back to text prompts.", exc,
    )
    self._runner = None
```

In `disconnect()`, add (before/after the existing teardown):

```python
if self._runner is not None:
    try:
        await self._runner.cleanup()
    except Exception:
        pass
    self._runner = None
```

## 4. Button builder (Mattermost format — bare-alphanumeric id!)

```python
def _make_button(
    self, button_id: str, label: str, action_id: str, context: Dict[str, Any],
    style: Optional[str] = None,
) -> Dict[str, Any]:
    """One Mattermost interactive button.

    button_id MUST be bare alphanumeric (no '_' or '-') — mattermost/mattermost#25747.
    Uses MM fields name/integration.url/integration.context (NOT Slack text/value).
    """
    btn: Dict[str, Any] = {
        "id": button_id,
        "type": "button",
        "name": label,
        "integration": {
            "url": self._effective_callback_url(),
            "context": {"action_id": action_id, **context},
        },
    }
    if style:
        btn["style"] = style
    return btn
```

## 5. Pending-state helpers

```python
def _register_action(self, payload: Dict[str, Any]) -> str:
    """Store a pending action and return its opaque id (embed it in button context)."""
    action_id = secrets.token_urlsafe(16)
    self._pending_actions[action_id] = payload
    return action_id

def _pop_action(self, action_id: str) -> Optional[Dict[str, Any]]:
    """Atomic pop — also the double-click guard."""
    return self._pending_actions.pop(action_id, None)
```

## 6. The callback handler (dispatch by kind)

This is the heart of Phase 1. Authorize FIRST (before popping, so an unauthorized
click can't consume the token), then pop, then dispatch. The per-kind branches call
methods that Phase 2 implements — define them as stubs returning a generic message
for now, or raise `NotImplementedError` guarded so the route still imports. Recommended:
implement the full dispatch now and leave the per-kind resolution to small private
methods `_resolve_approval/_resolve_slash/_resolve_update/_resolve_clarify` that
Phase 2 fills in (create no-op stubs returning a label string here).

```python
async def _handle_callback(self, request: Any) -> Any:
    from aiohttp import web
    try:
        body = await request.json()
    except Exception:
        return web.json_response({"ephemeral_text": "Malformed payload."}, status=400)
    if not isinstance(body, dict):
        return web.json_response({"ephemeral_text": "Malformed payload."}, status=400)

    user_id = body.get("user_id", "")
    context = body.get("context") or {}
    action_id = context.get("action_id", "")

    # Authorization — reuse MATTERMOST_ALLOWED_USERS (check BEFORE popping).
    allowed_csv = os.getenv("MATTERMOST_ALLOWED_USERS", "").strip()
    if allowed_csv:
        allowed = {u.strip() for u in allowed_csv.split(",") if u.strip()}
        if "*" not in allowed and user_id not in allowed:
            logger.warning("Mattermost: unauthorized callback click by user_id=%s", user_id)
            return web.json_response(
                {"ephemeral_text": "You are not allowed to perform this action."},
                status=403,
            )

    payload = self._pop_action(action_id)
    if payload is None:
        return web.json_response(
            {"ephemeral_text": "This prompt has already been resolved or expired."}
        )

    kind = payload.get("kind", "")
    try:
        if kind == "approval":
            update_msg = await self._resolve_approval(payload, context, user_id)
        elif kind == "slash":
            update_msg = await self._resolve_slash(payload, context, user_id)
        elif kind == "update":
            update_msg = await self._resolve_update(payload, context, user_id)
        elif kind == "clarify":
            update_msg = await self._resolve_clarify(payload, context, user_id)
        else:
            return web.json_response({"ephemeral_text": "Unknown action."}, status=400)
    except Exception as exc:
        logger.error("Mattermost: callback resolution failed (kind=%s): %s", kind, exc, exc_info=True)
        return web.json_response({"ephemeral_text": "Hermes could not process this action."}, status=500)

    return web.json_response({
        "update": {"message": update_msg, "props": {}},
        "ephemeral_text": "Recorded.",
    })
```

Phase-1 stubs (replace bodies in Phase 2):

```python
async def _resolve_approval(self, payload, context, user_id) -> str: return "Recorded."
async def _resolve_slash(self, payload, context, user_id) -> str: return "Recorded."
async def _resolve_update(self, payload, context, user_id) -> str: return "Recorded."
async def _resolve_clarify(self, payload, context, user_id) -> str: return "Recorded."
```

## Acceptance

```powershell
cd "c:\Users\Anton\github\clean-logic\hermes-agent"
python -c "import plugins.platforms.mattermost.adapter; print('import ok')"
```

Write a quick throwaway check (or rely on Phase 4 tests) that:
- Building the adapter and calling `_make_button("approveonce","Allow Once","once",{"action_id":"x"})`
  returns a dict with `name`, `integration.url`, `integration.context`, and a bare
  alphanumeric `id`.
- `_register_action({...})` then `_pop_action(id)` returns the payload once and `None`
  the second time (double-click guard).
- `connect()`/`disconnect()` start and stop the aiohttp server without leaving the
  port bound (idempotent; calling disconnect twice is safe).

Do NOT add prompt methods or change existing message handling in this phase.

## Output to report

- Confirmation the module imports.
- The new attributes/methods added.
- Confirmation the existing connect/disconnect message flow is untouched apart from
  the additive server start/stop.
