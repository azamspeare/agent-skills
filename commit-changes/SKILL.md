---
name: commit-changes
description: Commits changes, pushes to remote, and creates or updates a GitHub PR — using personal conventions. Invoke on "pls commit", "commit this", "push my changes", "create a PR", or at the end of a coding task.
disable-model-invocation: true
effort: medium
---

# commit-changes

Commit, push, and open/update a GitHub PR following the user's conventions.

## Tooling

- **`git`** — all local operations: `status`, `diff`, `add`, `commit`, `push`, branches.
- **`gh`** — GitHub-only operations: repo info, PR list/create/edit, `gh pr diff`.

`gh` has no working-tree commands (no `gh diff`, no `gh status`). Don't try.

Check once at start:
```bash
gh --version >/dev/null 2>&1 && echo "gh:ok" || echo "gh:missing"
```
If missing: still commit + push via `git`, skip the PR step, and tell the user to install from https://cli.github.com.

---

## Step 1 — Detect primary branch

```bash
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
# Fallback:
git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'
```
Store as `<primary>`. All PRs target it.

## Step 2 — Check current branch

```bash
git branch --show-current
```

Protected: `main`, `master`, `dev`, `qa`, `testing`.
On a protected branch → **§ Branch Redirect**, then Step 3. Otherwise → Step 3.

---

## § Branch Redirect

**Inspect the diff**, then pick a ticket ID and name.

### Ticket ID

Look for `[A-Z]+-\d+` in (1) the current branch name, (2) recent conversation. If neither, ask:
> Is there a ticket ID for this work (e.g. `AR-2000`)? Press enter to skip.

### Branch naming

- With ticket: `<ticket-id>-<slug>` → `AR-2000-employee-attribute-change`
- Without ticket: `<type>/<slug>` → `feat/user-profile-avatar`
- Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`
- Slug: lowercase, hyphen-separated, **≤5 words**, describes the change.

### Offer 3 suggestions derived from the diff

```
You're on a protected branch. Let's move your changes to a new branch.

  1. feat/short-description
  2. fix/another-option
  3. chore/third-option
  4. Type your own

Which one? (1/2/3 or type your own)
```

### Create the branch from `<primary>` — not from the current HEAD

Staging branches (`dev`, `qa`) usually contain commits not yet in `<primary>`. Branching from the current HEAD would sweep those into the PR diff. Branch from a fresh `<primary>` and carry the work across with a stash:

```bash
git fetch origin
git stash --include-untracked
git checkout -b <chosen-branch> origin/<primary>
git stash pop
```

If `git stash pop` conflicts: stop and tell the user:
> Local changes conflict with `<primary>`. Files: `<list>`. Resolve, run `git add <files>`, then re-invoke.

Don't auto-resolve. The stash is preserved in `git stash list`.

---

## Step 3 — Inspect

```bash
git status
git diff --stat
git diff
```
Nothing to commit → tell the user and stop.

## Step 4 — Stage only what the agent changed

**Never `git add -A` or `git add .`** — they silently stage secrets, unrelated user work, and build artifacts.

1. From this session, list the files you actually edited.
2. Intersect with `git status` — that's the staging set.
3. If `git status` shows other modified/untracked files you didn't touch, list them and ask whether to include.
4. If any path matches `*.env*` or contains `secret`/`credential`/`key`/`token`/`password`, ask before staging.

```bash
git add <file1> <file2> ...
git diff --cached --stat
```

## Step 5 — Commit

Format: `<type>(<optional scope>): <summary>`

- Types: `feat`, `fix`, `chore`, `docs`, `style`, `test`, `ci`, `refactor`
- Summary: prefer **≤6 words**, hard cap **10**. Lowercase, no trailing period. Describe the change, not the files.

Examples: `feat(auth): add Google OAuth login` · `fix(api): handle null user response` · `chore: update dependencies`

```bash
git commit -m "<type>(<scope>): <summary>"
```

**If a pre-commit hook fails:** the commit did not happen. Fix the real issue (lint/format/types/tests), re-stage, retry. No `--no-verify` unless the user explicitly asks.

## Step 6 — Push

```bash
git push origin <current-branch>
# If upstream not set:
git push --set-upstream origin <current-branch>
```
Push rejected → stop and show the message. **Never force-push** without explicit user instruction.

## Step 7 — Create or update the PR

Target is always `<primary>`.

```bash
gh pr list --head <current-branch> --json number,title,url
```

**No PR yet:**
```bash
gh pr create --title "<PR title>" --body "<PR description>" --base <primary>
```

**PR exists:** re-read the full branch diff (`gh pr diff <number>`) so the description reflects **all** changes on the branch, then:
```bash
gh pr edit <number> --body "<updated description>"
```

**PR title** — sentence case, no `type(scope):` prefix, no trailing period.
Commit `feat(auth): add Google OAuth login` → PR title `Add Google OAuth login to the sign-in flow`.

---

## PR description format

```markdown
## Summary

<2-4 sentences: what this PR does and why>

## Changes

- <change 1>
- <change 2>

## Testing

<How this was tested, or "N/A" for config/docs-only changes>
```

### ⚙️ Setup & Configuration (append only if any apply)

- `.env` / env config file changed
- New env vars added or renamed
- Infra / server / deployment files changed
- Manual migration required
- Any manual server/env step required

```markdown
## ⚙️ Setup & Configuration

> Action required before/after deploying this PR.

### Environment variables

| Variable   | Description  | Example      |
| ---------- | ------------ | ------------ |
| `VAR_NAME` | What it does | `some-value` |

### Steps

1. <Clear, numbered step>
2. <Next step>

### Notes

<Warnings, gotchas, anything the reviewer/deployer needs to know>
```

Keep it simple — anyone on the team should follow without asking.

---

## Step 8 — Report

```
✅ Committed: feat(scope): short message here
✅ Branch:    feat/my-feature-branch
✅ Pushed:    origin/feat/my-feature-branch
✅ PR:        https://github.com/org/repo/pull/42  [created | updated]
```
Say explicitly if a step was skipped (e.g. `gh` missing → no PR).

---

## Error handling

| Situation                                        | Action                                                                 |
| ------------------------------------------------ | ---------------------------------------------------------------------- |
| `gh` not installed                               | Commit + push via `git`; skip PR; tell user to install `gh`            |
| `gh` not authenticated                           | Tell user to run `gh auth login` and retry                             |
| Primary branch undetectable                      | Ask: "Is it `main` or `master`?"                                       |
| Nothing to commit                                | Tell user and stop                                                     |
| Detached HEAD                                    | Warn user; ask which branch to use                                     |
| Pre-commit hook failed                           | Fix underlying issue, re-stage, retry (never `--no-verify`)            |
| Push rejected (non-fast-forward / conflicts)     | Stop; ask user to resolve; do not force-push                           |
| Secret-looking file in staging set               | List it; ask before staging                                            |
| Files modified that the agent did not touch      | List them; ask whether to include                                      |
| `git stash pop` conflicts during branch redirect | Stop; list conflicting files; ask user to resolve, then re-invoke      |
