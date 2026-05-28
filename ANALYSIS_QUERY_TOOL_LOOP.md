# Claude Code 源码解剖：Query 主循环与工具执行

## 1. 总览

前一步已经追到：

```txt
REPL.onQuery()
  ↓
src/query.ts
```

第三步继续剖开 `query()` 内部。核心链路如下：

```txt
src/query.ts
  ↓ queryLoop()
  ↓ deps.callModel() = queryModelWithStreaming()
src/services/api/claude.ts
  ↓ Anthropic streaming API
  ↓ assistant message / tool_use blocks
src/query.ts
  ↓ runTools() or StreamingToolExecutor
src/services/tools/toolOrchestration.ts
src/services/tools/toolExecution.ts
  ↓ tool.call()
  ↓ tool_result user message
src/query.ts
  ↓ append assistant + tool_result
  ↓ next loop iteration
```

这就是 Claude Code 的 agentic loop：

```txt
用户消息
  → 模型生成
  → 如果没有 tool_use：结束本轮
  → 如果有 tool_use：执行工具
  → 把 tool_result 作为 user message 塞回上下文
  → 再次请求模型
```

---

## 2. `query()` 只是外壳，真正逻辑在 `queryLoop()`

文件：

```txt
src/query.ts
```

导出的 `query()` 是一个 async generator：

```ts
export async function* query(params: QueryParams) {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

观察：

- 它不是普通 `async function`，而是 `async function*`
- 所有 UI 消息、系统消息、工具结果、流事件都会通过 `yield` 往外吐
- `REPL` 或 headless print 模式负责消费这些 yielded messages
- 真正主循环在内部 `queryLoop()`

`queryLoop()` 维护一个跨迭代 `State`：

```ts
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

这里的 `turnCount` 不是用户输入次数，而是同一个用户请求里，模型和工具循环了多少次。

---

## 3. 每次循环开始：先整理上下文

`queryLoop()` 是一个无限循环：

```ts
while (true) {
  ...
}
```

每次进入循环，它不会直接调用模型，而是先做上下文整理：

```txt
messages
  ↓ getMessagesAfterCompactBoundary()
  ↓ applyToolResultBudget()
  ↓ snip compact
  ↓ microcompact
  ↓ context collapse
  ↓ autocompact
  ↓ append system/user context
```

这说明 Claude Code 的主循环并不是简单的：

```txt
messages → API
```

而是：

```txt
完整历史
  → 预算裁剪
  → 压缩/折叠
  → 工具结果瘦身
  → 生成本次 API 可承受的 messagesForQuery
```

关键变量：

```ts
let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]
```

之后所有 API 调用都基于 `messagesForQuery`，而不是原始 `messages`。

---

## 4. 调模型：`deps.callModel()`

`query.ts` 不直接 import API 函数，而是通过依赖注入：

```ts
const deps = params.deps ?? productionDeps()
```

生产依赖在：

```txt
src/query/deps.ts
```

```ts
export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}
```

所以主模型调用是：

```ts
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    fallbackModel,
    querySource,
    agents,
    mcpTools,
    taskBudget,
    ...
  },
})) {
  ...
}
```

重点：

- `callModel()` 也是 async generator
- 模型流式返回时，`query.ts` 一边消费一边 `yield` 给 UI
- 如果流里出现 `tool_use` block，会被收集到 `toolUseBlocks`
- `stop_reason === 'tool_use'` 不被信任，代码直接看内容里有没有 `tool_use`

关键逻辑：

```ts
if (message.type === 'assistant') {
  assistantMessages.push(message)

  const msgToolUseBlocks = message.message.content.filter(
    content => content.type === 'tool_use',
  ) as ToolUseBlock[]

  if (msgToolUseBlocks.length > 0) {
    toolUseBlocks.push(...msgToolUseBlocks)
    needsFollowUp = true
  }
}
```

`needsFollowUp` 是本轮是否要执行工具并继续模型调用的核心标志。

---

## 5. API 层：`queryModelWithStreaming()`

文件：

```txt
src/services/api/claude.ts
```

入口：

```ts
export async function* queryModelWithStreaming(...) {
  return yield* withStreamingVCR(messages, async function* () {
    yield* queryModel(...)
  })
}
```

真正请求发生在 `queryModel()` 中：

```ts
const result = await anthropic.beta.messages
  .create(
    { ...params, stream: true },
    { signal, headers }
  )
  .withResponse()
```

这里做了大量 API 前处理：

```txt
tools
  → dynamic tool search / deferred tools
  → toolToAPISchema()
  → MCP / agent / advisor tool schema

messages
  → normalizeMessagesForAPI()
  → ensureToolResultPairing()
  → strip unsupported blocks
  → strip excess media

system prompt
  → attribution header
  → CLI sysprompt prefix
  → prompt cache blocks
```

