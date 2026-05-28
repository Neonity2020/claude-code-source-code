# Claude Code 源码解剖：MCP 系统

## 1. 总览

前面已经拆到：

```txt
Tool 抽象
  ↓
权限系统
  ↓
Hooks 系统
```

第七步拆 MCP。MCP 在 Claude Code 里不是一条旁路，而是被完整并入工具系统：

```txt
MCP server config
  ↓
connectToServer()
  ↓
fetchToolsForClient()
  ↓
MCP tool -> Claude Code Tool
  ↓
appState.mcp.tools
  ↓
assembleToolPool()
  ↓
query.ts 发送给模型
  ↓
模型调用 mcp__server__tool
  ↓
MCPTool.call()
  ↓
client.callTool()
```

核心文件：

```txt
src/services/mcp/types.ts
src/services/mcp/config.ts
src/services/mcp/client.ts
src/services/mcp/useManageMCPConnections.ts
src/services/mcp/MCPConnectionManager.tsx
src/tools/MCPTool/MCPTool.ts
src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts
src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts
src/tools/McpAuthTool/McpAuthTool.ts
src/services/mcp/elicitationHandler.ts
src/services/mcp/mcpStringUtils.ts
src/tools.ts
```

一句话：Claude Code 把远端 MCP server 暴露的 tools、prompts、resources 转换成自己的 Tool、Command 和 Resource 状态，然后复用已有的工具执行、权限、hook、UI、transcript 体系。

---

## 2. 配置类型：MCP server 的 transport 模型

文件：

```txt
src/services/mcp/types.ts
```

MCP server 配置是一个 union：

```ts
type McpServerConfig =
  | { type?: 'stdio'; command; args; env? }
  | { type: 'sse'; url; headers?; headersHelper?; oauth? }
  | { type: 'http'; url; headers?; headersHelper?; oauth? }
  | { type: 'ws'; url; headers?; headersHelper? }
  | { type: 'sse-ide'; url; ideName; ideRunningInWindows? }
  | { type: 'ws-ide'; url; ideName; authToken?; ideRunningInWindows? }
  | { type: 'sdk'; name }
  | { type: 'claudeai-proxy'; url; id }
```

运行时会加上 scope：

```ts
type ScopedMcpServerConfig = McpServerConfig & {
  scope:
    | 'local'
    | 'user'
    | 'project'
    | 'dynamic'
    | 'enterprise'
    | 'claudeai'
    | 'managed'
  pluginSource?: string
}
```

连接状态也被建模成 union：

```ts
type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

这意味着 UI 和工具层不会只看到“有/没有连接”，而是能区分：

```txt
connected
failed
needs-auth
pending
disabled
```

---

## 3. 配置来源和优先级

文件：

```txt
src/services/mcp/config.ts
```

MCP 配置来源包括：

```txt
enterprise:
  managed-mcp.json

user:
  全局 config 的 mcpServers

project:
  当前目录及父目录的 .mcp.json

local:
  当前项目 local settings

dynamic:
  CLI / SDK / 运行时传入

plugin:
  enabled plugins 声明的 MCP servers

claudeai:
  claude.ai connectors
```

`getMcpConfigsByScope()` 负责读取单个 scope。`getClaudeCodeMcpConfigs()` 负责合并 Claude Code 本地可用的 MCP 配置。

合并策略：

```txt
enterprise config 存在
  enterprise 独占控制

否则：
  plugin < user < approved project < local
```

项目级 `.mcp.json` 还有批准流程：

```txt
projectServers
  ↓
getProjectMcpServerStatus(name) === 'approved'
  ↓
才进入连接集合
```

也就是说，项目仓库里的 `.mcp.json` 不会无条件启动外部进程或连接远端服务。

---

## 4. 企业策略：allowlist / denylist

同样在：

```txt
src/services/mcp/config.ts
```

策略入口：

```txt
allowedMcpServers
deniedMcpServers
allowManagedMcpServersOnly
pluginOnly policy
```

匹配方式有三类：

```txt
serverName
serverCommand
serverUrl
```

denylist 绝对优先：

```txt
isMcpServerAllowedByPolicy()
  ↓
先查 deniedMcpServers
  ↓
