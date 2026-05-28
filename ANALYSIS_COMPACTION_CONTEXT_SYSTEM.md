# Claude Code 源码解剖：Compaction / Context Management 系统

## 1. 总览

前九步已经串到 Session / Transcript：

```txt
启动入口
  ↓
REPL 输入
  ↓
query 主循环
  ↓
Tool 抽象
  ↓
权限系统
  ↓
Hooks 系统
  ↓
MCP 系统
  ↓
Agent / Subagent 系统
  ↓
Session / Transcript 持久化
```

第十步拆上下文管理。Claude Code 不是只靠 `/compact` 一个命令，而是有一组分层机制：

```txt
tool result budget
  单条 user message 内的 tool_result 过大时，把大结果持久化并替换成 preview

snip compact
  删除中间历史片段，保留链完整性

microcompact
  清理旧 tool_result，或用 cache_edits 删除缓存中的工具结果

context-collapse
  实验性按 span 折叠上下文，保留更细粒度历史

autocompact
  接近上下文窗口时自动总结旧历史

manual /compact
  用户主动触发总结

session-memory compact
  用独立 session memory 替代传统总结

reactive compact
  API 已经报 prompt-too-long 后的补救压缩
```

核心文件：

```txt
src/query.ts
src/services/compact/compact.ts
src/services/compact/autoCompact.ts
src/services/compact/microCompact.ts
src/services/compact/sessionMemoryCompact.ts
src/services/compact/postCompactCleanup.ts
src/services/compact/prompt.ts
src/commands/compact/compact.ts
src/utils/toolResultStorage.ts
src/utils/context.ts
src/utils/tokens.ts
src/hooks/useLogMessages.ts
src/utils/sessionStorage.ts
```

一句话：Claude Code 的上下文管理是“进入 API 前的多阶段投影 + 超阈值总结 + transcript 链重写”。REPL 可以保留完整 UI 历史，但真正发给模型的是压缩后的 API view。

---

## 2. query 主循环中的顺序

位置：

```txt
src/query.ts
```

每一轮 query 开始时先取 active context：

```ts
let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]
```

然后按顺序执行：

```txt
1. applyToolResultBudget()
2. snipCompactIfNeeded()
3. microcompact()
4. contextCollapse.applyCollapsesIfNeeded()
5. autoCompactIfNeeded()
6. blocking limit check
7. callModel()
8. reactive compact / retry recovery
```

代码顺序非常关键：

```txt
tool result budget
  在 microcompact 前，因为它只处理消息内容，不依赖 cache_edit

snip
  在 microcompact 前，因为 snip 会释放 token，autocompact 要减去 snipTokensFreed

microcompact
  在 autocompact 前，优先做更便宜、更细粒度的清理

context-collapse
  在 autocompact 前，如果 collapse 已经降到阈值以下，就不需要传统 summary compact

autocompact
  在 API call 前主动避免 prompt-too-long

reactive compact
  在 API call 失败后兜底
```

所以它不是一个单点策略，而是 pipeline。

---

## 3. active context：compact boundary 之后

压缩后，REPL 的 `messages` 可能仍保留旧消息用于 UI scrollback。

但进入 query 时：

```ts
getMessagesAfterCompactBoundary(messages)
```

会投影出 compact boundary 后的有效上下文。

REPL 里收到 compact boundary 时：

```ts
if (isCompactBoundaryMessage(newMessage)) {
  if (isFullscreenEnvEnabled()) {
    setMessages(old => [...getMessagesAfterCompactBoundary(old, {
      includeSnipped: true
    }), newMessage])
  } else {
    setMessages(() => [newMessage])
  }
  setConversationId(randomUUID())
}
```

普通模式：

```txt
压缩后 UI messages 直接重置成 [boundary]
```

fullscreen 模式：

```txt
保留一个 compact interval 的 scrollback
但 API view 仍从 boundary 后开始
```

这解释了一个重要分离：

```txt
UI 历史
  可以为了用户查看保留更多

API 上下文
  只取 compact boundary 之后
```

---

## 4. compact boundary 的作用

compact boundary 是一个 system message：

```ts
createCompactBoundaryMessage(trigger, preCompactTokenCount, lastMessageUuid)
```

它在 transcript 里承担两个角色：

