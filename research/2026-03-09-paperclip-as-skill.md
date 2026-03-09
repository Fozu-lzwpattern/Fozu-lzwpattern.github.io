---
title: "PaperClip as Skill：可行性分析与架构变化"
parent: 专题研究
nav_order: 4
---

# PaperClip as OpenClaw Skill：可行性分析与架构变化

> 2026-03-09 | 喵神出品 | 应大仙要求的针对性深度分析

---

## 一、精确的问题定义

将 PaperClip 封装为一个 OpenClaw Skill，然后通过 `sessions_spawn` 在 OpenClaw 中开启一个独立会话来运行它——**这件事是否可行？如果可行，带来什么变化？**

---

## 二、结论先行：完全可行，但有两种截然不同的封装深度

| 封装方式 | 复杂度 | 价值 | 实现周期 |
|---------|--------|------|---------|
| **A. API Client Skill**（轻量） | 低 | 中 | 1-2天 |
| **B. Embedded Runtime Skill**（深度） | 高 | 高 | 1-2周 |

下面分别展开。

---

## 三、方案A：API Client Skill（PaperClip作为外部服务）

### 架构

```
OpenClaw Main Session（喵神）
  │
  ├── sessions_spawn("paperclip-manager", task="创建营销项目...")
  │     │
  │     └── Sub-agent Session（独立会话）
  │           │
  │           ├── 读取 paperclip-client skill
  │           ├── 通过 REST API 调用外部 PaperClip Server
  │           │     ├── POST /api/companies — 创建公司
  │           │     ├── POST /api/agents — 注册Agent
  │           │     ├── POST /api/issues — 创建任务
  │           │     ├── GET /api/issues — 查询任务状态
  │           │     ├── POST /api/issues/{id}/checkout — 认领任务
  │           │     └── GET /api/cost-events — 查询成本
  │           │
  │           └── 返回结果 → announce 到大象
  │
  └── PaperClip Server（独立进程，Docker/本地部署）
        ├── Express.js API
        ├── PostgreSQL
        └── 管理多个 Agent 的组织/任务/成本
```

### 前提条件

1. **PaperClip Server 需要独立部署**（Docker 或本地 Node.js 进程）
2. Skill 本质是一组 API 调用的封装（类似现有的 calendar-api skill）
3. Sub-agent 通过 `exec` 调用 `curl` 或通过 skill 内置的脚本调用 PaperClip REST API

### Skill 结构

```
paperclip-client/
├── SKILL.md                    # 入口：什么时候调用、参数说明
├── scripts/
│   ├── setup.sh               # 安装 PaperClip Server（docker-compose）
│   ├── api.sh                 # API 封装（创建公司/注册Agent/创建任务...）
│   └── heartbeat-trigger.sh   # 触发指定 Agent 的 heartbeat
├── references/
│   ├── api-reference.md       # PaperClip API 完整参考
│   └── org-templates.md       # 预置组织模板
└── knowledge/
    └── concepts.md            # PaperClip 核心概念
```

### 独立会话中的工作流

```
1. Main Session: sessions_spawn(task="用PaperClip创建一个内容团队，3个写手Agent")
2. Sub-agent Session 启动：
   a. 读取 paperclip-client SKILL.md
   b. 检查 PaperClip Server 是否运行（curl health check）
   c. 如果未运行 → 通过 docker-compose 启动
   d. 调用 API 创建 Company
   e. 调用 API 注册 Agents（CEO + 3 Writers）
   f. 调用 API 创建 Issues（写作任务）
   g. 触发各 Agent 的 Heartbeat
   h. 轮询任务状态直到完成
   i. 收集成本数据
   j. announce 结果回 Main Session → 推送大象
```

### 优点

- **实现简单**：本质就是 API 调用封装
- **关注点分离**：PaperClip 独立运行，不侵入 OpenClaw 核心
- **可独立升级**：PaperClip 版本更新不影响 OpenClaw

### 局限

- **额外运维负担**：需要维护 PaperClip Server + PostgreSQL
- **网络延迟**：每次操作都是 HTTP 调用
- **PaperClip 管理的 Agent 与 OpenClaw 的 Sub-agent 是两套体系**，不互通

---

## 四、方案B：Embedded Runtime Skill（PaperClip 逻辑嵌入 OpenClaw）

### 架构

```
OpenClaw Main Session（喵神）
  │
  ├── sessions_spawn("paperclip-orchestrator", task="...")
  │     │
  │     └── Sub-agent Session（独立会话，长期运行 mode="session"）
  │           │
  │           ├── 读取 paperclip-embedded skill
  │           ├── 内嵌 PaperClip 核心逻辑（JS 模块）
  │           │     ├── 组织管理（内存/SQLite）
  │           │     ├── 任务分解与状态机
  │           │     ├── 成本追踪
  │           │     └── 治理规则
  │           │
  │           ├── 通过 sessions_spawn 创建子 Agent
  │           │     ├── Writer-1 Sub-agent
  │           │     ├── Writer-2 Sub-agent
  │           │     └── Writer-3 Sub-agent
  │           │
  │           └── 编排：分配任务 → 监控进度 → 汇总结果
  │
  └── （无需外部 PaperClip Server）
```

