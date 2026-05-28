# Claude Code 源码解剖：权限系统

## 1. 总览

第四步已经看到每个工具都有：

```ts
validateInput?()
checkPermissions()
call()
```

第五步继续拆权限系统。核心链路如下：

```txt
runToolUse()
  ↓
resolveHookPermissionDecision()
  ↓
canUseTool()
src/hooks/useCanUseTool.tsx
  ↓
hasPermissionsToUseTool()
src/utils/permissions/permissions.ts
  ↓
tool.checkPermissions()
  ↓
allow / deny / ask
  ↓
ask 时进入 UI / bridge / channel / classifier / hooks 竞速
```

权限系统的目标不是只返回 true/false，而是返回一个结构化 decision：

```txt
allow：直接执行工具
deny：把拒绝原因作为 tool_result 回给模型
ask：进入交互确认，或在 headless/auto 模式下转成 deny/allow
```

---

## 2. 权限数据结构

文件：

```txt
src/types/permissions.ts
```

用户可见权限模式：

```ts
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',
  'bypassPermissions',
  'default',
  'dontAsk',
  'plan',
] as const
```

内部还可能有：

```ts
type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

权限行为：

```ts
type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

核心返回类型：

```ts
type PermissionDecision =
  | { behavior: 'allow'; updatedInput?; userModified?; decisionReason? }
  | { behavior: 'ask'; message; suggestions?; pendingClassifierCheck?; ... }
  | { behavior: 'deny'; message; decisionReason }
```

工具内部 `checkPermissions()` 可以额外返回：

```ts
{ behavior: 'passthrough'; message; decisionReason?; suggestions? }
```

`passthrough` 的意思是：工具自己没有明确 allow/deny/ask，交给通用权限层继续决定。

---

## 3. 权限规则模型

规则来源：

```ts
type PermissionRuleSource =
  | 'userSettings'
  | 'projectSettings'
  | 'localSettings'
  | 'flagSettings'
  | 'policySettings'
  | 'cliArg'
  | 'command'
  | 'session'
```

规则值：

```ts
type PermissionRuleValue = {
  toolName: string
  ruleContent?: string
}
```

常见形态：

```txt
Bash
Bash(git status:*)
Read(/src/**)
Edit(/.claude/skills/foo/**)
mcp__github
mcp__github__create_issue
```

规则分三组放在 `ToolPermissionContext`：

```txt
alwaysAllowRules
alwaysDenyRules
alwaysAskRules
```

其中 `ruleContent` 为空时匹配整个工具；有值时交给工具或文件权限系统做内容级匹配。

---

## 4. `useCanUseTool()` 是交互协调层

文件：

```txt
src/hooks/useCanUseTool.tsx
```

类型：

```ts
export type CanUseToolFn = (
  tool,
  input,
  toolUseContext,
  assistantMessage,
  toolUseID,
  forceDecision?,
) => Promise<PermissionDecision>
```

主流程：

```txt
createPermissionContext()
  ↓
hasPermissionsToUseTool()
  ↓
如果 allow：resolve allow
  ↓
如果 deny：resolve deny
  ↓
如果 ask：进入 coordinator / swarm / classifier / interactive handler
```

核心分支：

```ts
if (result.behavior === 'allow') {
  resolve(ctx.buildAllow(...))
}

switch (result.behavior) {
  case 'deny':
    resolve(result)
  case 'ask':
    handleInteractivePermission(..., resolve)
}
```

所以 `useCanUseTool()` 不负责判断权限规则本身。它负责把 `ask` 变成某个最终决策：

```txt
用户点允许
用户拒绝
用户 abort
hook 自动允许/拒绝
Bash classifier 自动允许
bridge / channel 远程批准
coordinator / swarm worker 代处理
```

---

## 5. `PermissionContext` 封装 ask 的状态机

文件：

```txt
src/hooks/toolPermission/PermissionContext.ts
```

`createPermissionContext()` 会生成一个上下文对象，里面有：

```txt
logDecision()
persistPermissions()
resolveIfAborted()
cancelAndAbort()
tryClassifier()
runHooks()
buildAllow()
buildDeny()
handleUserAllow()
handleHookAllow()
pushToQueue()
removeFromQueue()
updateQueueItem()
```

几个关键点：

- `cancelAndAbort()` 会构造拒绝消息，并在主线程场景下 abort 当前 tool/query
- `handleUserAllow()` 会持久化用户选择的权限更新
- `runHooks()` 执行 `PermissionRequest` hooks，hook 可以 allow 或 deny
- `createResolveOnce()` 保证 UI、hook、classifier、remote bridge 多方竞速时只 resolve 一次

这个层把 React permission queue 和权限判定逻辑解耦了。

---

## 6. 通用规则判定：`hasPermissionsToUseToolInner()`

文件：

```txt
src/utils/permissions/permissions.ts
```

主入口：

