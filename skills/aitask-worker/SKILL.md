---
name: aitask-worker
description: AITask 本地索引与记忆同步守护进程。流式消费 `aitask-watch` 写入的 `~/.aitask/events.ndjson`，规范化后落到 `~/.aitask/state.db`，把语义子集异步同步到 OpenViking。当需要排查事件未入库、回填历史事件、调整 `--memory` 模式、配置 daemon/once 跑法、理解 valuable kinds 白名单时使用。
version: 1.0.0
allowed_tools:
  - Bash
  - Read
  - Edit
---

# Skill: aitask-worker

## 定位

`aitask-worker` 是 AITask 本地索引与记忆同步守护进程。它消费 `aitask-watch` 写入的 `~/.aitask/events.ndjson`，把事件规范化后落到 `~/.aitask/state.db`，并把语义子集异步同步到 OpenViking（通过 core 后端 `/api/projects/{id}/memory/...` 代理）。

它属于 Mailbox Worker Mode 的引擎层。

## 负责什么

- 流式消费 `~/.aitask/events.ndjson`，按 cursor 推进。可重放、可重启、幂等（`INSERT OR IGNORE` / UPSERT）。
- 规范化事件字段：`id` / `kind` / `scope` / `project` / `thread_id` / `from` / `to` / `body` / `wake` / `created_at` / `metadata`。
- 路由：发给 Agent 的进 `agent_inbox`、`scope=global` 或 `visibility=broadcast` 进 `global_feed`、其它进 `events` 主表。
- 维护各消费者 `cursors`（`worker:indexer` / `worker:openviking` / `worker:summarizer`）。
- 把高价值事件（见下方白名单）入 `memory_sync` 队列。
- 通过 core 后端把 `memory_sync.pending` 行批量同步到 OpenViking，写回 `memory_id` 和 `status`。
- 失败计数与重试退避（`memory_sync.retry_count`、`last_error`）。
- 可选生成 thread / project / agent 的轻量摘要写到 `summaries`。

## 不负责什么

- WebSocket 订阅与 NDJSON 写入——交给 `aitask-watch`。
- 决定事件该不该被 Agent 处理——交给 `aitask-agent-watch`。
- 把 ack / handled / failed / skipped 状态写到 OpenViking——这些是状态数据，永远只在 `state.db`。
- 唤醒任何 CLI / runner。
- TUI / 交互。
- OpenViking 长期存储治理（GC、配额）→ OpenViking 自身。
- 给 hook 提供事件源 → hook 直接读 events.ndjson。

## 输入

| 来源 | 内容 |
| --- | --- |
| `~/.aitask/events.ndjson` | append-only 事件流（主输入） |
| `~/.aitask/state.db` | 上次的 cursor / memory_sync / summaries 状态 |
| Core 后端 | OpenViking 写入代理（per-project 设置 + memory write API） |
| `~/.openviking/ovcli.conf` 或后端项目设置 | OpenViking 连接信息（已通过 backend 抽象） |
| 命令行参数 | `--once`、`--daemon`、`--memory openviking|none`、`--batch`、`--backfill-since`、`--limit`、`--dry-run`、`--quiet`、`--replay-from start` |
| 配置开关 | `worker.enable_openviking` / `worker.summary_strategy` |

## 输出

- 写 `~/.aitask/state.db` 的 events / agent_inbox / global_feed / cursors / memory_sync / summaries 表。
- 向 OpenViking 写入 memory / resource。
- stdout：每个 cycle 的 stats（`ingested=`、`routed=`、`memory_synced=`、`failed=`）。
- stderr：错误日志，带 `event_id` / `cause`。
- exit code：`--once` 完成 0；`--daemon` 收到 SIGTERM 退出 0。

## 核心命令

```bash
aitask-worker --once --memory none           # 仅本地索引，不打 OpenViking
aitask-worker --once --memory openviking     # 索引 + 同步一轮
aitask-worker --daemon --memory openviking   # 长驻

aitask-worker --backfill-since 2026-05-01T00:00:00Z --limit 1000  # 历史事件回填
aitask-worker --backfill-since 2026-05-01T00:00:00Z --dry-run     # 仅输出候选 + 统计，不写 memory_sync
aitask-worker --once --dry-run               # 预演，不写库
aitask-worker --replay-from start            # 强制全量重放（建库后初始化）
aitask sync --memory openviking              # 仅同步，不 ingest
```

兼容入口：

```bash
aitask worker ...   # 等价
```

## Valuable kinds（进入 `memory_sync` 的事件白名单）

按字典序：

- `broadcast`
- `context.handoff_created`
- `context_handoff`
- `memory_note`
- `mention`
- `note`
- `reply`
- `room.decision_pinned`
- `room.message`
- `room_message`
- `summary`
- `system_event`
- `task.delegated`
- `task.failed`
- `task.review_passed`
- `task.review_rejected`
- `task.review_task_created`
- `task.reviewed`
- `task.started`
- `task.submitted`
- `task.updated`
- `task_delegated`
- `task_done`
- `task_updated`

