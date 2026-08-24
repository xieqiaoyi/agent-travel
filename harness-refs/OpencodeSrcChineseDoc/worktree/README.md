# Worktree 模块详解

## 什么是 Worktree？

**Worktree（工作树）** 是 Git 的一个功能，可以让你在同一个仓库下有多个工作目录。OpenCode 用它来创建"沙盒"——一个独立的实验区域，AI 可以在里面自由发挥而不影响你的主代码。

### 通俗比喻

```
你的项目
├── 主工作区 (main)          ← 你正在写代码的地方
│   └── src/...
│
└── OpenCode 工作树 (sandboxes)
    ├── brave-falcon/        ← AI 的沙盒 #1
    │   └── src/...
    └── cosmic-wizard/       ← AI 的沙盒 #2
        └── src/...
```

就像给 AI 一间独立的"实验室"，它可以随便折腾，不会弄乱你的主代码。

---

## 为什么需要 Worktree？

| 场景          | 没有 Worktree                    | 有 Worktree                  |
| ------------- | -------------------------------- | ---------------------------- |
| AI 尝试新方案 | 直接改你的代码，失败了要手动恢复 | 在沙盒里试验，不满意直接删掉 |
| 并行任务      | 一次只能做一件事                 | 可以在不同沙盒处理不同任务   |
| 危险操作      | 可能破坏你的工作                 | 隔离环境，安全无忧           |

---

## 核心概念

### 1. 命名规则

Worktree 使用"形容词-名词"的随机命名：

```typescript
const ADJECTIVES = ["brave", "calm", "clever", "cosmic", "crisp", ...]
const NOUNS = ["cabin", "cactus", "canyon", "circuit", "comet", ...]

// 生成名称: brave-falcon, cosmic-wizard, clever-otter...
function randomName() {
  return `${pick(ADJECTIVES)}-${pick(NOUNS)}`
}
```

### 2. 分支命名

每个 worktree 对应一个 Git 分支：

```
worktree 名称: brave-falcon
分支名称: opencode/brave-falcon
```

### 3. 存储位置

```
~/.local/share/opencode/worktree/
└── proj_xxx/                    # 每个项目一个目录
    ├── brave-falcon/            # worktree #1
    │   ├── src/
    │   └── ...
    └── cosmic-wizard/           # worktree #2
        ├── src/
        └── ...
```

---

## 核心函数

### `create()` - 创建新的 Worktree

```typescript
export const create = fn(CreateInput.optional(), async (input) => {
  // 1. 只支持 git 项目
  if (Instance.project.vcs !== "git") {
    throw new NotGitError({ message: "Worktrees are only supported for git projects" })
  }

  // 2. 确定存储目录
  const root = path.join(Global.Path.data, "worktree", Instance.project.id)
  await fs.mkdir(root, { recursive: true })

  // 3. 生成唯一名称
  const info = await candidate(root, input?.name)
  // info = { name: "brave-falcon", branch: "opencode/brave-falcon", directory: "..." }

  // 4. 创建 Git worktree
  await $`git worktree add --no-checkout -b ${info.branch} ${info.directory}`

  // 5. 记录到项目的 sandboxes 列表
  await Project.addSandbox(Instance.project.id, info.directory)

  // 6. 异步初始化（不阻塞返回）
  setTimeout(async () => {
    // 检出文件
    await $`git reset --hard`.cwd(info.directory)

    // 启动项目（如果配置了启动命令）
    await runStartScripts(info.directory, { projectID, extra: input?.startCommand })

    // 发布就绪事件
    GlobalBus.emit("event", {
      type: Event.Ready.type,
      properties: { name: info.name, branch: info.branch },
    })
  }, 0)

  return info
})
```

### `remove()` - 删除 Worktree

```typescript
export const remove = fn(RemoveInput, async (input) => {
  // 1. 获取 worktree 列表
  const list = await $`git worktree list --porcelain`

  // 2. 找到目标 worktree
  const entry = entries.find((item) => path.resolve(item.path) === directory)

  // 3. 删除 worktree
  await $`git worktree remove --force ${entry.path}`

  // 4. 删除对应的分支
  const branch = entry.branch.replace(/^refs\/heads\//, "")
  await $`git branch -D ${branch}`

  return true
})
```

### `reset()` - 重置 Worktree

把 worktree 重置到主分支的最新状态（丢弃所有改动）：

```typescript
export const reset = fn(ResetInput, async (input) => {
  // 1. 不能重置主工作区
  if (directory === Instance.worktree) {
    throw new ResetFailedError({ message: "Cannot reset the primary workspace" })
  }

  // 2. 找到默认分支（origin/main 或 origin/master）
  const target = await findDefaultBranch()

  // 3. 拉取最新代码
  await $`git fetch ${remote} ${remoteBranch}`

  // 4. 硬重置到目标
  await $`git reset --hard ${target}`

  // 5. 清理未跟踪的文件
  await $`git clean -fdx`

  // 6. 更新子模块
  await $`git submodule update --init --recursive --force`

  // 7. 重新运行启动脚本
  queueStartScripts(worktreePath, { projectID })

  return true
})
```

