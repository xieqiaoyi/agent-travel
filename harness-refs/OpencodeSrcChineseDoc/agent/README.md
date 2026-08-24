# Agent 模块详解

## 什么是 Agent？

**Agent（代理）** 定义了 AI 的"人设"和行为模式。不同的 Agent 有不同的能力和权限。

### 通俗比喻

想象 AI 是同一个人，但可以切换不同的"工作模式"：

| Agent       | 比喻         | 能做什么           |
| ----------- | ------------ | ------------------ |
| **build**   | 动手干活模式 | 读写文件、运行命令 |
| **plan**    | 规划讨论模式 | 只规划不执行       |
| **explore** | 快速浏览模式 | 只搜索和阅读       |

---

## 核心概念

### 1. Agent 配置

```typescript
interface Agent {
  name: string // "build"
  description: string // "用于编写代码"
  model?: string // 默认模型
  mode: "primary" | "subagent" // 主模式或子任务模式
  hidden: boolean // 是否隐藏

  permission: {
    // 权限规则
    read: "allow" | "deny" | "ask"
    write: "allow" | "deny" | "ask"
    bash: "allow" | "deny" | "ask"
  }

  tools: string[] // 可用工具列表
  systemPrompt?: string // 系统提示词
}
```

### 2. 权限级别

| 级别    | 含义     | 适用场景             |
| ------- | -------- | -------------------- |
| `allow` | 直接允许 | 安全操作（如读文件） |
| `deny`  | 直接拒绝 | 危险操作             |
| `ask`   | 每次询问 | 敏感操作（如写文件） |

### 3. Agent 模式

- **primary**：可以作为主 Agent 使用
- **subagent**：只能作为子任务调用

---

## 内置 Agent

### build（默认）

用于编写代码，权限较宽松：

```json
{
  "name": "build",
  "description": "写代码、修改文件、运行命令",
  "mode": "primary",
  "permission": {
    "read": "allow",
    "write": "ask",
    "bash": "ask"
  },
  "tools": ["read", "write", "edit", "bash", "glob", "grep", ...]
}
```

### plan

用于规划，不执行实际操作：

```json
{
  "name": "plan",
  "description": "规划任务，不执行修改",
  "mode": "primary",
  "permission": {
    "read": "allow",
    "write": "deny",
    "bash": "deny"
  },
  "tools": ["read", "glob", "grep"] // 只有读取工具
}
```

### explore

用于快速浏览代码库：

```json
{
  "name": "explore",
  "description": "快速探索代码",
  "mode": "subagent",
  "permission": {
    "read": "allow",
    "write": "deny",
    "bash": "deny"
  }
}
```

---

## 使用 Agent

### 指定 Agent

```bash
# 命令行参数
opencode run --agent plan "分析这个项目的结构"

# 在对话中切换
> /agent explore
```

### 创建自定义 Agent

在 `opencode.json` 中添加：

```json
{
  "agents": {
    "reviewer": {
      "description": "代码审查专家",
      "permission": {
        "read": "allow",
        "write": "deny",
        "bash": "deny"
      },
      "systemPrompt": "你是一个严格的代码审查专家..."
    }
  }
}
```

---

## 文件结构

```
agent/
├── agent.ts       # Agent 管理（加载、合并、选择）
├── generate.txt   # 生成新 Agent 的提示模板
└── README.md      # 本文档
```

### agent.ts 核心 API

```typescript
export namespace Agent {
  // 获取所有 Agent
  export async function list(): Promise<Info[]>

  // 获取指定 Agent
  export async function get(name: string): Promise<Info>

  // 获取默认 Agent
  export async function defaultAgent(): Promise<string>
}
```

---

## 权限配置详解

### 基本语法

```json
{
  "permission": {
    "工具名": "allow" | "deny" | "ask"
  }
}
```

### 通配符匹配

```json
{
  "permission": {
    "*": "ask", // 默认所有工具都询问
    "read": "allow", // 读取允许
    "glob": "allow" // 搜索允许
  }
}
```

### 路径级别权限

```json
{
  "permission": {
    "write": {
      "allow": ["src/**"], // 允许写 src 目录
      "deny": ["node_modules/**"], // 禁止写依赖
      "ask": ["**"] // 其他询问
    }
  }
}
```

---

## 常见问题

### Q: Agent 和 Provider 有什么区别？

- **Provider**：决定用哪个 AI 模型（Claude、GPT）
- **Agent**：决定 AI 的行为模式（能做什么、不能做什么）

同一个 Provider 可以用不同的 Agent。

### Q: 如何限制 AI 只读不写？

使用 `plan` Agent 或创建自定义 Agent：

```json
{
  "agents": {
    "readonly": {
      "permission": {
        "*": "deny",
        "read": "allow",
        "glob": "allow",
        "grep": "allow"
      }
    }
  }
}
```

### Q: subagent 是什么？

subagent 是只能被其他 Agent 调用的"助手"。
比如 `explore` 是一个 subagent，可以被 `build` 调用来快速搜索代码。

```
用户 → build Agent → 调用 explore subagent → 返回结果 → build 继续
```
