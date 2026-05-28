# Claude Code 源码解剖：Agent / Subagent 系统

## 1. 总览

前七步已经串到 MCP：

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
```

第八步拆 Agent / Subagent。核心链路如下：

```txt
模型调用 AgentTool
  ↓
AgentTool.call()
  ↓
选择 AgentDefinition
  ↓
决定 sync / async / fork / worktree / remote / teammate
  ↓
runAgent()
  ↓
createSubagentContext()
  ↓
query()
  ↓
收集子 agent 消息
  ↓
finalizeAgentTool()
  ↓
返回给父 agent，或写入后台 task notification
```

核心文件：

```txt
src/tools/AgentTool/AgentTool.tsx
src/tools/AgentTool/runAgent.ts
src/tools/AgentTool/loadAgentsDir.ts
src/tools/AgentTool/agentToolUtils.ts
src/tools/AgentTool/resumeAgent.ts
src/tools/AgentTool/forkSubagent.ts
src/utils/forkedAgent.ts
src/tasks/LocalAgentTask/LocalAgentTask.tsx
src/tools/TaskOutputTool/TaskOutputTool.tsx
src/tools/TaskStopTool/TaskStopTool.ts
src/tools/SendMessageTool/SendMessageTool.ts
src/utils/sessionStorage.ts
```

一句话：Claude Code 的 subagent 不是独立执行器，而是“带隔离 ToolUseContext 的递归 query()”。AgentTool 只是负责创建子上下文、选择工具/模型/权限，然后重新进入同一套主循环。

---

## 2. AgentTool 的输入输出

文件：

```txt
src/tools/AgentTool/AgentTool.tsx
```

基础输入：

```ts
{
  description: string
  prompt: string
  subagent_type?: string
  model?: 'sonnet' | 'opus' | 'haiku'
  run_in_background?: boolean
}
```

多 agent / teammate 相关输入：

```ts
{
  name?: string
  team_name?: string
  mode?: PermissionMode
}
```

隔离相关输入：

```ts
{
  isolation?: 'worktree' | 'remote'
  cwd?: string
}
```

输出分几类：

```txt
同步完成:
  status: "completed"
  agentId
  content
  totalToolUseCount
  totalDurationMs
  totalTokens
  usage

后台启动:
  status: "async_launched"
  agentId
  outputFile
  canReadOutputFile

远程启动:
  status: "remote_launched"
  taskId
  sessionUrl
  outputFile

teammate 启动:
  status: "teammate_spawned"
```

AgentTool 是一个普通 Tool，所以它本身也会经过权限、hooks、tool execution 和 UI 渲染。

---

## 3. AgentDefinition：agent 的配置模型

文件：

```txt
src/tools/AgentTool/loadAgentsDir.ts
```

核心类型：

```ts
type BaseAgentDefinition = {
  agentType: string
  whenToUse: string
  tools?: string[]
  disallowedTools?: string[]
  skills?: string[]
  mcpServers?: AgentMcpServerSpec[]
  hooks?: HooksSettings
  color?: AgentColorName
  model?: string
  effort?: EffortValue
  permissionMode?: PermissionMode
  maxTurns?: number
  requiredMcpServers?: string[]
  background?: boolean
  initialPrompt?: string
  memory?: 'user' | 'project' | 'local'
  isolation?: 'worktree' | 'remote'
}
```

Agent 来源有三类：

```txt
built-in
  内置 agent，getSystemPrompt 是动态函数

custom
  user / project / local / policy settings 中的 markdown/json agent

plugin
  插件提供的 agent
```

最终 union：

```ts
type AgentDefinition =
  | BuiltInAgentDefinition
  | CustomAgentDefinition
  | PluginAgentDefinition
```

---

## 4. Agent 加载和覆盖优先级

入口：

```ts
getAgentDefinitionsWithOverrides(cwd)
```

加载来源：

```txt
内置 agents
  getBuiltInAgents()

自定义 agents
  loadMarkdownFilesForSubdir('agents', cwd)

插件 agents
  loadPluginAgents()
```

激活列表由：

```ts
getActiveAgentsFromList(allAgents)
```

合并规则：

```txt
built-in
plugin
userSettings
projectSettings
flagSettings
policySettings
```

同名 `agentType` 后面的覆盖前面的。也就是说，策略或更高优先级设置可以覆盖同名 agent。

简单模式：

```txt
CLAUDE_CODE_SIMPLE
  只返回内置 agents
