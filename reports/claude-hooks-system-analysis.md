# Claude Code Hooks 系统深度分析

事件驱动的跨平台声音通知与生命周期钩子系统

<table width="100%">
<tr>
<td><a href="../">← Back to Claude Code Best Practice</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## Table of Contents

1. [架构概览](#架构概览)
2. [事件执行流水线](#事件执行流水线)
3. [27 个 Hook 事件清单](#27-个-hook-事件清单)
4. [核心处理器 hooks.py](#核心处理器)
5. [配置系统与优先级](#配置系统与优先级)
6. [音频播放系统](#音频播放系统)
7. [Agent 钩子集成](#agent-钩子集成)
8. [特殊命令检测](#特殊命令检测)
9. [日志系统](#日志系统)
10. [容错与安全设计](#容错与安全设计)
11. [性能与异步设计](#性能与异步设计)
12. [关键指标汇总](#关键指标汇总)
13. [Sources](#sources)

---

<a id="架构概览"></a>

## 1. 架构概览

Hooks 系统在 Claude Code 的**代理循环之外**运行，响应生命周期事件触发自定义脚本。本仓库实现了一套跨平台声音通知系统，每当 Claude 执行特定操作时播放对应音效。

**核心文件结构**：

```
.claude/hooks/
├── scripts/
│   └── hooks.py              # 主事件处理器（481 行）
├── config/
│   ├── hooks-config.json      # 团队共享配置
│   └── hooks-config.local.json # 个人覆盖（git-ignored）
├── sounds/                    # 68 个音频文件，按事件组织
│   ├── pretooluse/
│   │   ├── pretooluse.mp3
│   │   ├── pretooluse.wav
│   │   ├── pretooluse-git-committing.mp3
│   │   └── pretooluse-git-committing.wav
│   ├── posttooluse/
│   ├── stop/
│   ├── agent_pretooluse/      # Agent 专用
│   └── ... (33 个文件夹)
└── logs/
    └── hooks-log.jsonl        # 事件审计日志
```

---

<a id="事件执行流水线"></a>

## 2. 事件执行流水线

```
Claude Code 触发事件
  │
  ├─ settings.json 中注册的 Hook 配置
  │
  ├─ 调用 hooks.py 处理器
  │    │
  │    ├─ 解析 CLI 参数（是否 --agent 模式）
  │    ├─ 从 stdin 读取事件 JSON 数据
  │    ├─ 记录日志到 hooks-log.jsonl（如启用）
  │    ├─ 检查 is_hook_disabled()（配置回退逻辑）
  │    ├─ get_sound_name() → 确定音频文件
  │    └─ play_sound() → 平台特定播放
  │
  └─ 始终返回 exit code 0（不中断 Claude）
```

---

<a id="27-个-hook-事件清单"></a>

## 3. 27 个 Hook 事件清单

| 类别 | 事件 | 数量 |
|------|------|------|
| **会话生命周期** | `SessionStart`, `SessionEnd` | 2 |
| **工具执行** | `PreToolUse`, `PostToolUse`, `PostToolUseFailure` | 3 |
| **权限与安全** | `PermissionRequest`, `PermissionDenied` | 2 |
| **用户交互** | `UserPromptSubmit`, `Notification`, `Stop`, `StopFailure` | 4 |
| **代理事件** | `SubagentStart`, `SubagentStop` | 2 |
| **文件与配置** | `FileChanged`, `ConfigChange`, `CwdChanged` | 3 |
| **工作区** | `WorktreeCreate`, `WorktreeRemove` | 2 |
| **系统** | `PreCompact`, `PostCompact`, `Setup` | 3 |
| **协作** | `TeammateIdle` | 1 |
| **任务管理** | `TaskCreated`, `TaskCompleted` | 2 |
| **知识** | `InstructionsLoaded` | 1 |
| **问询** | `Elicitation`, `ElicitationResult` | 2 |
| **合计** | | **27** |

---

<a id="核心处理器"></a>

## 4. 核心处理器 hooks.py

**执行入口**（`main()` 函数）：

```python
def main():
    # 1. 解析 CLI 参数 (--agent flag)
    # 2. 从 stdin 读取 Claude 传入的事件 JSON
    # 3. 记录日志 (如启用)
    # 4. 检查该 hook 是否被禁用
    # 5. 根据事件确定音频文件名
    # 6. 调用平台播放器播放音频
    # 7. exit code 0 (始终)
```

**settings.json 中的标准 Hook 配置模式**：

```json
{
  "type": "command",
  "command": "python3 ${CLAUDE_PROJECT_DIR}/.claude/hooks/scripts/hooks.py",
  "timeout": 5000,
  "async": true,
  "statusMessage": "PreToolUse"
}
```

**特殊超时配置**：

| Hook | Timeout | 原因 |
|------|---------|------|
| 标准事件 | 5000ms | 快速播放音效 |
| `Setup` | 30000ms | 初始化需要更多时间（6倍标准） |

**单次触发配置**（`"once": true`）：`SessionStart`、`SessionEnd`、`PreCompact` — 每会话仅触发一次。

**文件匹配器**（`"matcher"`）：`FileChanged` 事件仅在 `.envrc|.env|.env.local` 文件变更时触发。

---

<a id="配置系统与优先级"></a>

## 5. 配置系统与优先级

```
hooks-config.local.json （存在则优先读取，个人覆盖）
    ↓ [缺少的 key 回退到]
hooks-config.json （团队共享）
    ↓ [缺少的 key 回退到]
默认行为：假定 hook 已启用
```

**hooks-config.json 结构示例**：

```json
{
  "disableLogging": true,
  "disableAllHooks": false,
  "hooks": {
    "PreToolUse": { "disabled": false },
    "PostToolUse": { "disabled": false },
    "SessionStart": { "disabled": false },
    "Stop": { "disabled": false }
  }
}
```

**禁用方式**：

| 方式 | 说明 |
|------|------|
| `"disableAllHooks": true`（settings.local.json） | 全局禁用所有 hooks |
| `hooks.事件名.disabled: true`（hooks-config.json） | 禁用单个 hook |
| hooks-config.local.json 覆盖 | 个人覆盖团队配置 |

---

<a id="音频播放系统"></a>

## 6. 音频播放系统（跨平台）

### 平台检测与播放器

| 平台 | 播放器 | 说明 |
|------|--------|------|
| **macOS** | `afplay` | 系统原生 |
| **Linux** | `paplay` → `aplay` → `ffplay` → `mpg123` | 回退链 |
| **Windows** | `winsound` 模块 | 仅 WAV，同步播放 |

### 音频文件解析路径

```
.claude/hooks/sounds/{事件文件夹}/{音频名}.{mp3|wav}
```

### 音频文件组织（68 个文件）

**标准事件（27 个文件夹）**：每个文件夹包含 `.mp3` + `.wav` 两种格式

**Agent 专用事件（6 个文件夹）**：带 `agent_` 前缀
- `agent_pretooluse/`
- `agent_posttooluse/`
- `agent_permissionrequest/`
- `agent_posttoolusefailure/`
- `agent_stop/`
- `agent_subagentstop/`

**特殊音频**：`pretooluse/pretooluse-git-committing.mp3/wav` — git 提交时播放专属音效（ElevenLabs TTS，声音：Samara X）

---

<a id="agent-钩子集成"></a>

## 7. Agent 钩子集成

Agent 前置配置（frontmatter）仅支持 **6 个 hook 事件**（27 个的子集）：

| Agent Hook | 说明 |
|------------|------|
| `PreToolUse` | 工具执行前 |
| `PostToolUse` | 工具执行后 |
| `PermissionRequest` | 权限请求 |
| `PostToolUseFailure` | 工具执行失败 |
| `Stop` | 代理停止 |
| `SubagentStop` | 子代理停止 |

**调用方式**：

```bash
python3 ${CLAUDE_PROJECT_DIR}/.claude/hooks/scripts/hooks.py --agent=weather-agent
```

**行为差异**：
- 使用 `AGENT_HOOK_SOUND_MAP` 映射到 `agent_*` 前缀文件夹
- 日志记录中包含 `"invoked_by_agent": "weather-agent"` 字段
- 播放与标准事件不同的 agent 专属音效

---

<a id="特殊命令检测"></a>

## 8. 特殊命令检测

当 `PreToolUse` 触发时，系统检测 Bash 工具的命令内容，匹配特殊模式：

```python
BASH_PATTERNS = [
    (r'git commit', "pretooluse-git-committing")
]
```

**匹配逻辑**：如果 Bash 命令中包含 `git commit`，则播放 `pretooluse-git-committing.mp3/wav` 而非标准的 `pretooluse.mp3/wav`。

**可扩展性**：向 `BASH_PATTERNS` 列表添加新元组即可支持更多特殊命令音效。

---

<a id="日志系统"></a>

## 9. 日志系统

**位置**：`.claude/hooks/logs/hooks-log.jsonl`

**格式**：JSONL（每行一个 JSON 事件）

**当前状态**：日志已禁用（`"disableLogging": true`）

**日志条目结构**：

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "..." },
  "timestamp": "...",
  "invoked_by_agent": "weather-agent"
}
```

**隐私处理**：`transcript_path` 和 `cwd` 字段在写入日志前被剥离。

---

<a id="容错与安全设计"></a>

## 10. 容错与安全设计

| 机制 | 说明 |
|------|------|
| **始终 exit code 0** | 所有错误仅输出 stderr，不中断 Claude 工作 |
| **路径遍历防护** | 对 sound_name 进行清理防止目录穿越攻击 |
| **音频文件缺失** | 先尝试 .wav，再尝试 .mp3；均不存在则静默失败 |
| **播放器缺失** | 不支持的平台返回 None，不崩溃 |
| **JSON 解析错误** | 捕获并记录，不导致 hook 系统崩溃 |
| **配置文件缺失** | 假定 hook 已启用（return False） |

---

<a id="性能与异步设计"></a>

## 11. 性能与异步设计

| 特性 | 设计 |
|------|------|
| **所有 hooks 异步** | `"async": true`，后台非阻塞触发 |
| **标准超时** | 5000ms，操作应快速完成 |
| **Windows 播放** | `winsound.SND_SYNC` — 因脚本退出后异步播放会被终止 |
| **Unix/Linux** | `Popen` + `start_new_session=True` 实现真正后台运行 |
| **单次触发优化** | Session 级事件用 `"once": true` 避免重复触发 |

---

<a id="关键指标汇总"></a>

## 12. 关键指标汇总

| 指标 | 值 |
|------|-----|
| Hook 事件总数 | 27 |
| Agent 支持的 Hook | 6 |
| 音频文件夹（标准） | 27 |
| 音频文件夹（Agent） | 6 |
| 音频文件总数 | 68 |
| 配置文件 | 2（共享 + 可选本地覆盖） |
| 处理器脚本行数 | 481 |
| 支持的跨平台播放器 | 8+ |
| 特殊命令模式 | 1（git commit） |
| 默认超时 | 5000ms |
| Setup 超时 | 30000ms |
| Python 版本 | 3.x |

---

<a id="sources"></a>

## Sources

- [Claude Code Hooks 官方文档](https://code.claude.com/docs/en/hooks)
- [Claude Code Hooks Guide](https://code.claude.com/docs/en/hooks-guide)
- [claude-code-hooks 独立项目](https://github.com/shanraisshan/claude-code-hooks)
- 本仓库实现：`.claude/hooks/scripts/hooks.py`、`.claude/settings.json`
