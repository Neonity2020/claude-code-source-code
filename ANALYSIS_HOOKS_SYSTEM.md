# Claude Code 源码解剖：Hooks 系统

## 1. 总览

第五步拆到权限系统时已经看到一个关键插点：

```txt
runToolUse()
  ↓
runPreToolUseHooks()
  ↓
resolveHookPermissionDecision()
  ↓
canUseTool()
  ↓
tool.call()
  ↓
runPostToolUseHooks()
```

第六步继续拆 Hooks 系统。它是 Claude Code 里最像“插件总线”的部分：工具调用、用户提交、会话开始/结束、权限请求、compact、配置变更、文件变更、MCP elicitation、worktree 生命周期，都可以通过同一套 hook engine 接入。

核心文件：

```txt
src/schemas/hooks.ts
src/types/hooks.ts
src/utils/hooks.ts
src/services/tools/toolHooks.ts
src/hooks/toolPermission/PermissionContext.ts
src/query/stopHooks.ts
```

一句话：Hooks 系统把外部命令、LLM prompt、agent、HTTP、SDK callback、session function 都统一成 `HookInput -> HookJSONOutput`，再把结果折叠成权限决策、阻断反馈、附加上下文、输入改写或后台任务。

---

## 2. 配置 schema：四种可持久化 hook

文件：

```txt
src/schemas/hooks.ts
```

settings 里的 hook command 是一个 discriminated union：

```ts
type HookCommand =
  | { type: 'command'; command; if?; shell?; timeout?; statusMessage?; once?; async?; asyncRewake? }
  | { type: 'prompt'; prompt; if?; timeout?; model?; statusMessage?; once? }
  | { type: 'http'; url; if?; timeout?; headers?; allowedEnvVars?; statusMessage?; once? }
  | { type: 'agent'; prompt; if?; timeout?; model?; statusMessage?; once? }
```

配置结构是：

```ts
type Hooks = Partial<Record<HookEvent, HookMatcher[]>>

type HookMatcher = {
  matcher?: string
  hooks: HookCommand[]
}
```

也就是：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node ./guard.js",
            "if": "Bash(git *)"
          }
        ]
      }
    ]
  }
}
```

这里有两层过滤：

```txt
matcher：先按事件相关字段匹配，例如工具名 Bash / Write / Edit
if：再用权限规则语法匹配具体输入，例如 Bash(git *) / Read(*.ts)
```

`if` 的设计很重要：它避免了每次工具调用都 spawn 外部进程，只在真正匹配时运行 hook。

---

## 3. Hook 输出协议

文件：

```txt
src/types/hooks.ts
```

hook 可以输出普通文本，也可以输出 JSON。JSON 分两类：

```ts
type HookJSONOutput =
  | { async: true; asyncTimeout?: number }
  | SyncHookResponse
```

同步响应的通用字段：

```ts
{
  continue?: boolean
  suppressOutput?: boolean
  stopReason?: string
  decision?: 'approve' | 'block'
  reason?: string
  systemMessage?: string
  hookSpecificOutput?: ...
}
```

通用语义：

```txt
continue: false
  阻止当前链路继续推进，附带 stopReason

decision: "approve"
  产生 allow 权限行为

decision: "block"
  产生 deny / blockingError

systemMessage
  作为 hook_system_message 附件进入消息流

suppressOutput
  控制 stdout 是否写入 transcript / attachment
```

事件专属输出放在 `hookSpecificOutput` 里：

```txt
PreToolUse:
  permissionDecision
  permissionDecisionReason
  updatedInput
  additionalContext

PostToolUse:
  additionalContext
  updatedMCPToolOutput

UserPromptSubmit / SessionStart / Setup / SubagentStart:
  additionalContext

SessionStart / CwdChanged / FileChanged:
  watchPaths

PermissionRequest:
  allow / deny
  updatedInput
  updatedPermissions
  interrupt

PermissionDenied:
  retry

Elicitation / ElicitationResult:
  accept / decline / cancel
  content

WorktreeCreate:
  worktreePath
