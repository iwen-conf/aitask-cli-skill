---
name: aitask-agent-watch
description: AITask 中以"特定 Agent 身份"消费本地 inbox 的守护进程/单次任务执行器。读取 `state.db` 中目标 Agent 名下未处理的事件，渲染 prompt，可选调用 runner（claude / codex / gemini / 自定义 `--exec`），并把执行结果写回 state.db（done / failed / skipped）。当需要配置自动唤醒、`--exec` handler 脚本、`--wake` runner 映射、调试 prompt 渲染或排查 inbox 状态机时使用。
version: 1.0.0
allowed_tools:
  - Bash
  - Read
  - Edit
---

# Skill: aitask-agent-watch

## 定位

`aitask-agent-watch` 是 AITask 中以"某个 Agent 的身份"消费本地 inbox 的守护进程/单次任务执行器。它读取 `~/.aitask/state.db` 中目标 Agent 名下未处理的事件，渲染 prompt，可选调用 runner（claude / codex / gemini / 自定义 `--exec`），并把执行结果写回 `state.db`（done / failed / skipped）。

它属于 Mailbox Worker Mode 的执行层，**不**订阅 WebSocket、**不**写 events.ndjson、**不**直接写 OpenViking 长期记忆。

## 负责什么

- 以 `--agent <name>` 拉取 `agent_inbox WHERE agent=<name> AND status IN ('unread','seen')`。
- 跳过自己发出的事件（`from == <name>`，避免回声）。
- 加锁，避免同一 event 被并行处理（`runtime/agent-watch/<agent>.lock` + `agent_inbox` 的 `seen_at` claim）。
- 在调用 runner 前先 ack 事件（防重复处理）。
- 通过 core 后端拉相关上下文（`/api/projects/.../context/event`、`thread`），调用 OpenViking 召回。
- 调 `cli/internal/cli/command_render_prompt.go` 同款渲染逻辑组装最终 prompt（可由 `aitask render-prompt` 复用）。
- 调 runner：
  - `--exec ./handlers/x.sh`：把 prompt 通过 stdin 喂给外部脚本（**生产推荐**，显式 handler）。
  - `--wake claude|codex|gemini`：启动对应 CLI 的一次性执行（详见 wake 表）。
  - `--dry-run`：只渲染 prompt 不调 runner。
- 捕获 stdout / stderr / exit code，把 stdout 当做"任务结果"写一条 `task_done` 事件入 `events.ndjson` 和 `memory_sync`。
- 状态机：`unread → seen → acked → handled` / `failed` / `skipped`，并维护 `retry_count`、`last_error`。
- 通过 `aitask done` / `aitask fail` / `aitask skip` 子命令操作状态（避免直接写 SQL）。
- 维护 `agent-watch:<name>` cursor。

## 不负责什么

- 接 WebSocket 拿原始事件——交给 `aitask-watch`。
- NDJSON → state.db 的索引——交给 `aitask-worker`。
- OpenViking 长期写入——交给 `aitask-worker` + core 后端。
- 决定哪个 runner 该处理什么 event 的"业务逻辑"——本进程只按 `--agent` / `--exec` / `--wake` 选项执行；高级路由由人/上游脚本配置。
- 跨 Agent 协作仲裁（"应该 A 还是 B 处理"）——服务端的 `to` 字段已决策。
- 直接接管正在运行的 REPL / 注入 stdin——**第一版禁止** stdin / tmux send-keys 注入活跃会话；只跑一次性 runner。

## 输入

| 来源 | 内容 |
| --- | --- |
| `~/.aitask/state.db` | `agent_inbox`、`events`、`cursors` |
| Core 后端 | `/api/projects/.../context/event`、`/thread`，OpenViking 召回结果 |
| `aitask context` / OpenViking | 召回上下文（只读） |
| `aitask render-prompt` | 标准 prompt 模板 |
| 命令行参数 | `--agent`、`--once`、`--exec`、`--wake`、`--dry-run`、`--interval`、`--timeout`、`--max-retries`、`--quiet` |
| 环境变量 | `AITASK_PROFILE` 决定默认 `--agent` |
| Runner 进程 stdin | prompt（由本进程注入） |

## 输出

- 写 `~/.aitask/state.db.agent_inbox`（status / retry_count / last_error / handled_at / failed_at）。
- 写 `~/.aitask/events.ndjson` 一条 `task_done` 事件（runner 成功时）。
- 写 `state.db.memory_sync(event_id, status='pending')` 让 `aitask-worker` 把结果同步到 OpenViking。
- 可选向服务端发布 `task_done` / `task_failed`（通过 `aitask task submit` / `aitask task fail`）。
- stdout：每条 event 的处理结果摘要（`event_id`、`status`、`runner_exit`、耗时）。
- stderr：runner 失败原因、prompt 渲染失败原因。

## 核心命令

```bash
aitask-agent-watch --agent claude-code --once --dry-run
aitask-agent-watch --agent claude-code --once --exec ./handlers/claude.sh
aitask-agent-watch --agent codex --wake codex
aitask-agent-watch --agent gemini --interval 5s --max-retries 5

# 兼容入口
aitask watch --agent claude-code ...
```

### `--wake` runner 映射

| `--wake` 值 | 实际命令 | 备注 |
| --- | --- | --- |
| `claude` / `claude-code` | `claude -p "$prompt"` | 一次性 headless 模式 |
| `codex` | `codex exec "$prompt"` | 一次性 exec 模式 |
| `gemini` | `gemini "$prompt"` | 一次性调用 |

