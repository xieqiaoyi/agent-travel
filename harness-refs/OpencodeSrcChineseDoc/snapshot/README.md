# Snapshot 模块详解

## 什么是 Snapshot？

**Snapshot（快照）** 就像给你的项目文件拍照片。每次 AI 开始工作前拍一张，工作后再拍一张，就能知道 AI 改了什么。

### 通俗比喻

```
场景：AI 要帮你重构代码

拍照前 📸 (track)
├── src/main.js    (100行)
├── src/utils.js   (50行)
└── README.md      (20行)

AI 工作中...
✏️ 修改 main.js
✏️ 新增 helper.js
✏️ 删除 utils.js

拍照后 📸 (patch)
├── src/main.js    (80行)   ← 减少了 20 行
├── src/helper.js  (30行)   ← 新文件
└── README.md      (20行)   ← 没变

对比 (diff)
┌─────────────┬───────┬───────┐
│ 文件        │ +行数 │ -行数 │
├─────────────┼───────┼───────┤
│ main.js     │ 10    │ 30    │
│ helper.js   │ 30    │ 0     │
│ utils.js    │ 0     │ 50    │
└─────────────┴───────┴───────┘
```

---

## 核心概念

### 1. Git 对象存储

Snapshot 使用独立的 Git 仓库来存储快照，不会污染你的项目仓库：

```
~/.local/share/opencode/snapshot/
└── proj_xxx/                    # 每个项目一个 git 目录
    ├── objects/                 # Git 对象
    ├── refs/
    └── HEAD
```

### 2. 数据结构

```typescript
// 文件 diff 信息
interface FileDiff {
  file: string // 文件路径（相对路径）
  before: string // 修改前内容
  after: string // 修改后内容
  additions: number // 新增行数
  deletions: number // 删除行数
}

// 补丁信息
interface Patch {
  hash: string // Git tree hash
  files: string[] // 涉及的文件路径（绝对路径）
}
```

---

## 核心函数

### `track()` - 创建快照

记录当前所有文件的状态，返回一个 hash。

```typescript
export async function track() {
  // 1. 只支持 git 项目
  if (Instance.project.vcs !== "git") return

  // 2. 检查配置是否禁用
  const cfg = await Config.get()
  if (cfg.snapshot === false) return

  // 3. 初始化独立的 git 仓库（如果需要）
  const git = gitdir()
  if (await fs.mkdir(git, { recursive: true })) {
    await $`git init`.env({ GIT_DIR: git, GIT_WORK_TREE: Instance.worktree })
    // Windows 上禁用自动换行转换
    await $`git --git-dir ${git} config core.autocrlf false`
  }

  // 4. 添加所有文件到暂存区
  await $`git --git-dir ${git} --work-tree ${Instance.worktree} add .`

  // 5. 生成 tree hash（不创建 commit，只记录文件状态）
  const hash = await $`git --git-dir ${git} --work-tree ${Instance.worktree} write-tree`.text()

  return hash.trim() // 例如: "a1b2c3d4..."
}
```

### `patch()` - 获取变更的文件列表

对比某个快照和当前状态，返回变更的文件：

```typescript
export async function patch(hash: string): Promise<Patch> {
  // 获取变更的文件名列表
  const result = await $`git diff --name-only ${hash} -- .`

  return {
    hash,
    files: result.text().trim().split("\n")
      .map(x => path.join(Instance.worktree, x))  // 转为绝对路径
  }
}

// 返回示例
{
  hash: "a1b2c3d4",
  files: [
    "/project/src/main.js",
    "/project/src/helper.js"
  ]
}
```

### `diff()` - 获取完整 diff

返回 unified diff 格式的文本：

```typescript
export async function diff(hash: string) {
  const result = await $`git diff ${hash} -- .`
  return result.text().trim()
}

// 返回示例
// --- a/src/main.js
// +++ b/src/main.js
// @@ -1,5 +1,3 @@
// -old line
// +new line
```

### `diffFull()` - 获取详细文件对比

返回每个文件的完整前后内容和统计：

```typescript
export async function diffFull(from: string, to: string): Promise<FileDiff[]> {
  const result: FileDiff[] = []

  // 使用 --numstat 获取统计信息
  for await (const line of $`git diff --numstat ${from} ${to} -- .`.lines()) {
    const [additions, deletions, file] = line.split("\t")

    // 获取文件的前后内容
    const before = await $`git show ${from}:${file}`.text()
    const after = await $`git show ${to}:${file}`.text()

    result.push({
      file,
      before,
      after,
      additions: parseInt(additions),
      deletions: parseInt(deletions),
    })
  }

  return result
}
```

### `restore()` - 恢复到某个快照

```typescript
export async function restore(snapshot: string) {
  // 使用 read-tree 加载快照，checkout-index 恢复文件
  await $`git read-tree ${snapshot} && git checkout-index -a -f`
}
```

### `revert()` - 撤销特定文件的改动

