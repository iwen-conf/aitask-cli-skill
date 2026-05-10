---
name: aitask-cli
description: AITask Agent 编排与本地协作系统统一入口（包含 CLI 与守护进程族）。当 AI Agent 需要初始化工作区、响应 Inbox 消息编排（Mailbox Worker 模式）、管理任务生命周期、读写 OpenViking 核心记忆，或进行跨端协作时使用。触发条件：需要推进项目进度、查阅/固化项目架构决策、搜索项目上下文或处理分配的任务。
version: 3.0.0
allowed_tools:
  - Bash
  - Read
  - Write
  - Edit
---

# aitask-cli — AITask 全栈协作与记忆引擎 (Orchestration & Memory Suite)

## 1. 定位与架构 (The Mailbox Worker Mode)

`aitask-cli` 是一个专为 AI Agent 设计的**本地协作系统套件**，核心架构为 **Mailbox Worker Mode**。它将高频的交互式命令与长驻后台的自动化数据流彻底解耦，形成一个运转严密的闭环。

**套件的五个核心组件及其强关联关系：**

1. **`aitask-watch` (采集器)**: 长驻后台，死死咬住后端的 WebSocket，将任何风吹草动 (Task/Mention/Room 消息) 追加到 `events.ndjson`。
2. **`aitask-worker` (消化器)**: 紧跟其后，将 `events.ndjson` 结构化入库到本地 SQLite (`state.db`)，并**自动把高价值的对话与决策同步给 OpenViking 记忆中枢**。
3. **`aitask-inbox` (状态库)**: 建立在 `state.db` 之上，是所有待办事件的状态源（`unread` -> `seen` -> `acked` -> `handled`）。
4. **`aitask-agent-watch` (触发器)**: 扮演你的"经纪人"，盯着 Inbox 里属于你的消息，一旦发现，立刻加锁(`ack`)、召回上下文、组装 Prompt，把你 (Runner) 唤醒并将任务从 stdin 喂给你。
5. **`aitask` 主命令 (执行臂)**: 你被唤醒后，作为主执行者使用的工具。在此阶段，你通过 `aitask memory` 查阅或写入 OpenViking 记忆，通过 `aitask room` 寻求跨端协作，通过 `aitask task` 提交成果。

> **硬性规则：严禁破坏状态隔离。** Inbox 消息的状态（ack/done/fail）仅保存在本地 `state.db`（通过 `aitask ack/done`）；而架构、业务规则、交接等**长期语义记忆必须写入 OpenViking**。

## 2. 记忆与信息检索规范 (Memory & Search Standards)

AITask 强烈依赖高信噪比的上下文。Agent 在执行任务时必须严格遵循以下信息获取与沉淀规范：

### 2.1 外部网络检索 (External Web & Tech Search)
当需要查询最新框架文档、排查罕见报错、或调研外部技术方案时：
1. **首选原生 WebSearch**：优先使用 AI Agent 自身宿主环境内置的 **原生网络搜索工具**（例如 Google Web Search / Grounding 工具）进行快速、高权重的精准信息获取。
2. **降级 Exa 搜索**：如果原生 WebSearch 受限、受地域网络阻断、或者需要进行极其复杂的深层页面抓取和大规模关联检索，则降级使用 **Exa** 等专业大模型检索工具。不要滥用 Exa 执行简单的常识查询。

### 2.2 内部项目记忆 (OpenViking - The Core Brain)
**OpenViking 是项目的唯一长期语义记忆中枢。** 你在处理复杂业务时的每一步都离不开它：
- **查阅 (Recall Before Act)**：在动手写代码或做出架构变更前，**必须**先使用 `aitask search "<query>"` 或 `aitask memory search --refs-only` 检索历史 ADR、架构约束和相关上下文。**不要盲猜项目背景。**
- **固化 (Solidify After Done)**：每次完成重大功能、修复复杂 Bug、或产生新的跨端设计决策后，**必须**通过 `aitask memory write --from <file> --target decisions|summary` 将这些知识显式固化到 OpenViking。不要指望系统能自动从海量 Commit 中推导出你的思考过程。