再查 allowedMcpServers
```

命令型规则用于 stdio：

```txt
command + args
```

URL 型规则用于 remote transport：

```txt
sse / http / ws / claudeai-proxy
```

URL 支持通配符模式，最终转成 RegExp。

另外，SDK 型 server 被 `filterMcpServersByPolicy()` 豁免，因为它不是 CLI 自己 spawn 进程或打开网络连接，而是 SDK 控制通道里的占位 server。

---

## 5. 插件和 connector 去重

配置层还有一块重要逻辑：同一个 MCP server 可能同时来自手写配置、插件和 claude.ai connector。

去重签名：

```ts
getMcpServerSignature(config)
```

规则：

```txt
stdio:
  command + args

remote:
  url

sdk:
  null
```

插件去重：

```txt
dedupPluginMcpServers(pluginServers, manualServers)
```

手写配置优先于插件；多个插件重复时，先加载的插件优先。

claude.ai connector 去重：

```txt
dedupClaudeAiMcpServers(claudeAiServers, manualServers)
```

手写配置优先于 web UI connector。这样可以避免同一个 Slack/GitHub MCP 同时以两个 server 名称暴露，浪费上下文和工具列表 token。

---

## 6. AppState 里的 MCP 状态

文件：

```txt
src/state/AppStateStore.ts
```

MCP 运行时状态放在：

```ts
mcp: {
  clients: MCPServerConnection[]
  tools: Tool[]
  commands: Command[]
  resources: Record<string, ServerResource[]>
  pluginReconnectKey: number
}
```

含义：

```txt
clients:
  server 连接状态

tools:
  MCP tools 转成的 Claude Code Tool

commands:
  MCP prompts / MCP skills 转成的 slash commands

resources:
  按 server 分组的 MCP resources

pluginReconnectKey:
  /reload-plugins 后触发 MCP 重连和重新发现
```

这个状态是 REPL、ToolSearch、/mcp UI、AgentTool、权限系统共同读取的共享源。

---

## 7. 连接入口：`getMcpToolsCommandsAndResources()`

文件：

```txt
src/services/mcp/client.ts
```

主入口：

```ts
getMcpToolsCommandsAndResources(onConnectionAttempt, mcpConfigs?)
```

它做四件事：

```txt
读取所有 server config
  ↓
跳过 disabled servers
  ↓
按 local / remote 分组并发连接
  ↓
每个 server 连接完成后回调 onConnectionAttempt
```

并发策略：

```txt
local servers:
  stdio / sdk
  默认 batch size 3

remote servers:
  sse / http / ws / claudeai-proxy
  默认 batch size 20
```

这样设计是因为本地 stdio 会 spawn 进程，过高并发容易拖垮启动；远端连接主要是网络 IO，可以更激进。

每个 server 成功连接后会并行拉取：

```txt
fetchToolsForClient()
fetchCommandsForClient()
fetchMcpSkillsForClient()
fetchResourcesForClient()
```

然后回调：

```ts
onConnectionAttempt({
  client,
  tools,
  commands,
  resources,
})
```

注意：只要某个 server 支持 resources，系统会额外加入两个内置 helper tool：

```txt
ListMcpResourcesTool
ReadMcpResourceTool
```

---

## 8. 连接单个 server：`connectToServer()`

核心函数：

```ts
connectToServer(name, serverRef, serverStats?)
```

它按 transport 创建不同 MCP SDK transport：

```txt
sse:
  SSEClientTransport

http:
  StreamableHTTPClientTransport

ws:
  WebSocketTransport

sse-ide:
  SSEClientTransport

ws-ide:
  WebSocketTransport + IDE auth header

claudeai-proxy:
  StreamableHTTPClientTransport + claude.ai OAuth bearer

stdio:
  StdioClientTransport

sdk:
  不走 connectToServer，由 setupSdkMcpClients 处理
```

特殊 in-process server：

```txt
claude-in-chrome
computer-use
```

它们本质上是 stdio 风格配置，但实际在当前进程内通过 linked transport 启动，避免 spawn 巨大的子进程。

连接时创建 MCP SDK client：

```ts
new Client(
  {
    name: 'claude-code',
    title: 'Claude Code',
    version,
    description,
    websiteUrl,
  },
  {
    capabilities: {
      roots: {},
      elicitation: {},
    },
  },
)
```

同时注册 roots handler：

```txt
ListRoots request
  ↓