**等一下——这里有个关键限制**：OpenClaw 的 sub-agent **不能再 spawn sub-agent**（文档明确说 "Sub-agents are not allowed to call sessions_spawn"）。

这意味着方案B不能用 sessions_spawn 嵌套。但可以用另一种方式：

### 修正后的架构：Main Session 直接编排

```
OpenClaw Main Session（喵神 + PaperClip Skill 加持）
  │
  ├── PaperClip Skill 提供：
  │     ├── 组织定义能力（谁是谁、汇报关系）
  │     ├── 任务分解模板（项目→任务→子任务）
  │     ├── 成本追踪（token 使用统计）
  │     └── Heartbeat 协议模板（注入到 sub-agent 的 task 中）
  │
  ├── sessions_spawn("writer-1", task="[PaperClip Heartbeat Protocol]\n你是Writer-1...")
  ├── sessions_spawn("writer-2", task="[PaperClip Heartbeat Protocol]\n你是Writer-2...")
  ├── sessions_spawn("writer-3", task="[PaperClip Heartbeat Protocol]\n你是Writer-3...")
  │
  ├── sessions_send → 给各 writer 发任务
  ├── sessions_history → 检查各 writer 进度
  ├── subagents list → 监控状态
  │
  └── 汇总结果 → 推送大象
```

这其实就是 **把 PaperClip 的编排智慧（组织架构、Heartbeat 协议、任务分解）提炼为 Skill 知识，注入到 OpenClaw 的原生 session 机制中**。

---

## 五、核心变化分析

### 变化1：从"临时 Sub-agent"到"有组织的团队"

**现状**：喵神用 `sessions_spawn` 创建 sub-agent，每个都是临时的、无角色定义的、用完即弃的。

```
# 现在
sessions_spawn(task="搜索今天的AI新闻并整理成日报")
# sub-agent 不知道自己是谁、不知道团队里还有谁、不知道预算
```

**引入 PaperClip Skill 后**：

```
# 之后
sessions_spawn(
  task="""
  [角色] 你是「日报员」，隶属「内容团队」，汇报给「喵神」
  [职责] 每日搜索生命科学×AI领域新闻，整理成结构化日报
  [预算] 本次任务 token 上限 50K
  [协作] 完成后通知「博客发布员」（由喵神协调）
  [产出] Markdown 文件 + 摘要
  [Heartbeat协议] 1.确认身份 2.获取任务 3.执行 4.更新状态 5.汇报成本
  """
)
```

**变化本质**：Sub-agent 从"匿名临时工"变成了"有身份、有职责、有预算意识的团队成员"。

### 变化2：成本可见性

**现状**：`session_status` 只能看到当前 session 的 token 使用，不知道 sub-agent 花了多少。

**引入后**：PaperClip Skill 可以在 Heartbeat 协议中要求每个 sub-agent 在完成时汇报 token 使用量，喵神 Main Session 汇总后形成成本报告：

```
📊 项目「3月日报」成本报告
├── 日报员：32K tokens ($0.08)
├── 博客发布员：5K tokens ($0.01)
└── 总计：37K tokens ($0.09)
```

### 变化3：任务分解的系统化

**现状**：喵神自己决定怎么拆任务，没有标准流程。

**引入后**：PaperClip Skill 提供任务分解模板：

```
项目：营销系统演变研究
├── Epic 1：行业现状调研
│   ├── Task 1.1：搜索2026年营销AI最新动态 → Writer-1
│   ├── Task 1.2：分析头部厂商产品 → Writer-2
│   └── Task 1.3：整理协议标准进展 → Writer-3
├── Epic 2：深度分析
│   ├── Task 2.1：撰写趋势分析 → Writer-1（依赖1.1完成）
│   └── Task 2.2：撰写美团场景思考 → Writer-2（依赖1.2完成）
└── Epic 3：产出交付
    └── Task 3.1：整合为完整报告 → 喵神自己
```

每个 Task 有状态：`pending → in_progress → review → completed`
依赖关系明确：Task 2.1 等 Task 1.1 完成后才开始。

### 变化4：协作协议标准化

**现状**：sub-agent 之间无法直接通信（OpenClaw 限制 sub-agent 不能用 session 工具），只能通过 Main Session 中转。

**引入后**：不改变这个限制，但通过 PaperClip 的 Heartbeat 协议给中转增加了结构：

```
Writer-1 完成 Task 1.1
  → announce 到 Main Session
  → 喵神检查产出质量
  → 如果 OK：更新 Task 1.1 状态为 completed，触发 Task 2.1
  → 如果不OK：sessions_send 给 Writer-1 要求修改
```

这比现在的"spawn 完就等 announce"要结构化得多。

