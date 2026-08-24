# LSP 模块详解

## 什么是 LSP？

**LSP（Language Server Protocol）** 是一个标准协议，让编辑器和"语言服务器"能够交流。语言服务器是一个了解特定编程语言的程序，能提供：

- 代码补全
- 错误诊断
- 跳转定义
- 查找引用
- 符号搜索

### 通俗比喻

```
                    LSP 协议
VSCode / OpenCode ←─────────→ TypeScript 服务器
      ↑                              ↓
 "这行代码有什么问题？"      "第5行：'foo' 未定义"
      ↓                              ↑
 显示红色波浪线 ⚠️          分析代码，找出错误
```

就像请了一个"语言专家"，你问它问题，它告诉你答案。

---

## OpenCode 中的 LSP

OpenCode 内置了对多种语言的 LSP 支持，AI 可以通过 LSP 工具获取代码智能信息。

### 支持的语言服务器

| ID                           | 语言                  | 服务器                     |
| ---------------------------- | --------------------- | -------------------------- |
| `typescript-language-server` | TypeScript/JavaScript | typescript-language-server |
| `pyright`                    | Python                | pyright-langserver         |
| `ty`                         | Python (实验性)       | ty                         |
| `gopls`                      | Go                    | gopls                      |
| `rust-analyzer`              | Rust                  | rust-analyzer              |

---

## 核心概念

### 1. LSPServer - 服务器定义

描述如何启动一个语言服务器：

```typescript
interface LSPServer.Info {
  id: string              // 服务器标识
  extensions: string[]    // 支持的文件扩展名 [".ts", ".tsx", ".js"]
  root: (file: string) => Promise<string>  // 获取项目根目录
  spawn: (root: string) => Promise<{
    process: ChildProcess
    initialization?: object  // 初始化参数
  }>
}
```

### 2. LSPClient - 客户端连接

封装与语言服务器的通信：

```typescript
interface LSPClient.Info {
  serverID: string        // 对应的服务器 ID
  root: string            // 项目根目录
  connection: Connection  // JSON-RPC 连接
  diagnostics: Map<string, Diagnostic[]>  // 诊断信息缓存
  notify: {
    open: (path) => void   // 打开文件
    change: (path) => void // 文件改变
    close: (path) => void  // 关闭文件
  }
}
```

### 3. 诊断信息

```typescript
interface Diagnostic {
  range: {
    start: { line: number; character: number }
    end: { line: number; character: number }
  }
  severity: 1 | 2 | 3 | 4 // ERROR, WARN, INFO, HINT
  message: string
  source?: string // "typescript", "eslint", etc.
}
```

---

## 核心函数

### `init()` - 初始化 LSP 系统

```typescript
export async function init() {
  const state = {
    clients: [], // 已连接的客户端
    servers: {}, // 可用的服务器
    broken: new Set(), // 启动失败的服务器
    spawning: new Map(), // 正在启动的服务器
  }

  // 加载内置服务器
  for (const server of Object.values(LSPServer)) {
    state.servers[server.id] = server
  }

  // 加载配置中的自定义服务器
  for (const [name, item] of Object.entries(config.lsp ?? {})) {
    if (item.disabled) {
      delete state.servers[name]
      continue
    }
    state.servers[name] = {
      id: name,
      extensions: item.extensions,
      spawn: async (root) => ({
        process: spawn(item.command[0], item.command.slice(1), { cwd: root }),
        initialization: item.initialization,
      }),
    }
  }

  return state
}
```

### `getClients()` - 获取文件对应的客户端

```typescript
async function getClients(file: string) {
  const extension = path.parse(file).ext // ".ts"
  const result: LSPClient.Info[] = []

  for (const server of Object.values(servers)) {
    // 检查扩展名是否匹配
    if (!server.extensions.includes(extension)) continue

    // 获取项目根目录
    const root = await server.root(file)
    if (!root) continue

    // 查找或创建客户端
    let client = clients.find((x) => x.root === root && x.serverID === server.id)

    if (!client) {
      // 启动新的语言服务器
      const handle = await server.spawn(root)
      client = await LSPClient.create({
        serverID: server.id,
        server: handle,
        root,
      })
      clients.push(client)
    }

    result.push(client)
  }

  return result
}
```

### `diagnostics()` - 获取所有诊断信息

```typescript
export async function diagnostics() {
  const results: Record<string, Diagnostic[]> = {}

  for (const client of clients) {
    for (const [path, diags] of client.diagnostics.entries()) {
      results[path] = [...(results[path] || []), ...diags]
    }
  }

  return results
}

// 返回示例
{
  "/project/src/main.ts": [
    { range: {...}, severity: 1, message: "'foo' is not defined" },
    { range: {...}, severity: 2, message: "Unused variable 'x'" },
  ],
  "/project/src/utils.ts": [
    { range: {...}, severity: 1, message: "Type error" },
  ]
}
```

### `hover()` - 获取悬停信息

```typescript
export async function hover(input: { file: string; line: number; character: number }) {
  return run(input.file, (client) =>
    client.connection.sendRequest("textDocument/hover", {
      textDocument: { uri: pathToFileURL(input.file).href },
      position: { line: input.line, character: input.character },
    })
  )
}

// 返回示例
{
  contents: "function foo(): void",
  range: { start: { line: 5, character: 0 }, end: { line: 5, character: 3 } }
}
```

