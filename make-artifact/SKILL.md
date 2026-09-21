---
name: make-artifact
description: Explains a new feature to non-technical teammates (marketing, support, sales, ops) as a shareable doc they can read in 5 minutes and then test themselves. Reads the code to learn how the feature really works, asks only what code can't answer, and writes a plain-words doc — what it is, what changed, what each plan/role gets, how to try it, and a tick-box test checklist. Publishes a Claude Docs artifact (falls back to a markdown file). Invoke with "/make-artifact", "explain this feature to the team", "write a feature doc", "make an artifact for support", "how do I tell the team about this".
disable-model-invocation: true
effort: high
---

# make-artifact

Explain a new feature to people who do not read code, in a doc they can read fast and act on.

**Reader:** a marketer, a support agent, a salesperson, an ops person. Smart. Busy. Not technical.
**Assume the reader has ADHD.** Short lines. Tables over paragraphs. Every section answers one question.

**Output:** a Claude Docs artifact (shareable link). Falls back to `docs/features/{feature-name}.md` if Claude Docs is not available.

**Not the same as `blame-qa`.** blame-qa writes for a QA engineer testing a branch. This writes for the whole team learning a feature — what it is, why it exists, what it costs, who gets it, and how to try it.

---

## Step 0 — The voice rule (applies to every word you write)

Write so a smart 12-year-old gets it on the first read.

| Do | Don't |
| --- | --- |
| "A room where documents live" | "A tenant-scoped resource container" |
| "Only the owner can pay" | "Billing operations are gated by the owner role" |
| "It stops working until someone pays" | "The workspace enters a restricted state" |
| "$13 a month each" | "Incremental seat pricing applies" |

Hard rules:

- **No code words.** No function names, file names, endpoint paths, database tables, class names, language names.
- **No product-manager words.** No "leverage", "seamless", "robust", "empower", "streamline", "unlock".
- **Short sentences.** Under 20 words. One idea each.
- **Name the thing the user sees.** Say the real button label, the real badge text, the real screen name — in quotes: press **Switch to Team**, the badge says "Unpaid".
- **Short, but not thin.** Never drop a rule the reader needs just to be brief. If a rule is fiddly, give it its own line — don't bury it in a sentence.
- If the **caveman** skill is installed, do NOT use it here. This doc goes to non-technical colleagues; it must read like normal, kind English.

---

## Step 1 — Learn the feature from the code

Do this before asking the user anything. Come to them already knowing most of it.

If the feature is on the current branch:

```bash
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name' 2>/dev/null \
  || git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@'
git fetch origin <primary> >/dev/null 2>&1
git log origin/<primary>..HEAD --pretty="%s"
git diff --name-only origin/<primary>...HEAD
git diff origin/<primary>...HEAD
```

If the user named a feature that is already merged, search the codebase for it instead (models, config, routes, UI strings, migrations).

**Dig for the facts a teammate will ask about.** Not the implementation — the rules:

1. **Limits and numbers** — prices, quotas, caps, seat counts, file sizes, day counts, expiry windows. Read them from config, plan tables, seeders, constants. Never guess a number.
2. **Roles and permissions** — who can do what. Read policies, gates, middleware, role checks.
3. **States** — what states the thing can be in, what each one blocks. Read enums, status columns.
4. **Messages the user sees** — error text, badge labels, banner copy, button labels, email subjects. Read them from translation files / view templates and quote them exactly.
5. **Entry points** — where in the app someone starts this. Which screen, which button.
6. **Edge rules** — what is refused, what expires, what cannot be undone, what happens on failure.

Write these down. They become the tables in the doc.

## Step 2 — Ask only what the code cannot answer

Ask in **one** message, max 5 questions, each with your best guess so the user can just say "yes".

Only ask what you truly could not find:

- Who is this for / why was it built (the business reason)
- Who gets it and when (all users? paid only? gradual rollout? date?)
- Where should the team test it (staging URL, test accounts, a feature flag to switch on)
- Anything the code shows two ways of doing and you can't tell which ships

If the code answered everything, skip this step and say so.

## Step 3 — Confirm the title

Propose a title that a teammate would say out loud: `Team workspaces: how they work`, `Credits: buy, spend, run out`.

