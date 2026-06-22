# 模型到底怎么"看到"记忆的？—— 一次完整的 findRelevantMemories 追踪

> 基于 `src/memdir/findRelevantMemories.ts`、`src/memdir/memoryScan.ts`、`src/utils/attachments.ts` 源码

---

## 问题

作为开发者，我要在自己的 Agent 里实现类似 Claude Code 的记忆系统。
**模型到底在什么时候、以什么格式、收到什么记忆内容？**

---

## 完整链路（一次 API 调用前的记忆预取）

### Step 1: 用户输入一条消息

```
> 帮我给 webhook 回调加个 HMAC 签名校验
```

### Step 2: query.ts 的 while 循环走到 attachment 阶段

**源文件**: `src/query.ts:302`

```typescript
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
    state.messages,        // 当前所有消息
    state.toolUseContext,  // 工具上下文
)
```

这不是在主线程跑的——是一个 `using` 绑定，异步 prefetch，不阻塞。

### Step 3: startRelevantMemoryPrefetch 做关闸检查

**源文件**: `src/utils/attachments.ts:2361`

```typescript
// 关闸 1: 自动记忆是否启用？
if (!isAutoMemoryEnabled()) return undefined

// 关闸 2: feature flag
if (!getFeatureValue('tengu_moth_copse', false)) return undefined

// 关闸 3: 最后一个用户消息
const lastUserMessage = messages.findLast(m => m.type === 'user' && !m.isMeta)

// 关闸 4: 消息太少（单字词不触发）
const input = getUserMessageText(lastUserMessage)
// input = "帮我给 webhook 回调加个 HMAC 签名校验"
if (!input || !/\s/.test(input.trim())) return undefined  // ← 至少有一个空格

// 关闸 5: 已经注入的记忆内容超过会话上限
const surfaced = collectSurfacedMemories(messages)
if (surfaced.totalBytes >= MAX_SESSION_BYTES) return undefined
```

五项检查通过 → 进入真正的查找。

### Step 4: 收集最近使用的工具名

```typescript
const recentTools = collectRecentSuccessfulTools(messages, lastUserMessage)
// 例如: ["Bash", "Read", "Bash", "Grep"]
```

这些工具名会传给 Sonnet，告诉它**不要选这些工具的使用文档**（因为模型已经在用了，再注入是噪音）。

### Step 5: findRelevantMemories 被调用

**源文件**: `src/memdir/findRelevantMemories.ts:39`

```typescript
export async function findRelevantMemories(
    query: string,           // "帮我给 webhook 回调加个 HMAC 签名校验"
    memoryDir: string,       // "~/.claude/projects/payment-api-abc123/memory/"
    signal: AbortSignal,
    recentTools: string[],   // ["Bash", "Read", "Bash", "Grep"]
    alreadySurfaced: Set<string>,  // 本轮已展示过的记忆文件路径
): Promise<RelevantMemory[]>
```

### Step 6: scanMemoryFiles 扫描目录

**源文件**: `src/memdir/memoryScan.ts:35`

```typescript
const entries = await readdir(memoryDir, { recursive: true })
// 得到: ["prefer-ts.md", "tech-stack.md", "webhook-requirement.md",
//        "nested/sub-dir/deploy-guide.md"]   ← recursive 支持子目录

const mdFiles = entries.filter(
    f => f.endsWith('.md') && basename(f) !== 'MEMORY.md'
)
// 排除 MEMORY.md（已经在 system prompt 的管道A中注入了）

// 每个文件读前30行 → 解析 YAML frontmatter → 单次 IO
const { content, mtimeMs } = await readFileInRange(filePath, 0, 30)
const { frontmatter } = parseFrontmatter(content)

// 返回 header 数组，按 mtime 倒序，最多 200 个
```

扫描结果（举例）：