```txt
1. UI/运行时：标记“之前的上下文已经被 summary 替代”
2. sessionStorage：写入时 parentUuid=null，恢复时截断旧链
```

第九步已经看到：

```txt
parentUuid = null
logicalParentUuid = compact 前 parent
```

所以恢复时：

```txt
latest leaf
  ↓ parentUuid walk
compact boundary
  ↓ parentUuid=null
stop
```

这让 `/resume` 不会把 pre-compact 全量历史重新塞回来。

---

## 5. CompactionResult 数据结构

核心类型：

```ts
export interface CompactionResult {
  boundaryMarker: SystemMessage
  summaryMessages: UserMessage[]
  attachments: AttachmentMessage[]
  hookResults: HookResultMessage[]
  messagesToKeep?: Message[]
  userDisplayMessage?: string
  preCompactTokenCount?: number
  postCompactTokenCount?: number
  truePostCompactTokenCount?: number
  compactionUsage?: ReturnType<typeof getTokenUsage>
}
```

最终 post-compact messages 的顺序统一由：

```ts
buildPostCompactMessages(result)
```

生成：

```txt
boundaryMarker
summaryMessages
messagesToKeep
attachments
hookResults
```

这个顺序是 load-bearing：

```txt
boundary 先截断旧链
summary 接在 boundary 后
messagesToKeep 保留最近历史
attachments/hookResults 补回运行上下文
```

手动 `/compact` 返回后，slash command 还会把本次 slash command 自己的 synthetic messages 追加到 `messagesToKeep`：

```ts
const compactionResultWithSlashMessages = {
  ...result.compactionResult,
  messagesToKeep: [
    ...(result.compactionResult.messagesToKeep ?? []),
    ...slashCommandMessages,
  ],
}
```

这样 `/compact` 这次命令本身也会进入 transcript。

---

## 6. 传统 compact：compactConversation

主函数：

```ts
compactConversation(
  messages,
  context,
  cacheSafeParams,
  suppressFollowUpQuestions,
  customInstructions,
  isAutoCompact,
  recompactionInfo,
)
```

流程：

```txt
1. 计算 preCompactTokenCount
2. 执行 PreCompact hooks
3. 合并用户 instructions 和 hook instructions
4. 生成 compact prompt
5. streamCompactSummary()
6. 处理 prompt-too-long retry
7. 清 readFileState / loadedNestedMemoryPaths
8. 生成 post-compact attachments
9. 执行 SessionStart hooks(source='compact')
10. 创建 compact boundary
11. 创建 isCompactSummary user message
12. 记录 telemetry / prompt-cache baseline
13. reAppendSessionMetadata()
14. 执行 PostCompact hooks
15. 返回 CompactionResult
```

这里 summary 被包装成一个 user message：

```ts
createUserMessage({
  content: getCompactUserSummaryMessage(...),
  isCompactSummary: true,
  isVisibleInTranscriptOnly: true,
})
```

为什么是 user message？

```txt
因为下一轮 API 需要把 summary 当作用户侧上下文提示
而不是 assistant 自己刚说过的话
```

`isCompactSummary` 用于后续过滤和 firstPrompt 提取，避免 `/resume` 把 summary 当成用户第一问。

---

## 7. compact prompt 和 summary 文本

文件：

```txt
src/services/compact/prompt.ts
```

`getCompactPrompt(customInstructions)` 生成给 summary agent 的指令。

`getCompactUserSummaryMessage(summary, suppressFollowUpQuestions, transcriptPath)` 把 summary 包装成 post-compact 上下文。

其中会加入 transcript path 提示：

```txt
如果需要 compaction 前的具体细节，可以读取完整 transcript
```

这是一种折中：

```txt
模型默认只看 summary
但如果需要精确细节，仍可通过文件系统读 transcript
```

也就是说，compaction 不是完全删除历史，而是把历史移出主上下文，保留在 transcript 里。

---

## 8. streamCompactSummary：两条路径

`streamCompactSummary()` 有两条路径：

```txt
优先：runForkedAgent()
回退：queryModelWithStreaming()
```

优先路径：

```ts
runForkedAgent({
  promptMessages: [summaryRequest],
  cacheSafeParams,
  canUseTool: createCompactCanUseTool(),
  querySource: 'compact',
  forkLabel: 'compact',
  maxTurns: 1,
  skipCacheWrite: true,
})
```

