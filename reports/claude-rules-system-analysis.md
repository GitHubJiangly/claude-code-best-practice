# Claude Code Rules 系统深度分析

条件式指令加载 — 基于 Glob 模式匹配的上下文感知规则系统

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## Table of Contents

1. [什么是 Rules](#什么是-rules)
2. [Rules vs CLAUDE.md vs Skills 对比](#rules-vs-claudemd-vs-skills-对比)
3. [触发机制与执行流程](#触发机制与执行流程)
4. [两种 Rule 类型](#两种-rule-类型)
5. [前置字段语法](#前置字段语法)
6. [加载优先级体系](#加载优先级体系)
7. [本仓库的 Rules 实现](#本仓库的-rules-实现)
8. [最佳实践](#最佳实践)
9. [Sources](#sources)

---

<a id="什么是-rules"></a>

## 1. 什么是 Rules

Rules 是放置在 `.claude/rules/` 目录下的 Markdown 文件，为 Claude Code 提供**上下文感知的指令注入**。与 CLAUDE.md 每次会话全量加载不同，Rules 可以通过 Glob 模式匹配，仅在 Claude 操作特定文件时才加载到上下文中。

**核心价值**：按需加载、节省上下文空间、保持指令模块化。

---

<a id="rules-vs-claudemd-vs-skills-对比"></a>

## 2. Rules vs CLAUDE.md vs Skills 对比

| 特性 | CLAUDE.md | Rules | Skills |
|------|-----------|-------|--------|
| **位置** | 项目根目录 / 子目录 | `.claude/rules/*.md` | `.claude/skills/<name>/SKILL.md` |
| **加载方式** | 每次会话自动全量加载 | 无条件加载 或 按 Glob 模式匹配加载 | 按需调用（Skill 工具 / 预加载） |
| **适用场景** | 核心约定、构建命令 | 文件类型/目录特定规则 | 任务特定参考材料和可重复工作流 |
| **上下文占用** | 始终占用 | 匹配时才占用 | 调用时才占用 |
| **团队共享** | 纳入 git | 纳入 git | 纳入 git |
| **最佳用途** | 全项目通用指令 | 分领域的条件指令 | 可调用的专业能力 |

---

<a id="触发机制与执行流程"></a>

## 3. 触发机制与执行流程

### 完整触发流程

```
Claude Code 会话启动
  │
  ├─ 1. 加载全局 Rules
  │    └─ ~/.claude/rules/*.md （用户级，所有项目通用）
  │
  ├─ 2. 加载项目无条件 Rules
  │    └─ .claude/rules/*.md 中没有 paths 字段的文件
  │
  ├─ 3. 会话运行中...
  │    │
  │    ├─ Claude 读取/编辑文件 X
  │    │    │
  │    │    ├─ 遍历 .claude/rules/ 中所有带 paths 字段的 Rule
  │    │    ├─ 对文件 X 的路径进行 Glob 模式匹配
  │    │    ├─ 匹配成功 → 该 Rule 内容注入当前上下文
  │    │    └─ 匹配失败 → 跳过，不占用上下文
  │    │
  │    └─ 继续操作其他文件...
  │
  └─ 会话结束
```

### 关键机制

| 机制 | 说明 |
|------|------|
| **无条件加载** | 没有 `paths` 字段的 Rule 在会话启动时立即加载 |
| **条件加载** | 有 `paths` 字段的 Rule 仅在匹配文件被访问时加载 |
| **懒加载** | 条件 Rule 不会预先加载，仅在匹配时触发 |
| **用户级优先项目级** | 用户 Rules 先加载，项目 Rules 后加载（项目级优先级更高） |

---

<a id="两种-rule-类型"></a>

## 4. 两种 Rule 类型

### 4.1 无条件 Rule（全局生效）

没有 `paths` 前置字段，会话启动时自动加载，类似 `.claude/CLAUDE.md`。

```markdown
# Security Requirements

- Never commit secrets or API keys
- Always validate user input
- Use parameterized queries for database access
```

### 4.2 条件 Rule（Glob 匹配生效）

包含 `paths` 前置字段，仅当 Claude 操作匹配文件时加载。

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules

- All API endpoints must include input validation
- Use the standard error response format
- Include OpenAPI documentation comments
```

---

<a id="前置字段语法"></a>

## 5. 前置字段语法

### 新版语法（YAML frontmatter，推荐）

```markdown
---
paths:
  - "**/*.ts"
  - "**/*.tsx"
---

# TypeScript Rules
...
```

### 旧版语法（注释式，本仓库使用）

```markdown
# Glob: **/*.md

## Documentation Standards
...
```

两种语法功能等效。新版 `paths:` 支持多模式列表，更灵活。

### Glob 模式示例

| 模式 | 匹配范围 |
|------|----------|
| `**/*.md` | 所有目录下的 Markdown 文件 |
| `**/*.test.ts` | 所有 TypeScript 测试文件 |
| `src/components/*.tsx` | `src/components/` 下的 React 组件 |
| `presentation/**` | `presentation/` 目录下的所有文件 |
| `**/*.{ts,tsx}` | 所有 TypeScript 和 TSX 文件 |
| `src/api/**/*.ts` | `src/api/` 目录树下的所有 TS 文件 |

---

<a id="加载优先级体系"></a>

## 6. 加载优先级体系

Rules 在 Claude Code 的完整记忆层级中的位置：

```
加载顺序（先加载 = 低优先级，后加载 = 高优先级）：

1. ~/.claude/rules/*.md        ← 用户全局 Rules（最先加载）
2. ~/.claude/CLAUDE.md         ← 用户全局指令
3. .claude/rules/*.md          ← 项目 Rules（覆盖用户级）
4. CLAUDE.md（项目根目录）      ← 项目主指令
5. 子目录 CLAUDE.md            ← 懒加载，操作子目录时加载
6. 条件 Rules（paths 匹配）    ← 匹配文件时动态注入
```

**规则**：
- 用户级 Rules 先加载，项目级后加载 → **项目级优先级更高**
- 冲突时项目级指令覆盖用户级
- 条件 Rules 按需注入，不占用启动时上下文

---

<a id="本仓库的-rules-实现"></a>

## 7. 本仓库的 Rules 实现

本仓库在 `.claude/rules/` 下配置了 2 条 Rule：

### 7.1 markdown-docs.md — 文档标准规范

```markdown
# Glob: **/*.md

## Documentation Standards
- Keep files focused and concise — one topic per file
- Use relative links between docs, not absolute GitHub URLs
- Include back-navigation link at top of best-practice and report docs
- When adding a new concept or report, update the corresponding table in README.md
```

| 属性 | 值 |
|------|-----|
| **Glob 模式** | `**/*.md` |
| **触发条件** | Claude 读写项目中任何 Markdown 文件时 |
| **注入内容** | 文档结构规范、目录约定（best-practice/、reports/ 等）、格式要求 |
| **效果** | 确保所有 Markdown 文件遵循统一标准 |

**触发示例**：
- 编辑 `reports/claude-hooks-system-analysis.md` → 自动注入文档标准
- 创建 `best-practice/new-feature.md` → 自动注入导航链接和表格格式规范
- 修改 `README.md` → 自动注入 badge 和结构约定

### 7.2 presentation.md — 演示文稿委派规则

```markdown
# Glob: presentation/**

## Delegation Rule
Any request to update, modify, or fix the presentation MUST be handled
by the presentation-curator agent.
```

| 属性 | 值 |
|------|-----|
| **Glob 模式** | `presentation/**` |
| **触发条件** | Claude 操作 `presentation/` 目录下任何文件时 |
| **注入内容** | 强制委派指令 — 必须通过 `presentation-curator` agent 处理 |
| **效果** | 防止直接编辑演示文稿导致幻灯片编号、过渡和样式不一致 |

**触发示例**：
- 用户说 "修改第 3 张幻灯片" → Claude 读取 `presentation/index.html` → 触发 Rule → 自动委派给 `presentation-curator` agent

### 7.3 执行流程图

```
用户: "帮我更新 reports/xxx.md 文档"
  │
  ├─ Claude 准备读取/编辑 reports/xxx.md
  │
  ├─ Glob 匹配检查:
  │    ├─ markdown-docs.md 的 **/*.md  → ✅ 匹配！注入文档标准
  │    └─ presentation.md 的 presentation/** → ❌ 不匹配，跳过
  │
  └─ Claude 在文档标准指导下编辑文件
      └─ 自动遵循：相对链接、导航头、层级标题、表格格式

用户: "修改演示文稿的标题"
  │
  ├─ Claude 准备读取 presentation/index.html
  │
  ├─ Glob 匹配检查:
  │    ├─ markdown-docs.md 的 **/*.md  → ❌ 不匹配（不是 .md 文件）
  │    └─ presentation.md 的 presentation/** → ✅ 匹配！注入委派规则
  │
  └─ Claude 不直接编辑，而是委派给 presentation-curator agent
```

---

<a id="最佳实践"></a>

## 8. 最佳实践

| 实践 | 说明 |
|------|------|
| **一文件一主题** | 每个 Rule 文件聚焦一个领域（文档标准、测试规范、安全要求等） |
| **精确 Glob 模式** | 尽量缩小匹配范围，避免不必要的上下文注入 |
| **条件优于无条件** | 能用 `paths` 限定的 Rule 不要无条件加载，节省上下文 |
| **子目录组织** | `.claude/rules/` 支持子目录，大项目可按领域分类 |
| **团队共享** | Rules 文件纳入 git，确保团队成员获得一致的指令 |
| **与 CLAUDE.md 互补** | 全项目通用规则放 CLAUDE.md，文件/目录特定规则放 Rules |
| **避免重复** | Rules 和 CLAUDE.md 之间不要有冲突或重复的指令 |
| **`<important>` 标签** | 关键规则用 `<important if="...">` 包裹，防止被忽略 |

### 何时用 CLAUDE.md vs Rules

| 场景 | 推荐 |
|------|------|
| 构建/测试命令 | CLAUDE.md |
| 提交消息格式 | CLAUDE.md（或 settings.json） |
| TypeScript 代码风格 | Rules（`paths: ["**/*.ts"]`） |
| React 组件规范 | Rules（`paths: ["src/components/**"]`） |
| 测试约定 | Rules（`paths: ["**/*.test.*"]`） |
| API 文档标准 | Rules（`paths: ["src/api/**"]`） |
| 安全要求（全项目） | Rules（无 paths，无条件加载） |

---

<a id="sources"></a>

## Sources

- [Claude Code Memory 文档](https://code.claude.com/docs/en/memory)
- [Claude Code Rules 文档](https://code.claude.com/docs/en/memory#organize-rules-with-clauderules)
- [Claude Code Features Overview — CLAUDE.md vs Rules vs Skills](https://code.claude.com/docs/en/features-overview)
- [Claude Code .claude 目录结构](https://code.claude.com/docs/en/claude-directory)
- 本仓库实现：`.claude/rules/markdown-docs.md`、`.claude/rules/presentation.md`