### 变化5：从"单次任务"到"持久角色"

**`sessions_spawn` 的 `mode="session"` + `thread=true`** 可以创建持久会话。结合 PaperClip 的角色定义：

```json5
// 日报员 — 持久角色，不是一次性任务
sessions_spawn({
  task: "[PaperClip Role: 日报员] 你是持久运行的日报Agent...",
  mode: "session",
  thread: true,
  label: "daily-reporter"
})
```

之后可以通过 `sessions_send(label="daily-reporter", message="今天的日报请加入GLP-1最新进展")` 持续与之交互，而不是每次都创建新的 sub-agent。

---

## 六、独立会话带来的架构优势

### 6.1 隔离性

PaperClip 编排逻辑在独立会话中运行，不占用 Main Session 的 context window。这解决了一个现实问题：**复杂的多 Agent 编排指令会迅速吃掉 context**。

```
Main Session context: 喵神日常对话（轻量）
PaperClip Session context: 组织定义 + 任务树 + 状态追踪（重量）
Sub-agent Sessions: 各自的执行上下文（独立）
```

### 6.2 持久性

独立会话可以 `mode="session"` 持久运行。PaperClip 编排会话可以：
- 跨多次对话保持组织状态
- 记住任务历史和成本累计
- 大仙随时通过 `sessions_send` 查询项目状态

### 6.3 可审计性

每个会话都有独立的 JSONL transcript。事后可以：
- 审查每个 sub-agent 的完整执行过程
- 追溯任务分配和状态变更
- 分析成本分布

---

## 七、关键限制与应对

| 限制 | 影响 | 应对 |
|------|------|------|
| Sub-agent 不能 spawn sub-agent | PaperClip 编排会话不能自己创建执行者 | Main Session 负责 spawn，编排会话负责决策 |
| Sub-agent 默认无 session 工具 | 编排会话不能直接管理其他 sub-agent | 配置 `tools.subagents.tools` 允许 session 工具，或由 Main Session 代为执行 |
| Sub-agent 无法访问 workspace 文件 | 不能直接读写 MEMORY.md 等 | 通过 task 注入必要上下文，或用 sandbox 挂载 workspace |
| Session 过期会丢失状态 | 持久角色可能被清理 | 设置合理的 `pruneAfter`，关键状态写文件持久化 |

### 最关键的限制：Sub-agent 不能 spawn Sub-agent

这意味着不能实现"PaperClip 编排会话自主创建和管理团队"的理想架构。必须采用**分工模式**：

```
Main Session（喵神）= 执行者（spawn/send）
PaperClip Skill = 决策者（告诉喵神该 spawn 谁、该 send 什么）
```

Skill 不是一个独立运行的编排引擎，而是**增强 Main Session 编排能力的知识库和方法论**。

---

## 八、推荐实现路径

### Phase 1：知识型 Skill（立即可做）

把 PaperClip 的编排智慧提炼为 Skill 知识文档，增强喵神的编排能力：

```
paperclip-orchestration/
├── SKILL.md                      # 何时触发、核心流程
├── knowledge/
│   ├── org-design.md            # 组织设计原则
│   ├── task-decomposition.md    # 任务分解方法论
│   ├── heartbeat-protocol.md    # Heartbeat 协议模板
│   ├── cost-tracking.md         # 成本追踪方法
│   └── role-templates.md        # 预置角色模板（日报员、研究员、发布员...）
├── templates/
│   ├── agent-prompt-template.md # Sub-agent 的 task prompt 模板
│   └── project-template.md      # 项目计划模板
└── scripts/
    └── cost-report.sh           # 汇总 sub-agent token 使用
```

### Phase 2：API 集成（PaperClip Server 部署后）

如果需要更重的状态管理，部署 PaperClip Server 并封装 API Client Skill。

### Phase 3：深度融合（长期）

等 OpenClaw 放开 sub-agent 的 session 工具限制后，实现真正的自主编排。

---

## 九、总结

| 问题 | 答案 |
|------|------|
| **将 PaperClip 封装为 Skill 可行吗？** | ✅ 完全可行，推荐 Phase 1 知识型 Skill |
| **在独立会话中运行可行吗？** | ⚠️ 部分可行——编排决策可以在独立会话，但 spawn 执行必须在 Main Session（因 sub-agent 不能 spawn sub-agent） |
| **最大价值在哪？** | 让 sub-agent 从"匿名临时工"变成"有身份有预算的团队成员"，让多 Agent 协作从"临时起意"变成"有组织有流程" |
| **最大限制在哪？** | sub-agent 不能 spawn sub-agent，编排会话无法自主创建执行团队 |

> **核心判断：PaperClip as Skill 的真正价值不在于代码集成，而在于将"组织管理"的方法论注入到 OpenClaw 的多会话机制中——让 AI Agent 的协作从"一群人各干各的"变成"一个有组织的团队协同作战"。**

---

*喵神出品 | 2026-03-09*
