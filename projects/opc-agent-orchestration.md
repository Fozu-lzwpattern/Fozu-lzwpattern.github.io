---
title: "OPC — One-Person Company"
parent: 开源项目
nav_order: 1
---

# OPC — One-Person Company (Agent Orchestration)

> 让 OpenClaw 成为能指挥调度专业 Agent 团队的"一人公司"CEO

**版本**: v3.1 (2026-03-14)  
**作者**: 喵神 & 大仙  
**平台**: [OpenClaw](https://github.com/openclaw/openclaw)  
**开源仓库**: [Fozu-lzwpattern/OPC-agent-orchestration](https://github.com/Fozu-lzwpattern/OPC-agent-orchestration)

---

## 概述

OPC (One-Person Company) 是一个 OpenClaw Skill，实现了 **Multi-Agent 编排范式**：

```
用户（老板）→ OpenClaw（CEO）→ Sub-agents（专业员工）
```

用户只需说"我要做 X"，CEO 自动完成任务分解、角色定义、团队组建、执行监控、成果汇总。

**v3.0 亮点**：三层架构 + Context Intake 强制入口 + 工具发现标签体系 v2

---

## 核心特性

### 🧠 Phase 0：Context Intake（v3.0 新增）
- OPC 被触发后，**第一步必须理解背景目标，给出推进方案，等用户确认后才执行**
- 四维摄入框架：背景 / 目标 / 约束 / 范围
- 标准化方案输出格式（角色配置 + 协作模式 + 预算 + 耗时）
- 复杂度自动判断（简单/中等→1轮确认，复杂→2轮）

### 🏗️ 三层架构（v3.0 新增）
- **brain/**（决策层）：CEO 怎么想——四阶段流程、任务分解、角色设计、协作模式
- **engine/**（执行层）：CEO 怎么跑——项目状态管理、触发器、工具发现、自动归因
- **playbook/**（知识层）：参考素材——实战案例、Prompt 模板、Persona 库

### 🔍 工具发现 v2（v3.0 新增）
- domain × capability 双轴标签体系（12 × 8 = 96 标签组合）
- 标签匹配优先 + 关键词兜底双路匹配
- OPC 自排除：不把自己推荐给任务
- 新增 `enrich` 命令查看任意 Skill 的标签分析

### 🔔 触发器融入生命周期（v3.0 增强）
- Phase 3 每轮循环必须 `trigger-evaluate`（写入强制规范）
- `agent-complete` 内置 on_message 触发检查
- 项目关闭时自动停用相关触发器

### 🏢 组织编排（核心）
- OKR → Epic → Task 三层任务分解
- 角色卡规范：单一职责、Persona Priming、预算约束
- 4 种协作模式：串行流水线 / 并行分工 / 迭代反馈 / 分治汇总

### 🗄️ 状态持久化
- `project_state.py` — 项目全生命周期状态管理
- Compaction 后一键恢复：`restore <id>`
- 断点续传：失败后保存断点，新 Agent 从断点继续

---

## 文件结构（v3.0）

```
agent-orchestration-20260309-lzw/
├── SKILL.md              ← 入口（207行，v2.0 是 635 行）
├── CHANGELOG.md
├── brain/                ← CEO 决策层（怎么想）
│   ├── core-flow.md      ← 四阶段流程 + Phase 0 Context Intake
│   ├── task-decomposition.md
│   ├── role-design.md
│   ├── collaboration-patterns.md
│   ├── heartbeat-protocol.md
│   ├── proactive-reporting.md
│   └── cost-tracking.md
├── engine/               ← 执行引擎层（怎么跑）
│   ├── project_state.py  ← 核心：项目状态管理
│   ├── trigger_engine.py ← Aware 触发器
│   ├── tool_discovery.py ← 工具发现（标签体系 v2）
│   ├── diagnose_agent.py ← 自动归因
│   └── README.md         ← 命令速查
└── playbook/             ← 知识与模板层
    ├── persona-priming.md
    ├── philosophy.md
    ├── templates/
    └── scenarios/
        ├── marketing-campaign.md   ← 营销活动全链路
        └── research-project.md    ← 并行研究分治汇总
```

---

## 实战验证

### 2026-03-09 首次联调
- 315营销活动：3角色串行（策划→搭建→发布）
- 总消耗 67K tokens ($0.17)

### 2026-03-10 OPC + gundam-ops 三会场搭建
- 4.1愚人节「1主+2分」三会场大促
- 策划员×1 → 搭建员×3 并行
- 首次验证多 Skill 联动

### 2026-03-11 KangaBase MVP 工程（OPC opc-20260311-002）
- Phase 1 核心引擎：34文件/5270行，68/68测试通过
- Phase 2 文档体系：README + DESIGN + GUIDE + quickstart.sh
- Phase 3 WebUI：28文件/2668行，7页面全部200 OK
- 总消耗 ~600K tokens

### 2026-03-12 Kangas2 嵌入式认知研究（4研究员并行）
- 4 研究员并行调研（模型横评/认知架构/LoRA/MiniMind）
- 产出 3 份报告共 2620 行
- 总消耗 ~320K tokens

### 2026-03-14 格式塔科技深度分析（v3.0 验证）
- 3 研究员并行（公司基本面/技术路线/市场生态） + 1 整合员
- 产出：4500 字深度分析报告，综合评级 7.5/10
- 总消耗 40K tokens（$0.10）
- **完整验证 OPC v3.0 全链路**

---

## 版本历程

| 版本 | 日期 | 核心变更 |
|------|------|---------|
| v1.0 | 2026-03-09 | 首版：角色+任务+协作+成本+场景 |
| v1.3 | 2026-03-09 | Persona Priming + CEO 主动监控 |
| v1.4 | 2026-03-11 | 状态持久化 + 断点续传 + 自动归因 + 成本自动化 |
| v2.0 | 2026-03-12 | Aware 触发器 + 运行时工具自发现 |
| **v3.0** | **2026-03-14** | **三层架构 + Context Intake + 标签体系 v2** |

---

## 相关资源

- 🔗 [GitHub 开源仓库](https://github.com/Fozu-lzwpattern/OPC-agent-orchestration)
- 📖 [设计理念研究文章](/research/2026-03-09-opc-one-person-company)

---

*喵神 & 大仙 | 2026-03-14 | OpenClaw Skill* 🐱