```ts
export const hasPermissionsToUseTool: CanUseToolFn = async (...) => {
  const result = await hasPermissionsToUseToolInner(...)
  ...
}
```

内部规则顺序非常重要：

```txt
1a. 整个工具 deny rule
1b. 整个工具 ask rule
1c. 调用 tool.checkPermissions()
1d. 工具自己 deny
1e. requiresUserInteraction 工具即使 bypass 也要问
1f. 内容级 ask rule 优先于 bypass
1g. safetyCheck 优先于 bypass
2a. bypassPermissions / plan+bypass 可直接 allow
2b. 整个工具 allow rule
3. passthrough 转 ask
```

也就是说：

```txt
deny / ask / safetyCheck
  优先级高于 bypassPermissions
```

`bypassPermissions` 不是无条件无视所有安全机制。显式 deny、显式 ask、工具 safety check 仍能挡住。

---

## 7. 外层模式转换：dontAsk / auto / headless

`hasPermissionsToUseTool()` 在 inner 结果之后继续处理模式。

### dontAsk

```txt
ask → deny
```

也就是不弹窗，直接拒绝。

### auto

```txt
ask → 尝试自动判断
```

auto 模式处理顺序大致是：

```txt
safetyCheck 且不可 classifierApprovable → 保持 ask / headless deny
requiresUserInteraction → 保持 ask
PowerShell 默认不进 classifier
acceptEdits fast path
safe tool allowlist
auto mode classifier
denial tracking / fallback to prompt
```

classifier 允许则：

```ts
{ behavior: 'allow', decisionReason: { type: 'classifier', classifier: 'auto-mode' } }
```

classifier 阻止则：

```ts
{ behavior: 'deny', message: buildYoloRejectionMessage(...) }
```

### headless / async agent

如果不能弹 permission prompt：

```txt
先跑 PermissionRequest hooks
  ↓
hook 没结果
  ↓
auto deny
```

这避免后台 agent 卡在无法显示的确认框上。

---

## 8. 交互 ask：多路竞速

文件：

```txt
src/hooks/toolPermission/handlers/interactiveHandler.ts
```

`handleInteractivePermission()` 会把请求推入 UI 队列：

```ts
ctx.pushToQueue({
  assistantMessage,
  tool,
  description,
  input,
  toolUseContext,
  toolUseID,
  permissionResult,
  onAbort,
  onAllow,
  onReject,
  recheckPermission,
  onUserInteraction,
})
```

然后同时启动多条可能 resolve 的路径：

```txt
本地 UI allow/reject/abort
CCR bridge 远程响应
channel permission relay 响应
PermissionRequest hooks
Bash classifier auto-approval
```

所有路径都用 `claim()` 防止双重 resolve：

```ts
if (!claim()) return
resolveOnce(...)
```

所以 permission prompt 本质是一个竞速状态机，而不是单纯等待用户按键。

---

## 9. 用户允许后如何持久化规则

用户允许时，UI 会传入：

```txt
updatedInput
permissionUpdates
feedback
contentBlocks
```

`PermissionContext.handleUserAllow()` 会：

```txt
persistPermissionUpdates()
applyPermissionUpdates()
logDecision()
detect userModified
return allow decision
```

`permissionUpdates` 可以是：

```txt
addRules
replaceRules
removeRules
setMode
addDirectories
removeDirectories
```

也就是说用户在 permission dialog 里点的 “allow once / always allow / allow directory / accept edits” 都会被转成结构化 permission update。

---

## 10. 文件权限：读和写是两套策略

文件：

```txt
src/utils/permissions/filesystem.ts
```

`FileReadTool` 调：

```ts
checkReadPermissionForTool()
```

`FileEditTool` / `FileWriteTool` 调：

```ts
checkWritePermissionForTool()
```

写权限核心顺序：

```txt
1. deny rules
1.5 internal editable paths
1.6 session-scoped .claude/** allow bypass
1.7 safety validations
2. ask rules
3. acceptEdits mode + path inside working dir → allow
4. allow rules
5. default ask + suggestions
```

安全检查包括：

```txt
Windows suspicious path patterns
.claude config / commands / agents / skills
MCP CLI state files
dangerous files: .gitconfig, .bashrc, .zshrc, .mcp.json, .claude.json
dangerous dirs: .git, .vscode, .idea, .claude
symlink resolved paths
```

重要点：

```txt
safety validations 在 allow rules 之前
```

这防止用户不小心永久允许了危险路径。`.claude/skills/{name}/**` 这种场景只允许 session 级窄规则绕过。

---

## 11. Working directory 权限

文件权限还维护 “允许工作目录”：

```ts
allWorkingDirectories(context)
```

包含：

```txt
original cwd
additionalWorkingDirectories
```

判断路径是否在允许目录中：

```ts
pathInAllowedWorkingPath(path, toolPermissionContext)
```

