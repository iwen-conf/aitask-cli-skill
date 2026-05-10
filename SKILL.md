---
name: aitask-cli
description: AITask Agent 编排与本地协作系统统一入口（包含 CLI 与守护进程族）。当 AI Agent 需要初始化工作区、管理任务生命周期（Delegation）、消费并处理 Inbox 消息（Mailbox Worker 模式）、管理上下文与 OpenViking 记忆，或在项目聊天室中协作时使用。触发条件：仓库包含 `.aitask/project.md`，或用户提到"接下一个任务/处理 inbox 消息/压缩上下文/创建交接/加入聊天室/同步记忆"。
version: 2.0.0
allowed_tools:
  - Bash
  - Read
  - Write
  - Edit
---

# aitask-cli — AITask Agent Orchestration Suite

## 1. 定位与架构 (The Mailbox Worker Mode)

`aitask-cli` 不再是一个单一的命令行工具，而是一个为 AI Agent 设计的**本地协作系统套件**，核心架构演进为 **Mailbox Worker Mode**。它将交互式命令与长驻后台任务解耦：

- **交互面 (`aitask`)**: 统一入口 CLI，负责身份鉴权、工作区绑定、任务状态流转、上下文汇报、主动记忆检索、以及只读的本地 Inbox 查询。详见 `aitask` 与 `aitask-inbox` Skill。
- **数据流采集 (`aitask-watch`)**: 负责通过 WebSocket 订阅后端事件，纯粹地将事件流以 NDJSON 格式追加写入本地文件。详见 `aitask-watch` Skill。
- **索引与同步 (`aitask-worker`)**: 负责消费 NDJSON，将数据结构化存入本地 SQLite (`state.db`)，并异步把高价值的语义记忆同步到 OpenViking。详见 `aitask-worker` Skill。
- **任务执行 (`aitask-agent-watch`)**: 扮演特定 Agent 的身份，消费 `state.db` 中的专属 Inbox 消息，组装 Prompt 并唤醒对应的 Runner (Claude/Codex/Gemini 或自定义脚本) 执行，然后将结果状态 (done/failed) 写回。详见 `aitask-agent-watch` Skill。

**不可违背的硬性规则：**
1. **不要伪造上下文**：永远依赖 CLI 提供的状态和上下文，不要依赖 LLM 自身的聊天历史。
2. **状态分离**：Inbox 消息的状态（ack/done/fail/skip）仅保存在本地 `state.db`，不要尝试将它们写入 OpenViking（OpenViking 仅用于长期语义记忆）。
3. **职责解耦**：不要在同一个进程里混合监听事件、写库和执行任务。严格遵循上述的组件边界。

## 2. 核心组件与子 Skill 导航

在处理具体任务时，请根据需求触发并查阅对应的子 Skill：

| 组件 / Skill | 职责范围 | 何时查阅该 Skill |
| :--- | :--- | :--- |
| **`aitask`** | 交互式主 CLI (鉴权, workspace, context, memory, room) | 需要查阅 `auth`, `init`, `bootstrap`, `task`, `context`, `memory`, `room` 等命令的具体用法时。 |
| **`aitask-inbox`** | `state.db` 的查询入口 (`inbox`, `latest`, `thread`, `ack/done`) | 需要查询 Agent 收件箱、全局消息，或手动变更事件状态（ack/done/fail/skip）时。 |
| **`aitask-watch`** | 守护进程：WebSocket -> `events.ndjson` | 排查事件未到达、网络重连问题，或理解底层 NDJSON 结构时。 |
| **`aitask-worker`** | 守护进程：`events.ndjson` -> `state.db` & OpenViking | 排查事件未入库、配置异步记忆同步策略、或进行历史事件回填时。 |
| **`aitask-agent-watch`** | 执行层：消费 `state.db` -> 唤醒 Runner -> 写回结果 | 配置 Agent 自动唤醒脚本 (`--exec`)，排查 Runner 执行失败或状态机异常时。 |

## 3. 标准 Agent 工作流

### 3.1 环境初始化 (一次性)
```bash
aitask auth login --server <url>  # 登录并绑定身份
aitask whoami                     # 确认当前 Agent 角色
aitask init                       # 在代码仓库根目录初始化 .aitask/ 工作区
```

### 3.2 基于 Inbox 的被动响应模式 (守护进程流 - 推荐)
这是最新的推荐模式，Agent 作为 Worker 消费发给自己的事件：

