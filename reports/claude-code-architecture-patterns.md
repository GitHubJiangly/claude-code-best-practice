# Claude Code 核心架构设计模式

从 Agentic Loop 到 Context Engineering — 系统架构哲学与 8 大设计模式

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## Table of Contents

1. [Agentic Loop — 自治代理循环](#agentic-loop)
2. [Context Engineering — 上下文工程](#context-engineering)
3. [Configuration Hierarchy — 分层配置](#configuration-hierarchy)
4. [Memory Loading — 记忆加载](#memory-loading)
5. [Extension Architecture — 扩展性架构](#extension-architecture)
6. [Permission & Safety — 权限与安全](#permission-and-safety)
7. [Checkpointing & Rollback — 检查点与回滚](#checkpointing-and-rollback)
8. [Orchestration Patterns — 编排模式](#orchestration-patterns)
9. [Agent Invocation Patterns — 代理调用模式](#agent-invocation-patterns)
10. [核心设计原则总结](#核心设计原则总结)
11. [Sources](#sources)

---

<a id="agentic-loop"></a>

## 1. Agentic Loop — 自治代理循环

Claude Code 的根基是一个**自治代理循环**，而非单次 API 调用。

### 循环结构

```
用户输入
  ↓
┌─────────────────────────────────┐
│         Agentic Loop            │
│                                 │
│  Claude 推理 → 决定下一步行动   │
│       ↓                         │
│  工具调用（Read/Edit/Bash/...） │
│       ↓                         │
│  工具结果反馈到上下文           │
│       ↓                         │
│  Claude 再次推理 → 决定继续/停止│
│       ↓                         │
│  [循环直到任务完成或用户中断]   │
└─────────────────────────────────┘
  ↓
输出结果给用户
```

### Tool-Use 优先

Claude Code 的所有操作都围绕**工具调用**构建，而非知识检索：

| 工具类别 | 工具 | 用途 |
|----------|------|------|
| **文件读取** | Read, Glob, Grep | 搜索和读取代码 |
| **文件修改** | Edit, Write, NotebookEdit | 编辑和创建文件 |
| **系统执行** | Bash | 运行命令、测试、构建 |
| **网络访问** | WebFetch, WebSearch | 获取文档和搜索信息 |
| **代理编排** | Agent, Skill | 调用子代理和技能 |
| **用户交互** | AskUserQuestion, TodoWrite | 询问用户和管理任务 |
| **外部集成** | mcp__* | MCP 服务器提供的工具 |

**关键设计**：工具执行结果直接进入模型上下文，支持递归决策 — Claude 看到结果后决定下一步，而非预先规划所有步骤。

### Agentic Search > RAG

Boris Cherny 的架构决策：Claude Code 尝试过向量数据库（RAG），但最终放弃，改用 **agentic search**（glob + grep）：

| 方案 | 问题 |
|------|------|
| RAG（向量数据库） | 代码频繁变更导致索引过时；权限管理复杂 |
| Agentic Search | Claude 自主使用 Glob + Grep 搜索，始终读取最新代码 |

---

<a id="context-engineering"></a>

## 2. Context Engineering — 上下文工程

Claude Code 的上下文管理是整个系统最精密的设计之一。

### 动态加载策略

```
会话启动
├─ 加载基础系统提示（~269 tokens）
├─ 注入所有可用工具描述
├─ 加载 CLAUDE.md（祖先 → 当前目录，上行加载）
├─ 加载 .claude/rules/ 中的无条件 Rules
├─ 注入 skills 列表及描述（用于自动发现）
├─ 注入 agents 列表及描述（用于自动调用）
├─ 条件加载 MCP 工具
│   ├─ 工具定义 < 10% 上下文 → 全部加载
│   └─ 工具定义 > 10% 上下文 → 启用 Tool Search 延迟加载
└─ 准备就绪

工作期间
├─ 读取文件时 → 懒加载下行 CLAUDE.md + 匹配的条件 Rules
├─ 调用 Skill 时 → 注入完整 Skill 内容
├─ 启动 Agent 时 → 预加载 skills 完整注入到 Agent 上下文
├─ 工具调用结果 → 实时反馈到上下文
└─ 达到 ~95% 上下文 → 触发自动压缩（autocompact）

手动优化
├─ /compact [焦点指令] → 带焦点的定向压缩
├─ /clear → 完全重置上下文
└─ /context → 可视化上下文使用情况
```

### Prompt Caching

Thariq 的文章揭示了 Claude Code 的缓存策略：

```
系统提示（不变部分）→ 缓存命中率高 → 节省 ~90% 输入成本
CLAUDE.md + Rules   → 会话内稳定 → 可缓存
工具调用结果        → 每次不同 → 不可缓存
```

**缓存 TTL**：5 分钟。超过 300 秒未命中则需重新读取完整上下文。

### Tool Search（MCP 优化）

当 MCP 工具定义超过 10K tokens 时，Claude Code 启用搜索模式：
- 不加载所有工具描述
- 仅加载 3-5 个最常用工具
- 其他工具按需通过语义搜索发现

### Context Compaction

| 触发方式 | 时机 | 行为 |
|----------|------|------|
| 自动压缩 | ~95% 上下文（可通过 `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 调整） | 保留核心对话，总结中间步骤 |
| 手动 `/compact` | 建议在 50% 时执行 | 支持焦点指令定向压缩 |
| `/clear` | 切换任务时 | 完全重置上下文 |

**Agent Dumb Zone**：上下文超过 50% 后模型质量开始下降。Boris 建议在 50% 时手动 `/compact`。

---

<a id="configuration-hierarchy"></a>

## 3. Configuration Hierarchy — 分层配置

### 6 层优先级架构

```
优先级从高到低：

1. Managed Settings（组织强制）
   ├─ managed-settings.json（服务器托管）
   ├─ managed-settings.d/（drop-in 目录）
   └─ OS 级（macOS plist / Windows Registry）
   → ❌ 不可被下级覆盖

2. CLI 命令行参数
   → --model sonnet, --permission-mode auto
   → 仅当前会话有效

3. .claude/settings.local.json（个人项目设置）
   → git-ignored，个人覆盖团队配置

4. .claude/settings.json（团队共享设置）
   → 纳入 git，团队统一标准

5. ~/.claude/settings.json（全局个人默认）
   → 跨所有项目的个人偏好

6. 系统/环境变量
   → env 字段中的变量，最低优先级
```

### 关键规则

| 规则 | 说明 |
|------|------|
| **deny 最高权** | deny 规则不能被任何下级覆盖 |
| **数组合并** | 权限规则在所有级别合并，不是替换 |
| **确定性行为放 settings** | 如 `attribution.commit` 放 settings.json，不放 CLAUDE.md |

### Hooks 配置也遵循分层

```
hooks-config.local.json（个人覆盖，最高优先）
  ↓ fallback
hooks-config.json（团队共享）
  ↓ fallback
默认行为：hook 启用
```

---

<a id="memory-loading"></a>

## 4. Memory Loading — 记忆加载

### CLAUDE.md 双向加载模式

```
文件系统根
  ├─ CLAUDE.md          ← 上行加载（Ancestor）
  ├─ ...
  └─ /myproject/
      ├─ CLAUDE.md      ← 启动时加载（当前目录）
      ├─ frontend/
      │   └─ CLAUDE.md  ← 下行懒加载（Descendant）
      └─ backend/
          └─ CLAUDE.md  ← 下行懒加载（Descendant）
```

| 加载方向 | 时机 | 行为 |
|----------|------|------|
| **上行（Ancestor）** | 会话启动 | 从当前目录向上到根，加载所有 CLAUDE.md |
| **下行（Descendant）** | 操作文件时 | 仅在读取子目录文件时懒加载对应 CLAUDE.md |
| **兄弟（Sibling）** | 永不 | 在 frontend/ 工作时不会加载 backend/CLAUDE.md |

### Rules 条件加载

`.claude/rules/*.md` 通过 `paths` 前置字段实现 Glob 匹配加载：

```
Claude 读写文件 X
  ├─ 遍历 .claude/rules/ 中所有 Rule
  ├─ 无 paths 字段 → 启动时已加载（无条件）
  ├─ 有 paths 字段 → Glob 匹配文件 X 的路径
  │   ├─ 匹配成功 → 注入 Rule 内容
  │   └─ 匹配失败 → 跳过
  └─ 继续操作
```

### 4 层记忆体系

| 层级 | 位置 | 谁写 | 谁读 | 生命周期 |
|------|------|------|------|----------|
| **CLAUDE.md** | 项目目录 | 人工维护 | 主 Claude + 所有 Agent | 项目级持久 |
| **Rules** | `.claude/rules/` | 人工维护 | 主 Claude（按 Glob 匹配） | 项目级持久 |
| **自动记忆** | `~/.claude/projects/<hash>/memory/` | 主 Claude 自动 | 主 Claude 仅 | 用户级持久 |
| **Agent 记忆** | `~/.claude/agent-memory/<name>/` | Agent 自动 | 特定 Agent 仅 | 跨会话持久 |

---

<a id="extension-architecture"></a>

## 5. Extension Architecture — 扩展性架构

Claude Code 的扩展体系由 5 个独立可组合的机制构成：

### 扩展机制对比

```
┌──────────────────────────────────────────────────────────┐
│                  Claude Code 扩展体系                     │
│                                                          │
│  Commands ─── 用户触发的工作流编排入口                    │
│      ↕                                                   │
│  Agents ──── 隔离上下文中的自主角色                       │
│      ↕                                                   │
│  Skills ──── 可配置的领域知识包                           │
│      ↕                                                   │
│  Hooks ───── 代理循环外的事件驱动处理器                   │
│      ↕                                                   │
│  MCP ─────── 连接外部工具/数据库/API 的桥梁              │
└──────────────────────────────────────────────────────────┘
```

### Progressive Disclosure（渐进式信息披露）

Claude Code 的核心设计哲学之一 — 信息按需逐层展开：

```
Level 1: 描述（description）
  └─ Claude 看到 skill/agent 的一行描述，判断是否需要

Level 2: 主文件（SKILL.md / agent.md）
  └─ 调用时注入完整指令

Level 3: 子目录（references/ scripts/ examples/）
  └─ Agent 按需深入读取参考资料和示例

Level 4: 动态内容（!`command` 嵌入）
  └─ 运行时执行 shell 命令，注入动态输出
```

### Auto-Invocation 规则

| 机制 | 自动调用 | 触发条件 | 防止方式 |
|------|----------|----------|----------|
| **Agent** | ✅ | description 语义匹配 | 移除 PROACTIVELY |
| **Skill** | ✅ | description 语义匹配 | `disable-model-invocation: true` |
| **Command** | ❌ | 需用户 `/命令名` | N/A |
| **Hook** | ✅ | 事件触发（自动） | `disabled: true` 或 matcher 限制 |
| **Rule** | ✅ | Glob 模式匹配 | 精确 paths 限制 |

### Context Forking（上下文分叉）

Skills 和 Commands 支持 `context: fork`，在隔离子代理中运行：

```
主上下文                    Fork 上下文
├─ 用户对话历史              ├─ 全新上下文
├─ 调用 Skill(fork)  ──→    ├─ 注入 Skill 内容
├─ [等待结果]                ├─ 执行多步工具调用
├─ 接收最终结果 ←──          ├─ 中间步骤不可见
└─ 继续对话                  └─ 上下文销毁
```

**效果**：主上下文只看到最终结果，不被中间步骤污染。

---

<a id="permission-and-safety"></a>

## 6. Permission & Safety — 权限与安全

### Permission Modes 光谱

```
← 更安全                                    更自主 →

plan ──── default ──── acceptEdits ──── auto ──── bypassPermissions
 │           │              │            │              │
只读      需提示每个     自动接受      AI 分类器      全部跳过
         工具调用       文件编辑      判断安全性      ⚠️ 危险
```

### 权限三元组求值

```json
{
  "allow": ["Bash(npm run *)", "Edit(src/**)"],
  "ask":   ["Bash(rm *)", "Bash(git push *)"],
  "deny":  ["Read(.env)", "Bash(curl *)"]
}
```

**求值顺序**：deny（最高）→ ask → allow（最低）。第一个匹配的规则生效。

### Auto Mode 安全机制

```
用户启用 Auto Mode
  ↓
Claude 请求执行工具
  ↓
后台安全分类器评估
  ├─ 安全 → 自动批准
  ├─ 可疑 → 提示用户确认
  └─ 危险 → 自动拒绝
  
安全阀：3 次连续拒绝 或 20 次总拒绝 → 回退到手动提示模式
```

### Sandbox 隔离

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "denyWrite": ["./secrets/"],
      "allowRead": ["./config.json"]
    },
    "network": {
      "allowedDomains": ["*.internal.example.com"]
    }
  }
}
```

内部数据：沙箱减少 **84%** 的权限提示。

---

<a id="checkpointing-and-rollback"></a>

## 7. Checkpointing & Rollback — 检查点与回滚

### 自动检查点

Claude Code 自动跟踪所有文件编辑，支持细粒度回滚：

```
Turn 1: 编辑 src/auth.ts
  ├─ [自动保存检查点 1]
Turn 2: 编辑 src/api.ts + src/types.ts
  ├─ [自动保存检查点 2]
Turn 3: 编辑 src/auth.ts（再次）
  ├─ [自动保存检查点 3]

用户: Esc Esc 或 /rewind
  ├─ 选择回退到检查点 1、2 或 3
  ├─ 代码回滚到选定状态
  └─ 对话也回退到对应点
```

### 回滚工具

| 工具 | 用途 |
|------|------|
| `Esc Esc` | 快速回退到上一个检查点 |
| `/rewind` | 交互式选择回退点 |
| `/diff` | 查看每个 turn 的文件变更 |
| `/branch` | 创建对话分支（保留当前状态，开始新方向） |

### Git 集成策略

| 策略 | 说明 |
|------|------|
| **Squash Merge** | 每功能一个 commit，保持线性历史 |
| **小 PR** | Boris 的 p50 是 118 行（一天 141 个 PR） |
| **Worktree 并行** | 5 个 worktree = 5 个并行开发分支 |

---

<a id="orchestration-patterns"></a>

## 8. Orchestration Patterns — 编排模式

### 统一工作流阶段

所有主流开发工作流都收敛到同一模式：

```
Research（研究）→ Plan（规划）→ Execute（执行）→ Review（审查）→ Ship（发布）
```

### Command → Agent → Skill 三层编排

```
┌─────────────────────────────────────────────┐
│  Command 层（编排）                          │
│  ├─ 用户交互（AskUserQuestion）             │
│  ├─ 流程控制（顺序/并行调度）               │
│  └─ 结果汇总                                │
│         │                                    │
│    Agent 工具                                │
│         ↓                                    │
│  Agent 层（执行）                            │
│  ├─ 隔离上下文                               │
│  ├─ 预加载 Skills（领域知识）               │
│  ├─ 自主决策和工具调用                       │
│  └─ 仅返回最终结果                           │
│         │                                    │
│    Skill 工具                                │
│         ↓                                    │
│  Skill 层（能力）                            │
│  ├─ 领域知识注入                             │
│  ├─ 渐进式信息披露                           │
│  └─ 可预加载或动态调用                       │
└─────────────────────────────────────────────┘
```

### 并行编排模式

```
Command 编排器
  │
  ├─ 并行启动（同一消息中多个 Agent 调用）
  │    ├─ Agent A（研究方向 1）
  │    ├─ Agent B（研究方向 2）
  │    └─ Agent C（研究方向 3）
  │
  ├─ 等待全部完成
  │
  ├─ 交叉验证 + 合并结果
  │
  └─ 生成统一报告
```

### Agent Teams 并行

```bash
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude
```

```
主会话（编排者）
  ├─ tmux pane 1: Agent A（前端开发）
  ├─ tmux pane 2: Agent B（后端开发）
  └─ tmux pane 3: Agent C（测试编写）
  
共享任务队列 → 独立上下文 → 协调工作
```

### 扩展并行：本地 + 云端

```
本地终端
├─ 5 个 git worktree × 5 个 Claude 会话

claude.ai/code（云端）
├─ 5-10 个 web 会话（机器关了也能跑）

/remote-control → 从手机/平板继续
/teleport → 将 web 会话拉回本地
```

---

<a id="agent-invocation-patterns"></a>

## 9. Agent Invocation Patterns — 代理调用模式

从本仓库的实际实现中，可以提炼出 **6 种典型的 Agent 调用模式**。这些模式不是互斥的，实际使用中经常组合。

### 模式总览

| # | 模式 | 触发者 | 上下文 | 本仓库实例 |
|---|------|--------|--------|-----------|
| 1 | Command → Agent 显式编排 | Command 代码 | 隔离 | weather-orchestrator → weather-agent |
| 2 | PROACTIVELY 自动触发 | Claude 自动判断 | 隔离 | weather-agent、presentation-curator |
| 3 | Rule 强制委派 | Glob 匹配规则 | 隔离 | presentation.md Rule → presentation-curator |
| 4 | 并行 Agent 扇出 | Command 同一消息 | 多个隔离 | workflow-concepts → 2 个 Agent 并行 |
| 5 | Agent + 预加载 Skill | Agent 启动时 | Skill 注入 Agent | weather-agent + weather-fetcher |
| 6 | CLI 直接启动 | 用户命令行 | Agent 主导会话 | `claude --agent=weather-agent` |

---

### 9.1 Command → Agent 显式编排

**定义**：Command 在工作流的特定步骤中，通过 Agent 工具显式调用特定 Agent。这是最常见、最可控的调用模式。

**执行流程**：

```
用户输入 /weather-orchestrator
  │
  ├─ Command 执行 Step 1: AskUserQuestion（用户交互）
  │    └─ 用户选择 °C
  │
  ├─ Command 执行 Step 2: 显式调用 Agent
  │    │
  │    │  Agent(
  │    │    subagent_type = "weather-agent",
  │    │    description   = "Fetch Dubai weather data",
  │    │    prompt        = "Fetch temperature in Celsius...",
  │    │    model         = "haiku"
  │    │  )
  │    │
  │    ├─ 创建隔离上下文
  │    ├─ weather-agent 自主执行（WebFetch API）
  │    ├─ 返回: "24°C"
  │    └─ 隔离上下文销毁
  │
  ├─ Command 接收结果，继续 Step 3: 调用 Skill
  │    └─ Skill("weather-svg-creator") → 生成 SVG
  │
  └─ Command 汇总输出给用户
```

**本仓库实例**：

| Command | 调用的 Agent | 调用方式 |
|---------|-------------|----------|
| `/weather-orchestrator` | `weather-agent` | `subagent_type: weather-agent` |
| `/workflows:development-workflows` | `development-workflows-research-agent` | 多实例并行 |
| `/workflows:best-practice:workflow-concepts` | `workflow-concepts-agent` + `claude-code-guide` | 双 Agent 并行 |

**关键约束**：
- 必须使用 Agent 工具（或别名 Task），**不能**通过 Bash 调用
- Command 可以指定 `model` 覆盖 Agent 定义中的默认模型
- Command 等待 Agent 完成后才继续下一步

**适用场景**：
- 多步工作流中需要隔离执行的环节
- 需要精确控制调用时机和参数的场景
- Command 作为编排者协调多个 Agent 和 Skill

---

### 9.2 PROACTIVELY 自动触发

**定义**：Agent 的 `description` 字段中包含 `PROACTIVELY` 关键词，Claude 根据用户输入的语义自动判断是否调用该 Agent，无需用户显式触发。

**执行流程**：

```
用户输入: "迪拜现在多少度？"
  │
  ├─ Claude 扫描所有已注册 Agent 的 description
  │    ├─ time-agent: "display the current time in PKT"
  │    │   └─ 语义匹配 ❌（问的是温度不是时间）
  │    │
  │    ├─ weather-agent: "PROACTIVELY when you need to fetch
  │    │   weather data for Dubai, UAE"
  │    │   └─ 语义匹配 ✅（迪拜 + 温度 = 天气数据）
  │    │
  │    └─ presentation-curator: "PROACTIVELY...presentation slides"
  │        └─ 语义匹配 ❌（无关演示文稿）
  │
  ├─ Claude 自动生成 Agent 工具调用
  │    └─ Agent(subagent_type="weather-agent", prompt="用户问迪拜温度...")
  │
  ├─ weather-agent 在隔离上下文中执行
  │    └─ 返回: "24°C"
  │
  └─ Claude 将结果整合到回复中
      └─ "迪拜当前温度是 24°C。"
```

**本仓库的 PROACTIVELY Agent**：

```yaml
# weather-agent — 天气触发
description: Use this agent PROACTIVELY when you need to fetch
  weather data for Dubai, UAE. This agent fetches real-time
  temperature from Open-Meteo using its preloaded weather-fetcher skill.

# presentation-curator — 演示文稿触发
description: PROACTIVELY use this agent whenever the user wants to
  update, modify, or fix the presentation slides, structure,
  styling, or weights
```

**description 写作对比**：

```yaml
# ✅ 好的 — 明确触发条件和领域
description: |
  PROACTIVELY fetch real-time weather data when the user asks about:
  - Current temperature in Dubai, UAE
  - Weather conditions, humidity, wind speed

# ❌ 差的 — 太模糊，Claude 无法精确匹配
description: "Weather agent"

# ❌ 差的 — 没有 PROACTIVELY，不会自动触发
description: "Fetches weather data for Dubai"
```

**与 Skill 自动触发的区别**：

| 特性 | Agent 自动触发 | Skill 自动触发 |
|------|---------------|---------------|
| 上下文 | 隔离（独立窗口） | 当前上下文（除非 fork） |
| 中间步骤 | 对用户不可见 | 对用户可见 |
| 防止方式 | 移除 PROACTIVELY | `disable-model-invocation: true` |

**适用场景**：
- 特定领域的自动委派（天气、时间、数据查询）
- 用户不需要知道 Agent 存在的透明集成
- 触发条件清晰、领域边界明确的专用 Agent

---

### 9.3 Rule 强制委派

**定义**：通过 `.claude/rules/` 中的 Glob 匹配规则，当 Claude 操作特定文件时，强制将操作委派给指定 Agent。这不是建议，而是**强制约束**。

**执行流程**：

```
用户: "帮我修改第 3 张幻灯片的标题"
  │
  ├─ Claude 准备读取 presentation/index.html
  │
  ├─ Glob 匹配检查
  │    └─ .claude/rules/presentation.md
  │       └─ Glob: presentation/**  →  ✅ 匹配！
  │
  ├─ Rule 内容注入上下文:
  │    "Any request to update, modify, or fix the presentation
  │     MUST be handled by the presentation-curator agent.
  │     Always delegate via the Task tool — never edit directly."
  │
  ├─ Claude 遵循 Rule，不直接编辑，而是委派:
  │    Agent(
  │      subagent_type = "presentation-curator",
  │      description   = "修改第 3 张幻灯片标题",
  │      prompt        = "将第 3 张幻灯片标题改为..."
  │    )
  │
  ├─ presentation-curator 在隔离上下文中执行
  │    ├─ 预加载 3 个 Skills（框架/结构/样式）
  │    ├─ 修改幻灯片
  │    ├─ 验证完整性（编号、level 转换、样式）
  │    ├─ 自我进化（更新 Skills + Learnings）
  │    └─ 返回修改报告
  │
  └─ Claude 展示结果给用户
```

**本仓库实现**：

```markdown
# .claude/rules/presentation.md

# Glob: presentation/**

## Delegation Rule

Any request to update, modify, or fix the presentation
MUST be handled by the `presentation-curator` agent.
Always delegate presentation work to this agent via the
Task tool — never edit the presentation directly.
```

**为什么用 Rule 而非 PROACTIVELY**：

| 方面 | PROACTIVELY | Rule 强制委派 |
|------|-------------|--------------|
| 强制程度 | 建议性（Claude 可能跳过） | 强制性（注入为指令） |
| 触发条件 | 语义匹配用户意图 | Glob 匹配文件路径 |
| 绕过风险 | Claude 可能判断不需要 Agent | 只要操作匹配文件就触发 |
| 适用场景 | 通用领域委派 | 关键资源的严格保护 |

**适用场景**：
- 需要专家 Agent 才能安全修改的复杂资源（演示文稿、配置文件）
- 防止直接编辑导致结构破坏的场景
- 团队级强制规范（所有人都必须通过 Agent 操作）

---

### 9.4 并行 Agent 扇出

**定义**：Command 在**同一消息**中同时启动多个 Agent，各自在独立上下文中并行工作，全部完成后交叉验证并合并结果。

**执行流程**：

```
/workflows:best-practice:workflow-concepts 10
  │
  ├─ Phase 0: 并行启动（同一消息，关键！）
  │    │
  │    ├─ Agent 调用 1:
  │    │    Agent(
  │    │      subagent_type = "workflow-concepts-agent",
  │    │      model = "opus",
  │    │      prompt = "分析 README CONCEPTS 与官方文档的偏差..."
  │    │    )
  │    │    ├─ 读取本地 README.md
  │    │    ├─ WebFetch 官方文档 + Changelog
  │    │    └─ 返回: 结构化偏差报告
  │    │
  │    └─ Agent 调用 2（同时发出）:
  │         Agent(
  │           subagent_type = "claude-code-guide",
  │           prompt = "研究最新 Claude Code 特性完整列表..."
  │         )
  │         ├─ WebSearch 最新特性
  │         ├─ 获取版本号和日期
  │         └─ 返回: 独立特性清单
  │
  │    [两个 Agent 并行执行，互不干扰]
  │
  ├─ Phase 1: 等待两者完成
  │
  ├─ Phase 2: 交叉验证
  │    ├─ Agent A 发现的 vs Agent B 发现的
  │    ├─ 标记矛盾点（让用户决定）
  │    ├─ 合并互补信息
  │    └─ 标记 NEW / RECURRING / RESOLVED
  │
  ├─ Phase 2.5-2.7: 追加 Changelog + 更新 Badge + 验证 URL
  │
  └─ Phase 3: 生成统一报告 + 询问用户执行哪些修复
```

**本仓库的并行模式实例**：

| Command | Agent A | Agent B | 合并策略 |
|---------|---------|---------|----------|
| `workflow-concepts` | `workflow-concepts-agent` (opus) | `claude-code-guide` (haiku) | 交叉验证 + 矛盾标记 |
| `workflow-claude-settings` | `workflow-claude-settings-agent` | `claude-code-guide` | 同上 |
| `development-workflows` | `research-agent` 实例 1 | `research-agent` 实例 2+ | 按仓库分配 |

**为什么用两个 Agent 而非一个**：

| 单 Agent | 双 Agent 并行 |
|----------|--------------|
| 一个视角，可能遗漏 | 两个独立视角，互相补充 |
| 顺序执行，耗时长 | 并行执行，时间减半 |
| 无法交叉验证 | 矛盾点暴露盲区 |
| 上下文可能过载 | 各自上下文独立，更聚焦 |

**关键实现细节**：
- 必须在**同一消息**中发出多个 Agent 调用，否则会顺序执行
- 每个 Agent 可以使用不同模型（如 opus + haiku）
- Command 负责合并逻辑，Agent 只负责各自的研究

**适用场景**：
- 多源信息交叉验证（官方文档 vs 实际实现）
- 大规模研究任务的分片并行
- 需要多视角分析的复杂决策

---

### 9.5 Agent + 预加载 Skill 组合

**定义**：Agent 通过 `skills:` 字段在启动时将 Skill 的**完整内容**注入上下文，Agent 按 Skill 指令自主执行。这是"渐进式信息披露"的核心实现。

**执行流程**：

```
weather-agent 被调用
  │
  ├─ 1. Claude Code 读取 agent 定义
  │    └─ skills: ["weather-fetcher"]
  │
  ├─ 2. 查找 .claude/skills/weather-fetcher/SKILL.md
  │    └─ 读取完整内容（API URL、坐标、参数格式）
  │
  ├─ 3. 创建隔离上下文，注入:
  │    ├─ Agent 定义（Markdown 正文）
  │    ├─ weather-fetcher Skill 完整内容 ← 预加载
  │    └─ 调用者传递的 prompt
  │
  ├─ 4. Agent 启动时已完全理解:
  │    ├─ Open-Meteo API 的 URL 格式
  │    ├─ Dubai 坐标 (25.2048°N, 55.2708°E)
  │    ├─ 温度参数 (temperature_unit)
  │    └─ 响应解析方式
  │
  ├─ 5. Agent 自主执行 WebFetch
  │    └─ 无需额外查询文档，知识已在上下文中
  │
  └─ 6. 返回结果: {temperature: 24, unit: "Celsius"}
```

**预加载 vs 动态调用的对比**：

| 方面 | 预加载（`skills:` 字段） | 动态调用（`Skill()` 工具） |
|------|------------------------|--------------------------|
| **注入时机** | Agent 启动前，一次性 | 执行时按需，每次调用 |
| **上下文位置** | Agent 的系统提示中 | 当前对话上下文中 |
| **适合内容** | 频繁参考的领域规范 | 偶尔查询的参考资料 |
| **上下文成本** | 启动时固定占用 | 调用时临时占用 |
| **本仓库示例** | weather-fetcher → weather-agent | weather-svg-creator → Command 调用 |

**Skill 的渐进式信息披露**：

```
.claude/skills/weather-fetcher/
├── SKILL.md          ← Level 1: 主指令（预加载注入）
├── reference.md      ← Level 2: 详细参考（Agent 按需读取）
└── examples.md       ← Level 3: 示例（Agent 按需读取）
```

Agent 启动时只注入 SKILL.md，如果需要更多细节，Agent 可以自主读取 reference.md 和 examples.md。

**本仓库实例**：

| Agent | 预加载 Skills | 注入的知识 |
|-------|--------------|-----------|
| `weather-agent` | `weather-fetcher` | Open-Meteo API 调用规范 |
| `presentation-curator` | `vibe-to-agentic-framework` | 演示文稿概念框架 |
| | `presentation-structure` | 幻灯片结构规范 |
| | `presentation-styling` | CSS 样式指南 |

**适用场景**：
- Agent 需要领域专业知识才能正确执行
- 知识内容相对稳定，不需要每次动态获取
- 多个 Agent 可以共享同一个 Skill（不同 Agent 预加载同一 Skill）

---

### 9.6 CLI 直接启动

**定义**：通过 `--agent` 命令行参数将 Agent 作为主会话角色启动，整个 Claude Code 会话由该 Agent 的定义主导。

**执行流程**：

```bash
claude --agent=weather-agent
```

```
终端启动 Claude Code
  │
  ├─ 1. 解析 --agent=weather-agent 参数
  │
  ├─ 2. 读取 .claude/agents/weather-agent.md
  │    ├─ 解析前置字段（model, tools, skills, hooks...）
  │    └─ 读取 Markdown 正文作为系统指令
  │
  ├─ 3. 预加载 skills
  │    └─ weather-fetcher 完整内容注入
  │
  ├─ 4. 如果有 initialPrompt 字段
  │    └─ 自动作为首轮用户输入提交
  │
  ├─ 5. 会话开始
  │    ├─ Agent 定义作为系统指令
  │    ├─ 工具集限制为 agent 定义的 tools
  │    ├─ 权限模式为 agent 定义的 permissionMode
  │    └─ 用户可以直接对话
  │
  └─ 整个会话由 weather-agent 的角色和知识主导
```

**与其他模式的区别**：

| 方面 | CLI 启动 | Command/自动触发 |
|------|----------|-----------------|
| **会话角色** | Agent 是主角色 | Agent 是被调用的子角色 |
| **上下文** | Agent 定义 = 系统提示 | Agent 在隔离子上下文中 |
| **生命周期** | 整个会话 | 单次调用 |
| **用户交互** | 直接与 Agent 对话 | 通过 Command 间接交互 |

**配合 worktree 使用**：

```bash
# 在隔离的 git worktree 中启动 Agent 主导的会话
claude -w --agent=feature-developer

# 多个并行 Agent 会话
claude -w --agent=frontend-agent   # worktree 1
claude -w --agent=backend-agent    # worktree 2
claude -w --agent=test-agent       # worktree 3
```

**适用场景**：
- 专用终端会话（如专门的代码审查终端）
- 长时间运行的特定领域工作
- 配合 worktree 实现多 Agent 并行开发

---

### 模式组合实例

实际使用中，这些模式经常组合。以下是本仓库的两个典型组合：

**组合 1：天气系统（Pattern 1 + 2 + 5）**

```
Pattern 2: PROACTIVELY 自动触发
  └─ 用户提到"迪拜天气" → 自动匹配 weather-agent

Pattern 5: Agent + 预加载 Skill
  └─ weather-agent 启动时注入 weather-fetcher 知识

Pattern 1: Command → Agent 显式编排
  └─ /weather-orchestrator 也可以显式调用 weather-agent
```

**组合 2：文档漂移检测（Pattern 1 + 4 + 5）**

```
Pattern 1: Command 显式编排
  └─ /workflow-concepts 作为入口

Pattern 4: 并行 Agent 扇出
  └─ 同时启动 workflow-concepts-agent + claude-code-guide

Pattern 5: Agent + 预加载 Skill（潜在）
  └─ 各 Agent 可预加载领域知识
```

**组合 3：演示文稿保护（Pattern 2 + 3 + 5）**

```
Pattern 3: Rule 强制委派
  └─ presentation/** 匹配 → 强制委派

Pattern 2: PROACTIVELY 自动触发
  └─ 即使不触发 Rule，提到"演示文稿"也会自动匹配

Pattern 5: Agent + 预加载 Skill
  └─ presentation-curator 预加载 3 个 Skills
```

---

### 模式选择决策树

```
需要调用 Agent？
  │
  ├─ 用户是否需要知道 Agent 存在？
  │    ├─ 否 → Pattern 2: PROACTIVELY 自动触发
  │    └─ 是 → 继续判断
  │
  ├─ 是否需要强制保护特定文件/目录？
  │    ├─ 是 → Pattern 3: Rule 强制委派
  │    └─ 否 → 继续判断
  │
  ├─ 是否需要多个 Agent 并行？
  │    ├─ 是 → Pattern 4: 并行 Agent 扇出
  │    └─ 否 → 继续判断
  │
  ├─ Agent 是否需要领域专业知识？
  │    ├─ 是 → Pattern 5: Agent + 预加载 Skill
  │    └─ 否 → 继续判断
  │
  ├─ 是否是专用终端会话？
  │    ├─ 是 → Pattern 6: CLI 直接启动
  │    └─ 否 → Pattern 1: Command → Agent 显式编排
  │
  └─ 通常组合多个模式使用
```

---

<a id="核心设计原则总结"></a>

## 10. 核心设计原则总结

| 原则 | 体现 |
|------|------|
| **Vanilla 优先** | 简单任务不需要工作流，原生 Claude Code 就够了 |
| **分层覆盖** | 全局 → 项目 → 本地 → CLI，每层可覆盖上层 |
| **懒加载** | CLAUDE.md 下行懒加载、Rules 条件加载、MCP Tool Search |
| **渐进式披露** | Skills 是文件夹，信息按需逐层展开 |
| **上下文隔离** | Agent 在独立上下文运行，保持主上下文干净 |
| **显式胜过隐式** | 权限提示、hook 事件、工具描述都明确可审计 |
| **可回滚** | 自动检查点、/rewind、git worktree 隔离 |
| **可并行** | 5 个本地 worktree + 10 个云端会话 + Agent Teams |
| **模块化可组合** | Commands/Skills/Agents/Hooks/MCP 独立可组合 |
| **团队协作** | 共享 settings.json + CLAUDE.md + rules/，个人用 .local 覆盖 |

---

<a id="sources"></a>

## Sources

- [Claude Code 官方文档](https://code.claude.com/docs)
- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Features Overview](https://code.claude.com/docs/en/features-overview)
- [Prompt Caching Is Everything (Thariq)](https://x.com/trq212/status/2024574133011673516)
- [Session Management & 1M Context (Thariq)](../tips/claude-thariq-tips-16-apr-26.md)
- [Building Claude Code (Boris, Pragmatic Engineer)](https://youtu.be/julbw1JuAz0)
- [Agents vs Commands vs Skills 对比](claude-agent-command-skill.md)
- [Advanced Tool Use 报告](claude-advanced-tool-use.md)
- [Global vs Project Settings 报告](claude-global-vs-project-settings.md)
- [编排工作流详情](../orchestration-workflow/orchestration-workflow.md)
