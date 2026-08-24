# MCP 模块详解

## 什么是 MCP？

**MCP（Model Context Protocol）** 是一个让 AI 助手调用外部工具的标准协议。

### 通俗比喻

想象 AI 是一个刚入职的新员工：

- **没有 MCP**：ta 只能用公司自带的电脑和软件
- **有了 MCP**：ta 可以连接公司的数据库、调用各种 API、使用专业工具

```
                    MCP 协议
OpenCode ←──────────────────────→ 外部服务
   │                                  │
  AI 助手                          数据库查询
                                   API 调用
                                   专业工具
```

### MCP vs 内置工具

| 对比项   | 内置工具          | MCP 工具              |
| -------- | ----------------- | --------------------- |
| 来源     | OpenCode 自带     | 外部服务器            |
| 连接方式 | 直接调用          | 通过网络/进程         |
| 扩展性   | 需要改代码        | 只需配置              |
| 例子     | read、write、bash | 数据库、Slack、Notion |

---

## MCP 的工作原理

### 1. MCP Server（服务器）

MCP Server 是提供工具的"服务商"：

```
┌─────────────────────────────────────┐
│           MCP Server                │
│  ┌─────────────────────────────┐   │
│  │  工具1: 查询数据库           │   │
│  │  工具2: 发送 Slack 消息      │   │
│  │  工具3: 创建 Notion 页面     │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

### 2. 连接方式

MCP 支持两种连接方式：

**本地模式（local）**—— 启动本地进程

```
OpenCode → 启动子进程 → MCP Server
           ↑
       通过 stdio 通信
```

**远程模式（remote）**—— 连接网络服务

```
OpenCode → HTTP/SSE → 远程 MCP Server
           ↑
       通过网络通信
```

### 3. 通信流程

```
[1] OpenCode 启动时连接 MCP Server
         ↓
[2] 获取服务器提供的工具列表
         ↓
[3] AI 需要时调用某个工具
         ↓
[4] MCP Server 执行并返回结果
         ↓
[5] AI 继续处理
```

---

## 配置 MCP

在 `opencode.json` 中配置：

### 本地 MCP Server

```json
{
  "mcp": {
    "my-database": {
      "type": "local",
      "command": ["node", "mcp-server.js"],
      "environment": {
        "DATABASE_URL": "postgresql://..."
      }
    }
  }
}
```

### 远程 MCP Server

```json
{
  "mcp": {
    "notion": {
      "type": "remote",
      "url": "https://mcp.notion.so/v1",
      "headers": {
        "Authorization": "Bearer xxx"
      }
    }
  }
}
```

### 带 OAuth 认证的远程服务

```json
{
  "mcp": {
    "slack": {
      "type": "remote",
      "url": "https://mcp.slack.com/v1",
      "oauth": {
        "clientId": "your-client-id",
        "clientSecret": "your-client-secret",
        "scope": "chat:write,channels:read"
      }
    }
  }
}
```

---

## 文件结构

```
mcp/
├── index.ts           # MCP 核心（连接、工具调用）
├── auth.ts            # MCP 认证管理
├── oauth-provider.ts  # OAuth 提供者实现
├── oauth-callback.ts  # OAuth 回调处理
└── README.md          # 本文档
```

### index.ts —— MCP 核心

```typescript
export namespace MCP {
  // 添加 MCP 服务器
  export async function add(name: string, config: Config.Mcp)

  // 获取连接状态
  export async function status(): Promise<Record<string, Status>>

  // 获取所有 MCP 工具
  export async function tools(): Promise<Record<string, Tool>>

  // 连接服务器
  export async function connect(name: string)

  // 断开连接
  export async function disconnect(name: string)

  // OAuth 认证
  export async function authenticate(name: string): Promise<Status>
}

// 连接状态
export type Status =
  | { status: "connected" } // 已连接
  | { status: "disabled" } // 已禁用
  | { status: "failed"; error: string } // 连接失败
  | { status: "needs_auth" } // 需要认证
  | { status: "needs_client_registration"; error: string } // 需要注册
```

### auth.ts —— 认证管理

```typescript
export namespace McpAuth {
  // 保存 OAuth tokens
  export async function save(name: string, tokens: OAuthTokens)

  // 获取 tokens
  export async function get(name: string): Promise<OAuthTokens | undefined>

  // 删除认证
  export async function remove(name: string)

  // 检查 token 是否过期
  export async function isTokenExpired(name: string): Promise<boolean>
}
```

---

## 使用 MCP

### 1. 查看 MCP 状态

```bash
opencode mcp list
```

输出：

```
MCP Servers:
  notion      connected   3 tools
  slack       needs_auth  -
  database    failed      Connection refused
```

### 2. 连接 MCP 服务器

```bash
# 连接指定服务器
opencode mcp connect notion

# 如果需要 OAuth 认证
opencode mcp auth slack
```

### 3. 查看可用工具

```bash
opencode mcp tools
```

输出：

```
notion_create_page    创建 Notion 页面
notion_search         搜索 Notion
notion_update_page    更新页面
```

### 4. AI 使用 MCP 工具

当 AI 需要使用 MCP 工具时，会自动调用：

```
AI: "我来帮你创建一个 Notion 页面..."
    ↓
调用: notion_create_page({ title: "会议记录", ... })
    ↓
