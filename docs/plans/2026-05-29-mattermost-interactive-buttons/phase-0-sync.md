# Phase 0 — Fork sync & clean base (git only)

> Read `00-BACKGROUND.md` first. This phase has **no Python code changes** — it gets
> the repo onto the latest upstream with a clean plugin base so later phases apply.

## Goal

Bring the fork's `main` up to date with `upstream/main` (currently ~858 commits
behind), preserving the single local feature commit's intent, and remove two stale
artifacts from the old architecture:

- the deleted core adapter `gateway/platforms/mattermost.py` (upstream migrated
  Mattermost to a plugin in commit `af973e407`),
- the old core edit to `gateway/platforms/webhook.py` (`register_extra_route`), which
  the new architecture forbids.

## Context

- Local `main` is at commit `85b08f405` (your one feature commit) and is **1 ahead,
  858 behind** `upstream/main`.
- A test merge has already been validated to auto-merge cleanly; the only manual
  cleanups are the two file removals below.
- The on-premises install sits at `85b08f405`, so `main` must remain
  fast-forwardable from there (use a merge, NOT a rebase that rewrites `85b08f405`).

## Steps

```powershell
cd "c:\Users\Anton\github\clean-logic\hermes-agent"
git fetch upstream
git fetch origin

# Safety snapshot
git branch backup/pre-upstream-sync-$(Get-Date -Format yyyyMMdd) main

git checkout main
git merge upstream/main -m "Merge upstream/main into fork main"

# If the merge re-adds the deleted core adapter, remove it:
if (Test-Path gateway/platforms/mattermost.py) { git rm gateway/platforms/mattermost.py }

# Drop the old core webhook edit — restore upstream's version exactly:
git checkout upstream/main -- gateway/platforms/webhook.py

# Stage the removals/restores into the merge result
git add -A
git commit --amend --no-edit   # fold cleanups into the merge commit (it was created above)
```

Notes:
- If `git merge` reports conflicts (unlikely), resolve by taking upstream for any
  `gateway/` core file and deleting `gateway/platforms/mattermost.py`. The feature
  code is being rewritten in later phases, so do NOT try to preserve the old
  `gateway/platforms/mattermost.py` approval code here — it's captured in the phase
  docs.
- The old test `tests/gateway/test_mattermost_approval_buttons.py` will likely be
  present and **failing** after this phase (it imports the deleted module). That is
  expected; Phase 4 rewrites it. Do not delete it yet.

## Acceptance

```powershell
# 0 commits behind upstream:
git rev-list --count main..upstream/main      # must print 0

# No core changes remain vs upstream (only plugins/ work happens later):
git diff upstream/main --stat -- gateway/ tools/ hermes_cli/   # must be empty

# The deleted core adapter is gone; the plugin exists:
Test-Path gateway/platforms/mattermost.py                      # False
Test-Path plugins/platforms/mattermost/adapter.py              # True

# The plugin imports cleanly:
python -c "import plugins.platforms.mattermost.adapter; print('ok')"
```

If `git diff upstream/main --stat -- gateway/` shows `webhook.py` still changed, you
did not restore it — re-run `git checkout upstream/main -- gateway/platforms/webhook.py`.

## Output to report

- The merge commit SHA.
- Confirmation of the four acceptance checks.
- Whether the old test file is present (expected: yes, failing).