```

这解释了 Hooks 为什么能影响不同层面：同一套 JSON 协议里既有“给模型看的上下文”，也有“给权限系统看的决策”，还有“给工具执行看的输入/输出改写”。

---

## 4. Hook 输入基座

文件：

```txt
src/utils/hooks.ts
```

所有 hook input 都基于：

```ts
createBaseHookInput(permissionMode?, sessionId?, agentInfo?)
```

基础字段包括：

```txt
session_id
transcript_path
cwd
permission_mode
agent_id
agent_type
```

不同事件再补自己的字段：

```txt
PreToolUse:
  hook_event_name
  tool_name
  tool_input
  tool_use_id

PostToolUse:
  tool_name
  tool_input
  tool_response
  tool_use_id

UserPromptSubmit:
  prompt

SessionStart:
  source
  agent_type
  model

PermissionRequest:
  tool_name
  tool_input
  permission_suggestions
```

设计上，hook 不需要知道 Claude Code 内部对象，只接收一份结构化 JSON。

---

## 5. Hook 来源合并

核心函数：

```ts
getHooksConfig(appState, sessionId, hookEvent)
```

它把多个来源合并成一组 matcher：

```txt
settings snapshot
  getHooksConfigFromSnapshot()

registered hooks
  SDK callbacks
  plugin native hooks

session hooks
  agent / skill / frontmatter 注册的临时 hooks

session function hooks
  不能持久化的函数型 hooks，例如结构化输出约束
```

这里有两个企业策略开关：

```txt
shouldDisableAllHooksIncludingManaged()
  全部禁用，包括 managed hooks

shouldAllowManagedHooksOnly()
  只允许 policySettings / managed hooks
  跳过 plugin hooks 和 session hooks
```

所以 hook 配置不是简单读取 settings。它是“当前会话、当前 agent、当前策略、当前插件状态”共同决定的结果。

---

## 6. 匹配、去重和 `if` 条件

核心函数：

```ts
getMatchingHooks(appState, sessionId, hookEvent, hookInput, tools?)
```

它先根据事件类型计算 `matchQuery`：

```txt
PreToolUse / PostToolUse / PermissionRequest:
  tool_name

SessionStart:
  source

Setup:
  trigger

Notification:
  notification_type

SessionEnd:
  reason

SubagentStart / SubagentStop:
  agent_type

Elicitation / ElicitationResult:
  mcp_server_name

ConfigChange:
  source

InstructionsLoaded:
  load_reason

FileChanged:
  basename(file_path)
```

`matcher` 支持三种形式：

```txt
*
Write
Write|Edit
正则表达式，例如 ^(Write|Edit)$
```

匹配后再做去重：

```txt
command hook:
  shell + command + if + pluginRoot/skillRoot

prompt hook:
  prompt + if + pluginRoot/skillRoot

agent hook:
  prompt + if + pluginRoot/skillRoot

http hook:
  url + if + pluginRoot/skillRoot
```

callback / function hook 不去重，因为每个 callback 本身就是独立函数。

最后应用 `if` 条件：

```txt
if: "Bash(git *)"
  ↓
permissionRuleValueFromString()
  ↓
tool.preparePermissionMatcher(input)
  ↓
true / false
```

注意：`if` 目前只对工具相关事件有意义：

```txt
PreToolUse
PostToolUse
PostToolUseFailure
PermissionRequest
```

另外，HTTP hooks 被明确禁止用于 `SessionStart` 和 `Setup`，原因是这些早期阶段在 headless / sandbox 输入消费者启动前可能死锁。

---

## 7. 执行主循环：`executeHooks()`

文件：

```txt
src/utils/hooks.ts
```

主入口：

```ts
async function* executeHooks({...}): AsyncGenerator<AggregatedHookResult>
```

执行流程：

```txt
检查 disableAllHooks / CLAUDE_CODE_SIMPLE
  ↓
检查 workspace trust
  ↓
getMatchingHooks()
  ↓
如果全是 internal callback，走快速路径
  ↓
为每个 hook yield hook_progress
  ↓
并行执行所有 hook
  ↓