返回 file://<original cwd>
```

这让 MCP server 能知道当前 workspace root。

---

## 9. 连接缓存和重连

`connectToServer` 是 memoized：

```ts
export const connectToServer = memoize(...)
```

cache key：

```ts
getServerCacheKey(name, serverRef)
```

连接断开或 session 失效时会清缓存：

```txt
clearServerCache(name, config)
```

工具调用前会走：

```ts
ensureConnectedClient(client)
```

如果连接健康，命中 memoized client；如果 session expired 或 onclose 后 cache 被清，下一次会重新连接。

HTTP / claude.ai proxy 的 session 失效检测包括：

```txt
HTTP 404 + JSON-RPC -32001
McpError -32000 Connection closed
```

命中后：

```txt
clearServerCache()
  ↓
throw McpSessionExpiredError
  ↓
MCPTool.call() retry once
```

---

## 10. Transport 细节

### stdio

```ts
new StdioClientTransport({
  command,
  args,
  env: {
    ...subprocessEnv(),
    ...serverRef.env,
  },
  stderr: 'pipe',
})
```

stderr 被 pipe 住，避免 MCP server 的错误输出直接污染 UI，同时连接失败时可以写 debug log。

### SSE / HTTP

remote transport 会组合：

```txt
ClaudeAuthProvider
getMcpServerHeaders()
getProxyFetchOptions()
wrapFetchWithTimeout()
wrapFetchWithStepUpDetection()
```

HTTP transport 还会确保 POST 请求带上：

```txt
Accept: application/json, text/event-stream
```

这是 MCP Streamable HTTP spec 需要的。

### claude.ai proxy

claude.ai connector 不直接连用户配置 URL，而是连：

```txt
oauthConfig.MCP_PROXY_URL + MCP_PROXY_PATH(server_id)
```

请求带：

```txt
Authorization: Bearer <claude.ai OAuth token>
X-Mcp-Client-Session-Id: <session id>
```

401 时会尝试刷新 claude.ai OAuth token 并重试一次。

---

## 11. MCP tool 如何变成 Claude Code Tool

文件：

```txt
src/services/mcp/client.ts
src/tools/MCPTool/MCPTool.ts
```

`fetchToolsForClient(client)` 调 MCP：

```txt
tools/list
```

然后把每个 MCP tool 映射成 Claude Code `Tool`：

```ts
{
  ...MCPTool,
  name: buildMcpToolName(client.name, tool.name),
  mcpInfo: { serverName: client.name, toolName: tool.name },
  isMcp: true,
  inputJSONSchema: tool.inputSchema,
  description: tool.description,
  prompt: tool.description,
  call: async args => ...
}
```

命名规则：

```txt
mcp__<normalized-server>__<normalized-tool>
```

例如：

```txt
mcp__github__create_issue
```

`mcpInfo` 保留未规范化的原始 server/tool 名称，用于权限判断和真实调用。

MCP annotations 被映射到 Tool 能力：

```txt
readOnlyHint
  isReadOnly()
  isConcurrencySafe()

destructiveHint
  isDestructive()

openWorldHint
  isOpenWorld()

title
  userFacingName()
```

description 有长度上限：

```txt
MAX_MCP_DESCRIPTION_LENGTH = 2048
```

防止 OpenAPI 生成类 MCP server 把超长接口文档塞进工具描述，污染 prompt。

---

## 12. MCP tool 的权限语义

MCPTool 默认：

```ts
checkPermissions() {
  return {
    behavior: 'passthrough',
    message: 'MCPTool requires permission.',
    suggestions: [
      {
        type: 'addRules',
        rules: [{ toolName: fullyQualifiedName }],
        behavior: 'allow',
        destination: 'localSettings',
      },
    ],
  }
}
```

也就是说，MCP tool 不自己直接 allow，而是交给通用权限系统：

```txt
tool.checkPermissions()
  ↓
passthrough
  ↓
settings allow / deny / ask rules
  ↓
interactive / hook / bridge / classifier
```

权限规则使用完整工具名：

```txt
mcp__server__tool
mcp__server
```

`getToolNameForPermissionCheck()` 会优先用 `mcpInfo` 重新构造完整 MCP 名称，避免无前缀 SDK MCP tool 和内置工具同名时误命中内置工具规则。

工具池组装时还会提前过滤 deny 规则：

```ts
assembleToolPool(permissionContext, mcpTools)
  ↓
filterToolsByDenyRules(mcpTools, permissionContext)
```

这样被 deny 的 MCP tool 在模型看到工具列表前就被移除，而不是等到调用时才拒绝。

---

## 13. MCP tool 调用链路

MCP tool 的 `call()` 最终走：

```txt
MCPTool.call()
  ↓