`--exec <script>` 优先于 `--wake`。Script 接 stdin（prompt）+ stdout（结果）。生产推荐 `--exec`，显式 handler 比隐式 `--wake` 可控。

## 状态文件

- `~/.aitask/state.db.agent_inbox`：本进程是写入主体（与 `aitask ack/done/fail/skip` 共享）。
- `~/.aitask/runtime/agent-watch/<agent>.lock`：file lock，避免同一 Agent 多实例。
- `~/.aitask/events.ndjson`：写入 `task_done` 结果事件。
- `state.db.cursors WHERE consumer='agent-watch:<name>'`：本 watcher 进度。
- 不直接读 `~/.openviking/ovcli.conf`；上下文召回通过 core 后端。

## 与其他组件的关系

- 上游：`aitask-worker` 把事件路由进 `agent_inbox`。
- 下游：runner（claude / codex / gemini / 自定义脚本）的 stdin。
- 调用 core 后端做 context recall + memory write 代理。
- 通过 `aitask inbox` / `aitask ack` / `aitask done` / `aitask fail` / `aitask skip` 操作状态。
- 写 `task_done` 后，`aitask-worker` 下一轮会把它同步到 OpenViking。

## 常见流程

### 1. 单条事件处理

```text
1. acquire runtime/agent-watch/<agent>.lock
2. SELECT * FROM agent_inbox WHERE agent=? AND status IN ('unread','seen') ORDER BY created_at LIMIT N
3. for each row:
   a. UPDATE status='seen', seen_at=now
   b. acquire row claim (status='acked')
   c. fetch context: GET /api/projects/.../context/event?id=<event_id>
   d. render prompt via command_render_prompt 同款逻辑
   e. if --dry-run: print prompt, exit
   f. invoke runner (exec/wake) with prompt on stdin, capture stdout/stderr/exit
   g. on success: UPDATE status='handled', handled_at=now
                  append events.ndjson { kind:'task_done', body:stdout, ... }
                  INSERT memory_sync(event_id, status='pending')
   h. on failure: UPDATE status='failed', failed_at=now, last_error, retry_count++
4. release lock
```

### 2. 长驻

```bash
aitask-agent-watch --agent claude-code --interval 5s --exec ./handlers/claude.sh
# 每 5 秒走一次单次流程；SIGTERM 优雅退出
```

### 3. 调试

```bash
aitask-agent-watch --agent claude-code --once --dry-run --format prompt
# 只看会渲染出的 prompt 长啥样，不执行
```

### 4. 防止处理自己

```text
filter: from != $agent
原因：避免 echo 循环。
```

### 5. 并发安全

```text
- 同一 (event_id, agent) 由 UNIQUE 约束保证唯一。
- 状态变更使用 UPDATE WHERE status IN (...) 的 CAS 风格，避免覆盖更新。
- 多个 watcher 同名时 watch lock 拦截第二个。
```

## 失败与重试策略

- runner 退出码 ≠ 0：标 `failed`，`retry_count++`，`last_error=stderr_tail`，下个 cycle 再选。`retry_count > --max-retries` 后改 `skipped`，需人工 `aitask ack <id> --agent <name>` 复位。
- runner 超时：`--timeout`（默认 5min）触发 SIGKILL，按失败处理。
- prompt 渲染失败（context recall 失败）：fallback 不带召回的 prompt，记 warning；连续失败 `--max-retries` 次后转 `skipped`。
- 锁文件残留：进程意外退出会留 stale lock；新实例启动时检测 PID 不在则覆盖。
- ack 后 runner 还未执行就崩溃：下次启动看到 `status='acked'` 超过 N 分钟则降级回 `seen`。
- OpenViking 不可达：不影响本地 `handled` 标记，由 worker 下次重试同步。

## Agent 使用注意事项

- **绝对不要**用本进程注入正在跑的 REPL 的 stdin——`--exec` 调起独立短任务进程才是正确姿势。
- **第一版禁止** 用 tmux send-keys / stdin 注入正在运行的 REPL。
- `--wake` 模式要求宿主机上对应 Agent CLI 已安装；建议生产用 `--exec` 写显式 handler 脚本。
- 一个 Agent 名只能跑一份本进程，否则同一 event 可能被处理两次（lock 是软锁）。
- runner 的 stdout 会作为 `task_done.body` 写入 `events.ndjson` 并送 OpenViking，注意控制噪音（不要把整个工具调用日志原样输出）。
- 想测试 prompt 但不真跑：`--once --dry-run --format prompt`。
- 任何"全局广播"事件默认不会进 `agent_inbox`——只会进 `global_feed`，本进程不消费它。
- 不允许 watcher 互相 mention 制造死循环（路由层应已保证；watcher 自身仍 filter `from == self`）。
- 默认串行处理，单 watcher 进程同一时刻只跑一个 runner。
- 长任务：runner 应自行通过 `aitask task heartbeat` 续命，watcher 只关心退出码。
- 测试覆盖必须包含：
  1. ack 后 runner 崩溃 → 下次正确恢复；
  2. 同名 watcher 两次启动 → 第二个失败；
  3. self-mention 被忽略；
  4. runner 超时 → 进程被回收；
  5. OpenViking 不可用时 prompt 仍能渲染（无召回）。