为什么用 forked agent？

```txt
复用主线程 prompt cache prefix
system prompt / tools / messages prefix / thinking config 保持一致
降低 compaction 本身的 cache creation 成本
```

同时禁止工具：

```ts
createCompactCanUseTool()
  → deny: "Tool use is not allowed during compaction"
```

回退路径：

```ts
queryModelWithStreaming({
  messages: normalizeMessagesForAPI([
    ...getMessagesAfterCompactBoundary(messages),
    summaryRequest,
  ]),
  systemPrompt: ['You are a helpful AI assistant tasked with summarizing conversations.'],
  thinkingConfig: { type: 'disabled' },
  tools,
  maxOutputTokensOverride: COMPACT_MAX_OUTPUT_TOKENS,
  querySource: 'compact',
})
```

回退路径只允许有限工具：

```txt
FileReadTool
ToolSearchTool
MCP tools when ToolSearch enabled
```

summary 生成前还会：

```txt
stripImagesFromMessages()
stripReinjectedAttachments()
```

避免图片/文档/旧 skill listing 污染或撑爆 compaction 请求。

---

## 9. compact 自身也可能 prompt-too-long

如果 compact API 调用本身太长：

```txt
summary starts with PROMPT_TOO_LONG_ERROR_MESSAGE
```

会触发：

```ts
truncateHeadForPTLRetry(messagesToSummarize, summaryResponse)
```

策略：

```txt
按 API round 分组
根据 tokenGap 删除最老的若干组
如果 tokenGap 不可解析，则删 20% 组
至少保留一组可总结内容
必要时插入 meta user marker，避免 assistant-first
最多 MAX_PTL_RETRIES = 3
```

这是“压缩也压不动”时的最后兜底。

---

## 10. post-compact attachments

压缩后，旧上下文被 summary 替代，但模型继续工作仍需要一些状态。

`compactConversation()` 会恢复：

```txt
最近读过的文件片段
异步 agent 任务信息
当前 plan
plan mode instructions
已调用 skill 的内容
deferred tool schema delta
agent listing delta
MCP instructions delta
SessionStart hook outputs
```

对应函数：

```txt
createPostCompactFileAttachments()
createAsyncAgentAttachmentsIfNeeded()
createPlanAttachmentIfNeeded()
createPlanModeAttachmentIfNeeded()
createSkillAttachmentIfNeeded()
getDeferredToolsDeltaAttachment()
getAgentListingDeltaAttachment()
getMcpInstructionsDeltaAttachment()
processSessionStartHooks('compact')
```

这里有几个预算常量：

```ts
POST_COMPACT_MAX_FILES_TO_RESTORE = 5
POST_COMPACT_TOKEN_BUDGET = 50_000
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
```

设计目标：

```txt
summary 负责历史语义
attachments 负责当前可操作状态
```

否则压缩后模型会知道“做了什么”，但不知道“当前文件内容是什么”。

---

## 11. PreCompact / PostCompact / SessionStart hooks

压缩过程中触发三类 hook：

```txt
PreCompact
  compact 前执行
  可以追加 custom instructions
  可以 block compaction

SessionStart(source='compact')
  compact 成功后执行
  重新注入 CLAUDE.md / 环境上下文

PostCompact
  summary 生成后执行
  可以显示用户消息
```

UI spinner 会显示：

```txt
Running PreCompact hooks…
Compacting conversation
Running SessionStart hooks…
Running PostCompact hooks…
```

这也解释了为什么 compaction 不是纯模型调用，而是一个小型 session lifecycle。

---

## 12. 手动 /compact

入口：

```txt
src/commands/compact/compact.ts
```

流程：

```txt
/compact [instructions]
  ↓
messages = getMessagesAfterCompactBoundary(context.messages)
  ↓
如果无 custom instructions，先试 session memory compact
  ↓
如果 reactive-only，走 compactViaReactive()
  ↓
否则先 microcompactMessages()
  ↓
compactConversation()
  ↓
setLastSummarizedMessageId(undefined)
  ↓
suppressCompactWarning()
  ↓
getUserContext.cache.clear()
  ↓
runPostCompactCleanup()
  ↓
返回 type:'compact'
```

`processSlashCommand()` 收到 `type:'compact'` 后：

```txt
buildPostCompactMessages()
shouldQuery = false
```