ensureConnectedClient()
  ↓
callMCPToolWithUrlElicitationRetry()
  ↓
callMCPTool()
  ↓
client.callTool()
  ↓
processMCPResult()
```

调用时会传 `_meta`：

```txt
claudecode/toolUseId
```

并把 MCP SDK progress 转成 Claude Code progress：

```txt
mcp_progress started
mcp_progress progress
mcp_progress completed
mcp_progress failed
```

超时默认值很大：

```txt
DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000
```

可以用环境变量覆盖：

```txt
MCP_TOOL_TIMEOUT
```

MCP `isError: true` 不会被当成普通成功结果，而是抛出 `McpToolCallError`。如果 error 里带 `_meta`，会保留给 SDK 消费者。

---

## 14. MCP result 处理

文件：

```txt
src/services/mcp/client.ts
src/utils/mcpValidation.ts
src/utils/mcpOutputStorage.ts
```

MCP result 可能包含：

```txt
text
image
resource_link
blob
structuredContent
_meta
```

处理目标：

```txt
控制大小
必要时截断
图片降采样
二进制内容落盘
保留 structuredContent / _meta
转换成 ToolResultBlockParam 可接受的内容
```

大输出不会无脑塞进上下文。源码里有：

```txt
mcpContentNeedsTruncation()
truncateMcpContentIfNeeded()
persistBinaryContent()
persistToolResult()
```

这和普通工具的 result truncation 是同一类问题：MCP 是外部服务，返回内容不可控，必须在进入模型上下文前做大小和格式治理。

---

## 15. MCP prompts 转 slash commands

`fetchCommandsForClient(client)` 调：

```txt
prompts/list
```

然后转成 Claude Code `Command`：

```txt
name:
  mcp__<server>__<prompt>

source:
  mcp

getPromptForCommand(args):
  client.getPrompt({ name, arguments })
  ↓
  transformResultContent()
```

这就是 MCP prompts 出现在 slash command 体系里的原因。对主循环来说，它们不是工具，而是动态 command。

---

## 16. MCP resources

MCP resources 走两层：

```txt
resources/list
  ↓
appState.mcp.resources
```

以及两个 helper tool：

```txt
ListMcpResourcesTool
ReadMcpResourceTool
```

`ListMcpResourcesTool`：

```txt
输入：
  server?

行为：
  对已连接 clients 调 fetchResourcesForClient()
  返回 uri / name / mimeType / description / server
```

`ReadMcpResourceTool`：

```txt
输入：
  server
  uri

行为：
  resources/read
  text 直接返回
  blob 解码后保存到磁盘，再把路径和说明返回给模型
```

它们是普通 Claude Code Tool，所以同样经过权限、hooks、tool execution 和 transcript 体系。

---

## 17. needs-auth：认证伪工具

文件：

```txt
src/tools/McpAuthTool/McpAuthTool.ts
```

当 remote server 需要 OAuth，但当前没有 token 或 401 时，连接结果变成：

```txt
NeedsAuthMCPServer
```

此时不是完全隐藏 server，而是注入一个伪工具：

```txt
mcp__<server>__authenticate
```

这个工具的作用：

```txt
performMCPOAuthFlow(skipBrowserOpen: true)
  ↓
返回 authorization URL
  ↓
用户完成浏览器授权
  ↓
后台 reconnectMcpServerImpl()
  ↓
用真实 MCP tools 替换 authenticate 伪工具
```

claude.ai connector 的认证走 `/mcp` UI，不由这个工具直接发起。

---

## 18. Elicitation：MCP server 向用户要信息

文件：

```txt
src/services/mcp/elicitationHandler.ts
```

连接时 client 声明：

```txt
elicitation: {}
```

然后注册：

```ts
registerElicitationHandler(client, serverName, setAppState)
```

当 MCP server 发送 elicitation request：

```txt
ElicitRequest
  ↓
runElicitationHooks()
  ↓
如果 hook 给了响应，直接返回
  ↓
否则加入 appState.elicitation.queue
  ↓
UI 展示表单或 URL 等待状态
  ↓
用户响应
  ↓
runElicitationResultHooks()
  ↓
返回给 MCP server
```

URL 模式还有 completion notification：

```txt
ElicitationCompleteNotification
  ↓
标记 queue item completed
  ↓
