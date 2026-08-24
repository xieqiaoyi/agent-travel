# Provider 模块详解

## 什么是 Provider？

**Provider（提供者）** 是 AI 大脑的来源。不同公司提供不同的 AI 模型：

| Provider       | 模型示例                 | 特点       |
| -------------- | ------------------------ | ---------- |
| Anthropic      | Claude 4.5, Claude Haiku | 代码能力强 |
| OpenAI         | GPT-5, GPT-5-mini        | 通用性好   |
| Google         | Gemini 3 Pro             | 多模态     |
| Amazon Bedrock | Claude on AWS            | 企业级     |

### 通俗比喻

Provider 就像不同的"大脑供应商"：

- **Anthropic**：像一个深思熟虑的资深工程师
- **OpenAI**：像一个知识渊博的全能助手
- **Google**：像一个擅长多媒体的创意专家

OpenCode 让你可以自由选择用哪个"大脑"。

---

## 核心概念

### 1. Provider（提供者）

一个 Provider 代表一个 AI 服务商：

```typescript
interface Provider {
  id: string // "anthropic"
  name: string // "Anthropic"
  env: string[] // ["ANTHROPIC_API_KEY"]
  models: Model[] // 该 Provider 的所有模型
}
```

### 2. Model（模型）

一个具体的 AI 模型：

```typescript
interface Model {
  id: string           // "claude-sonnet-4-5"
  name: string         // "Claude Sonnet 4.5"
  providerID: string   // "anthropic"

  capabilities: {
    reasoning: boolean      // 支持思维链
    toolcall: boolean       // 支持工具调用
    attachment: boolean     // 支持附件
    input: { image, pdf, ... }   // 输入类型
    output: { text, ... }        // 输出类型
  }

  cost: {
    input: number        // 输入价格（$/1M tokens）
    output: number       // 输出价格
    cache: { read, write }
  }

  limit: {
    context: number      // 上下文长度
    output: number       // 最大输出
  }
}
```

### 3. API Key 认证

使用 AI 模型需要 API Key：

```bash
# 方式 1：环境变量
export ANTHROPIC_API_KEY="sk-ant-xxx"

# 方式 2：opencode auth
opencode auth login anthropic

# 方式 3：配置文件
# opencode.json
{
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "sk-ant-xxx"
      }
    }
  }
}
```

---

## 文件结构

```
provider/
├── provider.ts    # Provider 核心（加载、模型选择）
├── models.ts      # 模型数据库（从 models.dev 加载）
├── auth.ts        # 认证辅助
├── transform.ts   # 模型变体转换
└── README.md      # 本文档
```

### provider.ts —— 核心实现

```typescript
export namespace Provider {
  // 获取所有可用的 Provider
  export async function list(): Promise<Record<string, Info>>

  // 获取特定模型
  export async function getModel(providerID: string, modelID: string): Promise<Model>

  // 获取可调用的语言模型
  export async function getLanguage(model: Model): Promise<LanguageModel>

  // 获取默认模型
  export async function defaultModel(): Promise<{ providerID; modelID }>

  // 获取小模型（用于摘要等）
  export async function getSmallModel(providerID: string): Promise<Model>

  // 解析模型字符串
  export function parseModel(model: string): { providerID; modelID }
  // "anthropic/claude-sonnet-4-5" → {providerID: "anthropic", modelID: "claude-sonnet-4-5"}
}
```

### models.ts —— 模型数据库

从 models.dev 获取模型信息：

```typescript
export namespace ModelsDev {
  // 获取所有 Provider 和模型数据
  export async function get(): Promise<Record<string, Provider>>
}
```

### transform.ts —— 模型变体

处理模型的不同变体（如 "thinking" 版本）：

```typescript
export namespace ProviderTransform {
  // 获取模型的所有变体
  export function variants(model: Model): Record<string, Variant>
}
```

---

## 支持的 Provider

### 开箱即用

| Provider       | 环境变量                       | 说明           |
| -------------- | ------------------------------ | -------------- |
| anthropic      | `ANTHROPIC_API_KEY`            | Claude 系列    |
| openai         | `OPENAI_API_KEY`               | GPT 系列       |
| google         | `GOOGLE_GENERATIVE_AI_API_KEY` | Gemini         |
| amazon-bedrock | AWS 凭证                       | AWS 上的模型   |
| azure          | `AZURE_OPENAI_API_KEY`         | Azure OpenAI   |
| openrouter     | `OPENROUTER_API_KEY`           | 模型聚合       |
| groq           | `GROQ_API_KEY`                 | 快速推理       |
| deepinfra      | `DEEPINFRA_API_KEY`            | 开源模型       |
| github-copilot | OAuth                          | GitHub Copilot |

### 配置示例

```json
// opencode.json
{
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "sk-ant-xxx"
      }
    },
    "openai": {
      "options": {
        "apiKey": "sk-xxx",
        "baseURL": "https://api.openai.com/v1"
      }
    },
    "openrouter": {
      "options": {
        "apiKey": "sk-or-xxx"
      }
    }
  }
}
```

---

## 模型选择

### 默认模型

OpenCode 按以下优先级选择默认模型：

1. 配置文件中指定的 `model`
2. opencode provider 的 "big-pickle"
3. 按模型能力排序选择最佳

```json
// 指定默认模型
{
  "model": "anthropic/claude-sonnet-4-5"
}
```

### 小模型

用于会话摘要等轻量任务：

```json
{
  "small_model": "anthropic/claude-haiku-4-5"
}
```

### 模型排序优先级

