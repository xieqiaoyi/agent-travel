# Share 模块详解

## 什么是 Share？

**Share（分享）** 让你可以把 OpenCode 的对话分享给其他人。即使对方没有安装 OpenCode，也能通过链接查看：

- 完整的对话历史
- AI 的代码改动（diff）
- 文件变化统计

### 通俗比喻

```
你的 OpenCode 对话
      ↓
创建分享链接
      ↓
https://opncd.ai/share/abc123
      ↓
同事打开链接
      ↓
看到完整的对话和代码改动
```

就像发送聊天记录截图，但更加强大——对方能看到完整的上下文和代码 diff。

---

## 核心概念

### 1. Share 信息结构

```typescript
interface ShareInfo {
  id: string // 分享 ID
  url: string // 分享链接
  secret: string // 访问密钥（用于更新和删除）
}
```

### 2. 同步数据类型

```typescript
type Data =
  | { type: "session"; data: Session } // 会话信息
  | { type: "message"; data: Message } // 消息
  | { type: "part"; data: Part } // 消息部件
  | { type: "session_diff"; data: FileDiff[] } // 文件 diff
  | { type: "model"; data: Model[] } // 使用的模型
```

---

## 核心函数

### `create()` - 创建分享链接

```typescript
export async function create(sessionID: string) {
  // 1. 检查是否禁用
  if (disabled) return { id: "", url: "", secret: "" }

  // 2. 向分享服务请求创建
  const result = await fetch(`${await url()}/api/share`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ sessionID }),
  }).then((x) => x.json())

  // result = { id: "abc123", url: "https://opncd.ai/share/abc123", secret: "xyz789" }

  // 3. 保存分享信息
  await Storage.write(["session_share", sessionID], result)

  // 4. 触发完整同步
  fullSync(sessionID)

  return result
}
```

### `remove()` - 取消分享

```typescript
export async function remove(sessionID: string) {
  // 1. 获取分享信息
  const share = await get(sessionID)
  if (!share) return

  // 2. 调用删除 API
  await fetch(`${await url()}/api/share/${share.id}`, {
    method: "DELETE",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ secret: share.secret }),
  })

  // 3. 删除本地记录
  await Storage.remove(["session_share", sessionID])
}
```

### `sync()` - 增量同步数据

```typescript
async function sync(sessionID: string, data: Data[]) {
  // 1. 检查是否有分享
  const share = await get(sessionID)
  if (!share) return

  // 2. 防抖：1秒内的多次更新合并为一次请求
  const existing = queue.get(sessionID)
  if (existing) {
    // 添加到现有队列
    for (const item of data) {
      existing.data.set(item.id, item)
    }
    return
  }

  // 3. 创建新队列，1秒后发送
  const timeout = setTimeout(async () => {
    const queued = queue.get(sessionID)
    queue.delete(sessionID)

    await fetch(`${await url()}/api/share/${share.id}/sync`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        secret: share.secret,
        data: Array.from(queued.data.values()),
      }),
    })
  }, 1000)

  queue.set(sessionID, { timeout, data: new Map(data.map((d) => [d.id, d])) })
}
```

### `fullSync()` - 完整同步

创建分享时，同步所有历史数据：

```typescript
async function fullSync(sessionID: string) {
  // 获取所有数据
  const session = await Session.get(sessionID)
  const diffs = await Session.diff(sessionID)
  const messages = await Array.fromAsync(MessageV2.stream(sessionID))
  const models = await Promise.all(
    messages
      .filter((m) => m.info.role === "user")
      .map((m) => Provider.getModel(m.info.model.providerID, m.info.model.modelID)),
  )

  // 一次性发送所有数据
  await sync(sessionID, [
    { type: "session", data: session },
    ...messages.map((x) => ({ type: "message", data: x.info })),
    ...messages.flatMap((x) => x.parts.map((y) => ({ type: "part", data: y }))),
    { type: "session_diff", data: diffs },
    { type: "model", data: models },
  ])
}
```

---

## 事件订阅

Share 模块会自动订阅相关事件，实时同步更新：