流式事件处理的核心是把 Anthropic raw stream 组装回内部 `AssistantMessage`：

```txt
message_start
content_block_start
content_block_delta
content_block_stop
message_delta
```

在 `content_block_stop` 时，代码会创建并 yield 一个 assistant message：

```txt
partialMessage + contentBlocks
  → AssistantMessage
  → yield
```

所以 `query.ts` 看到的不是最原始 SSE chunk，而是已经组装好的 `AssistantMessage` 或系统/API 错误消息。

---

## 6. 没有工具：进入 Stop hooks，然后结束

模型流结束后，如果没有工具：

```ts
if (!needsFollowUp) {
  ...
  const stopHookResult = yield* handleStopHooks(...)
  ...
  return { reason: 'completed' }
}
```

这一步不是立即结束，而是先处理：

```txt
prompt-too-long / media error recovery
max_output_tokens recovery
API error early return
Stop hooks
token budget auto-continuation
```

Stop hooks 在：

```txt
src/query/stopHooks.ts
```

如果 hook 返回 blocking error，`query.ts` 会把错误作为 meta user message 放回上下文，然后 `continue` 重新问模型：

```ts
state = {
  messages: [
    ...messagesForQuery,
    ...assistantMessages,
    ...stopHookResult.blockingErrors,
  ],
  stopHookActive: true,
  transition: { reason: 'stop_hook_blocking' },
}
continue
```

也就是说 Stop hook 可以把 “模型已经结束” 改写成 “继续一轮，让模型修正”。

---

## 7. 有工具：进入工具执行层

如果模型产生了 `tool_use`，`query.ts` 会执行工具：

```ts
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message
    toolResults.push(...normalizeMessagesForAPI([update.message], tools))
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

这里有两种模式：

```txt
传统模式:
  模型流结束后
  → runTools()

StreamingToolExecutor 模式:
  模型流式吐出 tool_use 时
  → 立刻 addTool()
  → 工具可能和模型剩余输出并行执行
```

Streaming 模式的目的很明确：减少等待时间。只要模型已经给出了完整 tool input，就可以先跑工具，不必等整段 assistant response 完成。

---

## 8. 工具编排：串行与并行

文件：

```txt
src/services/tools/toolOrchestration.ts
```

传统工具执行入口：

```ts
export async function* runTools(
  toolUseMessages,
  assistantMessages,
  canUseTool,
  toolUseContext,
)
```

它先分批：

```ts
partitionToolCalls(toolUseMessages, currentContext)
```

分批规则：

```txt
连续 concurrency-safe 工具
  → 并行执行

非 concurrency-safe 工具
  → 单独串行执行
```

判断依据来自每个工具自己的：

```ts
tool.isConcurrencySafe(parsedInput.data)
```

默认并发上限：

```ts
CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || 10
```

这解释了为什么多个 `Read` / `Grep` / `Glob` 类工具可以同时跑，而编辑文件、执行 shell 等会更保守。

---

## 9. 单个工具执行：`runToolUse()`

文件：

```txt
src/services/tools/toolExecution.ts
```

单个工具的完整生命周期：

```txt
find tool
  ↓
inputSchema.safeParse()
  ↓
tool.validateInput()
  ↓
PreToolUse hooks
  ↓
permission decision / canUseTool()
  ↓
tool.call()
  ↓
mapToolResultToToolResultBlockParam()
  ↓
PostToolUse hooks
  ↓
createUserMessage(tool_result)
```

入口：

```ts
export async function* runToolUse(
  toolUse,
  assistantMessage,
  canUseTool,
  toolUseContext,
)
```

如果模型调用了不存在的工具，会返回一个 `tool_result` 错误，而不是直接抛异常：

```ts
createUserMessage({
  content: [{
    type: 'tool_result',
    content: `<tool_use_error>Error: No such tool available: ${toolName}</tool_use_error>`,
    is_error: true,
    tool_use_id: toolUse.id,
  }],
})
```

这很重要：工具错误也会作为模型可见的 `tool_result` 回到上下文，让模型自己恢复。

---

## 10. 权限系统卡在工具调用之前

工具真正执行前，会经过：

```ts
resolveHookPermissionDecision(
  hookPermissionResult,
  tool,
  processedInput,
  toolUseContext,
  canUseTool,
  assistantMessage,
  toolUseID,
)
```

如果结果不是 allow：

```ts
if (permissionDecision.behavior !== 'allow') {
  resultingMessages.push({
    message: createUserMessage({
      content: [{
        type: 'tool_result',
        content: errorMessage,
        is_error: true,
        tool_use_id: toolUseID,
      }],
    }),
  })
  return resultingMessages
}
```

所以权限拒绝并不是 “本地终止”，而是转换成：

```txt
tool_result(is_error=true)
```

然后交还给模型。这就是 Claude Code 可以在用户拒绝某个操作后继续解释、换方案或请求别的权限的原因。

---

## 11. 真正执行：`tool.call()`

权限通过后，才进入真实工具：

```ts
const result = await tool.call(
  callInput,
  {
    ...toolUseContext,
    toolUseId: toolUseID,
    userModified: permissionDecision.userModified ?? false,
  },
  canUseTool,
  assistantMessage,
  progress => { ... },
)
```

工具返回后，统一映射为 Anthropic API 的 tool result block：

```ts
const mappedToolResultBlock =
  tool.mapToolResultToToolResultBlockParam(result.data, toolUseID)
