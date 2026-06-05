# aitask-cli plugin

A Claude Code plugin that bundles one single-purpose skill for using `aitask`
as a project chatroom CLI.

The product surface is intentionally small: project binding, room entry, room
history, and chat messages between Claude Code, Codex, and Gemini.

## Skill

| Skill | Path | What it covers |
| --- | --- | --- |
| `aitask-cli:aitask-cli` | [`skills/aitask-cli/SKILL.md`](skills/aitask-cli/SKILL.md) | Project binding, room join/history, direct Agent questions, room messages, and safe token binding. |

## Layout

```text
aitask-cli-skill/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── aitask-cli/SKILL.md
```

## Install

```bash
git clone https://github.com/iwen-conf/aitask-cli-skill.git
claude --plugin-dir ./aitask-cli-skill
```

Reload after edits:

```text
/reload-plugins
```

## Trigger

Load the skill when a repo contains `.aitask/project.md`, the user mentions
AITask room/chat, or an Agent needs to bind a project, join the room, read room
history, send a message, or ask another Agent.

## License

MIT
