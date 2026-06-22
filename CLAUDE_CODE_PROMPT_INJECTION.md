# Claude Code 到底向模型注入了什么？—— 三条管道的完整拆解

> 基于 `src/constants/prompts.ts`、`src/context.ts`、`src/memdir/memdir.ts`、`src/utils/attachments.ts`、`src/services/api/claude.ts` 源码

---

## 总览：模型收到一次 API 调用时的完整画面

```
 anthropic.beta.messages.create({
   system: [                             ← 管道 A：系统提示块
     TextBlock("你是 Claude Code..."),    ← 静态指令
     TextBlock("Memory: ..."),           ← 记忆"说明书" + MEMORY.md 索引
     TextBlock("CWD: /projects/foo"),    ← 环境信息
     ...
   ],
   messages: [                           ← 管道 B：对话消息
     {role:"user", "CLAUDE.md 内容..."}, ← 用户上下文
     {role:"user", "帮我写个 webhook"},  ← 真正的用户消息
     {role:"assistant", "...",           ← 模型回复（含 tool_calls）
      tool_calls: [...]},
     {role:"tool", "stdout: ..."},       ← 工具结果
     {role:"user",                       ← 系统提醒（记忆内容等）
      "<system-reminder>有关记忆...</>"},
     ...
   ],
   tools: [                              ← 管道 C：工具定义
     {name:"Bash", description:"...", input_schema:{...}},
     {name:"Read", description:"...", input_schema:{...}},
     ...
   ]
 })
```

三条管道，各走各的路，互不相干。

---

## 管道 A：`system` — 系统性指令（缓存在服务端，跨轮复用）

**源码路径**: `src/constants/prompts.ts:getSystemPrompt()` → `src/services/api/claude.ts:buildSystemPromptBlocks()`

这是**不变的东西**——整个会话期间内容不变，被 Anthropic API 的 prompt cache 缓存，不消耗每轮的 input token。

### 组装顺序（从先到后）

```
┌─────────────────────────────────────────────────────┐
│ ① 属性头（Attribution Header）                       │
│    "x-anthropic-billing-header: ..."                │
│    来源: src/services/api/claude.ts:1358            │
├─────────────────────────────────────────────────────┤
│ ② CLI 前缀                                          │
│    "You are Claude Code, Anthropic's official CLI   │
│     for Claude."                                    │
│    来源: src/constants/system.ts                     │
├─────────────────────────────────────────────────────┤
│ ③ 静态指令段（可跨用户缓存，scope=global）            │
│    ├─ Intro: "You are an interactive agent..."      │
│    ├─ System: 操作指引、system-reminder 说明          │
│    ├─ Doing Tasks: 工程行为准则                       │
│    ├─ Actions: 谨慎操作指引                           │
│    ├─ Using Tools: 工具偏好（用 Read 不用 cat 等）     │
│    ├─ Tone and Style: 输出风格                        │
│    └─ Output Efficiency: 简洁要求                    │
│    来源: src/constants/prompts.ts:176-440            │
├─────────────────────────────────────────────────────┤
│ ★ 动态边界标记 __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__     │
│    以下内容不跨用户缓存                                │
│    来源: src/constants/prompts.ts:573                │
├─────────────────────────────────────────────────────┤
│ ④ Session Guidance（会话专属指引）                    │
│    ├─ AskUserQuestion 使用说明（如有该工具）           │
│    ├─ 非交互模式提示                                  │
│    ├─ Agent Tool 使用说明（子 agent 怎么用）           │
│    ├─ Skill Tool 说明                                │
│    └─ Plan Mode 说明                                 │
│    来源: src/constants/prompts.ts:352-400             │
├─────────────────────────────────────────────────────┤
│ ⑤ ★ 记忆"说明书" + MEMORY.md 索引                   │
│    ├─ "You have a persistent file-based memory..."   │
│    ├─ 四种记忆类型说明（user/feedback/project/reference）│
│    ├─ 何时保存 / 不保存什么                            │
│    └─ MEMORY.md 索引内容（一行一个文件指针）           │
│    来源: src/memdir/memdir.ts:loadMemoryPrompt()     │
│          被 src/constants/prompts.ts:495 调用并缓存    │
├─────────────────────────────────────────────────────┤
│ ⑥ 环境信息                                          │
│    ├─ Working directory: /projects/payment-api      │
│    ├─ Git branch: main (clean)                      │
│    ├─ Platform: darwin                               │
│    ├─ Shell: zsh                                     │
│    └─ "You are powered by the model claude-sonnet-4-6"│
│    来源: src/constants/prompts.ts:computeSimpleEnvInfo│
├─────────────────────────────────────────────────────┤
│ ⑦ 语言偏好（如有设置）                                │
│    来源: src/constants/prompts.ts:getLanguageSection  │
├─────────────────────────────────────────────────────┤
│ ⑧ MCP 服务器指引（如有连接）                          │
│    来源: src/constants/prompts.ts:MCP section        │
├─────────────────────────────────────────────────────┤
│ ⑨ Scratchpad 目录                                   │
│    "Write temp files to /tmp/claude-xxxxx/"          │
│    来源: src/constants/prompts.ts:scratchpad section  │
└─────────────────────────────────────────────────────┘
```

