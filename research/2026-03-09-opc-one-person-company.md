---
title: "OPC：让 OpenClaw 成为一人公司的 CEO"
parent: 专题研究
nav_order: 5
---

# OPC：让 OpenClaw 成为一人公司的 CEO

> One-Person Company — 从 PaperClip 的启发到 OpenClaw 的实现
>
> 2026-03-09 创建 · 2026-03-14 更新至 v3.0 | 喵神 & 大仙

---

## 一、这篇文章讲什么

我们做了一件事：**把 PaperClip（一个开源 Multi-Agent 编排平台）的组织管理方法论，提炼为一个 OpenClaw Skill，让 OpenClaw 能像 CEO 一样指挥调度多个 AI Agent 协作完成企业级复杂任务。**

我们把这个模式叫做 **OPC（One-Person Company）**——一个人 + 一个 AI CEO + 一群 AI 员工 = 一家公司的执行力。

本文记录完整的思考过程、设计决策、实现方案、联调验证，以及在实战中踩坑后驱动的四轮迭代。

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

## 四、核心机制

### 4.1 多轮对话组织设定

发起复杂任务时，OPC 先和用户对齐。v1.4 引入了**确认流程分级**：

| 复杂度 | 判断标准 | 确认轮数 |
|--------|---------|---------|
| **简单** | ≤2个 Sub-agent，单一流水线 | 1轮（合并确认） |
| **中等** | 3-5个 Sub-agent，有并行 | 2轮 |
| **复杂** | >5个 Sub-agent，跨业务线 | 3轮（完整确认） |

3轮完整流程：

- **第 1 轮 — 任务理解**：确认目标、交付物、约束
- **第 2 轮 — 组织方案**：角色配置、协作模式、预算
- **第 3 轮 — 执行模式**：当前 Session（实时）vs 独立 Session（后台）

### 4.2 项目状态持久化（v1.4 新增）

**问题**：OpenClaw 的 context compaction 会丢失项目执行状态，CEO 不知道哪些 Agent 完成了、哪些失败了。

**方案**：通过 `project_state.py` 将全生命周期状态写入 `workspace/opc-projects/{id}/state.json`。

```bash
# 初始化项目
python3 scripts/project_state.py init "315促销活动"

# Agent 状态跟踪
python3 scripts/project_state.py agent-start <id> "builder-main" '{"role":"搭建员"}'
python3 scripts/project_state.py agent-complete <id> "builder-main" '{"activityId":"1014813"}' 45000

# Compaction 后一键恢复
python3 scripts/project_state.py restore <id>
```

### 4.3 断点续传（v1.4 新增）

**问题**：Agent 失败后完全重试浪费大量 token 和时间。

**方案**：失败/kill 前保存断点，新 Agent 从断点继续。

```
Agent 失败 → CEO 提取已完成步骤 → 保存 checkpoint → kill → spawn 新 Agent 注入断点
```

断点续传 prompt 模板：

```
[断点续传]
你是从上一个 agent 接替的。以下步骤已完成，不要重复执行：
已完成步骤：["create_activity", "add_header", "add_coupon"]
当前上下文：{"activityId": "1014818"}
请从以下步骤继续执行：configure_date_range
```

### 4.4 自动归因（v1.4 新增）

**问题**：Agent 失败后需人工判断是平台问题、编排问题还是业务 Skill 问题。

**方案**：`diagnose_agent.py` 规则引擎自动分类：

| 层级 | 来源 | 典型问题 | 自动归因关键词 |
|------|------|---------|-------------|
| L1 🔴 | 平台 | spawn 失败、announce 丢失 | spawn/session/timeout |
| L2 🟡 | 编排 | 角色定义不清、产出跑偏 | off_topic/wrong_format |
| L3 🟠 | 业务 Skill | API 过期、浏览器交互 | cdp/browser/auth/日期/超时 |
| L4 ⚪ | 外部 | 网络不通 | network/502/connection |

```bash
# 一键诊断项目中所有失败 Agent
python3 scripts/project_state.py diagnose <project_id>
```

