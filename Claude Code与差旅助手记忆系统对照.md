# Claude Code 与差旅助手记忆系统对照

> 依据：claude-code-analysis-main 源码（src/memdir/、src/tools/AgentTool/、src/services/）
> 对照对象：差旅助手四层语义金字塔（L0–L3）+ PreferenceAgent + 异步 LLM 总结
> 用途：面试深挖"记忆系统设计"时的完整弹药库

---

## 一、Claude Code 的记忆分层全景

五层存储 + 两个机制：

| 层 | 存在哪 | 跨会话？ | 跨项目？ | 跨机器/跨人？ |
|---|---|---|---|---|
| ① CLAUDE.md | 项目目录 / `~/.claude` | ✅ | 全局的可以 | ✅（进 git） |
| ② Session Memory | 会话专属目录 | ❌（唯一不跨的） | ❌ | ❌ |
| ③ Auto Memory（memdir） | `~/.claude/projects/<项目>/memory/` | ✅ | ❌（按项目隔离） | ❌ |
| ④ Agent Memory | 按 scope 三选一 | ✅ | 看 scope | project scope 进 git 后可以 |
| ⑤ Team Memory | 受控同步目录 | ✅ | ✅ | ✅ |

两个"不是存储层"的机制：
- **Relevant Memory Recall**（`findRelevantMemories.ts`）：读机制，决定哪几条记忆本轮进上下文
- **Memory Compaction**：治理机制，Session Memory 直挂在压缩上做兜底

分层判据：**数据的主语是"会话"还是"实体"**。主语是会话 → 不跨；主语是实体（用户/项目/agent 类型/团队）→ 跨。底层全是 markdown 文件，分层不是按技术实现切的。

### 各层要点（源码级）

**③ Auto Memory（memdir 本体，最核心）**

- 存储：`MEMORY.md` 索引 + 每条记忆一个独立 topic 文件
  - `ENTRYPOINT_NAME = 'MEMORY.md'`，`MAX_ENTRYPOINT_LINES = 200`，`MAX_ENTRYPOINT_BYTES = 25_000`（`memdir.ts:34-38`）
  - 字节闸来历：线上观察到"200 行以内但 197KB"的坏案例（单行塞整段内容绕过行数限制）
- 四种记忆类型（`memoryTypes.ts`）：`user` / `feedback` / `project` / `reference`
- 写入硬规则：**可推导性门槛**——"NOT derivable from the current project state"；代码模式、架构、git 历史、CLAUDE.md 已有内容一律不存；用户明确要求存活动清单也要改道（"问什么是 surprising / non-obvious 的"）
- feedback 类型要求**同时记失败和成功**（只记纠正会让 agent 越来越保守）
- 正文结构：规则 + **Why:** + **How to apply:** 三段式
- project 类型要求**相对日期写入时转绝对日期**（"Thursday" → "2026-03-05"）
- 召回：**两阶段**（`findRelevantMemories.ts`）
  1. `scanMemoryFiles()` 扫所有文件 frontmatter（name + description），不读正文
  2. Sonnet `sideQuery` + JSON schema 选最多 5 个文件名（`max_tokens: 256`），才读正文
  - `recentTools` 降噪：正在用的工具的 API 文档是噪音，但它的坑（gotchas）必须留
  - `alreadySurfaced`：前几轮已展示过的不重复占额度
- 使用验证义务（`TRUSTING_RECALL_SECTION`）：记忆提到函数/文件先 grep 确认还存在，"记忆说 X 存在" ≠ "X 现在存在"
- prompt 是 eval 逼出来的：同一段正文只改标题措辞从 0/3 到 3/3，放错 section 从 3/3 掉到 0/3

**记忆整合（autoDream）**

- 三级闸门（cheapest first）：时间 ≥24h（一次 stat）→ 会话数 ≥5 → 抢锁（`autoDream.ts`）
- fork 一个 subagent 跑四阶段：Orient（先读已有，防重复）→ Gather（grep transcript，不全读）→ Consolidate（合并而非新建近重复、转绝对日期、删被推翻的事实）→ Prune（索引压回 200 行/25KB）
- `extractMemories` 后台 agent 补漏：主 agent 写过的范围跳过（`hasMemoryWritesSince`）

**④ Agent Memory**

