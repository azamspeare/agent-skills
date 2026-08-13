# 🛠️ My Agent Skills

A collection of personal agent skills for [Claude Code](https://code.claude.com) and other coding agents — built around my own workflow and conventions.

## Install

Requires [Node.js](https://nodejs.org). No global install needed.

```bash
# Install a specific skill
npx skills add azamspeare/agent-skills --skill blame-qa

# Install all skills at once
npx skills add azamspeare/agent-skills --all

# Install globally (available in every project)
npx skills add azamspeare/agent-skills --all -g
```

## Remove

```bash
# Remove a skill from the current project
npx skills remove blame-qa

# Remove a globally installed skill
npx skills remove blame-qa -g

# Remove several at once
npx skills remove take-changes ci-crashed -g
```

> **Note:** when a skill gets renamed in this repo, the old name stays installed on your machine — the installer treats the new name as a brand-new skill. Remove the old one, then re-install:
>
> ```bash
> npx skills remove <old-name> -g
> npx skills add azamspeare/agent-skills --all -g
> ```

---

## Skills

| Skill                                         | Description                                                                                                                                                                                                                                          |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`take-changes`](./take-changes/SKILL.md)     | Commits changes, pushes to remote, and creates or updates a GitHub PR. Handles protected branch detection, conventional commits, and auto-generated PR descriptions. Sweeps staged changes for explanation/guide comments and offers to delete them before pushing. Accepts an optional branch name (`/take-changes [branch]`) — validates it and skips suggestions. Trigger: _"/take-changes"_, _"pls commit"_, _"push my changes"_, _"create a PR"_ |
| [`defend-me`](./defend-me/SKILL.md)           | Audits PR review feedback **or a GitHub issue** against the real code — defends your code against the reviewer's claims. Verifies each claim and reports them grouped as **🟥 Must Have / 🟨 Nice to have / 🟩 Not necessary** with short, plain reasons. Report only. Trigger: _"/defend-me &lt;pr-review-url&gt;"_, _"/defend-me &lt;issue-url&gt;"_, _"audit this PR feedback"_ |
| [`blame-qa`](./blame-qa/SKILL.md)             | Generates a QA test doc from your branch changes. Code-traces regression impact and writes `docs/qa/{feature}-test-doc.md` with numbered test steps, expected results, and severity-tagged regression areas. Trigger: _"/blame-qa"_, _"write QA doc"_, _"what should QA test"_ |
| [`ci-crashed`](./ci-crashed/SKILL.md)         | Fixes a failing CI / GitHub Actions job end to end. Audits the failed logs, finds the root cause, fixes it like a senior engineer (security-aware), auto-commits + pushes, and watches the run until green — retrying while it makes progress, stopping if it's not code-fixable. Reports each fix as short before/after blocks. Trigger: _"/ci-crashed &lt;run-url&gt;"_, _"tests failed &lt;url&gt;"_, _"CI is red"_ |

---

## Compatibility

Works with Claude Code, Cursor, OpenCode, Codex, and [41+ other agents](https://github.com/vercel-labs/skills).

---

## License

MIT — use freely, modify as you like.