解析 stdout / HTTP body / callback return
  ↓
processHookJSONOutput()
  ↓
把结果折叠成 AggregatedHookResult 流式 yield 给调用方
```

它是 async generator，不是一次性 Promise。原因是调用方需要边跑边拿到：

```txt
progress attachment
blockingError
permissionBehavior
updatedInput
additionalContexts
updatedMCPToolOutput
permissionRequestResult
retry
preventContinuation
stopReason
```

权限行为有优先级：

```txt
deny > ask > allow
```

也就是说多个 PreToolUse hooks 同时返回不同意见时，最保守结果胜出。

---

## 8. REPL 外执行：`executeHooksOutsideREPL()`

有些事件不在主 query 流里执行，例如：

```txt
Notification
SessionEnd
PreCompact
PostCompact
ConfigChange
CwdChanged
FileChanged
InstructionsLoaded
Elicitation
WorktreeCreate
WorktreeRemove
```

这些走：

```ts
executeHooksOutsideREPL(...)
```

差异：

```txt
executeHooks()
  产出 AttachmentMessage / ProgressMessage
  可以把上下文反馈给模型
  用于工具、Stop、UserPromptSubmit 等 REPL 内链路

executeHooksOutsideREPL()
  返回 HookOutsideReplResult[]
  错误主要写 debug / stderr
  用于生命周期、通知、文件、配置等外部事件
```

外部链路仍然遵守 trust、managed policy、matcher、timeout、JSON block 等规则。

---

## 9. command hook：真实进程执行

command hook 由：

```ts
execCommandHook(...)
```

负责执行。

它做的事情比“spawn 一个命令”多得多：

```txt
选择 shell
  bash / powershell / 默认 shell

展开插件变量
  ${CLAUDE_PLUGIN_ROOT}
  ${CLAUDE_PLUGIN_DATA}
  ${user_config.X}

注入环境变量
  CLAUDE_PROJECT_DIR
  CLAUDE_ENV_FILE
  plugin env
  skill root

把 hook input JSON 写入 stdin

收集 stdout / stderr / exit code

处理 prompt request 协议

支持 abort / timeout / EPIPE

支持 async / asyncRewake
```

传统退出码语义：

```txt
0
  success

2
  blocking error
  stderr 会作为阻断反馈

其他非 0
  non_blocking_error
```

如果 stdout 是 JSON，则优先走 JSON 协议；不是 JSON，则按普通文本处理。

---

## 10. prompt / agent / http / callback / function hooks

除 command 以外还有几类 hook：

```txt
prompt hook
  用 LLM 评估 prompt
  $ARGUMENTS 可替换为 hook input JSON

agent hook
  启动一个 agentic verifier
  适合做更复杂的检查

http hook
  POST hook input JSON 到 URL
  返回值必须是 JSON

callback hook
  SDK / 内部注册的 TypeScript callback

function hook
  session 内临时函数 hook
  需要 messages，只在 REPL 上下文里使用
```

这几类 hook 最终都要归一到：

```txt
HookResult
  ↓
AggregatedHookResult
```

所以调用方不关心 hook 是 shell、HTTP、LLM 还是内存函数。

---

## 11. JSON 输出解析：`processHookJSONOutput()`

核心解析函数：

```ts
processHookJSONOutput({...})
```

它把 JSON 输出转换成内部结果：

```txt
continue: false
  preventContinuation = true
  stopReason = ...

decision: "approve"
  permissionBehavior = "allow"

decision: "block"
  permissionBehavior = "deny"
  blockingError = ...

systemMessage
  systemMessage = ...

hookSpecificOutput.additionalContext
  additionalContext = ...

hookSpecificOutput.updatedInput
  updatedInput = ...

hookSpecificOutput.updatedMCPToolOutput
  updatedMCPToolOutput = ...

PermissionRequest.decision
  permissionRequestResult = allow / deny

PermissionDenied.retry
  retry = true
