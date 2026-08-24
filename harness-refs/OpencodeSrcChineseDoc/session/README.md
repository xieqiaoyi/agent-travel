# Session 模块详解

## 什么是 Session？

**Session（会话）** 是你和 AI 的一次完整对话。就像微信聊天记录一样，Session 会保存：

- 你说的每一句话
- AI 的每一条回复
- AI 调用的每个工具
- 文件的每次改动

### 通俗比喻

Session 就像一本"对话日记"：

```
📖 会话日记 #sess_abc123
─────────────────────────
[10:00] 👤 用户：帮我写一个排序函数
[10:01] 🤖 AI：好的，我来写一个快速排序...
        📝 调用 write 工具，创建 sort.js
        ✅ 文件已创建
[10:02] 👤 用户：改成从大到小排序
[10:03] 🤖 AI：没问题，我修改一下...
        📝 调用 edit 工具，修改 sort.js
        ✅ 修改完成
```

---

## 核心概念

### 1. Message（消息）

一条消息代表对话中的一个发言：

```typescript
// 用户消息
{
  role: "user",
  parts: [
    { type: "text", text: "帮我写一个排序函数" }
  ]
}

// AI 消息
{
  role: "assistant",
  parts: [
    { type: "text", text: "好的，我来写..." },
    { type: "tool", tool: "write", state: { status: "completed" } }
  ]
}
```

### 2. Part（消息部件）

一条消息可以由多个部件组成：

| 类型          | 说明     | 示例                 |
| ------------- | -------- | -------------------- |
| `text`        | 文本内容 | "我来帮你写代码"     |
| `tool`        | 工具调用 | 调用 read/write/bash |
| `file`        | 文件附件 | 上传的图片/文档      |
| `reasoning`   | 思考过程 | Claude 的 thinking   |
| `step-start`  | 步骤开始 | AI 开始新一轮思考    |
| `step-finish` | 步骤结束 | 包含 token 用量      |
| `patch`       | 文件改动 | 记录文件 diff        |

### 3. Compaction（压缩）

当对话太长时，会超出 AI 的"记忆容量"（上下文限制）。
**Compaction** 会把旧的对话压缩成摘要：

```
压缩前（100 条消息，50000 tokens）：
- 消息1: "帮我写排序"
- 消息2: "好的，我来写..."
- ...100条详细对话...

压缩后（1 条摘要，2000 tokens）：
- 摘要: "用户请求实现排序功能，已完成快速排序的编写，
        包含升序和降序两个版本，文件保存在 sort.js"
```

### 4. Snapshot（快照）

每次 AI 开始工作时，会对当前文件状态拍个"快照"。
这样可以：

- 追踪文件变化
- 支持撤销操作
- 生成分享内容

---

## 文件结构

```
session/
├── index.ts        # 会话核心（创建、更新、查询）
├── processor.ts    # 对话处理器（调用 AI、处理工具）
├── prompt.ts       # 提示词构建
├── message.ts      # 消息结构（旧版，兼容用）
├── message-v2.ts   # 消息结构（新版）
├── llm.ts          # AI 模型调用
├── compaction.ts   # 会话压缩
├── summary.ts      # 会话摘要
├── instruction.ts  # 系统指令
├── retry.ts        # 重试逻辑
├── revert.ts       # 撤销功能
├── status.ts       # 会话状态
├── system.ts       # 系统提示词
├── todo.ts         # 任务列表
└── README.md       # 本文档
```

---

## 核心文件详解

### index.ts —— 会话管理中心

```typescript
export namespace Session {
  // 创建新会话
  export async function create(input: {
    projectID: string
    title?: string
    parentID?: string // 如果是 fork 的会话
  }): Promise<Info>

  // 获取会话
  export async function get(sessionID: string): Promise<Info>

  // 列出所有会话
  export async function list(): Promise<Info[]>

  // 更新消息
  export async function updateMessage(message: MessageV2.Info)

  // 更新消息部件
  export async function updatePart(part: MessageV2.Part)

  // 发送消息（用户输入）
  export async function prompt(input: {
    sessionID: string
    parts: MessageV2.Part[]
    model: Provider.Model
    agent: string
  })
}
```

### processor.ts —— 对话处理器（最重要！）

这是整个系统的"引擎"，负责：

1. 接收用户输入
2. 调用 AI 模型
3. 处理 AI 响应
4. 执行工具调用
5. 保存结果

