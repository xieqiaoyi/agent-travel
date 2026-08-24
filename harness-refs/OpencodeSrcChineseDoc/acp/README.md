# ACP 模块详解

## 什么是 ACP？

**ACP（Agent Client Protocol）** 是一个让 IDE 和 AI 助手通信的标准协议。

### 通俗比喻

想象你有一个翻译官：

- 你说中文（IDE 发送请求）
- 翻译官听懂了（ACP 协议解析）
- 翻译官用英文告诉 AI（转换为 OpenCode 内部调用）
- AI 回答了英文（OpenCode 处理结果）
- 翻译官翻译回中文（ACP 格式化响应）
- 你听到了答案（IDE 显示结果）

ACP 就是这个"翻译官"的工作手册。

### 为什么需要 ACP？

没有 ACP 时：

```
Zed 编辑器  →  需要专门写 Zed 适配器  →  OpenCode
VS Code     →  需要专门写 VS Code 适配器 →  OpenCode
Cursor      →  需要专门写 Cursor 适配器  →  OpenCode
```

有了 ACP 后：

```
Zed 编辑器  ─┐
VS Code     ─┼─  ACP 协议  →  OpenCode
Cursor      ─┘
```

一套协议，所有 IDE 都能用！

---

## 核心概念

### 1. JSON-RPC

ACP 使用 **JSON-RPC** 进行通信。这是一种用 JSON 格式传递"方法调用"的方式。

```json
// 请求：我想创建一个新会话
{
  "jsonrpc": "2.0",
  "method": "session/new",
  "params": {
    "cwd": "/home/user/project"
  },
  "id": 1
}

// 响应：好的，会话 ID 是 xxx
{
  "jsonrpc": "2.0",
  "result": {
    "sessionId": "sess_abc123"
  },
  "id": 1
}
```

### 2. ACP Agent（代理端）

在 ACP 协议中，OpenCode 扮演 **Agent（代理）** 角色：

- 接收 IDE 的请求
- 执行 AI 操作
- 返回结果给 IDE

```
IDE（客户端）  ←───JSON-RPC───→  OpenCode（代理端）
    ↓                                  ↓
  发送请求                         处理并响应
```

### 3. Session 映射

ACP 有自己的"会话"概念，OpenCode 也有自己的 Session。
这个模块负责在两者之间建立映射：

```
ACP 会话 ID: "acp_session_123"
    ↓ 映射
OpenCode Session ID: "sess_abc123"
```

---

## 文件结构

```
acp/
├── agent.ts      # ACP Agent 实现（核心！）
├── session.ts    # ACP 会话与 OpenCode Session 的映射
├── types.ts      # 类型定义
└── README.md     # 本文档
```

### agent.ts —— ACP 的核心

这是最重要的文件，实现了 ACP 协议的所有方法：

```typescript
export class Agent implements ACPAgent {
  // 初始化连接
  async initialize(params: InitializeRequest): Promise<InitializeResponse>

  // 创建新会话
  async newSession(params: NewSessionRequest)

  // 加载已有会话
  async loadSession(params: LoadSessionRequest)

  // 发送消息给 AI
  async prompt(params: PromptRequest)

  // 取消当前操作
  async cancel(params: CancelNotification)

  // ... 更多方法
}
```

### session.ts —— 会话管理器

负责维护 ACP 会话和 OpenCode Session 的对应关系：

```typescript
class ACPSessionManager {
  // 创建新会话
  async create(cwd: string, mcpServers: [], model: Model): ACPSession

  // 加载已有会话
  async load(sessionId: string, cwd: string, ...): ACPSession

  // 获取会话
  get(sessionId: string): ACPSession

  // 设置当前模型
  setModel(sessionId: string, model: Model)

  // 设置当前模式（Agent）
  setMode(sessionId: string, mode: string)
}
```

### types.ts —— 类型定义

```typescript
// ACP 配置
export interface ACPConfig {
  sdk: OpencodeClient // OpenCode SDK 客户端
  defaultModel?: {
    // 默认模型
    providerID: string
    modelID: string
  }
}

// ACP 会话状态
export interface ACPSession {
  id: string // 会话 ID
  cwd: string // 工作目录
  model?: Model // 当前模型
  modeId?: string // 当前 Agent 模式
}
```

---

## 工作流程详解

### 1. IDE 启动 OpenCode

```bash
# IDE 通过 stdio 启动 OpenCode 的 ACP 服务
opencode acp --cwd /path/to/project
```

### 2. 初始化握手

```
IDE                              OpenCode
 │                                   │
 │──── initialize ─────────────────→│
 │     {protocolVersion: 1}          │
 │                                   │
 │←─── response ─────────────────────│
 │     {agentCapabilities: {...}}    │
```

OpenCode 返回支持的功能：

- `loadSession: true` - 支持加载历史会话
- `mcpCapabilities` - 支持 MCP 工具
- `promptCapabilities` - 支持图片、嵌入内容

### 3. 创建会话

```
IDE                              OpenCode
 │                                   │
 │──── session/new ────────────────→│
 │     {cwd: "/project"}             │
 │                                   │
 │←─── response ─────────────────────│
 │     {sessionId: "...",            │
 │      models: [...],               │
 │      modes: [...]}                │
```

返回：

- 会话 ID
- 可用的 AI 模型列表
- 可用的 Agent 模式列表

### 4. 发送消息

