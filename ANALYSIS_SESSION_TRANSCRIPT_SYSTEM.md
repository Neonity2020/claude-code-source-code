# Claude Code 源码解剖：Session / Transcript 持久化系统

## 1. 总览

前八步已经串到 Agent：

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
```

第九步拆 Session / Transcript。它回答三个问题：

```txt
1. 当前对话如何落盘？
2. /resume、--continue、--resume 如何恢复？
3. compaction、rewind、subagent、worktree、remote session 如何不把链弄断？
```

核心文件：

```txt
src/utils/sessionStorage.ts
src/utils/sessionStoragePortable.ts
src/utils/conversationRecovery.ts
src/utils/sessionRestore.ts
src/hooks/useLogMessages.ts
src/commands/resume/resume.tsx
src/screens/REPL.tsx
src/types/logs.ts
```

一句话：Claude Code 的 session 持久化不是“把 messages 数组完整 dump 成 JSON”，而是一个 append-only JSONL 事件日志。真正恢复时，会从最新 leaf 沿 `parentUuid` 反向走链，重建一条有效对话。

---

## 2. 文件布局

项目级 transcript 根目录：

```ts
getProjectsDir()
  = join(getClaudeConfigHomeDir(), 'projects')

getProjectDir(cwd)
  = join(getProjectsDir(), sanitizePath(cwd))
```

主 session 文件：

```txt
~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl
```

代码：

```ts
export function getTranscriptPath(): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  return join(projectDir, `${getSessionId()}.jsonl`)
}
```

subagent sidechain 文件：

```txt
~/.claude/projects/<sanitized-cwd>/<sessionId>/subagents/agent-<agentId>.jsonl
```

代码：

```ts
export function getAgentTranscriptPath(agentId: AgentId): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  const sessionId = getSessionId()
  const base = join(projectDir, sessionId, 'subagents')
  return join(base, `agent-${agentId}.jsonl`)
}
```

agent sidecar metadata：

```txt
agent-<agentId>.meta.json
```

remote agent sidecar：

```txt
<sessionId>/remote-agents/remote-agent-<taskId>.meta.json
```

所以文件结构大概是：

```txt
~/.claude/projects/-Users-zion-project/
  <sessionId>.jsonl
  <sessionId>/
    subagents/
      agent-<agentId>.jsonl
      agent-<agentId>.meta.json
    remote-agents/
      remote-agent-<taskId>.meta.json
```

`getSessionProjectDir()` 是一个关键修正点：跨目录 resume、worktree、branch 场景下，当前进程 cwd 可能不等于 transcript 所在 project dir，所以代码把 sessionId 和 projectDir 作为一个原子状态处理。

---

## 3. JSONL Entry 类型

类型定义在：

```txt
src/types/logs.ts
```

主消息被序列化成：

```ts
type SerializedMessage = Message & {
  cwd: string
  userType: string
  entrypoint?: string
  sessionId: string
  timestamp: string
  version: string
  gitBranch?: string
  slug?: string
}
```

落盘消息再加链信息：

```ts
type TranscriptMessage = SerializedMessage & {
  parentUuid: UUID | null
  logicalParentUuid?: UUID | null
  isSidechain: boolean
  agentId?: string
  teamName?: string
  agentName?: string
  agentColor?: string
  promptId?: string
}
```

也就是说，每一条可恢复消息不是只记录自身内容，还记录：

```txt
这条消息是谁的孩子？
属于哪个 session？
当时 cwd 是什么？
是否 sidechain？
属于哪个 agent？
当时 git branch 是什么？
```

除 transcript message 外，JSONL 还混有 metadata/event entry：

```txt
summary
custom-title
ai-title
last-prompt
task-summary
tag
agent-name
agent-color
agent-setting
mode
worktree-state
pr-link
file-history-snapshot
attribution-snapshot
content-replacement
queue-operation
marble-origami-commit
marble-origami-snapshot
speculation-accept
```

这就是为什么 loader 不是简单 `JSON.parse(lines)` 后直接用，而是要把不同 entry 分流到不同 Map。

---

## 4. 哪些消息算 transcript message

核心判断：

```ts
export function isTranscriptMessage(entry: Entry): entry is TranscriptMessage {
  return (
    entry.type === 'user' ||
    entry.type === 'assistant' ||
    entry.type === 'attachment' ||
    entry.type === 'system'
  )
}
```

注意：`progress` 不是 transcript message。

原因在注释里写得很明确：

```txt
Progress messages are ephemeral UI state
and must not be persisted to the JSONL
or participate in the parentUuid chain.
```

也就是说：

```txt
assistant 正文
user 输入
tool_result
system checkpoint
attachment
  → 可以成为恢复链的一部分