```typescript
export namespace SessionProcessor {
  export function create(input: {
    assistantMessage: MessageV2.Assistant
    sessionID: string
    model: Provider.Model
    abort: AbortSignal
  }) {
    return {
      // 处理一轮对话
      async process(streamInput: LLM.StreamInput) {
        while (true) {
          // 调用 AI 获取流式响应
          const stream = await LLM.stream(streamInput)

          for await (const value of stream.fullStream) {
            switch (value.type) {
              case "text-delta":
                // AI 输出文本
                break
              case "tool-call":
                // AI 调用工具
                break
              case "tool-result":
                // 工具执行完成
                break
              // ...
            }
          }
        }
      },
    }
  }
}
```

### prompt.ts —— 提示词构建器

把各种信息组装成 AI 能理解的提示词：

```typescript
// 最终发给 AI 的提示词结构
;[
  // 系统提示词
  { role: "system", content: "你是一个编程助手..." },

  // 历史对话
  { role: "user", content: "帮我写排序" },
  { role: "assistant", content: "好的..." },

  // 当前问题
  { role: "user", content: "改成降序" },
]
```

### llm.ts —— AI 调用封装（核心！）

这是调用 AI 模型的"中转站"，负责：

- 组装系统提示词
- 合并各种配置
- 处理工具权限
- 发送请求给 AI

#### 参数合并优先级

配置从多个地方来，按优先级合并：

```
base（基础配置）
    ↓ 覆盖
model.options（模型配置）
    ↓ 覆盖
agent.options（Agent 配置）
    ↓ 覆盖
variant（用户选择的变体）
```

```typescript
const options = pipe(
  base, // 1. Provider 的默认配置
  mergeDeep(input.model.options), // 2. 模型特定配置
  mergeDeep(input.agent.options), // 3. Agent 配置（如 build/plan）
  mergeDeep(variant), // 4. 用户选择的变体
)
```

#### 系统提示词组装

```typescript
const system = []
system.push([
  // 1. Agent 的自定义提示词，否则用 Provider 的默认提示词
  ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),

  // 2. 调用时传入的额外提示词
  ...input.system,

  // 3. 用户消息中的自定义系统提示词
  ...(input.user.system ? [input.user.system] : []),
].join("\n"))

// 4. 插件可以修改系统提示词
await Plugin.trigger("experimental.chat.system.transform", { ... }, { system })
```

#### 工具权限过滤

```typescript
async function resolveTools(input) {
  // 获取被 Agent 权限禁用的工具
  const disabled = PermissionNext.disabled(Object.keys(input.tools), input.agent.permission)

  // 移除被禁用的工具
  for (const tool of Object.keys(input.tools)) {
    if (input.user.tools?.[tool] === false || disabled.has(tool)) {
      delete input.tools[tool] // 不给 AI 用这个工具
    }
  }
  return input.tools
}
```

#### LiteLLM 兼容性处理

有些 AI 代理（如 LiteLLM）要求：如果历史消息里有工具调用，请求里必须有 tools 参数。

```typescript
// 检测是否是 LiteLLM 代理
const isLiteLLMProxy = provider.options?.["litellmProxy"] === true || providerID.toLowerCase().includes("litellm")

// 如果历史消息有工具调用但当前没有工具，添加一个假工具
if (isLiteLLMProxy && Object.keys(tools).length === 0 && hasToolCalls(messages)) {
  tools["_noop"] = tool({
    description: "Placeholder for LiteLLM proxy compatibility",
    execute: async () => ({ output: "", title: "", metadata: {} }),
  })
}
```

#### 工具名修复

AI 有时候会把工具名弄错（比如大小写错误），OpenCode 会自动修复：

```typescript
async experimental_repairToolCall(failed) {
  // 尝试小写匹配
  const lower = failed.toolCall.toolName.toLowerCase()
  if (lower !== failed.toolCall.toolName && tools[lower]) {
    return {
      ...failed.toolCall,
      toolName: lower,  // 修复：Write → write
    }
  }

  // 实在修不了，返回 invalid
  return {
    ...failed.toolCall,
    toolName: "invalid",
  }
}
```

#### 调试提示词

设置环境变量 `OPENCODE_DEBUG_PROMPT=1` 可以导出完整的提示词：

```bash
OPENCODE_DEBUG_PROMPT=1 opencode

# 会输出到 /tmp/opencode-prompt-{sessionID}.json
# 包含：system、messages、tools 等所有信息
```

### compaction.ts —— 会话压缩

当会话太长时自动压缩，就像手机存储满了要清理旧照片一样。

#### 核心常量