也就是说 `/compact` 本身不会立刻继续问模型，而是直接替换当前消息数组。

---

## 13. autocompact 阈值

文件：

```txt
src/services/compact/autoCompact.ts
```

有效上下文窗口：

```ts
getEffectiveContextWindowSize(model)
  = getContextWindowForModel(model) - reservedTokensForSummary
```

summary 输出保留：

```ts
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000
```

autocompact buffer：

```ts
AUTOCOMPACT_BUFFER_TOKENS = 13_000
```

阈值：

```ts
getAutoCompactThreshold(model)
  = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
```

warning / error / blocking：

```ts
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

如果 auto compact 关闭，则 hard blocking limit 接近：

```txt
effectiveContextWindow - 3_000
```

这是为了给用户留一点空间手动 `/compact`。

---

## 14. shouldAutoCompact

`shouldAutoCompact()` 会先过滤一些递归/特殊场景：

```txt
querySource === 'session_memory'
querySource === 'compact'
  → false

querySource === context-collapse agent
  → false

DISABLE_COMPACT / DISABLE_AUTO_COMPACT
settings.autoCompactEnabled=false
  → false

reactive-only mode
  → false

context-collapse enabled
  → false
```

然后计算：

```ts
const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
```

这里不用简单 cumulative token，也不用 last response output token。

原因在 `utils/tokens.ts`：

```txt
tokenCountWithEstimation()
  用最后一个真实 API response usage 作为基线
  加上之后新增 messages 的 rough estimate
```

它还处理 parallel tool call：

```txt
多个 assistant record 共享同一个 message.id
tool_results 可能插在这些 split assistant 中间
所以要回退到同 responseId 的第一个 sibling
再估算其后的所有新增消息
```

这避免低估 tool_result。

---

## 15. autoCompactIfNeeded

入口：

```ts
autoCompactIfNeeded(messages, toolUseContext, cacheSafeParams, querySource, tracking, snipTokensFreed)
```

流程：

```txt
1. DISABLE_COMPACT 则退出
2. consecutiveFailures >= 3 则 circuit break
3. shouldAutoCompact()
4. 构造 RecompactionInfo
5. 先试 trySessionMemoryCompaction()
6. 成功则 runPostCompactCleanup() 并返回
7. 否则 compactConversation(..., isAutoCompact=true)
8. 成功 reset lastSummarizedMessageId + cleanup
9. 失败增加 consecutiveFailures
```

失败熔断：

```ts
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

这是为了避免不可恢复的大上下文每轮都打 compaction API。

query.ts 中成功 autocompact 后：

```txt
tracking = {
  compacted: true,
  turnId: uuid(),
  turnCounter: 0,
  consecutiveFailures: 0,
}

postCompactMessages = buildPostCompactMessages(compactionResult)
yield every postCompactMessage
messagesForQuery = postCompactMessages
```

也就是说 autocompact 可以在同一轮 query 里发生，然后继续进入模型调用。

---

## 16. AutoCompactTrackingState

类型：

```ts
type AutoCompactTrackingState = {
  compacted: boolean
  turnCounter: number
  turnId: string
  consecutiveFailures?: number
}
```

用于：

```txt
判断是不是同一 query chain 内重复 compaction
记录距离上次 compact 过了几轮
失败后熔断
telemetry 诊断 compact loop
```

每次成功 compact 后 reset：

```txt
compacted = true
turnId = new uuid
turnCounter = 0
```

失败则只更新：

```txt
consecutiveFailures
```

---

## 17. blocking limit

如果自动压缩没启用或被跳过，query 还有 hard blocking check：

```ts
calculateTokenWarningState(
  tokenCountWithEstimation(messagesForQuery) - snipTokensFreed,
  model,
)
```

命中：

```txt
yield PROMPT_TOO_LONG_ERROR_MESSAGE
return { reason: 'blocking_limit' }
```

但会跳过一些情况：

```txt
刚刚 compactionResult 成功
querySource === 'compact'
querySource === 'session_memory'
reactive compact enabled and auto compact allowed
context-collapse owns overflow handling
```

原因：

```txt
compact/session_memory 自己就是为了降低 token，如果先拦截会死锁
reactive compact 需要真实 API 413 才能接管
context-collapse 也需要真实 overflow 信号做 drain/recovery
```

---

## 18. microcompact

文件：

