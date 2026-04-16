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
7. [9 大 Agent 设计模式](#9-大-agent-设计模式)
8. [Agent 记忆系统](#agent-记忆系统)
9. [最佳实践](#最佳实践)
10. [Sources](#sources)

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

## 7. 9 大 Agent 设计模式

### 7.1 Specialist Agent — 领域专家代理

**定义**：配备专属预加载 Skills，专注单一领域的深度专家。

**架构**：
```
Agent 启动 → 注入预加载 Skill 完整内容 → Agent 拥有领域知识 → 自主执行
```

**本仓库实例**：

| Agent | 预加载 Skills | 领域 |
|-------|--------------|------|
| `weather-agent` | `weather-fetcher` | 天气数据获取 |
| `presentation-curator` | 3 个 skills（框架/结构/样式） | 演示文稿管理 |

**配置示例**：
```yaml
name: weather-agent
model: sonnet
skills:
  - weather-fetcher    # 完整 API 调用指令在启动时注入
memory: project        # 跨会话记忆历史读数
maxTurns: 5
```

**执行流程**：
```
weather-agent 启动
  ├─ 注入 weather-fetcher 技能内容（Open-Meteo API 规范）
  ├─ Agent 已理解如何调用 API
  ├─ 自主执行 WebFetch 获取温度
  └─ 仅返回 {temperature: 24, unit: "Celsius"} 给调用者
```

**预加载 vs 动态调用的区别**：

| 方面 | 预加载（skills: 字段） | 动态调用（Skill 工具） |
|------|----------------------|---------------------|
| 注入时机 | Agent 启动前 | 执行时按需 |
| 上下文成本 | 一次性（启动时） | 每次调用时 |
| 适合 | 频繁参考的领域规范 | 偶尔查询的参考资料 |

**适用场景**：数据获取 Agent、代码审查 Agent、文档生成 Agent。

---

### 7.2 Self-Evolving Agent — 自我进化代理

**定义**：每次执行后自动更新自身 Skills 和 Learnings 记录，防止知识漂移。

**本仓库实例**：`presentation-curator`

**进化机制（5 步工作流）**：
```
Step 1: 读取当前演示文稿状态
Step 2: 按预加载框架验证
Step 3: 执行修改
Step 4: 验证完整性
Step 5: 自我进化 ← 关键步骤
  ├─ 5a. 更新框架 Skill（level 转换表、章节范围）
  ├─ 5b. 更新结构 Skill（slide 范围、divider 格式）
  ├─ 5c. 同步跨文档一致性（settings docs、hooks docs）
  └─ 5d. 将新发现写入 Learnings 章节
```

**Learnings 记录示例**（来自 presentation-curator 实际记录）：
```markdown
## Learnings
- Hook-event references drifted across files. Treat `16 hook events` as canonical.
- Do not use shorthand agent names in examples (`frontend-eng`).
- Never hardcode `.weight-badge` in slide HTML; badges are runtime-injected by JS.
- The main presentation caps at High level (not Pro). Do not assign data-level="pro".
```

**优势**：长期适应性 + 防止知识漂移 + 无需人工维护。

**适用场景**：演示文稿/文档管理、代码标准维护、架构演变追踪。

---

### 7.3 Research Agent — 研究代理

**定义**：高轮次、只读权限、结构化输出的深度分析 Agent。

**本仓库实例**：`development-workflows-research-agent`（maxTurns: 30）

**关键配置**：
```yaml
name: development-workflows-research-agent
model: sonnet
maxTurns: 30                      # 高轮次允许深度探索
permissionMode: bypassPermissions  # 只读研究不需频繁确认
tools: [Read, Glob, Grep, WebFetch, WebSearch]  # 限制为只读工具
```

**6 步研究协议**：
```
Step 1: 获取 GitHub Stars（API → stargazers_count）
Step 2: 计数 Agents（.claude/agents/ 目录）
Step 3: 计数 Skills（.claude/skills/ 目录）
Step 4: 计数 Commands（.claude/commands/ 目录）
Step 5: 评估独特性（vs 其他仓库）
Step 6: 检查近期变更（releases + commits）
→ 返回结构化报告 + 置信度评分
```

**适用场景**：GitHub 仓库分析、代码库评估、文档漂移检测、竞品研究。

---

### 7.4 Parallel Agent Collaboration — 并行代理协作

**定义**：Command 同时启动多个 Agent 独立工作，等待全部完成后交叉验证并合并结果。

**本仓库实例**：`/workflows:best-practice:workflow-concepts`

**架构**：
```
/workflow-concepts [版本数] (Command 编排)
  │
  ├─ 并行启动（同一消息中发出多个 Agent 调用）
  │    ├─ workflow-concepts-agent (opus) → 本地文件 + 外部文档对比
  │    └─ claude-code-guide (haiku) → 独立 web 研究最新特性
  │
  ├─ 等待两者完成
  │
  ├─ 交叉验证
  │    ├─ Agent A 的发现 vs Agent B 的发现
  │    ├─ 标记矛盾点
  │    └─ 合并互补信息
  │
  └─ 生成统一报告（Missing / Changed / Deprecated + Action Items）
```

**为什么用两个 Agent 而非一个**：
- Agent A（专用）深入分析本地文件 + 特定外部源
- Agent B（通用）独立 web 搜索，可能发现 A 遗漏的最新变更
- 交叉验证消除盲区

**适用场景**：多源信息验证、跨领域分析、代码审查（多审查者并行）。

---

### 7.5 Test Time Compute — 多上下文验证

**定义**：同一模型在不同隔离上下文中进行独立验证，利用上下文隔离提高质量。

**核心原理**（Boris Cherny 的观察）：
> "使用分离的上下文窗口让结果更好 — 这就是为什么一个 agent 可以制造 bug，而另一个（使用完全相同的模型）能发现它们。"

**Writer + Reviewer 模式**：
```
Agent A: Writer（Sonnet）— 独立上下文
  ├─ 实现功能
  └─ 返回代码 + 说明

  ↓ [Command 协调]

Agent B: Reviewer（Sonnet，同一模型）— 全新独立上下文
  ├─ 接收 Agent A 的代码（无 A 的推理过程）
  ├─ 独立分析（不受 A 的思路污染）
  └─ 发现 A 遗漏的 bug 和边界情况
```

**为什么有效**：

| 因素 | 说明 |
|------|------|
| 上下文隔离 | Reviewer 不受 Writer 推理路径影响 |
| 认知多样性 | 新上下文产生不同的关注点 |
| 工程类比 | 就像代码审查 — 同事比作者更容易找到 bug |

**进阶：多专家审查**：
```
Writer Agent (Sonnet) → 实现
  ├─ Security Auditor (Opus) → 安全审查
  ├─ Performance Validator (Sonnet) → 性能检查
  └─ Edge Case Finder (Sonnet) → 边界情况
→ Command 汇总所有发现
```

**适用场景**：关键业务逻辑验证、安全敏感代码、PR 自动审查。

---

### 7.6 Background Agent — 后台代理

**定义**：始终以后台任务运行，不阻塞主会话。

**配置**：
```yaml
name: background-verifier
model: sonnet
background: true    # 关键字段
description: Verify completed work in the background
```

**执行模式**：
```
主会话：实现功能（占用屏幕）
  ├─ Claude 完成实现
  ├─ 触发 background-verifier（后台启动）
  └─ 继续接收用户下一个输入（不等待验证）

后台：
  ├─ 运行测试套件
  ├─ 检查代码质量
  └─ 完成后通知用户结果
```

**适用场景**：长时间验证任务、后台测试运行、CI/CD 集成。

---

### 7.7 Worktree Isolation Agent — 工作树隔离代理

**定义**：在临时 git worktree 中运行，实现上下文 + 文件系统双重隔离。

**配置**：
```yaml
name: feature-developer
model: sonnet
isolation: "worktree"    # 关键字段
```

**生命周期**：
```
Agent 启动
  ├─ 创建临时 git worktree（从 HEAD 分支）
  ├─ 在隔离的文件副本上工作
  └─ 上下文隔离 + 文件系统隔离

Agent 完成
  ├─ 若无文件更改 → 自动清理 worktree
  └─ 若有更改 → 保留 worktree 供用户审查
```

**Settings 配置优化**：
```json
{
  "worktree": {
    "sparsePaths": ["src/", "tests/"],
    "symlinkDirectories": ["node_modules/", ".venv/"]
  }
}
```
- `sparsePaths`：只检出指定目录（节省磁盘）
- `symlinkDirectories`：符号链接大目录（加速创建）

**适用场景**：并行特性开发、隔离实验性改动、多分支同时处理。

---

### 7.8 PROACTIVELY Auto-trigger — 主动触发模式

**定义**：通过 description 中的 `"PROACTIVELY"` 关键词，让 Claude 自动判断何时调用 Agent。

**触发机制**：
```
用户输入: "迪拜现在天气怎么样？"
  │
  ├─ Claude 扫描所有 Agent 的 description
  ├─ weather-agent: "PROACTIVELY fetch weather data for Dubai..."
  ├─ 语义匹配 ✅
  └─ 自动创建 Agent() 调用（用户无需显式触发）
```

**本仓库使用 PROACTIVELY 的 Agent**：

| Agent | 触发条件 |
|-------|----------|
| `weather-agent` | 提到天气/迪拜/温度 |
| `presentation-curator` | 操作演示文稿 |

**description 写作对比**：
```yaml
# ✅ 好的 description
description: |
  PROACTIVELY fetch real-time weather data when the user asks about:
  - Current temperature in Dubai, UAE
  - Weather conditions, humidity, wind speed
  - Climate queries for UAE

# ❌ 不好的 description
description: "Weather agent"       # 太模糊，无法精确匹配
description: "Get weather"         # 不说明何时使用
```

**适用场景**：领域问答、自动委派、触发条件清晰的专用 Agent。

---

### 7.9 Agent Teams — 并行代理团队

**定义**：多个 Agent 在 tmux 中并行工作于同一代码库，通过共享任务队列协调。

**启用**：
```bash
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude
```

**团队架构**：
```
                    主会话（编排者）
                         │
        ┌────────────────┼────────────────┐
        ↓                ↓                ↓
     Agent 1          Agent 2           Agent 3
   (tmux pane 1)   (tmux pane 2)   (tmux pane 3)
   Command 设计     Agent 逻辑      Skill 开发
```

**配合 Git Worktrees**（Boris 推荐的最大生产力提升）：
```bash
# 每个 Agent 在独立 worktree 中并行开发
claude -w  # worktree 1: 前端 Agent
claude -w  # worktree 2: 后端 Agent
claude -w  # worktree 3: 测试 Agent
```

**显示模式**：

| 模式 | 行为 |
|------|------|
| `auto` | 自动选择最佳显示 |
| `tmux` | 强制 tmux 多窗格 |
| `in-process` | 单会话文本分离 |

**适用场景**：大型特性并行开发、多模块同时重构、团队协作编码。

---

### 模式选择指南

| 需求 | 推荐模式 |
|------|----------|
| 单一领域深度任务 | **Specialist Agent** |
| 长期维护的文档/配置 | **Self-Evolving Agent** |
| 深度分析/只读调研 | **Research Agent** |
| 多源信息交叉验证 | **Parallel Collaboration** |
| 关键代码质量保证 | **Test Time Compute** |
| 不阻塞主会话的验证 | **Background Agent** |
| 需要文件系统隔离 | **Worktree Isolation** |
| 用户意图自动匹配 | **PROACTIVELY Trigger** |
| 大规模并行开发 | **Agent Teams** |

---

<a id="最佳实践"></a>

## 9. Agent 记忆系统

### 三个作用域

| 作用域 | 位置 | 版本控制 | 共享 | 最佳用途 |
|--------|------|--------|------|----------|
| `user` | `~/.claude/agent-memory/<name>/` | ❌ 否 | ❌ 否 | 跨项目个人知识（推荐默认） |
| `project` | `.claude/agent-memory/<name>/` | ✅ 是 | ✅ 是 | 项目特定的团队知识 |
| `local` | `.claude/agent-memory-local/<name>/` | ❌ 否 | ❌ 否 | 项目特定的个人笔记 |

### 自动注入机制

Agent 启动时自动读取 `MEMORY.md` 的**前 200 行**并注入系统提示。Agent 可自由读写该目录中的所有文件。

```
~/.claude/agent-memory/code-reviewer/
├── MEMORY.md                 # 前 200 行自动注入
├── react-patterns.md         # Agent 可读写
├── security-checklist.md     # Agent 可读写
└── archived-learnings.json   # Agent 可读写
```

### 与其他记忆系统的对比

| 系统 | 谁写 | 谁读 | 用途 |
|------|------|------|------|
| **CLAUDE.md** | 人工维护 | 主 Claude + 所有 Agent | 全局项目规则 |
| **自动记忆** | 主 Claude | 主 Claude 仅 | 会话上下文学习 |
| **Agent 记忆** | Agent 自动 | 特定 Agent 仅 | Agent 特定的跨会话学习 |

详见：[Agent Memory 报告](claude-agent-memory.md)

---

## 10. 最佳实践

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
| 预加载太多 Skills | 启动上下文过大，精选 3-5 个最关键的 |
| Agent 记忆无限增长 | 定期清理，按话题分离文件 |
| 不设置 color | 难以在日志中追踪，为每个 Agent 分配不同颜色 |
| 嵌套 Agent 过深 | 限制嵌套层数不超过 3 层 |

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
