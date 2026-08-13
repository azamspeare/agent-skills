---
name: defend-me
description: Audits PR review feedback or a GitHub issue against the actual repo. Fetches a PR review or an issue, breaks it into atomic items, verifies each claim against the real code, and reports them grouped as Must Have / Nice to have / Not necessary with short reasons. Report only — never edits code or posts replies. Invoke with "/defend-me <pr-review-url>", "/defend-me <issue-url>", "review this PR feedback", "audit these PR comments", "audit this issue", "is this review valid".
disable-model-invocation: true
effort: max
---

# defend-me

You're the defense. The reviewer (or issue) is the accuser. Take their claims against your code, examine each one against the evidence (the real code), and return a verdict per claim — guilty (real issue) or not.

Take a verbose PR review or a GitHub issue (the kind an AI reviewer produces), and turn it into a short, trustworthy verdict.

**The problem this solves:** Reviewers and issues come as walls of text. You can't tell which points are real. You then have to verify each one yourself, decide valid or not, and write a reply explaining what you fixed and what you didn't. This skill does the verifying for you and prints a short result.

**Core rule:** Do not trust the input. **Verify every claim against the actual code.** A reviewer (especially an AI one) invents issues, flags non-problems, and over-engineers. Your job is to check, not to agree.

**Scope:** Report only. **Never edit code. Never post replies or resolve threads on GitHub.** Just analyze and print.

---

## Step 0 — Be brief

All output must be short and simple. Plain words. A child should understand it.

- If the **caveman** skill is available, use it for all output.
- If not, still write short: simple terms, no jargon walls, no long paragraphs. One line per point.

Long explanations are the thing we are escaping. Don't recreate them.

---

## Step 1 — Parse the input

The input is a GitHub URL. First, find which kind it is by what's in the path:

| Path contains          | Kind          | Pull out                                                              |
| ---------------------- | ------------- | -------------------------------------------------------------------- |
| `/pull/` + `pullrequestreview-` | **PR review** | `owner`, `repo`, `pr`, `reviewId`                            |
| `/pull/` (no review id) | **whole PR**  | `owner`, `repo`, `pr` → fetch all reviews + review comments          |
| `/issues/`             | **issue**     | `owner`, `repo`, `issue`                                              |

Examples:
```
https://github.com/<owner>/<repo>/pull/<pr>#pullrequestreview-<reviewId>
https://github.com/<owner>/<repo>/issues/<issue>
```

If the user pasted **text** instead of a URL, skip fetching and use the pasted text as the body. Still verify against the repo (Step 3).

---

## Step 2 — Fetch the content

Check `gh` once:
```bash
gh --version >/dev/null 2>&1 && echo "gh:ok" || echo "gh:missing"
```
Missing → tell the user to install from https://cli.github.com, and ask them to paste the text instead. Then continue from pasted text.

Use the commands for the kind you found in Step 1.

**PR review** (has `reviewId`) — body + inline per-line comments:
```bash
gh api "repos/<owner>/<repo>/pulls/<pr>/reviews/<reviewId>" --jq '.body'
gh api "repos/<owner>/<repo>/pulls/<pr>/reviews/<reviewId>/comments" \
  --jq '.[] | {path, line, body}'
```

**Whole PR** (no review id) — all reviews + all review comments:
```bash
gh api "repos/<owner>/<repo>/pulls/<pr>/reviews" --jq '.[] | {id, user: .user.login, body}'
gh api "repos/<owner>/<repo>/pulls/<pr>/comments" --jq '.[] | {path, line, body}'
```

**Issue** — title, body, and discussion comments:
```bash
gh api "repos/<owner>/<repo>/issues/<issue>" --jq '{title, body}'
gh api "repos/<owner>/<repo>/issues/<issue>/comments" --jq '.[] | {user: .user.login, body}'
```

If the API errors (404 / not authed) → stop, show the error, ask the user to check `gh auth status` or paste the text.

---

## Step 3 — Audit each claim against the real code

This is the important step. Do **not** skip it.

