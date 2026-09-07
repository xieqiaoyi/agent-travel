# Claude Code Session Memory 滚动摘要机制——可借鉴要点

> 依据：claude-code-analysis-main 源码（src/services/SessionMemory/sessionMemory.ts、sessionMemoryUtils.ts）+ analysis/04-agent-memory.md 第 6、11 节
> 用途：短期记忆 / 会话内压缩设计的设计参考与面试弹药

---

## 一、定位：它是什么、不是什么

**是什么**：当前会话的滚动摘要层。会话变长之后，后台异步维护一份"到目前为止发生了什么"的文本摘要，随对话推进不断更新。

**不是什么**（三个常见误解）：
- **不是长期记忆**：生命周期绑定会话，会话结束就完成使命，不跨会话（CC 五层记忆里唯一不跨会话的）
- **不是上下文压缩本身**：它是压缩的**材料供给**——autocompact 触发时直接复用 Session Memory，保证摘要质量，而不是临时现写
- **不是结构化存储**：摘要就是纯文本，没有图谱、画布、JSON schema——CC 证明文本滚动摘要足以支撑长会话

**设计原则一句话**：摘要可以重算，原文不能丢——Session Memory 只影响"发给模型什么"，transcript 原文一字不删。

---

## 二、触发机制：三阈值 + 自然断点

阈值定义（sessionMemoryUtils.ts，源码原文数字）：

```
minimumMessageTokensToInit  = 10000   会话没这么长，根本不启用
minimumTokensBetweenUpdate  = 5000    启用后也不每轮更新
toolCallsBetweenUpdates     = 3       工具调用次数参与判断
```

触发判断（sessionMemory.ts:134 `shouldExtractMemory`）：

```ts
const shouldExtract =
  (hasMetTokenThreshold && hasMetToolCallThreshold) ||   // token 涨够了 且 工具调用够多
  (hasMetTokenThreshold && !hasToolCallsInLastTurn)      // token 涨够了 且 上一轮没有工具调用
```

两个设计点：

1. **token 阈值是必要条件**——对话短的时候摘要纯属浪费，10000 token 以下连初始化都不做
2. **`!hasToolCallsInLastTurn` 是自然断点保护**——上一轮没有 tool_use 才触发，防止在工具调用链中间截断，生成"引用了还不存在的结果"的孤立摘要

---

## 三、更新机制：后台异步，绝不阻塞

- 摘要更新由**后台 forked subagent** 执行——主对话该干嘛干嘛，摘要在副线程里慢慢写
- 更新完写回会话专属的 Session Memory 文件
- 失败不影响主链路：原文和最近上下文都在，下一轮还能重试

**核心取舍**：摘要的实时性让位于主对话的响应延迟。宁可摘要旧一轮，不让用户多等一秒。

---

## 四、存储与权限

```ts
// sessionMemory.ts:183
await fs.mkdir(sessionMemoryDir, { mode: 0o700 })   // 目录只有属主可读写执行
```

- 每个会话一个专属目录，文件权限 0700——摘要里可能有敏感对话内容，权限按最严格给
- 存储介质是文件不是数据库：会话级、临时、整份读给模型，文件是最匹配的形态

---

## 五、与压缩的联动（它真正的价值所在）

分析文档第 11 节（Memory Compaction）：**Session Memory 直挂在 compaction 上**——

```
平时：    Session Memory 按阈值静默更新（成本低，摘要短）
压缩时：  autocompact 触发 → 直接拿 Session Memory 当摘要底稿 →
          旧历史替换为摘要 + 保留最近原文
配合：    压缩同样遵守工具链断点保护（不在 tool_use 链中间切）
```

**为什么不压缩时才现写摘要**：现写意味着压缩那一刻要付出一次昂贵的全文总结调用，且用户正在等；滚动更新把这份成本摊到了平时每一次空闲时刻，压缩变成"取现成的"。这是**用平时的小成本换关键时刻的零延迟**。

---

## 六、极端变体：KAIROS 模式（写入与整理彻底分离）

memdir.ts 里的 KAIROS 分支展示了滚动摘要思想的极端形态：

- 会话中模型**只往 append-only 的日期日志文件追加**（"Write each entry as a short timestamped bullet... Do not rewrite or reorganize the log"）
- **MEMORY.md 索引由夜间 dream 任务从日志蒸馏**，模型日常不许直接编辑索引（"Read it for orientation, but do not edit it directly"）

启示：摘要/索引的"生成"和"使用"可以彻底分离——写的时候只管便宜地追加，读的时候拿蒸馏好的成品。

---

## 七、可借鉴清单（按落地成本排序）

| # | 机制 | 一句话描述 | 落地成本 |
|---|---|---|---|
| 1 | **惰性初始化** | 会话短不做摘要，token 门槛之下零成本 | 一个 if |
| 2 | **双阈值更新** | token 增量 + 工具调用数双条件，防止每轮都摘要 | 两个计数器 |
| 3 | **自然断点** | 只在"上一轮无工具调用"时切，保证摘要不自相矛盾 | 一个判断 |
| 4 | **异步副线程更新** | 摘要更新不阻塞主对话，失败可重试 | 一个后台任务 |
| 5 | **摘要预置给压缩** | 平时滚动维护，压缩时取现成的，关键时刻零额外延迟 | 压缩逻辑读摘要文件 |
| 6 | **文件存储 + 严格权限** | 会话级临时数据用文件，0700 权限 | 几行 IO |
| 7 | **写入整理分离（KAIROS）** | 会话中只追加原始记录，蒸馏交给离线任务 | 一条管线 |

**最值得抄的组合是 1+3+5**：惰性初始化保证短会话零开销，自然断点保证摘要质量，预置给压缩保证长会话收尾不卡顿——三个都是几十行代码，但把"会话摘要"从玩具变成生产级。

---

## 八、面试口径（30 秒版）

> "会话内摘要我参考了 Claude Code 的 Session Memory：它不是压缩时才现写摘要，而是平时按阈值滚动维护——会话超过 1 万 token 才初始化，之后每增长 5 千 token 且满足工具调用条件才更新，而且只在工具链的自然断点切，避免生成引用了未出现结果的孤立摘要。更新在后台副线程做，不阻塞主对话。真正压缩触发时直接取这份现成摘要替换旧历史，所以压缩那一刻几乎没有额外延迟。这个设计的本质是把摘要成本从'关键时刻的一次大调用'摊薄成'平时多次小调用'。"
