---
title: "OPC：让 OpenClaw 成为一人公司的 CEO"
parent: 专题研究
nav_order: 5
---

# OPC：让 OpenClaw 成为一人公司的 CEO

> One-Person Company — 从 PaperClip 的启发到 OpenClaw 的实现
>
> 2026-03-09 创建 · 2026-03-19 更新至 v5.2 | 喵神 & 大仙

---

## 一、这篇文章讲什么

我们做了一件事：**把 Multi-Agent 编排的组织管理方法论，提炼为一个 OpenClaw Skill，让 OpenClaw 能像 CEO 一样指挥调度多个 AI Agent 协作完成企业级复杂任务。**

我们把这个模式叫做 **OPC（One-Person Company）**——一个人 + 一个 AI CEO + 一群 AI 员工 = 一家公司的执行力。

本文记录完整的思考过程、设计决策、实现方案，以及在实战中踩坑后驱动的迭代历程（v1.0 → v5.2）。

---

## 二、起因：PaperClip 的启发

PaperClip 是一个开源的 Multi-Agent 编排平台，用"公司"的隐喻来组织 AI Agent——有组织架构、岗位职责、任务系统、预算管控。对它做了 76K 行源码的深度分析后，我们发现一个问题：

> PaperClip 是独立的重量级平台（PostgreSQL + Cloudflare Workers + Next.js），而我们已经有 OpenClaw 了。能不能把它最有价值的部分——组织管理方法论——提炼出来，轻量地注入 OpenClaw？

选择了**知识型 Skill** 方案：提炼方法论为 Markdown + 轻量脚本，零外部依赖，安装即用。

---

## 三、设计理念

### 3.1 一句话定义

```
用户（老板）→ OpenClaw CEO → Sub-agents（专业员工）
```

用户只说"我要做 X"，CEO 负责拆活儿、招人、盯进度、**亲自验收**、交结果。

### 3.2 五大原则

| 原则 | 含义 |
|------|------|
| **CEO 不干活，只管人** | Main Session 负责规划 / 分配 / 监控 / 验收 / 汇总 |
| **角色即能力边界** | 每个 Sub-agent 通过"角色卡"获得明确身份和预算 |
| **信任但验证** | 超时检查 + **CEO 独立验收** + 成本追踪 |
| **编排与执行分离** | OPC 管协作，业务 Skill 管操作 |
| **用户始终在回路** | 关键节点必须确认 |

### 3.3 与业务 Skill 的关系

```
    用户（老板）
        |
    OPC 编排层  ← CEO：拆任务、分角色、追进度、验收
        |
    业务执行层  ← 员工：gundam-ops / catclaw-search / ...
```

CEO 不需要懂高达怎么操作，搭建员不需要懂项目全局。**各司其职。**

---

## 四、核心机制

### 4.1 Phase 0：Context Intake

OPC 被触发后，**第一步必须理解背景，给出推进方案，等用户确认后才执行**。

**v3.1 新增：用户模型读取（追问用户之前先做）**

CEO 依次尝试读取以下文件（存在就读，不存在跳过）：

| 优先级 | 文件 | 说明 |
|--------|------|------|
| ① | workspace/opc-user-model.md | OPC 专属模型，越用越精准 |
| ② | ~/.openclaw/workspace/MEMORY.md | OpenClaw 官方长期记忆 |
| ③ | ~/.openclaw/workspace/USER.md | 用户扩展文件 |
| ④ | ~/.openclaw/workspace/memory/YYYY-MM-DD.md | 今日日志 |

标准化方案输出（用户确认后才 spawn）：

```markdown
## 📋 OPC 项目方案
背景理解：{一句话}
目标：{交付物 + 成功标准}
推进方案：
- 角色配置：{N 个角色}
- 协作模式：{串行 / 并行 / 混合}
- 预估预算：~{N}K tokens | 预计耗时：~{N} 分钟
确认后开始执行，是否有需要调整的地方？
```

### 4.2 项目状态持久化

**问题**：context compaction 会丢失项目执行状态。

**方案**：`project_state.py` 将全生命周期状态写入 `opc-projects/{id}/state.json`。