```typescript
[
  {
    filename: "webhook-requirement.md",
    filePath: "/Users/zhangsan/.claude/projects/payment-api-abc123/memory/webhook-requirement.md",
    mtimeMs: 1719000000000,
    description: "webhook 需要 HMAC 签名校验",
    type: "project"
  },
  {
    filename: "prefer-ts.md",
    filePath: "/Users/zhangsan/.claude/projects/payment-api-abc123/memory/prefer-ts.md",
    mtimeMs: 1718900000000,
    description: "张三偏好 TypeScript",
    type: "user"
  },
  {
    filename: "tech-stack.md",
    filePath: "/Users/zhangsan/.claude/projects/payment-api-abc123/memory/tech-stack.md",
    mtimeMs: 1718800000000,
    description: "payment-api 技术栈",
    type: "project"
  },
  {
    filename: "avoid-nestjs.md",
    filePath: "/Users/zhangsan/.claude/projects/payment-api-abc123/memory/avoid-nestjs.md",
    mtimeMs: 1718700000000,
    description: "张三不喜欢 NestJS 的装饰器风格",
    type: "user"
  },
]
```

### Step 7: 过滤 alreadySurfaced

```typescript
const fresh = memories.filter(m => !alreadySurfaced.has(m.filePath))
```

`alreadySurfaced` 是本轮对话中已经作为 `<system-reminder>` 注入过的记忆文件路径集合。
这保证了不会重复注入——如果第 2 轮已经注入了 `prefer-ts.md`，第 3 轮就不会再选它。

### Step 8: selectRelevantMemories —— 调用 Sonnet 做选择

**源文件**: `src/memdir/findRelevantMemories.ts:77`

```typescript
// 1. 格式化 header 清单
const manifest = formatMemoryManifest(fresh)
// 输出:
//   - [project] webhook-requirement.md (2026-06-22T...): webhook 需要 HMAC 签名校验
//   - [user] prefer-ts.md (2026-06-21T...): 张三偏好 TypeScript
//   - [project] tech-stack.md (2026-06-20T...): payment-api 技术栈
//   - [user] avoid-nestjs.md (2026-06-19T...): 张三不喜欢 NestJS

// 2. 构建工具提示
const toolsSection = recentTools.length > 0
    ? `Recently used tools: Bash, Read, Bash, Grep`
    : ""

// 3. 调用 Sonnet
const result = await sideQuery({
    model: getDefaultSonnetModel(),  // ★ 专用 Sonnet，不是主模型
    system: SELECT_MEMORIES_SYSTEM_PROMPT,
    skipSystemPromptPrefix: true,
    messages: [{
        role: 'user',
        content: `Query: 帮我给 webhook 回调加个 HMAC 签名校验

Available memories:
- [project] webhook-requirement.md (...): webhook 需要 HMAC 签名校验
- [user] prefer-ts.md (...): 张三偏好 TypeScript
- [project] tech-stack.md (...): payment-api 技术栈
- [user] avoid-nestjs.md (...): 张三不喜欢 NestJS

Recently used tools: Bash, Read, Bash, Grep`,
    }],
    max_tokens: 256,
    output_format: {
        type: 'json_schema',
        schema: {
            type: 'object',
            properties: {
                selected_memories: { type: 'array', items: { type: 'string' } },
            },
            required: ['selected_memories'],
            additionalProperties: false,
        },
    },
    signal,
    querySource: 'memdir_relevance',
})
```

### Step 9: Sonnet 的选择逻辑

**这是 `SELECT_MEMORIES_SYSTEM_PROMPT` 的内容（完整原文）**:

```
You are selecting memories that will be useful to Claude Code
as it processes a user's query. You will be given the user's
query and a list of available memory files with their filenames
and descriptions.

Return a list of filenames for the memories that will clearly
be useful to Claude Code as it processes the user's query (up to 5).
Only include memories that you are certain will be helpful based
on their name and description.

- If you are unsure if a memory will be useful in processing
  the user's query, then do not include it in your list.
  Be selective and discerning.

- If there are no memories in the list that would clearly
  be useful, feel free to return an empty list.

- If a list of recently-used tools is provided, do not select
  memories that are usage reference or API documentation for
  those tools (Claude Code is already exercising them).
  DO still select memories containing warnings, gotchas, or
  known issues about those tools — active use is exactly when
  those matter.
```

Sonnet 返回：
```json
{
  "selected_memories": [
    "webhook-requirement.md",
    "tech-stack.md"
  ]
}
```

### Step 10: 验证 + 回查

```typescript
// 验证 Sonnet 返回的文件名确实存在于目录中
const validFilenames = new Set(memories.map(m => m.filename))
const selected = parsed.selected_memories
    .filter(f => validFilenames.has(f))  // 只保留实际存在的
    .map(filename => byFilename.get(filename))
    .filter(m => m !== undefined)

// 返回 { path, mtimeMs } 对
return selected.map(m => ({ path: m.filePath, mtimeMs: m.mtimeMs }))
// [
//   { path: "/Users/.../webhook-requirement.md", mtimeMs: 1719000000000 },
//   { path: "/Users/.../tech-stack.md", mtimeMs: 1718800000000 },
// ]
```

