# Claude Code 源码解剖：REPL 输入到 Query 主循环

## 1. 总览

用户输入从 REPL 到模型主循环的核心链路如下：

```txt
PromptInput
  ↓ onSubmit
src/screens/REPL.tsx
  ↓ handlePromptSubmit()
src/utils/handlePromptSubmit.ts
  ↓ processUserInput()
src/utils/processUserInput/processUserInput.ts
  ↓ onQuery()
src/screens/REPL.tsx
  ↓ query()
src/query.ts
  ↓ callModel()
  ↓ runTools()
  ↓ loop until no tool_use
```

---

## 2. 用户输入入口：`PromptInput → onSubmit`

在 `src/screens/REPL.tsx` 末尾，`PromptInput` 接收：

```tsx
<PromptInput
  ...
  onSubmit={onSubmit}
/>
```

所以用户按 Enter 后，进入 `REPL.tsx` 里的：

```ts
const onSubmit = useCallback(async (input, helpers, speculationAccept, options) => {
  ...
})
```

`onSubmit` 主要做：

1. 滚动到底部
2. 恢复 proactive loop
3. 处理 immediate slash command
4. 保存历史
5. 清空输入框
6. remote mode 分流
7. 等待 SessionStart hooks
8. 调用 `handlePromptSubmit(...)`

关键调用：

```ts
await handlePromptSubmit({
  input,
  helpers,
  queryGuard,
  isExternalLoading,
  mode: inputMode,
  commands,
  getToolUseContext,
  messages: messagesRef.current,
  mainLoopModel,
  pastedContents,
  ideSelection,
  onQuery,
  ...
})
```

---

## 3. 中间调度：`handlePromptSubmit.ts`

文件：

```txt
src/utils/handlePromptSubmit.ts
```

`handlePromptSubmit` 是用户输入和 query 之间的调度器。

### A. 如果当前已有 query 正在跑，进入队列

```ts
if (queryGuard.isActive || isExternalLoading) {
  enqueue({
    value: finalInput.trim(),
    mode,
    pastedContents,
    skipSlashCommands,
    uuid,
  })

  clearBuffer()
  return
}
```

也就是说，Claude 正在执行工具或生成时，用户继续输入不会立即打断主流程，而是进入 `messageQueueManager`。

### B. 如果当前空闲，直接执行

```ts
await executeUserInput({
  queuedCommands: [cmd],
  messages,
  mainLoopModel,
  queryGuard,
  getToolUseContext,
  onQuery,
  ...
})
```

---

## 4. 输入预处理：`processUserInput`

文件：

```txt
src/utils/processUserInput/processUserInput.ts
```

`executeUserInput()` 会调用：

```ts
const result = await processUserInput({
  input: cmd.value,
  mode: cmd.mode,
  context: makeContext(),
  pastedContents,
  messages,
  querySource,
  ...
})
```

`processUserInput` 负责把用户输入变成内部消息。

主要分支：

```txt
普通文本
  → processTextPrompt()
  → createUserMessage()
  → shouldQuery = true

/bash 模式
  → processBashCommand()

/slash command
  → processSlashCommand()

图片/粘贴内容
  → resize / storeImages / attachmentMessages

hooks
  → UserPromptSubmit hooks
```

普通文本最终走：

```txt
src/utils/processUserInput/processTextPrompt.ts
```

它会创建：

```ts
const userMessage = createUserMessage({
  content: input,
  uuid,
  permissionMode,
})
```

然后返回：

```ts
return {
  messages: [userMessage, ...attachmentMessages],
  shouldQuery: true,
}
```

---

## 5. 回到 REPL：`onQuery`

`handlePromptSubmit` 处理完输入后，会调用 REPL 传入的：

```ts
onQuery(
  newMessages,
  abortController,
  shouldQuery,
  allowedTools,
  model,
  ...
)
```

`REPL.tsx` 中的 `onQuery` 会：

1. 用 `queryGuard.tryStart()` 防止并发 query
2. 把新用户消息加入 UI 消息列表
3. 调用 `onQueryImpl(...)`

关键：

```ts
setMessages(oldMessages => [...oldMessages, ...newMessages])

await onQueryImpl(
  latestMessages,
  newMessages,
  abortController,
  shouldQuery,
  additionalAllowedTools,
  mainLoopModelParam,
  effort,
)
```

---

## 6. 构造 query 上下文：`getToolUseContext`

在 `REPL.tsx` 中有一个非常关键的函数：