### 关键特性

| 特性 | 说明 |
|------|------|
| **缓存** | ③ 之前的静态段使用 `cache_scope: 'global'`（跨用户缓存）；④之后是 session-specific |
| **更新频率** | 整个会话期间不变。⑤ 被 `systemPromptSection()` 缓存，模块重新加载才重新读 MEMORY.md |
| **Token 消耗** | 只有第一次 API 调用计 input token，后续调用使用缓存 |

---

## 管道 B：`messages` — 对话内容（每轮都可能变）

**源码路径**: `src/query.ts`（消息组装）+ `src/context.ts`（用户上下文）+ `src/utils/attachments.ts`（附件注入）+ `src/memdir/findRelevantMemories.ts`（相关记忆）

这是**会变的东西**——每轮对话都可能追加新内容。

### 消息数组的完整结构（某轮 API 调用时）

```
messages = [
  ┌─────────────────────────────────────────────────┐
  │ ★ 用户上下文（只在最开始注入一次）                 │
  │ {role: "user", content: "                        │
  │  <system-reminder>                               │
  │  Today's date: 2026-06-22                        │
  │                                                  │
  │  Project CLAUDE.md contents:                     │
  │  # Payment API Guidelines                        │
  │  - Use Express + TypeScript                      │
  │  - Prisma for database                           │
  │  - Jest for testing                              │
  │  </system-reminder>"}                            │
  │  来源: src/context.ts:getUserContext()           │
  │        → src/query.ts:662 prependUserContext()    │
  ├─────────────────────────────────────────────────┤
  │ ★ 系统上下文（git 状态等）                         │
  │ {role: "user", content: "                        │
  │  <system-reminder>                               │
  │  Git status: branch main, 2 files modified       │
  │  Recent commits: ...                             │
  │  </system-reminder>"}                            │
  │  来源: src/context.ts:getSystemContext()          │
  │        → appendSystemContext()                    │
  ├─────────────────────────────────────────────────┤
  │ 真正的用户消息 1                                   │
  │ {role: "user", content: "帮我看看项目结构"}        │
  ├─────────────────────────────────────────────────┤
  │ 模型回复 1                                        │
  │ {role: "assistant", content: "...",               │
  │  tool_calls: [{name:"Glob", ...}, {name:"Read",...}]}│
  ├─────────────────────────────────────────────────┤
  │ 工具结果                                          │
  │ {role: "tool", tool_call_id:"...", content:"..."} │
  │ {role: "tool", tool_call_id:"...", content:"..."} │
  ├─────────────────────────────────────────────────┤
  │ 模型回复 2（继续）                                 │
  │ {role: "assistant", content: "项目用了 Express..."} │
  ├─────────────────────────────────────────────────┤
  │ ★ 相关记忆附件（每轮动态注入）                     │
  │ {role: "user", content: "                        │
  │  <system-reminder>                               │
  │  The following memories were surfaced as         │
  │  relevant:                                       │
  │                                                  │
  │  [user] prefer-typescript-express                │
  │  张三偏好 TypeScript + Express.js                 │
  │                                                  │
  │  [project] payment-api-tech-stack                │
  │  payment-api 使用 Prisma + PostgreSQL            │
  │  </system-reminder>"}                            │
  │  来源: src/utils/attachments.ts:2241             │
  │        → src/memdir/findRelevantMemories.ts:39   │
  ├─────────────────────────────────────────────────┤
  │ 用户消息 2                                        │
  │ {role: "user", content: "帮我写 webhook handler"} │
  ├─────────────────────────────────────────────────┤
  │ ...继续循环...                                    │
  └─────────────────────────────────────────────────┘
]
```