bash_progress / mcp_progress / sleep_progress
  → UI 临时状态，不进入 parentUuid 链
```

否则会出现一种非常隐蔽的问题：旧 progress 插进链里，resume 时 `buildConversationChain()` 找不到它，整条链被截断。

为兼容旧日志，loader 还有 `isLegacyProgressEntry()` 和 `progressBridge`，把旧 progress 节点桥接过去。

---

## 5. 写入入口：REPL 的 useLogMessages

REPL 中：

```ts
useLogMessages(messages, messages.length === initialMessages?.length)
```

位置：

```txt
src/screens/REPL.tsx
src/hooks/useLogMessages.ts
```

这里有一个重要优化：它不是每次 React render 都把全量 `messages` 交给 storage，而是维护：

```ts
lastRecordedLengthRef
lastParentUuidRef
firstMessageUuidRef
callSeqRef
```

增量路径：

```txt
messages 仍是同一个 head，长度增加
  ↓
只取 messages.slice(prevLength)
  ↓
带上 lastParentUuidRef 作为 parent hint
  ↓
recordTranscript(slice, ..., parentHint, allMessages)
```

非增量路径：

```txt
first message uuid 变化
  ↓
可能是 compaction / clear / fork
  ↓
交给 recordTranscript(fullArray) 自己 dedup 和重新建链
```

同 head 变短也被特殊处理：

```txt
same-head shrink
  tombstone / rewind / snip / partial-compact
```

这个 hook 的关键目标是：

```txt
UI 不阻塞
写入可异步
parentUuid 不因为异步 race 指到错误消息
compaction 后不会把 messagesToKeep 接到旧链上
```

---

## 6. recordTranscript：写入前的清洗和 dedup

入口：

```ts
export async function recordTranscript(
  messages: Message[],
  teamInfo?: TeamInfo,
  startingParentUuidHint?: UUID,
  allMessages?: readonly Message[],
): Promise<UUID | null>
```

第一步清洗：

```ts
const cleanedMessages = cleanMessagesForLogging(messages, allMessages)
```

清洗逻辑：

```ts
export function isLoggableMessage(m: Message): boolean {
  if (m.type === 'progress') return false
  if (m.type === 'attachment' && getUserType() !== 'ant') {
    ...
    return false
  }
  return true
}
```

外部用户还会做 REPL tool wrapper 剥离：

```ts
transformMessagesForExternalTranscript()
```

目的：

```txt
外部 transcript 不保留 REPL wrapper
恢复时模型看到的是原生 tool_use / tool_result 历史
```

第二步 dedup：

```ts
const messageSet = await getSessionMessages(sessionId)
```

如果消息 UUID 已经写过，就跳过。

但这段 dedup 有一个很细的规则：

```txt
已记录消息只有在它们形成 prefix 时，才更新 startingParentUuid
```

原因是 compaction 场景里：

```txt
new compact boundary / summary
  出现在数组前面
messagesToKeep
  是旧 UUID，出现在后面
```

如果简单把已记录的 `messagesToKeep` 当成 parent，新的 compact boundary 会被错误接到 pre-compact 链上。

所以注释里分了两类：

```txt
Growing-array callers:
  recorded messages are always a prefix → track parent

Compaction:
  new CB/summary appear FIRST, then recorded messagesToKeep
  → not a prefix → do not track parent
```

最后写入：

```ts
await getProject().insertMessageChain(
  newMessages,
  false,
  undefined,
  startingParentUuid,
  teamInfo,
)
```

---

## 7. insertMessageChain：给消息补链

核心函数：

```ts
Project.insertMessageChain(
  messages,
  isSidechain,
  agentId,
  startingParentUuid,
  teamInfo,
)
```

它做几件事：

```txt
1. 第一个 user/assistant 消息出现时 materialize session file
2. 获取当前 git branch
3. 获取当前 plan slug
4. 给每条消息补 parentUuid / sessionId / cwd / version / gitBranch
5. appendEntry()
6. 更新 parentUuid 游标
7. 缓存 last-prompt 给 /resume 列表显示
```

关键代码形态：

```ts
const transcriptMessage: TranscriptMessage = {
  parentUuid: isCompactBoundary ? null : effectiveParentUuid,
  logicalParentUuid: isCompactBoundary ? parentUuid : undefined,
  isSidechain,
  teamName: teamInfo?.teamName,
  agentName: teamInfo?.agentName,
  promptId: message.type === 'user' ? (getPromptId() ?? undefined) : undefined,
  agentId,
  ...message,
  userType: getUserType(),
  entrypoint: getEntrypoint(),
  cwd: getCwd(),
  sessionId,
  version: VERSION,
  gitBranch,
  slug,
}
```

tool_result 有特殊 parent：

```ts
if (
  message.type === 'user' &&
  'sourceToolAssistantUUID' in message &&
  message.sourceToolAssistantUUID
) {
  effectiveParentUuid = message.sourceToolAssistantUUID
}
```

原因：

```txt
tool_result 的逻辑 parent 应该是产生 tool_use 的 assistant 消息
不是简单接在上一条消息之后
```

compact boundary 也特殊：

```txt
parentUuid = null
logicalParentUuid = 原 parent
```

这会显式截断恢复链，但保留逻辑 parent 用于调试或分析。

---

## 8. Project 写队列

`Project` 是 sessionStorage 的内存写入管理器：

```ts
class Project {
  sessionFile: string | null = null
  pendingEntries: Entry[] = []
  writeQueues = new Map<string, Array<{ entry; resolve }>>()
  flushTimer
  activeDrain
}
```

写入不是每条消息立刻 `appendFile`，而是：

```txt
appendEntry()
  ↓
