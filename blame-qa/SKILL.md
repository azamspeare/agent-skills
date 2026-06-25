---
name: blame-qa
description: Generates a QA test document from the changes on the current branch. Analyzes branch commits vs the primary branch, code-traces what else those changes could break, and writes a markdown file for a manual QA engineer — in plain client-POV language (no code), with what changed, what to test, how to test (do/expected tables, edge cases), and severity-tagged regression areas. Invoke with "/blame-qa", "write QA doc", "generate test doc", "what should QA test", "make a test plan for my changes".
disable-model-invocation: true
effort: high
---

# blame-qa

After you commit, write the QA test doc for you. It explains **what changed**, **how to test it**, and **what else might have broken** (regression) — written for a QA engineer who didn't see your code.

**Output:** one markdown file at `docs/qa/{feature-name}-test-doc.md`.

**Audience:** a QA engineer. No code-level jargon. They should be able to test without asking you anything.

---

## Step 1 — Detect primary branch

```bash
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name' 2>/dev/null \
  || git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'
```
Store as `<primary>`.

## Step 2 — Gather the changes (branch vs primary)

```bash
git fetch origin <primary> >/dev/null 2>&1
git log origin/<primary>..HEAD --pretty="%s"        # commit subjects = intent
git diff --name-only origin/<primary>...HEAD        # changed files
git diff origin/<primary>...HEAD                    # full diff = what actually changed
```

If there are **no changes** vs `<primary>` → tell the user there's nothing to document and stop.

Read the diff to understand the real behavior change — not just file names. Group changes by **feature/area**, not by file.

### ⚠️ Translate everything to the QA engineer's world

The reader is a **manual QA engineer**. They do **not** read code. They do not know PHP, JS/TS, Python, or any language. They test by **opening the browser and clicking, like a user** (client point of view).

So when you describe a change, **never** mention code: no function names, no file names, no endpoint paths, no language terms, no class/model names as code. Translate the technical change into **what a tester would notice on the screen / in the app**, at a high level — abstraction, not detail.

Example of the right altitude:
- Diff: a bunch of routes for the `Document` model controller were renamed.
- ❌ Wrong (too technical): "Renamed `DocumentController` routes from `/api/docs` to `/api/documents`."
- ✅ Right (abstract, client POV): "Document related endpoints renamed."

Keep each change to **one short line**. Group by the feature the user sees (Documents, Login, Billing…), not by file.

## Step 3 — Find the ticket ID

Look for `[A-Z]+-\d+` in: the current branch name, then the commit subjects.
```bash
git branch --show-current | grep -oE '[A-Z]+-[0-9]+'
```
If found → put it at the top of the doc. If not → omit the ticket line (don't invent one).

## Step 4 — Confirm the feature name (filename)

Propose a `{feature-name}` slug derived from the changes (lowercase, hyphen-separated, ≤4 words). Then ask:

> I'll write the QA doc to `docs/qa/<proposed>-test-doc.md`. Good, or give me a different name?

Use what the user confirms. The final path is `docs/qa/{feature-name}-test-doc.md`.

## Step 5 — Code-trace regression impact

Regression = already-working features that might **break** because of this change. QA must re-test them. This is the part QA can't figure out alone, because it needs reading the code — which you do here, then translate to plain words for them.

For every changed function / method / endpoint / class / exported symbol / shared module / DB column, find what **depends on it**:

```bash
# Who calls or imports the changed thing? (adapt the term per language)
grep -rn "changedFunctionName" --include="*.php" --include="*.ts" --include="*.js" --include="*.py" . \
  | grep -v -F "$(git diff --name-only origin/<primary>...HEAD)"
```

Trace callers, importers, route consumers, event listeners, shared utilities, and anything reading a changed DB column or config. Each one a user can see becomes a **regression area**.

For each area, write it the QA way (plain, client POV — same translation rule as Step 2):
- **Area** — the feature the user sees (e.g. "Document sharing", "Invoice PDF").
- **Why affected** — one plain line (e.g. "uses the same document data that changed").
- **Severity** — High (directly depends on the changed thing; likely to break) · Med (shares data/screen but not the exact path) · Low (loosely related; check if time allows).
- **What to check** — one quick action.

**If the code trace finds nothing depending on the change** → don't pad. Write a single line in the doc:
> Nothing else is affected by these changes.

## Step 6 — Write the file

```bash
mkdir -p docs/qa
```
Write `docs/qa/{feature-name}-test-doc.md` (overwrite if it exists) using the template below.

## Step 7 — Report

```
✅ QA doc written: docs/qa/<feature-name>-test-doc.md
   Ticket:      AR-2000   (or "none")
   Test areas:  <n>   (How-to-test subsections)
   Regression:  <h> high · <m> med · <l> low  (or "none affected")
```

Remind the user it's a new/changed file — commit it (e.g. with `/take-shit`) if they want it in the PR.

---

## Document template

Fill every `<...>`. Keep everything in plain, client-POV language (see Step 2 rule). Delete the example rows.

```markdown
# QA Test Doc — <Feature Title>

> Ticket: AR-2000 · Branch: `<branch>` · Generated from changes vs `<primary>`

## Summary

<2-3 plain sentences: what this change does and why, from a user's view. No code terms.>

## What changed

A short, abstract list. One line each. What a user would notice, not how it was built.

- <e.g. Document related endpoints renamed.>
- <change 2>

## What to test

For each change above, one line telling QA what to focus on and why.

- <e.g. Check all Document operations completely — endpoints were renamed, so make sure document actions still work.>
- <focus 2>

## How to test

One subsection per item from "What to test". Dive in here.

#### <Item title — e.g. Document CRUD>

- **The change:** <one or two plain lines about what changed in this area>
- **How to test:** <plain description of how QA should approach it>

| What to do | Expected |
| ---------- | -------- |
| <action, e.g. Open a document and edit its name> | <result, e.g. Name saves and shows updated> |
| <action> | <expected> |

- **Other cases to try:** <related things worth testing — list or "none">
- **Edge cases / watch out:** <unexpected situations, side effects to check — list or "none">

#### <Next item title>

- **The change:** ...
- **How to test:** ...

| What to do | Expected |
| ---------- | -------- |
| <action> | <expected> |

- **Other cases to try:** ...
- **Edge cases / watch out:** ...

## Regression — re-test these

Already-working features that this change might break. Re-test them.

| Area | Why affected | Severity | What to check |
| ---- | ------------ | -------- | ------------- |
| <feature user sees> | <plain reason> | High | <quick check> |
| <feature> | <plain reason> | Med | <quick check> |

<!-- If the trace found nothing, delete the table and write the single line below instead: -->
<!-- Nothing else is affected by these changes. -->

## Notes for QA

<Env setup, test data, feature flags, accounts, or "none". Anything QA needs before starting.>
```

---

## Rules

- **Write for a manual QA engineer who can't read code.** No code in the doc — no function names, file names, endpoint paths, language terms, or model/class names as code. Translate everything to what a user sees in the browser.
- **Stay abstract.** "What changed" and "What to test" are one short line each. Detail lives only in the "How to test" tables.
- Every "How to test" subsection reflects a "What to test" item, which reflects a "What changed" item. Keep them aligned.
- Every action in a "What to do / Expected" table has an Expected result.
- Always include **Other cases to try** and **Edge cases / watch out** for each item — write "none" if truly nothing.
- Don't invent a ticket ID. Omit the line if there's none.
- Don't pad regression with unrelated areas — only what the code trace found. If nothing, write the single "Nothing else is affected" line.
- Overwrite the file on re-run; latest changes win.
- Report only the doc — don't commit it yourself unless asked.
