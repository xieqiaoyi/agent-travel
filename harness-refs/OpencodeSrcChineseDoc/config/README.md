# Config 模块详解

## 什么是 Config？

**Config（配置）** 模块负责加载和解析配置文件。OpenCode 支持多种配置格式和位置。

---

## 配置文件位置

按优先级从高到低：

1. **项目配置**：`./opencode.json` 或 `.opencode/config.json`
2. **全局配置**：`~/.config/opencode/config.json`
3. **默认值**：内置默认配置

---

## 配置格式

### JSON 格式

```json
// opencode.json
{
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5",

  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "sk-ant-xxx"
      }
    }
  },

  "agents": {
    "build": {
      "permission": {
        "write": "ask"
      }
    }
  },

  "mcp": {
    "notion": {
      "type": "remote",
      "url": "https://mcp.notion.so"
    }
  }
}
```

### Markdown 格式（带 frontmatter）

```markdown
---
model: anthropic/claude-sonnet-4-5
---

# 项目说明

这是一个 React 项目，请遵循以下规范：

- 使用 TypeScript
- 使用函数组件
```

---

## 配置项说明

### 基本配置

| 配置项        | 说明           | 示例                            |
| ------------- | -------------- | ------------------------------- |
| `model`       | 默认模型       | `"anthropic/claude-sonnet-4-5"` |
| `small_model` | 摘要用的小模型 | `"anthropic/claude-haiku-4-5"`  |

### Provider 配置

```json
{
  "provider": {
    "anthropic": {
      "name": "Anthropic",
      "options": {
        "apiKey": "xxx",
        "baseURL": "https://..."
      }
    }
  },
  "disabled_providers": ["groq"],
  "enabled_providers": ["anthropic", "openai"]
}
```

### Agent 配置

```json
{
  "agents": {
    "build": {
      "description": "写代码",
      "permission": {
        "read": "allow",
        "write": "ask"
      }
    }
  }
}
```

### MCP 配置

```json
{
  "mcp": {
    "my-server": {
      "type": "local",
      "command": ["node", "server.js"]
    }
  }
}
```

---

## 文件结构

```
config/
├── config.ts     # 配置加载和解析
├── markdown.ts   # Markdown frontmatter 解析
└── README.md     # 本文档
```

### config.ts 核心 API

```typescript
export namespace Config {
  // 获取配置（会缓存）
  export async function get(): Promise<Info>

  // 配置类型定义
  export interface Info {
    model?: string
    small_model?: string
    provider?: Record<string, ProviderConfig>
    agents?: Record<string, AgentConfig>
    mcp?: Record<string, McpConfig>
    // ...
  }
}
```

---

## 环境变量

某些配置可以通过环境变量设置：

```bash
# API Keys
export ANTHROPIC_API_KEY=xxx
export OPENAI_API_KEY=xxx

# 其他
export OPENCODE_MODEL=anthropic/claude-sonnet-4-5
```

---

## 配置合并规则

```
默认值 ← 全局配置 ← 项目配置 ← 环境变量

具体路径覆盖通配符
后加载的覆盖先加载的
```

---

## 常见问题

### Q: 配置不生效？

检查顺序：

1. 配置文件位置正确吗？
2. JSON 格式正确吗？
3. 有同名环境变量覆盖吗？

### Q: 如何使用 YAML？

目前只支持 JSON 和 Markdown frontmatter。
可以用工具转换 YAML 为 JSON。

### Q: 敏感信息怎么处理？

API Key 等敏感信息建议：

1. 使用环境变量
2. 使用 `opencode auth login`
3. 不要提交 opencode.json 到 git