- 只给自定义 agent 用，定义文件 `memory: user|project|local` 字段开启；**内建子 agent 全部没有记忆**
- 声明后自动两件事：memory prompt 拼进 system prompt + 注入 Write/Edit/Read 文件工具（缺一不可）
- 目录：user → `~/.claude/agent-memory/<type>/`；project → `<cwd>/.claude/agent-memory/<type>/`（可进 git）；local → `.claude/agent-memory-local/`
- user scope 支持 **snapshot 分发**：记忆像依赖一样随仓库初始化/升级
- `isAgentMemoryPath()` 归一化防 `..` 穿越——agent memory 是被权限系统特殊识别的存储边界

**② Session Memory（唯一不跨会话）**

- 阈值：`minimumMessageTokensToInit = 10000`、`minimumTokensBetweenUpdate = 5000`、`toolCallsBetweenUpdates = 3`
- 触发找自然断点（上一轮无 tool_use），防止在工具链中间截断生成孤立摘要
- 落盘但生命周期绑定会话：`fs.mkdir(dir, { mode: 0o700 })`

---

## 二、MEMORY.md：存什么、怎么更新

- **只存索引，绝不存正文**。源码原话："`MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`. It has no frontmatter. Never write memory content directly into `MEMORY.md`."
- 索引行无 frontmatter（frontmatter 属于 topic 文件，给召回选择器看）
- 硬限制：200 行 / 25KB；超限 `truncateEntrypointContent()` 先按行截、再在换行处按字节截，并追加 WARNING 告诉模型哪条限制触发、该怎么修——**截断本身是反馈给模型的修复指令**
- 软限制：双步写入法（先写 topic 文件，再加索引行）；更新时 frontmatter 必须与正文同步（description 是召回选择器唯一看到的东西）；autoDream Phase 4 兜底压缩
- KAIROS 模式极端形态：模型只写 append-only 日志，夜间 dream 蒸馏成索引，写入与整理彻底分离

**更新示例（feedback 记忆演化）**：

1. 写入：`feedback_testing.md`（frontmatter + 规则/Why/How）+ MEMORY.md 一行索引
2. 召回：manifest 扫描 → Sonnet 选中 → 才读正文
3. 更新（"现在改用 testcontainers"）：Edit 旧文件正文 + 同步改 frontmatter description + 改索引行；**不新建文件**（会产生冲突记忆）
4. 兜底：autoDream 发现矛盾事实时合并或删除

---

## 三、逐层对照：差旅助手四层金字塔 ↔ Claude Code

| 差旅助手 | Claude Code 对应 | 对应度 |
|---|---|---|
| **L0 原始对话**（append-only 证据层） | 会话 transcript JSONL | 完全对应 |
| **L1 原子事实**（偏好/事实，独立检索更新） | memdir topic 文件（frontmatter + 正文） | 高度对应 |
| **L2 行程场景**（含临时约束） | `project` 类型记忆 | 对应，差旅侧更结构化 |
| **L3 用户画像**（稳定总结，常驻） | `user` 类型记忆 + MEMORY.md 的常驻角色 | 部分对应（CC 拆成了两个东西） |
| 短期记忆滑动窗口 | 会话内上下文 + Session Memory | 对应 |
| PreferenceAgent（append/replace） | feedback 记忆的更新规则（prompt 软约束） | 哲学相同，差旅侧更硬 |
| 异步 LLM 总结沉淀 | `extractMemories` + `autoDream` 两级 | 高度对应（差旅侧缺第二级） |
| Mermaid 画布 + node_mapping + refs | Session Memory + "压缩不删原始数据" | 部分对应 |

### 逐层拆解

**L0 ↔ transcript**：最干净的一对。同原则"证据只追加永不删"。差异：CC 的 transcript 是整合任务的副产品，差旅的 L0 是被检索链显式引用的一等公民（`source_msg_id`）——**差旅的可追溯比 CC 更强**。

**L1 ↔ topic 文件**：都是"一条事实独立可更新"，更新都走"改旧条目而非新增近重复"。差异：CC 用语义四分法（user/feedback/project/reference），差旅用业务字段分类（hotel_brands/airlines/seat）——差旅更窄但可被程序直接消费。

**L2 ↔ project 记忆**：都存"正在发生什么"，都要求相对日期转绝对日期。差异：CC 是自由文本，差旅是结构化表 + 状态机（planning/ongoing/done）。

**L3 ↔ user 记忆 + MEMORY.md**：对应最弱的一层。CC 把"画像内容"（user 记忆文件，按需召回）和"常驻层"（MEMORY.md 索引，200 行闸）分给两个东西；**CC 没有整段画像常驻**。差旅给 L3 设 token 上限 = 向 CC 的"索引常驻、正文按需"靠拢。