```

SDK 非交互模式还可以用：

```txt
CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS
```

禁用内置 agent，得到空白 agent surface。

---

## 5. 内置 Agents

文件：

```txt
src/tools/AgentTool/builtInAgents.ts
```

基础内置：

```txt
general-purpose
statusline-setup
```

按 feature gate 加载：

```txt
Explore
Plan
Claude Code Guide
Verification
Coordinator workers
```

内置 agent 的特点：

```txt
source: "built-in"
baseDir: "built-in"
getSystemPrompt({ toolUseContext })
callback?
```

内置 agent 的 prompt 通常是动态构造的，能根据当前工具、模型、环境和 feature gate 变化。

---

## 6. AgentTool.prompt() 如何告诉模型可用 agents

`AgentTool.prompt()` 会读取：

```txt
agentDefinitions.activeAgents
tools
toolPermissionContext
allowedAgentTypes
```

然后过滤：

```txt
required MCP servers 是否满足
  ↓
Agent(...) permission deny 规则
  ↓
allowedAgentTypes 限制
```

最后调用：

```ts
getPrompt(filteredAgents, isCoordinator, allowedAgentTypes)
```

这意味着模型看到的可委派 agent 列表是动态的：

```txt
MCP server 未连接或未认证
  依赖它的 agent 不显示

权限规则 deny Agent(foo)
  foo 不显示

Agent 工具被限制为 Agent(a,b)
  只显示 a/b
```

---

## 7. Agent 选择流程

`AgentTool.call()` 里的核心选择逻辑：

```txt
如果 team_name + name
  spawnTeammate()

否则：
  subagent_type 存在
    选择该 AgentDefinition

  subagent_type 不存在且 fork gate 开启
    进入 fork path

  subagent_type 不存在且 fork gate 关闭
    使用 general-purpose
```

选择后会检查：

```txt
Agent(AgentName) 是否被 deny
requiredMcpServers 是否有对应 MCP tools
in-process teammate 是否允许 background
agent definition 是否强制 background
```

如果 required MCP server 还在 pending，会最多等待：

```txt
30 秒
```

避免 agent 过早启动时误判 MCP tools 不存在。

---

## 8. 模型、权限和工具池

模型解析：

```ts
getAgentModel(agentDefinition.model, mainLoopModel, modelParam, permissionMode)
```

优先级可以理解为：

```txt
AgentTool 输入 model
agent frontmatter model
parent mainLoopModel
permission mode 相关降级/覆盖
```

权限上下文：

```ts
workerPermissionContext = {
  ...appState.toolPermissionContext,
  mode: selectedAgent.permissionMode ?? 'acceptEdits'
}
```

工具池：

```ts
workerTools = assembleToolPool(workerPermissionContext, appState.mcp.tools)
```

然后 `runAgent()` 内还会用：

```ts
resolveAgentTools(agentDefinition, availableTools, isAsync)
```

根据 agent 的 `tools` / `disallowedTools` 做二次裁剪。

关键点：子 agent 的工具池不是直接继承父 agent 当前看到的工具。普通子 agent 会用自己的 permission mode 重新 assemble，避免父线程的局部限制意外泄漏。

---

## 9. Agent 工具过滤

文件：

```txt
src/tools/AgentTool/agentToolUtils.ts
```

`filterToolsForAgent()` 会过滤：

```txt
ALL_AGENT_DISALLOWED_TOOLS
CUSTOM_AGENT_DISALLOWED_TOOLS
ASYNC_AGENT_ALLOWED_TOOLS
```

特殊规则：

```txt
MCP tools
  所有 agents 都允许

ExitPlanMode
  plan mode agent 允许

async agent
  只允许 async-safe tools

in-process teammate
  可保留 AgentTool 和 task coordination tools
```

`resolveAgentTools()` 支持：

```txt
tools: undefined 或 ["*"]
  使用过滤后的全部工具

tools: ["Read", "Grep", "mcp__github__..."]
  只解析这些工具

disallowedTools
  从可用工具中移除

Agent(worker,researcher)
  解析出 allowedAgentTypes
