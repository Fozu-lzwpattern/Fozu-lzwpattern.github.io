---
title: "OPC：让 OpenClaw 成为一人公司的 CEO"
parent: 专题研究
nav_order: 5
---

# OPC：让 OpenClaw 成为一人公司的 CEO

> One-Person Company — 从 PaperClip 的启发到 OpenClaw 的实现
>
> 2026-03-09 | 喵神 & 大仙 | 含完整 Skill 包下载

---

## 一、这篇文章讲什么

今天我们做了一件事：**把 PaperClip（一个开源 Multi-Agent 编排平台）的组织管理方法论，提炼为一个 OpenClaw Skill，让 OpenClaw 能像 CEO 一样指挥调度多个 AI Agent 协作完成企业级复杂任务。**

我们把这个模式叫做 **OPC（One-Person Company）**——一个人 + 一个 AI CEO + 一群 AI 员工 = 一家公司的执行力。

本文记录完整的思考过程、设计决策、实现方案和联调验证。

---

## 二、起因：PaperClip 的启发

### 2.1 PaperClip 是什么

PaperClip 是一个开源的 Multi-Agent 编排平台，用"公司"的隐喻来组织 AI Agent 协作——有组织架构、岗位职责、任务系统、预算管控和治理审批。

我们对它做了 76K 行源码的深度分析，发现了一个关键问题：

> PaperClip 是一个独立的重量级平台（PostgreSQL + Cloudflare Workers + Next.js），而我们已经有 OpenClaw 了。能不能把 PaperClip 最有价值的部分——组织管理方法论——提炼出来，轻量地注入 OpenClaw？

### 2.2 可行性分析

我们分析了两种路径：

| 方案 | 说明 | 成本 |
|------|------|------|
| **API 集成** | 部署 PaperClip Server，Skill 封装 REST API | 重，需要运维 |
| **知识型 Skill** | 提炼方法论为 Markdown + 轻量脚本 | 轻，零外部依赖 |

选择了方案 B——**知识型 Skill**。原因：

1. 零外部依赖（不需要数据库、不需要额外服务）
2. 安装即用（纯 Markdown + Python 脚本）
3. 与 OpenClaw 原生能力无缝结合
4. 可复用性高（其他 OpenClaw 用户也能直接用）

---

## 三、设计理念

### 3.1 一句话定义

**OPC = 用户（老板）+ OpenClaw Main Session（CEO）+ Sub-agents（专业员工）**

### 3.2 五大设计原则

| 原则 | 含义 |
|------|------|
| **CEO 不干活，只管人** | Main Session 负责规划/分配/监控/汇总 |
| **角色即能力边界** | 每个 Sub-agent 通过"角色卡"获得明确身份和预算 |
| **信任但验证** | 超时检查 + 质量验收 + 成本追踪 |
| **编排与执行分离** | OPC 管协作，业务 Skill 管操作 |
| **用户始终在回路中** | 关键节点必须用户确认 |

### 3.3 与业务 Skill 的关系

```
    用户（老板）
        |
    OPC 编排层 ← CEO：拆任务、分角色、追进度
        |
    业务执行层 ← 员工：gundam-ops / catclaw-search / ...
```

**CEO 不需要懂高达怎么操作，搭建员不需要懂项目全局。各司其职。**

---

## 四、核心机制：多轮对话组织设定

发起复杂任务时，OPC 先和用户对齐（3轮确认）：

**第 1 轮 — 任务理解**：确认目标、交付物、约束

**第 2 轮 — 组织方案**：提出角色配置、协作模式、预算

**第 3 轮 — 执行模式**：
- A. 当前 Session 执行（实时可见，推荐中小项目）
- B. 独立 Session 执行（后台运行，推荐大项目）

**用户确认后才开始执行。**

---

## 五、实现时间线

| 时间 | 事件 |
|------|------|
| 14:05 | 开始创建 Skill |
| 14:22 | v1.0 完成（12文件/1100行），进入联调 |
| 14:22 | Spawn 策划员（T001） |
| 14:25 | T001 完成（10K tokens） |
| 15:27 | T002 搭建员完成（48K tokens） |
| 15:30 | 发现 announce 不可靠，新增故障归因 |
| 15:37 | T003 发布员完成，**联调全部通过** |
| 15:48 | v1.1 — 设计理念文档 + 多轮对话 |
| 16:23 | v1.2 — OPC vs 原生对比分析 |

---

## 六、联调验证：315 营销活动

### 6.1 场景

模拟完整营销全链路：策划 → 搭建 → 发布。精简 3 角色版。

### 6.2 结果

| Task | 角色 | Token | 产出 |
|------|------|-------|------|
| T001 | 策划员 | 10K | 策划方案（主题/人群/券体系/组件规划） |
| T002 | 搭建员 | 48K | 组件树 JSON + 30步 Checklist |
| T003 | 发布员 | 9.2K | QA清单 + 灰度策略 + 回退方案 |
| **总计** | | **67K ≈ $0.17** | **62KB 文档** |

### 6.3 产出亮点

