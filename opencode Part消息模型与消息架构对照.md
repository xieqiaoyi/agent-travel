# opencode Part 消息模型与消息架构对照

> 依据：harness-refs/opencode 源码（packages/schema/src/v1/session.ts、packages/opencode/src/session/message-v2.ts、session/processor.ts）
> 对照对象：差旅助手"固定前缀 + 稳定历史 + 动态区 + 双缓存断点"消息架构
> 用途：面试深挖"消息结构设计"时的完整弹药库

---

## 一、Part 模型全景

### 1.1 十二种 Part（schema/v1/session.ts:357）

```ts
export const Part = Schema.Union([
  TextPart,        // 文本内容
  SubtaskPart,     // 子任务（Task 工具派发记录）
  ReasoningPart,   // 模型推理内容
  FilePart,        // 文件引用/附件
  ToolPart,        // 工具调用与结果
  StepStartPart,   // 步骤开始边界
  StepFinishPart,  // 步骤结束边界
  SnapshotPart,    // 快照
  PatchPart,       // 补丁/变更
  AgentPart,       // agent 相关
  RetryPart,       // 重试记录
  CompactionPart,  // 压缩摘要
]).annotate({ discriminator: "type" })
```

- 判别联合（discriminated union），按 `type` 字段分发
- Part ID 前缀固定 `prt`（`Schema.String.check(Schema.isStartsWith("prt"))`）
- **显式版本化**：schema 在 `v1/` 目录，消息模块叫 `message-v2.ts`——消息结构本身有版本演进机制

### 1.2 存储与事件（message-v2.ts）

- Part 持久化在 **SQL 表**（`PartTable`），按 `message_id` + `id` 排序读取
- 变更走事件流：`PartUpdated` / `PartDelta` / `PartRemoved`——**消息不是被整体重写的对象，是 Part 粒度增删改的事件序列**
- 发送给模型前做**表示层转换**：Part 存储形态 → provider 消息格式（如 OpenAI 系不支持 tool result 里带媒体，就把媒体抽出来合成一条 user 消息附件，`SYNTHETIC_ATTACHMENT_PROMPT`）

**关键结论：存储形态和传输形态分离。** 存的是结构化 Part，发什么形态由 provider 适配器决定。

---

## 二、Part 模型 vs 差旅四件套：不在同一层

| | 差旅"固定前缀+稳定历史+动态区+双断点" | opencode Part 模型 |
|---|---|---|
| 层次 | **传输布局层** | **存储表示层** |
| 回答的问题 | 哪些字节放前面、断点打哪 | 一条消息由哪些类型化片段组成 |
| 优化目标 | KV cache 命中 | 可持久化、可回放、按类型分别处理 |
| 依赖关系 | 不依赖 Part 也能实现（dict 列表硬拼） | 有 Part 也不自动等于做了缓存布局 |

**准确表述：一个是摆盘，一个是食材分盒。Part 模型不是"另一种消息架构"，而是让缓存友好架构能干净落地的数据结构。**

---

## 三、Part 化如何支撑"稳定历史"三原则（核心价值）

差旅架构最难落实的三个原则，opencode 各有一个 Part 机制天然支撑：

### 3.1 compaction Part → 压缩不改写历史

没有类型化结构时，压缩只能"改旧消息"或"删旧消息"，"稳定历史字节不变"就破了。opencode 的做法：**压缩 = 追加一个 CompactionPart，旧 Part 一个字节不动**。

- `message-v2.ts:49` 的 `truncateToolOutput()`：压缩时只截断 tool 输出，保留 `[Tool output truncated for compaction: omitted N chars]` 标记
- dsh 同理：摘要由带 `surfaceOp: {op:'replace'}` 的新消息承载，事件流本身不删

**append-only 与压缩共存，靠的是"压缩产物也是新 Part"。**

### 3.2 StepStart/StepFinish Part → 断点的合法锚点

双断点要打在"稳定历史尾部"，但工具调用链中间不能切（tool_use/tool_result 配对完整性）。显式的 step 边界 Part 就是切割锚点：**断点只能落在 step-finish 之后**。

对比：CC 没有 Part，只能保守地"永远打最后一条消息"（单断点策略）；有 Part 化的结构，断点位置可以精细且安全。

### 3.3 Tool Part 类型化 → 选择性剪枝

压缩时"只剪贵的、不剪重要的"：tool 结果是 token 大头，按类型只截断 ToolPart，Text/Reasoning 不动。没有 Part 化的扁平消息结构，这种选择性剪枝只能靠正则猜边界。

---

## 四、三家消息表示对比

| 维度 | Claude Code | opencode | dsh | 差旅现状 |
|---|---|---|---|---|
| 表示单元 | 类型化 block（text/tool_use/tool_result/thinking） | **Part（12 类，判别联合）** | append-only 事件流 | role+content 扁平 dict |
| 持久化 | JSONL transcript | **SQL PartTable + 事件** | 事件日志 | JSON 文件 |
| 压缩方式 | compaction + Session Memory 摘要 | 追加 CompactionPart | surfaceOp replace 新消息 | 无 |
| 历史可变性 | 不改写（纪律保证） | **不改写（结构保证）** | 不改写（结构保证） | — |
| 传输适配 | block → provider 格式 | Part → provider 格式（含媒体合成） | surface → 请求 | 直接拼字符串 |
| 版本机制 | 无显式 | **schema v1 / message-v2** | SESSION_FORMAT_VERSION=0 | 无 |

谱系总结：**CC 靠纪律达成稳定（单断点 + 永不改历史），opencode/dsh 靠类型化结构达成稳定**——前者实现简单，后者可扩展（精细断点、选择性剪枝、按 Part 回放）。

---

## 五、对差旅项目的落地建议

现状：消息是 role+content 扁平结构，压缩（如果有）只能重写列表。

最小 Part 化改造：

1. 每段内容加 `part_type` 字段：`system-section` / `history` / `tool-result` / `dynamic` / `compaction-marker`
2. 压缩逻辑从"重写列表"改成"**追加 compaction 段 + 给旧 tool-result 段打截断标记**"
3. 断点只允许落在完整步骤之后（tool 调用与结果成对）

改动不大，但"稳定历史不可变"就从口头原则变成结构保证——面试被追问"你怎么保证压缩不破坏缓存前缀"时，答"压缩是追加 compaction 段，历史字节从不改写"就有数据结构背书。

---

## 六、面试口径（30 秒版）

> "我的消息架构和 opencode 的 Part 模型不在一层：我解决请求布局——什么放前面保缓存、断点打哪；它解决消息表示——一条消息由哪些类型化片段组成，text、tool、step 边界、compaction 摘要都是独立的 Part 类型，持久化在 SQL 里走事件流更新。Part 化的价值在于它是缓存友好架构的实现基座：compaction 作为独立 Part 让压缩变成追加而不是改写，稳定历史才真正成立；step-start/finish 给断点提供合法锚点；tool Part 类型化让压缩能选择性剪枝。CC 靠'单断点加永不改历史'的纪律达成同样的稳定，opencode 和 dsh 靠类型化结构达成——前者简单，后者可扩展。我的消息目前是扁平结构，Part 化是我规划的下一步改造。"
