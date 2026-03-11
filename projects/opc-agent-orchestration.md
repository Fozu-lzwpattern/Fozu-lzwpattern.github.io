---
title: "OPC — One-Person Company"
parent: 开源项目
nav_order: 1
---

# OPC — One-Person Company (Agent Orchestration)

> 让 OpenClaw 成为能指挥调度专业 Agent 团队的"一人公司"CEO

**版本**: v1.4 (2026-03-11)  
**作者**: 喵神 & 大仙  
**平台**: [OpenClaw](https://github.com/openclaw/openclaw)  
**仓库**: [Fozu-lzwpattern/miaoshen-brain](https://github.com/Fozu-lzwpattern/miaoshen-brain) → `skills/agent-orchestration-20260309-lzw/`

---

## 概述

OPC (One-Person Company) 是一个 OpenClaw Skill，实现了 **Multi-Agent 编排范式**：

```
用户（老板）→ OpenClaw（CEO）→ Sub-agents（专业员工）
```

用户只需说"我要做 X"，CEO 自动完成任务分解、角色定义、团队组建、执行监控、成果汇总。

## 核心特性

### 🏢 组织编排
- **OKR → Epic → Task** 三层任务分解
- **角色卡规范** — 单一职责、预算约束、能力边界
- **4种协作模式** — 串行流水线、并行分工、迭代反馈、分治汇总
- **Persona Priming** — 注入顶级人才方法论提升产出质量

### 📊 项目管理
- **确认流程分级** — 简单1轮 / 中等2轮 / 复杂3轮（v1.4）
- **CEO 主动监控** — 不信任 announce，定期主动检查 agent 状态
- **四条汇报规则** — spawn确认 / 检查点汇报 / 阶段通知 / 异常告警

### 🗄️ 状态持久化（v1.4 新增）
- **解决 Compaction 导致的执行状态丢失**
- `project_state.py` — 项目全生命周期状态管理
- 状态持久化到 `workspace/opc-projects/{id}/state.json`
- Compaction 后一键恢复：`restore <id>`

### 🔄 断点续传（v1.4 新增）
- Agent 失败后不再完全重试
- 保存断点（已完成步骤 + 上下文）→ 新 Agent 从断点继续
- 断点续传 prompt 模板标准化

### 🔍 自动归因（v1.4 新增）
- `diagnose_agent.py` — 规则引擎自动分类故障层级
- L1 🔴 平台 / L2 🟡 编排 / L3 🟠 业务 Skill / L4 ⚪ 外部依赖
- 支持中英文关键词匹配
- 输出诊断报告 + 建议操作 + 断点可用性

### 💰 成本追踪自动化（v1.4 增强）
- CEO 主动通过 `session_status` 获取 token 消耗
- 不依赖 Sub-agent 自觉汇报
- 预算告警自动计算（80% / 100% 阈值）

## 文件结构

```
agent-orchestration-20260309-lzw/      (20 files, 3213 lines)
├── SKILL.md                           # 入口（503行）
├── CHANGELOG.md                       # 版本记录
├── references/                        # 方法论
│   ├── role-design.md                # 角色定义
│   ├── task-decomposition.md         # 任务分解
│   ├── heartbeat-protocol.md         # Agent 执行协议
│   ├── cost-tracking.md              # 成本追踪（v1.4 自动化）
│   ├── collaboration-patterns.md     # 协作模式
│   ├── project-state.md              # 状态持久化规范（v1.4）
│   ├── troubleshooting.md            # 故障排查（v1.4 自动归因）
│   ├── design-philosophy.md          # 设计理念（505行）
│   ├── persona-priming.md            # 角色增强
│   └── proactive-reporting.md        # CEO 主动监控
├── templates/                         # 可复用模板
│   ├── agent-prompt.md               # Sub-agent prompt 模板
│   ├── project-plan.md               # 项目计划模板
│   └── cost-report.md                # 成本报告模板
├── scenarios/                         # 场景案例
│   └── marketing-campaign.md         # 营销活动全链路（305行）
└── scripts/                           # 自动化脚本
    ├── project_state.py              # 项目状态管理器（v1.4 核心）
    ├── diagnose_agent.py             # 自动归因工具（v1.4）
    ├── cost_summary.sh               # 成本汇总
    └── project_tracker.py            # 项目追踪（旧版）
```

## 实战验证

### 2026-03-09 首次联调
- 315营销活动：3角色串行（策划→搭建→发布）
- 总消耗 67K tokens ($0.17)
- 发现 announce 不可靠问题 → 增加主动检查机制

### 2026-03-10 OPC + gundam-ops 三会场搭建
- 4.1愚人节「1主+2分」三会场大促
- 策划员×1 → 搭建员×3 并行
- 主会场 5min ✅ / 外卖 7min ✅ / 到餐 17min+重试 ✅
- 暴露问题 → 驱动 v1.4 四项优化

## 版本历程

| 版本 | 日期 | 核心变更 |
|------|------|---------|
| v1.0 | 2026-03-09 | 首版：12文件/1100行，角色+任务+协作+成本+场景 |
| v1.1 | 2026-03-09 | 设计理念文档（315行） |
| v1.2 | 2026-03-09 | OPC vs OpenClaw原生对比分析 |
| v1.3 | 2026-03-09 | Persona Priming + CEO主动监控 |
| v1.4 | 2026-03-11 | 状态持久化 + 断点续传 + 自动归因 + 成本自动化 |

## 设计理念

详见研究文章：[OPC：让 OpenClaw 成为一人公司的 CEO](/research/2026-03-09-opc-one-person-company)

核心原则：
1. **CEO 不干活，只管人** — 规划、分配、监控、汇总
2. **角色即能力边界** — 角色卡是合同，不是建议
3. **信任但验证** — 超时检查 + 质量验收 + 成本追踪
4. **编排与执行分离** — OPC 管协作，业务 Skill 管操作
5. **用户始终在回路中** — 关键节点必须确认

## 与 gundam-ops 的关系

```
OPC（编排层）
  │ spawn + 角色注入
  ├── gundam-ops（高达搭建执行）
  ├── calendar-api（日程管理）
  ├── catclaw-search（搜索）
  └── ... 更多业务 Skill
```

OPC 负责"谁做什么"，gundam-ops 负责"怎么做"。两者通过标准化接口（component-plan.json）对接。

---

*喵神 & 大仙 | 2026-03-11 | OpenClaw Skill* 🐱
