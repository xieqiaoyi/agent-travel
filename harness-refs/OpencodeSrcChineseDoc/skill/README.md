# Skill 模块详解

## 什么是 Skill？

**Skill（技能）** 是可复用的提示词和工作流程，让团队把成功经验固化下来供 AI 参考。

### 通俗比喻

技能就像是给 AI 的"操作手册"：

- 不是让 AI 执行什么（那是 Tool 的工作）
- 而是告诉 AI 应该**怎么做**、**注意什么**

```
Tool（工具）= AI 的"手"   → 执行具体操作
Skill（技能）= AI 的"知识" → 指导如何操作
```

---

## Skill vs Tool 的区别

| 维度         | Skill（技能）          | Tool（工具）    |
| ------------ | ---------------------- | --------------- |
| **本质**     | 静态知识/指导文档      | 可执行的功能    |
| **格式**     | Markdown 文件          | TypeScript 代码 |
| **作用**     | 提供最佳实践、工作流程 | 执行具体操作    |
| **执行能力** | 无（只是文本）         | 有（执行代码）  |

### 协作关系

```
用户: "帮我写文件操作代码"
       ↓
AI: 先加载 skill（学习怎么做）
       ↓
AI: 调用 skill 工具 → 获取 "bun-file-io" 技能内容
       ↓
技能内容: "优先使用 Bun.file() 而不是 fs.readFile()..."
       ↓
AI: 根据技能指导，使用 write 工具写代码
```

---

## 技能文件格式

技能文件必须命名为 `SKILL.md`，包含 YAML frontmatter：

````markdown
---
name: bun-file-io
description: Use this when working on file operations in this repo
---

## When to Use

- Editing file I/O code
- Working with filesystem APIs

## Best Practices

- Prefer `Bun.file()` over Node `fs`
- Use `Bun.write()` for writing files
- Handle errors gracefully

## Examples

```typescript
// Good
const content = await Bun.file("path").text()

// Avoid
const content = fs.readFileSync("path", "utf-8")
```
````

````

### 必需字段

| 字段 | 说明 |
|------|------|
| `name` | 技能唯一标识符 |
| `description` | 告诉 AI 何时应该使用此技能 |

---

## 技能存储位置

按加载优先级：

| 位置 | 路径 | 说明 |
|------|------|------|
| Claude 全局 | `~/.claude/skills/**/SKILL.md` | 用户级 |
| Claude 项目 | `.claude/skills/**/SKILL.md` | 项目级 |
| OpenCode 全局 | `~/.config/opencode/skill/**/SKILL.md` | 用户级 |
| OpenCode 项目 | `.opencode/skill/**/SKILL.md` | 项目级 |

---

## AI 如何使用技能

### 1. 工具描述中列出可用技能

当 AI 启动时，`skill` 工具的描述会包含所有可用技能：

```xml
<available_skills>
  <skill>
    <name>bun-file-io</name>
    <description>Use when working on file operations...</description>
  </skill>
  <skill>
    <name>test-driven-development</name>
    <description>Use when implementing features with TDD...</description>
  </skill>
</available_skills>
````

### 2. AI 根据任务选择技能

```
用户: "帮我实现一个新功能"
       ↓
AI 思考: "这需要 TDD，让我加载相关技能"
       ↓
AI 调用: skill({ name: "test-driven-development" })
       ↓
获得技能内容: "1. 先写测试 2. 运行测试（失败）3. 写代码 4. 运行测试（通过）..."
       ↓
AI 按照技能指导执行
```

### 3. 权限检查

加载技能需要 `skill` 权限：

```typescript
await ctx.ask({
  permission: "skill",
  patterns: [params.name], // 技能名称
})
```

---

## 文件结构

```
skill/
├── index.ts      # 技能发现和管理
└── README.md     # 本文档
```

### 核心 API

```typescript
export namespace Skill {
  // 获取所有技能
  export async function all(): Promise<Info[]>

  // 获取单个技能
  export async function get(name: string): Promise<Info | undefined>
}

// 技能信息
interface Info {
  name: string // 技能名称
  description: string // 技能描述
  location: string // 文件路径
}
```

---

## 创建自定义技能

### 1. 创建目录和文件

```bash
mkdir -p .opencode/skill/my-skill
touch .opencode/skill/my-skill/SKILL.md
```

### 2. 编写技能内容

```markdown
---
name: my-coding-standards
description: Use when writing code in this project to follow team conventions
---

## Code Style

- Use 2 spaces for indentation
- Prefer `const` over `let`
- Use single word variable names when possible

## Naming Conventions

- camelCase for variables and functions
- PascalCase for classes and types
- UPPER_SNAKE_CASE for constants

## Error Handling

- Never use try/catch unless absolutely necessary
- Prefer Result types over throwing exceptions

## Examples

...
```

### 3. 验证技能被加载

```bash
opencode debug skill
```

---

## 最佳实践

### 技能内容建议

1. **明确使用场景**：在 description 中清楚说明何时使用
2. **提供具体示例**：给出代码示例而不是抽象描述
3. **列出常见陷阱**：帮助 AI 避免常见错误
4. **保持简洁**：技能内容会占用上下文，不要过长

### 技能组织建议

```
.opencode/skill/
├── coding-standards/SKILL.md    # 编码规范
├── testing/SKILL.md             # 测试指南
├── git-workflow/SKILL.md        # Git 工作流
└── debugging/SKILL.md           # 调试技巧
```

---

## 常见问题

### Q: 技能和系统提示词有什么区别？

- **系统提示词**：始终加载，影响所有对话
- **技能**：按需加载，只在相关任务时使用

### Q: 为什么我的技能没被加载？

检查：

1. 文件名是否是 `SKILL.md`
2. frontmatter 是否包含 `name` 和 `description`
3. 文件位置是否在支持的目录中

### Q: 技能会被自动使用吗？

不会。AI 需要主动调用 `skill` 工具来加载技能。
你可以在系统提示词中提醒 AI 在特定场景使用技能。