```

---

## 10. runAgent：递归进入 query()

文件：

```txt
src/tools/AgentTool/runAgent.ts
```

`runAgent()` 是真正执行子 agent 的地方：

```ts
export async function* runAgent({...}): AsyncGenerator<Message, void>
```

主流程：

```txt
解析 model / agentId
  ↓
构造 initialMessages
  ↓
构造 userContext / systemContext
  ↓
解析 permission mode
  ↓
解析工具列表
  ↓
构造 agent system prompt
  ↓
执行 SubagentStart hooks
  ↓
注册 agent frontmatter hooks
  ↓
预加载 agent skills
  ↓
初始化 agent-specific MCP servers
  ↓
createSubagentContext()
  ↓
recordSidechainTranscript(initialMessages)
  ↓
query(...)
  ↓
逐条 yield 子 agent 消息
  ↓
finally 清理 MCP/hooks/cache/file state/tasks
```

最关键的一行：

```ts
for await (const message of query({ ... })) {
  yield message
}
```

所以 subagent 和主线程使用同一个 `query()` 主循环。同样支持工具调用、权限、hooks、MCP、stop hooks、compact 等机制，只是上下文不同。

---

## 11. createSubagentContext：隔离边界

文件：

```txt
src/utils/forkedAgent.ts
```

`createSubagentContext(parentContext, overrides)` 默认做隔离：

```txt
readFileState
  clone

abortController
  默认 child controller，父 abort 会传播

getAppState
  包装后设置 shouldAvoidPermissionPrompts

setAppState
  默认 no-op

setResponseLength
  默认 no-op

nestedMemoryAttachmentTriggers
loadedNestedMemoryPaths
dynamicSkillDirTriggers
discoveredSkillNames
  新集合

contentReplacementState
  clone

queryTracking
  新 chainId，depth + 1
```

可显式共享：

```txt
shareSetAppState
shareSetResponseLength
shareAbortController
```

AgentTool 对 sync agent 通常会：

```txt
shareSetAppState: true
shareSetResponseLength: true
```

async agent 则更隔离：

```txt
setAppState no-op
独立 abortController
shouldAvoidPermissionPrompts: true
```

但有一个特殊通道永远能到 root store：

```txt
setAppStateForTasks
```

这是为了让后台 task 注册、进度、kill 不会因为子 agent 的 `setAppState` 被隔离成 no-op 而失效。

---

## 12. 子 agent 的权限行为

`runAgent()` 里的 `agentGetAppState()` 会动态修改 AppState：

```txt
如果 agent 定义 permissionMode
  覆盖当前 mode
  但不覆盖 bypassPermissions / acceptEdits / auto

如果 async agent 不能展示权限 UI
  shouldAvoidPermissionPrompts = true

如果 background agent 可以展示权限 UI
  awaitAutomatedChecksBeforeDialog = true

如果 allowedTools 指定
  替换 session allow rules
  保留 cliArg allow rules

如果 agent 定义 effort
  覆盖 effortValue
```

这解释了为什么 async subagent 一般不能卡住等待用户确认：它会倾向自动拒绝或依赖 hook/classifier 自动处理。

---

## 13. Subagent hooks 和 frontmatter hooks

启动时：

```txt
executeSubagentStartHooks(agentId, agentType)
```

如果 hook 返回 `additionalContext`，会作为 attachment message 加入 agent 初始消息。

agent 定义也可以带 frontmatter hooks：

```txt
agentDefinition.hooks
  ↓
registerFrontmatterHooks(rootSetAppState, agentId, hooks, ..., isAgent=true)
```

`isAgent=true` 的效果：

```txt
Stop hooks
  转成 SubagentStop hooks
```

策略限制：

```txt
strict plugin-only hooks
  用户自定义 agent 的 hooks 不注册
  plugin / built-in / policy trusted agent 可以注册
```

生命周期结束时：

```txt
clearSessionHooks(rootSetAppState, agentId)
```

---

## 14. Agent-specific MCP servers 和 skills

agent frontmatter 可以定义：

```txt
mcpServers
skills
```

MCP：

```txt
initializeAgentMcpServers()
  ↓
引用已有 server 名称，或 inline 定义 server config
  ↓
connectToServer()
  ↓
fetchToolsForClient()
  ↓
把 agent-specific MCP tools 加到该 agent 的工具池
```

inline MCP server 是 agent 生命周期内创建的，结束时清理；引用已有 server 则复用父上下文/全局 memoized 连接，不由 agent 清理。

Skills：

```txt
agentDefinition.skills
  ↓