```typescript
const PRUNE_MINIMUM = 20_000 // 至少要清理 20K tokens 才值得动手
const PRUNE_PROTECT = 40_000 // 保护最近 40K tokens 的工具调用结果
const PRUNE_PROTECTED_TOOLS = ["skill"] // skill 工具永远不清理（技能很重要！）
```

#### 三大核心函数

**1. `isOverflow()` - 检测是否"内存不够用"**

```typescript
export async function isOverflow(input: { tokens: MessageV2.Assistant["tokens"]; model: Provider.Model }) {
  // 如果配置禁用了自动压缩，返回 false
  if (config.compaction?.auto === false) return false

  // 如果模型没有上下文限制，返回 false
  if (input.model.limit.context === 0) return false

  // 计算已使用的 tokens
  const count = input.tokens.input + input.tokens.cache.read + input.tokens.output

  // 计算可用空间 = 总上下文 - 预留给输出的空间
  const output = Math.min(input.model.limit.output, 32000) || 32000
  const usable = input.model.limit.input || context - output

  // 用量 > 可用空间 → 需要压缩！
  return count > usable
}
```

**2. `prune()` - "修剪"旧的工具调用结果**

这个函数很聪明：它**只清理工具调用的输出结果**，保留对话本身。

```
修剪前：
  消息1: "帮我读一下 package.json"
  工具调用: read → { 输出: 5000 行内容 }  ← 这个会被清理
  消息2: "文件里有 lodash 依赖"

修剪后：
  消息1: "帮我读一下 package.json"
  工具调用: read → "[Old tool result content cleared]"  ← 只保留标记
  消息2: "文件里有 lodash 依赖"
```

工作流程：

```
从后往前遍历消息
    ↓
跳过最近 2 轮对话（保护最新内容）
    ↓
统计工具调用输出的 tokens
    ↓
超过 40K tokens 的部分 → 标记为"已压缩"
    ↓
至少清理 20K tokens 才执行（避免小打小闹）
```

**3. `process()` - 执行真正的压缩（生成摘要）**

当 prune 还不够用时，就要生成摘要了：

```typescript
// 使用专门的 "compaction" agent 来生成摘要
const agent = await Agent.get("compaction")

// 发送所有历史消息给 AI，让它总结
const defaultPrompt = `
  Provide a detailed prompt for continuing our conversation above.
  Focus on information that would be helpful for continuing the conversation,
  including what we did, what we're doing, which files we're working on,
  and what we're going to do next...
`

// 生成的摘要会标记 summary: true
const msg = await Session.updateMessage({
  ...
  summary: true,  // 标记这是摘要消息
  ...
})
```

### message-v2.ts —— 消息数据结构

定义消息的各种类型：

```typescript
export namespace MessageV2 {
  // 用户消息
  export interface User {
    id: string
    role: "user"
    sessionID: string
    parts: Part[]
  }

  // AI 消息
  export interface Assistant {
    id: string
    role: "assistant"
    sessionID: string
    agent: string
    parts: Part[]
    cost: number // 花费（美元）
    tokens: TokenUsage // token 用量
    error?: Error // 如果出错
  }

  // 消息部件类型
  export type Part = TextPart | ToolPart | FilePart | ReasoningPart | StepStartPart | StepFinishPart | PatchPart
}
```

---

### summary.ts —— 会话摘要生成

负责两种摘要：**会话级摘要**和**消息级摘要**。

#### 会话级摘要（`summarizeSession`）

统计整个会话改动了多少文件：

```typescript
// 收集所有 patch 部件中涉及的文件
const files = new Set(
  messages
    .flatMap((x) => x.parts)
    .filter((x) => x.type === "patch")
    .flatMap((x) => x.files),
)

// 计算 diff（新增/删除了多少行）
const diffs = await computeDiff({ messages })

// 更新会话摘要
await Session.update(sessionID, (draft) => {
  draft.summary = {
    additions: diffs.reduce((sum, x) => sum + x.additions, 0), // 总新增行数
    deletions: diffs.reduce((sum, x) => sum + x.deletions, 0), // 总删除行数
    files: diffs.length, // 改动文件数
  }
})
```

#### 消息级摘要（`summarizeMessage`）

为每条用户消息生成一个简短标题：

```typescript
// 使用 "title" agent 生成标题（用小模型，省钱！）
const agent = await Agent.get("title")
const model = agent.model
  ? await Provider.getModel(agent.model.providerID, agent.model.modelID)
  : await Provider.getSmallModel(userMsg.model.providerID)  // 优先用小模型

// 发送消息内容给 AI，让它生成标题
const stream = await LLM.stream({
  ...
  messages: [{
    role: "user",
    content: `The following is the text to summarize: <text>${textPart.text}</text>`
  }],
})

// 保存标题
userMsg.summary.title = await stream.text
```

