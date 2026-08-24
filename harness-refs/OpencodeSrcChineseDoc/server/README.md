# Server 模块详解

## 什么是 Server？

**Server（服务器）** 是 OpenCode 的 HTTP API 服务。TUI、Web 界面、IDE 插件都通过它与 OpenCode 核心通信。

### 通俗比喻

Server 就像一个"服务窗口"：

- CLI 直接使用内部函数
- TUI/Web/IDE 通过 HTTP API 调用

```
┌─────────────┐
│     CLI     │ ← 直接调用
├─────────────┤
│    TUI      │ ← HTTP API
├─────────────┤
│    Web      │ ← HTTP API
├─────────────┤
│    IDE      │ ← HTTP API
└─────────────┘
        ↓
   ┌─────────┐
   │ Server  │ ← Hono + Bun
   └─────────┘
```

---

## 技术栈

- **Bun**：高性能 JavaScript 运行时
- **Hono**：轻量 Web 框架（类似 Express）
- **SSE**：Server-Sent Events，用于实时推送

---

## API 路由

| 路由            | 功能          |
| --------------- | ------------- |
| `/session/*`    | 会话管理      |
| `/project/*`    | 项目信息      |
| `/config/*`     | 配置管理      |
| `/permission/*` | 权限管理      |
| `/mcp/*`        | MCP 管理      |
| `/file/*`       | 文件操作      |
| `/event`        | SSE 事件流    |
| `/pty/*`        | 伪终端        |
| `/provider/*`   | Provider 信息 |

---

## 文件结构

```
server/
├── server.ts       # 主服务器
├── error.ts        # 错误处理
├── event.ts        # SSE 事件
├── mdns.ts         # mDNS 发现
└── routes/         # 路由实现
    ├── session.ts
    ├── project.ts
    ├── config.ts
    ├── permission.ts
    ├── mcp.ts
    ├── file.ts
    ├── pty.ts
    ├── provider.ts
    ├── global.ts
    ├── tui.ts
    └── experimental.ts
```

---

## 核心概念

### 1. Hono 路由

```typescript
import { Hono } from "hono"

const app = new Hono()

app.get("/session/list", async (c) => {
  const sessions = await Session.list()
  return c.json(sessions)
})

app.post("/session/prompt", async (c) => {
  const body = await c.req.json()
  await Session.prompt(body)
  return c.json({ ok: true })
})
```

### 2. SSE 事件流

通过 Server-Sent Events 推送实时事件：

```typescript
// event.ts
app.get("/event", async (c) => {
  return streamSSE(c, async (stream) => {
    Bus.subscribeAll((event, payload) => {
      stream.writeSSE({
        event,
        data: JSON.stringify(payload),
      })
    })
  })
})
```

客户端接收：

```javascript
const source = new EventSource("http://localhost:10101/event")
source.onmessage = (e) => {
  const { type, payload } = JSON.parse(e.data)
  // 处理事件...
}
```

### 3. mDNS 发现

让局域网内的设备能发现 OpenCode 服务：

```typescript
// mdns.ts
export namespace MDNS {
  export async function publish(port: number) {
    // 发布 _opencode._tcp 服务
  }
}
```

---

## API 示例

### 会话 API

```bash
# 列出会话
GET /session/list?directory=/path/to/project

# 创建会话
POST /session/create
{ "directory": "/path/to/project" }

# 发送消息
POST /session/prompt
{
  "sessionID": "xxx",
  "parts": [{ "type": "text", "text": "hello" }],
  "model": { "providerID": "anthropic", "modelID": "claude-sonnet-4-5" }
}

# 取消操作
POST /session/abort
{ "sessionID": "xxx" }
```

### 配置 API

```bash
# 获取配置
GET /config?directory=/path

# 获取 Provider 列表
GET /config/providers?directory=/path
```

### 权限 API

```bash
# 回复权限请求
POST /permission/reply
{
  "requestID": "xxx",
  "reply": "once"  // "once" | "always" | "reject"
}
```

---

## 启动服务器

### 自动启动

运行 `opencode run` 时会自动启动服务器。

### 手动启动

```bash
opencode serve --port 10101
```

### 环境变量

```bash
# 指定端口
OPENCODE_PORT=10101

# 指定 host（默认 localhost）
OPENCODE_HOST=0.0.0.0
```

---

## SDK 生成

修改 API 后需要重新生成 SDK：

```bash
./script/generate.ts
```

这会更新 `@opencode-ai/sdk` 包，供 TUI 和其他客户端使用。

---

## 错误处理

所有 API 错误都统一格式：

```json
{
  "error": {
    "name": "SessionNotFoundError",
    "message": "Session xxx not found",
    "data": {
      "sessionID": "xxx"
    }
  }
}
```

```typescript
// error.ts
app.onError((err, c) => {
  if (err instanceof NamedError) {
    return c.json({ error: err.toObject() }, 400)
  }
  return c.json({ error: { message: err.message } }, 500)
})
```

---

## 常见问题

### Q: 端口被占用？

```bash
# 指定其他端口
opencode serve --port 10102

# 或使用环境变量
OPENCODE_PORT=10102 opencode run
```

### Q: 如何从外部访问？

默认只监听 localhost，要允许外部访问：

```bash
OPENCODE_HOST=0.0.0.0 opencode serve
```

**注意**：这会暴露到网络，请确保安全。生产环境建议配置密码和 CORS。

### Q: 在企业网络无法访问？

检查防火墙是否允许 Bun 使用的端口。
