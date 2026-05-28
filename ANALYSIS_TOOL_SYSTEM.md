# Claude Code 源码解剖：Tool 抽象与工具体系

## 1. 总览

第三步拆到 `query.ts` 执行工具：

```txt
tool_use
  ↓
runTools() / StreamingToolExecutor
  ↓
runToolUse()
  ↓
tool.call()
```

第四步继续看 `tool.call()` 背后的工具系统。核心链路如下：

```txt
src/Tool.ts
  ↓ 定义 Tool contract + ToolUseContext + buildTool()
src/tools.ts
  ↓ 注册/过滤/组装内置工具
src/services/mcp/client.ts
  ↓ 把 MCP tools 包装成 Tool
src/utils/api.ts
  ↓ toolToAPISchema()
src/services/api/claude.ts
  ↓ 把工具 schema 发送给 Anthropic API
```

一句话：Claude Code 把 Bash、Read、Edit、Agent、MCP tools 都压成同一种 `Tool` 对象，然后主循环只认 `Tool` 接口。

---

## 2. 核心接口：`Tool`

文件：

```txt
src/Tool.ts
```

`Tool` 是整个工具体系的中心抽象：

```ts
export type Tool<Input, Output, P> = {
  name: string
  inputSchema: Input
  inputJSONSchema?: ToolInputJSONSchema
  outputSchema?: z.ZodType<unknown>

  prompt(...): Promise<string>
  description(input, options): Promise<string>

  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>

  call(input, context, canUseTool, parentMessage, onProgress?)
    : Promise<ToolResult<Output>>

  mapToolResultToToolResultBlockParam(output, toolUseID)
    : ToolResultBlockParam

  renderToolUseMessage(...)
  renderToolResultMessage?(...)
  renderToolUseProgressMessage?(...)
  ...
}
```

可以把它分成四类能力：

```txt
给模型看的：
  name
  prompt()
  inputSchema / inputJSONSchema
  strict
  shouldDefer / alwaysLoad

执行前判断：
  validateInput()
  checkPermissions()
  isReadOnly()
  isConcurrencySafe()
  isDestructive()
  preparePermissionMatcher()

真正执行：
  call()
  interruptBehavior()
  contextModifier

给 UI 和 transcript 看的：
  renderToolUseMessage()
  renderToolResultMessage()
  renderToolUseProgressMessage()
  getToolUseSummary()
  userFacingName()
```

这解释了为什么工具代码看起来“胖”：它同时要服务模型 schema、权限系统、真实执行、UI 展示、日志和 transcript 搜索。

---

## 3. `ToolUseContext` 是工具运行时的环境包

同样在：

```txt
src/Tool.ts
```

`ToolUseContext` 是工具执行时拿到的上下文：

```ts
export type ToolUseContext = {
  options: {
    commands: Command[]
    mainLoopModel: string
    tools: Tools
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    agentDefinitions: AgentDefinitionsResult
    refreshTools?: () => Tools
    ...
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f): void
  setToolJSX?: SetToolJSXFn
  addNotification?: (...)
  setInProgressToolUseIDs(...)
  updateFileHistoryState(...)
  updateAttributionState(...)
  agentId?: AgentId
  messages: Message[]
  ...
}
```

这个对象让工具能做几件关键事：

- 读写全局 AppState
- 知道当前权限模式、MCP 连接、agent 身份
- 响应用户中断
- 更新 UI 状态和 progress
- 访问 `readFileState`，做文件读写安全检查
- 在子 agent 中继承父上下文

所以工具不是纯函数。它更像一个带上下文的运行单元。

---

## 4. `buildTool()` 给工具补默认行为

文件：

```txt
src/Tool.ts
```

多数工具都这样导出：

```ts
export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,
  ...
})
```

`buildTool()` 的作用是给常见方法补默认值：

```ts
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,
  isReadOnly: () => false,
  isDestructive: () => false,
  checkPermissions: input =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',
  userFacingName: () => '',
}
```

关键设计：

- 并发默认是 `false`，保守
- 只读默认是 `false`，保守
- destructive 默认是 `false`
- 权限默认 allow，但通用权限系统仍会在外层参与
- classifier 默认空，安全相关工具必须显式实现

