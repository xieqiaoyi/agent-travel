# Claude Code 与差旅助手 Prompt 拼接 / KV Cache 对照

> 依据：claude-code-analysis-main 源码（src/services/api/claude.ts、src/utils/api.ts、src/constants/prompts.ts、src/context.ts、src/services/api/promptCacheBreakDetection.ts）
> 对照对象：差旅助手"固定前缀 + 稳定历史 + 动态区 + 双缓存断点"消息架构
> 用途：面试深挖"上下文拼装与缓存优化"时的完整弹药库

---

## 一、Claude Code 的三条拼装线

CC 的上下文不是"一个大 system prompt"，而是三条独立的线，在会话生命周期的不同时刻触发：

| 时机 | 拼什么 | 入口 |
|---|---|---|
| ① 会话开始，每轮 API 请求 | 主 system prompt（分段组装） | `getSystemPrompt()` `constants/prompts.ts:444` |
| ② 会话开始，memoize 整个会话 | user context（CLAUDE.md 正文 + 日期） | `getUserContext()` `context.ts:155` |
| ③ 每次派 subagent 时 | subagent 自己的 system prompt | agent 定义的 `getSystemPrompt()` |

### 主 system prompt：分段 + 按名缓存

`getSystemPrompt()` 返回数组，每段用 `systemPromptSection('名字', ...)` 包起来，**一个会话内只算一次**：

```ts
// constants/prompts.ts:491-507
const dynamicSections = [
  systemPromptSection('session_guidance', ...),
  systemPromptSection('memory', () => loadMemoryPrompt()),  // 只有记忆使用规则，无正文
  systemPromptSection('env_info_simple', ...),
  systemPromptSection('language', ...),
  systemPromptSection('output_style', ...),
]
```

`memory` 段拼的是**记忆治理规则**（四种类型、What NOT to save、How to save、召回验证），不是记忆正文——正文走 user context（见下）。

### CLAUDE.md / MEMORY.md 正文：走 user context，不走 system prompt

```ts
// context.ts:155 —— memoize 整个会话
export const getUserContext = memoize(async () => {
  const claudeMd = getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))
  return { ...(claudeMd && { claudeMd }), currentDate: `Today's date is ...` }
})
```

- CLAUDE.md 总长上限 `MAX_MEMORY_CHARACTER_COUNT = 40000`（claudemd.ts:92）
- compaction 之后可无损重注入（`postCompactCleanup.ts` 注释：getUserContext 是 memoized 外层）
- **为什么放 user 侧**（推导）：CLAUDE.md 随目录变化，放 system prompt 会破坏最长公共前缀的缓存命中；放 user context，system 前缀保持稳定

### subagent：规则 + MEMORY.md 正文一起进 system prompt

```ts
// loadAgentsDir.ts:481-488
getSystemPrompt: () => systemPrompt + '\n\n' + loadAgentMemoryPrompt(name, parsed.memory)
```

`buildMemoryPrompt()` 同步读 MEMORY.md（React render 路径不能 async），25KB 硬截断，目录创建 fire-and-forget（写入必然晚于 mkdir 完成 + FileWriteTool 双保险）。

**分界逻辑一句话**：变化频率高的内容往对话侧放，保护 system prompt 前缀的缓存命中。

---

## 二、Loop 中每一轮的请求体结构

```
① system:   [块1 计费头][块2 静态前缀][块3 静态主体][块4 动态尾巴]  ← 分段各带不同缓存标记
② tools:    [工具 schema 数组，整体带 cache_control]
③ messages: [user context 前缀 + 完整历史 + 本轮新消息]            ← 尾部挂一个断点
```

### ① system 内部的分界标记与四块拆分

`prompts.ts:114`：

```ts
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

发请求前 `splitSysPromptPrefix()`（`utils/api.ts:321`）按边界切成最多 4 块，各带不同缓存作用域：

```
- Attribution header                cacheScope=null（不缓存）
- System prompt prefix              cacheScope=null
- Static content before boundary    cacheScope='global'  ← 固定前缀，跨用户共享
- Dynamic content after boundary    cacheScope=null      ← 动态区，不缓存
```

铁律（`prompts.ts:344` 注释）：

> Session-variant guidance that would **fragment the cacheScope:'global' prefix if placed before SYSTEM_PROMPT_DYNAMIC_BOUNDARY**.

会随会话变化的内容（日期、环境信息、记忆规则等动态段）必须排到边界之后；身份定义、工具规范排边界之前。**位置放错就是缓存事故。**

有 MCP 工具时降级为 3 块 org 级缓存（`skipGlobalCacheForSystemPrompt`），因为工具列表本身是动态的。

### ② tools 块：整体一个缓存单元

- 工具 schema 数组带自己的 `cache_control`（`api.ts:228`）
- 缓存键含 schema 哈希：`${tool.name}:${jsonStringify(inputJSONSchema)}`——name-only 键曾导致错误率 5.4% → 51%（PR#25424）
- **任何一个工具的 description 变了，整个 tools 块缓存失效**（生产数据：77% 的缓存失效来自工具 schema 悄悄变化）

### ③ messages：append-only + 尾部单断点

`addCacheBreakpoints()`（`claude.ts:3063`）：