```bash
python3 engine/project_state.py init "315促销活动"
python3 engine/project_state.py agent-start <id> "builder" '{"role":"搭建员"}'
python3 engine/project_state.py agent-complete <id> "builder" '{"activityId":"1014813"}' 45000
python3 engine/project_state.py restore <id>   # compaction 后一键恢复
```

### 4.3 断点续传

Agent 失败后不完全重试——保存断点，新 Agent 从断点继续：

```
Agent 失败 → 提取已完成步骤 → 保存 checkpoint → spawn 新 Agent 注入断点
```

### 4.4 自动归因

`diagnose_agent.py` 规则引擎自动分类故障层级：

| 层级 | 来源 | 典型问题 |
|------|------|---------|
| L1 🔴 | 平台 | spawn 失败、announce 丢失 |
| L2 🟡 | 编排 | 角色定义不清、产出跑偏 |
| L3 🟠 | 业务 Skill | API 过期、浏览器交互异常 |
| L4 ⚪ | 外部 | 网络不通、超时 |

### 4.5 Persona Priming

给每个角色注入顶级人才的方法论，激活 LLM 预训练中的深层专业知识。预置库覆盖营销（Kotler / Ogilvy）、技术（Fowler / Jeff Dean）、研究（Drucker / Nate Silver）等方向。**借鉴方法论，不是模仿人格。**

**v3.2 内置 LZW 顾问 Persona**：OPC-Skill 原作者的完整方法论文档，LLM 训练数据里没有，所以需要显式注入。这也开创了一个新范式：**用户可以把自己的思维框架沉淀为 Persona，注入到自己的 Agent 团队中。**

### 4.6 Aware 触发器（v2.0）

声明式事件驱动——在 `triggers.yaml` 中定义规则，CEO 不再手动轮询：

| 类型 | 用途 |
|------|------|
| cron | 定时触发 |
| once | 一次性 deadline |
| interval | 周期检查 |
| on_message | Agent A 完成 → 自动启动 Agent B |

### 4.7 工具发现（v3.0）

domain × capability 双轴标签体系（12 × 8），OPC 自排除，精准推荐可用 Skill。

### 4.8 用户模型自进化（v3.1）

每次项目关闭，CEO 自动将配置、消耗、Persona 效果写回 `opc-user-model.md`。

**进化飞轮：项目越多 → 预判越准 → 用户越惊喜 → 做更多项目。**

### 4.9 CEO 自主验收（v4.0 新增）

**问题**：Sub-agent 说"完成了"不等于真的完成了。CEO 之前完全依赖自述，没有独立核实。

**方案**：每个 Agent 完成后，CEO 亲自跑验收命令：

```bash
# 工程类
python3 engine/project_state.py verify <pid> <label> "npm test"
# 研究类
python3 engine/project_state.py verify <pid> <label> "wc -l outputs/*.md"
```

验收 PASS → `agent-complete`；FAIL → 精准反馈 → 最多重试 3 次 → 告警用户。

### 4.10 USER_VOICE 透传（v4.0 新增）

**问题**：CEO 转述用户意图时，风格要求被稀释，结果产出"正确但没灵魂"。

**方案**：在每个角色卡中嵌入 `[USER_VOICE]` 字段，保留用户原话，优先级高于 CEO 的任何转述。

让用户的意图**直达每个员工，不经过任何过滤**。

### 4.11 并行文件边界声明（v4.0 新增）

**问题**：并行 Sub-agent 写同一批文件 → 后者覆盖前者 → 成果丢失。

**方案**：spawn 并行 Agent 时明确声明文件管辖范围（owned_paths）。越界修改须先向 CEO 汇报裁决。

### 4.12 冻结项目唤醒（v4.0 新增）

**问题**：项目中途中断后，`opc-projects/` 里留下冻结状态的项目，无人唤醒。

**方案**：Heartbeat 巡逻时自动扫描：

```bash
python3 engine/project_state.py wake-frozen
# 输出 frozen（<24h）和 stale（>24h）项目列表
```


### 4.13 触发分级矩阵（v5.0 新增）

**问题**：OPC 的触发条件是文字描述，模糊，主模型靠感觉判断，导致该触发时没触发、不该触发时硬触发。

**方案**：四级量化判断矩阵：