> I'll write this as **"<proposed title>"**. Good, or a different name?

Filename slug (for the fallback file) = lowercase, hyphens, ≤ 4 words.

## Step 4 — Write the doc

**Publish a Claude Docs artifact.** Create it first with a skeleton (title, byline, one pending block per section), open it for the user, then fill **one section per call** while they watch. Follow the Claude Docs connector guidance the harness provides.

- Byline line = as-of date chip · the user.
- Use pipe tables for anything with a shape (plans, roles, states, badges).
- Use a check-list (`- [ ]`) for the test checklist so the reader can tick it.

**Fallback:** if Claude Docs is unavailable, write `docs/features/{feature-name}.md` with the same structure (`mkdir -p docs/features`).

## Step 5 — Report

```
✅ Feature doc ready: <link or path>
   Title:      <title>
   Sections:   <n>
   Test steps: <n> tick-boxes
   Open questions: <n>   (or "none")
```

If anything is still unknown, list those open questions — do not invent an answer.

---

## Document structure

Use the sections that fit. Drop the ones that don't. Never add sections the feature doesn't need.

### 1. Title + one-line byline

`# <Feature>: how it works` then the date chip · the author.

### 2. The opening paragraph (2–4 sentences, no heading)

What it is, in the simplest words, plus what it replaces.

> A workspace is a room where documents, recipients and files live. Until now every room had one person in it. Team workspaces let several people work in the same room, each with their own role.

This is the most important paragraph in the doc. Rewrite it until a stranger gets it in one read.

### 3. Why we built it (only if not obvious)

One or two lines. The customer problem, not the roadmap.

### 4. The core idea — a comparison table

If the feature adds a new kind of something, compare old vs new side by side.

| | Personal | Team |
| --- | --- | --- |
| Who works in it | One person | The owner plus invited people |
| Can invite people | No | Yes |

### 5. Plans and prices (if money is involved)

Every number the reader might be asked by a customer. One column per plan, one row per limit. Put the fine print in one line under the table ("Paying yearly costs the same as 10 months.").

### 6. Who can do what (if roles are involved)

One column per role, one row per action, cells are Yes / No / a short phrase. Under the table, add the rules a table can't show ("The owner cannot be removed.").

### 7. How it works, step by step

Numbered list for the main flow — what the user does, what happens next.

Then a short bullet list of the rules that bite: what expires, what is counted, what is refused.

### 8. What happens when things go wrong

The states, the badges, the blocked actions. A small table of badge → meaning works well. Say plainly what still works and what stops.

### 9. Test it yourself — the checklist

The part that turns readers into testers. Open with the setup in one line ("Use two test accounts with different emails.").

Group tick-boxes under short bold headings. Each tick-box is **one action and its expected result in one sentence**:

**Creating workspaces**
- [ ] Sign up: a free personal workspace is there.
- [ ] Create a second personal workspace: it is locked, shows "Unpaid" and asks for Starter.

Rules for the checklist:

- Start with the happy path, then the limits, then the failures.
- Every line must be doable by someone with only the app open — no terminal, no database.
- Quote the exact text they should see.
- Aim for 10–25 tick-boxes. If it needs more, the doc covers too much — split it.

### 10. Open questions (only if there are any)

Short list of what nobody has decided yet. Better than a confident wrong answer.

---

## Rules

- **Plain words only.** No code terms, no jargon, no marketing filler. Test: would a support agent understand this on their first day?
- **Short but complete.** Cut words, never rules. A missing rule creates a support ticket.
- **Never invent a number.** Prices, limits, day counts and quotas come from the code or from the user. If unknown, it goes in Open questions.
- **Quote what the user sees** — button labels, badge text, error messages — exactly as the code has them.
- **Tables carry the facts.** Anything with rows and columns (plans, roles, states) goes in a table, never in paragraphs.
- **Every doc ends with a checklist.** A doc the team can't act on is a doc they won't read.
- **One feature per doc.** If it's two features, write two docs.
- **Tick-boxes are one action each.** No "and then" twice in a line.
- Read the code before asking the user. Ask once, batched, with your best guess.
- Overwrite on re-run; latest version wins.