```typescript
export async function init() {
  // 会话更新
  Bus.subscribe(Session.Event.Updated, async (evt) => {
    await sync(evt.properties.info.id, [{
      type: "session",
      data: evt.properties.info,
    }])
  })

  // 消息更新
  Bus.subscribe(MessageV2.Event.Updated, async (evt) => {
    await sync(evt.properties.info.sessionID, [{
      type: "message",
      data: evt.properties.info,
    }])
    // 如果是用户消息，同时同步模型信息
    if (evt.properties.info.role === "user") {
      const model = await Provider.getModel(...)
      await sync(sessionID, [{ type: "model", data: [model] }])
    }
  })

  // 部件更新
  Bus.subscribe(MessageV2.Event.PartUpdated, async (evt) => {
    await sync(evt.properties.part.sessionID, [{
      type: "part",
      data: evt.properties.part,
    }])
  })

  // Diff 更新
  Bus.subscribe(Session.Event.Diff, async (evt) => {
    await sync(evt.properties.sessionID, [{
      type: "session_diff",
      data: evt.properties.diff,
    }])
  })
}
```

---

## 工作流程

### 创建分享

```
用户说："分享这个会话"
      ↓
create(sessionID)
      │
      │  POST /api/share
      │  返回: { id, url, secret }
      ↓
保存到本地
      │
      │  Storage.write(["session_share", sessionID], result)
      ↓
触发完整同步
      │
      │  fullSync(sessionID)
      │  → 发送所有历史数据
      ↓
返回分享链接
      │
      │  "https://opncd.ai/share/abc123"
```

### 实时同步

```
用户发送消息
      ↓
AI 回复
      ↓
Session.Event.Updated 触发
      ↓
sync() 被调用
      │
      │  加入队列（防抖）
      ↓
1秒后
      │
      │  POST /api/share/{id}/sync
      │  发送队列中的所有数据
      ↓
分享页面更新
```

---

## 数据存储

```
.opencode/state/
└── session_share/
    └── sess_xxx.json    # 分享信息
```

### session_share JSON 结构

```json
{
  "id": "abc123",
  "url": "https://opncd.ai/share/abc123",
  "secret": "xyz789secret"
}
```

---

## 配置选项

### 分享服务地址

```json
{
  "enterprise": {
    "url": "https://your-company.opncd.ai"
  }
}
```

默认使用 `https://opncd.ai`。

### 禁用分享

```bash
OPENCODE_DISABLE_SHARE=1 opencode
```

或

```bash
OPENCODE_DISABLE_SHARE=true opencode
```

---

## 安全机制

### Secret 密钥

每个分享都有一个 secret，用于：

- 更新数据（sync）
- 删除分享

```typescript
// 更新时需要 secret
await fetch(`/api/share/${id}/sync`, {
  body: JSON.stringify({
    secret: share.secret,  // 验证身份
    data: [...],
  }),
})

// 删除时需要 secret
await fetch(`/api/share/${id}`, {
  method: "DELETE",
  body: JSON.stringify({ secret: share.secret }),
})
```

### 数据范围

分享只包含：

- 对话历史
- 代码 diff
- 模型信息

不包含：

- 完整文件内容
- 敏感信息
- 环境变量

---

## 与其他模块的关系

```
Session
   │
   ├── 提供会话数据
   ├── 提供消息数据
   └── 提供 diff 数据
   ↓
Share
   │
   ├── 订阅事件
   ├── 同步数据
   └── 管理分享链接
   ↓
分享服务 (opncd.ai)
   │
   └── 存储并展示分享页面
```

---

## 常见问题

### Q: 分享链接会过期吗？

目前不会自动过期，需要手动删除。

### Q: 分享的数据是实时的吗？

是的，通过事件订阅实现实时同步。但有 1 秒的防抖延迟。

### Q: 可以撤销分享吗？

可以，使用 `ShareNext.remove(sessionID)` 或通过 UI 取消分享。

### Q: 企业环境如何使用？

配置 `enterprise.url` 指向企业私有部署的分享服务。

---

## 调试技巧

1. **查看分享信息**

   ```bash
   cat .opencode/state/session_share/sess_xxx.json | jq
   ```

2. **检查同步状态**

   ```bash
   # 查看日志
   tail -f ~/.local/state/opencode/logs/*.log | grep share
   ```

3. **手动触发同步**

   ```typescript
   await ShareNext.create(sessionID)
   ```

4. **测试分享 API**
   ```bash
   # 检查分享是否可访问
   curl https://opncd.ai/api/share/abc123
   ```