### `definition()` - 跳转到定义

```typescript
export async function definition(input: { file: string; line: number; character: number }) {
  return run(input.file, (client) =>
    client.connection.sendRequest("textDocument/definition", {
      textDocument: { uri: pathToFileURL(input.file).href },
      position: { line: input.line, character: input.character },
    }),
  )
}

// 返回示例
;[
  {
    uri: "file:///project/src/utils.ts",
    range: { start: { line: 10, character: 0 }, end: { line: 10, character: 20 } },
  },
]
```

### `references()` - 查找所有引用

```typescript
export async function references(input: { file: string; line: number; character: number }) {
  return run(input.file, (client) =>
    client.connection.sendRequest("textDocument/references", {
      textDocument: { uri: pathToFileURL(input.file).href },
      position: { line: input.line, character: input.character },
      context: { includeDeclaration: true },
    }),
  )
}
```

### `workspaceSymbol()` - 搜索工作区符号

```typescript
export async function workspaceSymbol(query: string) {
  return runAll((client) =>
    client.connection.sendRequest("workspace/symbol", { query })
      .then(result => result.filter(x => kinds.includes(x.kind)))  // 只返回类/函数等
      .then(result => result.slice(0, 10))  // 最多 10 个
  )
}

// 返回示例
[
  { name: "MyClass", kind: 5, location: { uri: "...", range: {...} } },
  { name: "myFunction", kind: 12, location: { uri: "...", range: {...} } },
]
```

---

## 符号类型

```typescript
enum SymbolKind {
  File = 1,
  Module = 2,
  Namespace = 3,
  Package = 4,
  Class = 5, // OpenCode 返回
  Method = 6, // OpenCode 返回
  Property = 7,
  Field = 8,
  Constructor = 9,
  Enum = 10, // OpenCode 返回
  Interface = 11, // OpenCode 返回
  Function = 12, // OpenCode 返回
  Variable = 13, // OpenCode 返回
  Constant = 14, // OpenCode 返回
  // ... 更多
}
```

---

## 配置选项

### 禁用所有 LSP

```json
{
  "lsp": false
}
```

### 禁用特定服务器

```json
{
  "lsp": {
    "pyright": {
      "disabled": true
    }
  }
}
```

### 自定义语言服务器

```json
{
  "lsp": {
    "my-lsp": {
      "command": ["my-language-server", "--stdio"],
      "extensions": [".myext"],
      "env": {
        "MY_VAR": "value"
      },
      "initialization": {
        "settings": {
          "someOption": true
        }
      }
    }
  }
}
```

---

## 工作流程

### 语言服务器启动流程

```
用户打开 .ts 文件
      ↓
getClients("main.ts")
      │
      │  检查扩展名 → ".ts"
      │  匹配服务器 → typescript-language-server
      ↓
检查是否已有客户端
      │
      │  没有 → 启动新服务器
      ↓
spawn()
      │
      │  启动进程：typescript-language-server --stdio
      ↓
LSPClient.create()
      │
      │  建立 JSON-RPC 连接
      │  发送 initialize 请求
      │  发送 initialized 通知
      ↓
返回客户端
      │
      │  现在可以发送请求了！
```

### 诊断信息流程

```
文件保存
      ↓
touchFile("main.ts")
      │
      │  通知语言服务器：文件已更新
      ↓
语言服务器分析
      │
      │  检查语法、类型错误
      ↓
发送 publishDiagnostics
      │
      │  { uri: "...", diagnostics: [...] }
      ↓
客户端缓存诊断信息
      │
      │  client.diagnostics.set(path, diags)
      ↓
AI 可以查询
      │
      │  LSP.diagnostics() → 返回所有诊断
```

---

## 与其他模块的关系

```
Tool (lsp 工具)
   │
   │  调用 LSP 函数获取代码信息
   ↓
LSP
   │
   ├── LSPServer: 定义如何启动服务器
   │
   ├── LSPClient: 管理与服务器的连接
   │
   └── 各种函数: diagnostics, hover, definition...
   ↓
AI
   │
   │  使用 lsp 工具获取代码智能
   │  例如：获取某行代码的定义位置
```

---

## 常见问题

### Q: 为什么 LSP 没有启动？

1. 检查是否安装了语言服务器
2. 检查配置中是否禁用了 LSP
3. 查看日志中的错误信息

### Q: 如何添加新的语言支持？

1. 安装语言服务器（如 `npm install -g @elm-tooling/elm-language-server`）
2. 在配置中添加：
   ```json
   {
     "lsp": {
       "elm": {
         "command": ["elm-language-server", "--stdio"],
         "extensions": [".elm"]
       }
     }
   }
   ```

### Q: 诊断信息不更新？

尝试：

1. 保存文件
2. 重启 OpenCode
3. 检查语言服务器进程是否在运行

---

## 调试技巧

1. **查看 LSP 状态**

   ```bash
   curl http://localhost:4096/lsp
   ```

2. **检查日志**

   ```bash
   tail -f ~/.local/state/opencode/logs/*.log | grep lsp
   ```

3. **手动测试语言服务器**

   ```bash
   # 确保服务器能正常启动
   typescript-language-server --stdio
   ```

4. **实验性 Python LSP**
   ```bash
   # 使用 ty 代替 pyright
   OPENCODE_EXPERIMENTAL_LSP_TY=1 opencode
   ```
