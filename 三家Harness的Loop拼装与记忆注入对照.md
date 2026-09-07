# 三家 Harness 的 Agent Loop 拼装与记忆注入对照

> 依据：claude-code-analysis-main 源码（src/query.ts、src/memdir/、src/tools/AgentTool/）、harness-refs/opencode 源码（session/prompt/、session/instruction.ts、schema/v1/session.ts）、harness-refs/deepseek-harness 文档（docs/subsystems/、packages/context/agent-instructions/）
> 内容：loop 每一轮发给模型的 prompt 长什么样、loop 里具体做了哪些操作、记忆/指令类内容何时拼进哪个位置
> 统一场景：用户说"查一下 validate.ts 是怎么校验 token 的，给我汇报"，agent grep → read → 汇报，然后用户追问"那把 42 行修了"。[不变] = 后续轮次一个字节不变，[新增] = 本轮多出

---

## 一、逐轮请求体实录

### 1.1 Claude Code

**第 1 轮**（用户刚说完话，第一次调模型）：

```jsonc
{
  "system": [                                    // 段落数组，不是一个大字符串
    "You are Claude Code, Anthropic's official CLI for Claude.",          // [不变]
    "# Tone and style\n- Only use emojis if the user explicitly requests it...\n- Your responses should be short and concise...",  // [不变]
    "# Memory\nYou have a persistent, file-based memory system at ...(四种类型、What NOT to save、How to save)",  // [不变]
    "# Environment\ncwd: /Users/x/auth\ndate: 2026-08-24"                  // [不变]
  ],
  "tools": [                                     // [不变]
    {"name":"Grep",...}, {"name":"Read",...}, {"name":"Edit",...},
    {"name":"Agent","description":"...(agent 清单，每个一行)"}
  ],
  "messages": [
    { user: [CLAUDE.md 渲染 + MEMORY.md 索引 + Today's date] },            // [不变] user context
    { user: "查一下 validate.ts 是怎么校验 token 的，给我汇报" }             // [新增] 唯一新内容
  ]
}
```

模型返回 `tool_use Grep` → harness 执行 → 追加。

**第 2 轮**：

```jsonc
"messages": [
  前两条原样,                                                             // 不变
  { assistant: [text "我先搜一下校验逻辑。", tool_use Grep] },             // [新增]
  { user: [tool_result "src/validate.ts:42: if (verifyToken(token, user.id))..."] }  // [新增] 工具结果以 user 身份进历史
]
```

**第 3 轮**：同样追加 Read 的 tool_use + tool_result（全文 80 行）。模型返回纯文本汇报，**无 tool_use → 循环结束**。

**第 4 轮**（新用户轮"那把 42 行修了"）：前面 8 条原样 + 一条新 user 消息，进入新的工具循环（Edit → 可能 Bash 测试 → 收尾）。

### 1.2 opencode

差异点：system 是按 provider 换的模板文件；AGENTS.md 不在 system 里，作为 user 消息附件；每次模型调用多一对 step 边界 Part。

**第 1 轮**：

```jsonc
"system": [
  // 全部来自 session/prompt/anthropic.txt（因为当前用 Claude）[不变]；换 GPT 时整块换 gpt.txt
  "You are OpenCode, the best coding agent on the planet...",
  "# Tone and style...", "# Professional objectivity...", "# Task Management..."
],
"messages": [
  { user: [instruction attachment：AGENTS.md 内容] },     // [不变] 有"已附加"追踪，不重复发
  { user: "查一下 validate.ts 是怎么校验 token 的，给我汇报" }  // [新增]
]
```

**第 2 轮**（历史多出 step 边界）：

```jsonc
"messages": [
  附件消息, 用户问题,                                       // 不变
  { assistant: [StepStartPart, text + tool_use grep, StepFinishPart] },  // [新增]
  { user: [tool_result] }                                   // [新增]
]
```

第 3、4 轮与 CC 同构。**opencode 与 CC 的逐轮差异只有三处：system 模板按模型切换、指令文件挂 user 侧、step 边界显式落历史。**

### 1.3 dsh

差异点：system 是注册表按 order 合并；AGENTS.md 用 `<system-reminder>` 框注入，且边走边增量注入。

