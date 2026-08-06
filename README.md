# agents-workflow

> 多Agent协同开发工作流 · 双模式自适应 · 40 个技能 · 全平台可用

[![Version](https://img.shields.io/badge/version-4.1.11-blue)](VERSION)
[![Skills](https://img.shields.io/badge/skills-40-green)](skills/)
## 注意
本工作流一开始出现的原因是reasonix早期版本原生能力尚有欠缺，遂开发工作流用以增强其能力，现随着reasonix和其他工具的持续更新迭代，原生能力已经越来越强，工作流的大部分机制已经被原生能力覆盖，工作流已经可有可无，建议不再全量安装，只需要让Ai自行评估可以引入工作流中的哪些机制用以增强工具能力即可。

## 双模式自动切换

本工作流自动检测coding工具能力，无需手动选择：

| 模式 | 条件 | 子 Agent | 并行 | 适用工具 |
|------|------|:------:|:----:|---------|
| **正常模式** | task 工具可用 | 独立会话派发 | ✅ | Reasonix 等支持 task 子Agent 调用的工具 |
| **inline 模式** | task 不可用 | 编排器亲自完成 | ❌ | Claude Code、Cursor、通用 LLM 对话等不支持 task 的工具 |

> 两种模式功能一致，仅子agent调度方式不同（12 项任务全覆盖），inline 模式仅执行速度略慢。详见 [技术手册 §5](WORKFLOW_MANUAL.md#5-双模式自动切换)。

## 多模态支持

本工作流内置 MiMo MCP 提供多模态能力（图片分析、音频转写、视频理解等）。当执行工作流的推理模型不具备多模态能力时（如 DeepSeek），自动调用 MiMo 实现，无需手动切换模型、不打断上下文。如果当前模型已原生支持多模态（如 GPT-4o、Claude），则无需安装。

## 适用场景：这套工作流对谁有价值

本工作流诞生于「AI 工具机制尚不完善」的环境，本质是为**工具缺机制、模型能力受限**的组合补齐短板。它的价值取决于你的工具与模型组合，**安装前先对照**：

| 你的环境 | 建议 |
|---------|------|
| 工具缺少计划批准 / 任务跟踪 / 子Agent 调度（早期 reasonix、通用 LLM 对话等） | ✅ **完整安装**——工作流为这类环境补齐缺失机制 |
| 模型上下文小、容易遗忘、跳步、虚报完成 | ✅ **完整安装**——机械流程 + 硬门控正是补短板 |
| 团队或新手需要标准化开发流程模板 | ✅ **完整安装**——12 任务流水线即流程教科书 |
| 工具已有 Plan 模式 + 原生任务跟踪 + 子Agent（ZCode、Claude Code 等现代工具） | ⚠️ **只取方法论技能**（brainstorming、arch-review、verification-before-completion 等），跳过 wf-* 编排层 |
| 大上下文 + 强推理模型 | ⚠️ **同上**——编排机制是冗余，方法论纪律仍是资产 |

**一句话定位**：工具机制越齐全、模型越强，需要整套工作流的地方就越少——编排机制是过渡性补丁，方法论技能是长期资产。拿不准时，先只装方法论技能试用，再决定是否引入完整流水线。

## 安装

把下面这句话发给你的 AI：

```
帮我安装 agents-workflow，仓库是 github.com/TonyQ-AI/agents-workflow
```

AI 会自动完成 clone、注册技能、配置 MCP、检测 MiMo 端点等全部操作。

已安装旧版的用户，对 AI 说「升级工作流」即可自动迁移。
## 使用

```
/wf-orchestrator 项目名: 需求描述
/wf-orchestrator 项目名: 需求描述 --lite
/wf-orchestrator 项目名: 需求描述 --from coding
```
详见 [技术手册 §5](WORKFLOW_MANUAL.md#5-双模式自动切换)

## 项目关系

```
reasonix-workflow（旧版，已发布）
       ↓ 整合
agents-workflow（新版 · 双模式 · 40 技能 · 自适应）
```

## 更新日志

### v4.1.11（2026-07-31）— 适用场景说明

- **适用场景章节**：README / INSTALL / 技术手册 §1.4 新增「这套工作流对谁有价值」对照表——工具缺机制、模型弱、需要流程模板 → 完整安装；已有 Plan 模式 + 原生任务跟踪 + 子Agent 的现代工具（ZCode、Claude Code 等）→ 只取方法论技能
- **定位明确**：编排机制是过渡性补丁，方法论技能是长期资产；拿不准先只装技能层试用
- **文档修正**：正常模式适用工具描述更正为「Reasonix 等支持 task 子Agent 调用的工具」；技术手册版本号统一

### v4.1.10（2026-07-24）— 包管理器优化

- **优先 pnpm**：项目依赖安装按 pnpm → yarn → npm 顺序选择，pnpm 全局缓存不重复下载
- **Python 同理**：uv → poetry → pip

### v4.1.9（2026-07-24）— 知识沉淀加固 + 引擎瘦身

- **知识沉淀移至编排器层**：之前放在引擎任务12末尾，跑到最后预算耗尽被跳过；现由编排器在引擎返回后执行，预算充足
- **路径修复**：知识沉淀改用相对路径 `docs/superpowers/knowledge/`，防止 subagent 写权限不足
- **complete_step 前置**：todo 更新前必须先 `complete_step` 签名，再 `todo_write`，否则系统报冲突
- **引擎瘦身**：任务12只生成 SUMMARY.md，去掉知识沉淀代码

### v4.1.8（2026-07-24）— 进度跟踪修复 + 多工具兼容

- **子任务完成检查**：标记阶段完成前先扫描 task_plan.md，子任务未完成禁止推进
- **串行任务修复**：防止多个 in_progress 冲突，确保 todo 列表串行有序
- **多工具兼容**：Reasonix等支持"todo-write"的工具上用 `todo_write` 同步状态，若工具不支持则降级为编辑 markdown
- **todo 冲突修复**：不再出现"secondary in_progress"和"missing complete_step"错误

### v4.1.7（2026-07-13）— 升级迁移体验

- **升级检测**：自动检测旧版 `reasonix-workflow` / `workflow-task`，展示迁移说明
- **升级交互**：告知用户去品牌化、双模式、MCP 修复、仓库迁移等变化后确认升级
- **旧版清理**：升级时自动清理 `.reasonix`、`reasonix-workflow`、`workflow-task` 残留
- **升级报告**：迁移风格报告，突出新版本变化

### v4.1.6（2026-07-13）— 安装体验优化

- **安装器智能检测**：自动检测本地是否已有 MiMo key，避免重复输入
- **安装器交互强化**：DeepSeek key 不再询问（用户已配好推理 API），MiMo key 改为强制交互
- **中性化**：移除安装器文档中 `reasonix.toml` 引用

### v4.1.5（2026-07-13）— 架构评审修复

- **参数传递修复**：`--parallel` 和 `--timeout` 参数补传入引擎 JSON
- **进度映射修复**：engine progress.md 标签映射表 emoji 乱码恢复
- **并行竞态修复**：任务5 并行模式加 `CHANGES.md` 存在检查，防止 tester 空跑
- **引擎清理**：删除重复身份描述段落
- **deployer 输入补全**：显式传入工作流目录和上游产出路径
- **文件名对齐**：arch-review SKILL.md 产出文件名与引擎 `03-arch-review.md` 一致
- **品牌清理**：29 处 Reasonix 品牌名替换为中性表述

### v4.1.4（2026-07-13）— 引擎修复 + 文档补全

- 修复 `wf-orchestrator-engine/SKILL.md` YAML 前页缺少闭合 `---` 分隔符
- 修复 `workflow-installer` 仓库 URL，全部指向 `Workflow-Dualmode`
- 补齐缺失文档：WORKFLOW_MANUAL.md、INSTALL.md、LICENSE、.gitignore

### v4.1.3（2026-07-13）— 全平台适配

- MCP 独立发布：`tonyq-mimo-mcp-server` v0.2.2，默认 Standard API
- 安装器自动检测 MiMo 端点类型
- 技能目录清理：移除 reasonix- 前缀
- 路径中性化：`.reasonix/skills` → `skills/`

### v4.1.2（2026-07-13）

- MCP 服务迁移：`mimo-mcp-server` → `tonyq-mimo-mcp-server`
- 工具名对齐：统一加 `mimo_` 前缀
- 环境变量修复：`MIMO_API_BASE` → `MIMO_API_URL`