### 消息数组中可能出现的所有 `<system-reminder>` 类型

| 类型 | 来源 | 内容 | 何时注入 |
|------|------|------|---------|
| **date** | `context.ts:getUserContext()` | 当前日期 | 会话开始时一次性注入 |
| **CLAUDE.md** | `context.ts:getUserContext()` | 项目 CLAUDE.md 文件内容 | 会话开始时一次性注入 |
| **git status** | `context.ts:getSystemContext()` | 分支名、改动文件列表 | 会话开始时一次性注入 |
| **relevant memories** | `attachments.ts:2241` → `findRelevantMemories.ts` | 选中的记忆文件全文（≤5个） | 每轮 API 调用前 |
| **function result clearing** | `prompts.ts` feature-gated | "旧工具结果会被清理" | 微压缩触发时 |
| **compact summary** | `compact.ts` | 压缩后的对话摘要 | 压缩被执行时 |
| **hook progress** | hooks 系统 | 钩子执行进度 | 钩子运行时 |
| **task notifications** | tasks 系统 | 子 Agent 完成通知 | 子 Agent 结束时 |

---

## 管道 C：`tools` — 工具定义（在 API 的 tools 参数中）

**源码路径**: `src/tools.ts:getTools()` → `src/utils/api.ts:toolToAPISchema()` → `src/services/api/claude.ts:paramsFromContext()`

这是**工具的 JSON Schema + 使用说明**。每个工具一个对象：

```json
{
  "name": "Bash",
  "description": "Execute a shell command in the terminal.\n\nIMPORTANT: Prefer dedicated tools (Read, Edit, Glob, Grep) over Bash when possible.\nUse Bash for: running tests, installing dependencies, git operations...\nAlways provide a clear description of what the command does.",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": {"type": "string", "description": "The shell command to execute"},
      "timeout": {"type": "integer", "default": 120000},
      "description": {"type": "string", "description": "Brief description"}
    },
    "required": ["command"]
  }
}
```

每个工具的 `description` 来自它自己的 `prompt.ts`：
- `src/tools/BashTool/prompt.ts:getSimplePrompt()`
- `src/tools/FileEditTool/prompt.ts:getEditToolDescription()`
- `src/tools/AgentTool/prompt.ts:getPrompt()`
- ...

### 关键特性

| 特性 | 说明 |
|------|------|
| **缓存** | 工具定义被包含在 prompt cache 前缀中 |
| **更新频率** | 会话启动时确定，MCP 服务器连接/断开时刷新 |
| **Token 消耗** | 随 system prompt 一起缓存 |

---

## 三种"记忆"的精确区分

Claude Code 里 "memory" 这个词有三个不同的指代，这是混乱的根源：

```
                    ┌──────────────────────────────────────┐
                    │          "记忆" 的三个含义            │
                    └──────────────────────────────────────┘

① 记忆"说明书"                         ② 记忆索引             ③ 记忆内容
   管道 A: system                        管道 A: system         管道 B: messages
   会话开始时注入                        会话开始时注入          每轮动态注入
   内容固定                               内容固定               内容随查询变化
       │                                      │                     │
       ▼                                      ▼                     ▼
"You have a persistent          "### Current Memory       <system-reminder>
file-based memory at            Index:                    The following memories
~/.claude/.../memory/           - [Prefer TS](...)        were surfaced:
                                 - [Tech Stack](...)
Each memory is one file                                  [user] prefer-ts
holding one fact, with           - [Webhook](...)"        张三偏好 TypeScript
frontmatter..."                                           + Express.js"

  ↑ 教模型"怎么用记忆"              ↑ 告诉模型"有哪些记忆"      ↑ 给模型看"记忆里写了什么"
```

---

## 一张图总结三条管道

