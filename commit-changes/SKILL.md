---
name: commit-changes
description: Commits changes and pushes to repo. Creates or updates a PR on GitHub. After finishing your work, call this skill, or simply write "pls, commit", "commit this", "push my changes", or "create a PR". Always use this skill at the end of any coding task when the user wants to save or share their work.
disable-model-invocation: true
effort: medium
---

# commit-changes

Commit changes, push to remote, and create or update a GitHub PR — following the user's personal conventions.

---

## Tool priority

**Always prefer `gh` CLI over raw `git` commands** when `gh` can do the job (diff, status, PR management). Fall back to plain `git` only if `gh` is not installed or a specific operation isn't supported by `gh`.

Check availability once at the start:

```bash
gh --version 2>/dev/null && echo "gh:available" || echo "gh:missing"
```

---

## Step 1 — Detect the primary branch

```bash
# With gh (preferred)
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'

# Fallback
git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'
```

Store this as `<primary>` (typically `main` or `master`). All PRs target this branch.

---

## Step 2 — Check current branch

```bash
git branch --show-current
```

**Protected / staging branches** (never commit directly to these):
`main`, `master`, `dev`, `qa`, `testing`

If the current branch **is not** a protected branch → proceed to Step 3.

If the current branch **is** a protected branch → go to **§ Branch Redirect** below before continuing.

---

## § Branch Redirect (only when on a protected branch)

The user has made changes on a staging branch. Do not commit here. Instead:

### Detect ticket ID (if any)

Search the conversation history and current session context for a Jira/Linear-style ticket ID — pattern: `[A-Z]+-\d+` (e.g. `AR-2000`, `SS-280`).

If found, all generated branch name options must be prefixed with that ticket ID:
`AR-2000-<slug>`

### Inspect the diff to understand the change

```bash
# With gh (preferred)
gh diff

# Fallback
git diff
```

### Generate 3 branch name options

Rules for branch names:

- Format: `<type>/<slug>` or `<ticket-id>-<slug>` (with ticket prefix when available)
- Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`
- Slug: **maximum 5 words**, hyphen-separated, lowercase
- With ticket: `AR-2000-employee-attribute-change` (no type prefix needed)
- Without ticket: `feat/user-profile-avatar-upload`

Generate 3 distinct, meaningful options based on what the diff actually does, then present them plus a free-text option:

```
I need to create a new branch for your changes (you're currently on a protected branch).

Here are 3 suggestions based on your changes:
  1. feat/short-description-here
  2. fix/another-option-here
  3. chore/third-option-here
  4. Type your own branch name

Which would you like? (1/2/3 or type your own)
```

Wait for user input, then:

```bash
git stash
git checkout -b <chosen-branch>
git stash pop
```

Continue to Step 3.

---

## Step 3 — Inspect changes

```bash
# With gh (preferred)
gh diff --stat
gh diff

# Fallback
git diff --stat
git diff
```

If there is nothing changed, tell the user and stop.

---

## Step 4 — Stage all changes

```bash
git add -A
```

---

## Step 5 — Write the commit message

Rules:

- Format: `<type>(<optional scope>): <summary>`
- **Summary must be maximum 6 words** — count strictly, no exceptions
- Types: `feat`, `fix`, `chore`, `docs`, `style`, `test`, `ci`, `refactor`
- Lowercase, no trailing period

Good examples:

- `feat(auth): add Google OAuth login` — 4 words ✓
- `fix(api): handle null user response` — 4 words ✓
- `chore: update dependencies` — 2 words ✓

Bad (too long — reject and retry):

- `feat(auth): add support for Google OAuth login flow` — 8 words ✗

```bash
git commit -m "<type>(<scope>): <summary>"
```

---

## Step 6 — Push to remote

```bash
git push origin <current-branch>
# If upstream not set:
git push --set-upstream origin <current-branch>
```

---

## Step 7 — Create or update the PR

PRs always target `<primary>` (detected in Step 1).

### Check if PR exists

```bash
gh pr list --head <current-branch> --json number,title,url
```

---

### If no PR exists → Create it

```bash
gh pr create \
  --title "<PR title>" \
  --body "<PR description>" \
  --base <primary>
```

**PR title:** Expanded, human-readable version of the commit summary.

---

### If PR already exists → Update the description

```bash
gh pr edit <number> --body "<updated PR description>"
```

The updated description must reflect **all changes on this branch** — not just the latest commit. Re-inspect the full branch diff before writing:

```bash
gh pr diff
# Fallback:
git diff <primary>..HEAD
```

---

## PR Description format

Always write the full PR description using this structure:

```markdown
## Summary

<2-4 sentences: what this PR does and why>

## Changes

- <change 1>
- <change 2>
- <change 3 ...>

## Testing

<How was this tested, or "N/A" for config/docs-only changes>
```

### ⚠️ Setup & Configuration section (conditional)

Include this section **only if** any of the following are true:

- `.env`, `.env.example`, or any environment config file was modified
- New environment variables were added or renamed
- Infrastructure, server config, or deployment files changed
- A migration must be run manually
- Any manual step is required on the server/environment to make this change work

If triggered, append at the end of the PR description:

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
3. ...

### Notes

<Warnings, gotchas, or anything the reviewer/deployer needs to know>
```

Keep this section **simple, clean, and unambiguous** — written so that anyone on the team can follow it without asking questions.

---

## Step 8 — Report to the user

Print a clean summary:

```
✅ Committed:  feat(scope): short message here
✅ Branch:     feat/my-feature-branch
✅ Pushed to:  origin/feat/my-feature-branch
✅ PR:         https://github.com/org/repo/pull/42  [created | updated]
```

---

## Error handling

| Situation                                       | Action                                                                                                         |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `gh` not installed                              | Fall back to `git` for all operations; skip PR creation; tell user to install `gh` from https://cli.github.com |
| Not authenticated with `gh`                     | Tell user to run `gh auth login` and retry                                                                     |
| Push rejected (conflicts)                       | Stop; tell user there are conflicts to resolve manually                                                        |
| Detached HEAD                                   | Warn user and ask which branch to use                                                                          |
| Nothing to commit after `git add -A`            | Tell user (possibly all files are in `.gitignore`)                                                             |
| Can't detect primary branch                     | Ask user: "Is your primary branch `main` or `master`?"                                                         |
| `git stash pop` conflicts after branch redirect | Warn user and list conflicting files                                                                           |