它会同时检查：

```txt
原始路径
resolved symlink path
大小写归一
macOS /private/var、/private/tmp 兼容
Windows POSIX path 兼容
```

所以 “路径在工作区内” 不是字符串前缀判断，而是经过一套跨平台归一和 symlink 检查。

---

## 12. Bash 权限：最复杂的工具权限

文件：

```txt
src/tools/BashTool/bashPermissions.ts
```

入口：

```ts
export async function bashToolHasPermission(input, context)
```

整体流程很长，核心阶段：

```txt
tree-sitter / legacy shell parse
too-complex / semantic danger → ask
sandbox auto-allow
exact rule match
Bash prompt deny/ask classifier
operator / pipe / redirection 检查
subcommand 拆分检查
path constraints
allow rule
read-only command allow
default passthrough/ask
```

Bash 的权限不是看整条字符串那么简单。它会尝试拆：

```txt
FOO=bar git status
cd dir && git status
echo x | xargs rm file
cmd > output
timeout 10 npm install
```

并且规则优先级特别保守：

```txt
deny / ask 规则先于 path constraints
deny 优先于 ask
deny/ask 可使用更激进 env var stripping
allow 规则不做同样激进 stripping，避免被环境变量绕过安全语义
compound command 不轻易被 allow prefix 覆盖
```

这部分代码里大量 SECURITY 注释，说明 Bash 权限是被真实绕过案例驱动出来的。

---

## 13. Bash classifier 有两类

Bash 相关 classifier 至少有两类用途。

### A. Bash prompt rule classifier

用于自然语言规则：

```txt
“允许运行测试命令”
“拒绝发布 npm package”
```

代码会对 deny 和 ask 描述并行分类：

```ts
const [denyResult, askResult] = await Promise.all([...])
```

deny 高置信命中优先返回 deny；ask 高置信命中返回 ask。

### B. Bash allow classifier

当普通权限结果是 ask 且有 `pendingClassifierCheck` 时，交互层可以在用户还没操作前自动批准：

```txt
pendingClassifierCheck
  ↓
awaitClassifierAutoApproval / executeAsyncClassifierCheck
  ↓
high confidence allow
  ↓
resolve allow
```

`useCanUseTool()` 里还有一个 2 秒 grace period：如果 speculative classifier 很快高置信 allow，就跳过弹窗。

---

## 14. Hooks 如何影响权限

权限系统有三类相关 hook 点：

```txt
PreToolUse
PermissionRequest
PostToolUse / PostToolUseFailure
```

第五步重点是 `PermissionRequest`：

```txt
PermissionContext.runHooks()
  ↓
executePermissionRequestHooks()
  ↓
hookResult.permissionRequestResult
  ↓
allow 或 deny
```

hook allow 可以带：

```txt
updatedInput
updatedPermissions
```

hook deny 可以带：

```txt
message
interrupt
```

在 headless/async agent 场景，`PermissionRequest` hooks 是 auto-deny 前最后的自动决策机会。

另外，`toolExecution.ts` 里 `resolveHookPermissionDecision()` 会先处理 `PreToolUse` hook 给出的 permission result，再进入 `canUseTool()`。这意味着 hook 可以比用户 prompt 更早改写权限流。

---

## 15. 权限拒绝如何回到模型

权限系统最终返回 deny 或 ask 被用户拒绝后，`toolExecution.ts` 会把它转成：

```txt
UserMessage {
  content: [
    {
      type: 'tool_result',
      is_error: true,
      tool_use_id,
      content: errorMessage
    }
  ]
}
```

所以权限拒绝不是隐藏在本地 UI 里，而是成为模型下一轮可见的工具结果。

这让模型可以继续：

```txt
解释被拒原因
选择更安全方案
请求更窄权限
改用 Read 而不是 Edit
终止任务
```

---

## 16. 本步结论

Claude Code 权限系统可以概括为：

```txt
Tool.checkPermissions()
  → 通用 rule/mode pipeline
  → auto/headless mode transform
  → interactive ask handler
  → hooks / classifier / user / remote bridge 竞速
  → PermissionDecision
  → toolExecution 转成 allow 或 tool_result error
```

最重要的设计点：

1. **权限不是布尔值，而是可解释、可持久化、可更新输入的 decision。**

2. **规则优先级保守：deny、ask、safetyCheck 可以压过 bypass 和 allow。**

3. **ask 是多方竞速：用户、hook、classifier、远程 bridge/channel 都可能给最终答案。**

4. **Bash 和 filesystem 是两套高风险专门权限系统，不只是通用 allowlist。**

下一步如果继续解剖，建议进入第六步：

```txt
Hooks 系统
  → hooks schema/config
  → PreToolUse / PostToolUse / PermissionRequest / Stop
  → hooks 如何执行 shell command
  → hooks 如何注入 messages、阻止 continuation、改写 input/permission
```
