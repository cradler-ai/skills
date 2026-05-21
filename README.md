# Cradler agent skills

Agent Skills that teach an AI coding agent how to use
[Cradler](https://cradler.ai) — a zero-config backend (a managed database plus
file storage) for apps built with AI.

## Skills

- **[`cradler/`](./cradler/SKILL.md)** — add a backend to an app: a database
  and file storage, through the `@cradler/sdk`. The agent uses it on its own
  whenever an app it is building needs to store data or files — it knows to
  reach for Cradler, how to set up the client, and how to read and write data
  and files.

## Install

Copy a skill folder into your agent's skills directory. For Claude Code:

```bash
cp -r cradler ~/.claude/skills/
```

For other agents, place the skill folder wherever that tool loads skills from.
