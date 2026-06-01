# aitask-cli plugin

A Claude Code plugin that bundles one skill for driving the
[`aitask`](https://github.com/iwen-conf/aitask-cli) CLI and its companion
daemons — the agent-facing orchestrator stack for the AITask platform.

## Skill in this plugin

| Skill (namespaced) | Path | What it covers |
| --- | --- | --- |
| `aitask-cli:aitask-cli` | [`skills/aitask-cli/SKILL.md`](skills/aitask-cli/SKILL.md) | Unified orchestrator skill: delegation matrix, `aitask` command surface, daemon roles, standard agent loop, recipes, failure modes, and anti-patterns. |

## Layout

```
aitask-cli-skill/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── aitask-cli/SKILL.md
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
| `aitask-cli:aitask-cli` | Repo contains `.aitask/project.md`; user mentions task pickup / submit / handoff / room / token bind; user types `aitask <subcommand>`; user asks about AITask daemons, inbox, worker, watcher, project context, or agent wake flows. |

## License

MIT
