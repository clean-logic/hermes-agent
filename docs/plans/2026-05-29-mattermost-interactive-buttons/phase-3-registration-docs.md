# Phase 3 — Registration hooks, config surface & docs

> Read `00-BACKGROUND.md` first. Depends on Phases 1-2.

## Goal

1. Add an `env_enablement_fn` so env-only Mattermost setups surface in
   `hermes gateway status` / `get_connected_platforms()` (parity with Discord/Teams).
2. Add the two new callback env vars to `plugin.yaml` `optional_env` (setup-wizard
   integration).
3. Document interactive buttons in the Mattermost user guide.

## Files

- EDIT: `plugins/platforms/mattermost/adapter.py` (`register()` + new `_env_enablement`).
- EDIT: `plugins/platforms/mattermost/plugin.yaml`.
- EDIT: `website/docs/user-guide/messaging/mattermost.md`.

## 1. `_env_enablement` (model on IRC/Teams)

The IRC plugin's hook is the template:

```python
# plugins/platforms/irc/adapter.py
def _env_enablement() -> dict | None:
    """Seed PlatformConfig.extra from env vars during gateway config load.
    Called BEFORE adapter construction so `gateway status` and
    get_connected_platforms() reflect env-only config. Returns None when not
    minimally configured. A special `home_channel` key becomes a HomeChannel.
    """
    server = os.getenv("IRC_SERVER", "").strip()
    channel = os.getenv("IRC_CHANNEL", "").strip()
    if not (server and channel):
        return None
    seed = {"server": server, ...}
    return seed
```

Mattermost version (minimal config = URL + token; surface the callback knobs too):

```python
def _env_enablement() -> dict | None:
    url = os.getenv("MATTERMOST_URL", "").strip()
    token = os.getenv("MATTERMOST_TOKEN", "").strip()
    if not (url and token):
        return None
    seed: dict = {"url": url}
    cb_host = os.getenv("MATTERMOST_CALLBACK_HOST", "").strip()
    cb_port = os.getenv("MATTERMOST_CALLBACK_PORT", "").strip()
    if cb_host:
        seed["callback_host"] = cb_host
    if cb_port:
        seed["callback_port"] = cb_port
    home = os.getenv("MATTERMOST_HOME_CHANNEL", "").strip()
    if home:
        seed["home_channel"] = {"chat_id": home}   # becomes a HomeChannel dataclass
    return seed
```

(Confirm the exact `home_channel` sub-dict shape against the IRC/Teams example before
finalizing — match whatever they return.)

## 2. Wire it into `register()`

The current `register()` ends roughly like this (inspect the live file):

```python
def register(ctx) -> None:
    ctx.register_platform(
        name="mattermost",
        label="Mattermost",
        adapter_factory=_build_adapter,
        check_fn=check_mattermost_requirements,
        is_connected=_is_connected,
        required_env=["MATTERMOST_URL", "MATTERMOST_TOKEN"],
        install_hint="pip install aiohttp",
        setup_fn=interactive_setup,
        apply_yaml_config_fn=_apply_yaml_config,
        allowed_users_env="MATTERMOST_ALLOWED_USERS",
        allow_all_env="MATTERMOST_ALLOW_ALL_USERS",
        cron_deliver_env_var="MATTERMOST_HOME_CHANNEL",
        standalone_sender_fn=_standalone_send,
        max_message_length=MAX_POST_LENGTH,
        # ...display fields...
    )
```

Add one line:

```python
        env_enablement_fn=_env_enablement,
```

`register_platform` forwards unknown kwargs to `PlatformEntry`; `env_enablement_fn` is
a valid field (Discord/Teams/IRC/ntfy/simplex all pass it). Do not add anything that
isn't a real `PlatformEntry` field or the dataclass will raise `TypeError`.

`Platform.MATTERMOST` is ALREADY in the gateway's `_UPDATE_ALLOWED_PLATFORMS`, so
`send_update_prompt` will be invoked automatically — no registration change needed
for that.

## 3. plugin.yaml — add callback env vars

Append to the existing `optional_env:` list in
`plugins/platforms/mattermost/plugin.yaml` (keep the existing entries; match their
rich-dict shape):

```yaml
  - name: MATTERMOST_CALLBACK_HOST
    description: "Bind host for the interactive-button callback server (default 127.0.0.1)."
    prompt: "Callback server host"
    password: false
  - name: MATTERMOST_CALLBACK_PORT
    description: "Bind port for the interactive-button callback server (default 18065)."
    prompt: "Callback server port"
    password: false
  - name: MATTERMOST_CALLBACK_URL
    description: "External URL Mattermost POSTs button clicks to (cross-host setups). Defaults to http://<host>:<port>/hermes-callback."
    prompt: "External callback URL"
    password: false
```

## 4. Docs — website/docs/user-guide/messaging/mattermost.md

Add a section "Interactive approval buttons" covering:

- **Why it matters:** Mattermost intercepts messages starting with `/`, so the
  plain-text `/approve` flow can't work — buttons are the only usable approval path
  (link issue #27587).
- **What's supported:** exec approval, slash confirm, update prompt, clarify.
- **Config:** `MATTERMOST_CALLBACK_HOST` / `MATTERMOST_CALLBACK_PORT` (default
  `127.0.0.1:18065`), and `MATTERMOST_CALLBACK_URL` for cross-host.
- **Mattermost server setting (single-host):** users must allow the server to POST to
  the local callback:
  `ServiceSettings.AllowedUntrustedInternalConnections: "127.0.0.1"`.
- **Auth:** clicks are gated by `MATTERMOST_ALLOWED_USERS` (`*` = allow all).

Keep prose concise and in the existing doc's voice. If your existing PR branch already
added a Mattermost approval doc section to this file, fold/replace it rather than
duplicating.

## Acceptance

```powershell
cd "c:\Users\Anton\github\clean-logic\hermes-agent"
python -c "import plugins.platforms.mattermost.adapter as m; print('ok')"
python -c "import yaml,io; yaml.safe_load(open('plugins/platforms/mattermost/plugin.yaml',encoding='utf-8')); print('yaml ok')"
```

- `register()` passes `env_enablement_fn` without raising at import/registration.
- plugin.yaml parses and lists the three new vars.
- Docs render in the existing structure (no broken markdown).

## Output to report

- The `_env_enablement` body and the `register()` line added.
- The plugin.yaml entries added.
- A short summary of the docs section.
