# Claude Code 记忆持久化全景：一个多轮对话的完整旅程

> 基于 `src/memdir/`、`src/services/extractMemories/`、`src/services/compact/`、`src/query/stopHooks.ts`、`src/query.ts` 源码追溯

---

## 序：Claude Code 有两套"记忆"，别搞混

在开始之前，必须区分 Claude Code 中两种不同性质的持久化：

| | Session Transcript | Memory Files |
|---|---|---|
| **存储位置** | `~/.claude/history.jsonl` | `~/.claude/projects/<slug>/memory/` |
| **格式** | JSONL（一行一个用户输入） | MEMORY.md 索引 + *.md 记忆文件 |
| **内容** | 用户每次输入的原始文本 | 结构化知识：偏好、事实、反馈 |
| **谁写入** | `addToHistory()` 同步写入 | `executeExtractMemories()` Fork Agent 写入 |
| **何时读** | 上箭头/Ctrl-R 搜索历史 | 会话启动注入 system prompt + 每轮选择相关记忆 |
| **跨会话** | ✅ （相同的 history.jsonl） | ✅ （相同的 memory 目录） |
| **生命周期** | 追加写入，永不删除 | 创建/更新/整合（Dream） |

**核心区别**：Session Transcript 像聊天记录——记录了"用户说了什么"。Memory Files 像笔记——提炼了"用户是什么样的人、项目有什么约定、上次怎么解决的"。

---

## 场景设定

假设一个叫张三的用户，在 `~/projects/payment-api` 项目中与 Claude Code 进行多轮对话。

项目路径：
```
~/projects/payment-api/
├── src/
├── .git/
└── .claude/
    └── projects/
        └── payment-api-abc123/     ← project slug
            └── memory/              ← ★ 记忆持久化目录
                ├── MEMORY.md        ← ★ 索引文件
                ├── *.md             ← ★ 记忆文件
```

记忆文件格式（`src/memdir/memoryTypes.ts`）：
```markdown
---
name: prefer-typescript-pnpm
description: 用户偏好 TypeScript 和 pnpm
metadata:
  type: user
---

张三偏好 TypeScript + pnpm，不喜欢 class-based 写法。
```

---

## 第一幕：首次会话启动

### Step 0: 用户启动 Claude Code

```bash
$ cd ~/projects/payment-api
$ claude
```

### Step 1: 系统提示组装——记忆"说明书"被注入

**源码路径**: `src/constants/prompts.ts:495` → `src/memdir/memdir.ts:loadMemoryPrompt()` → `src/services/api/claude.ts:1358`

当 REPL 就绪、准备发送第一条消息时，系统提示被组装。其中记忆相关的部分是这样产生的：

```
prompts.ts:495
  └─ systemPromptSection('memory', () => loadMemoryPrompt())
       └─ memdir.ts:419 loadMemoryPrompt()
            ├─ ensureMemoryDirExists()               ← 创建 memory 目录
            ├─ fs.readFile(MEMORY.md)                 ← 读索引文件
            ├─ truncateEntrypointContent()            ← 截断到 200行/25KB
            └─ buildMemoryLines()                     ← 生成记忆使用说明
```

`buildMemoryLines()` 生成的文本不会直接带记忆内容，而是带**记忆系统的使用说明**：

```markdown
# Memory
You have a persistent file-based memory at
~/.claude/projects/payment-api-abc123/memory/

This directory already exists — write to it directly with the Write tool.

Each memory is one file holding one fact, with frontmatter:
---
name: <short-kebab-case-slug>
description: <one-line summary>
metadata:
  type: user | feedback | project | reference
---

## Types of memory
- **user**: who the user is (role, expertise, preferences)
- **feedback**: guidance the user has given on how you should work
- **project**: ongoing work, goals, or constraints
- **reference**: pointers to external resources (URLs, docs)

## When to save
Save when the user explicitly tells you to remember something,
OR when you notice a recurring preference/pattern.

Before saving, check MEMORY.md for an existing file that already
covers it — update that file rather than creating a duplicate.

## What NOT to save
- Code patterns or past fixes (the repo already records these)
- Git history or file structure
- CLAUDE.md contents
```

这段"说明书"被缓存（`systemPromptSection` 内部用了 memo），整个会话期间不再变化。