enqueueWrite(filePath, entry)
  ↓
scheduleDrain()
  ↓
drainWriteQueue()
  ↓
appendToFile()
```

默认 flush interval：

```ts
FLUSH_INTERVAL_MS = 100
```

远程 session / CCR v2 会降到：

```ts
REMOTE_FLUSH_INTERVAL_MS = 10
```

队列按 filePath 分组：

```txt
主 session 文件一队
每个 subagent sidechain 文件一队
content-replacement 可能写主文件或 agent 文件
```

大 chunk 限制：

```ts
MAX_CHUNK_BYTES = 100 * 1024 * 1024
```

退出时注册 cleanup：

```ts
registerCleanup(async () => {
  await project?.flush()
  project?.reAppendSessionMetadata()
})
```

这保证进程退出时：

```txt
先把异步队列刷盘
再把 metadata 重新追加到尾部
```

---

## 9. 为什么要 lazy materialize session file

初始状态：

```ts
sessionFile: null
pendingEntries: []
```

如果只有 metadata，比如：

```txt
--name
agent setting
mode
worktree state
```

不会立刻创建 `<sessionId>.jsonl`。

只有出现第一条 user/assistant 消息时：

```ts
materializeSessionFile()
```

才真正创建文件，并把 pending entries 刷进去。

目的：

```txt
避免启动后没有真实对话，却留下 metadata-only session 文件
避免 /resume 列表被空 session 污染
```

这也解释了为什么很多保存函数只是 cache：

```ts
saveAgentSetting()
cacheSessionTitle()
saveMode()
saveWorktreeState()
```

它们会先写进 `Project.currentSession*` 字段，等 session materialize 或退出 reAppend。

---

## 10. appendEntry 的分流规则

`appendEntry()` 按 entry type 分流。

metadata 类永远可追加：

```txt
summary
custom-title
ai-title
last-prompt
task-summary
tag
agent-name
agent-color
agent-setting
pr-link
file-history-snapshot
attribution-snapshot
mode
worktree-state
context-collapse entries
```

transcript message 需要 dedup：

```ts
const messageSet = await getSessionMessages(sessionId)
const isNewUuid = !messageSet.has(entry.uuid)
```

sidechain 有例外：

```ts
const isAgentSidechain = entry.isSidechain && entry.agentId !== undefined
```

sidechain 本地写入绕过主 session dedup：

```txt
fork-inherited parent messages 可能和主 session UUID 相同
如果按主 session dedup，会导致 agent sidechain 文件不完整
```

但 sidechain 不会把 UUID 加到主 session messageSet：

```txt
否则主线程后续写同 UUID 时会被跳过
主链 parentUuid 可能指到只存在于 agent 文件的 UUID
resume 时断链
```

这是 sessionStorage 里最容易误改的地方之一。

---

## 11. remote persistence

本地写入后，主 transcript message 可能同步到远程。

CCR v2：

```ts
internalEventWriter('transcript', entry, {
  isCompaction,
  agentId,
})
```

v1 Session Ingress：

```ts
sessionIngress.appendSessionLog(sessionId, entry, remoteIngressUrl)
```

如果 v1 remote persistence 失败：

```ts
gracefulShutdownSync(1, 'other')
```

remote hydrate：

```ts
hydrateRemoteSession(sessionId, ingressUrl)
```

流程：

```txt
switchSession(sessionId)
  ↓
sessionIngress.getSessionLogs()
  ↓
覆盖写本地 <sessionId>.jsonl
  ↓
project.setRemoteIngressUrl()
```

CCR v2 hydrate：

```ts
hydrateFromCCRv2InternalEvents(sessionId)
```

流程：

```txt
读 foreground internal events
  ↓