1. **For a PR**, get the actual changes so you know what this PR really touched:
   ```bash
   gh pr diff <pr> --repo <owner>/<repo>
   ```
   **For an issue**, there is no diff — check the claim against the current repo as it is.
2. **Split the content into atomic items.** One issue = one item. A bullet that lists 3 things becomes 3 items. A "verified / no issue" summary line is **not** an item — skip praise and confirmations.
3. **For each item, open the real files and check the claim.** Use Read / Grep on the repo. Ask:
   - Does the code described actually exist?
   - For a PR: did **this PR** introduce it, or is it pre-existing?
   - Is the risk real, or theoretical with no real impact here?
   - Is the author over-engineering (asking for complexity the codebase doesn't need)?

Base every verdict on what you saw in the code — not on what the review or issue said.

---

## Step 4 — Categorize each item

Put every item in exactly one bucket, with a reason grounded in the code.

| Category         | Marker | Meaning                                                                                  |
| ---------------- | ------ | ---------------------------------------------------------------------------------------- |
| **Must Have**    | 🟥     | Real problem. Should be fixed. Affects correctness, security, data, or the request flow. |
| **Nice to have** | 🟨     | Real but minor. Fine to skip — no real impact on the codebase or the request flow.       |
| **Not necessary**| 🟩     | Not a real issue. Reviewer is wrong or over-engineered. Explain why it's wrong.          |

Rules:
- Don't soften. If it's not a real issue, say so and why.
- Don't inflate. Not every nitpick is "Must Have".
- Pre-existing issues this PR didn't introduce → usually **Nice to have** or note them as out-of-scope, unless they're truly dangerous.

---

## Step 5 — Output

Group the items under the three category headers, in this order: **🟥 Must Have → 🟨 Nice to have → 🟩 Not necessary**. Skip a header that has no items. Number items inside each category (`1)`, `2)`, …).

Exact shape:

```
🟥 Must Have

1) <short issue title>
<explanation — what's wrong, where it comes from, and why the verdict.
One flowing mini-paragraph, 1-3 short lines. NOT labeled sub-lines.>
- - - - - - - - - - - - - -
fix: <how to fix, brief and abstract — not full detail>
note: <only if genuinely needed — otherwise omit this line>

──────────────────────────

2) <next issue title>
...

🟨 Nice to have

1) ...

🟩 Not necessary

1) <short issue title>
<explanation — same shape>
- - - - - - - - - - - - - -
why: <why it's not worth doing — over-engineered, no real impact, wrong claim>
note: <only if needed>
```

Format rules:
- The explanation **merges** issue + root cause + reason into one short, plain caveman-style paragraph. Never split it into `issue:`/`root cause:`/`reason:` labeled lines — that's the old, hard-to-read shape.
- Dashed line (`- - -`) separates explanation from the action lines inside an item.
- Solid line (`───`) separates items inside the same category.
- **Must Have** and **Nice to have** items get a `fix:` line. **Not necessary** items get a `why:` line instead — never a fix.
- `note:` appears only when there is something extra worth saying (duplicate of another item, out-of-scope, pre-existing). No filler notes.
- Every line short. A child could read the whole report and get it.

End with a one-line tally:
```
Total: X must have · Y nice to have · Z not necessary
```

That's it. No long summary after. The blocks are the whole report.

---

## Example output (shape only)

```
🟨 Nice to have

1) Queued listener has no failed()
Listener ignores install errors — non-2xx response is swallowed, no failed()
handler. Copied from legacy SignatureSetup. Real but small: install retries
elsewhere, no data loss.
- - - - - - - - - - - - - -
fix: add ->throw() on the response + a failed() handler

🟩 Not necessary

1) uniqid() not a CSPRNG
Reviewer wants secure random for storage keys. Keys only name public-safe
images on a public CDN path — key secrecy buys nothing here.
- - - - - - - - - - - - - -
why: no security value — files are already public

Total: 0 must have · 1 nice to have · 1 not necessary
```

---

## Don't

- Don't edit code.
- Don't post replies or resolve threads.
- Don't trust the review without opening the files.
- Don't write long paragraphs. Short lines only.
- Don't list praise / "verified OK" lines as items.
