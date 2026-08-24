# Prompt 完整组装流程

本文档详细说明 OpenCode 如何组装发送给 AI 的完整提示词。

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    发送给 AI 的完整内容                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   System Prompt                          │   │
│  │  (系统提示词，定义 AI 的角色和行为规范)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            +                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Messages                               │   │
│  │  (对话历史 + 当前用户输入)                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            +                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Tools                                  │   │
│  │  (可用工具列表，每个工具有 description 和 parameters)    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 一、System Prompt 组装

### 组装位置

主要在 `session/llm.ts` 的 `stream()` 函数中：

```typescript
const system = []
system.push([
  // 1. Agent prompt 或 Provider prompt
  ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
  // 2. 额外系统提示词
  ...input.system,
  // 3. 用户消息中的自定义系统提示词
  ...(input.user.system ? [input.user.system] : []),
].join("\n"))

// 4. 插件可修改
await Plugin.trigger("experimental.chat.system.transform", { ... }, { system })
```

### 组装流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    System Prompt 组装流程                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. 基础系统提示词                                        │   │
│  │    - Agent 自定义 prompt（如果有）                       │   │
│  │    - 否则根据模型选择: anthropic.txt / beast.txt / ...   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. 环境信息                                              │   │
│  │    SystemPrompt.environment(model)                       │   │
│  │    - 模型名称、工作目录、平台、日期                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. 指令文件                                              │   │
│  │    InstructionPrompt.system()                            │   │
│  │    - AGENTS.md / CLAUDE.md                               │   │
│  │    - config.instructions 中的文件/URL                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 4. 用户自定义 system（如果有）                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 5. 插件修改                                              │   │
│  │    Plugin: experimental.chat.system.transform            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 各部分详解

#### 1. 基础系统提示词

**来源**: `session/system.ts` 的 `SystemPrompt.provider()`

根据模型类型选择不同的提示词文件：

| 模型                   | 提示词文件             |
| ---------------------- | ---------------------- |
| Claude (Anthropic)     | `prompt/anthropic.txt` |
| Claude (AWS Bedrock)   | `prompt/anthropic.txt` |
| Claude (Google Vertex) | `prompt/anthropic.txt` |
| Gemini                 | `prompt/gemini.txt`    |
| OpenAI o1/o3/o4-mini   | `prompt/beast.txt`     |
| 其他 OpenAI            | `prompt/openai.txt`    |
| DeepSeek reasoning     | `prompt/beast.txt`     |
| 其他                   | `prompt/default.txt`   |

```typescript
// system.ts
export function provider(model: Provider.Model) {
  if (model.api.id === "anthropic") return [PROMPT_ANTHROPIC, ...tool()]
  if (model.api.id === "google-generative-ai") return [PROMPT_GEMINI, ...tool()]
  // ...
}
```

#### 2. 环境信息

**来源**: `session/system.ts` 的 `SystemPrompt.environment()`

注入运行时环境信息：

```typescript
export async function environment(model: Provider.Model) {
  return [
    `You are powered by the model ${model.name}. The exact model ID is ${model.providerID}/${model.id}`,
    `<env>`,
    `  Working directory: ${Instance.directory}`,
    `  Is directory a git repo: ${Instance.project.vcs === "git" ? "yes" : "no"}`,
    `  Platform: ${process.platform}`,
    `  Today's date: ${new Date().toDateString()}`,
    `</env>`,
    // 最近修改的文件列表
    `<files>`,
    ...recentFiles,
    `</files>`,
  ]
}
```

#### 3. 指令文件

**来源**: `session/instruction.ts` 的 `InstructionPrompt.system()`

加载项目中的指令文件：

```typescript
// 搜索顺序
const INSTRUCTION_FILES = [
  "AGENTS.md", // 优先
  "CLAUDE.md", // 兼容 Claude Code
  "CURSOR.md", // 兼容 Cursor
  "COPILOT.md", // 兼容 Copilot
  ".cursorrules", // 兼容旧格式
  // ...
]