```
IDE                              OpenCode
 │                                   │
 │──── session/prompt ─────────────→│
 │     {sessionId: "...",            │
 │      prompt: [{type:"text",       │
 │               text:"写排序函数"}]} │
 │                                   │
 │←─── sessionUpdate ────────────────│
 │     {agent_message_chunk: "好的"} │
 │                                   │
 │←─── sessionUpdate ────────────────│
 │     {tool_call: "write", ...}     │
 │                                   │
 │←─── sessionUpdate ────────────────│
 │     {tool_call_update: "完成"}    │
```

AI 的响应通过 **sessionUpdate** 事件流式返回。

### 5. 权限请求

当 AI 要做敏感操作时：

```
IDE                              OpenCode
 │                                   │
 │←─── permission.asked ─────────────│
 │     {permission: "write",         │
 │      metadata: {path: "..."}}     │
 │                                   │
 │──── permission/reply ───────────→│
 │     {reply: "once"}  (允许一次)   │
```

---

## 事件类型详解

ACP 通过事件通知 IDE 各种状态变化：

### sessionUpdate 事件

| 事件类型              | 含义          | 示例              |
| --------------------- | ------------- | ----------------- |
| `agent_message_chunk` | AI 输出文本   | "我来帮你写..."   |
| `agent_thought_chunk` | AI 的思考过程 | "让我分析一下..." |
| `tool_call`           | 工具调用开始  | 开始执行 write    |
| `tool_call_update`    | 工具执行更新  | write 执行完成    |
| `plan`                | 任务计划更新  | Todo 列表变化     |
| `user_message_chunk`  | 用户消息      | 用于回放          |

### 权限事件

| 事件类型           | 含义        |
| ------------------ | ----------- |
| `permission.asked` | AI 请求权限 |

---

## 实际使用示例

### 在 Zed 中使用

1. 安装 OpenCode
2. Zed 会自动检测 ACP 支持
3. 打开 AI 面板，选择 OpenCode
4. 开始对话！

### 在 VS Code 中使用

1. 安装 OpenCode 扩展
2. 扩展会启动 ACP 服务
3. 使用 AI 面板或命令面板

### 调试 ACP

```bash
# 启动 ACP 服务并打印日志
opencode acp --cwd . --print-logs

# 查看 JSON-RPC 通信
# 日志会显示所有请求和响应
```

---

## 代码示例

### 处理工具调用

```typescript
// 当 AI 调用工具时，agent.ts 会：
case "tool-call": {
  // 1. 通知 IDE 工具开始执行
  await this.connection.sessionUpdate({
    sessionId,
    update: {
      sessionUpdate: "tool_call",
      toolCallId: part.callID,
      title: part.tool,           // 工具名：read/write/bash
      kind: toToolKind(part.tool), // 工具类型
      status: "pending",
    },
  })

  // 2. 工具执行中...

  // 3. 通知 IDE 工具执行完成
  await this.connection.sessionUpdate({
    sessionId,
    update: {
      sessionUpdate: "tool_call_update",
      toolCallId: part.callID,
      status: "completed",
      content: [...],  // 执行结果
    },
  })
}
```

### 处理权限请求

```typescript
// 当需要用户授权时
case "permission.asked": {
  // 1. 向 IDE 请求权限
  const res = await this.connection.requestPermission({
    sessionId,
    toolCall: {
      toolCallId: permission.id,
      title: permission.permission,  // "write"
      kind: "edit",
    },
    options: [
      { optionId: "once", name: "允许一次" },
      { optionId: "always", name: "始终允许" },
      { optionId: "reject", name: "拒绝" },
    ],
  })

  // 2. 根据用户选择处理
  await this.sdk.permission.reply({
    requestID: permission.id,
    reply: res.outcome.optionId,  // "once" / "always" / "reject"
  })
}
```

---

## 常见问题

### Q: ACP 和 MCP 有什么区别？

**ACP（Agent Client Protocol）**：

- 用于 IDE ↔ AI 助手 通信
- 让 IDE 能使用 AI 功能
- OpenCode 是 Agent（服务端）

**MCP（Model Context Protocol）**：

- 用于 AI 助手 ↔ 外部工具 通信
- 让 AI 能使用外部功能（数据库、API 等）
- OpenCode 是 Client（客户端）

简单说：

```
IDE ←─ACP─→ OpenCode ←─MCP─→ 外部工具
```

### Q: 为什么 Windows 上 ACP 有问题？

Windows 的终端编码可能不是 UTF-8，会导致 JSON-RPC 消息被截断。
解决方案：

```powershell
# 在 PowerShell 中设置 UTF-8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

### Q: 如何查看 ACP 通信日志？

```bash
# 方法 1：启动时打印日志
opencode acp --print-logs

# 方法 2：查看日志文件
# Windows: %LOCALAPPDATA%\opencode\logs
# Mac/Linux: ~/.local/state/opencode/logs
```

---

## 开发者指南

### 添加新的 ACP 方法

1. 在 `agent.ts` 中实现方法
2. 处理请求参数
3. 调用 OpenCode 内部 API
4. 格式化响应

```typescript
// 示例：添加一个获取文件列表的方法
async listFiles(params: ListFilesRequest): Promise<ListFilesResponse> {
  const session = this.sessionManager.get(params.sessionId)

  // 调用内部 API
  const files = await this.sdk.file.list({
    directory: session.cwd,
    pattern: params.pattern,
  })

  // 返回 ACP 格式的响应
  return {
    files: files.map(f => ({
      path: f.path,
      type: f.isDirectory ? "directory" : "file",
    })),
  }
}
```

### 调试技巧

1. 使用 `log.info()` 记录关键信息
2. 在 IDE 端查看网络请求
3. 使用 `--print-logs` 实时查看日志