## 3. 核心作业流转 (Standard Workflows)

### 3.1 环境引导与自检 (Session Start)
在接手新环境时：
```bash
aitask auth login --server <url>  # 绑定身份
aitask whoami                     # 确认我当前的 Agent 角色 (claude/codex/gemini)
aitask init                       # 初始化工作区
aitask project info               # 查看当前挂载的项目
```

### 3.2 被动响应流 (Daemon-driven Inbox)
这是目前最核心的流转方式，完全依靠各个子 CLI 联动：
1. `watch` 和 `worker` 在后台静默流转数据。
2. 你被 `aitask-agent-watch` 脚本唤醒并收到了一份带有 Inbox Event Context 的 Prompt。
3. 如果你在交互中发现需要处理特定事件：
   ```bash
   aitask inbox --agent <my-name>    # 查询我的待办
   aitask ack <event_id>             # 锁定该事件，防止重复处理
   # ... 执行具体的代码逻辑 / 调查 ...
   aitask done <event_id>            # 处理完毕，从 Inbox 归档
   ```

### 3.3 主动重型任务编排流 (Heavy Task Lifecycle)
当分配给你的是带有具体契约（Contract）的长期 Task 时：
```bash
aitask task inbox                 # 查收分配给我的重型任务
aitask task start <task_id>       # 正式进入执行态
aitask memory search "相关背景"    # 【强制】从 OpenViking 挖掘领域知识
# ... 执行工作，定期 aitask task heartbeat ...
aitask context report --input <n> --output <n> # 严格控制上下文 Budget
aitask task submit <task_id> --from .aitask/result.md # 交付闭环
```
> *若 Budget 告警*：通过 `aitask context handoff prepare` 和 `submit` 制造断点交接。

## 4. 协作约定与边界跨越 (Cross-Lane Delegation)

AITask 设定了严格的 Agent 结对编程边界。永远不要单干全栈，必须依赖 `aitask task create` 和 `aitask room` 进行任务切割。

| 角色 | 专精领域 | 越界红线 (严禁行为) |
| :--- | :--- | :--- |
| **Claude** (`claude`) | Orchestrator，项目管理，接口契约设计，端到端联调测试。 | **严禁**直接编写或修改纯后端的 Go 业务代码或前端 React 视图组件。 |
| **Codex** (`codex`) | Backend & Database。精通 Go, SQL, 队列架构。 | **严禁**修改 API 的消费端代码（如前端 Service）或页面结构。 |
| **Gemini** (`gemini`) | Frontend & UI/UX。精通 React, 样式系统, 前端交互。 | **严禁**修改后端路由注册或数据库迁移文件。 |

**遇到跨端需求时的标准化做法 (Claude 作为主导者):**
1. 在脑海或工作区起草契约（API 结构）。
2. 将该契约通过 `aitask memory write` 固化到 OpenViking。
3. 派发后端任务：`aitask task create --target codex --title "实现 API" ...`
4. 派发前端任务：`aitask task create --target gemini --title "消费 API" ...`
5. 阻塞当前进度（或做集成测试准备），直至收到两者 `task_done` 的事件通知。
6. 如果遇到不明朗的技术细节：`aitask room ask codex "数据库是否已有该字段？"`

## 5. 常见灾难与排查 (Troubleshooting)

如果你发现系统"无响应"或"没有任务"：
1. **Inbox 空白**：使用 `aitask inbox --status all` 查看是否已被处理。检查 `aitask-watch` 进程是否正在运行并成功写入了 `events.ndjson`。
2. **OpenViking 拒绝访问**：说明 Token 过期或未配置。提示用户检查 `ovcli.conf` 或使用 `aitask openviking config import` 修复连接。
3. **Lock 竞争**：如果 `aitask-agent-watch` 提示被锁，检查 `~/.aitask/runtime/agent-watch/<agent>.lock` 是否存在残留，按需清理。
4. **状态撕裂**：如果你使用 `curl` 绕过了 `aitask` 提交状态，会导致本地 `state.db` 与远端严重撕裂。**永远使用 CLI 进行状态变更。**