此时 memory 目录是空的（`MEMORY.md` 还不存在），所以实际注入的文本更简洁——只有使用说明，没有索引内容。

### Step 2: 用户发出第一条消息

```
> 帮我看看这个项目的结构，我想了解 payment-api 用了什么技术栈
```

### Step 3: 每轮对话的相关记忆预取（目前为空）

**源码路径**: `src/utils/attachments.ts:2361 startRelevantMemoryPrefetch()` → `src/memdir/findRelevantMemories.ts:39`

每轮发送 API 请求之前，会异步触发相关记忆的预取：

```
每一轮 (query.ts 的 while 循环):
  ├─ attachments.ts:2361 startRelevantMemoryPrefetch()
  │    ├─ 检查 isAutoMemoryEnabled()            ← 是否开启自动记忆
  │    ├─ 检查 feature('tengu_moth_copse')      ← feature flag
  │    ├─ 找到最后一个用户消息
  │    └─ 调用 getRelevantMemoryAttachments()
  │         └─ findRelevantMemories.ts:39 findRelevantMemories()
  │              ├─ scanMemoryFiles()             ← 扫描 memory 目录所有 .md
  │              ├─ 过滤 alreadySurfaced           ← 去掉本轮已展示过的
  │              └─ selectRelevantMemories()       ← ★ 调用 Sonnet 选 top 5
```

这是一个子查询（side query），用一个精简的 Sonnet 调用：
```
System: 你是记忆选择器。根据用户查询，从记忆清单中选出最相关的。
        只选真正有用的，最多 5 个。

Query: "帮我看看这个项目的结构..."

Available memories:
(空 — 目前还没有任何记忆文件)
```

因为 memory 目录是空的，`scanMemoryFiles()` 返回空列表，`findRelevantMemories()` 返回空，相关记忆附件为空——本轮没有记忆被注入。

### Step 4: Agent 响应

Agent 读取项目文件，回复：

```
这个项目用了 Express + TypeScript + Prisma，数据库是 PostgreSQL。
```

### Step 5: 对话结束——自动提取记忆（第一次写入！）

**源码路径**: `src/query.ts` → `src/query/stopHooks.ts:149 executeExtractMemories()`

当 Agent 完成响应（no more tool_calls），`handleStopHooks()` 被调用。记忆提取在这里以 **fire-and-forget** 方式触发：

```
stopHooks.ts:136-157
  if (!isBareMode()) {
      void executePromptSuggestion(...)      ← fire-and-forget 1
      void executeExtractMemories(...)       ← fire-and-forget 2 ★
      void executeAutoDream(...)             ← fire-and-forget 3
  }
```

`executeExtractMemories()` 内部（`src/services/extractMemories/extractMemories.ts`）：

```
runExtraction():
  ├─ 计算自上次游标以来的新消息数量
  ├─ 检查主 Agent 是否已经自己写了记忆（跳过）
  ├─ 节流检查：每 N 轮才执行（默认 1 轮一次）
  ├─ 预注入记忆清单（当前为空）
  └─ ★ Fork 一个子 Agent，用提取提示
       ├─ 提示："从以下对话中识别值得记住的信息，按 user/feedback/project/reference 分类"
       ├─ 工具权限：createAutoMemCanUseTool()
       │    ├─ Read/Grep/Glob: 无限制
       │    ├─ Bash: 只读
       │    └─ Edit/Write: 仅限 memory 目录 ★
       └─ 子 Agent 写出:
            ├─ ~/.claude/.../memory/prefer-typescript-express.md
            ├─ ~/.claude/.../memory/payment-api-tech-stack.md
            └─ MEMORY.md (更新索引)
```

**提取出的记忆文件内容**：

`prefer-typescript-express.md`:
```markdown
---
name: prefer-typescript-express
description: 张三使用 TypeScript + Express 技术栈
metadata:
  type: user
---

张三偏好 TypeScript + Express.js 作为后端框架。
```

`payment-api-tech-stack.md`:
```markdown
---
name: payment-api-tech-stack
description: payment-api 项目技术栈
metadata:
  type: project
---

payment-api 使用 Express + TypeScript + Prisma + PostgreSQL。
```

`MEMORY.md`:
```markdown
- [Prefer TypeScript Express](prefer-typescript-express.md) — 张三使用 TypeScript + Express
- [Payment API Tech Stack](payment-api-tech-stack.md) — payment-api 项目技术栈
```

