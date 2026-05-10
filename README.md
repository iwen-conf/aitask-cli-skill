# aitask-cli plugin bundle

A Claude Code plugin that bundles six skills for driving the
[`aitask`](https://github.com/iwen-conf/aitask-cli) CLI and its companion
daemons — the agent-facing orchestrator stack for the AITask platform.

## Skills in this plugin

| Skill (namespaced) | Path | What it covers |
| --- | --- | --- |
| `aitask-cli:aitask-cli` | [`skills/aitask-cli/SKILL.md`](skills/aitask-cli/SKILL.md) | Top-level orchestrator skill: delegation matrix, every `aitask` subcommand, standard agent loop, recipes, failure modes, anti-patterns. |
| `aitask-cli:aitask` | [`skills/aitask/SKILL.md`](skills/aitask/SKILL.md) | Chinese command-surface companion: `inbox`, `render-prompt`, `openviking config`, `search`, `summary`, compat entries (`events` / `worker` / `watch`). |
| `aitask-cli:aitask-watch` | [`skills/aitask-watch/SKILL.md`](skills/aitask-watch/SKILL.md) | WebSocket event-stream daemon. NDJSON append, system notifications, hooks integration. |
| `aitask-cli:aitask-worker` | [`skills/aitask-worker/SKILL.md`](skills/aitask-worker/SKILL.md) | Local indexer + OpenViking sync daemon. Includes the valuable-kinds whitelist and ingest pipeline. |
| `aitask-cli:aitask-agent-watch` | [`skills/aitask-agent-watch/SKILL.md`](skills/aitask-agent-watch/SKILL.md) | Per-agent inbox executor. `--exec` / `--wake` runner invocation, state machine, prompt rendering. |
| `aitask-cli:aitask-inbox` | [`skills/aitask-inbox/SKILL.md`](skills/aitask-inbox/SKILL.md) | The `aitask inbox` / `latest` / `thread` / `ack` / `done` / `fail` / `skip` query family. |

## Layout

```
aitask-cli-skill/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    ├── aitask-cli/SKILL.md
    ├── aitask/SKILL.md
    ├── aitask-watch/SKILL.md
    ├── aitask-worker/SKILL.md
    ├── aitask-agent-watch/SKILL.md
    └── aitask-inbox/SKILL.md
```

Skills are auto-discovered — no need to list them in `plugin.json`.

## Install

### Local development / testing

Clone and load with `--plugin-dir`:

```bash
git clone https://github.com/iwen-conf/aitask-cli-skill.git
claude --plugin-dir ./aitask-cli-skill
```

Reload after edits:

```text
/reload-plugins
```

### Via marketplace

Once published to a marketplace, install with:

```text
/plugin install aitask-cli@<marketplace-name>
```

See [Discover and install plugins](https://code.claude.com/docs/en/discover-plugins).

### Manual project-level install (no marketplace)

```bash
git clone --depth=1 https://github.com/iwen-conf/aitask-cli-skill.git /tmp/aitask-cli-skill
mkdir -p .claude/plugins
cp -R /tmp/aitask-cli-skill .claude/plugins/aitask-cli
```

### Manual user-global install

```bash
git clone --depth=1 https://github.com/iwen-conf/aitask-cli-skill.git /tmp/aitask-cli-skill
mkdir -p ~/.claude/plugins
cp -R /tmp/aitask-cli-skill ~/.claude/plugins/aitask-cli
```

## When each skill loads

| Skill | Triggers |
| --- | --- |
| `aitask-cli:aitask-cli` | Repo contains `.aitask/project.md`; user mentions task pickup / submit / handoff / room / token bind; user types `aitask <subcommand>`. |
| `aitask-cli:aitask` | Same triggers, used as the Chinese command-surface reference. |
| `aitask-cli:aitask-watch` | User mentions WebSocket event subscription, NDJSON, `events.ndjson`, system notifications, hooks auto-launch, `aitask events`. |
| `aitask-cli:aitask-worker` | User mentions `state.db` indexing, OpenViking sync, `--memory openviking`, daemon/once cycles, backfill, valuable kinds. |
| `aitask-cli:aitask-agent-watch` | User mentions `--wake`, `--exec` handler scripts, prompt rendering, runner invocation, inbox state machine. |
| `aitask-cli:aitask-inbox` | User mentions `aitask inbox`, `latest`, `thread`, `ack` / `done` / `fail` / `skip`, or "为什么我的 inbox 是空的". |

## License

MIT