结果: { pageId: "xxx", url: "https://notion.so/xxx" }
```

---

## OAuth 认证流程

一些 MCP 服务器需要 OAuth 认证：

```
[1] 用户运行: opencode mcp auth slack
         ↓
[2] 浏览器打开授权页面
    "Slack 请求访问您的工作区..."
         ↓
[3] 用户点击"授权"
         ↓
[4] 回调到本地服务器
    http://localhost:xxx/callback?code=xxx
         ↓
[5] 交换 access token
         ↓
[6] 保存 token，连接成功！
```

### 代码实现

```typescript
// oauth-provider.ts
export class McpOAuthProvider implements OAuthClientProvider {
  // 生成授权 URL
  async redirectToAuthorization(authorizationUrl: URL)

  // 保存 tokens
  async saveTokens(tokens: OAuthTokens)

  // 获取 tokens
  async tokens(): Promise<OAuthTokens | undefined>
}

// oauth-callback.ts
export namespace McpOAuthCallback {
  // 启动回调服务器
  export async function ensureRunning()

  // 等待回调
  export async function waitForCallback(state: string): Promise<string>
}
```

---

## 工具转换

MCP 服务器返回的工具需要转换成 OpenCode 可用的格式：

```typescript
// MCP 原始工具定义
{
  name: "create_page",
  description: "创建 Notion 页面",
  inputSchema: {
    type: "object",
    properties: {
      title: { type: "string" },
      content: { type: "string" }
    }
  }
}

// 转换为 AI SDK 工具
dynamicTool({
  description: "创建 Notion 页面",
  inputSchema: jsonSchema({ ... }),
  execute: async (args) => {
    return client.callTool({
      name: "create_page",
      arguments: args
    })
  }
})
```

---

## 常见 MCP 服务器

### 官方/热门 MCP

| 名称           | 功能        | 类型   |
| -------------- | ----------- | ------ |
| Notion MCP     | 读写 Notion | remote |
| Slack MCP      | 发送消息    | remote |
| GitHub MCP     | 操作仓库    | remote |
| PostgreSQL MCP | 数据库查询  | local  |
| Filesystem MCP | 文件操作    | local  |

### 配置示例

```json
{
  "mcp": {
    "github": {
      "type": "remote",
      "url": "https://mcp.github.com",
      "oauth": true
    },
    "postgres": {
      "type": "local",
      "command": ["npx", "@modelcontextprotocol/server-postgres"],
      "environment": {
        "POSTGRES_URL": "postgresql://localhost:5432/mydb"
      }
    }
  }
}
```

---

## 常见问题

### Q: MCP 和 ACP 有什么区别？

```
IDE ←─ACP─→ OpenCode ←─MCP─→ 外部工具

ACP: IDE 调用 OpenCode（OpenCode 是服务端）
MCP: OpenCode 调用外部工具（OpenCode 是客户端）
```

### Q: 为什么连接失败？

常见原因：

1. **网络问题**：检查 URL 是否正确
2. **认证问题**：尝试 `opencode mcp auth xxx`
3. **超时**：增加 timeout 配置
4. **服务器未启动**：确认本地服务器已运行

```json
{
  "mcp": {
    "myserver": {
      "type": "remote",
      "url": "https://...",
      "timeout": 60000 // 增加超时时间
    }
  }
}
```

### Q: 如何调试 MCP？

```bash
# 查看连接状态
opencode mcp list

# 打印详细日志
opencode run --print-logs

# 检查日志文件
# 搜索 "mcp" 相关日志
```

### Q: OAuth 认证失败？

1. 检查 `clientId` 和 `clientSecret` 是否正确
2. 确认回调 URL 已在服务商处注册
3. 检查 scope 是否正确
4. 尝试 `opencode mcp auth --remove xxx` 后重新认证

---

## 开发自己的 MCP Server

### 1. 使用官方 SDK

```bash
npm install @modelcontextprotocol/sdk
```

### 2. 实现服务器

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server"

const server = new McpServer({
  name: "my-mcp-server",
  version: "1.0.0",
})

// 注册工具
server.tool("my_tool", {
  description: "我的工具",
  inputSchema: {
    type: "object",
    properties: {
      input: { type: "string" },
    },
  },
  handler: async (args) => {
    return { result: `处理了: ${args.input}` }
  },
})

// 启动服务器
server.listen()
```

### 3. 配置 OpenCode 使用

```json
{
  "mcp": {
    "my-server": {
      "type": "local",
      "command": ["node", "my-mcp-server.js"]
    }
  }
}
```

---

## 高级配置

### 禁用特定 MCP

```json
{
  "mcp": {
    "dangerous-server": {
      "type": "local",
      "command": ["..."],
      "enabled": false // 禁用
    }
  }
}
```

### 自定义超时

```json
{
  "mcp": {
    "slow-server": {
      "type": "remote",
      "url": "https://...",
      "timeout": 120000 // 2 分钟
    }
  }
}
```

### 全局 MCP 超时

```json
{
  "experimental": {
    "mcp_timeout": 60000
  }
}
```

---

## 事件和通知

MCP 模块通过 Bus 广播事件：

```typescript
// 工具列表变化
Bus.publish(MCP.ToolsChanged, { server: "notion" })

// 浏览器打开失败（需要手动复制链接）
Bus.publish(MCP.BrowserOpenFailed, {
  mcpName: "slack",
  url: "https://slack.com/oauth/authorize?...",
})
```

这让 UI 可以实时响应 MCP 状态变化。
