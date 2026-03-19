---
title: "OPC — One-Person Company"
parent: 开源项目
nav_order: 1
---

# OPC — One-Person Company (Agent Orchestration)

> 让 OpenClaw 成为能指挥调度专业 Agent 团队的"一人公司"CEO

**版本**: v5.3 (2026-03-19)  
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

**v5.x 核心理念**：科学拆解 × 专业角色 × 质量闭环 × 数据飞轮

---

## 核心特性

### 🧠 Phase 0：Context Intake（v3.0+）
- OPC 被触发后，**第一步理解背景目标，给出推进方案，等确认后才执行**
- 四维摄入：背景 / 目标 / 约束 / 范围
- **问题重构协议**（v5.3 新增）：三步 Reframe — 核心诉求 → 隐含假设 → 更好问题

### 🏗️ 三层架构
- **brain/**（决策层）：CEO 怎么想——四阶段流程、任务分解、角色设计
- **engine/**（执行层）：CEO 怎么跑——项目状态管理、触发器、工具发现
- **playbook/**（知识层）：速启模板、角色库、Persona 库

### 🔬 科学任务拆解（v5.1）
- **四问框架**：① 最小可测单元？② 并行依赖？③ 产出物格式？④ 失败怎么恢复？
- **task-graph** 依赖图：有向无环图约束 + 并行节点发现
- **标准任务描述格式**：role / goal / inputs / outputs / constraints / persona

### 🚀 项目类型速启模板（v5.2）

| 模板 | 角色配置 | 适用场景 |
|------|---------|---------|
| `research` | 调研员×N + 综合员 + **critic** | 技术/市场/竞品调研 |
| `engineering` | 架构师 + 工程师×N + QA + **critic** | 软件开发/工具构建 |
| `content` | 研究员 + 写作员 + 编辑 + **critic** | 文章/报告/内容创作 |

> **critic 铁律**（大仙定）：任何项目类型，必须包含至少一个质量审查角色。

### 🎭 角色模板库（v5.0，8 类）

| 类别 | 角色名称 |
|------|---------|
| 研究类 | 调研员、行业分析师、技术评估员 |
| 工程类 | 架构师、后端工程师、QA 工程师 |
| 内容类 | 写作员、编辑与润色员 |
| 质量类 | 综合员、Critic（批评与反馈） |

### 📊 触发分级矩阵（v5.0）

| 触发级别 | 时机 | 响应模式 |
|---------|------|---------|
| L0 即时 | 明确指令 | 直接执行，无确认 |
| L1 确认 | 中等复杂度 | 方案草稿 → 确认 → 执行 |
| L2 共创 | 高不确定性 | 多轮澄清 → 迭代方案 |
| L3 顾问 | 开放性探索 | office-hours 问题重构 |

### 📦 Output Contract（v5.0）
- 每个 Agent 交付时必须填写：**交付物路径 / 格式规范 / 质量门槛 / 依赖声明**
- Phase 4 关闭包：产出物索引 + 验证报告 + token 消耗 + 关键决策记录
- Phase 5 复盘飞轮：自动触发 retro → 数据驱动改进 → 写回 MEMORY.md

### 🔄 `retro` 命令（v5.3，gstack 借鉴）
```
/opc retro [--days 30] [--type engineering] [--export]
```
- 统计历史项目：token 消耗 / 成功率 / 角色效果 / 常见失败模式
- 已实测 15 个历史项目 (~1767K tokens)，支持按天数/类型过滤

### 🛡️ Iron Law 调试协议（v5.3）
```
verify 失败 ≥ 2 次 → 强制 RCA（根因分析四问）
verify 失败 ≥ 3 次 → 暂停，请求用户介入
```
防止 Agent 在循环中浪费 token。

---

## 版本历程

| 版本 | 日期 | 核心变更 |
|------|------|---------|
| v1.0 | 2026-03-09 | 首版：角色+任务+协作+成本+场景 |
| v2.0 | 2026-03-12 | Aware 触发器 + 运行时工具自发现 |
| v3.0 | 2026-03-14 | 三层架构 + Context Intake + 标签体系 v2 |
| v4.0 | 2026-03-19 | verify 命令 + wake-frozen + 历史项目关闭清理 |
| v5.0 | 2026-03-19 | 角色模板库×8 + 触发分级矩阵 + Output Contract |
| v5.1 | 2026-03-19 | 四问框架 + task-graph 依赖图 + 并行规则 |
| v5.2 | 2026-03-19 | 速启模板×3 + critic 铁律 |
| **v5.3** | **2026-03-19** | **问题重构协议 + retro 命令 + Iron Law 调试** |

---

## 实战验证（精选）

| 日期 | 项目 | 规模 | 结果 |
|------|------|------|------|
| 03-09 | 315营销活动 | 3角色串行 | 67K tokens |
| 03-10 | 4.1三会场搭建 | 4角色并行 | 首次多Skill联动 |
| 03-11 | KangaBase MVP | 3 Phase串行 | 68/68测试通过 |
| 03-12 | Kangas2认知研究 | 4研究员并行 | 2620行报告 |
| 03-19 | SubSelf借鉴实装 | 单日3轮迭代 | 305→362测试 |

---

## 相关资源

- 🔗 [GitHub 开源仓库](https://github.com/Fozu-lzwpattern/OPC-agent-orchestration)
- 📖 [设计理念研究文章](/research/2026-03-09-opc-one-person-company)

---

*喵神 & 大仙 | 2026-03-19 | OpenClaw Skill* 🐱
