---
title: KangaBase 🦘
parent: 开源项目
nav_order: 2
---

# KangaBase 🦘 — Agent-Native Database

> 轻如 SQLite、易如 Supabase、为 Agent 而生

[![GitHub](https://img.shields.io/github/stars/Fozu-lzwpattern/kangabase?style=social)](https://github.com/Fozu-lzwpattern/kangabase)
[![Version](https://img.shields.io/badge/version-v0.1.0-blue)](https://github.com/Fozu-lzwpattern/kangabase/releases/tag/v0.1.0)
[![License](https://img.shields.io/badge/license-Apache%202.0-green)](https://github.com/Fozu-lzwpattern/kangabase/blob/main/LICENSE)

## 项目简介

KangaBase 是一个 **Agent-Native 数据库**——不是给 Agent 用的传统数据库，而是专门为 AI Agent 时代重新设计的数据基础设施。

核心设计：
- **YAML Schema**：声明式语义模型，Agent 和人类都能理解
- **操作契约（Contract）**：预定义操作白名单，Agent 只能"从菜单点菜"
- **Intent Registry**：架构枢纽，统一意图路由和安全白名单
- **SQLite 存储**：零配置、单文件、嵌入式

## 核心特性

| 特性 | 描述 |
|------|------|
| 📐 YAML 声明式 Schema | 人类可读的数据模型，自带语义信息 |
| 📜 操作契约 Contract | 预定义操作白名单，从根源杜绝越权 |
| 🧭 意图路由 Intent | 自然语言或结构化调用统一路由 |
| 🛡️ 沙箱预执行 Sandbox | 高风险操作先 dry-run，确认后执行 |
| 🔐 意图级权限 Policy | 基于角色的细粒度权限控制 |
| 📋 全链路审计 Audit | 每次操作都有迹可循 |
| 🎯 Agent SDK | 一行创建 Agent，结构化/NL 双模调用 |
| 🎨 WebUI | 极简风格管理后台，7个功能页面 |

## 设计理念

KangaBase 诞生于**3A 范式**（Assistant × Autopilot × Avatar）和 **Agentic Commerce**（asC/asB/A2A）的思考——当 Agent 成为商业世界的一等公民，它们需要一个理解"意图"而非"SQL"的数据库。

详见：[设计理念文档](https://github.com/Fozu-lzwpattern/kangabase/blob/main/docs/DESIGN.md)

## 快速开始

```python
import kangabase as kb

db = kb.open("shop.db")
db.load_schema("schemas/coupon.yaml")
db.load_contract("contracts/coupon_ops.yaml")
db.load_policy("policies/permissions.yaml")

agent = db.agent("coupon_bot", role="asB")
result = agent.execute("issue_coupon", {
    "user_id": "12345",
    "amount": 10,
    "campaign_id": "spring"
})
```

## 链接

- **GitHub**: [Fozu-lzwpattern/kangabase](https://github.com/Fozu-lzwpattern/kangabase)
- **设计文档**: [DESIGN.md](https://github.com/Fozu-lzwpattern/kangabase/blob/main/docs/DESIGN.md)
- **使用指南**: [GUIDE.md](https://github.com/Fozu-lzwpattern/kangabase/blob/main/docs/GUIDE.md)

---

*2026-03-11 · KangaBase v0.1.0 · Apache 2.0*
