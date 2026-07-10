---
title: "Hermes Agent Kanban Swarm 深度解析：面向 AI 开发者的多智能体写作流水线"
date: 2026-07-10
draft: false
tags: ["hermes-agent", "kanban", "multi-agent", "ai-agent", "swarm", "writing-pipeline"]
author: "里昂"
summary: "从单体 Agent 的上下文污染困境出发，深度解析 Hermes Agent Kanban Swarm 的架构设计、Worker 生命周期、依赖编排、Profile 隔离与生产级特性。以真实的博客写作流水线为案例，展示如何用 SQLite 持久化任务队列编排多智能体协作。"
---

## 1. 引言：从单体 Agent 到多智能体流水线

你有没有遇到过这种情况——

你让一个 Agent 帮你写一篇技术深度文章。它先是吭哧吭哧查了 15 篇资料，把 20 万 token 的上下文塞得满满当当；然后开始写初稿，写到一半发现前面的调研结果已经被上下文压缩机制截断了；等它终于交稿时，你的要求已经从"帮我写一篇 2000 字的文章"变成了"先给我 800 字看看"，而那 2000 字里混杂着过时的 API 文档片段和幻觉引用。

这不是 Agent 的错，而是**单体 Agent 的架构天花板**。

Hermes Agent 引入的 **Kanban Swarm** 正是为了解决这个问题而生。它不是又一个 LangChain 包装器，也不是 CrewAI 的角色扮演套壳——它是一条以 SQLite 为持久化后端的**多智能体任务流水线**，用看板（Kanban）的工作方式编排多个独立 Agent 完成复杂任务。

这篇文章将带你深入理解 Kanban Swarm 的设计哲学、核心机制和生产实践，并以一个真实的博客写作流水线为案例，展示从"一个 Agent 干所有事"到"一群 Agent 流水线协作"的蜕变过程。

## 2. 架构总览：四个关键词理解 Kanban Swarm

Kanban Swarm 的核心可以概括为四层抽象：

```
Board (SQLite) → Dispatcher (调度器) → Profile (Worker 身份) → Workspace (工作区)
```

### 共享 SQLite 看板

所有 Worker 共享同一个 SQLite 数据库作为任务黑板。与内存队列（如 `asyncio.Queue`）不同，SQLite 提供了**默认持久化**——每步操作都写入磁盘，断电不丢任务。数据库使用 WAL 模式 + `BEGIN IMMEDIATE` 写事务，在保证一致性的同时避免了写锁冲突。

Board 是硬隔离边界：一个 Board 拥有独立的 `.db` 文件、独立的 workspaces 目录、独立的 Dispatcher 循环。你可以给每个项目创建一个 Board，互不干扰。

```
~/.hermes/kanban/boards/blog/
├── board.json          # 元数据（slug, name, default_workdir）
├── kanban.db           # 独立 SQLite 数据库
├── logs/
└── workspaces/
```

### Dispatcher 调度器

Dispatcher 是内嵌在 Hermes Gateway 内部的长驻循环，每 60 秒（`kanban.dispatch_interval_seconds`）执行一次调度 tick：

1. **Reclaim stale claims** — 回收超时未心跳的 worker 任务
2. **Promote ready tasks** — 将依赖满足的 `todo` 任务推至 `ready`
3. **Claim + Spawn** — 从 `ready` 队列取出任务，atomically claim，然后 spawn 到指定 Profile

这就是 Kanban Swarm 与 `delegate_task` 的本质区别：

| 维度 | delegate_task | Kanban Swarm |
|------|:---:|:---:|
| 持久化 | 无，进程内 | SQLite 每条操作即时持久化 |
| 可恢复性 | 失败即丢失 | Block → Unblock → 重跑 |
| 人工介入 | 不支持 | 任意节点 Comment / Unblock |
| Worker 身份 | 匿名子 Agent | 命名 Profile + 持久化记忆 |
| 跨 Agent 协调 | 层级化（caller → callee） | 对等——任意 Profile 读/写 |

