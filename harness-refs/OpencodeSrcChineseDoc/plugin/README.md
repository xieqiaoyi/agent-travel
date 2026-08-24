# Plugin 模块详解

## 什么是 Plugin？

**Plugin（插件）** 是扩展 OpenCode 功能的方式。通过插件，你可以：

- 添加新的认证方式（如 Copilot、Codex）
- 注入自定义的系统提示词
- 监听和处理事件
- 添加新的工具

### 通俗比喻

```
OpenCode 核心
    │
    ├── 内置功能
    │   ├── 基本工具
    │   ├── Claude/OpenAI 认证
    │   └── ...
    │
    └── 插件
        ├── Copilot 认证插件
        ├── Anthropic 认证插件
        └── 你的自定义插件...
```

就像手机的 App Store，插件让你可以按需扩展功能。

---

## 插件类型

### 1. 内置插件（Internal Plugins）

直接集成在代码中：

```typescript
const INTERNAL_PLUGINS: PluginInstance[] = [
  CodexAuthPlugin, // OpenAI Codex 认证
  CopilotAuthPlugin, // GitHub Copilot 认证
]
```

### 2. 默认插件（Builtin Plugins）

从 npm 安装的官方插件：

```typescript
const BUILTIN = [
  "opencode-anthropic-auth@0.0.10", // Anthropic 认证
  "@gitlab/opencode-gitlab-auth@1.3.2", // GitLab 认证
]
```

### 3. 自定义插件

用户配置的插件：

```json
{
  "plugin": ["my-custom-plugin@1.0.0", "file:///path/to/local/plugin"]
}
```

---

## 核心概念

### 1. Plugin 接口

```typescript
type Plugin = (input: PluginInput) => Promise<Hooks>

interface PluginInput {
  client: OpencodeClient // API 客户端
  project: Project.Info // 当前项目信息
  worktree: string // 工作区路径
  directory: string // 当前目录
  serverUrl: string // 服务器 URL
  $: typeof Bun.$ // Bun shell
}
```

### 2. Hooks 接口

插件可以实现的回调：

```typescript
interface Hooks {
  // 认证相关
  auth?: AuthHook

  // 配置相关
  config?: (config: Config) => Promise<void>

  // 事件监听
  event?: (input: { event: BusEvent }) => void

  // 工具相关
  tool?: ToolHook

  // 聊天相关
  "chat.params"?: ChatParamsHook
  "chat.headers"?: ChatHeadersHook

  // 实验性功能
  "experimental.chat.system.transform"?: SystemTransformHook
  "experimental.session.compacting"?: CompactingHook
}
```

---

## 核心函数

### `init()` - 初始化插件系统

```typescript
export async function init() {
  const hooks = await state().then((x) => x.hooks)
  const config = await Config.get()

  // 让每个插件处理配置
  for (const hook of hooks) {
    await hook.config?.(config)
  }

  // 订阅所有事件，转发给插件
  Bus.subscribeAll(async (event) => {
    for (const hook of hooks) {
      hook.event?.({ event })
    }
  })
}
```

### `trigger()` - 触发 Hook

```typescript
export async function trigger<Name extends keyof Hooks>(name: Name, input: Input, output: Output): Promise<Output> {
  for (const hook of await list()) {
    const fn = hook[name]
    if (fn) {
      await fn(input, output) // 插件可以修改 output
    }
  }
  return output
}

// 使用示例
const result = await Plugin.trigger("chat.params", { sessionID, agent, model }, { temperature: 0.7, topP: 1.0 })
// result.temperature 可能被插件修改了
```

### `list()` - 获取所有已加载的插件

```typescript
export async function list() {
  return state().then((x) => x.hooks)
}
```

---

## 插件加载流程

```
OpenCode 启动
      ↓
加载内置插件
      │
      │  CodexAuthPlugin
      │  CopilotAuthPlugin
      ↓
加载默认插件
      │
      │  npm install opencode-anthropic-auth@0.0.10
      │  npm install @gitlab/opencode-gitlab-auth@1.3.2
      ↓
加载用户插件
      │
      │  npm install my-custom-plugin@1.0.0
      │  或直接 import("file:///path/to/plugin")
      ↓
初始化所有插件
      │
      │  for (plugin of plugins) {
      │    const hooks = await plugin(input)
      │    allHooks.push(hooks)
      │  }
      ↓
订阅事件
      │
      │  Bus.subscribeAll() → 转发给插件
```

---

## 编写自定义插件

### 基本结构