// 搜索位置
// 1. 项目根目录
// 2. .opencode/ 目录
// 3. config.instructions 中指定的文件/URL
```

输出格式：

```
Instructions from: /path/to/AGENTS.md
<instruction content>
```

#### 4. 用户自定义 system

用户在发送消息时可以指定额外的系统提示词：

```typescript
// API 调用时
await Session.prompt({
  sessionID: "...",
  system: "额外的系统指令",  // 可选
  parts: [...]
})
```

#### 5. 插件修改

插件可以通过 hook 修改系统提示词：

```typescript
// 插件示例
"experimental.chat.system.transform": async (input, output) => {
  output.system.push("Remember to be concise.")
}
```

---

## 二、Messages 组装

### 组装位置

主要在 `session/prompt.ts` 的 `loop()` 函数中。

### 组装流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Messages 组装流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 6. 消息预处理                                            │   │
│  │    insertReminders()                                     │   │
│  │    - Plan 模式提示词                                     │   │
│  │    - Build 切换提示词                                    │   │
│  │    - 队列消息的 <system-reminder> 包装                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 7. 文件/附件展开                                         │   │
│  │    createUserMessage()                                   │   │
│  │    - 文件 → synthetic "Called Read tool" + 内容          │   │
│  │    - 目录 → synthetic "Called list tool" + 列表          │   │
│  │    - @agent → "调用 task 子代理" 提示                    │   │
│  │    - MCP 资源 → 读取内容或错误信息                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 8. 消息转换                                              │   │
│  │    MessageV2.toModelMessages()                           │   │
│  │    - 历史对话（user/assistant/tool）                     │   │
│  │    - 当前用户输入                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 9. 步骤限制检查                                          │   │
│  │    - 如果是最后一步，追加 max-steps.txt                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 10. 插件最终修改                                         │   │
│  │     Plugin: experimental.chat.messages.transform         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 各部分详解

#### 6. 消息预处理 (insertReminders)

**来源**: `session/prompt.ts` 的 `insertReminders()`

根据当前模式注入特殊提示：

```typescript
// Plan 模式
if (agent.name === "plan") {
  userMessage.parts.push({
    type: "text",
    text: PROMPT_PLAN, // plan.txt 内容
    synthetic: true,
  })
}

// 从 Plan 切换到 Build
if (wasPlan && agent.name === "build") {
  userMessage.parts.push({
    type: "text",
    text: BUILD_SWITCH, // build-switch.txt 内容
    synthetic: true,
  })
}

// 多步骤时，包装队列中的用户消息
if (step > 1) {
  part.text = [
    "<system-reminder>",
    "The user sent the following message:",
    part.text,
    "",
    "Please address this message and continue with your tasks.",
    "</system-reminder>",
  ].join("\n")
}
```

#### 7. 文件/附件展开 (createUserMessage)

**来源**: `session/prompt.ts` 的 `createUserMessage()`

将用户附加的文件/目录/agent 展开为消息部件：

```typescript
// 文件 → 调用 Read 工具读取内容
if (part.mime === "text/plain") {
  pieces.push({
    type: "text",
    synthetic: true,
    text: `Called the Read tool with the following input: ${JSON.stringify(args)}`,
  })
  const result = await ReadTool.execute(args, ctx)
  pieces.push({
    type: "text",
    synthetic: true,
    text: result.output,
  })
}

// 目录 → 调用 list 工具列出内容
if (part.mime === "application/x-directory") {
  const result = await ListTool.execute(args, ctx)
  pieces.push({
    type: "text",
    synthetic: true,
    text: `Called the list tool with the following input: ${JSON.stringify(args)}`,
  })
  // ...
}

// @agent → 提示调用子代理
if (part.type === "agent") {
  pieces.push({
    type: "text",
    synthetic: true,
    text: " Use the above message and context to generate a prompt and call the task tool with subagent: " + part.name,
  })
}

// MCP 资源 → 读取资源内容
if (part.source?.type === "resource") {
  const resourceContent = await MCP.readResource(clientName, uri)
  // ...
}
```

#### 8. 消息转换 (toModelMessages)

**来源**: `session/message-v2.ts` 的 `MessageV2.toModelMessages()`

将内部消息格式转换为 AI SDK 需要的格式：

```typescript
// 输入: MessageV2.WithParts[]
// 输出: ModelMessage[]

[
  { role: "user", content: [...] },
  { role: "assistant", content: [...] },
  { role: "tool", content: [...] },
  // ...
]
```

#### 9. 步骤限制检查

**来源**: `session/prompt.ts` 的 `loop()` 函数

```typescript
const maxSteps = agent.steps ?? Infinity
const isLastStep = step >= maxSteps