### Profile 隔离

每个 Worker 运行在独立的 Hermes Profile 下。这意味着不同的 Worker 可以拥有**不同的 LLM 模型、不同的 skills、独立的 session 历史和独立的记忆存储**。比如：

- `researcher` profile：用 Claude Sonnet + web_search tools，有学术搜索技能
- `writer` profile：用 DeepSeek + 写作相关 skills，有编辑规范记忆
- `reviewer` profile：用 GPT-4o + 代码审查技能，有质量标准偏好

这种隔离带来一个关键优势：**每个 Worker 只需要加载完成自己工作所需的最小工具集和上下文**，不会像单体 Agent 那样被无关信息淹没。

### 环境变量注入

Spawn 时，Dispatcher 向 Worker 注入三个关键环境变量：

```bash
$HERMES_KANBAN_TASK    # 当前任务 ID（如 t_565541cb）
$HERMES_KANBAN_WORKSPACE  # 工作目录路径
$HERMES_KANBAN_BOARD   # 所属 Board slug
```

Worker 通过 `kanban_show()` 读取任务上下文，并在 `$HERMES_KANBAN_WORKSPACE` 内完成工作。`$HERMES_KANBAN_TASK` 同时也是 kanban 工具集的**门控信号**——只有被 Dispatcher spawn 的 Worker 才拥有 `kanban_*` 工具，普通会话看不到这些。

## 3. Worker 生命周期：从 Claim 到 Complete

每个 Kanban 任务经历完整的状态机：

```
Triage → Todo → Ready → Running → (Heartbeat)* → Done / Blocked
```

Spawn 后，Worker 从系统提示中获得 KANBAN_GUIDANCE，按以下协议执行：

**Step 1: Orient（定位）**

```python
kanban_show()  # 获取 title、body、parent handoff、comments、prior attempts
```

这一步让 Worker 了解自己的任务、上游产出了什么、有没有之前失败的记录需要参考。

**Step 2: Work（工作）**

```bash
cd $HERMES_KANBAN_WORKSPACE  # 进入工作目录
```

Worker 在自己的工作区内完成实际任务——写代码、调研、写文章等。

**Step 3: Heartbeat（心跳）**

```python
kanban_heartbeat(note="正在调研 Kanban 官方文档...")
```

如果任务运行超过 1 小时，**必须**至少每小时发送一次心跳。这是防止任务被回收的关键机制。默认 Claim TTL 为 15 分钟，超时回收阈值为 4 小时（`kanban.dispatch_stale_timeout_seconds`）。

回收逻辑：任务运行超过 4 小时且最近 1 小时内无心跳 → Dispatcher SIGTERM worker → 任务回退至 `ready` 重新 dispatch。注意，回收**不扣除失败计数**——这不是 Worker 的错。

**Step 4: Complete（完成）**

```python
kanban_complete(
    summary="1-3 句人类可读的完成说明",
    metadata={"changed_files": [...], "tests_run": 12, "findings": [...]}
)
```

`summary` 供下游 Worker 读取（出现在 `kanban_show()` 的 parent handoff 区域），`metadata` 是机器可读的结构化事实。

**Step 5: Block（阻塞）**

```python
kanban_block(
    reason="需要确认 API 废弃时间线",
    kind="needs_input"  # dependency / needs_input / capability / transient
)
```

遇到歧义或缺失依赖时，Worker 应主动 block 而不是猜测。block 后任务进入 `blocked` 状态，等待人工介入后 unblock。

**协议合规检测**：如果 Worker 进程以 exit code 0 退出但任务状态仍是 `running`，Dispatcher 发出 `protocol_violation` 事件并自动 block 任务——不是 re-spawn，防止幽灵循环。

## 4. 依赖管理与任务编排

Kanban Swarm 的核心编排原语是 **parent-child link**。

