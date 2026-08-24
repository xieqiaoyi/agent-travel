# Scheduler 模块详解

## 什么是 Scheduler？

**Scheduler（调度器）** 是一个周期性任务调度系统，用于管理需要定期运行的后台任务。

### 当前用途

目前主要用于**清理任务**：

| 任务                      | 间隔  | 作用                          |
| ------------------------- | ----- | ----------------------------- |
| `tool.truncation.cleanup` | 1小时 | 清理超过7天的工具输出截断文件 |
| `snapshot.cleanup`        | 1小时 | 对快照 git 仓库执行 gc 清理   |

---

## 核心概念

### 作用域

```
┌─────────────────────────────────────────────────────────┐
│                      Scheduler                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   ┌─────────────────────────────────────────────────┐  │
│   │  global（全局作用域）                            │  │
│   │  - 所有项目实例共享                              │  │
│   │  - 例如：truncation.cleanup                     │  │
│   └─────────────────────────────────────────────────┘  │
│                                                         │
│   ┌─────────────────────────────────────────────────┐  │
│   │  instance（实例作用域）                          │  │
│   │  - 每个项目独立                                  │  │
│   │  - 实例销毁时自动清理                            │  │
│   │  - 例如：snapshot.cleanup                       │  │
│   └─────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 任务定义

```typescript
interface Task {
  id: string // 任务唯一标识
  interval: number // 执行间隔（毫秒）
  run: () => Promise<void> // 执行函数
  scope?: "instance" | "global" // 作用域，默认 "instance"
}
```

---

## 工作机制

### 注册流程

```
Scheduler.register(task)
       ↓
确定作用域（global 或 instance）
       ↓
检查是否已存在（global 任务只注册一次）
       ↓
立即执行一次
       ↓
设置 setInterval 定时器
       ↓
timer.unref()  // 不阻止进程退出
```

### 清理机制

```
Instance 销毁
       ↓
遍历 instance 作用域的所有定时器
       ↓
clearInterval(timer)
       ↓
清空任务注册表
```

---

## 文件结构

```
scheduler/
├── index.ts      # Scheduler 实现
└── README.md     # 本文档
```

### 核心 API

```typescript
export namespace Scheduler {
  // 注册周期性任务
  export function register(task: Task): void
}
```

---

## 使用示例

### 注册清理任务

```typescript
// snapshot/index.ts
Scheduler.register({
  id: "snapshot.cleanup",
  scope: "instance",
  interval: 60 * 60 * 1000, // 1小时
  async run() {
    await $`git gc --prune=7.days`.cwd(repo).quiet()
  },
})
```

### 注册全局任务

```typescript
// tool/truncation.ts
Scheduler.register({
  id: "tool.truncation.cleanup",
  scope: "global",
  interval: 60 * 60 * 1000,
  async run() {
    // 清理超过7天的截断文件
    const files = await glob("*.txt", { cwd: outputDir })
    for (const file of files) {
      const stat = await Bun.file(file).stat()
      if (Date.now() - stat.mtime > 7 * 24 * 60 * 60 * 1000) {
        await fs.unlink(file)
      }
    }
  },
})
```

---

## 当前开发状态

| 功能         | 状态      |
| ------------ | --------- |
| 基础定时执行 | ✅ 已实现 |
| 双层作用域   | ✅ 已实现 |
| 生命周期管理 | ✅ 已实现 |
| 任务队列     | ❌ 未实现 |
| 重试策略     | ❌ 未实现 |
| 并发控制     | ❌ 未实现 |

### 与 Session 的关系

Scheduler 与 Session **没有直接关系**。两者共享 Instance 上下文，但独立运行：

- Session：处理用户对话
- Scheduler：运行后台清理任务

未来可能用于：

- 会话步骤调度
- 工具调用限流
- 长时间任务管理

---

## 实现细节

### timer.unref()

```typescript
const timer = setInterval(() => run(task), task.interval)
timer.unref() // 关键！
```

`unref()` 使定时器不阻止 Node.js 进程退出。
这样当用户退出 OpenCode 时，不会因为定时器而卡住。

### 错误处理

```typescript
async function run(task: Task) {
  await task.run().catch((error) => {
    log.error("run failed", { id: task.id, error })
  })
}
```

任务失败只记录日志，不影响后续调度。

---

## 常见问题

### Q: 为什么需要清理任务？

- **截断文件**：工具输出过长时会保存到临时文件，需要定期清理
- **Git 快照**：快照仓库会积累大量历史，需要 gc 释放空间

### Q: 任务会并发执行吗？

是的，当前实现没有并发控制。如果任务执行时间超过间隔，
下一次执行会在上一次还在运行时开始。

### Q: 如何调试任务？

查看日志中 `scheduler` 相关条目：

```bash
opencode run --print-logs
# 搜索 "scheduler"
```