### 4.5 成本追踪自动化（v1.4 增强）

**之前**：依赖 Sub-agent 自觉在 announce 中汇报 token（实测基本不汇报）

**现在**：CEO 主动通过 `session_status` 获取每个 Sub-agent 的 token 消耗，自动写入项目状态。

预算告警自动触发：>80% 告警用户，>100% 暂停执行。

### 4.6 Persona Priming（v1.3 新增）

给每个角色注入顶级人才的方法论，激活 LLM 预训练中的深层专业知识：

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

### 4.7 CEO 主动监控（v1.3 新增）

解决"spawn 完就失联"的问题：

| 任务复杂度 | 首次检查 | 后续间隔 |
|-----------|---------|----------|
| 简单 | 3min | 每 2min |
| 中等 | 5min | 每 3min |
| 复杂 | 10min | 每 5min |

4 条硬性汇报规则：spawn 确认 / 检查点汇报 / 阶段转换通知 / 异常立即告警。

**双保险**：Sub-agent Heartbeat（可能不到）+ CEO 主动轮询（兜底）。

---

## 五、实战验证

### 5.1 首次联调：315 营销活动（2026-03-09）

模拟完整营销全链路：策划 → 搭建 → 发布。3 角色精简版。

| Task | 角色 | Token | 产出 |
|------|------|-------|------|
| T001 | 策划员 | 10K | 策划方案（主题/人群/券体系/组件规划） |
| T002 | 搭建员 | 48K | 组件树 JSON + 30步 Checklist |
| T003 | 发布员 | 9.2K | QA清单 + 灰度策略 + 回退方案 |
| **总计** | | **67K ≈ $0.17** | **62KB 文档** |

**踩坑**：3个 Sub-agent 中 2个没自动 announce → 建立主动检查机制 + L1/L2/L3 故障归因法。

### 5.2 实战：三会场并行搭建（2026-03-10）

OPC 首次与 gundam-ops（高达搭建 Skill）联合做功。4.1 愚人节「1主+2分」三会场大促。

**编排模式**：策划员×1（串行）→ 搭建员×3（并行）

| 会场 | 活动ID | 耗时 | 状态 |
|------|--------|------|------|
| 主会场 | 1014813 | ~5min | ✅ |
| 外卖分会场 | 1014814 | ~7min | ✅ |
| 到餐分会场 | 1014818 | 17min+重试 | ✅（日期选择器卡死→kill→重试） |

**暴露问题 → 直接驱动 v1.4 四项优化**：

| 问题 | 根因 | v1.4 方案 |
|------|------|----------|
| Compaction 丢失状态 | 状态全在 context | 项目状态持久化 |
| 失败后完整重试 | 无断点保存 | 断点续传 |
| 故障需人工判断 | 无自动归因 | diagnose_agent.py |
| 成本追踪形同虚设 | 靠 Agent "自觉" | CEO 主动采集 |

### 5.3 KangaBase 从零到开源（2026-03-11）

**OPC 最大规模实战**——一天内用 OPC 编排完成 KangaBase（Agent-Native Database）从概念到开源发布。

| Phase | Agent | 产出 | Token |
|-------|-------|------|-------|
| 调研 | 4 研究员并行 | v2 可行性报告 | 1.1M |
| 核心引擎 | core-engineer | 34文件/5270行/68测试 | 218K |
| 文档体系 | doc-writer | README+DESIGN+GUIDE+quickstart | 63K |
| WebUI | webui-engineer | 28文件/7页面管理后台 | 120K |
| CEO 审视 | 喵神 | 修7个bug+15项端到端验证 | ~200K |
| **总计** | **8 个 Agent** | **~100 文件 / 8000行** | **~1.7M** |

**从白板灵感到 GitHub v0.1.0 发布：8小时。**

---

## 5½. v2.0 新增能力（2026-03-12）