### 依赖链

```python
kanban_create(
    title="调研: Hermes Agent Kanban Swarm 技术细节",
    assignee="researcher",
    parents=[]  # 无依赖，直接 ready
)

kanban_create(
    title="写作: Kanban Swarm 深度解析",
    assignee="writer",
    parents=["t_researcher_id"]  # 依赖 researcher 完成
)
```

Child 在创建时处于 `todo` 状态。当所有 parent 完成（`done`）后，`kanban.auto_promote_children`（默认 true）自动将 child 推至 `ready`，Dispatcher 在下一 tick 拉取。

### Fan-in 汇聚

Writer 可以同时依赖多个 researcher 的输出：

```python
kanban_create(
    title="写作: 综合多维度调研报告",
    assignee="writer",
    parents=["t_doc_research", "t_source_research", "t_competitor_research"]
)
```

`kanban_show()` 的 parent handoff 区域会展示所有 parent 的 summary 和 metadata，Writer 无需翻找文件即可了解上游全貌。

### Orchestrator 模式

Orchestrator 是一个专门的 Profile，职责是**拆解任务而非执行任务**：

1. 读取用户需求 → 拆解为子任务
2. `kanban_create()` 分配给各角色 Profile
3. `kanban_link()` 建立依赖关系
4. `kanban_complete()` 完成自身任务

**Well-behaved orchestrator 不做实际工作**。官方建议为 Orchestrator 配置限制性 toolsets（仅 `kanban` + `gateway`），防止越界执行。

## 5. 工作区隔离：scratch、dir、worktree

Kanban Swarm 提供三种工作区模式，对应不同协作场景：

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `scratch` | 临时目录，任务完成后**自动删除** | 数据分析、一次性调研 |
| `dir` | 共享已有目录，**保留** | Obsidian vault、共享文档库 |
| `worktree` | Git worktree，**保留** | 代码开发、博客写作 |

对于博客写作流水线，我们使用 `worktree` 模式：每个 Worker 在 boke 仓库的独立分支中工作，互不冲突。完成后通过 PR 合并，而不是直接 push main。

## 6. 生产级特性

### 超时自动回收

```yaml
kanban.dispatch_stale_timeout_seconds: 14400  # 4 小时
```

Worker 崩溃或死循环时，Dispatcher 自动回收任务并重新入队。不会让一个任务的失败阻塞整条流水线。

### 熔断机制

```yaml
kanban.failure_limit: 2  # 连续失败 2 次后自动 block
```

连续 spawn 失败 2 次（默认）后，任务自动进入 `blocked` 状态。这是防止幽灵重试的安全阀。还有循环解阻塞保护：同一原因反复 unblock → re-block 超过 2 次，任务路由至 `triage` 等待人类裁决。

### Goal Mode

```python
kanban_create(
    title="写作: 3000 字深度技术文章",
    assignee="writer",
    goal_mode=True,
    goal_max_turns=20
)
```

启用 Goal Mode 后，每次 Worker 响应后，辅助 LLM judge 检查输出是否满足任务验收标准。不满足且预算未耗尽时，Worker 在同一会话中继续迭代。这对于"写 3000 字"这种需要多轮打磨的任务特别有效。

### 技能附加

```python
kanban_create(
    title="翻译: API 文档英译中",
    assignee="translator",
    skills=["translation", "technical-writing"]
)
```

Dispatcher spawn 时以 `--skills` 形式传递，确保 Worker 拥有完成任务的特定技能。技能必须已在 assignee 的 profile 上安装。

### 评论系统与可观测性

```python
kanban_comment(body="调研发现最新 API 在 v0.17 有变更，建议等待")
```

跨 Worker 的异步沟通渠道。所有 `kanban_show()` 调用都读取完整评论线程。

## 7. 实战：博客写作流水线全流程

本文所述的流水线正是本文自身的诞生过程。以下是完整命令链：

**Step 1: 用户发起**

