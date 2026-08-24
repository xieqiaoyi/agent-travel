# Storage 模块详解

## 什么是 Storage？

**Storage（存储）** 负责把数据持久化到磁盘。所有的会话记录、配置、权限等都通过它保存。

### 通俗比喻

Storage 就像一个"文件柜管理员"：

- **命名空间**：不同的抽屉（session、share、permission）
- **路径**：抽屉里的文件夹结构
- **数据**：文件夹里的文档

```
文件柜 (.opencode/state/)
├── session/          ← 会话数据
│   └── proj_xxx/
│       └── sess_yyy/
├── share/            ← 分享数据
└── permission/       ← 权限数据
```

---

## 核心概念

### 命名空间路径

使用数组表示层级结构：

```typescript
// 路径示例
;["session", "proj_123", "sess_456", "info"]
// 对应文件: .opencode/state/session/proj_123/sess_456/info.json
```

### CRUD 操作

```typescript
// 写入
await Storage.write(["session", projectID, sessionID, "info"], data)

// 读取
const data = await Storage.read(["session", projectID, sessionID, "info"])

// 列出
const sessions = await Storage.list(["session", projectID])

// 删除
await Storage.remove(["session", projectID, sessionID])
```

---

## 文件结构

```
storage/
├── storage.ts   # Storage 实现
└── README.md    # 本文档
```

### storage.ts 核心 API

```typescript
export namespace Storage {
  // 写入数据
  export async function write<T>(path: string[], data: T): Promise<void>

  // 读取数据
  export async function read<T>(path: string[]): Promise<T | undefined>

  // 列出子项
  export async function list(path: string[]): Promise<string[]>

  // 删除数据
  export async function remove(path: string[]): Promise<void>

  // 获取物理文件路径
  export function filePath(path: string[]): string
}
```

---

## 数据存储位置

```
~/.local/state/opencode/    # 或 .opencode/state/
├── session/                # 会话数据
│   └── {projectID}/
│       └── {sessionID}/
│           ├── info.json       # 会话元信息
│           ├── messages/       # 消息
│           └── parts/          # 消息部件
├── share/                  # 分享数据
├── permission/             # 权限记录
└── auth/                   # 认证信息
```

---

## 使用示例

### 保存会话

```typescript
await Storage.write(["session", projectID, sessionID, "info"], {
  id: sessionID,
  title: "我的会话",
  created: Date.now(),
})
```

### 读取会话列表

```typescript
const sessions = await Storage.list(["session", projectID])
// ["sess_001", "sess_002", ...]

for (const id of sessions) {
  const info = await Storage.read(["session", projectID, id, "info"])
  console.log(info.title)
}
```

---

## 注意事项

1. **数据格式是 JSON**：不要手动编辑，容易破坏格式
2. **路径区分大小写**：在 Linux/Mac 上要注意
3. **自动创建目录**：write 会自动创建所需的目录结构