```typescript
const priority = [
  "gpt-5", // OpenAI 最新
  "claude-sonnet-4", // Claude 主力
  "big-pickle", // OpenCode 托管
  "gemini-3-pro", // Google
]
```

---

## 使用 Provider

### 1. 查看可用模型

```bash
opencode models list
```

输出：

```
anthropic
  claude-sonnet-4-5-20250514    Claude Sonnet 4.5
  claude-haiku-4-5-20250514     Claude Haiku 4.5

openai
  gpt-5                         GPT-5
  gpt-5-mini                    GPT-5 Mini
```

### 2. 切换模型

```bash
# 使用命令行参数
opencode run --model anthropic/claude-sonnet-4-5 "hello"

# 在对话中切换
> /model anthropic/claude-haiku-4-5
```

### 3. 添加 API Key

```bash
opencode auth login anthropic
# 系统会提示输入 API Key
```

---

## 特殊 Provider 处理

### Amazon Bedrock

需要 AWS 凭证：

```json
{
  "provider": {
    "amazon-bedrock": {
      "options": {
        "region": "us-east-1",
        "profile": "my-aws-profile"
      }
    }
  }
}
```

或使用环境变量：

```bash
export AWS_REGION=us-east-1
export AWS_ACCESS_KEY_ID=xxx
export AWS_SECRET_ACCESS_KEY=xxx
```

### Azure OpenAI

```json
{
  "provider": {
    "azure": {
      "options": {
        "apiKey": "xxx",
        "resourceName": "my-resource",
        "deploymentName": "my-deployment"
      }
    }
  }
}
```

### Google Vertex AI

```json
{
  "provider": {
    "google-vertex": {
      "options": {
        "project": "my-gcp-project",
        "location": "us-central1"
      }
    }
  }
}
```

需要设置：

```bash
export GOOGLE_CLOUD_PROJECT=my-project
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json
```

### GitHub Copilot

通过 OAuth 认证：

```bash
opencode auth login github-copilot
```

---

## 自定义 Provider

### 添加 OpenAI 兼容的服务

```json
{
  "provider": {
    "my-llm": {
      "name": "My LLM Service",
      "api": "https://my-llm.com/v1",
      "env": ["MY_LLM_API_KEY"],
      "npm": "@ai-sdk/openai-compatible",
      "models": {
        "my-model": {
          "name": "My Model",
          "limit": {
            "context": 128000,
            "output": 4096
          },
          "cost": {
            "input": 1.0,
            "output": 2.0
          }
        }
      }
    }
  }
}
```

### 禁用/启用 Provider

```json
{
  "disabled_providers": ["openrouter", "groq"],
  // 或者只启用特定的
  "enabled_providers": ["anthropic", "openai"]
}
```

---

## 模型能力

### 查看模型能力

```typescript
const model = await Provider.getModel("anthropic", "claude-sonnet-4-5")

model.capabilities = {
  reasoning: true, // 支持思维链
  toolcall: true, // 支持工具调用
  attachment: true, // 支持附件
  temperature: true, // 支持温度调节
  input: {
    text: true,
    image: true,
    pdf: true,
    audio: false,
    video: false,
  },
  output: {
    text: true,
    image: false,
    audio: false,
    video: false,
  },
  interleaved: true, // 支持交错思考
}
```

### 基于能力选择模型

```typescript
// 需要处理 PDF？
const pdfModels = models.filter((m) => m.capabilities.input.pdf)

// 需要思维链？
const reasoningModels = models.filter((m) => m.capabilities.reasoning)
```

---

## 常见问题

### Q: 为什么找不到某个模型？

可能原因：

1. 没有设置 API Key
2. Provider 被禁用
3. 模型名称错误

```bash
# 检查可用模型
opencode models list

# 检查 API Key
opencode auth list
```

### Q: 如何使用私有部署的模型？

配置自定义 baseURL：

```json
{
  "provider": {
    "openai": {
      "options": {
        "baseURL": "https://my-private-openai.com/v1"
      }
    }
  }
}
```

### Q: API 调用超时怎么办？

增加超时配置：

```json
{
  "provider": {
    "anthropic": {
      "options": {
        "timeout": 120000
      }
    }
  }
}
```

### Q: 如何查看 API 费用？

每次对话后会显示 token 用量和费用：

```
✓ 完成 (1,234 tokens, $0.0037)
```

也可以查看会话统计：

```bash
opencode stats
```

---

## 开发者指南

### Provider 加载顺序

```
1. 从 models.dev 加载模型数据库
        ↓
2. 从配置文件合并 provider 设置
        ↓
3. 检查环境变量中的 API Key
        ↓
4. 检查 opencode auth 存储的凭证
        ↓
5. 执行自定义 loader（如 Bedrock 凭证链）
        ↓
6. 过滤禁用的 provider/model
        ↓
7. 返回可用的 provider 列表
```

### 添加新 Provider

1. 在 `BUNDLED_PROVIDERS` 添加 SDK
2. 如果需要特殊处理，在 `CUSTOM_LOADERS` 添加 loader
3. 在 models.dev 添加模型数据（或本地配置）

```typescript
// provider.ts
const BUNDLED_PROVIDERS = {
  "@ai-sdk/my-provider": createMyProvider,
}

const CUSTOM_LOADERS = {
  async "my-provider"() {
    return {
      autoload: true,
      options: { ... },
      async getModel(sdk, modelID) {
        return sdk.languageModel(modelID)
      }
    }
  }
}
```

### 调试 Provider

```bash
# 启动时打印日志
opencode run --print-logs

# 搜索 provider 相关日志
grep "provider" ~/.local/state/opencode/logs/*.log
```