#### diff 计算（`computeDiff`）

```typescript
// 找到这次对话的起始快照和结束快照
let from: string | undefined // step-start 中的 snapshot
let to: string | undefined // step-finish 中的 snapshot

// 比较两个快照之间的差异
if (from && to) return Snapshot.diffFull(from, to)
```

---

### revert.ts —— 撤销功能

就像 Ctrl+Z 一样，可以撤销 AI 的改动。

#### 三大功能

**1. `revert()` - 执行撤销**

```
用户说："撤销到消息 msg_005"
    ↓
收集 msg_005 之后的所有 patch（文件改动记录）
    ↓
创建当前状态的快照（以便取消撤销）
    ↓
逆向应用所有 patches，恢复文件
    ↓
更新 session.revert 记录撤销状态
```

```typescript
export async function revert(input: RevertInput) {
  // 确保没有正在进行的对话
  SessionPrompt.assertNotBusy(input.sessionID)

  // 收集要撤销的 patches
  const patches: Snapshot.Patch[] = []
  for (const msg of all) {
    if (msg.info.id >= input.messageID) {
      for (const part of msg.parts) {
        if (part.type === "patch") {
          patches.push(part)
        }
      }
    }
  }

  // 保存当前状态快照（用于 unrevert）
  revert.snapshot = await Snapshot.track()

  // 执行撤销
  await Snapshot.revert(patches)
}
```

**2. `unrevert()` - 取消撤销（Ctrl+Shift+Z）**

```typescript
export async function unrevert(input: { sessionID: string }) {
  const session = await Session.get(input.sessionID)
  if (!session.revert) return session

  // 恢复到撤销前的快照
  if (session.revert.snapshot) {
    await Snapshot.restore(session.revert.snapshot)
  }

  // 清除撤销状态
  await Session.update(sessionID, (draft) => {
    draft.revert = undefined
  })
}
```

**3. `cleanup()` - 清理撤销状态**

当用户继续对话时，需要真正删除被撤销的消息：

```typescript
export async function cleanup(session: Session.Info) {
  // 删除被撤销的消息
  for (const msg of remove) {
    await Storage.remove(["message", sessionID, msg.info.id])
    await Bus.publish(MessageV2.Event.Removed, { ... })
  }

  // 删除部分消息中被撤销的 parts
  for (const part of removeParts) {
    await Storage.remove(["part", messageID, part.id])
    await Bus.publish(MessageV2.Event.PartRemoved, { ... })
  }
}
```

---

### todo.ts —— 任务列表

存储 AI 的任务列表（通过 TodoWrite 工具使用）。

```typescript
export namespace Todo {
  // 任务结构
  export const Info = z.object({
    id: z.string(), // 唯一标识
    content: z.string(), // 任务描述
    status: z.string(), // pending | in_progress | completed | cancelled
    priority: z.string(), // high | medium | low
  })

  // 更新任务列表
  export async function update(input: { sessionID: string; todos: Info[] }) {
    await Storage.write(["todo", sessionID], input.todos)
    Bus.publish(Event.Updated, input) // 通知 UI 更新
  }

  // 获取任务列表
  export async function get(sessionID: string) {
    return Storage.read<Info[]>(["todo", sessionID])
  }
}
```

**与 TodoWrite 工具的关系：**

```
AI 调用 TodoWrite 工具
    ↓
Tool.execute("todowrite", { todos: [...] })
    ↓
Todo.update({ sessionID, todos })
    ↓
Bus.publish(Event.Updated)
    ↓
UI 收到事件，显示任务列表
```

---

## 完整工作流程

### 1. 用户输入 → AI 响应

```
用户输入: "帮我写一个 hello world"
           ↓
    [1] Session.prompt()
           ↓
    [2] 创建用户消息 (MessageV2.User)
           ↓
    [3] 创建 AI 消息占位 (MessageV2.Assistant)
           ↓
    [4] SessionProcessor.process()
           ↓
    [5] LLM.stream() 调用 AI
           ↓
    [6] 处理流式响应
           ↓
    [7] AI 输出: "好的，我来写..."
           ↓
    [8] AI 调用 write 工具
           ↓
    [9] 执行工具，创建文件
           ↓
    [10] 保存结果，更新 Session
```

### 2. 工具调用流程