**第 1 轮**：

```jsonc
"system": [
  "[order -100] harness 身份段",          // PromptSection 注册表按序合并 [不变]
  "[order 0]    部署人格段",
  "[order 100+] 工具指导段（各插件包注册）"
],
"messages": [
  { user: "<system-reminder>\nThe following workspace instructions may be relevant to your work... They do not override system, developer, or direct user instructions.\n\nInstructions from: ~/.dsh/AGENTS.md\n...\n\nInstructions from: AGENTS.md\n测试必须连真库...\n</system-reminder>" },  // [不变] 基线
  { user: "查一下 validate.ts..." }        // [新增]
]
```

**第 2 轮**（read 触碰新目录，发现新 AGENTS.md）：

```jsonc
"messages": [
  基线 reminder, 用户问题,                                        // 不变
  { assistant: [tool_use read packages/app/src/validate.ts] },    // [新增]
  { user: [tool_result] },                                        // [新增]
  { user: "<system-reminder>\nAdditional instructions from: packages/app/AGENTS.md\n\nThese instructions apply to work under `packages/app`...\n错误处理必须返回 RFC7807 格式\n</system-reminder>" }  // [新增] dsh 独有的增量注入
]
```

文件改 → `Updated instructions from:`；文件删 → `Instructions removed:` 墓碑；SHA-1 没变绝不重发。

### 1.4 逐轮行为对照

| 轮次动作 | CC | opencode | dsh |
|---|---|---|---|
| system 每轮变不变 | 不变（段落会话内缓存） | 不变（模板+agent prompt） | 不变（注册表合并结果） |
| 项目规矩在哪 | user context（CLAUDE.md 渲染） | user 附件（防重发追踪） | user `<system-reminder>`（可增量追加） |
| 每轮新增什么 | assistant 消息 + tool_result | 同左 + step 边界 Part | 同左 + 可能的新目录 reminder / 快照 |
| 历史会被改写吗 | 不会 | 不会 | 不会 |
| 循环何时停 | 模型不再发 tool_use | 同左 | 同左 |

---

## 二、CC 主循环每轮的完整操作序列（query.ts queryLoop）

用户一个请求进入 `while(true)`，每圈循环做 15 件事（源码顺序）：

```
① 取消息     从最近 compact 边界之后截取（getMessagesAfterCompactBoundary）
② 工具结果预算  applyToolResultBudget：tool_result 总大小超预算就替换内容（第一级压缩，不调模型）
③ snip 压缩   snipCompactIfNeeded：剪掉低价值消息段（第二级）
④ 微压缩     microcompact：按 tool_use_id 细粒度清理（第三级，感知缓存）
⑤ 上下文折叠   applyCollapsesIfNeeded：可折叠段收视图（第四级；注释原话：折叠若能降到阈值下，
              就不值得花一次 LLM 调用做摘要，还能保住细粒度上下文）
⑥ 拼 system   appendSystemContext(systemPrompt, systemContext)，system 此时定型
⑦ 自动压缩    autocompact：超阈值才调模型生成摘要（第五级）；成功换消息列表，
              失败累计 consecutiveFailures（熔断器）
⑧ 选模型     按权限模式、plan 模式是否超 200k 动态选（每轮可不同）
⑨ 阻塞闸     关闭自动压缩时，逼近硬上限提前阻断（给用户留手动 /compact 空间）
⑩ 发请求     流式调用 API
⑪ 流式工具执行 StreamingToolExecutor：模型还在吐 token，已闭合的 tool_use block 立刻开始执行
⑫ 预取消费    记忆召回预取（用户轮开始发一次）+ Skill 发现预取的结果转为附件注入
⑬ 权限检查    每个工具调用过 canUseTool
⑭ 追加消息    assistant + tool_result + 附件全部 append
⑮ 判断去留    有 tool_use → 下一圈；没有 → 结束（stop-hook 可再救一次）
```

三个关键发现：
1. **压缩是五级流水线，便宜的先跑**——确定性操作优先于模型操作
2. **工具在模型没说完时就开始跑**（流式工具执行）
3. **记忆召回/Skill 发现是预取**，延迟藏进模型流式和工具执行的等待里；记忆预取整个用户轮只发一次（prompt 在轮内不变）