写主 session jsonl
  ↓
读 subagent internal events
  ↓
按 agent_id 分组
  ↓
写 agent-<agentId>.jsonl
```

所以远程恢复本质还是先把远程事件“水合”为本地 JSONL，再走统一 loader。

---

## 12. 读取入口：loadTranscriptFile

主 loader：

```ts
export async function loadTranscriptFile(filePath, opts?)
```

返回：

```ts
{
  messages: Map<UUID, TranscriptMessage>
  summaries: Map<UUID, string>
  customTitles: Map<UUID, string>
  tags: Map<UUID, string>
  agentNames: Map<UUID, string>
  agentColors: Map<UUID, string>
  agentSettings: Map<UUID, string>
  prNumbers: Map<UUID, number>
  prUrls: Map<UUID, string>
  prRepositories: Map<UUID, string>
  modes: Map<UUID, string>
  worktreeStates: Map<UUID, PersistedWorktreeSession | null>
  fileHistorySnapshots: Map<UUID, FileHistorySnapshotMessage>
  attributionSnapshots: Map<UUID, AttributionSnapshotMessage>
  contentReplacements: Map<UUID, ContentReplacementRecord[]>
  agentContentReplacements: Map<AgentId, ContentReplacementRecord[]>
  contextCollapseCommits: ContextCollapseCommitEntry[]
  contextCollapseSnapshot
  leafUuids: Set<UUID>
}
```

读取不是单纯全量 parse。大文件会先做优化：

```txt
readTranscriptForLoad()
  跳过 compact boundary 之前的大段 stale content
  跳过 attribution snapshot 等高成本内容

scanPreBoundaryMetadata()
  只恢复 compact boundary 前的 session-scoped metadata

walkChainBeforeParse()
  byte-level 先裁掉死分支
```

然后：

```ts
const entries = parseJSONL<Entry>(buf)
```

逐条分流：

```txt
TranscriptMessage → messages Map
summary → summaries
custom-title → customTitles
tag → tags
file-history-snapshot → fileHistorySnapshots
content-replacement → contentReplacements / agentContentReplacements
marble-origami-commit → contextCollapseCommits
marble-origami-snapshot → contextCollapseSnapshot
```

读完后会做两个结构修复：

```ts
applyPreservedSegmentRelinks(messages)
applySnipRemovals(messages)
```

最后计算 leaf：

```txt
parentUuids = 所有消息的 parentUuid
terminalMessages = 不被任何消息当 parent 的消息
leafUuids = terminal 往回找最近的 user/assistant
```

这一步决定 `/resume` 从哪条消息作为最新链尾开始。

---

## 13. parentUuid 链：恢复的核心模型

写入时每条消息记录：

```txt
message.uuid
message.parentUuid
```

恢复时：

```ts
buildConversationChain(messages, leafMessage)
```

算法：

```txt
current = leafMessage
while current:
  push current
  current = messages.get(current.parentUuid)
reverse()
```

这意味着 JSONL 可以保存多条分支：

```txt
root
  ↓
A
  ↓
B
  ↓
C

用户 rewind 到 A 后继续：

root
  ↓
A
  ├─ B → C     stale branch
  └─ D → E     latest branch
```

文件里 B/C 仍存在，因为 JSONL append-only。但 resume 选择最新 leaf E 后，只沿 E → D → A → root 恢复。

这就是为什么 Claude Code 可以支持：

```txt
rewind
fork-session
compact
snip
branch
resume
```

而不需要重写整个 transcript。

---

## 14. 并行 tool_result 的 DAG 修复

`buildConversationChain()` 后还会调用：

```ts
recoverOrphanedParallelToolResults(messages, transcript, seen)
```

背景：

```txt
模型一次 assistant response 里可能并行发出多个 tool_use
streaming 时每个 content_block_stop 可能落成一个 AssistantMessage
这些 assistant message 有不同 uuid，但 message.id 相同
每个 tool_result 的 parentUuid 又指向各自的 assistant uuid
```

这不是单链，而是 DAG。

普通 parent walk 只能沿一条路径走，会丢 sibling tool_use/tool_result。

修复逻辑：

```txt
按 assistant.message.id 找 sibling group
按 tool_result.parentUuid 找 tool_result
如果某个 group 已经在链上
  把 off-chain siblings 和对应 tool_result 插回 anchor 后面