### Step 11: 读取记忆文件全文

**源文件**: `src/utils/attachments.ts` (getRelevantMemoryAttachments)

```typescript
// 对每个选中的文件，读取全文（有限制）
const content = await readFile(filePath)
// 限制: 单文件最大 MAX_BYTES_PER_MEMORY，总计 MAX_SESSION_BYTES
```

### Step 12: 包装为 system-reminder 附件

```typescript
const attachment = {
    type: 'relevant_memories',
    memories: [
        {
            name: 'webhook-requirement',
            type: 'project',
            content: 'webhook 回调需要 HMAC-SHA256 签名校验...'
        },
        {
            name: 'tech-stack',
            type: 'project',
            content: 'payment-api 使用 Express + TypeScript + Prisma + PostgreSQL'
        }
    ]
}
```

### Step 13: 注入到 messages 数组

在下一轮 API 调用时，messages 变成：

```typescript
messages = [
    // ... 之前的消息 ...

    // ★ 新注入的 system-reminder
    {
        role: 'user',
        content: `<system-reminder>
The following memories were surfaced as relevant to the
current conversation. They reflect background context written
before this conversation — verify information with the user
if something contradicts what you see.

<memory>
<name>webhook-requirement</name>
<type>project</type>
webhook 回调需要 HMAC-SHA256 签名校验，密钥从环境变量
WEBHOOK_SECRET 读取，签名放在 X-Signature header 中。
</memory>

<memory>
<name>tech-stack</name>
<type>project</type>
payment-api 使用 Express + TypeScript + Prisma + PostgreSQL，
测试框架是 Jest + Supertest。
</memory>
</system-reminder>`
    },

    // 用户真正的消息
    {
        role: 'user',
        content: '帮我给 webhook 回调加个 HMAC 签名校验'
    },
]
```

### Step 14: 模型"看到"了记忆

现在模型收到的 API 请求中，`messages` 数组已经包含了这段 `<system-reminder>`。模型读完这段后**自动知道**：
- 已经有 webhook 的上下文（签名方式、密钥来源）
- 项目用 Express + TypeScript + Jest
- 不需要用户重复这些信息

---

## 总结：三条管道 + 一个选择器

```
                 ┌────────────────────────────┐
                 │     管道A: system           │
                 │  (缓存，会话级，不变)        │
                 │                            │
                 │  "You have persistent       │
                 │   file-based memory..."     │
                 │  MEMORY.md 索引内容:         │
                 │  - prefer-ts.md            │
                 │  - tech-stack.md           │
                 │  - webhook-requirement.md  │
                 │  - avoid-nestjs.md         │
                 │  ← 模型知道"有哪些记忆文件"   │
                 └────────────────────────────┘

                 ┌────────────────────────────┐
                 │     管道B: messages         │
                 │  (每轮变化)                  │
                 │                            │
                 │  <system-reminder>          │
                 │    webhook-requirement.md   │
                 │    全文内容...               │
                 │    tech-stack.md            │
                 │    全文内容...               │
                 │  </system-reminder>         │
                 │  ← 模型看到"记忆里写了什么"   │
                 └────────────────────────────┘

  ┌──────────────────────────────────────────────────┐
  │             findRelevantMemories()               │
  │             (每轮运行，异步 prefetch)              │
  │                                                  │
  │  scanMemoryFiles()      selectRelevantMemories() │
  │  扫描目录→header清单  →  Sonnet 子查询→选 ≤5 个    │
  │                                                  │
  │  → 读全文 → 包装为 <system-reminder> → 注入消息     │
  └──────────────────────────────────────────────────┘
```

**核心洞察**: 记忆系统分了两次"看到"：
1. 管道A：模型看到**记忆索引**（有什么文件）——这是静态的，"图书馆目录"
2. 管道B：模型看到**记忆内容**（文件里写了什么）——这是动态的，"每次借 5 本书"

中间的选择器（Sonnet 子查询）就像图书管理员，根据用户的问题从目录中挑出最相关的几本书，把全文递到模型面前。
