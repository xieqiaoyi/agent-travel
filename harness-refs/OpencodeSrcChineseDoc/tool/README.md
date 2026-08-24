# Tool 模块详解

## 什么是 Tool？

**Tool（工具）** 是 AI 的"手"。AI 模型本身只能"思考"和"说话"，
但通过工具，AI 可以真正操作你的电脑：读文件、写代码、运行命令。

### 通俗比喻

想象 AI 是一个坐在你旁边的程序员朋友：

- **没有工具时**：ta 只能"嘴上说说"，比如"你应该这样改代码..."
- **有工具后**：ta 可以直接动手，打开文件、修改代码、运行测试

```
AI 的思考: "我需要先看看这个文件的内容..."
    ↓
调用 read 工具: read("src/main.js")
    ↓
得到文件内容: "function hello() { ... }"
    ↓
AI 继续思考: "我知道怎么改了..."
    ↓
调用 edit 工具: edit("src/main.js", oldString, newString)
    ↓
文件被修改 ✅
```

---

## 内置工具一览

### 文件操作类

| 工具        | 作用     | 使用场景     |
| ----------- | -------- | ------------ |
| `read`      | 读取文件 | 查看代码内容 |
| `write`     | 创建文件 | 新建文件     |
| `edit`      | 修改文件 | 修改已有文件 |
| `multiedit` | 批量修改 | 一次改多处   |

### 搜索类

| 工具         | 作用         | 使用场景             |
| ------------ | ------------ | -------------------- |
| `glob`       | 搜索文件名   | 找 `*.js` 文件       |
| `grep`       | 搜索文件内容 | 找包含 "TODO" 的文件 |
| `codesearch` | 代码搜索     | 使用 Sourcegraph     |
| `ls`         | 列出目录     | 查看文件夹结构       |

### 命令执行类

| 工具   | 作用     | 使用场景                    |
| ------ | -------- | --------------------------- |
| `bash` | 运行命令 | `npm install`、`git status` |

### 网络类

| 工具        | 作用     | 使用场景      |
| ----------- | -------- | ------------- |
| `webfetch`  | 获取网页 | 查阅文档、API |
| `websearch` | 网络搜索 | 搜索解决方案  |

### 辅助类

| 工具       | 作用       | 使用场景       |
| ---------- | ---------- | -------------- |
| `question` | 向用户提问 | 信息不足时询问 |
| `task`     | 创建子任务 | 复杂任务分解   |
| `todo`     | 任务列表   | 跟踪进度       |
| `skill`    | 加载技能   | 使用专业技能   |
| `lsp`      | 语言服务   | 代码补全、诊断 |

---

## 文件结构

```
tool/
├── tool.ts           # 工具基类和定义方式
├── registry.ts       # 工具注册中心
├── truncation.ts     # 输出截断
│
├── read.ts + read.txt       # 读取文件
├── write.ts + write.txt     # 创建文件
├── edit.ts + edit.txt       # 修改文件
├── multiedit.ts + multiedit.txt
│
├── glob.ts + glob.txt       # 文件搜索
├── grep.ts + grep.txt       # 内容搜索
├── ls.ts + ls.txt           # 列出目录
├── codesearch.ts + codesearch.txt
│
├── bash.ts + bash.txt       # 命令执行
│
├── webfetch.ts + webfetch.txt
├── websearch.ts + websearch.txt
│
├── question.ts + question.txt
├── task.ts + task.txt
├── todo.ts + todowrite.txt + todoread.txt
├── skill.ts
├── lsp.ts + lsp.txt
│
├── apply_patch.ts + apply_patch.txt
├── plan.ts + plan-enter.txt + plan-exit.txt
├── batch.ts + batch.txt
├── external-directory.ts
├── invalid.ts
│
└── README.md
```

### 文件命名规则

- `xxx.ts` —— 工具的 TypeScript 实现
- `xxx.txt` —— 工具的描述文件（给 AI 看的说明书）

---

## 核心文件详解

### tool.ts —— 工具定义基类

所有工具都用 `Tool.define()` 定义：

```typescript
export namespace Tool {
  // 工具上下文
  export type Context = {
    sessionID: string // 当前会话
    messageID: string // 当前消息
    agent: string // 当前 Agent
    abort: AbortSignal // 取消信号
    messages: Message[] // 历史消息
    metadata(input) // 设置元数据
    ask(input) // 请求权限
  }

  // 定义工具
  export function define<P, M>(
    id: string,
    init: () => Promise<{
      description: string // 工具描述
      parameters: ZodSchema // 参数验证
      execute(args, ctx) // 执行函数
    }>,
  )
}
```