```txt
src/services/compact/microCompact.ts
```

microcompact 的目标不是生成 summary，而是清理 tool_result。

支持两条路径：

```txt
time-based microcompact
  冷 cache 场景，直接改消息内容，把旧 tool_result 替换成占位文本

cached microcompact
  warm cache 场景，不改本地消息，通过 cache_edits 删除缓存里的 tool result
```

compactable tools：

```txt
Read
Bash / shell tools
Grep
Glob
WebSearch
WebFetch
Edit
Write
```

时间触发逻辑：

```ts
evaluateTimeBasedTrigger(messages, querySource)
```

条件：

```txt
config.enabled
querySource 是 main thread
存在 last assistant
距离 last assistant 超过 gapThresholdMinutes
```

触发后：

```txt
收集 compactable tool_use ids
保留最近 keepRecent 个
其余 tool_result.content 替换为:
"[Old tool result content cleared]"
```

cached microcompact：

```txt
注册 tool_result
计算可删除工具结果
生成 cache_edits block
pendingCacheEdits 交给 API 层插入
本地 messages 不变
API 返回后，根据 cache_deleted_input_tokens 生成 microcompact boundary message
```

这比 summary compact 更轻：

```txt
不需要 summary 模型调用
不改变本地 transcript 内容，或只清工具结果
尽量保留对话结构
```

---

## 19. tool result budget

文件：

```txt
src/utils/toolResultStorage.ts
```

query 开头先执行：

```ts
applyToolResultBudget(messagesForQuery, contentReplacementState, writeToTranscript)
```

它处理的是“单个 API user message 中聚合 tool_result 过大”。

状态：

```ts
type ContentReplacementState = {
  seenIds: Set<string>
  replacements: Map<string, string>
}
```

核心不变量：

```txt
一个 tool_use_id 第一次看到时，就冻结命运

如果被替换：
  后续永远用同一个 replacement string

如果未替换：
  后续永远不再替换
```

目的：

```txt
保持 prompt cache prefix 字节稳定
resume 后也做同样选择
```

候选分组不是按本地 message 简单分，而是按 API wire message 分：

```txt
normalizeMessagesForAPI 会合并连续 user messages
所以 budget 也必须把连续 user/tool_result 合并成同一组看
```

超预算时：

```txt
选择最大的 fresh tool_result
persistToolResult()
替换成 preview/stub
写 ContentReplacementEntry 到 transcript
```

resume 时：

```ts
reconstructContentReplacementState(messages, records)
```

把同样的 replacement string 恢复出来，避免代码版本变化导致 preview 模板不同。

---

## 20. session memory compact

文件：

```txt
src/services/compact/sessionMemoryCompact.ts
```

它是传统 summary compact 的替代路径。

触发前提：

```txt
tengu_session_memory enabled
tengu_sm_compact enabled
session memory 文件存在
session memory 不为空模板
```

配置：

```ts
DEFAULT_SM_COMPACT_CONFIG = {
  minTokens: 10_000,
  minTextBlockMessages: 5,
  maxTokens: 40_000,
}
```

核心思想：

```txt
旧历史已经被 session memory 抽取过
compact 时不再调用 summarizer
直接把 session memory 作为 compact summary
同时保留一段最近 messages
```

决定保留哪些消息：

```ts
calculateMessagesToKeepIndex(messages, lastSummarizedIndex)
```

规则：

```txt
从 lastSummarizedMessageId 之后开始
向前扩展，直到满足:
  minTokens
  minTextBlockMessages
或达到 maxTokens

不得跨越最近 compact boundary 的 floor
不得切断 tool_use/tool_result pair
不得丢失同 message.id 的 thinking fragments
```

如果是 resumed session，没有 lastSummarizedMessageId：

```txt
认为无法定位边界
从 no messages kept 开始，靠 minTokens/minTextBlockMessages 往回扩
```

生成结果时：

```txt
boundary
summaryMessages = session memory content
messagesToKeep = 最近保留片段
hookResults = SessionStart('compact')
attachments = plan attachment
```

如果 postCompactTokenCount 仍超过 autocompactThreshold：

```txt
返回 null
回退传统 compact
```

---

## 21. preserved segment

当 compaction 保留了 `messagesToKeep`，它们通常已经在 transcript 里写过，`recordTranscript()` 会 dedup 跳过。

