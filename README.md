# agents-workflow

> 多Agent协同开发工作流 · 双模式自适应 · 40 个技能 · 全平台兼容

[![Version](https://img.shields.io/badge/version-4.1.7-blue)](VERSION)
[![Skills](https://img.shields.io/badge/skills-40-green)](skills/)

## 双模式自动切换

本工作流自动检测coding工具能力，无需手动选择：

| 模式 | 条件 | 子 Agent | 并行 | 适用工具 |
|------|------|:------:|:----:|---------|
| **正常模式** | task 工具可用 | 独立会话派发 | ✅ | Reasonix、ZCode 等支持 task 的工具 |
| **inline 模式** | task 不可用 | 编排器亲自完成 | ❌ | Claude Code、Cursor、通用 LLM 对话等不支持 task 的工具 |

> 两种模式功能一致，仅子agent调度方式不同（12 项任务全覆盖），inline 模式仅执行速度略慢。详见 [技术手册 §5](WORKFLOW_MANUAL.md#5-双模式自动切换)。

## 多模态支持

本工作流内置 MiMo MCP 提供多模态能力（图片分析、音频转写、视频理解等）。当执行工作流的推理模型不具备多模态能力时（如 DeepSeek），自动调用 MiMo 实现，无需手动切换模型、不打断上下文。如果当前模型已原生支持多模态（如 GPT-4o、Claude），则无需安装。

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

### v4.1.8（2026-07-24）— 进度跟踪修复 + 多工具兼容

- **子任务完成检查**：标记阶段完成前先扫描 task_plan.md，子任务未完成禁止推进
- **串行任务修复**：防止多个 in_progress 冲突，确保 todo 列表串行有序
- **多工具兼容**：Reasonix 上用 `todo_write` 同步状态，其他工具降级为编辑 markdown
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