## 状态文件

- `~/.aitask/state.db`：本进程是写入主体之一（与 `aitask` 的 ack/done/fail/skip 共享）。
- `~/.aitask/runtime/worker.lock`：daemon 模式下的 file lock，避免重复实例。
- `~/.aitask/events.ndjson`：只读，配合 `cursors.consumer='worker:indexer'` 推进 offset。

## 与其他组件的关系

- 上游：`aitask-watch` 的 NDJSON。
- 下游：
  - `aitask inbox/latest/thread`（`aitask-inbox` 子命令族）读 `state.db`。
  - `aitask-agent-watch` 读 `agent_inbox` 拿待处理事件。
  - `aitask search/context/summary` 联合查询 `state.db` + OpenViking。
- 调用 core 后端的 `/api/projects/{id}/memory/write` 等 REST 端点；OpenViking 不可达时把对应行标 `failed`，下个周期重试。
- 同侪：与多个 hook 并存，cursor 各自独立。

## 常见流程

### 1. 单次同步（ingest 一批新事件）

```text
1. lock runtime/worker.lock
2. SELECT cursors WHERE consumer='worker:indexer'
3. tail events.ndjson 自 offset 起
4. for each line:
   a. JSON 解析 + 字段规范化
   b. INSERT OR IGNORE INTO events  (idempotent on id)
   c. 路由到 agent_inbox / global_feed
   d. 高价值事件 -> INSERT memory_sync(status='pending')
5. UPDATE cursors offset / event_id / updated_at
6. if --memory openviking:
   a. SELECT memory_sync WHERE status='pending' LIMIT batch
   b. JOIN events 拿正文
   c. POST /api/projects/{id}/memory/write
   d. UPDATE memory_sync SET status='synced'/'failed', retry_count, last_error, openviking_id
7. release lock
```

### 2. 历史回填

```bash
aitask-worker --backfill-since 2026-05-01T00:00:00Z --limit 1000 --memory openviking
```

只重读 `events.ndjson` 中 created_at >= since 的行，把没在 `memory_sync` 里的入队，触发同步。回填与 `--once` / `--daemon` 共用同一把 worker lock，不会并发 ingest。

### 3. daemon 长驻

```bash
aitask-worker --daemon --memory openviking
# 内部循环：每 N 秒跑一次 --once 流程；收到 SIGTERM/SIGINT 优雅退出
```

### 4. summary 刷新

```text
触发：thread 新增 N 条事件 / project 周期 / agent 周期
1. 拉相关事件正文
2. 调用 LLM 或本地摘要器生成
3. UPSERT summaries(scope, scope_id, summary)
4. 把 summary 也作为一条 OpenViking memory 同步
5. summaries.memory_id 记录 OpenViking 侧 ID
```

## 失败与重试策略

- ingest 错误（解析失败）：写入 `events.raw_json` 并标 `kind='unknown'`，offset 仍推进，不阻塞流。
- `state.db` 锁：SQLite WAL + 短事务，写失败指数退避 ≤3 次，仍失败下一个 cycle 再试。
- OpenViking 不可用：标记 `failed`，`retry_count++`，上限默认 5；超过后转 `pending-cooldown`，每 N 分钟回收。
- 401 / 403：标 `failed` 不重试，stderr 提示用户检查项目 OpenViking 设置或 `aitask openviking config import`。
- summary 生成失败：保留旧 summary，不覆盖。
- daemon 重启 / 异常退出：因为所有进度走 `cursors.offset`，可重放、可恢复；worker.lock 自动失效。
- 不重复入库：`events.id` PRIMARY KEY；`memory_sync.event_id` PRIMARY KEY。

## Agent 使用注意事项

- 不要让两个 worker 同时跑——`runtime/worker.lock` 是软锁，靠你别绕过。
- 不要把 `--dry-run` 用作"演练后立即 commit"——它不写库，不更新 cursor，可能造成误以为的"已处理"。
- 长事件正文（>4KB）请考虑预先在 `aitask-watch` 截断，或在 worker 入 `memory_sync` 前打 `summary` 标签。
- 不要把 ack / handled / 自定义状态机写入 OpenViking——OpenViking 是记忆，不是消息队列。
- `--memory none` 适合离线机器或 CI 烟雾测试。
- 不要在 worker 里发送通知或调用 runner，那是 agent-watch 的职责。
- 不要把"我自己写的事件"自动 ack：worker 完全不操作 inbox.status。
- 不要把 worker 与 watch 合并成一个进程——它们的失败域必须隔离。
- 测试覆盖必须包含：
  1. 重放幂等；
  2. 部分行损坏时其余行继续 ingest；
  3. OpenViking 拒绝（401/429/500）时 `retry_count` 正确；
  4. cursor 落后于 NDJSON 实际大小时正确续跑；
  5. 同时启动两个 worker → 第二个被 worker.lock 拦截。
