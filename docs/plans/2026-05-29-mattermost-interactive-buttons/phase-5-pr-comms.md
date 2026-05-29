# Phase 5 — PR refresh & upstream communication

> Read `00-BACKGROUND.md` first. Depends on Phases 0-4 complete AND the repo owner's
> review pass. This phase pushes branches and posts messages — **only run the
> push/post steps after the owner approves the diff.**

## Goal

1. Land the synced + re-architected work on `main` and the PR branch.
2. Refresh PR #29373 (your PR) so it's mergeable and reflects the new scope.
3. Post the superseding messages on competing PR #26537 and issue #27587.

## Context

- `origin` = `clean-logic/hermes-agent` (the fork); `upstream` = `NousResearch/hermes-agent`.
- Your PR: **#29373** (`feat/mattermost-approval-buttons`), base `upstream/main`.
- Competing PR (to supersede): **#26537** by `shawnfeng0` (open, not merged, no reviews).
- Linked issue: **#27587** ("Mattermost: Add interactive button-based approval").
- The on-prem install is at commit `85b08f405` and must remain fast-forwardable from
  `origin/main` — do NOT force-push or rewrite that commit's history.

## Step 1 — push synced main

```powershell
cd "c:\Users\Anton\github\clean-logic\hermes-agent"
git checkout main
git push origin main
```

## Step 2 — refresh the PR branch

```powershell
git checkout feat/mattermost-approval-buttons
git merge main          # bring the branch up to the new architecture
# resolve any conflicts in favor of the new plugin implementation
git push origin feat/mattermost-approval-buttons
```

Then verify mergeability:

```powershell
gh pr view 29373 --repo NousResearch/hermes-agent --json mergeable,mergeStateStatus
```

## Step 3 — update PR #29373 body

```powershell
gh pr edit 29373 --repo NousResearch/hermes-agent --body-file <path-to-body.md>
```

Body draft (`body.md`):

> ## Summary
> Adds full interactive-button support to the Mattermost gateway **plugin** — exec
> approval, slash confirm, update prompt, and clarify — bringing it to parity with
> Discord and beyond.
>
> Re-architected onto the bundled-plugin model introduced in `af973e407` (Mattermost
> moved from `gateway/platforms/mattermost.py` to `plugins/platforms/mattermost/`).
> Interactive callbacks are served by a **plugin-owned aiohttp server** (the Teams/Line
> pattern) — **zero core edits**.
>
> ### What it does
> - `send_exec_approval` — Allow Once / Allow Session / Always Allow / Deny → `resolve_gateway_approval`
> - `send_slash_confirm` — Approve Once / Always / Cancel → `tools.slash_confirm.resolve`
> - `send_update_prompt` — Yes / No update decisions
> - `send_clarify` — per-option buttons → `resolve_gateway_clarify`
> - Local HTTP callback server (three-tier `callback_host`/`callback_port`, default
>   `127.0.0.1:18065`), `MATTERMOST_ALLOWED_USERS` auth, double-click guard,
>   original-message updates on click.
>
> ### Why buttons (not text)
> Mattermost intercepts messages starting with `/`, so the plain-text `/approve`
> fallback in #27587 is impossible to action. Buttons are the only working approval
> path.
>
> ### Relation to #26537
> Supersedes @shawnfeng0's #26537 (same goal). This PR is plugin-native (his targets
> the now-deleted core adapter), adds `send_clarify`, full registration hooks, and
> tests + docs. Credit to his callback-server design and review exchange.
>
> Fixes #27587 (primary item; the slash-command-registration secondary item is out of
> scope).

## Step 4 — comment on competing PR #26537 (polite; competing, not collaborative)

```powershell
gh pr comment 26537 --repo NousResearch/hermes-agent --body-file <path-to-comment-b.md>
```

Draft:

> Hi @shawnfeng0 — following up on our earlier exchange. I had bandwidth to take this
> the rest of the way, so I've pushed a fully reworked version in #29373 and I think
> it's now ready to supersede this PR.
>
> Since we last talked, upstream migrated Mattermost from the core adapter to a
> bundled plugin (`af973e407`), which means both of our original diffs targeted a file
> that no longer exists. #29373 is rebuilt on the new plugin architecture: the
> interactive callbacks run on a plugin-owned aiohttp server (no core edits), it covers
> exec approval, slash confirm, update prompt **and** clarify, and it ships tests +
> docs.
>
> Your review on my earlier PR genuinely shaped this — the `MATTERMOST_ALLOWED_USERS`
> auth gate and the configurable callback host/port both came directly from your
> feedback and your implementation, and I've credited that in the PR. Thank you for
> that.
>
> I went solo rather than combining forces just to keep momentum while I had the time
> to iterate quickly — no reflection on your work, which I appreciated. I'd be glad to
> have your eyes on #29373 if you have a moment, and of course the maintainers can pick
> whichever they prefer.

## Step 5 — comment on issue #27587

```powershell
gh issue comment 27587 --repo NousResearch/hermes-agent --body-file <path-to-comment-c.md>
```

Draft:

> Update: #29373 now fully implements interactive button approval for the Mattermost
> gateway and is ready for review.
>
> It's built on the new bundled-plugin architecture (no core edits), serves button
> callbacks via a plugin-owned aiohttp server, and covers the primary ask here (exec
> approval) plus slash confirm, update prompt, and clarify. This directly addresses the
> root cause noted in this issue — that `/`-prefixed text replies are intercepted by
> Mattermost, making the plain-text approval fallback unusable — by giving users
> clickable buttons instead. Tests and docs included.
>
> The secondary slash-command-registration item remains out of scope. Ready to merge
> from my side.

## Step 6 — owner's on-prem update (informational; the owner runs this)

```powershell
git fetch origin
git pull --ff-only origin main
# then the usual dependency refresh, e.g.: uv pip install -e ".[all]"
```

## Acceptance

- `gh pr view 29373 ... --json mergeable` reports `MERGEABLE` (or at least not
  CONFLICTING) and CI is green.
- PR body updated; comments posted on #26537 and #27587.
- `git rev-list --count main..upstream/main` == 0.

## Output to report

- New PR head SHA and mergeable status.
- Links/confirmation of the two comments posted.
