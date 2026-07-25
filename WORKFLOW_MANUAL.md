# agents-workflow — 完整技术手册 v4.1.10

> 版本：4.1.8 · 最后更新：2026-07-24

---

## 目录

1. [架构概览](#1-架构概览)
2. [双引擎 + 双模式设计](#2-双引擎--双模式设计)
3. [完整执行流程](#3-完整执行流程)
4. [硬门控机制](#4-硬门控机制)
5. [双模式自动切换](#5-双模式自动切换)
6. [进度跟踪与断点续跑](#6-进度跟踪与断点续跑)
7. [原始需求交叉验证](#7-原始需求交叉验证)
8. [知识沉淀](#8-知识沉淀)
9. [子Agent调度](#9-子agent调度)
10. [MCP 多模态集成](#10-mcp-多模态集成)
11. [技能目录](#11-技能目录)
12. [v4.x 版本演进](#12-v4x-版本演进)

---

## 1. 架构概览

### 1.1 一句话描述

用户输入一句话需求，编排器自动串起从需求澄清到版本登记的全流程。**自动检测平台能力**，task 可用时派发子Agent并行执行，不可用时编排器直接执行——同一套技能，所有平台通吃。

### 1.2 整体架构

```
用户: /wf-orchestrator MediaManager: 新增批量导出

编排器 (wf-orchestrator) — 主会话
  ├── 自动检测: task 工具是否可用？
  ├── 解析参数 + 创建会话目录
  ├── 交互式需求澄清（与用户一问一答）
  ├── 产出: task_plan.md + specs/*.md
  └── 分支:
        ├── task 可用 → task(wf-orchestrator-engine) 独立上下文执行
        └── task 不可用 → 编排器直接执行 12 项任务（inline 模式）

执行引擎 (wf-orchestrator-engine) — 独立上下文（仅正常模式）
  ├── 接收 JSON 包裹
  ├── 启动自检（显示项目/模式/范围）
  ├── 12项任务依次执行
  └── 返回 JSON 摘要给编排器
```

### 1.3 v4.x 核心设计决策

| 设计决策 | 原因 |
|:---------|:-----|
| 双引擎拆分 | 编排器交互产生大量上下文，引擎独立会话不受干扰 |
| 双模式自动切换 | 同一套技能覆盖所有平台，无需维护多版本 |
| 任务清单替代阶段编号 | 消除计数矛盾。AI不数"第几个"，只做"下一项" |
| 硬门控 + 回退上限3次 | 架构问题必须修，但不能无限循环 |
| fallback兜底 | task()不可用时编排器直执行，工作流不中断 |
| 原文交叉验证 | 每层子Agent对照用户确认的specs校验，防需求漂移 |

---

## 2. 双引擎 + 双模式设计

### 2.1 为什么拆分（v3.x -> v4.0）

v3.x 单引擎存在三个问题：

1. **上下文膨胀** — 规划阶段50轮问答后，执行阶段还要背这段历史
2. **子Agent混乱** — 子Agent可能翻到前面对话，误解当前任务
3. **恢复困难** — 中断后要从头回溯整个对话历史

### 2.2 引擎1：编排器 (wf-orchestrator)

四步操作：

- **步骤1** — 参数解析、创建会话目录、**自动检测 task 可用性**、检测中断恢复
- **步骤2** — 交互式需求澄清（brainstorming逐轮问答），产出 task_plan.md + specs
- **步骤3** — 根据 task 可用性分支：
  - **正常模式**：打包JSON → task(wf-orchestrator-engine)
  - **inline模式**：编排器直接执行 12 项任务
- **步骤4** — 接收结果，展示报告

### 2.3 引擎2：执行引擎 (wf-orchestrator-engine)

12项任务清单（正常模式由独立 subagent 执行，inline 模式由编排器直接执行）：

1. 领域建模  2. 架构设计  3. 架构评审(硬门控)  4. 编码
5. 测试  6. 架构扫描(硬门控)  7. 审查  8. 部署
9. 文档检查  10. 分支收尾  11. 版本登记  12. 总结+知识沉淀

启动时自检输出关键信息（项目/模式/范围）。

### 2.4 v4.0 -> v4.1 主要演进

| | v4.0 | v4.1 |
|:--|:-----|:-----|
| 描述格式 | "阶段 4/13：架构评审" | ">>> 架构评审" |
| 计数方式 | 编号（常出错） | 任务清单（不数数） |
| 评分标准 | 35/60 | 18/24（统一） |
| --to | 声明未实现 | 每个任务检查 |
| --parallel | 声明未实现 | 任务4/5并行调度 |
| --timeout | 硬编码300 | 参数化 |
| 回退上限 | 无 | 最多3次 |
| fallback | 硬编码false | 自动检测 + 失败2次自动降级 |
| 平台适配 | 单版本 | **双模式自动切换** |

---

## 3. 完整执行流程

### 3.1 正常模式（task 工具可用时）

编排器完成交互式规划后，将 JSON 包裹传给执行引擎，引擎在独立上下文中按清单执行：

```
启动自检 -> 显示项目/模式/范围 -> 逐项执行
```

**任务1: 领域建模** — 引擎直执行
- 读取 specs + task_plan
- 注入 domain-modeling 方法论
- 提取实体、定义关系、建立通用语言
- 产出: domain-model.md
- 交叉验证: 概念是否与specs一致

**任务2: 架构设计** — task(wf-architect) Pro模型
- 传入 specs + domain-model + task_plan + 项目根目录
- 如为回退重试，注入上次评审意见
- 产出: 02-design.md

**任务3: 架构评审(硬门控)** — 引擎直执行
- 注入 arch-review，6维评分(满分24)
- Go(>=18): 产出 03-arch-review.md，进任务4
- No-Go(<18): 回退任务2，最多3次，超限强制Go

**任务4: 编码** — task(wf-developer) Flash模型
- 启动前验证: 03-arch-review.md 必须存在且为Go
- 传入 specs + design + task_plan + 项目根目录
- --parallel时同时启动任务5
- 注入 TDD/系统化调试/UI设计(按需)
- 产出: 代码 + CHANGES.md

**任务5: 测试** — task(wf-tester) Flash模型
- --parallel时等待已在任务4启动的测试完成
- 注入 dispatching-parallel-agents + verification
- 产出: TEST_REPORT.md

**任务6: 架构扫描(硬门控)** — 引擎直执行
- 扫描代码结构，检测循环依赖、模块耦合
- 无严重问题: 产出 architecture-scan.html，进任务7
- 有严重问题: 回退任务4，最多3次

**任务7: 审查** — task(wf-reviewer) Pro模型
- 启动前验证: architecture-scan.html 必须存在
- 注入代码审查 + 中文审查 + 反馈处理
- 产出: 05-review.md

**任务8: 部署** — task(wf-deployer) Flash模型
- 检测项目类型，执行构建命令
- 产出: DEPLOY.md

**任务9: 文档检查** — 引擎直执行
- 注入 chinese-documentation
- 检查排版、术语、标点
- OCR处理图片型文档
- 产出: doc-check-report.md

**任务10: 分支收尾** — 引擎直执行
- git add + chinese-commit-conventions 格式提交
- 询问是否推送/创建PR
- 注入 finishing-a-development-branch + git-worktrees

**任务11: 版本登记** — 引擎直执行
- 注入 vt，更新 .vt.json

**任务12: 生成总结** — 引擎直执行
- 读取所有产出摘要，写入 SUMMARY.md
- 知识沉淀: 提炼关键发现写入 KNOWLEDGE.md
- 返回 JSON 摘要给编排器

### 3.2 inline/兜底模式（task 不可用时）

编排器检测到 task 不可用后，自动切换为 inline 模式：

1. 输出：「⚠️ 兜底模式 — task 不可用，编排器直接执行所有阶段」
2. 编排器**直接执行**上述 12 项任务（参考 wf-orchestrator-engine 流程）
3. 每个任务完成后更新 progress.md + findings.md + checkpoint
4. 子Agent任务（任务2/4/5/7/8）由编排器代为完成
5. 功能完整，仅性能略低于正常模式（无法并行派发子Agent）

### 3.3 硬门控介入

任务3（架构评审）No-Go -> 回退任务2（最多3次）
任务6（架构扫描）严重问题 -> 回退任务4（最多3次）
任务4启动前检查03-arch-review.md必须Go
任务7启动前检查architecture-scan.html必须存在

### 3.4 --lite 轻量模式

跳过任务3（架构评审）+ 任务6（架构扫描），适合小型修复

### 3.5 --parallel 并行

任务4/5同时启动 wf-developer + wf-tester（仅正常模式，inline 模式串行执行）

### 3.6 各任务产出

| 任务 | 产出文件 | 执行者 |
|:-----|:--------|:------|
| 1 领域建模 | domain-model.md | 引擎 |
| 2 架构设计 | 02-design.md | wf-architect |
| 3 架构评审 | 03-arch-review.md | 引擎 |
| 4 编码 | 代码 + CHANGES.md | wf-developer |
| 5 测试 | TEST_REPORT.md | wf-tester |
| 6 架构扫描 | architecture-scan.html | 引擎 |
| 7 审查 | 05-review.md | wf-reviewer |
| 8 部署 | DEPLOY.md | wf-deployer |
| 9 文档检查 | doc-check-report.md | 引擎 |
| 10 分支收尾 | git commit + PR | 引擎 |
| 11 版本登记 | .vt.json | 引擎 |
| 12 总结 | SUMMARY.md | 引擎 |

---

## 4. 硬门控机制

### 4.1 架构评审门控（任务3）

6维评分，每维1-4分，满分24。Go: >=18/24 (75%)

| 维度 | 说明 |
|:-----|:-----|
| 功能完整性 | 是否覆盖所有需求点 |
| 可扩展性 | 架构是否支持未来扩展 |
| 性能 | 是否有性能考量 |
| 安全性 | 安全设计 |
| 可维护性 | 清晰、可测试 |
| 成本 | 技术选型合理 |

No-Go时回退任务2（最多3次，超限强制Go+记录风险）

### 4.2 架构扫描门控（任务6）

检测循环依赖、模块耦合、架构腐化。严重问题回退任务4（最多3次）

### 4.3 双重验证

- 任务4启动前: 必须03-arch-review.md存在且为Go
- 任务7启动前: 必须architecture-scan.html存在
- 不满足则回退到对应修复任务

---

## 5. 双模式自动切换

### 设计理念

同一套技能，适配所有 AI 工具。有 `task()` 子 Agent 能力的工具走正常模式（并行、模型隔离），没有的自动降级 inline 模式（编排器自己干）。用户无需感知差异，说同一句话即可。

### 触发条件

1. 编排器启动时自动检测 task 工具可用性
2. task() 连续失败 2 次自动降级

### 架构对比

```
正常模式（task 可用）:
┌──────────────────────┐
│  编排器 (主会话)       │  与用户交互、规划
│  → 产出 specs + plan  │
└──────┬───────────────┘
       │ task(wf-orchestrator-engine)
       ▼
┌──────────────────────┐
│  引擎 subagent        │  独立会话，干净上下文
│  ┌─ 任务1: 领域建模  │  引擎直执行
│  ├─ 任务2: 架构设计  │  → task(wf-architect)   独立会话
│  ├─ 任务3: 架构评审  │  引擎直执行（硬门控）
│  ├─ 任务4: 编码      │  → task(wf-developer)   独立会话
│  ├─ 任务5: 测试      │  → task(wf-tester)      独立会话
│  ├─ 任务6: 架构扫描  │  引擎直执行（硬门控）
│  ├─ 任务7: 审查      │  → task(wf-reviewer)    独立会话
│  ├─ 任务8: 部署      │  → task(wf-deployer)    独立会话
│  ├─ 任务9~12: ...    │  引擎直执行
│  └─ 返回摘要         │
└──────────────────────┘

inline 模式（task 不可用）:
┌──────────────────────────────────┐
│  编排器 (主会话) — 一镜到底       │
│                                  │
│  规划 → 产出 specs + plan        │
│  ┌─ 任务1: 领域建模              │  注入 domain-modeling 技能，自己做
│  ├─ 任务2: 架构设计              │  注入 wf-architect 技能指引，自己做
│  ├─ 任务3: 架构评审（硬门控）    │  注入 arch-review 技能，自己做
│  ├─ 任务4: 编码                  │  注入 TDD/调试 技能，自己写代码
│  ├─ 任务5: 测试                  │  注入 verification 技能，自己跑测试
│  ├─ 任务6: 架构扫描（硬门控）    │  注入 arch-review 用法二，自己做
│  ├─ 任务7: 审查                  │  注入 三合一审查 技能，自己做
│  ├─ 任务8: 部署                  │  注入 wf-deployer 技能指引，自己做
│  ├─ 任务9~12: 文档/分支/版本/总结│  注入对应技能，自己做
│  └─ 展示结果                     │
└──────────────────────────────────┘
```

### 子 Agent 任务的 inline 等价实现

正常模式通过 `task()` 派发子 Agent 到独立会话；inline 模式下编排器注入对应的技能指引，亲自完成：

| 任务 | 正常模式 | inline 模式（编排器怎么做） |
|------|---------|--------------------------|
| 架构设计 | `task(wf-architect)` | 读取 wf-architect 技能指引，以架构师身份产出 `02-design.md` |
| 编码 | `task(wf-developer)` | 读取 wf-developer 技能指引，以开发者身份写代码 |
| 测试 | `task(wf-tester)` | 读取 wf-tester 技能指引，以测试者身份跑测试、写报告 |
| 审查 | `task(wf-reviewer)` | 读取 wf-reviewer 技能指引，以审查者身份审查代码 |
| 部署 | `task(wf-deployer)` | 读取 wf-deployer 技能指引，以部署者身份提供部署方案 |

**不是跳过任务**，而是换了一种方式完成——编排器注入技能指引后扮演对应角色。

### 模式差异对照

| 维度 | 正常模式 | inline 模式 |
|:-----|:--------|:-----------|
| 触发条件 | task 工具可用 | task 不可用 / 连续失败 2 次 |
| 引擎执行 | `task(wf-orchestrator-engine)` 独立 subagent | 编排器直接执行 12 项任务 |
| 子 Agent 派发 | `task()` 隔离上下文，互不污染 | 编排器注入技能指引，亲自完成 |
| 上下文隔离 | ✅ 每个子 Agent 有独立会话 | ❌ 所有任务共享同一会话 |
| 并行执行 | ✅ 任务 4+5 可同时启动 | ❌ 必须串行（同一会话无法并行） |
| 模型隔离 | ✅ 架构用 Pro，编码用 Flash | ❌ 所有任务用同一个模型 |
| 硬门控 | ✅ 正常执行 | ✅ 正常执行（逻辑不变） |
| 交叉验证 | ✅ 对照 specs 校验 | ✅ 对照 specs 校验 |
| 进度跟踪 | ✅ progress + checkpoint | ✅ progress + checkpoint |
| 断点恢复 | ✅ --from/--to 支持 | ✅ --from/--to 支持 |
| 知识沉淀 | ✅ KNOWLEDGE.md | ✅ KNOWLEDGE.md |
| 功能完整性 | ✅ 12 项任务全覆盖 | ✅ 12 项任务全覆盖 |
| 执行速度 | ⚡ 快（并行 + 模型匹配） | 🐢 较慢（串行 + 单模型） |
| 质量 | ⭐⭐⭐ 最高 | ⭐⭐ 略低（上下文膨胀影响） |

### 启动告知

```
正常: ✅ 正常模式 — 通过 task 派发子Agent
兜底: ⚠️ 兜底模式 — task 不可用，编排器直接执行所有阶段
```

---

## 6. 进度跟踪与断点续跑

三文件: task_plan.md + progress.md + findings.md
加 checkpoint.json 记录断点

--from 恢复时验证前置文件存在
--to 命中跳任务12
阶段名映射: planning > domain_modeling > architecture > arch_review > coding > testing > arch_scan > review > deploy > doc_check > branch_finish > version_tracking

progress标签映射表确保中文标签与英文checkpoint值对应

---

## 7. 原始需求交叉验证

每个子Agent收到两份输入: 前序衍生文档(承上) + 用户确认的specs(校验)
偏离时显式标注。防需求层层传递中漂移

---

## 8. 防幻觉体系

工作流构建了 6 层嵌套的防幻觉机制，防止 AI 虚构进度、跳过关键步骤、或产出与需求不符：

### 8.1 全局层 — AGENTS.md 进度检查规则

每个项目根目录的 `AGENTS.md` 声明硬性规则：**任何操作开始之前，必须先检查 `docs/superpowers/plans/task_plan.md` 是否存在**，读取当前进度。覆盖所有对话和场景，不论是否使用编排器。防止 Agent 跨会话丢失上下文、在错误断点处虚构进度。

### 8.2 编排器层 — SSOT + 产出验证 + 进度兜底

| 机制 | 说明 |
|------|------|
| **中断恢复** | 扫描 `workflow/` 下未完成的 checkpoint，询问用户恢复/新建/放弃 |
| **用户批准门控** | 规划产出 specs + task_plan，用户确认后才执行 |
| **SSOT** | `specs/*.md` 声明为原始需求基准，后续所有阶段强制对照 |
| **产出物验证** | 每个子Agent完成后验证文件已生成，不存在则记录失败 |
| **进度兜底** | 子Agent执行完后编排器独立检查 progress.md，漏更新则补写 |

### 8.3 执行引擎层 — 自检 + 前置验证 + 进度注入

| 机制 | 说明 |
|------|------|
| **启动自检** | 执行前输出项目/模式/范围，连续 2 次 task 失败自动降级 |
| **from_phase 验证** | 从中间阶段恢复时检查 specs、task_plan 等上游文件存在 |
| **进度文件注入** | 每次 task() 前注入 task_plan + progress + findings 到子Agent prompt |

### 8.4 硬门控层 — 两道关口，反复回退

两道门控（架构评审、架构扫描）都是可回退的硬门控，形成闭环验证：

```
架构设计 → 架构评审（No-Go → 回退设计，最多3次）
编码 → 测试 → 架构扫描（有问题 → 回退编码，最多3次）
```

每道门控启动前有前置验证：编码前检查评审通过，审查前检查扫描完成。

### 8.5 交叉验证层 — 5个子Agent全部对照原始需求

所有子Agent（架构/开发/测试/审查/部署）自动读取 `specs/*-design.md` 对照。每层职责：

| 阶段 | 验证目标 |
|------|---------|
| 领域建模 | 提取的概念是否与 specs 一致，有无遗漏 |
| 架构设计 | 设计方案是否满足 specs 中的需求 |
| 编码 | 实现是否覆盖 specs 中的 P0/P1 需求 |
| 测试 | 测试范围是否对应 specs 需求 |
| 审查 | 最终代码与原始 specs 是否一致 |
| 部署 | 交付物是否满足原始需求 |

### 8.6 验证铁律层 — `verification-before-completion`

被引擎任务5（测试阶段）强制引用注入的核心规则：

- **铁律**：没有新鲜验证证据，不许宣称完成。必须在此会话中运行过验证命令。
- **门控 5 步**：确定证明命令 → 运行完整命令 → 检查退出码 → 验证输出 → 才能做结论
- **红线清单**：使用"应该""大概""似乎"等模糊措辞 = 尚未验证
- **防止合理化**："有信心"≠"已验证"，"代理说成功了"需独立验证

### 8.7 技能自检清单

关键技能内建自检：领域建模 7 项自查（概念有无遗漏？关系标注？业务规则？）、架构评审 7 项自查（专家视角？维度评分？依赖图？Go/No-Go？）。

---

## 9. 知识沉淀

每次会话结束提炼 -> 写入 KNOWLEDGE.md -> 下次启动自动注入
格式: 关键决策/经验教训/领域知识/注意事项/数据洞察

---

## 10. 子Agent调度

5个task子Agent: wf-architect/wf-developer/wf-tester/wf-reviewer/wf-deployer
Pro模型(架构/审查) Flash模型(编码/测试/部署)
调用格式: task(subagent_name, prompt, model, timeout)
产物验证: 1重试2标记失败3继续
inline模式下由编排器直接执行，无模型隔离

---

## 11. MCP 多模态集成

### 10.1 当前配置

```toml
[[plugins]]
name    = "mimo-multimodal"
command = "npx"
args    = ["-y", "tonyq-mimo-mcp-server"]
env     = {
  MIMO_API_URL  = "https://api.xiaomimimo.com/v1/chat/completions",
  MIMO_API_KEY  = "${MI*******KEY}"
}
call_timeout_seconds = 600
```

**MiMo 工具清单（7个）：**

| MCP 工具名 | 对应技能 | 用途 |
|:-----------|:---------|:-----|
| `mimo_understand_image` | mimo-image-understanding | 图片分析/OCR/UI审查 |
| `mimo_understand_audio` | mimo-audio-understanding | 音频转写/理解 |
| `mimo_understand_video` | mimo-video-understanding | 视频内容分析 |
| `mimo_speech_recognition` | — | 语音识别(ASR) |
| `mimo_speech_synthesis` | mimo-tts | 预置音色语音合成 |
| `mimo_voice_design` | mimo-tts | 文本描述自定义音色 |
| `mimo_voice_clone` | mimo-tts | 音频样本克隆音色 |

### 10.2 v4.1.2 迁移说明

**为什么从 `mimo-mcp-server` 换成 `tonyq-mimo-mcp-server`？**

原 `mimo-mcp-server` 由第三方 alionsss 发布。npm 发布版（v0.1.2）的工具名加了 `mimo_` 前缀（如 `mimo_understand_image`），但 GitHub 源码未同步，仍然用旧名（`understand_image`）。导致 npx 拉到的版本和缓存文件不一致，MCP 连接断开。

解决方案：fork 到 `TonyQ-AI/mimo-mcp-server`，修复源码对齐 npm 版，以 `tonyq-mimo-mcp-server` 名称发布到 npm，完全脱离对 alionsss 的依赖。

**为什么 `MIMO_API_BASE` 改成 `MIMO_API_URL`？**

MCP Server 源码通过 `process.env.MIMO_API_URL` 读取 API 地址，默认值是 Token Plan 端点（`token-plan-cn.xiaomimimo.com`）。如果用标准 API key 但配的是 `MIMO_API_BASE`，服务器读不到自定义地址，默认走 Token Plan 端点导致 401。

**为什么工具名加了 `mimo_` 前缀？**

npm 发布版 v0.1.2 将所有工具名加了 `mimo_` 前缀（`understand_image` → `mimo_understand_image`），但缓存文件仍用旧名。调用时名字不匹配，MCP Server 返回 `read: EOF`。修复方式：
1. 更新缓存文件中 7 个工具名匹配新名
2. 重新发布 npm 包时保持工具名一致

**为什么配置文件不能直接写 key？**

`redact_tool_output` 安全机制会自动将显示中的密钥替换为 `*`。如果直接复制显示中的 key（带星号）写入配置文件，实际传入的是带星号的无效 key。必须通过环境变量引用（`${MI*******KEY}`）或使用脚本从 `.env` 文件读取写入。

---

## 12. 技能目录

40技能: 编排2 + 子Agent5 + 参考1 + 架构4 + 方法7 + 审查4 + 提交5 + 辅助3 + MCP5 + 扩展4

---

## 13. v4.x 版本演进

### v4.1.10 (2026-07-24) — 包管理器优化

- 包管理器优先顺序：pnpm → yarn → npm，Python: uv → poetry → pip

### v4.1.9 (2026-07-24) — 知识沉淀加固 + 引擎瘦身

- 知识沉淀移至编排器层，防止引擎预算耗尽被跳过
- 知识沉淀路径改为相对路径，防止 subagent 写权限不足
- 进度更新先 complete_step 再 todo_write，防止 todo 冲突
- 引擎任务12只生成 SUMMARY.md，去掉 17 行知识沉淀代码

### v4.1.8 (2026-07-24) — 进度跟踪修复 + 多工具兼容

- 子任务完成检查：标记阶段完成前扫描 task_plan.md，未完成禁止推进
- 串行任务修复：防止多个 in_progress 冲突
- 多工具兼容：todo_write 同步（Reasonix）+ 编辑 markdown 降级（其他工具）

### v4.1.7 (2026-07-13) — 升级迁移体验

- 升级流程新增旧版检测（reasonix-workflow/workflow-task），展示迁移说明交互
- 升级时自动清理旧版残留目录
- 升级报告改为迁移风格，突出新版本变化

### v4.1.6 (2026-07-13) — 安装体验优化

- 安装器智能检测本地 MiMo key，已有则询问直接使用/重新输入/跳过
- 安装器去掉 DeepSeek key 询问（推理 API 已在工具中配置）
- 安装器 MiMo key 改为强制交互，确保不跳过配置
- 安装器文档中性化，移除 reasonix.toml 引用

### v4.1.5 (2026-07-13) — 架构评审修复

- 修复 `--parallel` 和 `--timeout` 参数未传入引擎 JSON 的 bug
- 修复 engine progress.md 标签映射表 emoji 乱码
- 修复并行模式下 tester 读取 CHANGES.md 的竞态问题
- 删除 engine 重复身份描述段落
- deployer 任务补全上游产出文件路径传入
- arch-review 产出文件名对齐引擎 `03-arch-review.md`
- 全面清理 Reasonix 品牌名，实现品牌中性化

### v4.1.4 (2026-07-13) — 引擎修复

- 修复 `wf-orchestrator-engine/SKILL.md` YAML 前页缺少闭合 `---` 分隔符
- 补齐缺失文档：WORKFLOW_MANUAL.md、INSTALL.md、LICENSE、.gitignore
- workflow-installer 仓库 URL 修正为 Workflow-Dualmode

### v4.1.3 (2026-07-13) — 全平台适配 · 双模式发布

- **Dualmode 正式发布**：合并 workflow-task（task依赖）和 workflow-inline（无task依赖），自动检测平台能力
- MCP 独立发布：`tonyq-mimo-mcp-server` v0.2.2，默认 Standard API
- 安装器自动检测 MiMo 端点类型
- 技能目录清理：移除 reasonix- 前缀，路径中性化

### v4.1.2 (2026-07-13) — MCP 独立发布

- 自建 npm 包 `tonyq-mimo-mcp-server`，脱离对 alionsss 的依赖
- `MIMO_API_BASE` → `MIMO_API_URL`，URL 补全 `/chat/completions`
- bin 名统一为 `tonyq-mimo-mcp-server`，npx 可直接找到入口
- 配置文件禁止硬编码 key，改用环境变量引用

### v4.0 (2026-07-12) — 双引擎架构

- 编排器+引擎拆分
- 硬门控(Go/No-Go + 扫描回退)
- 启动自检
- task_plan.md统一基准

### v3.x (2026-07-07~12) — 13阶段扩展

- 8->13阶段、交互澄清、领域建模、交叉验证、知识沉淀

### v2.x (2026-07-01~06) — 基础流水线

- 8阶段、TDD/调试、checkpoint追踪

---

> 版本: 4.1.10 · 最后更新: 2026-07-24