```typescript
export async function revert(patches: Patch[]) {
  const files = new Set<string>()

  for (const item of patches) {
    for (const file of item.files) {
      if (files.has(file)) continue // 避免重复处理

      // 从快照中恢复文件
      const result = await $`git checkout ${item.hash} -- ${file}`

      if (result.exitCode !== 0) {
        // 如果文件在快照中不存在，说明是新增的，需要删除
        await fs.unlink(file)
      }

      files.add(file)
    }
  }
}
```

### `cleanup()` - 定期清理旧快照

```typescript
const prune = "7.days"

export async function cleanup() {
  // 使用 git gc 清理 7 天前的对象
  await $`git gc --prune=${prune}`
}

// 由 Scheduler 每小时执行一次
Scheduler.register({
  id: "snapshot.cleanup",
  interval: 60 * 60 * 1000, // 1 小时
  run: cleanup,
  scope: "instance",
})
```

---

## 工作流程

### 完整的 AI 工作流程

```
用户发送消息
      ↓
[step-start] 创建快照
      │
      │  const startSnapshot = await Snapshot.track()
      │  // 记录 startSnapshot 到 step-start part
      ↓
AI 开始工作
      │
      │  调用 write/edit 工具
      │  每次工具调用后记录 patch
      ↓
[step-finish] 再次创建快照
      │
      │  const endSnapshot = await Snapshot.track()
      │  const patch = await Snapshot.patch(startSnapshot)
      │  // 记录 endSnapshot 和 patch 到 step-finish part
      ↓
生成会话摘要
      │
      │  const diffs = await Snapshot.diffFull(startSnapshot, endSnapshot)
      │  // 统计：+100 行 / -50 行 / 5 文件
      ↓
用户可以撤销
      │
      │  await Snapshot.revert(patches)
      │  // 恢复到 startSnapshot 的状态
```

### 撤销流程详解

```
用户说："撤销 AI 的改动"
      ↓
收集所有 patches
      │
      │  patches = [
      │    { hash: "snap1", files: ["a.js", "b.js"] },
      │    { hash: "snap2", files: ["c.js"] },
      │  ]
      ↓
保存当前状态（用于取消撤销）
      │
      │  currentSnapshot = await Snapshot.track()
      ↓
执行撤销
      │
      │  await Snapshot.revert(patches)
      │  // a.js → 恢复到 snap1 版本
      │  // b.js → 恢复到 snap1 版本
      │  // c.js → 恢复到 snap2 版本
      ↓
用户可以取消撤销
      │
      │  await Snapshot.restore(currentSnapshot)
```

---

## 数据存储位置

```
~/.local/share/opencode/
├── snapshot/                    # 快照 git 仓库
│   └── proj_xxx/
│       ├── objects/
│       └── ...
└── state/
    └── session_diff/            # diff 结果缓存
        └── sess_xxx.json
```

### session_diff JSON 结构

```json
[
  {
    "file": "src/main.js",
    "before": "const x = 1;",
    "after": "const x = 2;",
    "additions": 1,
    "deletions": 1
  },
  {
    "file": "src/new.js",
    "before": "",
    "after": "// new file",
    "additions": 1,
    "deletions": 0
  }
]
```

---

## 与其他模块的关系

```
Session
   │
   ├── processor.ts
   │      │
   │      │  step-start: track()
   │      │  step-finish: patch(), track()
   │      ↓
   ├── summary.ts
   │      │
   │      │  computeDiff() → diffFull()
   │      ↓
   └── revert.ts
          │
          │  revert() → Snapshot.revert()
          │  unrevert() → Snapshot.restore()
          ↓
       Share
          │
          │  读取 session_diff 展示在分享页面
```

---

## 配置选项

```json
{
  "snapshot": false // 禁用快照功能
}
```

禁用后：

- `track()` 返回 undefined
- 无法查看文件 diff
- 无法使用撤销功能
- 分享页面不显示文件改动

---

## 常见问题

### Q: 为什么使用独立的 Git 仓库？

1. **不污染项目仓库**：快照不会出现在 `git status` 或 `git log` 中
2. **无需 commit**：使用 `write-tree` 只记录文件状态，不创建提交历史
3. **自动清理**：7 天前的快照会被自动垃圾回收

### Q: 快照占用多少空间？

- 使用 Git 的对象存储，相同内容只存一份
- 每小时清理 7 天前的数据
- 一般占用几十 MB

### Q: 非 Git 项目怎么办？

当前版本不支持非 Git 项目的快照功能。如果 `Instance.project.vcs !== "git"`，所有快照函数都会跳过。

### Q: 二进制文件怎么处理？

二进制文件（additions/deletions 为 "-"）会被记录，但 `before`/`after` 内容为空字符串。

---

## 调试技巧

1. **查看快照 git 仓库**

   ```bash
   cd ~/.local/share/opencode/snapshot/proj_xxx
   git log --oneline  # 查看记录
   git show <hash>    # 查看某个快照的文件
   ```

2. **查看 diff 缓存**

   ```bash
   cat ~/.local/share/opencode/state/session_diff/sess_xxx.json | jq
   ```

3. **强制清理**
   ```bash
   rm -rf ~/.local/share/opencode/snapshot/proj_xxx
   ```