**关键设计点**：
- 提取是 fire-and-forget，不阻塞用户看到结果
- 子 Agent 的工具权限被严格限制——只能写 memory 目录
- 游标追踪确保同一条消息不会被重复提取

---

## 第二幕：同一会话的后续对话

### Step 6: 用户继续对话

```
> 帮我写一个支付回调的 webhook handler
```

### Step 7: 相关记忆再次被查询（这次有内容了！）

**源码路径**: 同一流程，`startRelevantMemoryPrefetch()` → `findRelevantMemories()`

这次 `scanMemoryFiles()` 返回了两个记忆：

```
[user] prefer-typescript-express.md (2026-06-18): 张三使用 TypeScript + Express
[project] payment-api-tech-stack.md (2026-06-18): payment-api 项目技术栈
```

Sonnet 子查询判断两条都相关，返回两条。

`readMemoriesForSurfacing()` 读取它们的内容，然后作为 `<system-reminder>` 注入到消息中：

```xml
<system-reminder>
The following memories were surfaced as relevant to the current conversation.
They reflect background context written before this conversation — verify
information with the user if something contradicts what you see.

<memory>
<name>prefer-typescript-express</name>
<type>user</type>
张三偏好 TypeScript + Express.js 作为后端框架。
</memory>

<memory>
<name>payment-api-tech-stack</name>
<type>project</type>
payment-api 使用 Express + TypeScript + Prisma + PostgreSQL。
</memory>
</system-reminder>
```

**Agent 就自动知道**：用 TypeScript 写、Express 框架、Prisma ORM——不需要用户再说一遍。

### Step 8: 对话结束——再次提取

这次 `executeExtractMemories()` 的游标已经前进了，只处理新增消息。如果新对话中产生了新的有价值信息（比如用户说 "webhook 回调要加签名校验"），子 Agent 可能会创建：

```
~/.claude/.../memory/webhook-signature-requirement.md
```

并更新 `MEMORY.md` 追加一行索引。

---

## 第三幕：上下文太长，触发自动压缩

### Step 9: Token 超阈值

**源码路径**: `src/services/compact/autoCompact.ts:shouldAutoCompact()` → `src/query.ts` (before API call)

经过几十轮对话后，消息的 token 数超过 `contextWindow - 13000`：

```
autoCompact.ts:shouldAutoCompact()
  ├─ estimateTokens(messages) > (contextWindow - AUTOCOMPACT_BUFFER_TOKENS)
  └─ 返回 true
```

### Step 10: 压缩执行

**源码路径**: `src/services/compact/compact.ts:compactConversation()`

压缩过程：

```
compactConversation():
  ├─ 发送全部消息到 API，生成结构化摘要
  │    ├─ 1. Primary Request and Intent
  │    ├─ 2. Key Technical Concepts
  │    ├─ 3. Files and Code Sections
  │    ├─ 4. Errors and fixes
  │    ├─ 5. Problem Solving
  │    ├─ 6. All user messages
  │    ├─ 7. Pending Tasks
  │    ├─ 8. Current Work
  │    └─ 9. Optional Next Step
  │
  ├─ 压缩后重建上下文附件:
  │    ├─ 重新读取最近 5 个文件（50K token 预算）
  │    ├─ 保留 Plan 内容
  │    └─ 保留 Skill 内容（25K token 预算）
  │
  ├─ context.readFileState.clear()           ← 清除文件读缓存
  └─ context.loadedNestedMemoryPaths.clear() ← 清除嵌套 CLAUDE.md 路径
```

**压缩对记忆的影响**：

| 对象 | 影响 |
|------|------|
| **memory 目录下的 .md 文件** | ❌ 不受影响 —— 原封不动 |
| **MEMORY.md 索引** | ❌ 不受影响 |
| **系统提示中的记忆"说明书"** | ❌ 不受影响 —— 被 `systemPromptSection` 缓存了 |
| **`loadedNestedMemoryPaths`** | ✅ 被清除 —— 旧的 memory 附件从压缩后的 transcript 中移除了 |
| **`readFileState`** | ✅ 被清除 |
| **已展示的相关记忆 tracking** | ✅ 自然重置 —— `collectSurfacedMemories()` 基于消息扫描，压缩后旧附件没了 |