这会导致磁盘 parentUuid 仍是旧链：

```txt
old A → old B → keep[0] → keep[1]
```

但新逻辑希望是：

```txt
boundary → summary → keep[0] → keep[1]
```

所以 compact 会调用：

```ts
annotateBoundaryWithPreservedSegment(boundary, anchorUuid, messagesToKeep)
```

写入：

```ts
compactMetadata.preservedSegment = {
  headUuid: keep[0].uuid,
  anchorUuid,
  tailUuid: keep.at(-1).uuid,
}
```

恢复时 `sessionStorage.applyPreservedSegmentRelinks()` 读取这段 metadata，把链修回来。

不同 compact 类型 anchor 不同：

```txt
suffix-preserving
  anchor = last summary message

prefix-preserving partial compact
  anchor = boundary itself
```

---

## 22. partial compact

函数：

```ts
partialCompactConversation(allMessages, pivotIndex, context, cacheSafeParams, userFeedback, direction)
```

方向：

```txt
direction='from'
  总结 pivot 之后的消息，保留 earlier prefix
  prompt cache for kept earlier messages preserved

direction='up_to'
  总结 pivot 之前的消息，保留 later suffix
  summary 在 kept messages 前，prompt cache invalidated
```

它会：

```txt
分出 messagesToSummarize / messagesToKeep
执行 PreCompact hooks
生成 partial compact prompt
streamCompactSummary()
生成 post-compact attachments
执行 SessionStart hooks
创建 boundary + summaryMessages
annotateBoundaryWithPreservedSegment()
执行 PostCompact hooks
```

REPL 里消息选择器触发 partial compact 时，会先把 UI messages 投影到 active compact view：

```ts
const compactMessages = getMessagesAfterCompactBoundary(messages)
const messageIndex = compactMessages.indexOf(message)
```

如果用户选中了 snipped 或 pre-compact 消息：

```txt
That message is no longer in the active context...
```

---

## 23. reactive compact

当前源码里 `reactiveCompact` 是 feature-gated dynamic import：

```ts
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null
```

本工作区没有对应源码文件，所以只能从调用点看接口。

query API 失败后会尝试：

```txt
contextCollapse.recoverFromOverflow()
reactiveCompact.tryReactiveCompact(...)
```

manual `/compact` 在 reactive-only mode 下走：

```ts
reactiveCompactOnPromptTooLong(messages, cacheSafeParams, {
  customInstructions,
  trigger: 'manual',
})
```

可推断 reactive compact 的职责：

```txt
等真实 API prompt-too-long / media-size 错误出现
根据错误 gap 或媒体限制选择截断/压缩策略
返回 CompactionResult
让 query 用 postCompactMessages 重试
```

query 中也会 withhold prompt-too-long/media-size 错误：

```txt
先不 yield 给用户
等 recovery 判断是否能成功
```

如果 reactive compact 成功：

```txt
buildPostCompactMessages()
messagesForQuery = postCompactMessages
continue with transition reason 'reactive_compact_retry'
```

这是被动兜底层。

---

## 24. context-collapse

`contextCollapse` 也是 feature-gated dynamic import：

```ts
const contextCollapse = feature('CONTEXT_COLLAPSE')
  ? require('./services/contextCollapse/index.js')
  : null
```

本工作区没有 `src/services/contextCollapse` 源码文件，但调用点透露了设计：

```txt
applyCollapsesIfNeeded(messagesForQuery, toolUseContext, querySource)
  在 autocompact 前运行

projectView(apiView)
  /context 命令里用于展示模型实际看到的 collapsed view

restoreFromEntries()
  sessionRestore 里从 transcript 的 marble-origami entries 恢复状态

resetContextCollapse()
  post compact cleanup 时重置

recoverFromOverflow()
  API overflow 后 drain staged collapse
```

sessionStorage 里持久化：

```txt
marble-origami-commit
marble-origami-snapshot
```

query 注释说明它的模型：

```txt
collapsed view 是对 REPL full history 的 read-time projection
summary messages live in collapse store, not REPL array
projectView() 每次重放 commit log
```

这和传统 compact 的区别：

```txt
传统 compact
  直接向 REPL messages yield boundary + summary，替换上下文

context-collapse
  不 yield，运行时投影 API view
  通过 commit log 持久化折叠状态
  目标是保留更 granular 的上下文，而不是一刀切 summary
```