```
L0（不触发）：单一工具调用，预计 < 5 分钟
L1（轻量OPC）：2-3 步串行，省略 Phase 0 确认
L2（标准OPC）：触发任意 3 条——"完整"/"全链路"关键词 / 步骤 > 5 / 涉及 2+ 领域 / 文件 > 3 个 / token > 50K
L3（重型OPC）：触发任意 2 条——跨天 / 3+ 领域 / token > 200K / 复杂依赖链
```

**自适应指挥**：L1 省略方案确认直接 spawn，L2 方案卡限 5 行，L3 完整结构化。

### 4.14 角色模板库（v5.0 新增）

**问题**：CEO 每次临场写角色卡，质量不稳定。好的时候很好，差的时候像凑字数。

**方案**：`playbook/roles/` 目录，8 个标准角色模板，统一六要素结构：

| 角色 | 核心产出 | 典型 Persona |
|------|---------|-------------|
| researcher 研究员 | 调研报告、竞品分析 | Nate Silver / Mary Meeker |
| engineer 工程师 | 代码、测试 | Fowler / Jeff Dean |
| writer 写作者 | 文章、文档 | LZW / Paul Graham |
| analyst 数据分析师 | 数据报告、指标 | Nate Silver |
| integrator 整合员 | 多角色产出合并 | Drucker |
| ux-designer 体验设计师 | 交互方案、设计规格 | Jony Ive / Don Norman |
| biz-analyst 商业分析师 | 商业模型、战略建议 | Christensen / LZW |
| critic 评价与反馈 | 质量审查、改进建议 | 动态选择 |

**角色 × Persona 两个维度**：模板定义"做什么"，Persona 定义"怎么思考"，可自由组合。

`critic` 角色专门用于内容质量的专家评审，区别于机器 `verify`——前者审内容，后者跑命令。

### 4.15 Output Contract（v5.0 新增）

**问题**：Agent 输出什么格式、放哪个路径，全靠 CEO 临时口头交代，产出不一致，复盘时找不到文件。

**方案**：项目初始化时，CEO 声明产出协议：

```yaml
outputs:
  - label: research_report
    path: workspace/reports/{date}-{topic}.md
    min_lines: 80
    required_sections: ["核心发现", "参考来源"]
    required: true
  - label: code_package
    path: workspace/{repo}/
    test_cmd: "npm test"
    required: true
```

`verify` 命令对照 contract 执行——不再临场判断，有合同就按合同验。

### 4.16 Phase 4 交付包 + Phase 5 复盘（v5.0 新增）

**问题**：OPC 的交付只有"结果在 xxx 路径"，用户需要自己去拼凑，缺少"可以直接用"的感觉。项目结束后踩过的坑也随之消失，没有沉淀。

**Phase 4 交付包**——CEO 主动生成，不等用户追问：

```
📦 OPC 项目交付
━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ 核心结论（一段话，即使不读报告也能做决策）
📁 产出清单（路径 + 一句话描述）
💡 CEO 建议（下一步可以做什么）
⚠️ 注意事项（质量瑕疵 / 未完成项 / 需人工确认）
💰 消耗摘要（token + 耗时 + Agent 数量）
━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Phase 5 自动复盘**——`close` 命令触发，五步完成：
1. 预估 vs 实际消耗对比（偏差 > 50% 记录原因）
2. 验收失败模式 → 写入 `opc-user-model.md`
3. 产出质量自评（1-5分）
4. 一句话教训 → 写入 `playbook/scenarios/`
5. 全量项目记录更新到用户模型

**复盘飞轮：每个项目的踩坑都沉淀为系统知识，OPC 越用越强。**

### 4.17 任务拆解科学化（v5.1 新增）

**问题**：任务拆解粒度没有标准，并行/串行靠感觉，spawn 前的任务描述质量不稳定。

**四问框架**——CEO 拆出每个 Agent 前，必须回答：

```
Q1. 唯一产出是什么？（不能是"研究一下X"，要是具体文件路径）
Q2. 验收标准是什么？（可量化：行数/测试通过/章节存在）
Q3. 依赖什么输入？（上游产出/用户素材/无）
Q4. 文件边界是什么？（owned_paths）

