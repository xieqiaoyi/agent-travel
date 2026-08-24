# Util 模块详解

## 什么是 Util？

**Util（工具）** 目录包含整个项目共用的基础工具函数。这些都是"幕后英雄"，虽然不直接面对用户，但几乎所有模块都在使用它们。

---

## 文件列表

| 文件            | 功能       | 使用场景         |
| --------------- | ---------- | ---------------- |
| `log.ts`        | 日志系统   | 调试、监控       |
| `context.ts`    | 上下文管理 | 跨函数传递数据   |
| `lock.ts`       | 互斥锁     | 防止并发冲突     |
| `queue.ts`      | 任务队列   | 按序执行任务     |
| `defer.ts`      | 延迟执行   | 类似 Go 的 defer |
| `lazy.ts`       | 惰性初始化 | 按需加载         |
| `timeout.ts`    | 超时控制   | 限制操作时间     |
| `signal.ts`     | 信号处理   | AbortSignal 工具 |
| `filesystem.ts` | 文件系统   | 路径处理         |
| `token.ts`      | Token 计算 | 估算 token 数量  |
| `format.ts`     | 格式化     | 数字、时间格式化 |
| `color.ts`      | 颜色处理   | 终端颜色         |
| `iife.ts`       | 立即执行   | 代码组织         |
| `fn.ts`         | 函数工具   | 通用函数         |
| `wildcard.ts`   | 通配符     | 模式匹配         |
| `keybind.ts`    | 键绑定     | 快捷键处理       |
| `locale.ts`     | 本地化     | 语言处理         |
| `archive.ts`    | 归档       | 压缩解压         |
| `eventloop.ts`  | 事件循环   | 异步控制         |
| `rpc.ts`        | RPC 工具   | 远程调用         |
| `scrap.ts`      | 抓取工具   | 网页处理         |

---

## 核心工具详解

### log.ts —— 日志系统

```typescript
import { Log } from "@/util/log"

// 创建 logger
const log = Log.create({ service: "my-module" })

// 记录日志
log.info("操作完成", { id: 123 })
log.error("发生错误", { error: e })
log.debug("调试信息", { data: xxx })

// 计时
{
  using _ = log.time("operation")
  // 耗时操作...
} // 自动输出: "operation took 123ms"
```

### context.ts —— 上下文管理

类似 React 的 Context，用于跨函数传递数据：

```typescript
import { Context } from "@/util/context"

// 定义 context
const UserContext = Context.create<{ id: string }>()

// 提供值
Context.provide(UserContext, { id: "user123" }, async () => {
  // 在这个范围内可以获取值
  const user = Context.get(UserContext)
  console.log(user.id) // "user123"
})
```

**原理**：基于 Node/Bun 的 `AsyncLocalStorage`，让异步调用链能访问同一个上下文。

### lock.ts —— 互斥锁

防止并发操作冲突：

```typescript
import { Lock } from "@/util/lock"

const lock = Lock.create()

// 只有一个操作能同时执行
await lock.acquire(async () => {
  await writeFile(...)
})
```

### timeout.ts —— 超时控制

```typescript
import { withTimeout } from "@/util/timeout"

// 限制操作时间
const result = await withTimeout(
  fetchData(),
  30000, // 30 秒超时
)
```

### token.ts —— Token 估算

```typescript
import { estimateTokens } from "@/util/token"

const count = estimateTokens("Hello, world!")
// 约 4 tokens
```

### defer.ts —— 延迟执行

类似 Go 语言的 defer，用于清理：

```typescript
import { Defer } from "@/util/defer"

async function process() {
  const defer = Defer.create()

  const file = await openFile()
  defer.add(() => file.close()) // 确保最后关闭

  const conn = await connect()
  defer.add(() => conn.disconnect())

  // 处理逻辑...

  await defer.run() // 执行所有清理
}
```

### lazy.ts —— 惰性初始化

只在第一次使用时初始化：

```typescript
import { Lazy } from "@/util/lazy"

const config = Lazy.create(async () => {
  return await loadConfig() // 只加载一次
})

const cfg = await config.get() // 第一次会加载
const cfg2 = await config.get() // 直接返回缓存
```

### wildcard.ts —— 通配符匹配

```typescript
import { matchWildcard } from "@/util/wildcard"

matchWildcard("src/**/*.ts", "src/foo/bar.ts") // true
matchWildcard("*.js", "app.ts") // false
```

---

## 使用建议

1. **日志要有意义**：记录关键操作和错误，方便调试
2. **善用 Lock**：写文件、修改状态时防止并发问题
3. **设置合理超时**：避免操作无限等待
4. **用 defer 做清理**：确保资源被正确释放
5. **不要循环依赖**：util 不应依赖业务模块

---

## 常见问题

### Q: 日志文件在哪里？

```bash
# 查看日志路径
opencode --print-logs

# 日志位置
# Windows: %LOCALAPPDATA%\opencode\logs
# Mac/Linux: ~/.local/state/opencode/logs
```

### Q: 如何调整日志级别？

```bash
opencode run --log-level DEBUG
```

或设置环境变量：

```bash
export LOG_LEVEL=DEBUG
```
