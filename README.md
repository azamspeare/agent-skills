# 🛠️ My Agent Skills

A collection of personal agent skills for [Claude Code](https://code.claude.com) and other coding agents — built around my own workflow and conventions.

## Install

Requires [Node.js](https://nodejs.org). No global install needed.

```bash
# Install a specific skill
npx skills add azamspeare/agent-skills --skill commit-changes

# Install all skills at once
npx skills add azamspeare/agent-skills --all

# Install globally (available in every project)
npx skills add azamspeare/agent-skills --all -g
```

---

## Skills

| Skill                                         | Description                                                                                                                                                                                                                        |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`commit-changes`](./commit-changes/SKILL.md) | Commits changes, pushes to remote, and creates or updates a GitHub PR. Handles protected branch detection, conventional commits, and auto-generated PR descriptions. Trigger: _"pls commit"_, _"push my changes"_, _"create a PR"_ |

---

## Compatibility

Works with Claude Code, Cursor, OpenCode, Codex, and [41+ other agents](https://github.com/vercel-labs/skills).

---

## License

MIT — use freely, modify as you like.