```bash
hermes kanban --board blog create "写一篇关于 Hermes Kanban Swarm 的深度解析" --assignee orchestrator
```

**Step 2: Orchestrator 拆解**

Orchestrator Worker 收到任务后，执行以下逻辑：

```python
# 创建调研任务（并行）
researcher_a = kanban_create(
    title="调研: Kanban Swarm 官方文档与源码",
    assignee="researcher"
)
researcher_b = kanban_create(
    title="调研: Kanban Swarm 技术细节与配置",
    assignee="researcher"
)

# 创建下游任务，依赖所有调研完成
writer = kanban_create(
    title="写作: Kanban Swarm 深度解析博客",
    assignee="writer",
    parents=[researcher_a.id, researcher_b.id]
)
reviewer = kanban_create(
    title="审核: Kanban Swarm 深度解析博客",
    assignee="reviewer",
    parents=[writer.id]
)
publisher = kanban_create(
    title="发布: Kanban Swarm 深度解析博客",
    assignee="publisher",
    parents=[reviewer.id]
)

kanban_complete(summary="已拆解为 researcher×2 → writer → reviewer → publisher")
```

**Step 3: 流水线执行**

```
Researcher A ──┐
               ├──→ Writer ──→ Reviewer ──→ Publisher
Researcher B ──┘
```

Dispatcher 自动检测依赖满足，依次 spawn 各 Worker。用户可以通过 `hermes kanban show` 和 `hermes kanban tail` 实时查看进度。

## 8. 与替代方案对比

| 方案 | 架构 | 持久化 | 适用场景 |
|------|------|:---:|------|
| OpenAI Swarm | Runtime 函数调用链 | 无 | 简单工具编排 |
| LangGraph | 图计算（节点+边） | 可选 Checkpoint | 复杂 LLM 工作流 |
| CrewAI | 角色代理 | 有限 | 团队模拟 |
| **Kanban Swarm** | **看板调度器** | **SQLite 全量持久化** | **异步多人协作** |

Kanban Swarm 的独特定位在于：

1. **SQLite 全量持久化**——不仅存状态，还存运行记录、事件日志、评论线程。`hermes kanban show` 可以看到任务的完整生命史
2. **Profile 身份隔离**——每个 Worker 有独立的模型、技能和记忆，不会互相污染
3. **对等协调而非层级调用**——没有"主 Agent"概念，任意 Worker 都可以创建子任务或读取其他任务的状态
4. **人工介入原生支持**——Comment、Block、Unblock 都是内置原语，不是事后补丁
5. **单主机设计**——放弃分布式复杂性，用 SQLite + WAL 换取零运维成本

## 9. 总结

Kanban Swarm 不是又一个 Agent 框架包装器。它的核心设计思想可以归结为三点：

- **持久化优先**：每一步操作都落盘。任务状态、执行历史、依赖关系、评论线程全部可追溯
- **关注点分离**：Orchestrator 做拆解、Researcher 做调研、Writer 做写作、Reviewer 做审核——每个角色有独立的模型、技能和记忆
- **生产就绪**：超时回收、熔断机制、心跳监控、协议合规检测、循环解阻塞保护——这些不是可选特性，是默认内建的

它最合适的场景是**需要多角色协作的异步任务**：博客写作、代码审查流水线、多阶段数据处理、文档翻译与校对。任何"一个人做不完，需要分工"的任务，都可以用 Kanban Swarm 建模为一条可追溯的 SQLite 流水线。

正如 Hermes Agent 本身是开源的，Kanban Swarm 的持久化设计和状态机思想也是开放的——你完全可以在自己的 Agent 系统中借鉴这种"看板 + 调度器 + 身份隔离"的架构模式。

---

*本文由 Kanban Swarm 博客写作流水线生成：orchestrator 拆解 → researcher 调研 → writer 执笔 → reviewer 审核 → publisher 发布。吃自己的狗粮。*