```

然后再包成内部 user message：

```ts
createUserMessage({
  content: [toolResultBlock, ...optionalFeedback],
  toolUseResult,
  sourceToolAssistantUUID: assistantMessage.uuid,
})
```

注意：Claude Code 内部把 tool result 表示为 `UserMessage`。这和 Anthropic API 协议一致：工具结果是下一轮请求里由 user role 提供的 `tool_result`。

---

## 12. 工具执行后：拼上下文并继续循环

工具执行完成后，`query.ts` 会收集：

```txt
messagesForQuery
assistantMessages
toolResults
attachments
memory attachments
skill attachments
queued command attachments
```

然后构造下一轮状态：

```ts
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  turnCount: nextTurnCount,
  pendingToolUseSummary: nextPendingToolUseSummary,
  transition: { reason: 'next_turn' },
}
```

循环重新开始：

```txt
tool_result 已经在 messages 里
  ↓
重新做 compact / budget / attachment
  ↓
再次 callModel()
```

如果设置了 `maxTurns`，这里会截断：

```ts
if (maxTurns && nextTurnCount > maxTurns) {
  yield createAttachmentMessage({ type: 'max_turns_reached', ... })
  return { reason: 'max_turns' }
}
```

---

## 13. StreamingToolExecutor 的关键设计

文件：

```txt
src/services/tools/StreamingToolExecutor.ts
```

它解决的问题：

```txt
模型还在流式输出
但某些 tool_use 已经完整出现
是否可以提前执行？
```

设计：

```txt
addTool(block)
  ↓
判断工具是否 concurrency-safe
  ↓
能跑就 executeTool()
  ↓
结果先 buffer
  ↓
getCompletedResults() 非阻塞吐出已完成结果
  ↓
getRemainingResults() 在模型结束后收尾
```

它还处理了并行工具失败：

- 只有 Bash 错误会取消 sibling tools
- 用户中断会按工具的 `interruptBehavior()` 决定是取消还是阻塞
- streaming fallback 时会 discard 已经开始但属于旧响应的工具结果

这说明工具执行并不是一个简单的 `Promise.all`，而是一个带顺序约束、取消传播、结果缓冲的调度器。

---

## 14. 这一层的关键抽象

### `AssistantMessage`

模型输出。可能包含：

```txt
text
thinking
tool_use
server tool blocks
api error synthetic message
```

### `UserMessage`

用户输入，也包括工具结果：

```txt
普通 user text
tool_result
meta recovery instruction
permission denial result
```

### `ToolUseBlock`

模型要求执行工具：

```ts
{
  type: 'tool_use',
  id,
  name,
  input,
}
```

### `ToolUseContext`

工具执行上下文，包含：

```txt
abortController
app state getter/setter
permission context
MCP clients/resources
tool list
agent id
query tracking
UI callbacks
```

这是工具层能访问 REPL 状态、权限状态、MCP 状态和中断状态的核心对象。

---

## 15. 本步结论

Claude Code 的核心不是 “调用一次 Claude API”，而是一个 async-generator 驱动的状态机：

```txt
queryLoop state
  → context preparation
  → streaming model call
  → collect assistant/tool_use
  → execute tools with permissions/hooks
  → convert tool output into user tool_result
  → append results
  → continue until no tool_use
```

最重要的三个设计点：

1. **所有中间产物都通过 `yield` 流出**  
   UI、headless、SDK 可以边生成边消费。

2. **工具错误和权限拒绝都回写为 `tool_result`**  
   模型能看到失败原因，并继续恢复。

3. **上下文不是原样累积，而是每轮都重建可发送视图**  
   compact、tool result budget、media stripping、tool schema filtering 都发生在 API 调用前。

下一步如果继续解剖，建议进入第四步：

```txt
Tool 抽象与内置工具体系
  → src/Tool.ts
  → src/tools.ts
  → BashTool / FileReadTool / FileEditTool
  → MCP tool 如何被包装成 Tool
```