### registry.ts —— 工具注册中心

管理所有可用的工具：

```typescript
export namespace ToolRegistry {
  // 获取所有工具
  export async function all(): Promise<Record<string, Tool>>

  // 根据 Agent 过滤工具
  export async function forAgent(agent: Agent): Promise<Record<string, Tool>>
}
```

### truncation.ts —— 输出截断

当工具输出太长时自动截断：

```typescript
export namespace Truncate {
  // 截断输出
  export async function output(
    content: string,
    options: {},
    agent?: Agent,
  ): Promise<{
    content: string // 截断后的内容
    truncated: boolean // 是否被截断
    outputPath?: string // 完整内容保存路径
  }>
}
```

---

## 工具详解

### read —— 读取文件

```typescript
// AI 调用
read({
  filePath: "/project/src/main.js",
  offset: 0,      // 从第几行开始（可选）
  limit: 100,     // 读取多少行（可选）
})

// 返回
{
  title: "src/main.js (100 lines)",
  output: "1: function hello() {\n2:   console.log('hi');\n3: }"
}
```

**特点**：

- 返回带行号的内容
- 支持分页读取大文件
- 自动检测二进制文件

### write —— 创建文件

```typescript
// AI 调用
write({
  filePath: "/project/src/new-file.js",
  content: "console.log('hello');",
})

// 需要权限确认
// "AI 想要创建 new-file.js，允许吗？"
```

**注意**：

- 如果文件已存在会警告
- 需要权限确认（可配置）

### edit —— 修改文件

```typescript
// AI 调用
edit({
  filePath: "/project/src/main.js",
  oldString: "console.log('hi')",
  newString: "console.log('hello')",
})

// 返回
{
  title: "Edited src/main.js",
  output: "Successfully replaced content"
}
```

**特点**：

- 精确匹配替换
- 如果 oldString 不存在会报错
- 如果 oldString 出现多次会报错
- 可用 `replaceAll: true` 替换所有

### bash —— 执行命令

```typescript
// AI 调用
bash({
  command: "npm install lodash",
  workdir: "/project",       // 工作目录（可选）
  timeout: 30000,            // 超时（可选）
})

// 返回
{
  title: "npm install lodash",
  output: "added 1 package...",
  metadata: {
    exitCode: 0
  }
}
```

**安全限制**：

- 默认需要权限确认
- 危险命令会警告
- 支持取消执行

### glob —— 搜索文件名

```typescript
// AI 调用
glob({
  pattern: "**/*.ts",
  path: "/project/src",
})

// 返回
{
  output: "src/main.ts\nsrc/utils/helper.ts\n..."
}
```

### grep —— 搜索内容

```typescript
// AI 调用
grep({
  pattern: "TODO", // 支持正则
  include: "*.ts", // 文件过滤
  path: "/project",
})

// 返回
{
  output: "src/main.ts:10: // TODO: implement this\n..."
}
```

---

## 如何创建自定义工具

### 1. 创建工具文件

```typescript
// tool/mytool.ts
import { z } from "zod"
import { Tool } from "./tool"

export const MyTool = Tool.define(
  "mytool", // 工具 ID
  async () => ({
    description: "这是我的自定义工具",

    parameters: z.object({
      input: z.string().describe("输入参数"),
      count: z.number().optional().describe("次数"),
    }),

    async execute(args, ctx) {
      // 1. 检查权限（如果需要）
      await ctx.ask({
        permission: "mytool",
        patterns: [args.input],
        metadata: { input: args.input },
      })

      // 2. 执行逻辑
      const result = doSomething(args.input, args.count ?? 1)

      // 3. 返回结果
      return {
        title: `MyTool: ${args.input}`,
        metadata: { processed: true },
        output: result,
      }
    },
  }),
)
```

### 2. 创建描述文件

```text
// tool/mytool.txt
这是我的自定义工具，用于执行 XXX 操作。

使用场景：
- 当需要 XXX 时
- 当用户要求 XXX 时

参数说明：
- input: 要处理的输入内容
- count: 执行次数（默认 1）

示例：
mytool({ input: "hello", count: 3 })

注意事项：
- 输入不能为空
- count 最大为 10
```

