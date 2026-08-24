# Bus 模块详解

## 什么是 Bus？

**Bus（事件总线）** 是模块间通信的"消息中心"。就像公司的公告板，谁有消息就贴上去，感兴趣的人自己来看。

### 通俗比喻

```
Session 模块: "我完成了一条消息！"
    ↓ 发布到 Bus
    ↓
Bus 广播
    ↓
CLI 收到: "好的，我显示出来"
TUI 收到: "好的，我更新界面"
Web 收到: "好的，我推送给浏览器"
```

---

## 核心概念

### 1. 发布/订阅模式

- **发布（Publish）**：发送事件
- **订阅（Subscribe）**：监听事件

```typescript
// 发布事件
Bus.publish(Session.Event.MessageUpdated, {
  sessionID: "xxx",
  messageID: "yyy",
})

// 订阅事件
Bus.subscribe(Session.Event.MessageUpdated, (payload) => {
  console.log(`消息 ${payload.messageID} 更新了`)
})
```

### 2. 事件定义

使用 `BusEvent.define` 定义带类型的事件：

```typescript
// 定义事件
export const MessageUpdated = BusEvent.define(
  "session.message.updated", // 事件名
  z.object({
    // payload schema
    sessionID: z.string(),
    messageID: z.string(),
  }),
)

// 使用时有完整类型提示
Bus.publish(MessageUpdated, {
  sessionID: "xxx", // ✓ 类型安全
  messageID: "yyy",
})
```

---

## 常用事件

### Session 相关

| 事件                      | 触发时机     |
| ------------------------- | ------------ |
| `session.message.updated` | 消息更新     |
| `session.part.updated`    | 消息部件更新 |
| `session.error`           | 发生错误     |

### Permission 相关

| 事件                 | 触发时机   |
| -------------------- | ---------- |
| `permission.asked`   | 请求权限   |
| `permission.replied` | 权限已回复 |

### Server 相关

| 事件                       | 触发时机 |
| -------------------------- | -------- |
| `server.instance.disposed` | 实例销毁 |

---

## 文件结构

```
bus/
├── bus-event.ts  # 事件定义
├── index.ts      # Bus 实现
├── global.ts     # 全局事件桥接
└── README.md     # 本文档
```

### index.ts 核心 API

```typescript
export namespace Bus {
  // 发布事件
  export function publish<T>(event: BusEvent<T>, payload: T)

  // 订阅事件
  export function subscribe<T>(event: BusEvent<T>, handler: (payload: T) => void): () => void // 返回取消订阅函数

  // 订阅所有事件
  export function subscribeAll(handler: (event: string, payload: any) => void)
}
```

---

## 与 SSE 集成

Bus 事件会通过 Server-Sent Events 推送给 Web 客户端：

```
Bus 事件
    ↓
server/event.ts 订阅
    ↓
转换为 SSE 格式
    ↓
推送给浏览器/TUI

浏览器端:
const source = new EventSource('/event')
source.onmessage = (e) => {
  const event = JSON.parse(e.data)
  // 处理事件...
}
```

---

## 使用示例

### 监听消息更新

```typescript
import { Bus } from "@/bus"
import { Session } from "@/session"

// 订阅
const unsubscribe = Bus.subscribe(Session.Event.PartUpdated, (payload) => {
  if (payload.delta) {
    // 流式输出，显示增量
    process.stdout.write(payload.delta)
  }
})

// 完成后取消订阅
unsubscribe()
```

### 监听权限请求

```typescript
Bus.subscribe(Permission.Event.Asked, async (payload) => {
  const answer = await promptUser(`允许 ${payload.permission} 操作吗？`)

  await Permission.reply({
    id: payload.id,
    reply: answer ? "allow" : "deny",
  })
})
```

---

## 注意事项

1. **事件处理要快**：不要在 handler 中执行耗时操作
2. **payload 必须可序列化**：因为可能要通过 SSE 传输
3. **记得取消订阅**：避免内存泄漏
