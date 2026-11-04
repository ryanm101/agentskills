# Agent Skills

Personal collection of skills for AI coding agents (Claude Code, etc.).

## Skills

| Skill | Description |
|---|---|
| [`evidence-driven-reverse-engineering`](skills/evidence-driven-reverse-engineering/SKILL.md) | Disciplined, evidence-based rigor for reverse engineering opaque targets — binaries, protocols, file formats, firmware. Requires [REA](https://github.com/morluto/rea) and [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills). |
| [`re`](skills/re/SKILL.md) | Quick, safe reverse-engineering pass on a vendor binary symbol. |

## Install

Via the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add ryanm101/agentskills             # install all skills
npx skills add ryanm101/agentskills --skill re  # install one skill
```

Or clone and copy a `skills/<name>/` directory into your agent's skills path (e.g. `~/.claude/skills/`).
