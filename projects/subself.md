---
title: "SubSelf — 数字分身"
parent: 开源项目
nav_order: 3
---

# SubSelf — 数字分身

> 不是更好的 AI 助手，而是你在 Agent 时代的另一个生命形态

**版本**: v1.4 (2026-03-19)  
**作者**: 喵神 & 大仙  
**开源仓库**: [Fozu-lzwpattern/subself](https://github.com/Fozu-lzwpattern/subself)  
**定位**: asC（作为消费者的 Agent）自然人阶段可运行原型

---

## 核心理念

SubSelf 回答一个不同于 OpenClaw 的问题：

> OpenClaw 回答"怎么让 AI 更好用"，SubSelf 回答"在 Agent 时代你是谁"

三层产品哲学：

| 层次 | 内容 |
|------|------|
| 能力哲学 | 记住你 → 代表你 → 成就你 |
| 进化哲学 | 自然人 → 社会人 → 契约人 |
| 生命坐标系 | 时间轴 / 存在轴 / 能力轴 / 情感轴 / 关系轴 |

---

## 架构

全新自研技术栈（不依赖 OpenClaw）：

```
packages/
├── core/          Fastify + KangaBase + 业务逻辑
│   ├── kernel/    gateway.ts（HTTP + WebSocket）
│   ├── agent/     loop.ts（Agent 对话主循环）
│   ├── db/        schema + migrations（011个）
│   ├── memory/    记忆三层架构 + 知识图谱
│   ├── skills/    Skill 能力清单管理
│   └── goals/     目标追踪 + 情报监控
├── webui/         React + Vite（9页面）
└── desktop/       Tauri v2 桌面壳（macOS .dmg）
```

---

## 核心功能

### 🧠 三层记忆架构（OpenFang 借鉴）

| 层级 | 衰减规则 | 晋升条件 |
|------|---------|---------|
| episodic | 7天未访问 × 0.9，最低 0.1 | 访问3次 + 置信≥0.7 → semantic |
| semantic | 30天未访问 × 0.95 | 用户/系统显式标记 → core |
| core | 永不衰减（confidence=1.0） | — |

记忆有"生命感"：最近的、被反复访问的记忆更强，久未访问的自然沉淀而非删除。

### 🕸 知识图谱

理解因果关系，而非存储扁平事实：

```
你 → THEORIZED → Agentic Commerce
           ↓ MANIFESTS_AS
         SubSelf / KangaBase / Kangas
```

- 6种实体类型：Person / Project / Theory / Concept / Goal / Event
- 13种关系类型：CREATED / THEORIZED / MANIFESTS_AS / CARES_ABOUT / WORKING_ON...
- WebUI：SVG Force-Layout 可视化，节点点击展开关联关系

### 🎯 目标情报监控

为每个目标设置搜索监控，定期追踪与目标相关的世界动态：

- 频率：每天 / 每周 / 每月
- 推送门槛：任何变化 / 显著变化 / 重大事件
- 主动推送而非被动等待——**成就你，而非服务你**

### 🔐 Skill 权限声明

每个 Skill 在激活前声明权限范围：

| 风险级别 | 标准 | 典型 Skill |
|---------|------|-----------|
| 🟢 低 | 无网络/无文件写 | 搜索、计算 |
| 🟡 中 | 需要网络或文件读写 | Web搜索、Git |
| 🔴 高 | system_exec / file_delete | 系统操作 |

用户在授权时就知道自己授权了什么，信任建立在透明之上。

### ⚙️ Skill 可配置参数

Skill 支持用户自定义参数（select / toggle / text / number），无需修改 Prompt 就能调整行为。用户覆盖默认值，系统持久化，一键重置。

### 🌱 Evolution 四阶段

| Phase | 名称 | 内容 |
|-------|------|------|
| 0 | 感知 | 记录所有交互 |
| 1 | 理解 | 对话后提取偏好 → user_model |
| 2 | 适应 | 每次对话前动态生成 System Prompt |
| 3 | 预测 | 工具执行后自动探索 → Skill 推荐 |

### 📊 生命坐标系

五轴雷达图，持续量化你与分身的共生状态：时间轴、存在轴、能力轴、情感轴、关系轴。

---

## WebUI 页面

| 页面 | 功能 |
|------|------|
| Chat | 对话主界面，TTS 语音输出 |
| Identity | 身份认知 + 传记记忆 + 关系图谱 |
| Life | 五轴生命坐标系雷达图 |
| Goals | 长期目标管理 + 情报监控 |
| Capabilities | 能力边界 + Skill 权限声明 |
| Memory | 记忆管理 + 反馈纠正 |
| Evolution | 进化阶段控制面板 |
| Scheduler | 定时任务管理 |
| Settings | 配置 + TTS + 主题 |

---

## 版本历程

| 版本 | 日期 | 核心里程碑 |
|------|------|---------|
| v1.0.0 | 2026-03-16 | 端到端打通，第一句话说出，154/154测试 |
| v1.1.0 | 2026-03-17 | 沙箱进程隔离 + capability_profile SQL VIEWs |
| v1.2.0 | 2026-03-17 | Evolution 四阶段全部打通 + DaXiang ChannelAdapter |
| v1.3.0 | 2026-03-18 | 传记记忆 + 动态System Prompt + 长期目标 GoalTracker |
| v1.4.0 | 2026-03-18 | 生命坐标系 /life 页面，五轴雷达图，257/257测试 |
| v1.5.0 | 2026-03-18 | TTS Kokoro 语音合成（按需安装），299/299测试 |
| v1.6.0 | 2026-03-18 | Lucide 图标全面统一，Memory Feedback 纠错面板 |
| **v1.7.0** | **2026-03-19** | **OpenFang借鉴五点：三层记忆+知识图谱+目标监控+Skill权限+可配置参数，362/362测试** |

---

## 相关资源

- 🔗 [GitHub 开源仓库](https://github.com/Fozu-lzwpattern/subself)
- 📖 [产品哲学文章：记住你·代表你·成就你](/research/)

---

*喵神 & 大仙 | 2026-03-19 | asC 数字分身原型* 🐱