```typescript
// my-plugin/index.ts
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (input) => {
  console.log("插件初始化", input.project.name)

  return {
    // 配置处理
    async config(config) {
      console.log("收到配置", config)
    },

    // 事件监听
    event({ event }) {
      console.log("收到事件", event.type)
    },

    // 修改聊天参数
    async "chat.params"(input, output) {
      output.temperature = 0.5 // 降低温度
    },

    // 修改系统提示词
    async "experimental.chat.system.transform"(input, output) {
      output.system.push("Remember to be concise.")
    },
  }
}

export default MyPlugin
```

### 认证插件示例

```typescript
export const MyAuthPlugin: Plugin = async (input) => {
  return {
    async auth(req) {
      if (req.providerID !== "my-provider") return

      // 从环境变量获取 API key
      const apiKey = process.env.MY_API_KEY
      if (!apiKey) {
        return { error: "MY_API_KEY not set" }
      }

      return {
        headers: {
          Authorization: `Bearer ${apiKey}`,
        },
      }
    },
  }
}
```

---

## 常用 Hooks

### `chat.params` - 修改聊天参数

```typescript
"chat.params": async (input, output) => {
  // input: { sessionID, agent, model, provider, message }
  // output: { temperature, topP, topK, options }

  if (input.agent.name === "code-review") {
    output.temperature = 0  // 代码审查用确定性回答
  }
}
```

### `chat.headers` - 添加请求头

```typescript
"chat.headers": async (input, output) => {
  // input: { sessionID, agent, model, provider, message }
  // output: { headers: {} }

  output.headers["X-Custom-Header"] = "value"
}
```

### `experimental.chat.system.transform` - 修改系统提示词

```typescript
"experimental.chat.system.transform": async (input, output) => {
  // input: { sessionID, model }
  // output: { system: string[] }

  output.system.push(`
    Additional context:
    - Current time: ${new Date().toISOString()}
    - Project: ${input.model.providerID}
  `)
}
```

### `experimental.session.compacting` - 自定义压缩提示词

```typescript
"experimental.session.compacting": async (input, output) => {
  // input: { sessionID }
  // output: { context: string[], prompt?: string }

  output.context.push("Remember to preserve code examples.")
  // 或者完全替换提示词
  // output.prompt = "Custom compaction prompt..."
}
```

---

## 配置选项

### 添加插件

```json
{
  "plugin": ["my-npm-plugin@1.0.0", "file:///path/to/local-plugin"]
}
```

### 禁用默认插件

```bash
OPENCODE_DISABLE_DEFAULT_PLUGINS=1 opencode
```

---

## 与其他模块的关系

```
Config
   │
   │  plugin: ["plugin1", "plugin2"]
   ↓
Plugin
   │
   ├── 加载插件
   ├── 管理 hooks
   └── 触发回调
   ↓
   ├── Provider (auth hook)
   ├── LLM (chat.params, chat.headers)
   ├── Session (system.transform, compacting)
   └── Bus (event hook)
```

---

## 常见问题

### Q: 插件加载失败怎么办？

1. 检查插件是否已发布到 npm
2. 检查版本号是否正确
3. 查看日志中的错误信息
4. 本地插件使用 `file://` 前缀

### Q: 如何调试插件？

```typescript
export const MyPlugin: Plugin = async (input) => {
  console.log("Plugin loaded!", input) // 会输出到日志

  return {
    event({ event }) {
      console.log("Event:", event.type, event.properties)
    },
  }
}
```

### Q: 插件执行顺序是什么？

1. 内置插件（按定义顺序）
2. 默认插件（按 BUILTIN 数组顺序）
3. 用户插件（按配置顺序）

多个插件实现同一个 hook 时，会按顺序依次调用。

### Q: 如何发布插件到 npm？

```bash
# 1. 创建 package.json
npm init

# 2. 添加依赖
npm install @opencode-ai/plugin

# 3. 编写代码
# src/index.ts

# 4. 发布
npm publish
```

---

## 内置插件详解

### CodexAuthPlugin

处理 OpenAI Codex（CLI）的 OAuth 认证：

```typescript
// 当 provider 是 openai 且使用 oauth 时
if (provider.id === "openai" && auth?.type === "oauth") {
  // 处理 token 刷新
  // 添加认证头
}
```

### CopilotAuthPlugin

处理 GitHub Copilot 的认证：

```typescript
// 使用 GitHub OAuth 获取 Copilot token
// 支持 token 缓存和刷新
```

---

## 调试技巧

1. **查看已加载的插件**

   ```typescript
   const plugins = await Plugin.list()
   console.log(plugins.length, "plugins loaded")
   ```

2. **检查插件日志**

   ```bash
   tail -f ~/.local/state/opencode/logs/*.log | grep plugin
   ```

3. **测试 hook 触发**
   ```typescript
   const result = await Plugin.trigger("chat.params", input, defaultOutput)
   console.log("After plugins:", result)
   ```