压缩后，下一轮对话仍然会运行 `findRelevantMemories()`，Agent 仍然能找到相关记忆。

---

## 第四幕：关闭终端，第二天回来

### Step 11: 新会话启动

```bash
$ cd ~/projects/payment-api
$ claude
```

### Step 12: 记忆加载（关键！）

**源码路径**: `src/constants/prompts.ts:495 systemPromptSection('memory', () => loadMemoryPrompt())`

新的 session，`systemPromptSection` 的缓存失效（新的模块加载），`loadMemoryPrompt()` 重新执行：

```
loadMemoryPrompt():
  ├─ ensureMemoryDirExists()  ← 目录已存在，无需创建
  ├─ fs.readFile(MEMORY.md)   ← ★ 读到之前的索引！
  └─ truncateEntrypointContent(MEMORY.md内容)
```

这次，因为 `MEMORY.md` 已有内容，注入到系统提示的"说明书"后面多了索引：

```markdown
# Memory
You have a persistent file-based memory...
[使用说明...]

### Current Memory Index:
- [Prefer TypeScript Express](prefer-typescript-express.md) — 张三使用 TypeScript + Express
- [Payment API Tech Stack](payment-api-tech-stack.md) — payment-api 项目技术栈
- [Webhook Signature Requirement](webhook-signature-requirement.md) — webhook 需要签名校验
```

**重点**：这里的 `MEMORY.md` 内容不是记忆本身，而是**索引**——一行一个记忆文件指针。Agent 看到索引后，如果需要详细信息，可以用 Read 工具打开具体的 `.md` 文件。

### Step 13: 用户新消息

```
> 继续昨天的 webhook 工作，帮我写单元测试
```

### Step 14: 相关记忆预取

`findRelevantMemories()` 扫描目录，发现 3 个记忆。Sonnet 子查询判断 `webhook-signature-requirement.md` 最相关（包含了上次的 webhook 上下文），`payment-api-tech-stack.md` 次相关（包含了测试框架信息）。返回 2 个。

这两个记忆的内容被注入到 `<system-reminder>` 中。Agent 自动知道：
- 项目用 Express + TypeScript + Prisma
- 上次写的 webhook 需要签名校验
- 应该用 Jest + Supertest 写测试（如果 tech-stack 记忆里记录了测试框架）

---

## 第五幕：三个月后，Dream 整合

### Step 15: Auto Dream 触发

**源码路径**: `src/services/autoDream/autoDream.ts:140`

条件满足：
- 距离上次整合 > 24 小时 ✅
- 新增会话 > 5 个 ✅
- 文件锁未占用 ✅

```
autoDream.ts:initAutoDream()
  └─ forkAgent(getDreamPrompt())
       └─ 子 Agent 读取 memory 目录所有文件
       └─ 整合、去重、提炼
       └─ 可能将 3 个零散记忆合并为 1 个更精炼的
       └─ 更新 MEMORY.md
```

---

## 完整时间线总览