触发 Notification hook
```

这就是第六步 Hooks 和 MCP 的交叉点：MCP 的用户询问可以被 hook 自动处理，也可以在用户响应后被 hook 二次审计。

---

## 19. Interactive 和 print 模式的连接策略

Interactive REPL：

```txt
main.tsx
  ↓
启动 prefetchAllMcpResources() 暖连接
  ↓
REPL 挂载后 useManageMCPConnections()
  ↓
server 逐个更新 appState.mcp
```

关键设计：

```txt
MCP 不阻塞 REPL render
MCP 不阻塞 turn 1 TTFT
慢 server 可以 turn 2+ 再出现
```

Print / non-interactive：

```txt
main.tsx
  ↓
先把 clients 标成 pending
  ↓
await getMcpToolsCommandsAndResources()
  ↓
单轮执行前尽量拿到 MCP tools
```

因为 print 模式通常只有一轮，晚到的 MCP tool 没有第二轮可用，所以它更倾向等待连接完成。

---

## 20. `useManageMCPConnections()`

文件：

```txt
src/services/mcp/useManageMCPConnections.ts
src/services/mcp/MCPConnectionManager.tsx
```

REPL 里通过：

```tsx
<MCPConnectionManager>
  ...
</MCPConnectionManager>
```

挂载 MCP 管理器。

`useManageMCPConnections()` 负责：

```txt
初始化连接
批量写入 appState.mcp
监听 tools/list_changed
监听 prompts/list_changed
监听 resources/list_changed
处理 reconnect
处理 toggle enabled / disabled
处理 pluginReconnectKey
注册 channel permission callbacks
注册 elicitation handler
```

状态更新做了 16ms batch：

```txt
pendingUpdatesRef
  ↓
flushPendingUpdates()
  ↓
一次 setAppState 合并 clients/tools/commands/resources
```

这样避免几十个 MCP server 同时连接完成时频繁重渲染。

---

## 21. MCP tools 如何进入模型工具列表

文件：

```txt
src/tools.ts
src/screens/REPL.tsx
src/query.ts
```

REPL 中：

```txt
appState.mcp.tools
  ↓
useMergedTools()
  ↓
assembleToolPool(toolPermissionContext, mcp.tools)
```

`assembleToolPool()`：

```txt
getTools(permissionContext)
  ↓
filter MCP tools by deny rules
  ↓
built-ins sort by name
  ↓
MCP tools sort by name
  ↓
concat
  ↓
uniqBy(name)
```

内置工具放在前面，所以同名冲突时内置工具优先。源码里明确提到这是为了 prompt-cache 稳定性：不要让 MCP tools 插入内置工具中间，破坏服务端工具 cache 的边界。

最后 `query.ts` 读取当前 appState：

```txt
appState.mcp.tools
  ↓
tools
  ↓
API tool schema
  ↓
模型可调用
```

---

## 22. 名称和权限规则

文件：

```txt
src/services/mcp/mcpStringUtils.ts
```

核心函数：

```ts
buildMcpToolName(serverName, toolName)
getMcpPrefix(serverName)
mcpInfoFromString(toolString)
getToolNameForPermissionCheck(tool)
```

规则：

```txt
server/tool 名称会 normalize
工具名格式为 mcp__server__tool
server prefix 为 mcp__server__
```

权限规则可以写：

```txt
mcp__github
mcp__github__create_issue
```

已知限制：

```txt
server name 如果包含 "__"，mcpInfoFromString 会解析不准确
```

源码注释认为这在实践中少见。

---

## 23. 心智模型

MCP 系统可以分成四层：

```txt
配置层
  .mcp.json / settings / enterprise / plugin / dynamic / claude.ai

连接层
  stdio / sse / http / ws / sdk / claudeai-proxy

发现层
  tools/list
  prompts/list
  resources/list

集成层
  Tool
  Command
  Resource
  appState.mcp
  permissions
  hooks
  transcript
```

Claude Code 没有为 MCP 单独做一套执行系统。它只在边界处做转换：

```txt
MCP tool
  ↓
Claude Code Tool

MCP prompt
  ↓
Claude Code Command

MCP resource
  ↓
ServerResource + helper tools
```

转换完成后，主循环不需要知道工具来自内置实现还是 MCP server。

---

## 24. 下一步

现在主链路已经覆盖：

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

下一步适合拆：

```txt
Agent / Subagent 系统
  AgentTool 如何启动子 agent
  子 agent 如何继承工具和权限
  transcript / agentId / result 如何回到主线程
```