```

这里是 hook 从“外部协议”进入 Claude Code 内部控制流的关键转换点。

---

## 12. Tool hooks 桥接层

文件：

```txt
src/services/tools/toolHooks.ts
```

这个文件把通用 hook engine 接到工具执行系统。

### PreToolUse

入口：

```ts
runPreToolUseHooks(...)
```

它消费 `executePreToolHooks()` 的结果并产出：

```txt
message
hookPermissionResult
hookUpdatedInput
preventContinuation
stopReason
additionalContext
stop
```

关键行为：

```txt
blockingError
  转成 deny 权限结果

permissionBehavior allow / ask / deny
  转成 hookPermissionResult

updatedInput 且无 permission decision
  作为 passthrough 输入改写

additionalContext
  作为 hook_additional_context attachment

abort
  产出 hook_cancelled 并 stop
```

### 权限决策合并

入口：

```ts
resolveHookPermissionDecision(...)
```

这里有一个非常关键的安全不变量：

```txt
hook allow 不会绕过 settings deny / ask 规则
```

流程是：

```txt
hook allow
  ↓
checkRuleBasedPermissions()
  ↓
如果没有 deny/ask 规则，才跳过交互确认
  ↓
否则 deny/ask 规则继续生效
```

如果 hook 返回 `ask`，它不是直接拒绝，而是通过 `forceDecision` 进入正常 `canUseTool()` 交互流程。

### PostToolUse

入口：

```ts
runPostToolUseHooks(...)
```

它处理：

```txt
hook_progress / hook_success / hook_error
blockingError
preventContinuation
additionalContext
updatedMCPToolOutput
```

`updatedMCPToolOutput` 只对 MCP tool 生效，用来让 PostToolUse hook 改写 MCP 返回值。

### PostToolUseFailure

入口：

```ts
runPostToolUseFailureHooks(...)
```

失败 hook 用于工具调用失败后的补充处理，例如：

```txt
记录失败
追加上下文
给模型反馈失败原因
```

---

## 13. 工具执行链路里的 hook 顺序

文件：

```txt
src/services/tools/toolExecution.ts
```

关键接入点：

```txt
runToolUse()
  ↓
runPreToolUseHooks()
  ↓
resolveHookPermissionDecision()
  ↓
tool.call()
  ↓
runPostToolUseHooks()
```

失败时：

```txt
tool.call() / validation / execution failure
  ↓
runPostToolUseFailureHooks()
```

权限被拒时：

```txt
permission denied
  ↓
executePermissionDeniedHooks()
  ↓
可能 retry
```

所以 PreToolUse 发生在权限确认前，PostToolUse 发生在工具成功后，PostToolUseFailure 发生在失败后，PermissionDenied 发生在拒绝后。

---

## 14. PermissionRequest hook

文件：

```txt
src/hooks/toolPermission/PermissionContext.ts
src/utils/hooks.ts
```

当正常权限系统准备向用户弹出确认时，会触发：

```ts
executePermissionRequestHooks(...)
```

它给 hook 的 input 包括：

```txt
tool_name
tool_input
permission_suggestions
```

hook 可以返回：

```txt
allow:
  updatedInput?
  updatedPermissions?

deny:
  message?
  interrupt?
```

这相当于在“用户确认弹窗”之前加了一个自动化决策层。SDK / structured IO 也会走同一套 PermissionRequest hook，所以 headless 模式可以用 hook 自动响应权限请求。

---

## 15. Stop hooks：让模型停下前再检查一次

文件：

```txt
src/query/stopHooks.ts
src/utils/hooks.ts
```

Stop hooks 的调用点不是工具系统，而是 query 收尾阶段：

```txt
assistant 准备停止
  ↓
handleStopHooks()
  ↓
executeStopHooks()
  ↓
如果 hook block / preventContinuation
  ↓
把反馈作为 user/meta message 加回 query
  ↓
模型继续一轮
```

这类 hook 适合做：

```txt
回答质量检查
提交前检查
任务未完成提醒
结构化输出约束
自动记忆提取
```

`Stop` 还有一个保护字段：

```txt
stop_hook_active
```

用于避免 Stop hook 自己触发 Stop hook 后无限递归。

---

## 16. 生命周期 hooks

`src/utils/hooks.ts` 里还有大量事件包装器：

```txt
executeUserPromptSubmitHooks()
  用户提交 prompt 后，进入主 query 前