也就是说，工具作者只需要实现核心差异，框架补齐统一 contract。

---

## 5. 内置工具注册：`tools.ts`

文件：

```txt
src/tools.ts
```

`getAllBaseTools()` 是内置工具全集的源头：

```ts
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    GlobTool,
    GrepTool,
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    ...
    ListMcpResourcesTool,
    ReadMcpResourceTool,
    ToolSearchTool,
  ]
}
```

它不是简单静态数组，而是按环境和 feature gate 条件组装：

```txt
USER_TYPE === ant
feature('PROACTIVE')
feature('AGENT_TRIGGERS')
ENABLE_LSP_TOOL
CLAUDE_CODE_SIMPLE
isWorktreeModeEnabled()
isAgentSwarmsEnabled()
isToolSearchEnabledOptimistic()
...
```

这说明 Claude Code 的工具池是运行时变体，不同构建、环境变量、账号类型、实验开关会得到不同工具。

---

## 6. 可用工具过滤：权限和模式先裁剪工具池

`getTools(permissionContext)` 会从全集里筛可用工具。

特殊 simple mode：

```ts
if (CLAUDE_CODE_SIMPLE) {
  return [BashTool, FileReadTool, FileEditTool]
}
```

普通模式：

```ts
const tools = getAllBaseTools().filter(tool => !specialTools.has(tool.name))
let allowedTools = filterToolsByDenyRules(tools, permissionContext)
...
return allowedTools.filter(tool => tool.isEnabled())
```

这里有一个重要点：

```ts
filterToolsByDenyRules()
```

如果某个工具被 blanket deny，它会在发给模型之前就被移除，而不是等模型调用时再拒绝。

所以权限系统有两层：

```txt
请求前：
  工具池过滤，不让模型看到某些工具

调用时：
  runToolUse() → checkPermissions / canUseTool
```

---

## 7. 内置工具 + MCP 工具合并

`assembleToolPool()` 是 REPL 和 agent 使用的统一组装函数：

```ts
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

几个细节很关键：

- built-in tools 和 MCP tools 都按 name 排序，保持 prompt cache 稳定
- built-in tools 放在前面
- `uniqBy(..., 'name')` 让内置工具在同名冲突时优先
- MCP tools 也会被 deny rules 过滤

也就是说，工具池不是“谁连接上就追加谁”，而是为了缓存稳定和冲突规则专门排序合并。

---

## 8. 代表工具一：`FileReadTool`

文件：

```txt
src/tools/FileReadTool/FileReadTool.ts
```

它是典型只读工具：

```ts
export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,
  maxResultSizeChars: Infinity,
  strict: true,

  isConcurrencySafe() {
    return true
  },
  isReadOnly() {
    return true
  },
  getPath({ file_path }) {
    return file_path || getCwd()
  },
  checkPermissions(input, context) {
    return checkReadPermissionForTool(...)
  },
  async call(...) {
    ...
  },
  mapToolResultToToolResultBlockParam(data, toolUseID) {
    ...
  },
})
```

重点行为：

- `isConcurrencySafe = true`，多个 Read 可以并行
- `isReadOnly = true`，权限上更容易自动放行
- `maxResultSizeChars = Infinity`，Read 结果不走外部持久化，因为让模型再 Read 持久化文件会形成循环
- `validateInput()` 做路径 deny、二进制文件、PDF 页范围、设备文件等检查
- `call()` 会更新 `readFileState`，后续 Edit/Write 用它判断文件是否先读过、是否被外部改动
- `mapToolResultToToolResultBlockParam()` 按 text/image/pdf/notebook 等输出不同 API block

Read 工具不只是读文件，它还建立了文件安全状态。

---

## 9. 代表工具二：`FileEditTool`

文件：

```txt
src/tools/FileEditTool/FileEditTool.ts
```

它是典型写工具：

```ts
export const FileEditTool = buildTool({
  name: FILE_EDIT_TOOL_NAME,
  maxResultSizeChars: 100_000,
  strict: true,
  checkPermissions(input, context) {
    return checkWritePermissionForTool(...)
  },
  async validateInput(input, toolUseContext) {
    ...
  },
  async call(input, context, _, parentMessage) {
    ...
  },
})
```

`validateInput()` 做大量写前安全检查：

```txt
old_string === new_string
文件是否存在
是否是 notebook
是否先 Read 过
文件是否在 Read 后被外部修改
old_string 是否唯一匹配
settings 文件编辑是否合法
team memory 是否引入 secret
```

`call()` 的关键顺序：

```txt
expand path
discover/activate skills
diagnosticTracker.beforeFileEdited()
mkdir parent
file history backup
重新读取文件并确认未 stale
生成 patch
writeTextContent()
通知 LSP / VSCode
更新 readFileState
记录 analytics
返回 patch/result
```

这里最重要的是 stale write 防护：

```txt
Read 记录 timestamp/content
  ↓
