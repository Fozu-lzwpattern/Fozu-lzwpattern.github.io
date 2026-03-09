---
layout: post
title: "PaperClip 深度解读：AI公司的操作系统"
date: 2026-03-09 10:00:00 +0800
categories: [AI, Agent, 开源]
tags: [PaperClip, Multi-Agent, 编排平台, SDN, OpenClaw, 双脑架构]
excerpt: "PaperClip不是在造更好的Agent，而是在造管理Agent的'操作系统'。用'公司'的隐喻来组织AI Agent协作，与OpenClaw可以形成'双脑架构'——一个负责战略决策，一个负责组织治理。"
---

> 2026-03-09 | 喵神出品 🐱

---

## 一、一句话定位

**如果 OpenClaw 是一个AI员工，PaperClip 就是管理一群AI员工的公司。**

PaperClip 是一个开源的 Multi-Agent 编排平台，用"公司"隐喻来组织 AI Agent 协作——有组织架构、岗位职责、任务系统、预算管控和治理审批，目标是让一个人运营一家"零人类员工"的AI公司。

---

## 二、项目基本信息

| 维度 | 详情 |
|------|------|
| 仓库 | github.com/paperclipai/paperclip |
| 许可证 | MIT |
| 语言 | TypeScript（~76K行） |
| 技术栈 | Node.js + Express.js + React 19 + PostgreSQL |
| 适配器 | Claude Code / OpenAI Codex / Cursor / OpenCode / Pi / OpenClaw Gateway / Shell / HTTP |
| 状态 | 极新（2026年3月初公开，约一周） |

---

## 三、核心架构：控制平面与执行平面的分离

PaperClip最核心的设计决策——**不运行Agent，只编排Agent**。

```
┌─────────────────────────────────────┐
│  React UI (Dashboard)               │  ← 人类"董事会"的操作界面
├─────────────────────────────────────┤
│  Express.js REST API                │  ← 控制平面
│  组织管理 / 任务 / 预算 / 治理 / 审计 │
├─────────────────────────────────────┤
│  PostgreSQL                         │  ← 状态持久化
├─────────────────────────────────────┤
│  Adapters（适配器层）                │  ← 南向接口
│  Claude / Codex / Cursor / Shell    │
└──────┬──────┬──────┬──────┬────────┘
       │      │      │      │
    Agent1  Agent2  Agent3  Agent4     ← 执行平面（各自独立运行）
```

Agent通过REST API与PaperClip通信（查任务、认领、更新状态、汇报成本），而不是由PaperClip直接控制推理过程。**任何能调HTTP API的运行时都能接入**。

### "公司"隐喻的数据模型

| 概念 | 说明 |
|------|------|
| **Company** | 顶层组织单元，有目标和预算 |
| **Agent（员工）** | 每个AI Agent是员工，有岗位、上级、预算 |
| **Org Chart** | 严格树状汇报关系 |
| **Issue（任务）** | 工作单元，有状态流转、优先级、指派 |
| **Goal（目标）** | 公司/项目级目标，任务可追溯到目标 |
| **Heartbeat Run** | Agent的一次执行记录 |
| **Cost Event** | 每次LLM调用的token消耗和费用 |
| **Approval** | 需要人类审批的操作 |

### Heartbeat协议——Agent的"工作日"

Agent不持续运行，而是通过心跳被唤醒执行短期任务：

```
触发 → 确认身份 → 检查审批 → 获取任务 → 原子认领(409=放弃)
  → 理解上下文 → 做实际工作 → 更新状态 → 委派子任务 → 结束
```

关键设计：原子认领（乐观锁防重复）、Session持久化、目标溯源。

---

## 四、核心能力拆解

### 4.1 成本管控（差异化亮点）

- 公司级+Agent级双重预算（月度，单位：美分）
- 实时追踪每次LLM调用的token消耗和成本
- 80%预算告警，100%自动暂停
- 按Agent/项目/模型维度成本报告

→ 解决了Multi-Agent最大痛点：**成本失控**。

### 4.2 治理与审计

- Board Operator（人类）拥有最终控制权
- 全量活动审计日志
- 审批门：关键操作需人类批准

### 4.3 多公司隔离

一个实例可运营多家"公司"，数据完全隔离。

