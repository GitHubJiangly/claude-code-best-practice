# Claude Code 最佳实践概览

从 Vibe Coding 到 Agentic Engineering — 技术体系与使用指南

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## Table of Contents

1. [核心架构概念（6大组件）](#核心架构概念)
2. [编排工作流模式](#编排工作流模式)
3. [Memory 记忆系统](#memory-记忆系统)
4. [实战 Tips 精华](#实战-tips-精华)
5. [开发工作流生态](#开发工作流生态)
6. [使用建议与学习路径](#使用建议与学习路径)
7. [Sources](#sources)

---

<a id="核心架构概念"></a>

## 1. 核心架构概念（6大组件）

Claude Code 的扩展体系由 6 大核心组件构成，每个组件解决不同层面的问题：

| 组件 | 位置 | 核心定位 |
|------|------|----------|
| **Commands** | `.claude/commands/<name>.md` | 注入当前上下文的提示模板 |
| **Subagents** | `.claude/agents/<name>.md` | 在隔离上下文中运行的自主角色 |
| **Skills** | `.claude/skills/<name>/SKILL.md` | 可配置、可预加载、可自动发现的知识包 |
| **Hooks** | `.claude/hooks/` | 代理循环之外的事件驱动处理器 |
| **MCP Servers** | `.claude/settings.json` | 连接外部工具/数据库/API 的桥梁 |
| **Settings** | `.claude/settings.json` | 分层优先级配置系统 |

更多对比详情参见：[Agents vs Commands vs Skills](claude-agent-command-skill.md)

---

### 1.1 Commands（命令）

**位置**: `.claude/commands/<name>.md`

Commands 是 Markdown 文件，用 `/命令名` 调用，将提示内容注入当前对话上下文。

**核心用途**：
- 封装重复性工作流（每天执行多次的操作）
- 编排多步骤流程（协调 Agent 和 Skill）
- 团队可共享（文件纳入 git 版本控制）

**最佳实践**：
- 用 commands 代替 standalone agents 做工作流编排（更轻量）
- 每天做超过一次的事情，都应该做成 command 或 skill
- 使用 `$ARGUMENTS` 占位符接收用户输入参数

详见：[Commands 最佳实践](../best-practice/claude-commands.md) · [Commands 实现](../implementation/claude-commands-implementation.md)

---

### 1.2 Subagents（子代理）

**位置**: `.claude/agents/<name>.md`

Subagents 在独立的上下文窗口中运行，拥有自己的工具集、权限、模型和记忆。

**YAML 前置配置**：

| 字段 | 说明 | 示例 |
|------|------|------|
| `name` | 代理标识符 | `weather-agent` |
| `model` | 模型别名 | `haiku` / `sonnet` / `opus` / `inherit` |
| `tools` | 工具白名单 | `Bash, Read, Write, Agent` |
| `skills` | 预加载技能 | `weather-fetcher` |
| `maxTurns` | 最大轮次 | `10` |
| `permissionMode` | 权限模式 | `acceptEdits` / `bypassPermissions` |
| `memory` | 记忆范围 | `user` / `project` / `local` |
| `isolation` | 隔离模式 | `worktree` |
| `hooks` | 生命周期钩子 | `PreToolUse`, `Stop` 等 |

**调用方式**（必须使用 Agent 工具，不能通过 bash）：

```
Agent(subagent_type="weather-agent", description="获取天气", prompt="...", model="haiku")
```

**最佳实践**：
- 创建功能专用的子代理（如 "前端验证代理"、"安全审查代理"），而非通用的 "后端工程师" 代理
- 每个代理配备专属 skills 实现渐进式信息披露
- 用 "test time compute" — 分离上下文让结果更好；一个 agent 制造 bug，另一个发现 bug

详见：[Subagents 最佳实践](../best-practice/claude-subagents.md) · [Subagents 实现](../implementation/claude-subagents-implementation.md)

---

### 1.3 Skills（技能）

**位置**: `.claude/skills/<name>/SKILL.md`

Skills 是文件夹而非文件，可包含 `references/`、`scripts/`、`examples/` 子目录，实现渐进式信息披露。

**两种使用模式**：

| 模式 | 说明 | 调用方式 |
|------|------|----------|
| **Agent Skill**（预加载） | 通过 `skills:` 字段预加载到 agent 中，作为领域知识 | agent 定义中配置 |
| **Invoked Skill**（主动调用） | 在运行时通过 Skill 工具调用 | `Skill("skill-name")` |

**YAML 前置配置**：

| 字段 | 说明 |
|------|------|
| `description` | 触发条件（写给模型看的 "何时触发我？"） |
| `context: fork` | 在隔离子代理中运行，主上下文只看结果 |
| `allowed-tools` | 允许使用的工具（免权限提示） |
| `hooks` | 技能范围内的生命周期钩子 |
| `disable-model-invocation` | 设为 `true` 防止自动调用 |

**最佳实践**：
- description 字段是触发器，不是摘要 — 为模型写的 "何时触发"
- 每个 skill 应有 Gotchas 章节记录 Claude 的失败点
- 不要事无巨细 — 聚焦于推动 Claude 脱离默认行为的内容
- 给目标和约束，不要给死板的步骤指令
- 嵌入脚本和库，让 Claude 组合而非重建样板代码

详见：[Skills 最佳实践](../best-practice/claude-skills.md) · [Skills 实现](../implementation/claude-skills-implementation.md)

---

### 1.4 Hooks（钩子）

**位置**: `.claude/hooks/`

Hooks 在代理循环之外运行，响应 Claude Code 的生命周期事件。

**支持的事件**：

| 事件 | 触发时机 |
|------|----------|
| `PreToolUse` | 工具执行前 |
| `PostToolUse` | 工具执行后 |
| `UserPromptSubmit` | 用户提交提示 |
| `Stop` | 代理停止 |
| `Notification` | 通知事件 |
| `SubagentStart` / `SubagentStop` | 子代理生命周期 |
| `PreCompact` | 上下文压缩前 |
| `SessionStart` / `SessionEnd` | 会话生命周期 |
| `PermissionRequest` | 权限请求 |
| `TaskCompleted` | 任务完成 |
| `ConfigChange` | 配置变更 |

**实用 Hook 模式**：

| 模式 | 说明 |
|------|------|
| PostToolUse → 自动格式化 | Claude 生成代码后，hook 补齐最后 10% 格式 |
| Stop → 验证工作 | 在每轮结束时验证输出或推动继续 |
| PermissionRequest → Opus 审批 | 路由到 Opus 模型扫描攻击并自动审批安全操作 |
| PreToolUse → 技能计量 | 测量 skill 使用率，发现热门或触发不足的技能 |
| Skill hooks → 按需防护 | `/careful` 阻止破坏性命令，`/freeze` 阻止目录外编辑 |

详见：[Hooks 项目](https://github.com/shanraisshan/claude-code-hooks)

---

### 1.5 MCP Servers（模型上下文协议）

**位置**: `.claude/settings.json`、`.mcp.json`

MCP 为 Claude Code 提供连接外部工具、数据库和 API 的能力。

**本仓库配置的 MCP**：

| MCP Server | 用途 |
|------------|------|
| **Playwright** | 浏览器自动化（页面交互、截图、表单填写） |
| **Context7** | 实时获取任何库/框架的最新文档 |
| **DeepWiki** | 获取 GitHub 仓库的深度 Wiki 信息 |

详见：[MCP 最佳实践](../best-practice/claude-mcp.md)

---

### 1.6 Settings（配置系统）

**分层优先级**（从高到低）：

```
1. Managed（组织强制，不可覆盖）
2. CLI 命令行参数（单次会话覆盖）
3. .claude/settings.local.json（个人项目设置，git-ignored）
4. .claude/settings.json（团队共享设置）
5. ~/.claude/settings.json（全局个人默认）
```

**关键配置能力**：权限管理、模型选择、输出样式、沙箱隔离、快捷键、状态栏、归属标注等。

**最佳实践**：确定性行为（如提交格式 `attribution.commit`）放 `settings.json`，不要放 CLAUDE.md。

详见：[Settings 最佳实践](../best-practice/claude-settings.md) · [Global vs Project Settings](claude-global-vs-project-settings.md)

---

<a id="编排工作流模式"></a>

## 2. 编排工作流模式：Command → Agent → Skill

本仓库通过天气系统演示了核心编排架构：

```
用户执行 /weather-orchestrator （Command）
  │
  ├─ Step 1: 询问用户要 °C 还是 °F
  │
  ├─ Step 2: Agent 工具 → weather-agent（Subagent）
  │    └─ agent 按预加载的 weather-fetcher 技能（Skill）获取数据
  │    └─ 调用 Open-Meteo API
  │    └─ 返回: 温度 + 单位
  │
  ├─ Step 3: Skill 工具 → weather-svg-creator（Skill）
  │    └─ 接收温度数据
  │    └─ 生成 SVG 天气卡片 + Markdown 摘要
  │
  └─ 最终: 展示结果给用户
```

### 组件分工

| 层级 | 组件 | 职责 |
|------|------|------|
| **编排层** | `/weather-orchestrator` (Command) | 用户交互 + 流程编排 |
| **数据层** | `weather-agent` (Subagent) | 数据获取（含预加载 skill） |
| **渲染层** | `weather-svg-creator` (Skill) | 数据可视化输出 |

### 两种 Skill 模式对比

| | Agent Skill（预加载） | Invoked Skill（主动调用） |
|---|---|---|
| **示例** | `weather-fetcher` | `weather-svg-creator` |
| **加载方式** | agent 的 `skills:` 字段 | `Skill("name")` 工具 |
| **上下文** | 嵌入 agent 上下文 | 当前对话上下文 |
| **用途** | 为 agent 提供领域知识 | 在流程中执行特定任务 |

**核心价值**：数据来源（Agent）与数据呈现（Skill）完全解耦，易于维护、扩展和测试。

详见：[编排工作流详情](../orchestration-workflow/orchestration-workflow.md)

---

<a id="memory-记忆系统"></a>

## 3. Memory 记忆系统

Claude Code 的记忆通过多层文件实现持久化上下文：

| 位置 | 作用 | 加载方式 |
|------|------|----------|
| `CLAUDE.md`（项目根目录） | 项目级持久指令 | 所有会话自动加载 |
| `.claude/rules/*.md` | 按 glob 模式匹配的分领域规则 | 匹配文件时自动加载 |
| `~/.claude/rules/` | 全局个人规则 | 所有项目自动加载 |
| `~/.claude/projects/<project>/memory/` | 项目级自动记忆 | 对应项目自动加载 |
| 子目录 `CLAUDE.md` | 子目录级指令 | 访问子目录时加载（祖先+后代） |

### CLAUDE.md 最佳实践

| 原则 | 说明 |
|------|------|
| **200 行以内** | 每个文件控制在 200 行以内，保证可靠遵守 |
| **分拆指令** | 用 `.claude/rules/` 分拆大型指令集 |
| **确定性行为放 settings** | 如提交格式、权限等放 `settings.json` |
| **保持代码库整洁** | 部分迁移的框架会混淆模型选择错误模式 |
| **"run the tests" 测试** | 任何开发者启动 Claude 说 "run the tests" 应一次成功 |
| **`<important if="...">` 标签** | 包裹关键规则防止被忽略 |

详见：[Memory 最佳实践](../best-practice/claude-memory.md)

---

<a id="实战-tips-精华"></a>

## 4. 实战 Tips 精华

本仓库收集了 69 条来自 Boris Cherny（Claude Code 创始人）、Thariq（Anthropic）和社区的实战技巧，按类别精选如下：

### 提示与规划

| 技巧 | 来源 |
|------|------|
| 始终从 plan mode 开始，用 Opus 做规划、Sonnet 写代码 | Boris |
| 写详细 spec，减少歧义后再交给 Claude 执行 | Boris |
| 让 Claude 用 AskUserQuestion 工具采访你，然后新会话执行 spec | Thariq |
| 起第二个 Claude 以 staff engineer 身份审查你的计划 | Boris |

### 工作流管理

| 技巧 | 来源 |
|------|------|
| 50% 上下文时手动 `/compact`，避免 agent 呆滞区 | 社区 |
| 始终开启 thinking mode + Explanatory 输出样式 | Boris |
| Claude 偏离时用 `Esc Esc` 或 `/rewind` 回退 | 社区 |
| `/rename` 重要会话，`/resume` 稍后继续 | Cat |
| `/loop` 本地循环（最多7天），`/schedule` 云端定时（机器关了也跑） | 社区 |

### Git / PR 规范

| 技巧 | 来源 |
|------|------|
| PR 保持小而聚焦 — Boris 的 p50 是 118 行 | Boris |
| 始终 squash merge — 干净线性历史 | Boris |
| 每小时至少提交一次，任务完成就提交 | 社区 |
| `@claude` 标注同事 PR 自动生成 lint 规则 | Boris |

### 调试技巧

| 技巧 | 来源 |
|------|------|
| 截图分享给 Claude 是最有效的 bug 报告方式 | 社区 |
| 用 MCP（Playwright / Chrome DevTools）让 Claude 直接看控制台 | 社区 |
| 长时间终端命令作为后台任务运行，方便查看日志 | 社区 |
| agentic search（glob + grep）胜过 RAG | Boris |

更多 Tips 详见 README 的 [TIPS AND TRICKS](../#-tips-and-tricks-69) 部分。

---

<a id="开发工作流生态"></a>

## 5. 开发工作流生态

所有主流工作流都遵循相同的架构模式：**Research → Plan → Execute → Review → Ship**

| 仓库 | 星数 | 核心特色 |
|------|------|----------|
| [Everything Claude Code](https://github.com/affaan-m/everything-claude-code) | 156k | 直觉评分、AgentShield、多语言规则 |
| [Superpowers](https://github.com/obra/superpowers) | 152k | TDD 优先、Iron Laws、全计划审查 |
| [Spec Kit](https://github.com/github/spec-kit) (GitHub) | 88k | 规格驱动、宪法模式、22+ 工具 |
| [gstack](https://github.com/garrytan/gstack) (Garry Tan) | 72k | 角色 persona、并行冲刺 |
| [Get Shit Done](https://github.com/gsd-build/get-shit-done) | 53k | 200K 新上下文、波次执行 |
| [BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD) | 45k | 完整 SDLC、agent personas |
| [OpenSpec](https://github.com/Fission-AI/OpenSpec) | 40k | delta specs、brownfield 支持 |
| [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) | 29k | 团队编排、tmux workers |
| [Compound Engineering](https://github.com/EveryInc/compound-engineering-plugin) | 14k | 复合学习、插件市场 |
| [HumanLayer](https://github.com/humanlayer/humanlayer) | 10k | RPI 模式、300k+ LOC |

---

<a id="使用建议与学习路径"></a>

## 6. 使用建议与学习路径

### 学习路径

```
1. 先读概念 — 理解 Commands、Agents、Skills、Hooks 各自的定位
2. 动手实验 — 克隆本仓库，跑 /weather-orchestrator，听 hook 声音，跑 agent teams
3. 迁移实践 — 让 Claude 参考本仓库为你的项目定制最佳实践
```

### 日常习惯

| 习惯 | 说明 |
|------|------|
| 每日更新 Claude Code | `npm install -g @anthropic-ai/claude-code@latest` |
| 每日读 Changelog | 了解新功能和改进 |
| 50% 上下文时 `/compact` | 避免模型质量下降 |
| 复杂任务先 plan mode | 先规划再执行 |
| 每小时至少 commit 一次 | 保持进度可追溯 |

### 热门新特性

| 特性 | 说明 |
|------|------|
| **Routines** | 云端自动化，定时/API/GitHub 事件驱动 |
| **Agent Teams** | 多代理并行，tmux + git worktrees |
| **Ultraplan** | 云端规划，浏览器审阅 |
| **Auto Mode** | 安全分类器替代手动权限提示 |
| **Agent SDK** | Python/TypeScript SDK 构建生产级代理 |
| **Voice Dictation** | `/voice` 语音输入，20 种语言 |
| **Remote Control** | `/rc` 从任何设备继续会话 |
| **Ralph Wiggum Loop** | 长时间自主开发循环 |

---

<a id="sources"></a>

## Sources

- [Claude Code 官方文档](https://code.claude.com/docs)
- [Claude Code Best Practices (Official)](https://code.claude.com/docs/en/best-practices)
- [Boris Cherny Tips 合集](../tips/claude-boris-13-tips-03-jan-26.md)
- [Thariq Skills 指南](../tips/claude-thariq-tips-17-mar-26.md)
- [编排工作流详情](../orchestration-workflow/orchestration-workflow.md)
- [Agents vs Commands vs Skills 对比](claude-agent-command-skill.md)
- [claude-code-hooks 项目](https://github.com/shanraisshan/claude-code-hooks)