四问全是"是" → spawn | 有"否" → 继续拆
```

**task-graph 命令**——依赖关系显式声明，自动识别可并行节点：

```bash
python3 engine/project_state.py task-graph <pid> add researcher_a "" reports/a.md
python3 engine/project_state.py task-graph <pid> add integrator "researcher_a" reports/final.md
python3 engine/project_state.py task-graph <pid> show
# 输出：可并行启动: ['researcher_a'] | 串行: ['integrator']
```

**并行判断规则**：三个条件全满足才并行（输入无依赖 + owned_paths 不重叠 + 产出不需互相参考）。

### 4.18 项目类型速启模板 + critic 铁律（v5.2 新增）

**问题**：每次项目都要从零想"用哪些角色、怎么组合、验收标准是什么"，认知负担重。
**原则**：不做重量级"专业模式"（会降低灵活性），做轻量"速启模板"——预设角色组合 + Output Contract + task-graph，继承后只需填参数。

**三个速启模板**（`playbook/scenarios/`）：

| 模板 | 触发词 | 角色组合 |
|------|--------|---------|
| research-project.yaml | 调研/分析/竞品 | researcher×N + integrator + **critic** |
| engineering-project.yaml | 开发/重构/MVP | engineer×N + (ux-designer) + **critic** |
| content-project.yaml | 文章/博客/文档 | (researcher) + writer + **critic** |

**critic 铁律（大仙提出）**：

> 无论什么项目类型，都必须包含至少一个 critic 角色。
> critic 在所有主产出完成后执行，评分 < 3 的产出打回重做，不进交付包。

verify（机器验收）和 critic（内容质量审查）是两道不同的关卡，缺一不可：

```
verify → 文件存在？命令通过？（机器）
critic → 内容质量够格？逻辑严密？（专家）

两道都通过 → Phase 4 交付包
```


---

## 五、实战验证

### 5.1 315 营销活动（v1.0，2026-03-09）

3 角色串行：策划 → 搭建 → 发布。67K tokens（$0.17）。踩坑：announce 不可靠 → 建立主动检查机制。

### 5.2 三会场并行搭建（v1.4，2026-03-10）

OPC 首次与 gundam-ops 联合做功。策划员×1 → 搭建员×3 并行。

| 会场 | 耗时 | 结果 |
|------|------|------|
| 主会场 | ~5min | ✅ |
| 外卖分会场 | ~7min | ✅ |
| 到餐分会场 | 17min+重试 | ✅（断点续传）|

暴露 4 个问题 → 驱动 v1.4。

### 5.3 KangaBase 从零到开源（v2.0，2026-03-11）

| Phase | 产出 | Token |
|-------|------|-------|
| 4 研究员并行调研 | 可行性报告 | 1.1M |
| 核心引擎 | 34文件/5270行/68测试通过 | 218K |
| 文档体系 | README+DESIGN+GUIDE | 63K |
| WebUI | 28文件/7页面 | 120K |
| **总计（8 Agent）** | **~100文件/8000行** | **~1.7M** |

**从白板灵感到 GitHub v0.1.0 发布：8 小时。**

### 5.4 Kangas2 嵌入式认知研究（v2.0，2026-03-12）

4 研究员并行（模型横评 / 认知架构 / LoRA / MiniMind）→ 汇总员。3 份报告共 2620 行，~320K tokens。

### 5.5 格式塔科技深度分析（v3.1，2026-03-14）

4 角色（Mary Meeker / Jeff Dean / Clayton Christensen / Peter Drucker Persona）并行研究后汇总。4500 字报告，综合评级 7.5/10，40K tokens（$0.10）。

### 5.6 SubSelf 持续迭代（v4.0，2026-03-18）

SubSelf 数字分身项目多轮迭代中，OPC 密集并行，暴露了 v3.x 的系统性缺陷并直接驱动 v4.0：

| 实战问题 | v4.0 解决方案 |
|---------|-------------|
| 三路并行写同一批文件，后者覆盖前者 | 4.11 文件所有权声明 |
| 报告风格丢失"有判断视角"的要求 | 4.10 USER_VOICE 透传 |
| 12 个历史 OPC 项目积压无人唤醒 | 4.12 wake-frozen 命令 |

单日运行 5 个 OPC 子项目，全部正常完成并提交。

---

## 六、OPC vs OpenClaw 原生 Multi-Agent

**原生机制 = 零件，OPC = 组装图纸。**

| 能力 | 原生 | OPC |
|------|------|-----|
| spawn sub-agent | ✅ | ✅ + 角色卡 + Persona |
| 任务分解 | ❌ | OKR → Epic → Task |
| 状态持久化 | ❌ | project_state.py |
| 断点续传 | ❌ | checkpoint 机制 |
| 自动归因 | ❌ | L1-L4 分层归因 |
| 事件驱动 | ❌ | Aware 触发器 + Focus |
| 工具发现 | ❌ | 标签体系 v2 |
| Context Intake | ❌ | Phase 0 强制确认 |
| 用户模型进化 | ❌ | opc-user-model.md 自学习 |
| CEO 自主验收 | ❌ | **v4.0 verify 命令** |
| 意图原话透传 | ❌ | **v4.0 USER_VOICE** |
| 并行文件隔离 | ❌ | **v4.0 owned_paths** |
| 冻结项目唤醒 | ❌ | **v4.0 wake-frozen** |
| 触发分级判断 | ❌ | **v5.0 L0-L3 矩阵** |
| 角色模板库 | ❌ | **v5.0 8个标准角色** |
| 产出协议 | ❌ | **v5.0 Output Contract** |
| 结构化交付包 | ❌ | **v5.0 Phase 4** |
| 自动复盘飞轮 | ❌ | **v5.0 Phase 5** |
| 任务依赖图声明 | ❌ | **v5.1 task-graph** |
| 拆解四问框架 | ❌ | **v5.1 spawn 强制自检** |
| 速启模板 | ❌ | **v5.2 research/engineering/content** |
| critic 铁律 | ❌ | **v5.2 任何项目必须有质量审查** |

---

## 七、设计哲学

> **OPC 不是让 AI 更聪明，而是让 AI 更有组织。**

```
个人 = 一个聪明人干所有事（有上限）
公司 = 很多人各干一件事（上限消失）