### 3. 注册工具

在 `registry.ts` 中添加：

```typescript
import { MyTool } from "./mytool"

// 在 BUILTIN_TOOLS 中添加
const BUILTIN_TOOLS = {
  // ...现有工具...
  mytool: MyTool,
}
```

---

## 工具执行流程

```
AI 决定调用工具
    ↓
[1] 参数验证
    Tool.parameters.parse(args)
    ↓
[2] 权限检查
    如果是敏感操作，调用 ctx.ask()
    ↓
[3] 等待用户确认（如果需要）
    "AI 想要执行 xxx，允许吗？"
    ↓
[4] 执行工具逻辑
    execute(args, ctx)
    ↓
[5] 输出截断（如果太长）
    Truncate.output(result.output)
    ↓
[6] 返回结果给 AI
    AI 继续思考或结束
```

---

## 权限系统集成

工具可以请求不同类型的权限：

```typescript
// 读取权限
await ctx.ask({
  permission: "read",
  patterns: ["/path/to/file"],
})

// 写入权限
await ctx.ask({
  permission: "write",
  patterns: ["/path/to/file"],
  metadata: { content: "..." },
})

// 执行权限
await ctx.ask({
  permission: "bash",
  patterns: ["npm install"],
  metadata: { command: "..." },
})
```

权限配置在 `opencode.json`：

```json
{
  "agents": {
    "build": {
      "permission": {
        "read": "allow",
        "write": "ask",
        "bash": "ask"
      }
    }
  }
}
```

---

## \*.txt 描述文件的作用

AI 模型需要知道每个工具是干什么的、怎么用。
`.txt` 文件就是给 AI 看的"说明书"：

```text
// read.txt（简化版）
读取文件内容。

参数：
- filePath: 文件的绝对路径
- offset: 从第几行开始（可选，默认 0）
- limit: 读取多少行（可选，默认 2000）

使用建议：
- 优先读取完整文件
- 对于大文件，使用 offset 和 limit 分页
- 文件不存在会返回错误

示例：
read({ filePath: "/project/src/main.js" })
read({ filePath: "/project/big.log", offset: 1000, limit: 100 })
```

这些描述会被拼接到发给 AI 的提示词中。

---

## 常见问题

### Q: 工具和 MCP 工具有什么区别？

**内置工具**：

- 代码在 OpenCode 仓库中
- 直接调用本地功能
- 性能好，可靠

**MCP 工具**：

- 来自外部 MCP 服务器
- 通过网络调用
- 扩展性好，但需要配置

### Q: 为什么有些工具需要权限？

安全考虑。AI 可能会：

- 误删重要文件
- 执行危险命令
- 读取敏感信息

权限系统让你可以控制 AI 的行为。

### Q: 工具输出太长怎么办？

`truncation.ts` 会自动处理：

- 截断过长的输出
- 把完整内容保存到临时文件
- 告诉 AI 可以用 `read` 工具查看完整内容

### Q: 如何调试工具？

```bash
# 启动时打印日志
opencode run --print-logs

# 在工具代码中添加日志
import { Log } from "../util/log"
const log = Log.create({ service: "tool.mytool" })
log.info("executing", { args })
```

---

## 最佳实践

### 1. 参数验证要严格

```typescript
parameters: z.object({
  filePath: z.string().min(1).describe("文件路径，必须是绝对路径"),
  content: z.string().describe("文件内容"),
})
```

### 2. 返回有意义的 title

```typescript
return {
  title: `Created ${path.basename(filePath)}`, // 好
  // title: "Done",  // 不好，信息量太少
}
```

### 3. 提供详细的错误信息

```typescript
if (!fileExists) {
  throw new Error(`文件不存在: ${filePath}\n` + `建议：使用 glob 工具搜索正确的文件路径`)
}
```

### 4. 支持取消操作

```typescript
async execute(args, ctx) {
  // 检查是否被取消
  ctx.abort.throwIfAborted()

  // 长时间操作时定期检查
  for (const item of items) {
    ctx.abort.throwIfAborted()
    await processItem(item)
  }
}
```

### 5. 合理设置权限

```typescript
// 只读操作：通常不需要权限
// 写入操作：需要权限
// 执行命令：需要权限
// 网络请求：视情况而定
```