---

## 三、记忆/指令类内容的注入时机

### 3.1 Claude Code：四种内容四条路

| 路 | 注入内容 | 位置 | 时机 |
|---|---|---|---|
| 1 | 记忆**使用规则**（类型分类、What NOT to save、写入法） | system 的 memory 段 | 会话开始，缓存一次，全程不变 |
| 2 | **MEMORY.md 索引正文**（≤200 行/25KB） | user context（与 CLAUDE.md 同路） | 会话开始 memoize；压缩后重注入 |
| 3 | **topic 文件正文** | 尾部附件消息 `<relevant-memory>` | 每用户轮预取一次（frontmatter 扫描 → Sonnet 选 ≤5 个），命中才注入 |
| 4 | **子 agent 的记忆** | 子 agent 自己的 system prompt | spawn 瞬间（规则 + 它自己的 MEMORY.md 索引一起，同步读） |

子 agent 注入的三个关键点：
- 每次 spawn 现拼现读（`fs.readFileSync`，所以有 25KB 硬上限），拿到的永远是当时最新的记忆
- 四个 prompt 变体里只有 `buildMemoryPrompt` 带正文；主会话的 `loadMemoryPrompt` 只有规则
- **完全隔离**：只拼 `<项目>/.claude/agent-memory/<agentType>/` 这一个目录；主线程 Auto Memory、其他 agent 的记忆、主会话历史它都看不到，派发 prompt 参数是它唯一的任务来源

### 3.2 opencode：没有长期记忆库，注入两样

1. **AGENTS.md 指令文件**：`globUp` 从 cwd 到项目根 + 全局目录收集，作为 user 消息附件；按 assistant 消息追踪已附加文件，不重发；内容变化才补新附件
2. **压缩摘要**：溢出时保留 durable transcript，只替换发给模型的表示为 rolling summary + 有预算的最近尾巴；provider-native 的 reasoning/tool 消息不跨压缩边界（signature 失效）

### 3.3 dsh：三种注入时机 + 一条压缩联动

1. **基线**：会话第一次符合条件的 pre-step，`$DSH_HOME/AGENTS.md` → 项目根到 cwd 每级的 `AGENTS.md`/`CLAUDE.md` + `.local.md` overlay，同目录重复按内容折叠，一条 reminder 注入
2. **增量**：干活中触碰新目录 → `Additional instructions from:`；改 → `Updated`；删 → `Instructions removed:` 墓碑；SHA-1 digest 去重；恢复会话只追加差异
3. **动态快照**：PromptContext 物化为 user 角色快照，位置在已保留历史之后；内容没变或被压缩挤掉就不重记
4. **压缩联动**：某 scope 的指令被 compaction 挤出可见表层后，该 scope 重新启用——压缩不会让 agent 忘掉目录规矩

### 3.4 注入时机总表

| 注入内容 | CC | opencode | dsh |
|---|---|---|---|
| 记忆/指令使用规则 | system memory 段（会话初一次） | 模板自带 | 注册表 system 段 |
| 索引/基线正文 | user context（会话初 + 压缩后重注入） | 指令附件（防重发） | 基线 reminder（首个 pre-step） |
| 记忆条目正文 | 召回时附件注入（每用户轮预取一次） | 无此层 | 无此层 |
| 子 agent 记忆 | spawn 瞬间拼进自己的 system（规则+索引） | task_id 续聊带上下文 | subagent 独立上下文 |
| 压缩后行为 | user context 重注入 | 摘要顶替旧历史 | scope 重新启用 + 快照重记 |

---

## 四、核心规律（三条）

1. **system 和 tools 全会话冻结**，每轮只往消息尾部追加"模型的决策（assistant）+ 工具的结果（user 身份）"；唯一例外是 dsh 发现新目录规矩时多追加一条 reminder
2. **规则进 system（永不变），正文进 user 侧（按需/防重发）**——易变内容永远不进 system，这是三家共同的纪律
3. **子 agent 的记忆只在它出生那一刻给它，且只给它自己的那份**；主会话历史、其他 agent 的记忆一律不可见
