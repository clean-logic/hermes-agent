# Background — Mattermost interactive buttons (READ FIRST)

You are implementing one phase of a multi-phase feature for **Hermes Agent**, an
open-source AI agent. You do **not** have access to the conversation that produced
this plan — everything you need is in these phase documents. Read this background,
then your assigned phase file (`phase-N-*.md`).

## Repository

- Local checkout: `c:\Users\Anton\github\clean-logic\hermes-agent` (a fork; `origin` =
  `clean-logic/hermes-agent`, `upstream` = `NousResearch/hermes-agent`).
- Shell is PowerShell on Windows. Chain commands with `;` (NOT `&&`). Tests can also
  run under the bash wrapper if available.
- Python package layout: top-level modules (`gateway/`, `tools/`, `plugins/`, etc.).

## What we are building

Full interactive-button support for the **Mattermost** gateway adapter, matching
Discord's feature set and superseding a competing PR (#26537). Four interactive
"prompt" surfaces, all rendered as Mattermost interactive-message buttons:

| Surface | Purpose | Buttons |
|---|---|---|
| exec approval | approve a dangerous shell command | Allow Once / Allow Session / Always Allow / Deny |
| slash confirm | confirm a slash command | Approve Once / Always / Cancel |
| update prompt | answer a `hermes update` y/n question | Yes / No |
| clarify | answer the agent's clarify question | one button per choice (+ "Other") |

### Why buttons (not text)

Mattermost intercepts any message starting with `/`, so the gateway's plain-text
approval fallback ("reply `/approve once`") is impossible to action on Mattermost.
Interactive buttons are the only working path. This is the core motivation (issue
#27587).

## Critical architecture rules

1. **Plugin-only. ZERO core edits.** Mattermost is a bundled plugin at
   `plugins/platforms/mattermost/` (adapter.py, __init__.py, plugin.yaml). Do NOT
   modify `gateway/`, `tools/`, `hermes_cli/`, or any other core file. An older
   approach added `register_extra_route` to `gateway/platforms/webhook.py` — that is
   explicitly forbidden now. The button-click HTTP callback must be served by a
   **plugin-owned aiohttp server** that the adapter starts itself (the pattern the
   Teams and Line plugins already use).
2. **Don't break prompt caching / message flow.** The new code is additive.
3. **Cache-safe fallback.** When interactive buttons aren't configured, the `send_*`
   methods must return `SendResult(success=False)` so the gateway falls back to its
   existing text prompt unchanged.

## Mattermost interactive-button format (MUST follow exactly)

Mattermost is NOT Slack. Buttons live in a post's `props.attachments[].actions[]`.
Each action MUST use Mattermost fields:

- `name` = visible label (NOT `text`).
- `integration.url` = the callback URL Mattermost POSTs to on click.
- `integration.context` = an arbitrary dict echoed back in the POST body.
- Action `id` MUST be **bare alphanumeric** (no `_`, no `-`) — per
  mattermost/mattermost#25747, ids containing `_`/`-` are silently rejected and the
  handler is never registered. Use ids like `approveonce`, `approvesession`.
- Optional `style`: `"primary"` / `"danger"` / `"good"`.

On click, Mattermost POSTs JSON `{ user_id, post_id, channel_id, context: {...} }`
to `integration.url`. The HTTP response may include
`{"update": {"message": "...", "props": {...}}, "ephemeral_text": "..."}` to edit the
original post (show the chosen option and remove buttons) and show the clicker a
toast.

## SendResult (gateway/platforms/base.py)

```python
@dataclass
class SendResult:
    success: bool
    message_id: Optional[str] = None
    error: Optional[str] = None
    raw_response: Any = None
```

## Resolver APIs (call these from the click handler; all live in core — import, don't edit)

```python
# tools/approval.py
def resolve_gateway_approval(session_key: str, choice: str, resolve_all: bool = False) -> int
#   choice in {"once","session","always","deny"}; use resolve_all=False (per-click).

# tools/slash_confirm.py
async def resolve(session_key: str, confirm_id: str, choice: str, timeout=...) -> Optional[str]
#   choice in {"once","always","cancel"}; returns optional follow-up text to post.

# tools/clarify_gateway.py
def resolve_gateway_clarify(clarify_id: str, response: str) -> bool
#   response = the canonical choice text. For the "Other" button, instead call
#   mark_awaiting_text(clarify_id) so the next user message is captured.
def mark_awaiting_text(clarify_id: str) -> bool

# update prompt has NO resolver function — the click writes the answer ("y"/"n")
# to a file the detached update process reads:
#   from hermes_constants import get_hermes_home
#   path = get_hermes_home() / ".update_response"   # atomic write: tmp then replace
```

## Reference implementations to copy from (read but DO NOT edit)

- `plugins/platforms/discord/adapter.py` — richest reference; has all four `send_*`
  methods and their button/resolver wiring (the View classes show exact choice
  strings and resolver calls).
- `plugins/platforms/teams/adapter.py` — the plugin-owned aiohttp server lifecycle
  (`web.AppRunner` + `web.TCPSite` in `connect()`, `runner.cleanup()` in
  `disconnect()`).
- The fork's **previous** Mattermost implementation (now on the old core file) is the
  gold reference for the MM-specific button format and click handler. Retrieve it
  with: `git show 85b08f405:gateway/platforms/mattermost.py`. The key methods are
  `send_exec_approval`, `_approval_button`, and `_handle_approval_action` (embedded in
  phase-1 and phase-2 docs).
- The fork's previous tests: `git show 85b08f405:tests/gateway/test_mattermost_approval_buttons.py`.

## Config (env vars; three-tier resolution)

- `MATTERMOST_CALLBACK_HOST` — bind host for the callback server (default `127.0.0.1`).
- `MATTERMOST_CALLBACK_PORT` — bind port (default `18065`).
- `MATTERMOST_CALLBACK_URL` — optional explicit external URL embedded in buttons (for
  cross-host setups where Mattermost reaches Hermes at a different address).
- `MATTERMOST_ALLOWED_USERS` — existing comma-separated allowlist; reuse it to gate
  button clicks. `*` means allow all.

Resolution order for host/port: `config.extra.get("callback_host"/"callback_port")`
→ env var → default.

## House conventions (from AGENTS.md)

- Use `get_hermes_home()` / `display_hermes_home()` from `hermes_constants` for any
  `~/.hermes` path — never hardcode.
- Tests: stdlib + pytest + `unittest.mock` only, no network. Do NOT write
  change-detector tests (asserting snapshots of data that's expected to change).
- Keep skill/plugin descriptions terse; follow existing file style.

## Per-phase workflow expected of you

1. Read this background + your phase file.
2. Inspect the current files named in your phase before editing.
3. Implement only your phase's scope.
4. Run the verification commands in your phase's "Acceptance" section.
5. Report what you changed and any deviations.
