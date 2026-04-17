# Claude Memory 系统深度分析

四层记忆体系 — 从会话上下文到跨项目持久知识的完整内存架构

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## Table of Contents

1. [Memory 系统概览](#memory-系统概览)
2. [CLAUDE.md 文件加载机制](#claudemd-文件加载机制)
3. [Rules 懒加载机制](#rules-懒加载机制)
4. [Agent Memory 持久记忆](#agent-memory-持久记忆)
5. [Auto Memory 自动记忆](#auto-memory-自动记忆)
6. [四层记忆体系对比](#四层记忆体系对比)
7. [完整执行流程](#完整执行流程)
8. [本仓库实现示例](#本仓库实现示例)
9. [最佳实践](#最佳实践)
10. [Sources](#sources)

---

<a id="memory-系统概览"></a>

## 1. Memory 系统概览

Claude Code 拥有四种相互补充的记忆机制，覆盖从临时会话到永久跨项目知识的全谱：

| 记忆类型 | 谁写 | 谁读 | 持久化 | 作用域 |
|---------|------|------|--------|--------|
| **CLAUDE.md** | 人工维护 | 主 Claude + 所有 Agent | ✅ 是 | 项目/全局 |
| **Rules（.claude/rules/）** | 人工维护 | 主 Claude（条件加载） | ✅ 是 | 文件路径匹配时 |
| **Auto Memory** | 主 Claude 自动 | 主 Claude 仅 | ✅ 是 | 每项目/每用户 |
| **Agent Memory** | Agent 自动 | 特定 Agent 仅 | ✅ 是 | user/project/local |

**核心设计原则**：上下文按需加载，避免无关内容膨胀上下文窗口。

---

<a id="claudemd-文件加载机制"></a>

## 2. CLAUDE.md 文件加载机制

### 2.1 双向加载机制

Claude Code 使用两种不同的机制加载 `CLAUDE.md` 文件：

```
启动时
  ├─ 向上扫描（Ancestor Loading）← 立即加载
  │    /home/user/CLAUDE.md
  │    /home/user/projects/CLAUDE.md
  │    /home/user/projects/myapp/CLAUDE.md  ← 当前目录
  │
  └─ 向下懒加载（Descendant Lazy Loading）← 按需加载
       /home/user/projects/myapp/frontend/CLAUDE.md  ← 访问 frontend/ 时加载
       /home/user/projects/myapp/backend/CLAUDE.md   ← 访问 backend/ 时加载
```

#### 向上加载（Ancestor Loading）

启动 Claude Code 时，它从**当前工作目录向上**遍历目录树直到文件系统根目录，加载沿途找到的所有 `CLAUDE.md` 文件。**启动时立即全部加载**。

#### 向下懒加载（Descendant Lazy Loading）

子目录中的 `CLAUDE.md` **不会在启动时加载**，只有当 Claude 在会话中读取那些子目录中的文件时，才会触发对应 `CLAUDE.md` 的懒加载。

### 2.2 Monorepo 加载场景

**典型 Monorepo 结构**：

```
/mymonorepo/
├── CLAUDE.md          # 根级指令（所有组件共享）
├── frontend/
│   └── CLAUDE.md      # 前端专属指令
├── backend/
│   └── CLAUDE.md      # 后端专属指令
└── api/
    └── CLAUDE.md      # API 专属指令
```

**场景一：从根目录启动**（`cd /mymonorepo && claude`）

| 文件 | 启动时加载 | 原因 |
|------|-----------|------|
| `/mymonorepo/CLAUDE.md` | ✅ 是 | 当前工作目录 |
| `/mymonorepo/frontend/CLAUDE.md` | ❌ 否（懒加载） | 访问 frontend/ 文件时才加载 |
| `/mymonorepo/backend/CLAUDE.md` | ❌ 否（懒加载） | 访问 backend/ 文件时才加载 |
| `/mymonorepo/api/CLAUDE.md` | ❌ 否（懒加载） | 访问 api/ 文件时才加载 |

**场景二：从子目录启动**（`cd /mymonorepo/frontend && claude`）

| 文件 | 启动时加载 | 原因 |
|------|-----------|------|
| `/mymonorepo/CLAUDE.md` | ✅ 是 | 祖先目录，向上加载 |
| `/mymonorepo/frontend/CLAUDE.md` | ✅ 是 | 当前工作目录 |
| `/mymonorepo/backend/CLAUDE.md` | ❌ 否 | 兄弟目录，永不加载 |
| `/mymonorepo/api/CLAUDE.md` | ❌ 否 | 兄弟目录，永不加载 |

### 2.3 四条核心规则

1. **祖先目录始终在启动时加载** — 向上遍历，所有祖先 CLAUDE.md 都被加载
2. **后代目录懒加载** — 子目录 CLAUDE.md 只在访问相关文件时才加载
3. **兄弟目录永不加载** — 同级其他分支的 CLAUDE.md 不会被加载
4. **全局 CLAUDE.md** — `~/.claude/CLAUDE.md` 对所有 Claude Code 会话生效

### 2.4 CLAUDE.md 文件层级

```
~/.claude/CLAUDE.md          # 全局：所有项目、所有会话
  ↓
/project/CLAUDE.md           # 项目：团队共享（git 跟踪）
  ↓
/project/CLAUDE.local.md     # 本地：个人覆盖（git 忽略）
  ↓
/project/subdir/CLAUDE.md    # 子目录：懒加载（路径相关）
```

---

<a id="rules-懒加载机制"></a>

## 3. Rules 懒加载机制

### 3.1 两种定义格式

`.claude/rules/` 目录下的 Markdown 文件定义条件加载规则：

**新格式（YAML 前置字段）**：
```yaml
---
paths:
  - "presentation/**"
---

## Delegation Rule
Any request to modify the presentation MUST be handled by the presentation-curator agent.
```

**旧格式（文件顶部 Glob 注释）**：
```markdown
# Glob: **/*.md

## Documentation Standards
- Keep files focused and concise
```

### 3.2 触发机制

```
用户请求涉及文件操作
  │
  ├─ Claude 检查 .claude/rules/ 目录
  ├─ 对比文件路径与 rules 的 paths/Glob 模式
  ├─ 匹配 → 规则内容注入当前上下文
  └─ 不匹配 → 规则不加载（节省上下文）
```

本仓库的两条 Rules：

| 文件 | 路径模式 | 作用 |
|------|---------|------|
| `markdown-docs.md` | `**/*.md` | 所有 Markdown 文件的文档规范 |
| `presentation.md` | `presentation/**` | 强制委派演示文稿操作给 Agent |

---

<a id="agent-memory-持久记忆"></a>

## 4. Agent Memory 持久记忆

**引入版本**：Claude Code v2.1.33（2026 年 2 月）

### 4.1 配置方式

在 Agent 定义的 YAML 前置字段中添加 `memory` 字段：

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Write, Edit, Bash
model: sonnet
memory: user          # user / project / local 三选一
---

You are a code reviewer. Review code and update your memory with discovered patterns.
```

### 4.2 三种作用域

| 作用域 | 存储位置 | 版本控制 | 团队共享 | 最佳用途 |
|--------|---------|---------|---------|---------|
| `user` | `~/.claude/agent-memory/<name>/` | ❌ 否 | ❌ 否 | 跨项目个人知识（推荐默认） |
| `project` | `.claude/agent-memory/<name>/` | ✅ 是 | ✅ 是 | 项目特定的团队知识 |
| `local` | `.claude/agent-memory-local/<name>/` | ❌ 否 | ❌ 否 | 项目特定的个人笔记 |

这三个作用域与 Settings 层级完全对应：
- `user` ↔ `~/.claude/settings.json`
- `project` ↔ `.claude/settings.json`
- `local` ↔ `.claude/settings.local.json`

### 4.3 目录结构

```
~/.claude/agent-memory/code-reviewer/     # user scope 示例
├── MEMORY.md                              # 主文件（前 200 行自动注入）
├── react-patterns.md                      # 按主题分离的扩展记忆
├── security-checklist.md                  # Agent 可自由读写
└── archived-learnings.json                # 结构化存储
```

### 4.4 执行流程

```
Agent 启动
  ├─ 1. 读取 MEMORY.md 前 200 行
  ├─ 2. 注入 Agent 系统提示
  ├─ 3. Read/Write/Edit 工具自动启用（无需声明）
  │
  执行过程
  ├─ 4. Agent 可自由读写记忆目录中的任何文件
  ├─ 5. 发现新模式 → 写入 MEMORY.md 或专题文件
  │
  MEMORY.md 超过 200 行时
  └─ 6. Agent 将详情移入主题文件（如 react-patterns.md），MEMORY.md 保留摘要
```

### 4.5 实际用法示例

调用 Agent 时显式提醒使用记忆：

```
# 调用前检查记忆
"Review this PR, and check your memory for patterns you've seen before."

# 调用后更新记忆
"Before starting, review your memory. After completing, update your memory with what you learned."
```

---

<a id="auto-memory-自动记忆"></a>

## 5. Auto Memory 自动记忆

### 5.1 工作原理

Auto Memory 是 Claude Code 主实例的自动学习系统，无需用户手动配置：

```
会话中
  ├─ Claude 注意到重要信息（用户偏好、纠错、项目规律）
  ├─ 判断是否值得跨会话记住
  └─ 写入 ~/.claude/memory/ 对应项目的记忆文件

下次会话
  └─ 自动注入之前保存的相关记忆
```

### 5.2 /memory 命令

用户可以手动管理 Auto Memory：

```bash
# 打开记忆编辑器查看/修改已保存的记忆
/memory
```

### 5.3 与 Agent Memory 的区别

| 特性 | Auto Memory | Agent Memory |
|------|-------------|--------------|
| 写入者 | 主 Claude 自动 | 特定 Agent 自动 |
| 读取者 | 主 Claude 仅 | 对应 Agent 仅 |
| 触发方式 | Claude 自主判断 | Agent 代码主动写入 |
| 管理方式 | `/memory` 命令 | 直接编辑文件 |

---

<a id="四层记忆体系对比"></a>

## 6. 四层记忆体系对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude Memory 体系                            │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│   CLAUDE.md  │    Rules     │ Auto Memory  │  Agent Memory      │
│  项目指令     │  条件规则     │  自动学习     │  Agent 持久知识     │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│ 人工维护      │ 人工维护      │ Claude 自动  │ Agent 自动          │
│ 启动时加载    │ 路径匹配加载  │ 会话间自动    │ Agent 启动时加载     │
│ 所有人读取    │ Claude 读取   │ 主 Claude    │ 仅对应 Agent        │
│ git 跟踪      │ git 跟踪      │ 用户级       │ 3 作用域可选         │
└──────────────┴──────────────┴──────────────┴────────────────────┘
```

### 完整对比表

| 维度 | CLAUDE.md | Rules | Auto Memory | Agent Memory |
|------|-----------|-------|-------------|--------------|
| **写入者** | 人工 | 人工 | 主 Claude | 特定 Agent |
| **读取者** | 所有（主+Agent） | 主 Claude | 主 Claude 仅 | 对应 Agent 仅 |
| **加载时机** | 启动/懒加载 | 路径匹配时 | 每次会话 | Agent 启动时 |
| **持久化** | 文件系统 | 文件系统 | 文件系统 | 文件系统 |
| **版本控制** | 是（通常） | 是 | 否 | 可选 |
| **团队共享** | 是 | 是 | 否 | 可选（project 作用域） |
| **自动增长** | 否 | 否 | 是 | 是 |
| **容量限制** | 无（但影响上下文） | 无 | 无限制 | 200 行自动注入 |

---

<a id="完整执行流程"></a>

## 7. 完整执行流程

### 7.1 Claude Code 启动序列

```
claude 启动
  │
  ├─ 1. 扫描全局记忆
  │    └─ ~/.claude/CLAUDE.md → 注入上下文
  │
  ├─ 2. 向上扫描祖先 CLAUDE.md
  │    └─ /project/CLAUDE.md → 注入上下文
  │    └─ /project/CLAUDE.local.md → 注入上下文（若存在）
  │
  ├─ 3. 加载 Auto Memory
  │    └─ 注入相关的跨会话记忆
  │
  ├─ 4. 扫描 .claude/rules/ 目录
  │    └─ 记录所有规则及其 paths/Glob 模式（暂不注入内容）
  │
  └─ 5. 就绪，等待用户输入
```

### 7.2 会话中的动态加载

```
用户请求（涉及文件操作）
  │
  ├─ 文件路径匹配 .claude/rules/ 某规则
  │    └─ 规则内容 → 注入当前上下文
  │
  ├─ Claude 读取某子目录文件
  │    └─ 该子目录 CLAUDE.md → 懒加载注入上下文
  │
  └─ Claude 判断值得记忆的信息
       └─ 写入 Auto Memory（会话结束后持久化）
```

### 7.3 Agent 启动与记忆序列

```
Agent 被调用
  │
  ├─ 1. 解析 YAML 前置字段
  │    └─ memory: user/project/local → 确定记忆目录
  │
  ├─ 2. 读取记忆
  │    └─ MEMORY.md 前 200 行 → 注入 Agent 系统提示
  │
  ├─ 3. 预加载 Skills（若配置了 skills: 字段）
  │    └─ 技能内容 → 注入 Agent 上下文
  │
  ├─ 4. 创建隔离上下文，开始执行
  │    └─ Agent 可使用 Read/Write/Edit 自由管理记忆文件
  │
  └─ 5. 任务完成
       ├─ 将新发现写入 MEMORY.md 或专题文件
       └─ 返回结果给调用者，隔离上下文销毁
```

---

<a id="本仓库实现示例"></a>

## 8. 本仓库实现示例

### 8.1 weather-agent 的 project 级记忆

```yaml
---
name: weather-agent
memory: project     # 存储于 .claude/agent-memory/weather-agent/
---
```

实际文件：`.claude/agent-memory/weather-agent/MEMORY.md`

Agent 每次获取温度后，将历史读数写入此文件，下次启动时自动注入历史数据，实现跨会话的温度趋势追踪。

### 8.2 presentation-curator 的 Self-Evolution 模式

`presentation-curator` Agent 没有配置 `memory` 字段，而是采用**直接更新自身 Skills** 的方式实现知识持久化：

```
执行后（Step 5: Self-Evolution）
  ├─ 5a. 更新 vibe-to-agentic-framework/SKILL.md
  ├─ 5b. 更新 presentation-structure/SKILL.md
  ├─ 5c. 同步跨文档一致性
  └─ 5d. 追加 Learnings 到自身 Agent 定义文件
```

这是一种特殊的"记忆"模式：**通过更新 Skills 和 Agent 定义来持久化知识**，而非使用 `memory` 字段。

### 8.3 本仓库 CLAUDE.md 结构

```
/project/CLAUDE.md          # 启动时加载，包含：
  ├─ 语言设置（中文响应）
  ├─ 仓库概览
  ├─ 关键组件说明（Weather System、Skill 定义结构）
  ├─ 关键架构模式（子代理编排、配置层级）
  ├─ 工作流最佳实践
  ├─ Git Commit 规则（每文件单独提交）
  └─ 文档规范
```

---

<a id="最佳实践"></a>

## 9. 最佳实践

### CLAUDE.md 编写原则

| 原则 | 说明 |
|------|------|
| **根目录放共享规范** | 编码风格、Commit 格式、PR 模板等全局约定 |
| **组件目录放局部指令** | 框架特定模式、组件架构、局部测试约定 |
| **CLAUDE.local.md 放个人偏好** | 个人工作习惯，添加到 `.gitignore` |
| **全局 CLAUDE.md 放全局偏好** | 适用所有项目的个人规范 |
| **保持简洁** | 每个 CLAUDE.md 控制在 200 行以内，确保可靠执行 |

### Agent Memory 使用建议

| 建议 | 说明 |
|------|------|
| **默认使用 user 作用域** | 跨项目个人知识，不影响团队 |
| **团队知识用 project 作用域** | 存入 git，团队成员共享 Agent 积累的知识 |
| **显式提示 Agent 使用记忆** | 调用时加入"先查看记忆，完成后更新记忆"的指令 |
| **按主题分文件存储** | 避免 MEMORY.md 无限增长，将详细内容分入专题文件 |
| **定期清理过期记忆** | 删除不再适用的模式和规则 |

### 何时用哪种记忆

| 场景 | 推荐记忆方式 |
|------|------------|
| 项目编码规范 | CLAUDE.md（根目录） |
| 子系统特定规则 | CLAUDE.md（子目录，懒加载） |
| 文件类型操作规范 | Rules（Glob 匹配条件加载） |
| Agent 领域知识 | Agent memory（project 作用域） |
| Agent 个人偏好 | Agent memory（user 作用域） |
| 跨会话用户偏好 | Auto Memory（/memory 命令查看） |
| 长期维护的动态知识 | Self-Evolving Agent（更新 Skills） |

### 常见陷阱

| 陷阱 | 解决方案 |
|------|---------|
| CLAUDE.md 过长导致上下文膨胀 | 拆分到子目录，利用懒加载 |
| Agent 记忆不更新 | 在 Agent 提示中显式要求更新记忆 |
| 团队记忆冲突 | project 作用域 + git 管理，像代码一样 review |
| 忘记 local 文件加 gitignore | 检查 `.gitignore` 包含 `CLAUDE.local.md` 和 `.claude/agent-memory-local/` |
| 记忆文件超过 200 行 | 配置 Agent 按主题分离文件，MEMORY.md 只保留摘要 |

---

<a id="sources"></a>

## Sources

- [Claude Code 官方文档 - Memory 机制](https://code.claude.com/docs/en/memory)
- [Claude Code 官方文档 - Sub-agents](https://code.claude.com/docs/en/sub-agents)
- [Agent Memory 报告](claude-agent-memory.md)
- [Claude Memory 最佳实践](../best-practice/claude-memory.md)
- [Boris Cherny on X - CLAUDE.md 加载机制说明](https://x.com/bcherny/status/2016339448863355206)
- [Humanlayer - Writing a good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
- [Claude Code v2.1.33 Release Notes](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- 本仓库实现：`.claude/agent-memory/weather-agent/MEMORY.md`、`presentation-curator.md`（Self-Evolution 模式）