// 如果是最后一步，追加提示
messages: [
  ...MessageV2.toModelMessages(sessionMessages, model),
  ...(isLastStep
    ? [
        {
          role: "assistant",
          content: MAX_STEPS, // max-steps.txt: "你必须在这一步完成任务..."
        },
      ]
    : []),
]
```

#### 10. 插件最终修改

```typescript
await Plugin.trigger("experimental.chat.messages.transform", {}, { messages: sessionMessages })
```

---

## 三、Tools 组装

### 组装位置

主要在 `session/prompt.ts` 的 `resolveTools()` 函数中。

### 工具来源

```
Tools
├── 内置工具 (ToolRegistry)
│   ├── read, write, edit, bash, glob, grep...
│   ├── task, todowrite, todoread
│   ├── webfetch, websearch, codesearch
│   └── skill (description 中包含 <available_skills> 列表)
│
├── MCP 工具 (MCP.tools())
│   └── 从配置的 MCP 服务器获取
│
└── 自定义工具
    ├── config 目录下的 tool/*.ts
    └── 插件提供的工具
```

### 工具注册流程

```typescript
// prompt.ts: resolveTools()
const tools: Record<string, AITool> = {}

// 1. 内置工具
for (const item of await ToolRegistry.tools(model, agent)) {
  tools[item.id] = tool({
    description: item.description,
    inputSchema: jsonSchema(schema),
    execute: async (args, options) => {
      // ...
    },
  })
}

// 2. MCP 工具
for (const [key, item] of Object.entries(await MCP.tools())) {
  tools[key] = item
}

return tools
```

### Skills 特殊处理

**来源**: `tool/skill.ts`

Skills 信息是通过 skill 工具的 **description** 传递给 AI 的：

```typescript
const description = [
  "Load a skill to get detailed instructions for a specific task.",
  "Skills provide specialized knowledge and step-by-step guidance.",
  "Use this when a task matches an available skill's description.",
  "Only the skills listed here are available:",
  "<available_skills>",
  ...accessibleSkills.flatMap((skill) => [
    `  <skill>`,
    `    <name>${skill.name}</name>`,
    `    <description>${skill.description}</description>`,
    `  </skill>`,
  ]),
  "</available_skills>",
].join(" ")
```

---

## 四、最终发送

### 发送位置

`session/llm.ts` 的 `stream()` 函数：

```typescript
return streamText({
  // 系统提示词
  messages: [...system.map((x) => ({ role: "system", content: x })), ...input.messages],
  // 工具
  tools,
  // 其他参数
  temperature: params.temperature,
  topP: params.topP,
  maxOutputTokens,
  // ...
})
```

### 完整数据结构

```typescript
{
  // System Prompt (数组，会被拼接)
  system: [
    "You are Claude, an AI assistant...",  // 基础提示词
    "<env>Working directory: /project...</env>",  // 环境信息
    "Instructions from: /project/AGENTS.md\n...",  // 指令文件
  ],

  // Messages (对话历史)
  messages: [
    { role: "user", content: "帮我写一个排序函数" },
    { role: "assistant", content: "好的，我来..." },
    { role: "tool", content: { toolCallId: "...", result: "..." } },
    // ...
  ],

  // Tools (可用工具)
  tools: {
    read: { description: "...", inputSchema: {...}, execute: fn },
    write: { description: "...", inputSchema: {...}, execute: fn },
    skill: { description: "...<available_skills>...</available_skills>", ... },
    mcp_server_tool: { description: "...", ... },
    // ...
  },
}
```

---

## 五、调试方法

### 导出完整提示词

设置环境变量：

```bash
OPENCODE_DEBUG_PROMPT=1 opencode
```

会在 `/tmp/opencode-prompt-{sessionID}.json` 生成完整的提示词文件：

```json
{
  "sessionID": "sess_xxx",
  "model": { ... },
  "agent": { ... },
  "system": ["...", "..."],
  "messages": [...],
  "tools": ["read", "write", "skill", ...]
}
```

### 查看日志

```bash
tail -f ~/.local/state/opencode/logs/*.log | grep -E "(llm|prompt|system)"
```

---

## 六、相关文件索引

| 文件                     | 作用                                 |
| ------------------------ | ------------------------------------ |
| `session/llm.ts`         | System Prompt 组装、调用 AI          |
| `session/system.ts`      | 环境信息、Provider 提示词选择        |
| `session/instruction.ts` | 指令文件加载                         |
| `session/prompt.ts`      | Messages 组装、Tools 解析            |
| `session/message-v2.ts`  | 消息格式转换                         |
| `tool/registry.ts`       | 内置工具注册                         |
| `tool/skill.ts`          | Skill 工具（description 含技能列表） |
| `mcp/index.ts`           | MCP 工具获取                         |
| `plugin/index.ts`        | 插件 hook 触发                       |
| `prompt/*.txt`           | 各模型的基础提示词                   |