因为本地没有源码，这里只记录调用面，不展开内部算法。

---

## 25. context usage 命令

文件：

```txt
src/commands/context/context-noninteractive.ts
src/commands/context/context.tsx
```

`/context` 不是直接分析 raw REPL messages，而是模拟 query 的 API view：

```txt
apiView = getMessagesAfterCompactBoundary(messages)
if context-collapse:
  apiView = projectView(apiView)
microcompactMessages(apiView)
analyzeContextUsage()
```

这很重要：

```txt
/context 显示的是模型将实际看到的上下文
不是 UI 中完整 scrollback 的 token 数
```

输出里还会显示：

```txt
Autocompact buffer
Free space
MCP tools
Custom agents
Skills
System prompt sections
Context strategy: collapse(...)
```

---

## 26. token 计数

关键函数：

```txt
src/utils/tokens.ts
```

几类计数：

```ts
getTokenUsage(message)
getTokenCountFromUsage(usage)
tokenCountFromLastAPIResponse(messages)
finalContextTokensFromLastResponse(messages)
messageTokenCountFromLastAPIResponse(messages)
getCurrentUsage(messages)
tokenCountWithEstimation(messages)
```

最关键的是：

```ts
tokenCountWithEstimation()
```

它用于：

```txt
autocompact threshold
session memory threshold
blocking limit
preCompactTokenCount
```

它的原则：

```txt
用最后一个真实 API response usage 表示已知上下文大小
再粗估该 response 后新增的 messages
```

这样比“累计所有 turn token”准确，因为 context window 是当前请求大小，不是历史总消耗。

---

## 27. 1M context 和窗口大小

文件：

```txt
src/utils/context.ts
```

默认：

```ts
MODEL_CONTEXT_WINDOW_DEFAULT = 200_000
```

1M context 判断：

```txt
模型名带 [1m]
模型 capability max_input_tokens
beta header
实验开关
ant model config
```

环境变量可覆盖：

```txt
CLAUDE_CODE_DISABLE_1M_CONTEXT
CLAUDE_CODE_MAX_CONTEXT_TOKENS
CLAUDE_CODE_AUTO_COMPACT_WINDOW
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE
CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE
```

autocompact 用的是：

```txt
context window
  - reserved summary output tokens
  - autocompact buffer
```

所以模型窗口变大后，自动压缩阈值也会变大。

---

## 28. postCompactCleanup

文件：

```txt
src/services/compact/postCompactCleanup.ts
```

压缩后需要清理一堆缓存：

```ts
resetMicrocompactState()
resetContextCollapse()
getUserContext.cache.clear()
resetGetMemoryFilesCache('compact')
clearSystemPromptSections()
clearClassifierApprovals()
clearSpeculativeChecks()
clearBetaTracingState()
clearSessionMessagesCache()
```

但要注意 subagent：

```txt
agent:* querySource 和 main thread 在同进程共享模块级状态
subagent compact 不能重置 main thread 的 context-collapse / memory cache
```

所以：

```ts
const isMainThreadCompact =
  querySource === undefined ||
  querySource.startsWith('repl_main_thread') ||
  querySource === 'sdk'
```

只有 main thread compact 才清主线程全局缓存。

另一个重要选择：

```txt
不 reset invoked skill content
不 reset sentSkillNames
```

原因：

```txt
已调用 skill 的内容需要跨多次 compaction 保留
完整 skill_listing 每次重注入成本太高
```

---

## 29. 与 sessionStorage 的关系

compaction 对 transcript 的影响：

```txt
1. compact boundary 被写成 system transcript message
2. summary 被写成 isCompactSummary user message
3. messagesToKeep 可能 dedup 跳过，但 preservedSegment metadata 修复恢复链
4. metadata 会 reAppend 到尾部，保证 /resume lite scan 可见
5. sessionStorage loader 会在 compact boundary 处截断 pre-compact 历史
6. 大文件读取会直接跳过 pre-compact bytes
```

这就是为什么 compaction 不是纯内存操作。

它必须同时更新：

```txt
REPL messages
query API view
transcript parentUuid chain
/resume metadata
prompt cache baseline
context-analysis view
```

---

## 30. 与 prompt cache 的关系

上下文管理里有多处 prompt cache 保护：

