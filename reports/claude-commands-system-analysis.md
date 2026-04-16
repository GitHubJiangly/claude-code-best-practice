# Claude Code Commands 系统深度分析

斜杠命令 — 注入当前上下文的提示模板与工作流编排器

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## Table of Contents

1. [什么是 Commands](#什么是-commands)
2. [Commands vs Subagents vs Skills](#commands-vs-subagents-vs-skills)
3. [自定义 Command 的结构与前置字段](#自定义-command-的结构与前置字段)
4. [触发机制与执行流程](#触发机制与执行流程)
5. [本仓库的 Commands 实现](#本仓库的-commands-实现)
6. [70 个官方内置命令分类](#70-个官方内置命令分类)
7. [最佳实践](#最佳实践)
8. [Sources](#sources)

---

<a id="什么是-commands"></a>

## 1. 什么是 Commands

Commands（命令）是放在 `.claude/commands/` 目录下的 Markdown 文件，用户通过 `/命令名` 在 Claude Code 中调用。调用后，Command 的内容被**注入当前对话上下文**，Claude 按照其中的指令执行操作。

**核心特点**：
- **轻量级**：本质是 Markdown 文件，无需编写代码
- **可编排**：可调用 Agent 工具、Skill 工具、Bash 命令
- **可共享**：纳入 git 版本控制，团队共享
- **参数化**：支持 `$ARGUMENTS` 占位符接收用户输入
- **可嵌套**：支持子目录组织，如 `.claude/commands/workflows/`

---

<a id="commands-vs-subagents-vs-skills"></a>

## 2. Commands vs Subagents vs Skills

| 特性 | Commands | Subagents | Skills |
|------|----------|-----------|--------|
| **位置** | `.claude/commands/<name>.md` | `.claude/agents/<name>.md` | `.claude/skills/<name>/SKILL.md` |
| **上下文** | 注入当前上下文（内联） | 独立隔离上下文 | 注入当前上下文（或 fork 隔离） |
| **调用方式** | `/命令名` 或自动发现 | `Agent()` 工具 | `Skill()` 工具或预加载 |
| **核心定位** | 工作流编排入口 | 自主执行的独立角色 | 可配置的专业知识包 |
| **适用场景** | 多步骤工作流、重复性操作 | 需要隔离上下文的复杂任务 | 领域知识注入、可重用能力 |
| **保留主上下文** | ✅ 是 | ❌ 否（独立上下文） | ✅ 是（除非 `context: fork`） |

**关键区别**：Commands 在当前上下文内执行，保留完整对话历史；Subagents 在全新上下文中运行，完成后仅返回结果。

详见：[Agents vs Commands vs Skills 对比](claude-agent-command-skill.md)

---

<a id="自定义-command-的结构与前置字段"></a>

## 3. 自定义 Command 的结构与前置字段

### 文件结构

```markdown
---
description: 命令做什么（显示在自动完成中）
model: haiku
argument-hint: [参数提示]
---

# 命令标题

## 指令内容
...
```

### 14 个前置字段（YAML Frontmatter）

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 显示名称和 `/slash-command` 标识符，默认取目录名 |
| `description` | string | 命令描述，显示在自动完成中，也用于自动发现（推荐设置） |
| `when_to_use` | string | 触发短语或示例请求，追加到 description 后（1536 字符上限） |
| `argument-hint` | string | 自动完成时显示的参数提示（如 `[issue-number]`） |
| `disable-model-invocation` | boolean | 设为 `true` 阻止 Claude 自动调用 |
| `user-invocable` | boolean | 设为 `false` 从 `/` 菜单隐藏（仅作为背景知识） |
| `paths` | string/list | Glob 模式限制激活条件（匹配文件时才可用） |
| `allowed-tools` | string | 命令激活时免权限提示的工具列表 |
| `model` | string | 命令运行时使用的模型（`haiku`/`sonnet`/`opus`） |
| `effort` | string | 模型努力程度（`low`/`medium`/`high`/`max`） |
| `context` | string | 设为 `fork` 在隔离子代理上下文中运行 |
| `agent` | string | `context: fork` 时的子代理类型 |
| `shell` | string | `` !`command` `` 块的 shell 类型（`bash`/`powershell`） |
| `hooks` | object | 此命令范围内的生命周期钩子 |

### 参数传递

用户输入的参数通过 `$ARGUMENTS` 占位符注入 Command 内容：

```bash
/weather-orchestrator celsius    # $ARGUMENTS = "celsius"
/workflow-concepts 10            # $ARGUMENTS = "10"
```

### 子目录组织

Commands 支持子目录结构，调用时使用 `:` 分隔：

```
.claude/commands/
├── time-command.md              → /time-command
├── weather-orchestrator.md      → /weather-orchestrator
└── workflows/
    ├── development-workflows.md → /workflows:development-workflows
    └── best-practice/
        ├── workflow-concepts.md → /workflows:best-practice:workflow-concepts
        └── workflow-claude-settings.md → /workflows:best-practice:workflow-claude-settings
```

---

<a id="触发机制与执行流程"></a>

## 4. 触发机制与执行流程

### 两种触发方式

| 方式 | 说明 |
|------|------|
| **手动调用** | 用户输入 `/命令名 [参数]` |
| **自动发现** | Claude 根据 `description` 和 `when_to_use` 判断是否自动调用（可通过 `disable-model-invocation: true` 禁止） |

### 完整执行流程

```
用户输入 /weather-orchestrator
  │
  ├─ 1. Claude Code 在 .claude/commands/ 中查找匹配文件
  │    └─ 找到 weather-orchestrator.md
  │
  ├─ 2. 解析 YAML 前置字段
  │    ├─ description → 用于显示
  │    ├─ model: haiku → 切换到 haiku 模型
  │    ├─ argument-hint → 参数提示
  │    └─ allowed-tools → 设置免权限工具
  │
  ├─ 3. 将 Markdown 正文注入当前对话上下文
  │    └─ $ARGUMENTS 被替换为用户输入的参数
  │
  ├─ 4. Claude 按注入的指令逐步执行
  │    ├─ Step 1: AskUserQuestion（用户交互）
  │    ├─ Step 2: Agent() 调用子代理（数据获取）
  │    └─ Step 3: Skill() 调用技能（数据渲染）
  │
  └─ 5. 输出结果到当前对话
```

### context: fork 模式

当设置 `context: fork` 时，执行流程改变：

```
用户输入 /命令名
  │
  ├─ 创建隔离子代理上下文
  ├─ 命令内容在子代理中执行
  ├─ 中间工具调用对主上下文不可见
  └─ 仅最终结果返回到主上下文
```

---

<a id="本仓库的-commands-实现"></a>

## 5. 本仓库的 Commands 实现

本仓库共有 **8 个自定义 Command**，分为三类：

### 5.1 核心演示命令

#### /weather-orchestrator — 编排工作流入口

```yaml
---
description: Fetch weather data for Dubai and create an SVG weather card
model: haiku
---
```

| 属性 | 值 |
|------|-----|
| **角色** | Command → Agent → Skill 编排模式的入口 |
| **模型** | haiku（轻量快速） |
| **执行步骤** | ① 询问温度单位 → ② 调用 weather-agent → ③ 调用 weather-svg-creator |
| **关键约束** | Agent 必须用 Agent 工具调用，Skill 必须用 Skill 工具调用 |

#### /time-command — 简单工具命令

```yaml
---
description: Display the current time in Pakistan Standard Time (PKT, UTC+5)
---
```

| 属性 | 值 |
|------|-----|
| **角色** | 单步执行的工具命令 |
| **执行步骤** | 运行 `TZ='Asia/Karachi' date` 命令并格式化输出 |

### 5.2 工作流自动化命令（6 个）

位于 `.claude/commands/workflows/` 子目录下：

| 命令 | 调用方式 | 功能 |
|------|----------|------|
| **development-workflows** | `/workflows:development-workflows` | 并行研究 10 个开发工作流仓库，更新 README 表格 |
| **workflow-concepts** | `/workflows:best-practice:workflow-concepts` | 检测 README CONCEPTS 部分与官方文档的偏差 |
| **workflow-claude-settings** | `/workflows:best-practice:workflow-claude-settings` | 追踪 Settings 报告变更 |
| **workflow-claude-commands** | `/workflows:best-practice:workflow-claude-commands` | 追踪 Commands 报告变更 |
| **workflow-claude-skills** | `/workflows:best-practice:workflow-claude-skills` | 追踪 Skills 报告变更 |
| **workflow-claude-subagents** | `/workflows:best-practice:workflow-claude-subagents` | 追踪 Subagents 报告变更 |

### 5.3 workflow-concepts 命令架构（典型复杂命令）

这是本仓库最复杂的 Command，展示了高级编排模式：

```
/workflows:best-practice:workflow-concepts [版本数]
  │
  ├─ Phase 0: 并行启动 2 个 Agent
  │    ├─ workflow-concepts-agent → 本地文件读取 + 外部文档获取
  │    └─ claude-code-guide agent → 独立 web 研究最新特性
  │
  ├─ Phase 0.5: 读取验证清单（如存在）
  │
  ├─ Phase 1: 读取历史 Changelog 对比
  │
  ├─ Phase 2: 合并两个 Agent 的发现，生成结构化报告
  │    ├─ Missing / Changed / Deprecated Concepts
  │    ├─ URL 准确性验证
  │    ├─ Badge 准确性验证
  │    └─ 标记 NEW / RECURRING / RESOLVED
  │
  ├─ Phase 2.5-2.7: 追加 Changelog + 更新 Badge + 验证 URL
  │
  └─ Phase 3: 询问用户执行哪些修复操作
```

---

<a id="70-个官方内置命令分类"></a>

## 6. 70 个官方内置命令分类

Claude Code 内置 70 个斜杠命令，按功能分为 10 类：

| 类别 | 数量 | 代表命令 |
|------|------|----------|
| **Auth（认证）** | 5 | `/login`, `/logout`, `/setup-bedrock`, `/setup-vertex`, `/upgrade` |
| **Config（配置）** | 11 | `/config`, `/permissions`, `/sandbox`, `/voice`, `/theme`, `/statusline` |
| **Context（上下文）** | 7 | `/context`, `/cost`, `/usage`, `/extra-usage`, `/insights`, `/stats` |
| **Debug（调试）** | 6 | `/doctor`, `/help`, `/powerup`, `/release-notes`, `/feedback`, `/tasks` |
| **Export（导出）** | 2 | `/copy`, `/export` |
| **Extensions（扩展）** | 8 | `/agents`, `/hooks`, `/mcp`, `/skills`, `/plugin`, `/chrome`, `/ide` |
| **Memory（记忆）** | 1 | `/memory` |
| **Model（模型）** | 6 | `/model`, `/effort`, `/fast`, `/plan`, `/ultraplan`, `/passes` |
| **Project（项目）** | 6 | `/init`, `/diff`, `/review`, `/security-review`, `/add-dir`, `/team-onboarding` |
| **Remote（远程）** | 10 | `/remote-control`, `/schedule`, `/teleport`, `/autofix-pr`, `/desktop`, `/web-setup` |
| **Session（会话）** | 8 | `/clear`, `/compact`, `/rename`, `/resume`, `/rewind`, `/branch`, `/btw`, `/exit` |

---

<a id="最佳实践"></a>

## 7. 最佳实践

### 创建 Command 的原则

| 原则 | 说明 |
|------|------|
| **每天做超过一次的事 → 做成 Command** | 节省重复提示，团队共享 |
| **用 Commands 编排，不用 standalone Agents** | Commands 更轻量，保留上下文 |
| **Command 编排，Agent 执行，Skill 赋能** | 三层架构分离关注点 |
| **始终设置 `description`** | 帮助自动发现和自动完成 |
| **复杂工作流用子目录** | 如 `workflows/best-practice/` 组织层级 |

### Command 编写模式

| 模式 | 说明 | 示例 |
|------|------|------|
| **单步工具命令** | 执行一个操作并返回 | `/time-command` |
| **多步编排命令** | 顺序调用多个工具/Agent/Skill | `/weather-orchestrator` |
| **并行研究命令** | 并行启动多个 Agent 后合并结果 | `/workflows:best-practice:workflow-concepts` |
| **交互式命令** | 使用 AskUserQuestion 收集用户偏好 | `/weather-orchestrator`（Step 1） |
| **参数化命令** | 通过 `$ARGUMENTS` 接收用户输入 | `/workflow-concepts 10` |

### 日常高频命令

| 命令 | 使用场景 |
|------|----------|
| `/compact` | 上下文达 50% 时手动压缩 |
| `/plan` | 复杂任务先规划再执行 |
| `/model` | 切换模型（Opus 规划 / Sonnet 编码） |
| `/context` | 查看上下文使用情况 |
| `/rewind` | Claude 偏离时回退 |
| `/rename` + `/resume` | 命名和恢复重要会话 |
| `/diff` | 查看未提交变更 |
| `/doctor` | 诊断安装和配置问题 |

---

<a id="sources"></a>

## Sources

- [Claude Code Slash Commands 文档](https://code.claude.com/docs/en/slash-commands)
- [Claude Code Interactive Mode](https://code.claude.com/docs/en/interactive-mode)
- [Claude Code Common Workflows](https://code.claude.com/docs/en/common-workflows)
- [Commands 最佳实践](../best-practice/claude-commands.md)
- [Commands 实现](../implementation/claude-commands-implementation.md)
- [编排工作流详情](../orchestration-workflow/orchestration-workflow.md)
- 本仓库实现：`.claude/commands/` 目录下全部 8 个命令文件