Edit 前再次检查 mtime/content
  ↓
如果用户或 linter 改过，拒绝写入
```

所以 Claude Code 的文件编辑安全依赖 `Read → Edit` 这条状态链。

---

## 10. 代表工具三：`BashTool`

文件：

```txt
src/tools/BashTool/BashTool.tsx
```

Bash 是最复杂的工具之一：

```ts
export const BashTool = buildTool({
  name: BASH_TOOL_NAME,
  maxResultSizeChars: 30_000,
  strict: true,

  isConcurrencySafe(input) {
    return this.isReadOnly?.(input) ?? false
  },
  isReadOnly(input) {
    return checkReadOnlyConstraints(...)
  },
  checkPermissions(input, context) {
    return bashToolHasPermission(input, context)
  },
  async call(input, toolUseContext, ..., onProgress) {
    ...
  },
})
```

几个核心点：

- Bash 是否可并发，取决于命令是否只读
- `preparePermissionMatcher()` 会解析复合命令，避免 `ls && git push` 绕过 `Bash(git *)` 这类 hook/规则
- `checkPermissions()` 进入专门的 bash permission 系统
- `call()` 通过 `runShellCommand()` 异步 generator 执行，并持续发 progress
- 大输出会通过 `persistedOutputPath` 转成 `<persisted-output>` 给模型
- background task 会返回 task id 和输出路径

所以 BashTool 本身是一个小型 shell runner + 权限解释器 + progress streamer。

---

## 11. 工具如何变成 API schema

文件：

```txt
src/utils/api.ts
```

转换函数：

```ts
export async function toolToAPISchema(tool, options): Promise<BetaToolUnion>
```

它把内部 `Tool` 转成 Anthropic API 的 tool schema：

```ts
base = {
  name: tool.name,
  description: await tool.prompt(...),
  input_schema:
    tool.inputJSONSchema ?? zodToJsonSchema(tool.inputSchema),
}
```

然后按条件附加：

```txt
strict
eager_input_streaming
defer_loading
cache_control
```

这里也做了缓存：

```txt
getToolSchemaCache()
```

原因是 tool schema 会影响 prompt cache。工具描述、schema 顺序、beta 字段如果频繁变化，会导致大块系统提示缓存失效。

---

## 12. MCP 工具如何包装成 Tool

基础模板：

```txt
src/tools/MCPTool/MCPTool.ts
```

它定义一个默认 MCPTool：

```ts
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',
  checkPermissions() {
    return {
      behavior: 'passthrough',
      message: 'MCPTool requires permission.',
    }
  },
  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return { tool_use_id: toolUseID, type: 'tool_result', content }
  },
})
```

真正转换发生在：

```txt
src/services/mcp/client.ts
```

`fetchToolsForClient()` 读取 MCP server 的 `tools/list`，然后：

```ts
return toolsToProcess.map((tool): Tool => {
  const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
  return {
    ...MCPTool,
    name: skipPrefix ? tool.name : fullyQualifiedName,
    mcpInfo: { serverName: client.name, toolName: tool.name },
    isMcp: true,
    inputJSONSchema: tool.inputSchema,
    async prompt() { return tool.description ?? '' },
    isConcurrencySafe() {
      return tool.annotations?.readOnlyHint ?? false
    },
    isReadOnly() {
      return tool.annotations?.readOnlyHint ?? false
    },
    isDestructive() {
      return tool.annotations?.destructiveHint ?? false
    },
    async call(args, context, ..., onProgress) {
      const mcpResult = await callMCPToolWithUrlElicitationRetry(...)
      return { data: mcpResult.content, mcpMeta: ... }
    },
  }
})
```

MCP tool 的名字通常是：

```txt
mcp__server__tool
```

工具名构造在：

```txt
src/services/mcp/mcpStringUtils.ts
```

```ts
export function buildMcpToolName(serverName, toolName) {
  return `${getMcpPrefix(serverName)}${normalizeNameForMCP(toolName)}`
}
```

这让 MCP 工具可以进入同一套：

```txt
toolToAPISchema
runToolUse
permission rules
rendering
tool_result mapping
```

---

## 13. `inputSchema` 与 `inputJSONSchema`

内置工具多数使用 Zod：

```ts
get inputSchema() {
  return inputSchema()
}
```

发送给 API 时：

```ts
zodToJsonSchema(tool.inputSchema)
```

MCP 工具已经从 server 拿到 JSON Schema，所以直接用：

```ts
inputJSONSchema: tool.inputSchema as Tool['inputJSONSchema']
```

转换层优先级：

```ts
tool.inputJSONSchema
  ? tool.inputJSONSchema
  : zodToJsonSchema(tool.inputSchema)
