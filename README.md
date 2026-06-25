# 🛠️ My Agent Skills

A collection of personal agent skills for [Claude Code](https://code.claude.com) and other coding agents — built around my own workflow and conventions.

## Install

Requires [Node.js](https://nodejs.org). No global install needed.

```bash
# Install a specific skill
npx skills add azamspeare/agent-skills --skill take-shit

# Install all skills at once
npx skills add azamspeare/agent-skills --all

# Install globally (available in every project)
npx skills add azamspeare/agent-skills --all -g
```

---

## Skills

| Skill                                         | Description                                                                                                                                                                                                                                          |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`take-shit`](./take-shit/SKILL.md)           | Commits changes, pushes to remote, and creates or updates a GitHub PR. Handles protected branch detection, conventional commits, and auto-generated PR descriptions. Accepts an optional branch name (`/take-shit [branch]`) — validates it and skips suggestions. Trigger: _"/take-shit"_, _"pls commit"_, _"push my changes"_, _"create a PR"_ |
| [`defend-me`](./defend-me/SKILL.md)           | Audits PR review feedback **or a GitHub issue** against the real code — defends your code against the reviewer's claims. Verifies each claim and reports every item as **Must have / Preferable / Invalid** with a short reason. Report only. Trigger: _"/defend-me &lt;pr-review-url&gt;"_, _"/defend-me &lt;issue-url&gt;"_, _"audit this PR feedback"_ |
| [`blame-qa`](./blame-qa/SKILL.md)             | Generates a QA test doc from your branch changes. Code-traces regression impact and writes `docs/qa/{feature}-test-doc.md` with numbered test steps, expected results, and severity-tagged regression areas. Trigger: _"/blame-qa"_, _"write QA doc"_, _"what should QA test"_ |

---

## Compatibility

Works with Claude Code, Cursor, OpenCode, Codex, and [41+ other agents](https://github.com/vercel-labs/skills).

---

## License

MIT — use freely, modify as you like.
