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
9. [核心设计原则总结](#核心设计原则总结)
10. [Sources](#sources)

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

<a id="核心设计原则总结"></a>

## 9. 核心设计原则总结

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