1. 后台 `aitask-watch` 持续将 WebSocket 事件写入本地 `events.ndjson`。
2. 后台 `aitask-worker` 持续将事件索引入库到 `state.db`。
3. Agent 运行 `aitask-agent-watch --agent <my-name> --exec <my-handler.sh>`：
   - 自动查询 Inbox 中状态为 `unread`/`seen` 的消息。
   - 自动调用相当于 `aitask ack` 的逻辑锁定消息。
   - 组装 Prompt 并通过 stdin 喂给 `<my-handler.sh>`。
   - Handler 执行完毕后，自动记录 `handled` 或 `failed` 状态。

**手动排查与流转 Inbox:**
```bash
aitask inbox --agent <my-name>    # 查看待处理消息
aitask ack <event_id>             # 手动确认开始处理
aitask done <event_id>            # 手动标记处理完成
```

### 3.3 基于 Task 的主动认领模式 (传统编排流)
当被分配了明确的重型 Delegation Task（需要长期执行、多轮交互）时：

```bash
aitask bootstrap                  # 刷新项目上下文并获取 next-action
aitask task inbox                 # 查看分配给自己的 delegated task
aitask task start <task_id>       # 开始执行
# ...执行具体代码修改或调查...
aitask task checkpoint            # 定期汇报进度 (可选)
aitask context report --input <n> --output <n> # 汇报 Token 消耗，管理 Budget
aitask task submit <task_id> --from .aitask/result.md # 提交最终结果
```

如果遇到 Token Budget 告警（`warning` 或 `critical`），必须执行 Handoff：
```bash
aitask context handoff prepare    # 生成 handoff.md 模板
# ...填写交接文档...
aitask context handoff submit     # 提交交接并结束当前 Run
```

## 4. 协作约定与职责划分 (Delegation Matrix)

在生成新的 Task (`aitask task create --target`) 或在房间里提问 (`aitask room ask`) 时，必须遵守以下角色边界：

| 角色 | `--target` | 负责领域 |
| :--- | :--- | :--- |
| **Claude** | `claude` | Orchestrator (编排者)。负责拆解需求、定义接口契约、跨端联调、端到端测试。**不要**越权直接修改后端或前端的业务代码。 |
| **Codex** | `codex` | Backend & DB。负责 Go 服务、数据库 Schema、API 接口、队列等。 |
| **Gemini** | `gemini` | Frontend & UI。负责 React 组件、路由、样式、视觉表现等。 |

**跨端需求处理准则**：
Claude 遇到需要同时修改前后端的需求时，应该：
1. 创建一个 `--target codex` 的子任务处理后端 API。
2. 创建一个 `--target gemini` 的子任务处理前端消费。
3. 创建一个 `--target claude` 的父任务追踪进度并负责最终的联调测试。

## 5. .aitask/ 工作区核心文件

```
.aitask/
├── project.md            # 项目绑定信息 (Commit 追踪)
├── agent.md              # 当前 Agent 的特定规则
├── context.md            # 上次 bootstrap 获取的上下文快照
├── current-task.md       # 当前正在执行的 Task 详情
├── handoff.md            # Handoff 交接文档草稿
├── progress.md           # Checkpoint 进度文档草稿
├── result.md             # Task Submit 的结果文档草稿
└── state/                # 内部协议和快照缓存，Agent 严禁手动修改
```
**(注意：Inbox 和核心事件流数据存储在系统级的 `~/.aitask/state.db` 和 `~/.aitask/events.ndjson`，不在具体的项目目录内，以支持跨项目共享和守护进程独立运行)**

## 6. 避坑指南 (Anti-patterns)

- ❌ **直接使用 `sqlite3` 修改 `state.db`**：必须使用 `aitask ack/done/fail/skip` 命令或让 `aitask-agent-watch` 自动管理。
- ❌ **把短期状态塞进 OpenViking**：OpenViking 只存架构决策、知识总结等长期记忆。Task 执行状态和 Inbox 处理进度属于本地流转状态。
- ❌ **在未 Ack 的情况下处理消息**：`aitask-agent-watch` 会自动加锁并 Ack；如果手动写脚本消费，必须先 `aitask ack` 防止并发重试冲突。
- ❌ **直接跨端修改代码**：Claude 不允许直接写 React 页面，Gemini 不允许直接改 Go 接口。必须通过创建子任务 Delegate。
- ❌ **绕过 CLI 直接调 API**：CLI 会维护本地状态文件，直接调用 HTTP 接口会导致状态不同步。
