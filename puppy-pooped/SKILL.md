---
name: puppy-pooped
description: Fixes a failing CI / GitHub Actions job end to end. Pulls the failed run logs, finds the root cause, explains it in short plain terms, fixes the code like a senior engineer (security-aware), auto-commits and pushes, then watches the run until it passes — retrying itself while it keeps making progress. Stops and explains if the failure is not code-fixable. Invoke with "/puppy-pooped <action-run-url>", "tests failed <url>", "CI is red", "fix the failing job", "audit this action failure".
disable-model-invocation: true
effort: max
---

# puppy-pooped

The puppy pooped in CI. Find the mess, explain it simply, clean it up, and don't leave until the floor is clean (the job is green).

Full loop: **audit the failure → root cause → fix like a senior → commit + push → watch the run → if still red, go again.**

**Core promises:**
- Approach like a **senior engineer**. Make **no careless mistakes**. Fix the real cause, not the symptom.
- **Consider security** on every change (secrets, auth, injection, permissions). Never weaken security to make a test pass.
- **Auto commit + push** each attempt. **Watch** until pass.
- **Retry yourself** while you're making progress. **Stop and explain** if it's not code-fixable.

---

## Step 0 — Be brief in all explanations

When you explain anything to the user (root cause, fix, final report): short, simple, plain — a child could follow.
- If the **caveman** skill is available, use it.
- If not, still write short. No log dumps, no walls of text.

The point is to save the user from reading the giant logs. Don't recreate them.

---

## Step 1 — Parse the input

Input is an Actions URL, e.g.:
```
https://github.com/<owner>/<repo>/actions/runs/<runId>            (a run)
https://github.com/<owner>/<repo>/actions/runs/<runId>/job/<jobId> (a specific job)
```
Pull out `owner`, `repo`, `runId`, and `jobId` (if present).

Check `gh` once:
```bash
gh --version >/dev/null 2>&1 && echo "ok" || echo "missing"
```
Missing → stop, tell the user to install from https://cli.github.com.

## Step 2 — Pull only the failed logs

Full logs are huge. Pull just the failed steps:
```bash
gh run view <runId> --repo <owner>/<repo> --log-failed
# If a job is specified or you need a single job:
gh run view --job <jobId> --repo <owner>/<repo> --log-failed
```
Also get the run's overview (which jobs/steps failed, the branch, the workflow):
```bash
gh run view <runId> --repo <owner>/<repo>
```

Make sure you're on the **same branch** the run was for. If not, check it out:
```bash
git fetch origin && git checkout <run-branch>
```

## Step 3 — Find the root cause

Read the failed log. Find the **real** cause — the first true error, not the noise after it.

Then map it to the code: open the failing test/file, trace what it calls. Use Read / Grep. Understand *why* it fails before touching anything.

Tell the user, short:
> **Root cause:** <one or two plain lines>

## Step 4 — Is this even code-fixable?

Before editing, decide what kind of failure this is:

| Kind | Examples | Action |
| ---- | -------- | ------ |
| **Code** | failing test, type error, lint, bad logic, broken import, build error | Fix it (Step 5) |
| **Not code** | missing/expired secret or env var, runner down, network/rate limit, external service 5xx, permissions/token, dependency registry outage | **Stop.** Don't blindly edit code. Explain what's wrong and the manual step needed. |
| **Maybe flaky** | passes locally, timing/race, intermittent | Note it. You may re-run once (Step 7) before deciding. |

If **not code-fixable** → jump to § Stop and Explain.

## Step 5 — Fix it like a senior engineer

- Fix the **root cause**, not the symptom. Don't delete/skip a test to make it pass. Don't loosen assertions just to go green.
- **Security first:** never commit secrets, never widen permissions, never disable auth/validation, never add `--no-verify`-style bypasses to "win". If the only way to pass weakens security, **stop and explain instead**.
- Keep the change minimal and in the project's style.
- If tests can run locally, run them before pushing to confirm the fix.

## Step 6 — Commit + push (auto)

Use the project's conventions (same spirit as the `take-shit` skill: never `git add .`, stage only what you changed, conventional commit, never force-push unless the user said so).

```bash
git add <files you changed>
git commit -m "fix(ci): <short real fix>"
git push origin <current-branch>
```

This is **automatic** — don't ask for confirmation. (Exception: if a fix would weaken security or needs a force-push, stop and ask.)

## Step 7 — Watch the run until it finishes

A push starts a new run. Find it and watch it:
```bash
# newest run for this branch
NEW=$(gh run list --repo <owner>/<repo> --branch <branch> --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$NEW" --repo <owner>/<repo> --exit-status --compact
```
`--exit-status` returns non-zero if it failed → that's your pass/fail signal.

- **Passed (exit 0)** → § Final Report.
- **Failed** → § Retry Loop.

## § Retry Loop — keep going while making progress

Loop back to Step 2 on the **new** failed run. But guard against spinning forever:

- Track the root cause / error signature of each attempt.
- **Keep retrying** as long as the error **changes** or you have a **new, different fix** to try (you're making progress).
- **Stop** if two attempts in a row hit the **same failure** and you have **no new idea** → § Stop and Explain. Repeating the same fix is not progress.
- Also stop the moment a failure turns out to be **not code-fixable** (Step 4).

Each loop: audit new logs → new root cause → new fix → commit + push → watch. No attempt cap while progress is real.

## § Stop and Explain

When you can't fix it (not code-fixable, or stuck with no new idea), don't keep pushing. Stop and tell the user, short:
- **What's failing:** <plain>
- **Why it's not something I can fix in code:** <plain>
- **What you need to do:** <the manual step — e.g. "add `SIGNATURE_SERVICE_TOKEN` to repo secrets">

## § Final Report

When the run is green, drop this, short (caveman/brief):

```
✅ CI is green.

What was wrong: <plain root cause>
How I fixed it: <plain, what changed and why>
Attempts:       <n>
Commits:        <list of fix commits>
Run:            <url of the passing run>
```

---

## Rules

- Senior-engineer mindset: real root cause, minimal correct fix, project style.
- **Never** make a test pass by deleting/skipping it, loosening assertions, weakening security, or bypassing checks.
- Auto commit + push each attempt; never force-push without asking.
- Stop on non-code failures — don't edit code blindly.
- Don't loop forever: same failure twice with no new idea = stop and explain.
- All explanations short and plain (caveman if available). Never dump raw logs at the user.