```ts
// Exactly one message-level cache_control marker per request.
const markerIndex = skipCacheWrite ? messages.length - 2 : messages.length - 1
```

- 消息级断点**只有一个**，永远打在最后一条消息
- 下一轮新消息追加，断点前移；上一轮断点之前的内容全部变成命中缓存的稳定前缀
- 历史字节永不改写（append-only）——变的只有断点位置

**为什么是"一个"而不是两个**（`claude.ts:3078-3088` 注释原文）：

> Mycro's turn-to-turn eviction frees local-attention KV pages at any cached prefix position NOT in cache_store_int_token_boundaries. **With two markers the second-to-last position is protected and its locals survive an extra turn even though nothing will ever resume from there** — with one marker they're freed immediately.

服务端按断点位置保护 local-attention KV 页；两个断点会让倒数第二个位置的页多存活一轮却永远不会被续算，纯浪费显存。

**fork 特例**：fire-and-forget 的旁路查询（subagent/sideQuery）把断点挪到**倒数第二条**——那是与主线程的最后一个共享前缀点，缓存写入变 no-op merge，fork 不会把自己独有的尾巴写进缓存池。

---

## 三、缓存失效自检系统（运维面）

`services/api/promptCacheBreakDetection.ts`——每轮对请求各部分做指纹对比：

- system 哈希、tools 哈希、**每个工具单独的 schema 哈希**
- beta 头列表、model、effort、extra body params
- 专抓"不该破缓存却破了"的变量：AFK 头、超额 TTL、cache-editing 头——修复手段是 **sticky-on latched**（一旦开启就会话内锁死，避免状态翻转破缓存）
- 发现失效就生成 diff 文件，精确定位"是哪个工具的 description 变了"

**对应方法论**：架构防止失效（四件套），监控定位失效（指纹对比）。

---

## 四、对照：差旅四件套 ↔ Claude Code

| 差旅架构 | Claude Code 对应 | 实现证据 |
|---|---|---|
| **固定前缀** | 边界前的静态 system 块（`global` 作用域）+ tools schema 块 | `splitSysPromptPrefix` 四块拆分 |
| **稳定历史** | messages append-only，历史字节不变，只有断点前移 | `addCacheBreakpoints` 只改最后一条 |
| **动态区** | 边界后的 system 段 + user context（CLAUDE.md/日期放 user 消息前缀，不进 system） | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` + `getUserContext` |
| **双缓存断点** | system/tools 块断点 + 消息尾部断点（消息级只有 1 个） | `buildSystemPromptBlocks` + `markerIndex` |

**结论**：四件套与 CC 完全同构，一件不缺。差旅甚至做对了 CC 也做的事——把易变内容（日期、用户画像）推出 system prompt 放进 user 消息侧。

### 唯一实质差异：断点数量

- 差旅：双断点（固定前缀后一个 + 稳定历史尾部一个），显式声明两段共享边界
- CC：消息级刻意只用一个，依赖对服务端 KV 页淘汰机制的了解

**面试话术（差异转优势）**：

> "我的双断点一个打在固定前缀之后、一个打在稳定历史尾部，分别保护系统段和对话段。Claude Code 的消息级断点收敛成了一个，源码注释里写了原因：服务端按断点位置保护 local-attention KV 页，两个断点会让中间那批页多存活一轮却永远不会被复用。这个优化依赖对服务端淘汰机制的了解；在只提供隐式前缀缓存的服务商上，双断点能显式声明两段共享边界，收益更确定——断点数量要看服务端的缓存接口形态来选。"

---

## 五、CC 其他值得引用的缓存细节

1. **fork child 继承 thinking config**：普通 subagent 关 thinking 省 token，但 fork（并行旁路）**必须继承父配置**，否则 API 请求前缀不一致、prompt cache 不命中（`runAgent.ts` 注释）
2. **MEMORY.md 同步读 + 硬截断**：常驻内容必须限死大小（200 行/25KB），否则阻塞 + prompt 膨胀双杀
3. **CLAUDE.md memoize 会话级**：会话内不重读磁盘，靠 `clearMemoryFileCaches()` 失效
4. **缓存键教训**：只拿工具名做缓存键 → 错误率 5.4% 升到 51%（PR#25424），key 必须含 schema 哈希

---

## 六、面试口径（30 秒版）

> 我针对多轮链路里动态内容破坏 KV Cache 的问题，设计了"固定前缀 + 稳定历史 + 动态区 + 双缓存断点"的消息架构。后来对照 Claude Code 源码，发现思路完全同构：它用 SYSTEM_PROMPT_DYNAMIC_BOUNDARY 把 system prompt 切成静态块（global 缓存）和动态块（不缓存），易变内容如日期和项目说明放 user 消息前缀不进 system，对话历史 append-only 只在尾部打缓存断点。两个差异点：一是它的消息级断点只有一个，注释里解释了服务端 KV 页按断点保护、多断点浪费显存的机制；二是它有一套缓存失效自检，每轮对各段做指纹哈希，能定位到具体哪个工具的 schema 变了——77% 的失效来自工具描述悄悄变化。我的双断点在只提供隐式缓存的服务商上收益更确定，自检机制我也做了同款。