```

这是 transcript 恢复层对 API streaming 拓扑的补偿。

---

## 15. compaction 和 preserved segment

compact boundary 是一种 `system` transcript message，但写入时：

```txt
parentUuid = null
logicalParentUuid = compact 前 parent
```

它会把 resume 链显式截断。

大文件读取时还有 pre-compact skip：

```txt
只读取最后一个 compact boundary 之后的内容
避免 parse 几十/几百 MB 旧上下文
```

但某些 compaction 保留了一个 preserved segment：

```txt
boundary.compactMetadata.preservedSegment
```

这时旧消息的 parentUuid 仍然指向 pre-compact 链，需要：

```ts
applyPreservedSegmentRelinks(messages)
```

它会做三件事：

```txt
1. 验证 preserved tail → head 是否完整
2. 把 preserved head 接到 anchor
3. 把 anchor 的其他 child 接到 preserved tail
4. 删除 absolute-last-boundary 之前不需要的旧消息
```

还会把 preserved assistant 的 usage 清零：

```txt
避免 resume 后立即因为旧 input_tokens 触发 auto-compact spiral
```

---

## 16. snip 的链修复

snip 和 compact 不同。

compact 通常截断前缀：

```txt
[old prefix] [boundary] [new chain]
```

snip 删除中间段：

```txt
A → B → C → D → E
删除 C/D
需要 A → B → E
```

因为 JSONL append-only，C/D 仍在文件里。恢复时如果不处理，parent walk 会把被删内容又带回来。

修复函数：

```ts
applySnipRemovals(messages)
```

它从 boundary metadata 里拿：

```txt
removedUuids
```

然后：

```txt
1. 删除 removed messages
2. 记录每个 deleted uuid 的 parentUuid
3. 对 survivor，如果 parentUuid 指向 deleted uuid
   就沿 deletedParent 往前找第一个未删除祖先
4. 改写 survivor.parentUuid
```

这保证 snip 后 resume 不会恢复被用户删掉的上下文。

---

## 17. /resume 列表：lite log

`/resume` 不能一上来 parse 所有 JSONL，否则大项目会很慢。

列表加载走 lite path：

```ts
getSessionFilesLite(projectDir)
```

只做：

```txt
readdir
stat
按 mtime 排序
构造 messages: [] 的 LogOption
```

然后 progressive enrich：

```ts
enrichLogs(allLogs, startIndex, count)
```

每个文件只读：

```txt
head 64KB
tail 64KB
```

常量：

```ts
LITE_READ_BUF_SIZE = 65536
```

从 head 提取：

```txt
isSidechain
projectPath
teamName
firstPrompt
agentSetting
```

从 tail 提取：

```txt
lastPrompt
customTitle / aiTitle
summary
tag
gitBranch
pr link
```

所以 `/resume` 首屏不是完整读取 session，而是：

```txt
先 stat-only
再读 head/tail metadata
用户选中后才 loadFullLog()
```

---

## 18. /resume 命令入口

文件：

```txt
src/commands/resume/resume.tsx
```

无参数：

```txt
loadSameRepoMessageLogs(worktreePaths)
  ↓
filterResumableSessions()
  ↓
LogSelector
  ↓
用户选中
  ↓
如果 lite，则 loadFullLog()
  ↓
context.resume(sessionId, fullLog, entrypoint)
```

带 UUID：

```txt
validateUuid(arg)
  ↓
在 same-repo logs 里找
  ↓
找不到则 getLastSessionLog(sessionId) 直接查文件
```

带 title：

```txt
searchSessionsByCustomTitle(arg, { exact: true })
```

跨项目恢复：

```txt
checkCrossProjectResume()
```

如果不是 same-repo worktree，会提示用户到对应目录运行 resume 命令，而不是直接在错误 cwd 下恢复。

---

## 19. loadFullLog：从 lite 到完整对话

`loadFullLog(log)`：

```txt
loadTranscriptFile(log.fullPath)
  ↓
找最新 user/assistant leaf
  ↓
buildConversationChain(messages, leaf)
  ↓
removeExtraFields()
  ↓
填充 LogOption.messages
```

同时恢复 metadata：

```txt
summary
customTitle
tag
agentName
agentColor
agentSetting
mode
worktreeSession
pr link
gitBranch
fileHistorySnapshots
attributionSnapshots
contentReplacements
contextCollapse entries
```

注意：

```txt
forked sessions 可能复制 source chain
root message 的 sessionId 仍是 source session
所以 metadata 要按 leafMessage.sessionId 取
```

这是为什么代码里很多地方都从 leaf 推 sessionId。

---

## 20. loadConversationForResume：统一恢复入口

文件：

```txt
src/utils/conversationRecovery.ts
```

入口：

```ts
loadConversationForResume(source, sourceJsonlFile)
```

source 支持：

```txt
undefined
  --continue，加载最近 session

string
  --resume <sessionId>

