---
title: "gundam-ops — 高达智能搭建"
parent: 开源项目
nav_order: 2
---

# gundam-ops — 高达3.0营销活动智能搭建技能

> Agent 增强版：意图理解→参数推断→反向确认→自动执行

**版本**: v2.1 (2026-03-10)  
**作者**: 大仙 & 喵神  
**平台**: [OpenClaw](https://github.com/openclaw/openclaw)  
**仓库**: [Fozu-lzwpattern/miaoshen-brain](https://github.com/Fozu-lzwpattern/miaoshen-brain) → `skills/gundam-ops-202603091158-lzw/`

---

## 概述

gundam-ops 是高达3.0营销活动搭建平台的完整操作技能，支持两种运行模式：

- **(A) Agent 智能模式**：意图理解→参数推断→反向确认→自动执行
- **(B) OPC 编排模式**：接收 component-plan.json → 自动搭建 → 返回标准结果

## 核心特性

### 🧠 Agent 智能层
- **意图理解** — 自然语言→结构化搭建需求
- **参数推断** — 三分类：必填/可推断/可选
- **反向确认** — 批量确认优于逐个询问
- **组件推荐引擎** — 5种活动类型自动推荐组件组合

### 🔧 自动化执行
- **E2E 全链路** — 创建→组件添加→配置→保存→发布（24-29秒）
- **Snapshot 注入法** — 一次性加载完整组件树，比逐步操作快5-10倍
- **574个组件索引** — 全量组件元数据（components-index.json, 149KB）
- **React Fiber 通用模式** — 绕过 AntD 表单限制直接操作

### 🛡️ 可靠性
- **分层错误处理** — L1~L4 自动分级处理
- **32条自愈规则** — 断点续传、自动重试
- **发布诊断修复** — 自动检测并修复发布前问题
- **方法选择决策树** — Snapshot注入 vs 逐步UI，场景化推荐

### 🔗 OPC 集成
- 标准接口：component-plan.json（输入）/ execution-result.json（输出）
- 4种标准消息：PROGRESS / PARAM_CONFIRM / RESULT / ERROR_ESCALATION
- 推荐与 OPC（agent-orchestration）联合使用

## 文件规模

```
49 files / ~11,000 lines
├── SKILL.md (500+ lines)
├── references/ (18 files)
├── knowledge/ (12 files, incl. 574-component index)
├── templates/ (2 files)
└── scripts/ (8 files)
```

## 实战验证

### 2026-03-10 三会场并行搭建
- 4.1愚人节「1主+2分」三会场
- OPC 策划员→3搭建员并行
- 活动ID: 1014813 / 1014814 / 1014818
- 总耗时: ~15分钟完成三个会场

## 版本历程

| 版本 | 核心变更 |
|------|---------|
| v1.0 | 基础操作：活动创建/组件/发布 |
| v2.0 | Agent增强：意图理解/自愈/E2E/OPC集成 (13新增文件) |
| v2.1 | 大仙合并版：574组件索引/头图BASE框架/知识库版本管理 |

---

*大仙 & 喵神 | 2026-03-10 | OpenClaw Skill* 🐱