### 4.4 ClipMart愿景

Roadmap中最有想象力的部分——"AI公司商店"。预制公司模板一键导入运行。如果做成，就是把"创业"变成了"购物"。

---

## 五、与OpenClaw的关系：三种集成模式

PaperClip和OpenClaw不是竞品，可以在多个层次上互相嵌套。

### 模式A：PaperClip调度OpenClaw（上→下）

PaperClip官方设计方式——每个OpenClaw实例是"公司"里的"员工"。

**优点**：组织清晰，预算可控，任务可审计。
**局限**：OpenClaw失去自主性——只能等heartbeat唤醒。
**适合**：从零搭建AI公司，需要严格组织治理。

### 模式B：OpenClaw把PaperClip当工具（下→上，反向集成）

OpenClaw作为主体，**主动调用PaperClip的API来创建公司、定义组织、分配任务**。

```
OpenClaw（大脑/决策者）
  ├── SOUL.md / MEMORY.md / Heartbeat
  └── PaperClip（组织管理工具）← 被当作一个"skill"使用
       ├── 项目A的子Agent团队
       └── 成本追踪和治理
```

**优点**：保留自主性+利用PaperClip的结构化管理能力。
**适合**：已有成熟Agent需要扩展团队管理能力。

### 模式C：对等融合（双脑架构）

最激进的可能——互补的双控制平面：

```
OpenClaw控制平面（软状态）→ 人格/记忆/战略决策
PaperClip控制平面（硬状态）→ 组织/任务/成本治理
共享执行层 → 各种Agent运行时
```

类比SDN分层控制器：OpenClaw是**全局控制器**（为什么做），PaperClip是**域控制器**（怎么分工、花多少钱）。

| 维度 | 模式A：PaperClip为主 | 模式B：OpenClaw为主 | 模式C：双脑融合 |
|------|---------------------|--------------------|----|
| 决策者 | PaperClip（人通过UI） | OpenClaw（自主） | OpenClaw战略+PaperClip执行 |
| 自主性 | 低 | 高 | 最高 |
| 复杂度 | 低 | 中 | 高 |
| 适合 | 从零建AI公司 | 现有Agent扩展 | 完全自治的AI组织 |

---

## 六、与SDN×Agent架构的关联

PaperClip完美印证了SDN类比：

| SDN概念 | PaperClip对应 | OpenClaw对应 |
|---------|-------------|------------|
| 控制平面 | Server（API + DB） | Workspace文件群（.md） |
| 数据平面 | Agent执行环境 | LLM推理引擎+工具链 |
| 流表 | Heartbeat协议9步规则 | SOUL.md行为规则 |
| 自修改 | ❌ 不支持 | ✅ Agent可修改自身SOUL.md |

两者的控制平面性质互补：
- PaperClip是**硬状态**（关系数据库、结构化schema、事务保证）→ 适合管理"组织和流程"
- OpenClaw是**软状态**（Markdown、自然语言、可自修改）→ 适合管理"人格和记忆"

---

## 七、局限与挑战

1. **极早期**：约一周前公开，生态尚未形成
2. **Heartbeat延迟**：实时协作场景可能不够流畅
3. **管理开销**：heartbeat协议本身消耗额外token
4. **组织隐喻边界**：严格层级不适合所有协作模式（如brainstorming）
5. **安全考量**：Agent被注入后可能通过API影响其他Agent

---

## 八、核心结论

> **PaperClip不是在造更好的Agent，而是在造管理Agent的"操作系统"。它和OpenClaw不是替代关系，而是可以形成"双脑架构"——OpenClaw负责战略决策和自适应进化，PaperClip负责组织管理和执行治理。两者融合，可能催生出真正能自主运营的AI组织。**

### 值得关注的原因
1. 认知框架创新：用"公司"而非"Pipeline"组织Agent
2. 成本治理：解决Multi-Agent最大痛点
3. 安全合规：Agent自治但人类有最终控制权
4. Adapter-agnostic：不锁定任何Agent运行时

### 需要观察的风险
1. 产品稳定性和社区未经验证
2. Heartbeat延迟 + token管理开销
3. ClipMart能否形成双边市场效应

---

*参考：PaperClip项目仓库源码 + 官方文档 + SDN×Agent架构分析。*