LogOption
  /resume picker 已经选好的 log

sourceJsonlFile
  --resume 指向某个 .jsonl 文件
```

流程：

```txt
定位 log/messages
  ↓
lite log 则 loadFullLog()
  ↓
copyPlanForResume()
  ↓
copyFileHistoryForResume()
  ↓
checkResumeConsistency()
  ↓
restoreSkillStateFromMessages()
  ↓
deserializeMessagesWithInterruptDetection()
  ↓
processSessionStartHooks('resume')
  ↓
返回 messages + metadata + interruption state
```

这是非 UI 和 UI 共用的中心化恢复函数。

---

## 21. deserializeMessages：恢复前清理

恢复出来的 messages 不能直接塞回 query。

`deserializeMessagesWithInterruptDetection()` 会处理：

```txt
legacy attachment type migration
非法 permissionMode 清理
未配对 tool_use 过滤
orphaned thinking-only assistant 过滤
whitespace-only assistant 过滤
中断状态检测
必要时追加 NO_RESPONSE_REQUESTED sentinel
```

中断检测：

```txt
最后相关消息是 assistant
  → 认为 turn 已完成

最后相关消息是普通 user
  → interrupted_prompt

最后相关消息是 tool_result
  → 可能 interrupted_turn

最后相关消息是 attachment
  → interrupted_turn
```

如果检测到 interrupted turn：

```txt
插入一个 meta user:
"Continue from where you left off."
```

这样 API 恢复时不会拿到一个结构不完整的末尾。

---

## 22. processResumedConversation：恢复成运行态

文件：

```txt
src/utils/sessionRestore.ts
```

`processResumedConversation()` 做的是“把加载出来的 LogOption 变成当前进程状态”。

关键步骤：

```txt
1. coordinator/normal mode 匹配
2. 非 fork 时 switchSession(resumedSessionId, transcriptDir)
3. resetSessionFilePointer()
4. restoreCostStateForSession()
5. fork 时复制 content-replacement 到新 session
6. restoreSessionMetadata()
7. restoreWorktreeForResume()
8. adoptResumedSessionFile()
9. restore context-collapse state
10. restore agent setting/model
11. saveMode()
12. compute initial AppState
```

非 fork：

```ts
switchSession(asSessionId(sid), transcriptPath ? dirname(transcriptPath) : null)
resetSessionFilePointer()
restoreSessionMetadata(result)
restoreWorktreeForResume(result.worktreeSession)
adoptResumedSessionFile()
```

fork：

```txt
保留新的 startup sessionId
把原 session 的 messages 写入新 transcript
复制 contentReplacement，避免 budget replacement 状态丢失
不继承原 session 的 worktree ownership
```

---

## 23. REPL 中的 mid-session /resume

交互式 `/resume` 不完全走 startup path。

REPL 中 `context.resume` 会：

```txt
SessionEnd hooks for 当前 session
processSessionStartHooks('resume')
copyPlanForResume / copyPlanForFork
restoreSessionStateFromLog()
restoreAgentFromSession()
restoreReadFileState()
reset loading / abort state
保存当前 session cost
switchSession()
resetSessionFilePointer()
clearSessionMetadata()
restoreSessionMetadata()
exitRestoredWorktree()
restoreWorktreeForResume()
adoptResumedSessionFile()
restoreRemoteAgentTasks()
```

这里一个细节很重要：

```txt
clearSessionMetadata() 必须先调用
```

因为 `restoreSessionMetadata()` 是“有值才覆盖”，如果不清理，新的 session 没有 agentName 时，会继承旧 session 的 agentName，并在退出时写回错误 transcript。

---

## 24. adoptResumedSessionFile

恢复后必须把 `Project.sessionFile` 指向已有 transcript：

```ts
export function adoptResumedSessionFile(): void {
  const project = getProject()
  project.sessionFile = getTranscriptPath()
  project.reAppendSessionMetadata(true)
}
```

为什么不能等第一条新 user message 再 lazy materialize？

```txt
如果用户 --continue / --resume 后直接退出
没有新消息触发 materialize
cleanup 的 reAppendSessionMetadata 会因为 sessionFile=null 直接返回
导致 --name、mode、tag 等 metadata 没有写回
```

所以 resume 后要主动 adopt 已存在文件。

---

## 25. session metadata 的尾部重写

`reAppendSessionMetadata()` 会把缓存的 metadata 再追加到文件尾部：

```txt
last-prompt
custom-title
tag
agent-name
agent-color
agent-setting
mode
worktree-state
pr-link
```

原因：

```txt
/resume 的 lite metadata 只读文件尾部 64KB
如果 title/tag 很早写入，后来大量消息把它挤出 tail window
/resume 列表就显示不出用户命名
```

所以它在两个时机会被调用：

```txt
compaction 前后
进程退出 cleanup
resume adopt 时
```

它还会先读尾部，吸收外部 SDK 写入的更新：

```txt
custom-title / tag 可能由外部进程写
tail 中的新值比内存 cache 更可信
```

---

## 26. worktree session 状态

worktree 状态 entry：

```ts
type WorktreeStateEntry = {
  type: 'worktree-state'
  sessionId: UUID
  worktreeSession: PersistedWorktreeSession | null
}
```

语义：

```txt
undefined
  从未进入 worktree