---

## 工作流程

### 创建 Worktree 的完整流程

```
用户说："帮我在沙盒里试验一下"
      ↓
检查是否是 Git 项目
      │
      │  if (vcs !== "git") throw NotGitError
      ↓
生成唯一名称
      │
      │  尝试最多 26 次，直到找到：
      │  - 目录不存在
      │  - 分支不存在
      │
      │  结果: { name: "brave-falcon", branch: "opencode/brave-falcon" }
      ↓
创建 Git worktree
      │
      │  git worktree add --no-checkout -b opencode/brave-falcon ~/.../brave-falcon
      ↓
返回信息给用户
      │
      │  "沙盒 brave-falcon 正在创建..."
      ↓
异步初始化（后台）
      │
      ├── git reset --hard          # 检出文件
      ├── 运行项目启动命令            # npm install, etc.
      └── 发布 Ready 事件            # 通知 UI 更新
```

### Worktree 生命周期

```
创建
  │
  │  create({ name: "my-feature" })
  ↓
使用
  │
  │  AI 在 worktree 中工作
  │  - 创建/修改文件
  │  - 运行测试
  │  - 提交代码
  ↓
重置（可选）
  │
  │  reset({ directory: "..." })
  │  - 丢弃所有改动
  │  - 回到主分支状态
  ↓
删除
  │
  │  remove({ directory: "..." })
  │  - 删除 worktree
  │  - 删除 Git 分支
```

---

## 事件系统

```typescript
export const Event = {
  // Worktree 准备就绪
  Ready: BusEvent.define(
    "worktree.ready",
    z.object({
      name: z.string(), // "brave-falcon"
      branch: z.string(), // "opencode/brave-falcon"
    }),
  ),

  // Worktree 创建失败
  Failed: BusEvent.define(
    "worktree.failed",
    z.object({
      message: z.string(),
    }),
  ),
}
```

---

## 错误类型

| 错误                                | 原因                           |
| ----------------------------------- | ------------------------------ |
| `WorktreeNotGitError`               | 项目不是 Git 仓库              |
| `WorktreeNameGenerationFailedError` | 尝试 26 次后仍无法生成唯一名称 |
| `WorktreeCreateFailedError`         | `git worktree add` 命令失败    |
| `WorktreeStartCommandFailedError`   | 启动脚本执行失败               |
| `WorktreeRemoveFailedError`         | 删除 worktree 失败             |
| `WorktreeResetFailedError`          | 重置 worktree 失败             |

---

## 与其他模块的关系

```
Project
   │
   │  sandboxes: string[]  # 记录所有 worktree 路径
   ↓
Worktree
   │
   │  创建/删除时更新 Project.sandboxes
   ↓
Instance
   │
   │  Instance.worktree: 当前工作区路径
   │  containsPath(): 判断路径是否在工作区内
   ↓
Permission
   │
   │  检查文件是否在允许的路径范围内
```

---

## 配置选项

### 项目启动命令

```json
{
  "commands": {
    "start": "npm install && npm run build"
  }
}
```

创建 worktree 后会自动运行这个命令。

### 额外启动命令

```typescript
await Worktree.create({
  name: "my-feature",
  startCommand: "npm run setup:test", // 项目命令之后运行
})
```

---

## 常见问题

### Q: Worktree 和普通 clone 有什么区别？

| 特性      | Worktree            | Clone    |
| --------- | ------------------- | -------- |
| 共享 .git | 是                  | 否       |
| 磁盘占用  | 只有文件            | 完整仓库 |
| 分支切换  | 不影响其他 worktree | 独立     |
| 适用场景  | 临时工作区          | 独立副本 |

### Q: Worktree 会占用多少空间？

只存储工作区文件，不复制 .git 目录。一个中型项目大约几十 MB。

### Q: 如何手动清理 worktree？

```bash
# 列出所有 worktree
git worktree list

# 删除特定 worktree
git worktree remove --force /path/to/worktree

# 删除对应分支
git branch -D opencode/brave-falcon
```

### Q: worktree 内的改动如何合并到主分支？

```bash
cd /path/to/worktree
git add .
git commit -m "My changes"
git checkout main
git merge opencode/brave-falcon
```

---

## 调试技巧

1. **查看所有 worktree**

   ```bash
   git worktree list
   ```

2. **检查 OpenCode worktree 目录**

   ```bash
   ls ~/.local/share/opencode/worktree/proj_xxx/
   ```

3. **查看项目的 sandbox 记录**

   ```bash
   cat .opencode/state/project/proj_xxx.json | jq .sandboxes
   ```

4. **强制清理无效 worktree**
   ```bash
   git worktree prune
   ```
