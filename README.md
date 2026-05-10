# aitask-cli skill bundle

A bundle of Claude Code / Anthropic skills that teach AI agents how to drive
the [`aitask`](https://github.com/iwen-conf/aitask-cli) CLI and its companion
daemons — the agent-facing orchestrator stack for the AITask platform.

## Skills in this repo

| Skill | Path | What it covers |
| --- | --- | --- |
| `aitask-cli` | [`SKILL.md`](SKILL.md) | The rich agent-facing skill: delegation matrix, every `aitask` subcommand, standard agent loop, recipes, failure modes, anti-patterns. |
| `aitask` | [`skills/aitask/SKILL.md`](skills/aitask/SKILL.md) | Chinese command-surface companion: `inbox`, `render-prompt`, `openviking config`, `search`, `summary`, compat entries (`events` / `worker` / `watch`). |
| `aitask-watch` | [`skills/aitask-watch/SKILL.md`](skills/aitask-watch/SKILL.md) | WebSocket event-stream daemon. NDJSON append, system notifications, hooks integration. |
| `aitask-worker` | [`skills/aitask-worker/SKILL.md`](skills/aitask-worker/SKILL.md) | Local indexer + OpenViking sync daemon. Includes the valuable-kinds whitelist and ingest pipeline. |
| `aitask-agent-watch` | [`skills/aitask-agent-watch/SKILL.md`](skills/aitask-agent-watch/SKILL.md) | Per-agent inbox executor. `--exec` / `--wake` runner invocation, state machine, prompt rendering. |
| `aitask-inbox` | [`skills/aitask-inbox/SKILL.md`](skills/aitask-inbox/SKILL.md) | The `aitask inbox` / `latest` / `thread` / `ack` / `done` / `fail` / `skip` query family. |

## Install

### Project-level (recommended, ships with the repo)

```bash
# Just the orchestrator skill
mkdir -p .claude/skills/aitask-cli
curl -fsSL https://raw.githubusercontent.com/iwen-conf/aitask-cli-skill/main/SKILL.md \
  -o .claude/skills/aitask-cli/SKILL.md

# Or pull the whole bundle (sparse-checkout style)
git clone --depth=1 https://github.com/iwen-conf/aitask-cli-skill.git /tmp/aitask-cli-skill
mkdir -p .claude/skills
cp /tmp/aitask-cli-skill/SKILL.md .claude/skills/aitask-cli/SKILL.md
cp -R /tmp/aitask-cli-skill/skills/. .claude/skills/
```

### User-global

```bash
mkdir -p ~/.claude/skills
git clone --depth=1 https://github.com/iwen-conf/aitask-cli-skill.git /tmp/aitask-cli-skill
mkdir -p ~/.claude/skills/aitask-cli
cp /tmp/aitask-cli-skill/SKILL.md ~/.claude/skills/aitask-cli/SKILL.md
cp -R /tmp/aitask-cli-skill/skills/. ~/.claude/skills/
```

## When each skill loads

| Skill | Triggers |
| --- | --- |
| `aitask-cli` | Repo contains `.aitask/project.md`; user mentions task pickup / submit / handoff / room / token bind; user types `aitask <subcommand>`. |
| `aitask` | Same triggers, used as the Chinese command-surface reference. |
| `aitask-watch` | User mentions WebSocket event subscription, NDJSON, `events.ndjson`, system notifications, hooks auto-launch, `aitask events`. |
| `aitask-worker` | User mentions `state.db` indexing, OpenViking sync, `--memory openviking`, daemon/once cycles, backfill, valuable kinds. |
| `aitask-agent-watch` | User mentions `--wake`, `--exec` handler scripts, prompt rendering, runner invocation, inbox state machine. |
| `aitask-inbox` | User mentions `aitask inbox`, `latest`, `thread`, `ack` / `done` / `fail` / `skip`, or "为什么我的 inbox 是空的". |

## License

MIT