object
  当前 session 最后处在该 worktree

null
  曾经进入，但后来退出了
```

恢复：

```ts
restoreWorktreeForResume(worktreeSession)
```

如果当前已经有 fresh worktree：

```txt
优先 fresh worktree
并把 fresh state 写回
```

否则：

```txt
尝试 chdir 到 transcript 记录的 worktreePath
目录不存在则 saveWorktreeState(null)
成功则 setCwd / setOriginalCwd / restoreWorktreeSession
清 prompt/memory/plans cache
```

mid-session `/resume` 前还会：

```ts
exitRestoredWorktree()
```

避免从 worktree session 切到非 worktree session 后 cwd 残留。

---

## 27. file history / attribution / todos 恢复

session 不是只恢复消息，还恢复附属状态。

写入：

```ts
recordFileHistorySnapshot()
recordAttributionSnapshot()
```

恢复：

```ts
restoreSessionStateFromLog(result, setAppState)
```

包括：

```txt
fileHistoryRestoreStateFromLog()
attributionRestoreStateFromLog()
contextCollapse restoreFromEntries()
TodoWrite state extraction
```

TodoWrite 在非 v2 task 模式下从 transcript 里扫描最后一个 `TodoWrite` tool_use：

```ts
extractTodosFromTranscript(messages)
```

这说明 transcript 也是 AppState 的部分 source of truth。

---

## 28. content-replacement 持久化

content replacement 用于记录某些大内容被替换成 stub 的决策。

entry：

```ts
type ContentReplacementEntry = {
  type: 'content-replacement'
  sessionId: UUID
  agentId?: AgentId
  replacements: ContentReplacementRecord[]
}
```

主线程：

```txt
agentId absent
  → 写主 session file
  → /resume 按 sessionId 恢复
```

subagent：

```txt
agentId present
  → 写 agent transcript file
  → AgentTool resume 按 agentId 恢复
```

fork-session 特殊处理：

```txt
fork 使用新的 sessionId
原 transcript 里的 content-replacement entry 仍 keyed by 旧 sessionId
所以 processResumedConversation 在 fork path 里重新 recordContentReplacement()
```

否则恢复后会找不到 replacement 记录，导致大内容重新进入上下文。

---

## 29. subagent transcript

第八步已经看到：

```ts
recordSidechainTranscript(messages, agentId, startingParentUuid)
```

底层仍是：

```ts
insertMessageChain(..., isSidechain=true, agentId)
```

读取：

```ts
getAgentTranscript(agentId)
```

流程：

```txt
loadTranscriptFile(agentFile)
  ↓
filter msg.agentId === agentId && msg.isSidechain
  ↓
找最新 leaf
  ↓
buildConversationChain()
  ↓
过滤回该 agentId
  ↓
remove isSidechain / parentUuid
```

还有：

```ts
loadSubagentTranscripts(agentIds)
loadAllSubagentTranscriptsFromDisk()
```

后者直接扫：

```txt
<sessionId>/subagents/agent-*.jsonl
```

用于 task 被内存淘汰后仍能从磁盘读回 subagent transcript。

---

## 30. session list 的过滤

`/resume` 不显示所有 JSONL。

过滤规则包括：

```txt
当前 session 不显示
sidechain 不显示
teamName 存在的不显示
metadata-only session 尽量过滤
```

代码：

```ts
filterResumableSessions(logs, currentSessionId)
```

```ts
enrichLog()
  if (enriched.isSidechain) return null
  if (enriched.teamName) return null
```

这解释了为什么 subagent transcript 有独立文件，但不会污染 `/resume` 主会话列表。

---

## 31. tombstone 删除

当 streaming 失败留下 orphaned message 时：

```ts
removeTranscriptMessage(targetUuid)
```

底层：

```ts
Project.removeMessageByUuid()
```

优化路径：

```txt
读文件尾部 64KB
找 `"uuid":"<target>"`
如果找到完整行
  truncate 到行首
  重写尾部剩余内容