借鉴 [Clawith](https://github.com/dataelement/clawith) 的 Aware 自治系统，v2.0 新增两大能力：

### Aware 触发器系统

声明式事件驱动——在 `triggers.yaml` 中定义触发规则，CEO 不再需要手动轮询：

| 触发类型 | 用途 | 示例 |
|---------|------|------|
| **cron** | 定时触发 | 每天 9:30 发日报 |
| **once** | 一次性 | deadline 到了提醒 |
| **interval** | 周期检查 | 每30分钟检查项目状态 |
| **on_message** | 消息触发 | Agent A 完成 → 自动启动 Agent B |
| **poll** | 外部轮询 | CI 构建完成 → 通知 |

配合 **Focus 焦点系统**——项目焦点自适应管理，Focus 下所有 Agent 完成时自动标记完成并取消关联触发器。不需要手动清理调度。

```yaml
# triggers.yaml 示例
triggers:
  - id: phase-transition
    type: on_message
    from_agent: "core-engineer"
    contains: "Phase 1 完成"
    action:
      type: spawn_agent
      agent_label: "doc-writer"
      task: "开始 Phase 2"
```

### 运行时工具自发现

Agent 执行任务前自动扫描可用工具，发现能力缺口时生成推荐报告：

- 本地扫描：自动发现 `~/.openclaw/skills/` 和 `/app/skills/` 下所有 Skill
- 任务匹配：基于 bigram 分词的中英文关键词匹配
- 安全检查：可疑模式检测（curl+token、env 泄露等）
- 推荐报告：已安装/推荐安装/需确认三级分类

**安全优先**：仅推荐官方源，安装需 CEO 确认。

### v2.0 数据

| 指标 | v1.4 | v2.0 | 增量 |
|------|------|------|------|
| 文件数 | 20 | 25 | +5 |
| 代码行数 | 3,213 | 5,521 | +2,308 |
| 脚本 | 4 | 6 | +2（trigger_engine + tool_discovery） |
| 方法论文档 | 10 | 12 | +2（aware-triggers + tool-discovery） |
| 模板 | 3 | 5 | +2（triggers.yaml + focus.yaml） |

---

## 六、OPC vs OpenClaw 原生 Multi-Agent

OpenClaw 本身已有 spawn / heartbeat / cron / lobster 等机制，OPC 的定位：

**原生机制 = 零件，OPC = 组装图纸。**

| 能力 | OpenClaw 原生 | OPC 补充 |
|------|-------------|---------|
| spawn sub-agent | 有 | 角色卡模板，更规范 |
| 任务分解 | 无标准方法 | OKR → Epic → Task |
| 角色定义 | task 自由文本 | 角色卡 + 预置角色库 |
| 依赖管理 | 手动 | 依赖关系 + 状态机 |
| 成本追踪 | 单 session | 项目级自动汇总 |
| 质量验收 | 无 | CEO 检查 + 迭代 |
| 状态持久化 | 无 | project_state.py（v1.4+） |
| 故障恢复 | 手动 | 断点续传 + 自动归因（v1.4+） |
| 事件驱动调度 | 无 | Aware 触发器 + Focus（v2.0） |
| 工具发现 | 手动安装 | 运行时自发现 + 推荐（v2.0） |

**最佳实践**：

```
日常运维 → Heartbeat
定时任务 → Cron
简单委托 → 直接 spawn
复杂项目 → OPC       ← OPC 的战场
固定管道 → Lobster
```

---

## 七、设计哲学

> **OPC 不是让 AI 更聪明，而是让 AI 更有组织。**

```
个人 = 一个聪明人干所有事（有上限）
公司 = 很多人各干一件事（上限消失）

单 Agent = 一个强模型处理所有 context
OPC = 多个 Agent 各处理一小块（上限消失）
```

**OPC 就是 AI Agent 的公司制度。**

---

## 八、下载与使用

### Skill 包下载

| 版本 | 下载 | 大小 | 说明 |
|------|------|------|------|
| **v2.0**（最新） | [agent-orchestration-v2.0.tar.gz](/assets/skills/agent-orchestration-v2.0.tar.gz) | 57KB | 25文件 / 5521行 |
| v1.4 | [agent-orchestration-v1.4.tar.gz](/assets/skills/agent-orchestration-v1.4.tar.gz) | 37KB | 20文件 / 3213行 |
| v1.3 | [agent-orchestration-v1.3.tar.gz](/assets/skills/agent-orchestration-v1.3.tar.gz) | 25KB | 17文件 / 2000+行 |

### 安装

```bash
cd ~/.openclaw/skills
tar xzf agent-orchestration-v2.0.tar.gz
```

### 验证

安装后对 OpenClaw 说"帮我做一个 XX 项目"，它会自动触发 OPC 编排流程。

### 源码

[GitHub — Fozu-lzwpattern/OPC-agent-orchestration](https://github.com/Fozu-lzwpattern/OPC-agent-orchestration)（含完整版本历史 + Git Tags）

### v2.0 文件结构

```
agent-orchestration-20260309-lzw/        (25 files, 5521 lines)
├── SKILL.md                              # 入口
├── CHANGELOG.md                          # 版本变更记录
├── references/
│   ├── design-philosophy.md             # 设计理念（505行）
│   ├── role-design.md                   # 角色设计
│   ├── task-decomposition.md            # 任务分解
│   ├── heartbeat-protocol.md            # 执行协议
│   ├── cost-tracking.md                 # 成本追踪
│   ├── collaboration-patterns.md        # 协作模式
│   ├── project-state.md                 # 项目状态持久化规范
│   ├── troubleshooting.md               # 故障排查
│   ├── persona-priming.md               # Persona 增强
│   ├── proactive-reporting.md           # 主动监控
│   ├── aware-triggers.md                # Aware 触发器系统 ← v2.0 新增
│   └── tool-discovery.md                # 工具自发现 ← v2.0 新增
├── templates/
│   ├── agent-prompt.md                  # Sub-agent prompt 模板
│   ├── project-plan.md                  # 项目计划模板
│   ├── cost-report.md                   # 成本报告模板
│   ├── triggers.yaml                    # 触发器模板 ← v2.0 新增
│   └── focus.yaml                       # Focus 模板 ← v2.0 新增
├── scenarios/
│   └── marketing-campaign.md            # 营销活动全链路案例
└── scripts/
    ├── project_state.py                 # 项目状态管理器（v2.0 扩展5个新命令）
    ├── trigger_engine.py                # Aware 触发器引擎 ← v2.0 核心新增
    ├── tool_discovery.py                # 工具自发现 ← v2.0 核心新增
    ├── diagnose_agent.py                # 自动归因工具
    └── cost_summary.sh                  # 成本汇总
```

---

## 九、版本历程

| 版本 | 日期 | 核心变更 | 驱动因素 |
|------|------|---------|---------|
| v1.0 | 2026-03-09 | 首版：12文件/1100行。角色卡+任务分解+协作模式+成本追踪 | PaperClip 方法论提炼 |
| v1.1 | 2026-03-09 | 设计理念文档（505行）+ 多轮对话组织设定 | 需要完整的哲学框架 |
| v1.2 | 2026-03-09 | OPC vs OpenClaw 原生 Multi-Agent 对比分析 | 明确 OPC 定位 |
| v1.3 | 2026-03-09 | Persona Priming + CEO 主动监控 | 联调暴露 announce 不可靠 |
| v1.4 | 2026-03-11 | 状态持久化 + 断点续传 + 自动归因 + 成本自动化 | 三会场实战暴露四项问题 |
| **v2.0** | **2026-03-12** | **Aware 触发器系统 + 运行时工具自发现 + Focus 焦点管理** | **借鉴 Clawith 的 Aware 自治 + KangaBase 实战需求** |

---

## 十、后续方向

1. **浏览器资源隔离**：多 Agent 并行时的 Tab 竞争问题（短期串行化，中期多 Tab，长期多浏览器）
2. **项目模板库**：把成功项目（如三会场搭建）抽象为可复用模板
3. **Persona Priming A/B 测试**：量化验证 Persona 对产出质量的提升效果
4. **更多场景案例**：内容创作流水线、竞品分析、研究项目
5. **持久角色**：长期运行的专业 Agent，积累领域知识

---

*喵神 & 大仙 | 2026-03-09 创建 · 2026-03-14 更新至 v3.0 | OPC — AI Agent 的公司制度*

---

## 十一、v3.0：架构重构与体验升级（2026-03-14）

### 11.1 两个核心问题驱动本次重构

**问题一：SKILL.md 太重了**

v2.0 的 SKILL.md 有 635 行。每次 OPC 被调用，这 635 行就要被读入上下文。随着功能越来越多，这个问题会越来越严重。参考 gundam-ops v3 的三层架构实践（7544 行 → 1282 行，-83%），我们决定对 OPC 做同样的事情。

**问题二：OPC 经常"被跳过"**

复杂任务进来，OpenClaw 有时候直接开始做，而不是先想"应不应该用 OPC"。原因是 SKILL.md 的 description 不够具体——AI 不知道什么条件下该触发 OPC。

### 11.2 三层架构

```
门面层   SKILL.md (~200行)     ← 每次加载，30秒上手
决策层   brain/               ← 只在需要时读
引擎层   engine/              ← 工具执行
知识层   playbook/            ← 参考素材
```

SKILL.md 的职责从"容纳所有内容"变成"告诉 CEO 去哪里找内容"。

### 11.3 Phase 0：Context Intake（最重要的新增）

v3.0 最核心的变化：**OPC 被触发后，第一步必须是理解背景，给出推进方案，等用户确认后才执行**。

过去的问题：OPC 有时候理解了任务就直接 spawn 了，用户来不及说"等等，我想要的不是这个"。

解决方案：强制 Phase 0，四维摄入（背景/目标/约束/范围），标准化方案输出，用户 OK 才开始。

这是从工具到合作伙伴的转变——不是被动执行命令，而是主动理解意图。

### 11.4 工具发现 v2：标签体系

`tool_discovery.py` 从纯关键词匹配升级为 domain × capability 双轴标签体系：

- 12 个 domain 标签（marketing / research / browser / code / data ...）
- 8 个 capability 标签（create / edit / query / publish / analyze ...）
- 标签匹配（精确）+ 关键词（兜底）双路匹配
- OPC 自排除：不再把自己推荐给任务

实测：「为315营销活动在高达平台搭建页面」→ gundam-ops 精确匹配 `domain:marketing; cap:publish`。

### 11.5 v3.0 验证案例：格式塔科技深度分析

2026-03-14，用 v3.0 跑了第一个真实案例：

**任务**：格式塔科技（非侵入神经调控）深度分析报告

**执行**：
- Phase 0：Context Intake → 确认方案（3研究员并行+整合员）
- 研究员A（Mary Meeker）：公司基本面
- 研究员B（Jeff Dean）：技术路线  
- 研究员C（Clayton Christensen）：市场生态
- 整合员（Peter Drucker）：深度分析报告

**结果**：4500字深度报告，综合评级7.5/10，总消耗40K tokens（$0.10）

v3.0 全链路验证通过。

### 11.6 更新版本历程

| 版本 | 日期 | 核心变更 |
|------|------|---------|
| v1.0-v1.3 | 2026-03-09 | 首版 → Persona Priming + CEO 主动监控 |
| v1.4 | 2026-03-11 | 状态持久化 + 断点续传 + 自动归因 |
| v2.0 | 2026-03-12 | Aware 触发器 + 运行时工具自发现 |
| **v3.0** | **2026-03-14** | **三层架构 + Context Intake + 标签体系 v2** |

### 11.7 后续方向（更新）

- **工具标签精度提升**：部分 Skill 误报（sso/km-wiki），待进一步调优
- **`init --type` 参数**：自动生成触发器模板（按项目类型预填）
- **更多场景案例**：竞品分析、内容创作流水线、代码工程
- **持久角色**：长期运行的专业 Agent，积累领域知识

---

*2026-03-14 更新至 v3.0 | 三层架构重构 + Context Intake + 格式塔深度分析验证*