```txt
compact summary 优先用 forked agent 共享 cache prefix
tool-result budget 记录 exact replacement string，保持字节一致
cached microcompact 用 cache_edits，不改本地 messages
session memory compact 避免 compaction API call
post compact notifyCompaction() 重置 cache break baseline
time-based microcompact notifyCacheDeletion()
```

如果不做这些，压缩虽然省上下文，但会造成大量 cache miss。

所以系统目标不是简单“少 token”，而是：

```txt
少上下文
少 cache_creation
恢复后字节稳定
不误报 prompt-cache break
```

---

## 31. 端到端：自动压缩

```txt
query()
  ↓
messagesForQuery = after compact boundary
  ↓
applyToolResultBudget()
  ↓
snipCompactIfNeeded()
  ↓
microcompactMessages()
  ↓
contextCollapse.applyCollapsesIfNeeded()
  ↓
autoCompactIfNeeded()
  ↓
shouldAutoCompact(tokenCountWithEstimation - snipFreed)
  ↓
trySessionMemoryCompaction()
  ↓ fallback
compactConversation(isAutoCompact=true)
  ↓
buildPostCompactMessages()
  ↓
yield boundary / summary / attachments / hooks
  ↓
messagesForQuery = postCompactMessages
  ↓
callModel()
```

---

## 32. 端到端：手动 /compact

```txt
用户输入 /compact [instructions]
  ↓
processSlashCommand()
  ↓
commands/compact.call()
  ↓
messages = getMessagesAfterCompactBoundary()
  ↓
trySessionMemoryCompaction()
  ↓ fallback
microcompactMessages()
  ↓
compactConversation(isAutoCompact=false)
  ↓
返回 { type:'compact', compactionResult }
  ↓
processSlashCommand()
  ↓
append slash-command synthetic messages into messagesToKeep
  ↓
buildPostCompactMessages()
  ↓
setMessages()
  ↓
shouldQuery=false
```

---

## 33. 端到端：恢复 post-compact session

```txt
/resume
  ↓
loadTranscriptFile()
  ↓
大文件可跳过 pre-compact bytes
  ↓
parse compact boundary + summary + preservedSegment
  ↓
applyPreservedSegmentRelinks()
  ↓
leaf parentUuid walk stops at boundary
  ↓
deserializeMessages()
  ↓
REPL initialMessages = post-compact chain
  ↓
useLogMessages ignore initial-only resume
```

关键点：

```txt
summary 替代旧历史
transcript 保留旧历史文件
loader 不把旧历史恢复进 active context
```

---

## 34. 关键设计结论

1. Claude Code 的 context management 是 pipeline，不是单个 `/compact`。

2. query 每轮都先构造 API view，UI messages 和模型上下文是分离的。

3. compact boundary 是运行时、UI、transcript 恢复共同依赖的分界线。

4. traditional compact 用 summary message 替代旧历史，并补回文件、plan、skill、agent、MCP、hook 上下文。

5. autocompact 的阈值基于 context window，而不是累计 token 花费。

6. `tokenCountWithEstimation()` 是阈值判断的核心，专门处理 streaming split assistant 和 interleaved tool_result。

7. microcompact 优先于 autocompact，因为它更便宜、更细粒度。

8. tool-result budget 通过 exact replacement record 维持 resume 和 prompt cache 稳定。

9. session-memory compact 避免重新总结旧历史，用已抽取 memory 作为 summary。

10. post-compact cleanup 必须区分 main thread 和 subagent，否则共享模块状态会被误清。

11. compaction 与 sessionStorage 强耦合：压缩结果必须能被 `/resume` 按同一条 parentUuid 链恢复。

---

## 35. 下一步

到这里，Claude Code 的核心运行链路已经覆盖：

```txt
启动
REPL
query
Tool
Permission
Hooks
MCP
Agent/Subagent
Session/Transcript
Compaction/Context
```

下一步适合继续拆：

```txt
第十一步：Message / Streaming / Tool Execution 细节
```

原因是 query 主循环里还有一块最核心但最复杂的逻辑：

```txt
streaming assistant content_block
tool_use 检测
StreamingToolExecutor
tool_result 归并
缺失 tool_result 修复
fallback/tombstone
max output tokens retry
stop hooks
```

这会把“模型流式输出如何变成工具调用和下一轮输入”完整补上。