**PreferenceAgent ↔ feedback 更新规则**：同一问题的两种解法。CC 靠 prompt 求模型自觉；差旅是 LLM 输出 `{type, value, action}` 结构化动作 + 编排层校验 + 数据库部分唯一索引兜底。**这是差旅相对 CC 唯一"工程上更优"的点**。

**异步总结 ↔ extractMemories + autoDream**：CC 是两级（单次提取 + 周期整合），差旅只有一级。缺的第二级 = 语义冲突消解兜底（"优先汉庭"和"优先全季"并存问题）。

---

## 四、两个高频追问的标准答法

### Q：原始会话不是短期记忆吗？

原始对话有两种形态，**短期记忆只是它的一个视图，L0 是它的持久化本体**：

| 概念 | 生命周期 | 内容 |
|---|---|---|
| 会话内工作上下文（短期） | 会话结束即失效 | 最近 10 轮（滑动窗口） |
| L0 原始对话 | 永久 | 全量对话 append-only |
| 摘要 | 视层而定 | 压缩后的结论 |

同一条消息说出时同时进两处：进滑动窗口（短期视图）+ INSERT L0（永久证据）。**摘要可以死，证据不能死**——CC 同样：Session Memory 不跨会话，transcript 永久保留。Mermaid 画布同理：画布是短期视图可丢，refs/{tool_call_id}.md 是 L0 级证据。

### Q：多层记忆怎么在 PostgreSQL 存？

四张表 + 外键保证下钻可追溯 + status/version 保证更新不删历史：

- `mem_l0_message`：append-only，索引 (user_id, session_id, created_at)
- `mem_l1_fact`：核心表。三段式字段（fact_value/why/how_to_apply）、action_scope（long_term/ephemeral 区分临时污染）、source_msg_id 外键指 L0、status=active/superseded + replaced_by 软删除链、embedding vector(1024) 挂 L1 不挂 L0
  - **replace 用"置 superseded + 写新行"不用 UPDATE/DELETE**——历史链完整可回溯
  - 部分唯一索引只对 replace 型生效：`UNIQUE(user_id, fact_type) WHERE status='active' AND fact_type IN ('home_location', ...)`——数据库层兜住 PreferenceAgent 判错的后果
- `mem_l2_scene`：start_date DATE NOT NULL **逼着写入前转绝对日期**；临时约束 = L1 行 scope=ephemeral + scene_id 绑定
- `mem_l3_profile`：每用户一行，token_estimate 超预算触发异步重压缩

下钻 = 四条 SQL：L3 主键查画像 → L2 按目的地/日期定位场景 → L1 读 active 事实 → L0 沿 source_msg_id 核验。对比纯向量检索会把三次不同行程的酒店全召回——**分层把"语义相似"收窄成"业务相关"**。

---

## 五、CC 有而差旅设计缺的（优化清单）

1. **写入门槛**：可推导性过滤——行程记录里能查到的不重复存 L1。落点：PreferenceAgent prompt 加 NOT-TO-SAVE 清单 + 编排层查重
2. **L1 三段式**：规则 + Why + How to apply，Why 用于边缘情况判断（靠窗 vs 省钱冲突时）
3. **L3 token 上限**：常驻层必须设闸（如 500 token），超限走已有的异步总结管道重压缩
4. **冲突消解兜底**：异步沉淀前加冲突扫描，矛盾偏好标记 replace 并保留变更原因（对标 autoDream 的 Deleting contradicted facts）
5. **召回精排**：向量 top-k 之后加 LLM 精排（对标两阶段召回）

## 六、差旅有而 CC 没有的（防守点）

1. **结构化动作链路**：append/replace 由编排层校验执行，不靠 prompt 自觉
2. **临时约束/长期偏好的显式区分**："这次希尔顿"进 L2、"以后汉庭"进 L1，三信号硬判断
3. **L0 显式可追溯**：source_msg_id 锚点，CC 的召回不保证回到原文

---

## 七、面试口径（30 秒版）

> 我设计四层金字塔时参考过 Claude Code 的 memdir 实现，两者是同构的：L0 对它的 transcript，L1 对 topic 文件，L2 对 project 记忆，L3 对 user 记忆加索引层，都遵守"证据只追加、摘要可重算、索引与正文分离、召回不全量加载"。差异在两端：写入端我用 PreferenceAgent 输出 append/replace 结构化动作、编排层校验执行，比 CC 的纯 prompt 约束更可控；整合端我只有单次异步提取，没有 CC autoDream 那样的周期性整合，语义冲突消解是我下一步要补的。我还从 CC 借鉴了两点：写入的可推导性门槛，和 L1 的 Why/How-to-apply 三段式结构。
