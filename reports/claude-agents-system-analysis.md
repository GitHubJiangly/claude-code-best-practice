# Claude Code Agents（子代理）系统深度分析

隔离上下文中的自主角色 — 自定义工具、权限、模型和持久记忆

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## Table of Contents

1. [什么是 Agents](#什么是-agents)
2. [Agents vs Commands vs Skills](#agents-vs-commands-vs-skills)
3. [16 个前置字段详解](#16-个前置字段详解)
4. [触发机制与执行流程](#触发机制与执行流程)
5. [5 个官方内置 Agent 类型](#5-个官方内置-agent-类型)
6. [本仓库的 Agents 实现](#本仓库的-agents-实现)
7. [Agent 设计模式](#agent-设计模式)
8. [最佳实践](#最佳实践)
9. [Sources](#sources)

---

<a id="什么是-agents"></a>

## 1. 什么是 Agents

Agents（子代理）是放在 `.claude/agents/` 目录下的 Markdown 文件，定义了在**独立隔离上下文**中运行的自主角色。与 Commands 和 Skills 在当前上下文内执行不同，Agent 拥有全新的上下文窗口、独立的工具集、权限模式、模型选择和持久记忆。

**核心特点**：
- **上下文隔离**：在全新的上下文窗口中运行，完成后仅结果返回调用者
- **自主决策**：有自己的工具集和权限，可独立完成复杂任务
- **可预加载 Skills**：通过 `skills:` 字段将领域知识注入 agent 上下文
- **可嵌套调用**：Agent 可以调用其他 Agent（但必须通过 Agent 工具，不能用 bash）
- **持久记忆**：通过 `memory` 字段跨会话保存项目/用户级学习

---

<a id="agents-vs-commands-vs-skills"></a>

## 2. Agents vs Commands vs Skills

| 特性 | Agents | Commands | Skills |
|------|--------|----------|--------|
| **位置** | `.claude/agents/<name>.md` | `.claude/commands/<name>.md` | `.claude/skills/<name>/SKILL.md` |
| **上下文** | 独立隔离上下文 | 注入当前上下文 | 注入当前上下文（或 fork 隔离） |
| **调用方式** | `Agent()` 工具 / 自动触发 | `/命令名` | `Skill()` 工具 / 预加载 |
| **保留主上下文** | ❌ 否 | ✅ 是 | ✅ 是（除非 fork） |
| **自主程度** | 高（独立决策） | 低（按指令执行） | 低（知识注入） |
| **适用场景** | 复杂隔离任务 | 工作流编排 | 领域知识注入 |

**关键区别**：Agent 在全新上下文运行，中间步骤对调用者不可见，仅返回最终结果。这使主上下文保持清洁、聚焦。

详见：[Agents vs Commands vs Skills 对比](claude-agent-command-skill.md)

---

<a id="16-个前置字段详解"></a>

## 3. 16 个前置字段详解

### 核心字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 唯一标识符，小写字母加连字符 |
| `description` | string | 何时调用此 agent。使用 `"PROACTIVELY"` 关键词启用自动触发 |
| `model` | string | 模型选择：`haiku`/`sonnet`/`opus`/`inherit`（默认继承） |
| `tools` | string/list | 工具白名单。省略则继承所有工具。支持 `Agent(agent_type)` 语法限制可调用的子代理 |
| `disallowedTools` | string/list | 工具黑名单，从继承或指定列表中移除 |

### 执行控制

| 字段 | 类型 | 说明 |
|------|------|------|
| `maxTurns` | integer | 最大执行轮次，超过后 agent 停止 |
| `permissionMode` | string | 权限模式：`default`/`acceptEdits`/`auto`/`bypassPermissions`/`plan` |
| `effort` | string | 努力程度：`low`/`medium`/`high`/`max` |

### 知识与集成

| 字段 | 类型 | 说明 |
|------|------|------|
| `skills` | list | 预加载到 agent 上下文的技能列表（完整内容注入，非仅可用） |
| `mcpServers` | list | 此 agent 可用的 MCP 服务器 |
| `hooks` | object | 生命周期钩子（支持 `PreToolUse`、`PostToolUse`、`Stop` 等） |
| `memory` | string | 持久记忆范围：`user`/`project`/`local` |

### 高级特性

| 字段 | 类型 | 说明 |
|------|------|------|
| `background` | boolean | 设为 `true` 始终作为后台任务运行 |
| `isolation` | string | 设为 `"worktree"` 在临时 git worktree 中运行 |
| `initialPrompt` | string | 作为主会话 agent 时自动提交的首轮提示 |
| `color` | string | 任务列表和转录中的显示颜色 |

---

<a id="触发机制与执行流程"></a>

## 4. 触发机制与执行流程

### 三种触发方式

| 方式 | 说明 | 示例 |
|------|------|------|
| **显式调用** | 代码或 Command 中通过 Agent 工具调用 | `Agent(subagent_type="weather-agent", ...)` |
| **自动触发** | description 含 `"PROACTIVELY"` 时 Claude 自动判断调用 | weather-agent、presentation-curator |
| **CLI 启动** | 通过 `--agent` 参数作为主会话 agent | `claude --agent=weather-agent` |

### 完整执行流程

```
触发 Agent（显式调用 / 自动触发 / CLI 启动）
  │
  ├─ 1. Claude Code 在 .claude/agents/ 中查找匹配定义
  │
  ├─ 2. 解析 YAML 前置字段
  │    ├─ model → 选择运行模型
  │    ├─ tools / disallowedTools → 配置工具集
  │    ├─ permissionMode → 设置权限模式
  │    ├─ maxTurns → 设置最大轮次
  │    ├─ skills → 预加载技能内容到上下文
  │    ├─ mcpServers → 连接 MCP 服务器
  │    └─ hooks → 注册生命周期钩子
  │
  ├─ 3. 创建全新的隔离上下文窗口
  │    ├─ 注入 agent 定义（Markdown 正文）
  │    ├─ 注入预加载 skills 的完整内容
  │    └─ 注入调用者传递的 prompt
  │
  ├─ 4. Agent 在隔离上下文中自主执行
  │    ├─ 使用被允许的工具完成任务
  │    ├─ 生命周期 hooks 在对应事件时触发
  │    ├─ 达到 maxTurns 或任务完成时停止
  │    └─ 如果配置了 memory → 写入持久记忆
  │
  ├─ 5. 仅最终结果返回调用者
  │    └─ 中间工具调用和推理过程对调用者不可见
  │
  └─ 隔离上下文销毁
```

### Agent 嵌套调用规则

```
✅ 正确方式：通过 Agent 工具调用
   Agent(subagent_type="weather-agent", description="...", prompt="...")

❌ 错误方式：通过 bash 命令调用
   Bash("claude --agent=weather-agent ...")  ← 不支持
```

**关键限制**：Subagents 不能通过 bash 命令调用其他 subagents。必须使用 Agent 工具（旧版名称 Task 仍可用作别名）。

---

<a id="5-个官方内置-agent-类型"></a>

## 5. 5 个官方内置 Agent 类型

| Agent | 模型 | 工具 | 用途 |
|-------|------|------|------|
| **general-purpose** | inherit | 全部 | 复杂多步任务（默认类型） |
| **Explore** | haiku | 只读（无 Write/Edit） | 快速代码库搜索和探索 |
| **Plan** | inherit | 只读（无 Write/Edit） | Plan mode 中的预规划研究 |
| **statusline-setup** | sonnet | Read, Edit | 配置状态栏 |
| **claude-code-guide** | haiku | Glob, Grep, Read, WebFetch, WebSearch | 回答 Claude Code 功能问题 |

---

<a id="本仓库的-agents-实现"></a>

## 6. 本仓库的 Agents 实现

本仓库共有 **10 个自定义 Agent**，分为三类：

### 6.1 核心演示 Agent

#### weather-agent — 天气数据获取

```yaml
name: weather-agent
model: sonnet
color: green
maxTurns: 5
permissionMode: acceptEdits
memory: project
skills:
  - weather-fetcher
hooks:
  PreToolUse: [...]
  PostToolUse: [...]
```

| 属性 | 值 |
|------|-----|
| **角色** | Command → Agent → Skill 架构中的数据获取层 |
| **预加载 Skill** | `weather-fetcher`（Open-Meteo API 调用指令） |
| **记忆** | project 级，跨会话保存历史温度读数 |
| **Hooks** | PreToolUse + PostToolUse + PostToolUseFailure（播放音效） |
| **自动触发** | description 含 `PROACTIVELY`，提到天气/迪拜时自动调用 |

#### presentation-curator — 演示文稿守护者

```yaml
name: presentation-curator
model: sonnet
color: magenta
skills:
  - presentation/vibe-to-agentic-framework
  - presentation/presentation-structure
  - presentation/presentation-styling
```

| 属性 | 值 |
|------|-----|
| **角色** | 演示文稿的唯一编辑者，通过 Rules 强制委派 |
| **预加载 Skills** | 3 个（框架概念、结构规范、样式指南） |
| **自我进化** | 每次执行后更新自身 skills 和 Learnings 记录 |
| **自动触发** | description 含 `PROACTIVELY`，操作演示文稿时自动调用 |

#### time-agent — 时间工具

```yaml
name: time-agent
model: haiku
maxTurns: 3
```

| 属性 | 值 |
|------|-----|
| **角色** | 轻量工具 agent，获取巴基斯坦时间 |
| **模型** | haiku（最快最便宜） |
| **限制** | 最多 3 轮，快速完成 |

### 6.2 研究型 Agent

#### development-workflows-research-agent

```yaml
name: development-workflows-research-agent
model: sonnet
color: cyan
maxTurns: 30
permissionMode: bypassPermissions
```

| 属性 | 值 |
|------|-----|
| **角色** | 研究 GitHub 开源工作流仓库 |
| **特点** | 高轮次（30）、绕过权限（只读研究） |
| **协议** | 6 步研究流程：Stars → Agents → Skills → Commands → Uniqueness → Changes |

### 6.3 工作流 Agent（6 个）

位于 `.claude/agents/workflows/` 子目录下：

| Agent | 模型 | 功能 |
|-------|------|------|
| **workflow-concepts-agent** | opus | 分析 README CONCEPTS 与官方文档的偏差 |
| **workflow-claude-settings-agent** | sonnet | 追踪 Settings 报告变更 |
| **workflow-claude-commands-agent** | sonnet | 追踪 Commands 报告变更 |
| **workflow-claude-skills-agent** | sonnet | 追踪 Skills 报告变更 |
| **workflow-claude-subagents-agent** | sonnet | 追踪 Subagents 报告变更 |
| **development-workflows-research-agent** (workflows/) | sonnet | 重复定义（与根目录版本相同） |

---

<a id="agent-设计模式"></a>

## 7. Agent 设计模式

### 7.1 专家 Agent（领域专用）

配备专属 skills，专注单一领域：

```
weather-agent + weather-fetcher skill → 天气数据专家
presentation-curator + 3 个 skills → 演示文稿专家
```

### 7.2 研究 Agent（只读分析）

高轮次、bypassPermissions、结构化输出：

```
development-workflows-research-agent → GitHub 仓库分析
workflow-concepts-agent → 文档偏差检测
```

### 7.3 自我进化 Agent

每次执行后更新自身知识，防止知识漂移：

```
presentation-curator → Step 5: 更新 skills + Learnings 记录
```

### 7.4 并行 Agent 协作

Command 同时启动多个 Agent，合并结果：

```
/workflow-concepts → 并行启动 workflow-concepts-agent + claude-code-guide
                   → 等待两者完成 → 交叉验证 → 合并报告
```

### 7.5 Test Time Compute（分离上下文验证）

利用上下文隔离实现独立验证：

```
Agent A（同一模型）→ 实现功能
Agent B（同一模型）→ 独立审查，发现 Agent A 的 bug
```

---

<a id="最佳实践"></a>

## 8. 最佳实践

### Agent 设计原则

| 原则 | 说明 |
|------|------|
| **功能专用而非通用** | 创建 "天气 agent"、"安全审查 agent" 而非 "后端工程师" |
| **配备专属 Skills** | 每个 agent 预加载领域知识，实现渐进式信息披露 |
| **合理设置 maxTurns** | 简单任务 3-5 轮，研究任务 20-30 轮 |
| **选择合适的模型** | haiku 用于简单工具任务，sonnet 用于常规工作，opus 用于复杂分析 |
| **PROACTIVELY 关键词** | 在 description 中使用此词触发自动调用 |

### 何时用 Agent vs Command

| 场景 | 推荐 |
|------|------|
| 需要保持主上下文连续性 | Command |
| 需要隔离上下文避免污染 | Agent |
| 多步工作流编排 | Command 调用 Agent |
| 长时间独立研究 | Agent（background: true） |
| 并行处理多个任务 | 多个 Agent 并行 |
| 简单重复操作 | Command |

### 常见陷阱

| 陷阱 | 解决 |
|------|------|
| 用 bash 调用 agent | 必须用 Agent 工具 |
| Agent 描述太通用 | 写明具体触发条件和领域 |
| 不设 maxTurns | 设置合理上限避免无限循环 |
| 所有任务都用 Agent | 简单任务直接用 Command 更高效 |
| 忽略 permissionMode | 只读研究用 bypassPermissions 减少中断 |

---

<a id="sources"></a>

## Sources

- [Claude Code Subagents 文档](https://code.claude.com/docs/en/sub-agents)
- [Claude Code CLI Reference](https://code.claude.com/docs/en/cli-reference)
- [Subagents 最佳实践](../best-practice/claude-subagents.md)
- [Subagents 实现](../implementation/claude-subagents-implementation.md)
- [编排工作流详情](../orchestration-workflow/orchestration-workflow.md)
- [Agent Memory 报告](claude-agent-memory.md)
- 本仓库实现：`.claude/agents/` 目录下全部 10 个 agent 文件
