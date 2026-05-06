# aitask-cli skill

A Claude Code / Anthropic skill that teaches AI agents how to drive the
[`aitask`](https://github.com/iwen-conf/aitask-cli) CLI — the agent-facing
orchestrator for the AITask platform.

## Install

Drop the skill into your Claude Code skills directory:

```bash
# project-level (recommended, ships with the repo)
mkdir -p .claude/skills/aitask-cli
curl -fsSL https://raw.githubusercontent.com/iwen-conf/aitask-cli-skill/main/SKILL.md \
  -o .claude/skills/aitask-cli/SKILL.md

# or user-global
mkdir -p ~/.claude/skills/aitask-cli
curl -fsSL https://raw.githubusercontent.com/iwen-conf/aitask-cli-skill/main/SKILL.md \
  -o ~/.claude/skills/aitask-cli/SKILL.md
```

## What's inside

The skill is a single `SKILL.md` covering:

- Hard rules of the AITask delegation model.
- Every `aitask` subcommand verified against the CLI source.
- The standard agent loop (bootstrap → task current → start → submit).
- `.aitask/` workspace file map (what to edit, what not to touch).
- Recipes for context handoff, memory writes, cross-agent room asks.
- Failure-mode table mapping CLI error strings to fixes.
- Anti-patterns the agent must avoid.

## Trigger conditions

Claude Code auto-loads the skill when:

- The repo contains `.aitask/project.md`.
- The user mentions picking up tasks, submitting results, compacting
  context, creating a handoff, joining the room, or binding an agent token.
- The user types `aitask <subcommand>` directly.

## License

MIT
