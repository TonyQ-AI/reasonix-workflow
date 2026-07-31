# agents-workflow 安装指南

> 一键安装/升级多Agent协同开发工作流 · 双模式自适应 · 40 个技能

---

## 适用场景

本工作流为**工具缺机制、模型能力受限**的环境补齐短板。安装前先对照 [README 适用场景表](README.md#适用场景这套工作流对谁有价值)：

- **✅ 需要完整安装**：工具缺少计划批准 / 任务跟踪 / 子Agent 调度；模型上下文小、易遗忘跳步；团队或新手需要标准化流程模板
- **⚠️ 只需方法论技能**：工具已有 Plan 模式 + 原生任务跟踪 + 子Agent（ZCode、Claude Code 等现代工具），或使用大上下文 + 强模型——跳过 wf-* 编排层，只取 `brainstorming`、`arch-review`、`verification-before-completion` 等技能

> 拿不准？先只装方法论技能试用，再决定是否引入完整流水线。

---

## 快速安装

把下面这句话发给你的 AI：

```
帮我安装 agents-workflow，仓库是 github.com/TonyQ-AI/agents-workflow
```

根据AI提示操作即可。

## 手动安装

```bash
git clone --depth 1 https://github.com/TonyQ-AI/agents-workflow.git
```

在工具配置中添加技能路径：

```toml
[skills]
paths = ["agents-workflow/skills"]
```
---

## 双模式说明

本工作流安装后**自动检测**平台能力，无需手动选择：

| 模式 | 条件 | 执行方式 |
|------|------|---------|
| **正常模式** | task 工具可用 | 编排器 → task(引擎 subagent) → task(子Agent) |
| | | 适用：Reasonix 等支持 task 子Agent 调用的工具 |
| **inline 模式** | task 不可用 | 编排器直接执行所有阶段 |
| | | 适用：Claude Code、Cursor、通用 LLM 对话 等 |

---

## 多模态支持

本工作流内置 MiMo MCP 提供多模态能力（图片分析、音频转写、视频理解等）。当执行工作流的推理模型不具备多模态能力时（如 DeepSeek），自动调用 MiMo 实现，无需手动切换模型、不打断上下文。如果当前模型已原生支持多模态（如 GPT-4o、Claude），则无需安装。

---

## 一键升级

已有旧版工作流的用户，直接说：

```
升级工作流
```

AI 会自动执行：
1. 拉取最新 40 个技能
2. 更新 MCP 配置（`mimo-mcp-server` → `tonyq-mimo-mcp-server`）
3. 修复环境变量（`MIMO_API_BASE` → `MIMO_API_URL`）
4. 清除 MCP 缓存
5. 更新 AGENTS.md

---

## 使用

```bash
# 标准全流程
/wf-orchestrator 项目名: 需求描述

# 轻量模式（跳过架构评审+扫描）
/wf-orchestrator 项目名: 需求描述 --lite

# 从指定阶段开始
/wf-orchestrator 项目名: 需求描述 --from coding

# 指定结束阶段
/wf-orchestrator 项目名: 需求描述 --to testing
```

---

## 系统要求

| 组件 | 要求 |
|------|------|
| AI 工具 | 支持 skill 调用的任意平台（自动适配 task 能力） |
| npx | Node.js >= 18，用于运行 `tonyq-mimo-mcp-server` |
| API Key | DeepSeek（工作流推理）+ MiMo（多模态，可选） |
| 网络 | 可访问 github.com 和 npmjs.com |

---

## 常见问题

### MCP 报 401 / 404
- 确认 `MIMO_API_URL` 完整：`https://api.xiaomimimo.com/v1/chat/completions`
- 确认 API key 在 `.env` 中正确
- 运行「升级工作流」自动修复配置

### npx 找不到包
- 确认使用 `tonyq-mimo-mcp-server` 而非 `mimo-mcp-server`
- 运行 `npx --yes tonyq-mimo-mcp-server` 测试

### 工具名不匹配
- 新工具名统一加 `mimo_` 前缀：`mimo_understand_image` 等
- 重启清除 MCP 缓存即可

### 我的工具已有 Plan 模式 / 任务跟踪 / 子Agent，还需要装吗？
- 大概率**不需要完整安装**：编排机制（wf-* 技能）是为缺机制的工具设计的补丁层
- 建议只取方法论技能：`brainstorming`、`arch-review`、`verification-before-completion`、`systematic-debugging` 等，直接复制到你的技能目录即可
- 判断标准见 [README 适用场景表](README.md#适用场景这套工作流对谁有价值)

### wf-orchestrator 没反应
- 确认 skills/ 下有 `wf-orchestrator` 和 `wf-orchestrator-engine`
- 确认 AGENTS.md 存在
- 确认 DeepSeek API key 有效
- **inline 模式无需 wf-orchestrator-engine**，编排器会直接执行

---

## 相关链接

- GitHub 仓库：https://github.com/TonyQ-AI/agents-workflow
- MiMo MCP 包：https://www.npmjs.com/package/tonyq-mimo-mcp-server