executeSessionStartHooks()
  session startup / resume / clear / compact

executeSetupHooks()
  init / maintenance

executeSubagentStartHooks()
  子 agent 启动

executePreCompactHooks()
executePostCompactHooks()
  compact 前后

executeSessionEndHooks()
  会话结束

executeNotificationHooks()
  通知事件

executeConfigChangeHooks()
  settings / skills / policy 等配置变更

executeCwdChangedHooks()
executeFileChangedHooks()
  cwd / 文件监听事件

executeInstructionsLoadedHooks()
  CLAUDE.md / rules 加载审计

executeElicitationHooks()
executeElicitationResultHooks()
  MCP elicitation 请求和结果

executeWorktreeCreateHook()
executeWorktreeRemoveHook()
  worktree 生命周期
```

这些 wrapper 的共同模式：

```txt
构造 hookInput
  ↓
选择 matchQuery
  ↓
executeHooks() 或 executeHooksOutsideREPL()
  ↓
把 AggregatedHookResult 映射成调用方需要的返回值
```

---

## 17. 安全边界

Hooks 是 RCE 能力，所以源码里有多层边界。

### Workspace trust

```ts
shouldSkipHookDueToTrust()
```

交互模式下，如果 workspace 未被信任，所有 hooks 都跳过。非交互 / SDK 路径会跳过这项交互式 trust 检查。

### Managed policy

```txt
disableAllHooks
  禁掉所有 hooks

allowManagedHooksOnly
  只允许 managed / policy hooks
```

### Timeout

默认工具 hook 超时：

```ts
TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000
```

`SessionEnd` 有更短的默认超时常量：

```ts
SESSION_END_HOOK_TIMEOUT_MS_DEFAULT = 1500
```

每个 hook 还可以配置自己的 `timeout`。

### Shell 和环境

command hook 的环境是显式组装的：

```txt
CLAUDE_PROJECT_DIR
CLAUDE_ENV_FILE
plugin env
skill root
```

插件变量也只在 plugin hook 上展开，避免普通 settings hook 获得不该有的 plugin 上下文。

### HTTP header env 白名单

HTTP hook 的 header 环境变量需要列在：

```txt
allowedEnvVars
```

否则不会插值。

---

## 18. 性能设计

Hooks 在工具热路径上，所以源码里有几处明确优化：

```txt
hasHookForEvent()
  先做轻量存在性检查，避免构造 hook input 和完整匹配

internal callback fast-path
  全部是内部 callback 时跳过 progress/span/JSON 处理

lazy stringify hookInput
  同一批 command/prompt/agent/http hooks 共享一次 JSON.stringify

if condition
  在 spawn 前过滤

callback/function-only dedup fast-path
  避免对内部 hooks 做多轮 Map/filter
```

这说明 Hooks 不是边缘功能，它在主工具执行路径上频繁出现，必须控制开销。

---

## 19. 心智模型

可以把 Hooks 系统理解成三层：

```txt
配置层
  settings / plugin / skill / session / SDK callback

执行层
  command / prompt / agent / http / callback / function

语义层
  permission decision
  blocking feedback
  additional context
  input rewrite
  MCP output rewrite
  lifecycle side effect
```

源码中最关键的抽象不是某一种 hook，而是：

```txt
HookInput
  ↓
HookJSONOutput
  ↓
AggregatedHookResult
```

一旦外部机制被压成这个协议，它就能被工具链路、权限链路、Stop 链路和生命周期链路复用。

---

## 20. 下一步

前六步已经串起来：

```txt
启动入口
  ↓
REPL 输入
  ↓
query 主循环
  ↓
tool 抽象
  ↓
permission 系统
  ↓
hooks 系统
```

下一步可以继续拆两个方向之一：

```txt
MCP 系统
  看外部 MCP server 如何变成 Claude Code Tool

Agent / Subagent 系统
  看 AgentTool 如何启动子 agent、隔离上下文、回收结果
```