```

慢路径：

```txt
小于 50MB 才全文件 rewrite
超过则跳过，避免 OOM
```

这仍然遵守 append-only 的大方向，但为“刚写入的坏尾巴”提供了有限删除能力。

---

## 32. resume consistency check

恢复后会检查：

```ts
checkResumeConsistency(chain)
```

它找最近的：

```txt
system subtype = turn_duration
```

里面记录了当时 messageCount。

然后比较：

```txt
expected = checkpoint.messageCount
actual = checkpoint 在恢复链中的 index
delta = actual - expected
```

目的：

```txt
监控 write → load round-trip 是否一致
发现 snip/compact/parallel tool_result 造成的链漂移
```

这不是功能必需，但它说明 sessionStorage 的设计已经把“恢复链和运行时链不一致”当成一类核心风险在监控。

---

## 33. 为什么不是单纯全量数组存储

如果只保存：

```json
{ "messages": [...] }
```

会遇到这些问题：

```txt
1. 每次写入都要 rewrite 大文件
2. streaming 中途 crash 会损坏完整数组
3. rewind/fork/branch 需要复制或删除历史
4. compaction 需要大规模重写
5. subagent 和主 session 难以共享/隔离
6. 大 session 的 /resume 列表会很慢
7. metadata 更新容易被大消息淹没
```

Claude Code 的设计是：

```txt
append-only JSONL
  + parentUuid chain
  + metadata entry
  + lite head/tail scan
  + compaction boundary
  + sidechain files
  + remote hydration
```

代价是 loader 复杂很多，但换来：

```txt
写入快
crash 更可恢复
支持分支
支持大文件优化
支持 subagent 独立恢复
支持远程事件复原
```

---

## 34. 端到端链路

正常写入：

```txt
REPL messages state 变化
  ↓
useLogMessages()
  ↓
recordTranscript()
  ↓
cleanMessagesForLogging()
  ↓
getSessionMessages() dedup
  ↓
Project.insertMessageChain()
  ↓
Project.appendEntry()
  ↓
enqueueWrite()
  ↓
drainWriteQueue()
  ↓
<sessionId>.jsonl
```

`/resume` picker：

```txt
/resume
  ↓
loadSameRepoMessageLogs()
  ↓
getSessionFilesLite()
  ↓
enrichLogs()
  ↓
read head/tail 64KB
  ↓
LogSelector
```

选中恢复：

```txt
LogSelector select
  ↓
loadFullLog()
  ↓
loadTranscriptFile()
  ↓
buildConversationChain()
  ↓
deserializeMessagesWithInterruptDetection()
  ↓
REPL context.resume()
  ↓
switchSession()
  ↓
restoreSessionMetadata()
  ↓
restoreWorktreeForResume()
  ↓
adoptResumedSessionFile()
  ↓
setMessages()
```

`--continue`：

```txt
loadConversationForResume(undefined)
  ↓
loadMessageLogs()
  ↓
选择最近 session
  ↓
同样 loadFullLog / deserialize / processResumedConversation
```

subagent：

```txt
runAgent()
  ↓
recordSidechainTranscript()
  ↓
<sessionId>/subagents/agent-<agentId>.jsonl
  ↓
getAgentTranscript(agentId)
  ↓
buildConversationChain()
```

---

## 35. 关键设计结论

1. Session 文件是事件日志，不是状态快照。

2. `parentUuid` 是恢复正确性的核心，不是附属 metadata。

3. `/resume` 选择的是“最新 leaf 对应的一条链”，不是 JSONL 里的所有消息。

4. `progress` 不参与链，否则 UI 临时状态会破坏恢复。

5. compaction、snip、parallel tool_result 都会改变普通单链假设，所以 loader 有专门修复逻辑。

6. metadata 被设计成 append-only last-wins，并通过 `reAppendSessionMetadata()` 保证 lite scan 可见。

7. subagent sidechain 是独立 JSONL 文件，但复用同一套 transcript loader 和 parent chain 模型。

8. remote session 先 hydrate 到本地 JSONL，再走统一恢复路径。

9. resume 不只是恢复 messages，还恢复 AppState、agent setting、mode、worktree、file history、attribution、context-collapse、content replacement。

10. 这个系统的复杂度来自一个目标：在长期、可中断、可分支、可压缩、可后台运行的 agent 会话里，尽量不丢上下文。

---

## 36. 下一步

到这里，Claude Code 的主链路已经覆盖：

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
```

下一步适合继续拆：

```txt
第十步：Compaction / Context Management 系统
```

原因是 sessionStorage 里已经多次出现：

```txt
compact boundary
context-collapse commit
preserved segment
content replacement
auto-compact
```

这些是 Claude Code 能长期运行、不爆 context window 的核心机制。