单 Agent = 一个强模型处理所有 context
OPC    = 多个 Agent 各处理一小块（上限消失）
```

**OPC 就是 AI Agent 的公司制度。** v4.0 补上了这套制度缺失的最后一环：**质量管控**。CEO 不能只靠员工自报，必须亲自验货。

---

## 八、下载与使用

### Skill 包下载

| 版本 | 下载 | 核心变更 |
|------|------|---------|
| **v5.2**（最新） | [agent-orchestration-v5.2.tar.gz](/assets/skills/agent-orchestration-v5.2.tar.gz) | 速启模板×3 + critic铁律 + 任务拆解四问 + task-graph |
| v5.0 | [agent-orchestration-v5.0.tar.gz](/assets/skills/agent-orchestration-v5.0.tar.gz) | 角色模板库 + 触发矩阵 + Output Contract + 交付包 + 复盘 |
| v4.0 | [agent-orchestration-v4.0.tar.gz](/assets/skills/agent-orchestration-v4.0.tar.gz) | CEO 验收 + USER_VOICE + 文件边界 + 冻结唤醒 |
| v3.2 | [agent-orchestration-v3.2.tar.gz](/assets/skills/agent-orchestration-v3.2.tar.gz) | 内置 LZW 顾问 Persona |
| v3.1 | [agent-orchestration-v3.1.tar.gz](/assets/skills/agent-orchestration-v3.1.tar.gz) | 用户模型自学习 + 三层架构 |
| v3.0 | [agent-orchestration-v3.0.tar.gz](/assets/skills/agent-orchestration-v3.0.tar.gz) | 三层架构 + Context Intake |
| v2.0 | [agent-orchestration-v2.0.tar.gz](/assets/skills/agent-orchestration-v2.0.tar.gz) | Aware 触发器 + 工具自发现 |
| v1.4 | [agent-orchestration-v1.4.tar.gz](/assets/skills/agent-orchestration-v1.4.tar.gz) | 状态持久化 + 断点续传 + 自动归因 |

### 安装

```bash
cd ~/.openclaw/skills
tar xzf agent-orchestration-v5.2.tar.gz
```

对 OpenClaw 说"帮我做一个完整的 XX 项目"，会自动触发 OPC 编排流程。

**源码**：[GitHub — Fozu-lzwpattern/OPC-agent-orchestration](https://github.com/Fozu-lzwpattern/OPC-agent-orchestration)

### v4.0 文件结构

```
agent-orchestration-20260309-lzw/
├── SKILL.md              ← 入口（v5.2，含触发分级矩阵+速启模板引用）
├── CHANGELOG.md
├── brain/                ← CEO 决策层
│   ├── core-flow.md      ← Phase 4 交付包 + Phase 5 复盘
│   ├── verification.md   ← CEO 自主验收规则库
│   ├── task-decomposition.md
│   ├── role-design.md
│   ├── collaboration-patterns.md
│   └── cost-tracking.md
├── engine/
│   ├── project_state.py  ← +verify, +wake-frozen, +close(retrospective)
│   ├── trigger_engine.py
│   ├── tool_discovery.py
│   └── diagnose_agent.py
└── playbook/
    ├── personas/lzw.md
    ├── roles/            ← ⭐ v5.0 角色模板库（8个）
    │   ├── researcher.md / engineer.md / writer.md / analyst.md
    │   ├── integrator.md / ux-designer.md / biz-analyst.md
    │   └── critic.md
    ├── scenarios/            ← ⭐ v5.2 速启模板
    │   ├── research-project.yaml
    │   ├── engineering-project.yaml
    │   └── content-project.yaml
    └── templates/
        ├── agent-prompt.md       ← v5.1 标准六要素格式
        ├── output-contract.yaml  ← v5.0 产出协议模板
        └── opc-user-model.md