getSkillToolCommands()
  ↓
resolveSkillName()
  ↓
skill.getPromptForCommand()
  ↓
作为 meta user message 加入 initialMessages
```

这让 agent 可以在启动时自动预加载特定 skill 内容。

---

## 15. Sync agent：父线程等待结果

同步路径：

```txt
shouldRunAsync === false
  ↓
runWithAgentContext(syncAgentContext)
  ↓
runAgent()
  ↓
收集 agentMessages
  ↓
finalizeAgentTool()
  ↓
返回 status: completed
```

同步 agent 会把关键进度向父 tool 透传：

```txt
agent_progress
bash_progress
powershell_progress
token count
tool use snippets
```

同步 agent 运行超过阈值会显示 background hint：

```txt
PROGRESS_THRESHOLD_MS = 2000
```

同时它已经注册成 foreground task，可以被用户 background。

---

## 16. Foreground agent 转 background

同步 agent 启动时会：

```txt
registerAgentForeground()
```

这会创建一个 foreground `local_agent` task：

```txt
isBackgrounded: false
backgroundSignal: Promise<void>
```

如果用户或自动计时器触发 background：

```txt
backgroundAgentTask()
  ↓
isBackgrounded = true
  ↓
resolve backgroundSignal
```

AgentTool 的 sync loop 里一直 race：

```txt
agentIterator.next()
  vs
backgroundSignal
```

一旦 background：

```txt
agentIterator.return()
  ↓
重新用 runAgent({ isAsync: true }) 继续执行
  ↓
立即向父 agent 返回 async_launched
```

这是一处很细的设计：不是把同一个 async iterator 硬塞到后台，而是先关闭前台 iterator，再用同样参数启动后台 continuation。

---

## 17. Async agent：后台任务生命周期

后台路径：

```txt
shouldRunAsync === true
  ↓
registerAsyncAgent()
  ↓
void runAsyncAgentLifecycle(...)
  ↓
立即返回 async_launched
```

`registerAsyncAgent()` 创建：

```txt
LocalAgentTaskState {
  type: "local_agent"
  status: "running"
  agentId
  prompt
  selectedAgent
  abortController
  progress
  pendingMessages
  outputFile symlink -> agent transcript
}
```

后台执行由：

```ts
runAsyncAgentLifecycle()
```

负责：

```txt
消费 runAgent() 消息
  ↓
更新 progress tracker
  ↓
updateAsyncAgentProgress()
  ↓
emitTaskProgress()
  ↓
finalizeAgentTool()
  ↓
completeAgentTask()
  ↓
enqueueAgentNotification()
```

失败或取消：

```txt
AbortError
  killAsyncAgent()
  enqueue killed notification

其他错误
  failAsyncAgent()
  enqueue failed notification
```

---

## 18. Task notification：后台结果如何回到模型

文件：

```txt
src/tasks/LocalAgentTask/LocalAgentTask.tsx
```

后台 agent 完成后：

```ts
enqueueAgentNotification(...)
```

会写入 message queue：

```xml
<task-notification>
  <task_id>...</task_id>
  <tool_use_id>...</tool_use_id>
  <output_file>...</output_file>
  <status>completed|failed|killed</status>
  <summary>...</summary>
  <result>...</result>
  <usage>...</usage>
  <worktree>...</worktree>
</task-notification>
```

`query.ts` 会在主循环中 drain task-notification：

```txt
主线程
  drain agentId === undefined 的通知

subagent
  只 drain 发给自己 agentId 的 task-notification
```

这就是后台 agent 能在未来一轮重新唤醒父 agent 的机制。

---

## 19. TaskOutput / TaskStop

文件：

```txt
src/tools/TaskOutputTool/TaskOutputTool.tsx
src/tools/TaskStopTool/TaskStopTool.ts
```

`TaskOutputTool`：

```txt
输入:
  task_id
  block = true
  timeout = 30000

行为:
  block=false 立即返回当前状态
  block=true 轮询等待 task 完成
  local_agent 优先返回内存中的 clean final result
  否则读取 outputFile
```

虽然 prompt 已标记 deprecated，建议直接 Read output file，但它仍兼容旧 transcripts 和 SDK 用户。

`TaskStopTool`：

```txt
输入:
  task_id

行为:
  stopTask()
  ↓
