# Question 模块详解

## 什么是 Question？

**Question（提问）** 模块允许 AI 在执行任务过程中主动向用户提问。当 AI 遇到模糊或不完整的指令时，可以暂停执行并请求澄清。

### 通俗比喻

想象 AI 是你的助手，在工作时遇到不确定的地方会问你：

- "你想用哪种数据库？"
- "这个功能应该放在哪个模块？"
- "你更偏好函数式还是面向对象风格？"

---

## Question vs Permission 的区别

| 维度         | Question                  | Permission         |
| ------------ | ------------------------- | ------------------ |
| **目的**     | 澄清需求、收集用户偏好    | 控制工具执行权限   |
| **触发时机** | AI 主动调用 question 工具 | 工具执行前自动检查 |
| **用户交互** | 回答问题（选择/输入）     | 批准/拒绝/永久批准 |
| **典型场景** | "用哪个框架？"            | "允许写入文件吗？" |

---

## 工作流程

```
AI 需要澄清信息
       ↓
调用 question 工具
       ↓
Question.ask() 创建请求，返回 Promise（等待中）
       ↓
Bus.publish(Event.Asked) 发布事件
       ↓
UI 收到事件，显示问题给用户
       ↓
用户选择答案或输入自定义回答
       ↓
UI 调用 POST /question/:id/reply
       ↓
Question.reply() 解析 Promise
       ↓
AI 收到答案，继续执行
```

---

## 问题格式

### 数据结构

```typescript
// 问题定义
const Question = {
  question: "Which database should we use?", // 完整问题
  header: "Database Choice", // 简短标题（最多30字符）
  options: [
    // 选项列表
    { label: "PostgreSQL (Recommended)", description: "Best for complex queries" },
    { label: "SQLite", description: "Good for simple use cases" },
    { label: "MongoDB", description: "For document-based data" },
  ],
  multiple: false, // 是否允许多选
  custom: true, // 是否允许自定义输入（默认 true）
}

// 用户回答
const Answer = ["PostgreSQL (Recommended)"] // 选中的 label 数组
```

### 使用规则

1. **推荐选项放第一位**：在 label 末尾加 `(Recommended)`
2. **不要加 "Other" 选项**：当 `custom: true` 时，UI 会自动添加自定义输入
3. **多选**：设置 `multiple: true` 允许选择多个

---

## 文件结构

```
question/
├── index.ts      # Question 核心逻辑
└── README.md     # 本文档
```

### 核心 API

```typescript
export namespace Question {
  // 发起提问（AI 调用）
  export async function ask(input: {
    sessionID: string
    questions: Info[]
    tool?: { messageID: string; callID: string }
  }): Promise<Answer[]>

  // 用户回复
  export async function reply(input: { requestID: string; answers: Answer[] }): Promise<void>

  // 用户拒绝回答
  export async function reject(requestID: string): Promise<void>
  // 抛出 RejectedError

  // 获取待处理的问题
  export function pending(): Request[]
}
```

---

## 实际示例

### 示例 1：单选问题

```typescript
const answers = await Question.ask({
  sessionID: ctx.sessionID,
  questions: [
    {
      question: "Which testing framework do you prefer?",
      header: "Test Framework",
      options: [
        { label: "Jest (Recommended)", description: "Most popular, good for React" },
        { label: "Vitest", description: "Fast, Vite-native" },
        { label: "Mocha", description: "Flexible, classic choice" },
      ],
      multiple: false,
      custom: true,
    },
  ],
})
// answers[0] = ["Jest (Recommended)"] 或 ["My custom answer"]
```

### 示例 2：多选问题

```typescript
const answers = await Question.ask({
  sessionID: ctx.sessionID,
  questions: [
    {
      question: "Which features should I implement?",
      header: "Features",
      options: [
        { label: "Authentication", description: "User login/logout" },
        { label: "Dashboard", description: "Data visualization" },
        { label: "API", description: "REST endpoints" },
      ],
      multiple: true, // 允许多选
    },
  ],
})
// answers[0] = ["Authentication", "API"]
```

### 示例 3：Plan Exit 确认

```typescript
const answers = await Question.ask({
  sessionID: ctx.sessionID,
  questions: [
    {
      question: `Plan at ${plan} is complete. Switch to build agent?`,
      header: "Build Agent",
      custom: false, // 不允许自定义
      options: [
        { label: "Yes", description: "Start implementing" },
        { label: "No", description: "Continue refining plan" },
      ],
    },
  ],
})
```

---

## 权限控制

`question` 本身也是一种权限，可以在 Agent 配置中控制：

```typescript
// build agent - 允许提问
build: {
  permission: {
    question: "allow"
  }
}

// explore agent - 禁止提问（子任务不应打断用户）
explore: {
  permission: {
    question: "deny"
  }
}
```

---

## 事件系统

```typescript
// 发起提问
Bus.publish(Question.Event.Asked, {
  request: { id, sessionID, questions },
})

// 用户回复
Bus.publish(Question.Event.Replied, {
  requestID,
  answers,
})

// 用户拒绝
Bus.publish(Question.Event.Rejected, {
  requestID,
})
```

---

## HTTP API

| 方法 | 端点                   | 操作               |
| ---- | ---------------------- | ------------------ |
| GET  | `/question`            | 获取所有待处理问题 |
| POST | `/question/:id/reply`  | 回复问题           |
| POST | `/question/:id/reject` | 拒绝问题           |

---

## 常见问题

### Q: 和 Permission 什么时候用哪个？

- **Question**：需要用户提供信息来继续工作
- **Permission**：需要用户授权执行某个操作

### Q: 用户拒绝回答会怎样？

AI 会收到 `RejectedError`，提示 "The user dismissed this question"。
AI 可以选择用默认值继续，或者停止任务。

### Q: 为什么子 Agent 不能提问？

避免子任务频繁打断用户。主 Agent (build/plan) 负责与用户交互，
子 Agent (explore/general) 应该独立完成任务。