```
                          anthropic.beta.messages.create()

  ┌───────────────────────┬──────────────────────────────┬────────────────────┐
  │    system (管道A)      │       messages (管道B)        │    tools (管道C)    │
  │    系统提示块数组        │       对话消息数组              │    工具定义数组       │
  │    缓存在服务端          │       每轮都可能变              │    缓存在服务端       │
  ├───────────────────────┼──────────────────────────────┼────────────────────┤
  │                       │                              │                    │
  │ ① 属性头              │ ① 用户上下文                  │ Bash               │
  │ ② CLI前缀             │   - 日期                      │  ├─ description    │
  │ ③ 静态指令            │   - CLAUDE.md 内容            │  └─ input_schema   │
  │ ④ Session 指引        │ ② 系统上下文                  │ Read               │
  │ ⑤ 记忆"说明书"        │   - git branch                │  ├─ description    │
  │    + MEMORY.md 索引   │   - 改动文件列表               │  └─ input_schema   │
  │ ⑥ 环境信息            │ ③ 真正的对话                  │ Edit               │
  │    - cwd, platform    │   - user 消息                 │  ├─ description    │
  │    - model name       │   - assistant 消息            │  └─ input_schema   │
  │ ⑦ 语言偏好            │   - tool 结果                 │ Grep               │
  │ ⑧ MCP 指引            │ ④ 系统提醒                   │  ...               │
  │ ⑨ Scratchpad         │   - 相关记忆内容 ★            │                    │
  │                       │   - 压缩摘要                   │                    │
  │                       │   - 钩子进度                   │                    │
  │                       │   - 任务通知                   │                    │
  │                       │                              │                    │
  │ 更新: 会话级缓存       │ 更新: 每轮追加               │ 更新: 会话级缓存     │
  └───────────────────────┴──────────────────────────────┴────────────────────┘
```

---

## 那"相关记忆内容"到底什么时候出现？

这是最容易被误解的点。用一个具体的时间线说明：

```
轮次 1 (用户: "帮我看看项目")
  │
  ├─ API 调用前: findRelevantMemories("帮我看看项目")
  │   → memory 目录为空 → 无附件注入
  │
  ├─ API 调用
  │
  └─ API 调用后: executeExtractMemories()
      → Fork Agent 写到磁盘:
         prefer-ts.md, tech-stack.md, MEMORY.md

轮次 2 (用户: "写 webhook")
  │
  ├─ API 调用前: findRelevantMemories("写 webhook")
  │   → scanMemoryFiles() 找到 2 个文件
  │   → Sonnet 子查询判断都相关
  │   → 两种记忆的全文内容被读取
  │   → 作为 <system-reminder> 注入 messages ★
  │
  ├─ API 调用 (模型现在"看见"了记忆内容)
  │
  └─ API 调用后: executeExtractMemories()
      → 游标前进，可能提取新记忆

轮次 3 (用户: "加签名校验")
  │
  ├─ API 调用前: findRelevantMemories("加签名校验")
  │   → alreadySurfaced = {prefer-ts.md, tech-stack.md}（去重）
  │   → 可选的新文件 webhook-requirement.md 被选中
  │   → 只注入 webhook-requirement.md 的内容 ★
  │
  └─ ...
```

**关键**: 记忆内容不是一直在那里的。每轮重新扫描、重新选择、重新注入。`alreadySurfaced` 确保不重复注入同样的内容。

---

## 源码路径速查

| 注入什么 | 去哪 | 源文件 | 何时 |
|---------|------|--------|------|
| 静态指令 | system | `constants/prompts.ts:176-440` | 会话启动 |
| Session 指引 | system | `constants/prompts.ts:352-400` | 会话启动 |
| 记忆"说明书"+索引 | system | `memdir/memdir.ts:419 loadMemoryPrompt()` → `prompts.ts:495` | 会话启动，缓存 |
| 环境信息 | system | `constants/prompts.ts:computeSimpleEnvInfo` | 会话启动 |
| 日期 | messages | `context.ts:getUserContext()` | 会话启动 |
| CLAUDE.md | messages | `context.ts:getUserContext()` | 会话启动 |
| git 状态 | messages | `context.ts:getSystemContext()` | 会话启动 |
| 相关记忆内容 | messages | `memdir/findRelevantMemories.ts:39` → `attachments.ts:2241` | 每轮 |
| 压缩摘要 | messages | `services/compact/compact.ts` | 压缩时 |
| 工具定义 | tools | `tools.ts:getTools()` → `utils/api.ts:toolToAPISchema()` | 会话启动 |