```
AI 说: "我要创建一个文件"
           ↓
    [1] 解析 tool-call 事件
        { tool: "write", args: { path: "hello.js", content: "..." } }
           ↓
    [2] 检查权限
        Permission.check("write", "hello.js")
           ↓
    [3] 如果需要确认，等待用户
        "AI 想要创建 hello.js，允许吗？[Y/n]"
           ↓
    [4] 执行工具
        Tool.execute("write", args)
           ↓
    [5] 返回结果给 AI
        { success: true, message: "文件已创建" }
           ↓
    [6] AI 继续思考或结束
```

### 3. 会话压缩流程

```
对话进行中...tokens 累积...
           ↓
    [1] 检查 token 用量
        isOverflow() → true，超过限制了！
           ↓
    [2] 触发压缩
        SessionCompaction.compact()
           ↓
    [3] 调用小模型生成摘要
        "总结这段对话的要点..."
           ↓
    [4] 用摘要替换旧消息
        100条消息 → 1条摘要
           ↓
    [5] 继续对话
```

---

## 数据存储

Session 数据保存在 `.opencode/state/session/` 目录：

```
.opencode/state/session/
├── proj_xxx/              # 项目目录
│   ├── sess_abc123/       # 会话目录
│   │   ├── info.json      # 会话元信息
│   │   ├── messages/      # 消息
│   │   │   ├── msg_001.json
│   │   │   └── msg_002.json
│   │   └── parts/         # 消息部件
│   │       ├── part_001.json
│   │       └── part_002.json
│   └── sess_def456/
│       └── ...
```

### info.json 结构

```json
{
  "id": "sess_abc123",
  "projectID": "proj_xxx",
  "title": "排序函数实现",
  "time": {
    "created": 1699000000000,
    "updated": 1699001000000
  },
  "share": {
    "url": "https://share.opencode.ai/xxx"
  }
}
```

---

## 事件系统

Session 通过 Bus 广播事件，让其他模块（如 UI）知道状态变化：

```typescript
// 消息更新事件
Bus.publish(Session.Event.MessageUpdated, {
  sessionID: "sess_abc123",
  messageID: "msg_001",
})

// 部件更新事件
Bus.publish(Session.Event.PartUpdated, {
  sessionID: "sess_abc123",
  part: { ... },
  delta: "新增的文本",  // 流式输出时的增量
})

// 错误事件
Bus.publish(Session.Event.Error, {
  sessionID: "sess_abc123",
  error: { name: "RateLimitError", ... },
})
```

---

## 常见问题

### Q: Session 和 Chat 有什么区别？

Chat 只是"聊天"，Session 还包括：

- 工具调用记录
- 文件改动追踪
- 权限管理
- 可分享、可恢复

### Q: 为什么需要 Compaction？

AI 模型有上下文长度限制（比如 Claude 是 200K tokens）。
超过限制后 AI 就"记不住"之前的对话了。
Compaction 把旧对话压缩成摘要，节省空间。

### Q: 如何查看会话历史？

```bash
# 列出所有会话
opencode session list

# 继续某个会话
opencode session resume sess_abc123
```

### Q: 数据存在哪里？

- 会话数据：`.opencode/state/session/`
- 日志文件：`~/.local/state/opencode/logs/`

---

## 开发者指南

### 添加新的消息部件类型

1. 在 `message-v2.ts` 定义类型
2. 在 `processor.ts` 处理该类型
3. 在 UI 层添加渲染

```typescript
// 1. 定义类型
export interface MyCustomPart {
  type: "my-custom"
  data: any
}

// 2. 处理类型
case "my-custom-event":
  await Session.updatePart({
    type: "my-custom",
    data: value.data,
  })
  break
```

### 处理 AI 流式响应

```typescript
// processor.ts 中的关键代码
for await (const value of stream.fullStream) {
  switch (value.type) {
    case "text-delta":
      // 文本增量，追加到当前文本
      currentText.text += value.text
      await Session.updatePart({ part: currentText, delta: value.text })
      break

    case "tool-call":
      // 工具调用开始
      const part = await Session.updatePart({
        type: "tool",
        tool: value.toolName,
        state: { status: "running", input: value.input },
      })
      break

    case "tool-result":
      // 工具执行完成
      await Session.updatePart({
        ...part,
        state: { status: "completed", output: value.output },
      })
      break
  }
}
```

### 调试技巧

1. 使用 `--print-logs` 查看详细日志
2. 检查 `.opencode/state/session/` 的 JSON 文件
3. 在 `processor.ts` 添加 `log.info()` 追踪流程