```
时间 ─────────────────────────────────────────────────────►

SESSION 1 (周一 9:00)                    SESSION 5 (周二 10:00)      SESSION N (周五)

┌─ 启动 ─────────────────────────────────────────────────────────────┐
│ loadMemoryPrompt()           loadMemoryPrompt()                      │
│ MEMORY.md 为空               MEMORY.md 有 3 条索引                  │
│ → 只有"说明书"               → "说明书" + 索引                      │
└────────────────────────────────────────────────────────────────────┘

┌─ Turn 1 ───────────────────────────────────────────────────────────┐
│                                                                     │
│ findRelevantMemories()      findRelevantMemories()                  │
│   scanMemoryFiles() → 空      scanMemoryFiles() → 3 files           │
│   → 无附件                    → Sonnet 选 2 个最相关                   │
│                               → 注入 <system-reminder>              │
│                                                                     │
│ [用户消息 + LLM 调用 + 工具执行]                                     │
│                                                                     │
│ executeExtractMemories()     executeExtractMemories()               │
│   → 创建 memory 文件           → 游标前进，增量提取                   │
│   → 更新 MEMORY.md            → 可能创建新文件                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─ Turn N: Token 太多，触发压缩 ─────────────────────────────────────┐
│ compactConversation()                                                │
│   → 生成摘要                                                        │
│   → readFileState.clear()                                           │
│   → loadedNestedMemoryPaths.clear()                                 │
│   → memory 文件在磁盘上：不受影响                                     │
│   → 系统提示中的记忆"说明书"：不受影响（已缓存）                       │
│   → 压缩后下一轮：findRelevantMemories() 重新扫描，重新注入            │
└─────────────────────────────────────────────────────────────────────┘

┌─ SESSION 之间（如周一晚）──────────────────────────────────────────┐
│ history.jsonl：存储了所有用户输入（原始文本）                          │
│ memory/*.md：存储在项目目录下，跨会话持续存在                          │
└─────────────────────────────────────────────────────────────────────┘

┌─ 三个月后 ─────────────────────────────────────────────────────────┐
│ autoDream 触发                                                       │
│   → 扫描所有会话 transcript + memory 文件                             │
│   → 整合、去重、提炼                                                  │
│   → 更新 MEMORY.md                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键源码路径速查

| 动作 | 源文件 | 函数 | 触发时机 |
|------|--------|------|---------|
| **加载记忆"说明书"** | `src/memdir/memdir.ts:419` | `loadMemoryPrompt()` | 会话启动时，被 `prompts.ts:495` 调用并缓存 |
| **构建系统提示** | `src/constants/prompts.ts` | `getSystemPrompt()` | 会话启动时 |
| **注入记忆到 API** | `src/services/api/claude.ts:1358` | `queryModel()` | 每次 API 调用前 |
| **扫描记忆文件** | `src/memdir/memoryScan.ts` | `scanMemoryFiles()` | 记忆预取时（每轮） |
| **选择相关记忆** | `src/memdir/findRelevantMemories.ts:39` | `findRelevantMemories()` | 每轮 API 调用前 |
| **Sonnet 子查询** | `src/memdir/findRelevantMemories.ts:77` | `selectRelevantMemories()` | 同上，选 top 5 |
| **注入相关记忆附件** | `src/utils/attachments.ts:2241` | `getRelevantMemoryAttachments()` | 同上，封装为 system-reminder |
| **提取新记忆** | `src/services/extractMemories/extractMemories.ts` | `executeExtractMemories()` | 每轮对话结束（fire-and-forget） |
| **Fork Agent 提取** | `src/services/extractMemories/extractMemories.ts` | `runExtraction()` | 同上，子 Agent 写 memory 文件 |
| **触发提取** | `src/query/stopHooks.ts:149` | `handleStopHooks()` | query.ts 每轮结束 |
| **自动压缩** | `src/services/compact/autoCompact.ts` | `autoCompactIfNeeded()` | token 超阈值时（每轮前检查） |
| **压缩执行** | `src/services/compact/compact.ts:387` | `compactConversation()` | 压缩触发后 |
| **清除记忆状态** | `src/services/compact/compact.ts:521-522` | — | 压缩时，clear readFileState + loadedNestedMemoryPaths |
| **写历史** | `src/history.ts` | `addToHistory()` | 用户每次提交输入 |
| **自动整合** | `src/services/autoDream/autoDream.ts:140` | `initAutoDream()` | 24h 间隔 + 5+ 新会话 |

---

## 三个核心设计精髓

### 1. 记忆"说明书" vs 记忆"内容" 分离

`loadMemoryPrompt()` 注入的是**如何读写记忆的指引**，不是记忆内容本身。实际的记忆内容是通过 `findRelevantMemories()` → Sonnet 子查询 → 注入 `<system-reminder>` 附件来提供的。这二者走不同的管道：
- 说明书：系统提示 → 缓存 → 每轮都在 → 不占消息 token
- 内容：消息附件 → 每轮重新选择 → 只注入相关的 5 条

### 2. 压缩不触碰磁盘文件

当上下文太长需要压缩时，历史消息被摘要化，但 `memory/` 目录下的 `.md` 文件完好无损。压缩后 `loadedNestedMemoryPaths` 和 `readFileState` 被清除，但下一轮的 `findRelevantMemories()` 会重新扫描目录、重新注入相关记忆。这意味着**记忆是持久化真相，压缩只是临时的上下文管理**。

### 3. Fire-and-Forget 提取 + Fork Agent

记忆提取不阻塞用户——`void executeExtractMemories()` 在后台运行。提取由一个受限的子 Agent 完成，它只能读写 memory 目录，不能碰项目代码。游标追踪确保增量处理，不浪费 API 调用。
