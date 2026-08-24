# CLI 模块详解

## 什么是 CLI？

**CLI（Command Line Interface）** 是 OpenCode 的命令行界面。你在终端输入的 `opencode xxx` 命令都在这里实现。

---

## 可用命令

### 核心命令

| 命令                  | 说明                 |
| --------------------- | -------------------- |
| `opencode run`        | 启动对话（默认命令） |
| `opencode run "消息"` | 直接发送消息         |

### 会话管理

| 命令                           | 说明     |
| ------------------------------ | -------- |
| `opencode session list`        | 列出会话 |
| `opencode session resume <id>` | 恢复会话 |

### 认证管理

| 命令                              | 说明          |
| --------------------------------- | ------------- |
| `opencode auth login <provider>`  | 登录 Provider |
| `opencode auth list`              | 列出已登录    |
| `opencode auth logout <provider>` | 登出          |

### 模型管理

| 命令                   | 说明         |
| ---------------------- | ------------ |
| `opencode models list` | 列出可用模型 |

### MCP 管理

| 命令                       | 说明            |
| -------------------------- | --------------- |
| `opencode mcp list`        | 列出 MCP 服务器 |
| `opencode mcp auth <name>` | MCP OAuth 认证  |

### 其他

| 命令               | 说明          |
| ------------------ | ------------- |
| `opencode upgrade` | 升级 OpenCode |
| `opencode stats`   | 查看使用统计  |
| `opencode --help`  | 查看帮助      |

---

## 文件结构

```
cli/
├── cmd/              # 命令实现
│   ├── run.ts        # run 命令
│   ├── auth.ts       # auth 命令
│   ├── models.ts     # models 命令
│   ├── mcp.ts        # mcp 命令
│   ├── session.ts    # session 命令
│   ├── upgrade.ts    # upgrade 命令
│   └── tui/          # TUI 相关
├── bootstrap.ts      # 命令前置处理
├── ui.ts             # UI 组件
├── error.ts          # 错误处理
├── network.ts        # 网络配置
├── upgrade.ts        # 升级逻辑
└── README.md         # 本文档
```

---

## 关键文件

### bootstrap.ts —— 命令前置处理

每个命令执行前都会：

1. 创建 Instance（工作目录上下文）
2. 加载配置
3. 初始化日志
4. 确保结束时清理

```typescript
export async function bootstrap(options: { cwd: string; command: string }) {
  // 创建实例
  const instance = await Instance.provide(options.cwd)

  try {
    // 执行命令...
  } finally {
    // 清理
    await Instance.dispose()
  }
}
```

### ui.ts —— UI 组件

提供各种终端 UI 元素：

```typescript
export namespace UI {
  // Logo
  export function logo(): string

  // 彩色输出
  export function success(msg: string)
  export function error(msg: string)
  export function warning(msg: string)

  // 进度指示
  export function spinner(msg: string): Spinner

  // Markdown 渲染
  export function markdown(content: string): string
}
```

### error.ts —— 错误处理

把异常转换为友好的错误信息：

```typescript
export function FormatError(error: unknown): string | undefined {
  if (error instanceof Provider.ModelNotFoundError) {
    return `找不到模型 ${error.modelID}\n建议：${error.suggestions.join(", ")}`
  }
  // ...
}
```

---

## 使用示例

### 基本对话

```bash
# 交互模式
opencode run

# 直接提问
opencode run "帮我写一个排序函数"

# 指定模型
opencode run --model anthropic/claude-haiku-4-5 "hello"
```

### 调试

```bash
# 打印详细日志
opencode run --print-logs

# 设置日志级别
opencode run --log-level DEBUG
```

### 管道使用

```bash
# 从标准输入读取
cat file.txt | opencode run "总结这个文件"

# 输出到文件
opencode run "写一个 README" > README.md
```

---

## TTY 检测

CLI 会自动检测是否在交互终端中运行：

- **交互终端**：显示彩色输出、进度条
- **管道/重定向**：纯文本输出

```typescript
if (process.stdout.isTTY) {
  // 彩色输出
  UI.success("完成！")
} else {
  // 纯文本
  console.log("完成")
}
```

---

## 添加新命令

1. 在 `cli/cmd/` 创建命令文件
2. 定义 yargs 命令
3. 在 `index.ts` 注册

```typescript
// cli/cmd/mycommand.ts
export const MyCommand: CommandModule = {
  command: "mycommand",
  describe: "我的命令",
  builder: (yargs) => {
    return yargs.option("name", {
      type: "string",
      description: "名称",
    })
  },
  handler: async (args) => {
    // 实现逻辑
  },
}

// index.ts
import { MyCommand } from "./cli/cmd/mycommand"
cli.command(MyCommand)
```
