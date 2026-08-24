# Permission 模块详解

## 什么是 Permission？

**Permission（权限）** 系统控制 AI 能做什么、不能做什么。就像你手机上的应用权限一样。

### 通俗比喻

想象 AI 是你雇的助手：

- **allow（允许）**：随便做，不用问我
- **ask（询问）**：每次都要问我同意
- **deny（禁止）**：绝对不能做

---

## 权限类型

| 权限    | 含义          | 默认  |
| ------- | ------------- | ----- |
| `read`  | 读取文件      | allow |
| `write` | 创建/覆盖文件 | ask   |
| `edit`  | 修改文件      | ask   |
| `bash`  | 执行命令      | ask   |
| `fetch` | 网络请求      | allow |

---

## 工作流程

```
AI 想要写文件
      ↓
[1] 检查权限规则
      ↓
[2] 规则说 "ask"
      ↓
[3] 弹出确认框
    "AI 想要创建 hello.js，允许吗？"
      ↓
[4] 用户选择：
    - "允许一次" → 执行
    - "始终允许" → 执行 + 记住
    - "拒绝" → 不执行
```

---

## 配置权限

### 在 opencode.json 中

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

### 路径级别控制

```json
{
  "permission": {
    "write": {
      "allow": ["src/**", "test/**"],
      "deny": [".env", "*.secret", "node_modules/**"],
      "ask": ["**"]
    }
  }
}
```

---

## 文件结构

```
permission/
├── next.ts     # 新版权限引擎
├── arity.ts    # 配置校验
├── index.ts    # 导出
└── README.md   # 本文档
```

### next.ts 核心 API

```typescript
export namespace PermissionNext {
  // 检查权限
  export async function check(input: {
    permission: string // "write"
    patterns: string[] // ["/path/to/file"]
    sessionID: string
    ruleset: Rules
  }): Promise<"allow" | "deny">

  // 请求权限（会弹出确认框）
  export async function ask(input: {
    permission: string
    patterns: string[]
    sessionID: string
    metadata: any
    ruleset: Rules
  }): Promise<void> // 拒绝时抛出 RejectedError
}
```

---

## 用户交互

当触发 `ask` 权限时：

```
┌─────────────────────────────────────────┐
│  AI 请求权限                             │
├─────────────────────────────────────────┤
│  操作: write                             │
│  路径: src/hello.js                      │
│                                         │
│  [允许一次]  [始终允许]  [拒绝]           │
└─────────────────────────────────────────┘
```

### 选项说明

| 选项     | 效果                 |
| -------- | -------------------- |
| 允许一次 | 这次允许，下次还会问 |
| 始终允许 | 同类操作不再询问     |
| 拒绝     | 本次不执行           |

---

## 常见问题

### Q: 如何让 AI 自由发挥？

设置所有权限为 allow（不推荐）：

```json
{
  "permission": {
    "*": "allow"
  }
}
```

### Q: 如何完全禁止 AI 执行命令？

```json
{
  "permission": {
    "bash": "deny"
  }
}
```

### Q: 权限配置优先级？

```
1. 具体路径匹配（如 "src/main.js"）
2. 通配符匹配（如 "src/**"）
3. 工具级别默认（如 "write": "ask"）
4. 全局默认（如 "*": "ask"）
```