```

---

## 九、版本历程

| 版本 | 日期 | 核心变更 | 驱动因素 |
|------|------|---------|---------|
| v1.0 | 2026-03-09 | 首版：角色卡 + 任务分解 + 协作模式 + 成本追踪 | PaperClip 方法论提炼 |
| v1.3 | 2026-03-09 | Persona Priming + CEO 主动监控 | 联调暴露 announce 不可靠 |
| v1.4 | 2026-03-11 | 状态持久化 + 断点续传 + 自动归因 + 成本自动化 | 三会场实战 |
| v2.0 | 2026-03-12 | Aware 触发器 + 工具自发现 + Focus 焦点管理 | KangaBase 实战 |
| v3.0 | 2026-03-14 | 三层架构 + Context Intake + 工具发现标签体系 v2 | SKILL.md 过重 |
| v3.1 | 2026-03-14 | 用户模型自学习：Phase 0 读取 + Phase 4 写回 | 越用越好用 |
| v3.2 | 2026-03-16 | 内置 LZW 顾问 Persona | 作者思维框架可被 Agent 借鉴 |
| v4.0 | 2026-03-19 | CEO 验收 + USER_VOICE + 文件边界 + 冻结唤醒 + 阶段成本推送 | SubSelf 迭代暴露五大系统性缺陷 |
| v5.0 | 2026-03-19 | 角色模板库×8 + 触发分级矩阵 + Output Contract + Phase4交付包 + Phase5复盘 | 大仙系统性反馈 |
| v5.1 | 2026-03-19 | 任务拆解科学化（四问框架+task-graph+并行规则+标准描述） | 任务拆解粒度/依赖/质量三个根本性问题 |
| **v5.2** | **2026-03-19** | **速启模板×3 + critic铁律（任何项目必须有质量审查）** | **大仙：任何项目都该有评价和反馈角色** |

---

## 十、后续方向

1. **`init --type` 参数**：命令行直接继承速启模板，零配置启动
2. **更多速启模板**：product-analysis / data-analysis（按实战积累添加）
3. **Agent 记忆继承**：专业 Agent 积累领域知识，跨项目复用
4. **复盘驱动优化**：Phase 5 积累足够数据后，自动推荐角色配置改进
5. **critic 评分历史分析**：识别哪类任务描述容易触发低分，自动优化模板

---

*喵神 & 大仙 | 2026-03-09 创建 · 2026-03-19 更新至 v5.2 | OPC — AI Agent 的公司制度*