```

这就是为什么同一个 `Tool` contract 能同时容纳 TypeScript 内置工具和外部 MCP 工具。

---

## 14. 工具结果的双重表示

每个工具执行后先返回内部结果：

```ts
return {
  data,
  newMessages?,
  contextModifier?,
  mcpMeta?,
}
```

然后 `runToolUse()` 会调用：

```ts
tool.mapToolResultToToolResultBlockParam(result.data, toolUseID)
```

得到模型可见的：

```txt
ToolResultBlockParam
```

同时 UI 可能用：

```txt
renderToolResultMessage()
```

这两个东西不一定一样。例如：

- `FileReadTool`：模型看到文件内容和行号，UI 只显示 “Read N lines”
- `BashTool`：模型可能看到 `<persisted-output>` 包装，UI 显示 stdout/stderr
- `FileEditTool`：模型看到 “file updated successfully”，UI 显示 diff

所以要区分：

```txt
模型视图：mapToolResultToToolResultBlockParam()
用户视图：renderToolResultMessage()
```

---

## 15. 工具体系的关键设计

### A. 一个统一接口覆盖所有工具

内置工具、MCP 工具、Agent 工具、Skill 工具都实现 `Tool`。

主循环不关心工具类型，只做：

```txt
findToolByName()
validateInput()
checkPermissions()
call()
map result
```

### B. 权限前置但错误回写给模型

`validateInput` 或权限拒绝不会让程序崩掉，而是转成 `tool_result(is_error=true)`。

模型因此能恢复、换策略、解释失败。

### C. prompt cache 稳定性是工具池设计重点

`tools.ts` 排序、`toolToAPISchema()` 缓存、built-in prefix 优先、MCP 去重，都是为了避免工具 schema 轻微变化导致系统提示缓存失效。

### D. 文件工具共享 `readFileState`

Read、Edit、Write 不是孤立工具。

`readFileState` 串起：

```txt
Read 建立文件快照
  ↓
Edit/Write 检查是否先读过
  ↓
防止 stale write
  ↓
写后更新快照
```

这是 Claude Code 文件安全模型的核心。

---

## 16. 本步结论

Claude Code 的工具系统可以概括为：

```txt
Tool contract
  → buildTool 默认实现
  → tools.ts 组装内置工具
  → MCP client 包装外部工具
  → assembleToolPool 合并/过滤/排序
  → toolToAPISchema 发给模型
  → runToolUse 统一执行
  → tool_result 回写上下文
```

这套设计让 Claude Code 能把非常不同的能力统一进 agent loop：

```txt
shell command
file read/write/edit
web fetch/search
subagent
MCP server tool
skills
task/workflow tools
```

下一步如果继续解剖，建议进入第五步：

```txt
权限系统
  → useCanUseTool
  → permissions.ts
  → Bash permission classifier
  → File permission dialog
  → allow/deny/ask rules
  → hooks 如何改写 permission decision
```