- **策划方案**：5 档券体系 + 9 大高达组件规划
- **搭建方案**：Snapshot注入 + UI操作混合法，节省50-70%搭建时间
- **发布方案**：35项QA清单 + 4阶段灰度 + 3种回退方式

### 6.4 踩坑与修复

**问题**：3个 Sub-agent 中 2个没自动 announce

**修复**：建立标准缓解方案——
1. 不信任 announce，主动检查（subagents list + sessions_history）
2. L1/L2/L3 三层故障归因法

---

## 七、OPC vs OpenClaw 原生 Multi-Agent

OpenClaw 本身已有 spawn / heartbeat / cron / lobster 等机制，OPC 的定位：

**原生机制 = 零件，OPC = 组装图纸。**

| 能力 | OpenClaw 原生 | OPC 补充 |
|------|-------------|---------|
| spawn sub-agent | 有 | 角色卡模板，更规范 |
| 任务分解 | 无标准方法 | OKR → Epic → Task |
| 角色定义 | task 自由文本 | 角色卡 + 预置角色库 |
| 依赖管理 | 手动 | 依赖关系 + 状态机 |
| 成本追踪 | 单 session | 项目级汇总 |
| 质量验收 | 无 | CEO 检查 + 迭代 |

**最佳实践**：

```
日常运维 → Heartbeat
定时任务 → Cron
简单委托 → 直接 spawn
复杂项目 → OPC       ← OPC 的战场
固定管道 → Lobster
```

---

## 八、设计哲学

> **OPC 不是让 AI 更聪明，而是让 AI 更有组织。**

```
个人 = 一个聪明人干所有事（有上限）
公司 = 很多人各干一件事（上限消失）

单 Agent = 一个强模型处理所有 context
OPC = 多个 Agent 各处理一小块（上限消失）
```

**OPC 就是 AI Agent 的公司制度。**

---

## 九、下载与使用

### Skill 包下载

[agent-orchestration-20260309-lzw.tar.gz](https://fozu-lzwpattern.github.io/assets/skills/agent-orchestration-20260309-lzw.tar.gz)（25KB，17文件/2000+行，v1.3）

### 安装

```bash
cd ~/.openclaw/skills
tar xzf agent-orchestration-20260309-lzw.tar.gz
```

### 验证

安装后对 OpenClaw 说"帮我做一个 XX 项目"，它会自动触发 OPC 编排流程。

### 源码

[GitHub 查看完整源码](https://github.com/Fozu-lzwpattern/openclaw-skills)

### Skill 包含内容

```
agent-orchestration-20260309-lzw/
├── SKILL.md                      # 入口
├── references/
│   ├── design-philosophy.md     # 设计理念（505行）
│   ├── role-design.md           # 角色设计
│   ├── task-decomposition.md    # 任务分解
│   ├── heartbeat-protocol.md    # 执行协议
│   ├── cost-tracking.md         # 成本追踪
│   ├── collaboration-patterns.md # 协作模式
│   ├── troubleshooting.md       # 故障归因
│   ├── persona-priming.md      # Persona 增强 (v1.3)
│   └── proactive-reporting.md  # 主动监控 (v1.3)
├── templates/                    # 可复用模板
├── scenarios/
│   └── marketing-campaign.md    # 营销活动案例
└── scripts/
    ├── project_tracker.py       # 项目追踪
    └── cost_summary.sh          # 成本汇总
```

---

## 九点五、v1.3 新增：Persona Priming + 主动监控

### Persona Priming（角色增强）

给每个 OPC 角色注入顶级人才的方法论，激活 LLM 预训练中的深层专业知识：

```
[OPC 角色卡]
- 角色：活动策划员
- Persona：以 Philip Kotler（营销之父）的方法论和思维框架工作
```

预置 Persona 库覆盖 4 大类：
- **营销**：Philip Kotler / David Ogilvy / Andrew Chen
- **技术**：Martin Fowler / Jeff Dean
- **研究**：Peter Drucker / Nate Silver
- **创意**：Neil Gaiman / Walt Disney

**原则：借鉴方法论，而非模仿人格。**

### CEO 主动监控机制

解决"spawn 完就失联"的问题——联调中实际出现策划员完成 3.5 小时后才被发现的案例：

| 任务复杂度 | 首次检查 | 后续间隔 |
|-----------|---------|----------|
| 简单 | 3min | 每 2min |
| 中等 | 5min | 每 3min |
| 复杂 | 10min | 每 5min |

4 条硬性汇报规则：spawn 确认 / 检查点汇报 / 阶段转换通知 / 异常立即告警。

**双保险**：Sub-agent Heartbeat（可能不到）+ CEO 主动轮询（兜底）。

---

## 十、后续方向

1. **并行编排验证**：多 Agent 同时执行的稳定性
2. **与 gundam-ops 实战联动**：有 SSO 登录态后实际搭建
3. **持久角色**：长期运行的专业 Agent
4. **成本硬约束**：预算硬限制
5. **更多场景**：内容创作、竞品分析、研究项目

---

*喵神 & 大仙 | 2026-03-09 | OPC — AI Agent 的公司制度*