getTaskByType(task.type).kill()
  ↓
agent task 调 killAsyncAgent()
  shell task 调 shell kill
```

agent 被 stop 后，后台 lifecycle catch 到 AbortError，会发送带 partial result 的 killed notification。

---

## 20. SendMessage 和可寻址后台 agent

文件：

```txt
src/tools/SendMessageTool/SendMessageTool.ts
src/tasks/LocalAgentTask/LocalAgentTask.tsx
```

如果 AgentTool 输入带：

```txt
name
```

异步 agent 注册成功后会写：

```txt
agentNameRegistry: name -> agentId
```

SendMessage 可以把消息路由到：

```txt
teammate inbox
broadcast
remote bridge
UDS peer
local async agent pendingMessages
```

对 local async agent，消息进入：

```ts
queuePendingMessage(taskId, msg, setAppState)
```

`LocalAgentTaskState.pendingMessages` 会在子 agent 后续回合边界被 drain，作为新的用户消息继续该 agent。

这解释了为什么后台 agent 不是“只能等结果”的任务，也可以继续被父 agent 或用户交互驱动。

---

## 21. Sidechain transcript

文件：

```txt
src/utils/sessionStorage.ts
```

子 agent transcript 单独写在：

```txt
<projectDir>/<sessionId>/subagents/agent-<agentId>.jsonl
```

核心函数：

```ts
getAgentTranscriptPath(agentId)
recordSidechainTranscript(messages, agentId, parentUuid?)
getAgentTranscript(agentId)
writeAgentMetadata(agentId, metadata)
readAgentMetadata(agentId)
```

`runAgent()` 在 query 前记录 initial messages：

```txt
recordSidechainTranscript(initialMessages, agentId)
writeAgentMetadata(agentId, { agentType, worktreePath, description })
```

query 中每条 recordable message 会继续追加：

```txt
assistant
user
progress
compact_boundary
```

sidechain transcript 的作用：

```txt
UI 展示 subagent 详细过程
feedback / transcript share 收集 subagent 历史
TaskOutput outputFile symlink
resumeAgentBackground() 恢复后台 agent
```

---

## 22. Resume 后台 agent

文件：

```txt
src/tools/AgentTool/resumeAgent.ts
```

恢复流程：

```txt
resumeAgentBackground(agentId, prompt)
  ↓
getAgentTranscript(agentId)
  ↓
readAgentMetadata(agentId)
  ↓
过滤坏消息:
    whitespace-only assistant
    orphaned thinking-only
    unresolved tool uses
  ↓
reconstructForSubagentResume()
  ↓
选择原 agentType
  ↓
恢复 worktree cwd
  ↓
registerAsyncAgent()
  ↓
runAsyncAgentLifecycle()
```

fork agent 恢复有特殊逻辑：必须重建父 system prompt，保持 cache-identical prefix；否则会退化成 general-purpose，破坏 fork 的上下文语义和 prompt cache。

---

## 23. Fork subagent

文件：

```txt
src/tools/AgentTool/forkSubagent.ts
src/utils/forkedAgent.ts
```

fork path 的触发：

```txt
subagent_type omitted
fork feature enabled
```

普通 subagent：

```txt
新 system prompt
简单 user prompt
自己的工具池
thinking disabled
```

fork subagent：

```txt
继承父 system prompt
继承父 messages 前缀
继承父 tools
继承 thinkingConfig
useExactTools = true
```

目标是：

```txt
让 fork 的 API request 前缀与父线程 cache-critical params 完全一致
  ↓
命中 Anthropic prompt cache
```

为了避免递归 fork：

```txt
如果当前 querySource 是 fork agent
或消息里检测到 fork child 标记
  拒绝再次 fork
```

---

## 24. Worktree / cwd / remote isolation

AgentTool 支持：

```txt
isolation: "worktree"
cwd: "/abs/path"
isolation: "remote"   // ant-only
```

worktree：

```txt
createAgentWorktree(slug)
  ↓
runWithCwdOverride(worktreePath)
  ↓
agent 在独立 git worktree 中执行
  ↓
完成后检查 hasWorktreeChanges()
  ↓
无变更则 removeAgentWorktree()
  ↓
有变更则保留并在结果/notification 中返回路径
```

fork + worktree 会额外注入 notice，提醒 child 将父路径映射到 worktree 路径并重新读取文件。

remote：

```txt
checkRemoteAgentEligibility()
  ↓