```ts
const getToolUseContext = useCallback((messages, newMessages, abortController, mainLoopModel) => {
  const s = store.getState()

  const computeTools = () => {
    const state = store.getState()
    const assembled = assembleToolPool(state.toolPermissionContext, state.mcp.tools)
    const merged = mergeAndFilterTools(...)
    return merged
  }

  return {
    abortController,
    options: {
      commands,
      tools: computeTools(),
      mainLoopModel,
      mcpClients,
      mcpResources,
      thinkingConfig,
      ...
    },
    getAppState: () => store.getState(),
    setAppState,
    messages,
    setMessages,
    readFileState,
    ...
  }
})
```

这个 `toolUseContext` 是后面工具执行、权限判断、MCP、状态更新的核心上下文对象。

---

## 7. 进入核心模型循环：`query.ts`

`onQueryImpl` 中真正进入模型主循环：

```ts
for await (const event of query({
  messages: messagesIncludingNewMessages,
  systemPrompt,
  userContext,
  systemContext,
  canUseTool,
  toolUseContext,
  querySource: getQuerySourceForREPL()
})) {
  onQueryEvent(event)
}
```

这里进入：

```txt
src/query.ts
```

---

## 8. `query.ts` 主循环

`query.ts` 核心是：

```ts
export async function* query(params: QueryParams) {
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  ...
}
```

真正逻辑在：

```ts
async function* queryLoop(...)
```

它是一个 `while (true)` 循环：

```ts
while (true) {
  // 准备 messagesForQuery
  // microcompact / autocompact / context collapse
  // callModel()
  // 收集 tool_use
  // runTools()
  // 拼 tool_result
  // 如果还有后续，continue 下一轮
}
```

核心阶段：

### A. 准备上下文

```ts
let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]
```

然后做：

- microcompact
- context collapse
- autocompact
- token limit 检查
- system prompt 拼接

### B. 调模型

```ts
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    mcpTools: appState.mcp.tools,
    agents: toolUseContext.options.agentDefinitions.activeAgents,
    ...
  },
})) {
  yield message

  if (message.type === 'assistant') {
    // 收集 tool_use blocks
  }
}
```

### C. 如果模型没有 tool use，则结束

```ts
if (!needsFollowUp) {
  return { reason: 'completed' }
}
```

### D. 如果有 tool use，执行工具

```ts
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

工具结果会转成 `tool_result`，再进入下一轮模型调用。

---

## 9. 工具执行：`runTools`

文件：

```txt
src/services/tools/toolOrchestration.ts
```

核心：

```ts
export async function* runTools(
  toolUseMessages,
  assistantMessages,
  canUseTool,
  toolUseContext,
)
```

它会先分批：

```ts
partitionToolCalls(...)
```

规则：

```txt
并发安全的工具 → 并发执行
非并发安全工具 → 串行执行
```

最大并发：

```ts
CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || 10
```

实际单个工具执行在：

```txt
src/services/tools/toolExecution.ts
```

---

## 10. UI 如何收到流式结果

`query()` 是 async generator，每 yield 一个事件，REPL 都会处理：

```ts
for await (const event of query(...)) {
  onQueryEvent(event)
}
```

`onQueryEvent` 调用：

```ts
handleMessageFromStream(event, newMessage => {
  setMessages(oldMessages => [...oldMessages, newMessage])
})
```

所以 UI 更新链路是：

```txt
query.ts yield event
  ↓
REPL.onQueryEvent()
  ↓
handleMessageFromStream()
  ↓
setMessages()
  ↓
Messages 组件重新渲染
```

---

## 11. 第二层总结

完整用户输入链路：

```txt
用户按 Enter
  ↓
PromptInput.onSubmit
  ↓
REPL.onSubmit
  ↓
handlePromptSubmit
  ├─ 如果 query 正在运行：enqueue()
  └─ 如果空闲：executeUserInput()
        ↓
      processUserInput()
        ├─ 普通文本 → processTextPrompt()
        ├─ /命令 → processSlashCommand()
        ├─ bash模式 → processBashCommand()
        └─ 图片/附件/hooks
        ↓
      onQuery()
        ↓
      onQueryImpl()
        ├─ 构造 toolUseContext
        ├─ 加载 systemPrompt/userContext/systemContext
        └─ query()
              ↓
            callModel()
              ↓
            收集 tool_use
              ↓
            runTools()
              ↓
            tool_result 回灌
              ↓
            下一轮 callModel()
              ↓
            直到 completed / aborted / max_turns
```

---

## 12. 下一步

继续解剖 **`query.ts` 内部模型调用与工具循环**，也就是 Claude Code 的真正 agent loop。
