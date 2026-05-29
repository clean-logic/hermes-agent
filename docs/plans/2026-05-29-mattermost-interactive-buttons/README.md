# Mattermost interactive buttons — phased implementation plan

Internal planning docs for reworking PR
[#29373](https://github.com/NousResearch/hermes-agent/pull/29373) (Mattermost
interactive approval/confirm/clarify buttons) onto the new bundled-plugin
architecture, superseding the competing PR
[#26537](https://github.com/NousResearch/hermes-agent/pull/26537) and closing the
primary item of issue
[#27587](https://github.com/NousResearch/hermes-agent/issues/27587).

> These are working notes, **not** upstream documentation. Do not include this folder
> in the upstream PR.

## How to use

Each phase is meant to be handed to an implementing LLM/session that has **no access**
to the originating conversation. Start every phase by reading
[`00-BACKGROUND.md`](00-BACKGROUND.md), then the specific phase file.

| Phase | File | Scope |
|------|------|-------|
| — | [`00-BACKGROUND.md`](00-BACKGROUND.md) | Shared context: architecture rules, Mattermost button format, resolver APIs, reference implementations. **Read first.** |
| 0 | [`phase-0-sync.md`](phase-0-sync.md) | Fork sync to upstream; drop stale core adapter + webhook edit (git only). |
| 1 | [`phase-1-callback-infra.md`](phase-1-callback-infra.md) | Plugin-owned aiohttp callback server, route, auth, pending-state, button helper. |
| 2 | [`phase-2-prompt-methods.md`](phase-2-prompt-methods.md) | `send_exec_approval` / `send_slash_confirm` / `send_update_prompt` / `send_clarify` + resolvers. |
| 3 | [`phase-3-registration-docs.md`](phase-3-registration-docs.md) | `env_enablement_fn`, `plugin.yaml` callback vars, user docs. |
| 4 | [`phase-4-tests.md`](phase-4-tests.md) | Hermetic tests for all surfaces + auth + double-click + fallback. |
| 5 | [`phase-5-pr-comms.md`](phase-5-pr-comms.md) | Push, refresh PR #29373, post supersede messages on #26537 and #27587. |

## Core decisions (summary)

- **Plugin-only, zero core edits.** Mattermost is now a bundled plugin
  (`plugins/platforms/mattermost/`). Button-click callbacks are served by a
  plugin-owned `aiohttp` server (the Teams/Line pattern) — the old approach of editing
  `gateway/platforms/webhook.py` is forbidden.
- **Four interactive surfaces** (exec approval, slash confirm, update prompt, clarify)
  — one more than #26537 (which lacks clarify), plus full registration hooks, tests,
  and docs.
- **Why buttons:** Mattermost intercepts `/`-prefixed messages, so the plain-text
  `/approve` fallback is unusable; buttons are the only working approval path.
- **Merge, don't rebase** the fork sync — the on-prem install must remain
  fast-forwardable from commit `85b08f405`.

## Sequencing

```
Phase 0 (git sync) -> Phase 1 (infra) -> Phase 2 (methods) -> Phase 3 (reg+docs) -> Phase 4 (tests) -> [owner review] -> Phase 5 (push + comms)
```