teleportToRemote()
  ↓
registerRemoteAgentTask()
  ↓
返回 remote_launched
```

远程 agent 不在本地 `runAgent()` 中执行，而是交给 CCR remote session。

---

## 25. Teammate / swarm 路径

AgentTool 也复用来 spawn teammate：

```txt
team_name + name
  ↓
spawnTeammate()
  ↓
返回 teammate_spawned
```

限制：

```txt
teammate 不能再 spawn teammate
  因为 team roster 是 flat array

in-process teammate 不能 spawn background agent
  生命周期绑定 leader process
```

teammate 通信主要靠：

```txt
SendMessageTool
teammate mailbox
team file
shutdown / plan approval structured messages
```

这条路径和普通 subagent 有交叉，但不是同一个执行模型：teammate 可以是 tmux/外部进程/in-process teammate，而普通 subagent 是本进程内递归 query。

---

## 26. 结果收敛：finalizeAgentTool()

文件：

```txt
src/tools/AgentTool/agentToolUtils.ts
```

`finalizeAgentTool()` 从 agentMessages 中提取：

```txt
最后一个 assistant 的 text blocks
totalToolUseCount
totalDurationMs
totalTokens
usage
agentId
agentType
```

如果最后一条 assistant 只有 tool_use，没有 text，会向前找最近的 text assistant message。

它还会发 analytics：

```txt
tengu_agent_tool_completed
tengu_cache_eviction_hint
```

同步路径把结果直接作为 AgentTool tool_result 返回给父 agent；异步路径把结果写入 task state，并通过 task notification 回到消息队列。

---

## 27. Handoff classifier

在 auto mode + feature gate 下，subagent 交回结果前会跑：

```ts
classifyHandoffIfNeeded()
```

它会把 subagent transcript 交给安全 classifier：

```txt
Sub-agent has finished and is handing back control...
```

如果 classifier block：

```txt
同步结果:
  在 agentResult.content 前插入 SECURITY WARNING

异步结果:
  在 notification 的 finalMessage 前插入 warning
```

如果 classifier unavailable，也会插入提醒父 agent 谨慎验证的 warning。

---

## 28. 清理逻辑

`runAgent()` finally 做了大量清理：

```txt
mcpCleanup()
clearSessionHooks(agentId)
cleanupAgentTracking(agentId)
readFileState.clear()
initialMessages.length = 0
unregisterPerfettoAgent(agentId)
clearAgentTranscriptSubdir(agentId)
删除 AppState.todos[agentId]
killShellTasksForAgent(agentId)
killMonitorMcpTasksForAgent(agentId)
```

AgentTool 外层也会：

```txt
clearInvokedSkillsForAgent(agentId)
clearDumpState(agentId)
cleanupWorktreeIfNeeded()
unregisterAgentForeground()
complete / fail / kill task
```

这块的存在说明 subagent 是高频、长生命周期对象；如果不清理，文件状态、hooks、todos、shell 任务、dump prompts、MCP 连接都会慢慢泄漏。

---

## 29. 心智模型

Agent 系统可以理解成五层：

```txt
定义层
  AgentDefinition
  built-in / custom / plugin

选择层
  AgentTool.call()
  subagent_type / fork / teammate / remote / worktree

上下文层
  createSubagentContext()
  permission / tools / model / abort / AppState isolation

执行层
  runAgent()
  query()

任务层
  LocalAgentTask
  TaskOutput
  TaskStop
  SendMessage
  task-notification
```

最核心的抽象关系是：

```txt
AgentTool is a Tool
Subagent is query() with a different ToolUseContext
Background agent is query() plus LocalAgentTask lifecycle
```

所以 Claude Code 没有维护一套独立的“agent runtime”。它通过递归复用 query loop，把 agent 能力自然挂到已有的工具、权限、hooks、MCP 和 transcript 系统上。

---

## 30. 下一步

现在已经覆盖：

```txt
启动入口
REPL 输入
query 主循环
Tool 抽象
权限系统
Hooks 系统
MCP 系统
Agent / Subagent 系统
```

下一步适合继续拆：

```txt
Session / Transcript 持久化系统
  主会话 JSONL
  sidechain transcript
  resume / compact / parentUuid chain
  tool result replacement
```
