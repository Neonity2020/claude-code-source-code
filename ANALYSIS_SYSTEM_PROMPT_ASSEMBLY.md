# Claude Code 源码解剖：System Prompt 组装系统

## 1. 总览

前十步已经串到 Compaction / Context Management：

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
  ↓
Compaction / Context Management
```

第十一步拆 System Prompt 组装。这是 Claude Code 最核心的子系统之一——它决定了模型看到什么、怎么行为、知道哪些上下文。

Claude Code 的 system prompt 不是一段硬编码的文本，而是一个**分层的、有缓存策略的、会话感知的数组拼接管道**：

```txt
静态层 (跨组织可缓存)
  身份声明 + 行为规范 + 工具使用指南 + 风格约束
    ↓
── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ──
    ↓
动态层 (每会话/每轮重新计算)
  session guidance + memory + env + language + MCP instructions + scratchpad
    ↓
systemContext 追加
  git status + cache breaker
    ↓
userContext 前置
  CLAUDE.md 文件链 + currentDate
```

核心文件：

```txt
src/constants/prompts.ts                  # 主 prompt 构建器，~915 行
src/constants/systemPromptSections.ts     # section 缓存框架
src/utils/systemPrompt.ts                 # buildEffectiveSystemPrompt 优先级路由
src/utils/systemPromptType.ts             # SystemPrompt 品牌类型
src/utils/queryContext.ts                 # fetchSystemPromptParts 入口
src/context.ts                            # userContext + systemContext 构建
src/utils/claudemd.ts                     # CLAUDE.md 文件发现与加载，~1479 行
src/memdir/memdir.ts                      # 自动记忆 prompt 构建
src/utils/api.ts                          # splitSysPromptPrefix 缓存分割
src/services/api/claude.ts                # buildSystemPromptBlocks → API blocks
src/constants/outputStyles.ts             # 输出风格配置
```

一句话：Claude Code 的 system prompt 是"**静态行为指令 + 动态上下文注入 + prompt cache 三层分割**"的流水线。前半段是跨组织可缓存的通用行为规范，后半段是每轮可能变化的会话状态，两者通过一个边界标记分离，分别送到 API 的不同 cache scope。

---

## 2. 整体调用链

从 `QueryEngine.ts` 的 `ask()` 方法开始：

```txt
QueryEngine.ask()
  │
  ├── fetchSystemPromptParts()            // 并行获取三个上下文块
  │     ├── getSystemPrompt()             // → 默认 system prompt 数组
  │     ├── getUserContext()              // → CLAUDE.md + currentDate
  │     └── getSystemContext()            // → git status + cache breaker
  │
  ├── buildEffectiveSystemPrompt()        // 优先级路由：override > coordinator > agent > custom > default
  │
  └── query()
        ├── appendSystemContext()          // systemContext 追加到 systemPrompt 末尾
        ├── prependUserContext()           // userContext 插入 messages 首条之前
        │
        ├── splitSysPromptPrefix()         // 按 DYNAMIC_BOUNDARY 分割 → cache 分组
        └── buildSystemPromptBlocks()      // → TextBlockParam[] 带 cache_control
```

最终送到 Anthropic API 的 system 参数是一个 `TextBlockParam[]`，每个 block 可以带独立的 `cache_control.scope`：

```ts
// global scope: 跨组织共享缓存
{ text: "静态行为指令...", cache_control: { scope: 'global' } }
// org scope: 组织内缓存
{ text: "其余内容...", cache_control: { scope: 'org' } }
// null scope: 不缓存
{ text: "动态内容...", cache_control: undefined }
```

---

## 3. getSystemPrompt()：默认 prompt 的十一个 section

文件：`src/constants/prompts.ts`

`getSystemPrompt()` 是默认 prompt 的主构建函数。它返回一个 `string[]` 数组，每个元素是一个独立的 section。以下是完整的 section 列表和位置关系：

### 3.1 静态层（DYNAMIC_BOUNDARY 之前）

这些内容对所有用户都一样（除了 `USER_TYPE === 'ant'` 的内部增强），适合跨组织缓存：

| 序号 | Section | 函数 | 作用 |
|------|---------|------|------|
| 1 | 身份声明 | `getSimpleIntroSection()` | "You are an interactive agent..." + 网络安全约束 |
| 2 | System | `getSimpleSystemSection()` | 工具执行、权限提示、system-reminder、hooks、自动压缩 |
| 3 | Doing tasks | `getSimpleDoingTasksSection()` | 编码规范、注释哲学、安全约束、反馈渠道 |
| 4 | Actions | `getActionsSection()` | 风险评估、确认策略、破坏性操作保护 |
| 5 | Using your tools | `getUsingYourToolsSection()` | 工具优先级（Read > cat, Edit > sed）、并行调用、task 工具 |
| 6 | Tone and style | `getSimpleToneAndStyleSection()` | 无 emoji、file:line 引用、GitHub issue 格式 |
| 7 | Output efficiency | `getOutputEfficiencySection()` | 内部用户：详细散文式；外部用户：极简直接 |

### 3.2 内部 vs 外部的 prompt 差异

`USER_TYPE === 'ant'` 条件编译是一个贯穿整个 prompt 的分流器。内部用户（Anthropic 员工）的 prompt 包含以下外部用户看不到的额外内容：

**Doing tasks section 的内部增强：**

```ts
// 内部用户多了四条规则：
"Default to writing no comments. Only add one when the WHY is non-obvious..."
"Don't explain WHAT the code does, since well-named identifiers already do that..."
"Don't remove existing comments unless you're removing the code they describe..."
"Before reporting a task complete, verify it actually works..."

// 以及：
"If you notice the user's request is based on a misconception, say so..."
"Report outcomes faithfully: if tests fail, say so with the relevant output..."
"If the user reports a bug with Claude Code itself, recommend /issue or /share..."
```

**Output efficiency section 的内部增强：**

内部用户获得一段~200词的详细沟通指南，包括：
- 流畅散文、完整句子、无碎片化表达
- 避免语义回溯（每句话线性可读）
- 倒金字塔结构（先说行动，再说原因）
- 根据用户专业度调整详细程度

外部用户只得到：

```
Go straight to the point. Try the simplest approach first.
Keep your text output brief and direct.
```

**Ant-only 的长度锚点：**

```ts
systemPromptSection('numeric_length_anchors', () =>
  'Length limits: keep text between tool calls to ≤25 words. Keep final responses to ≤100 words...'
)
```

### 3.3 动态层（DYNAMIC_BOUNDARY 之后）

通过 `systemPromptSection()` 注册，有缓存机制。在 `/clear` 或 `/compact` 时清空：

| Section | 缓存策略 | 计算内容 |
|---------|---------|---------|
| `session_guidance` | 缓存 | 工具可用性→Agent/Skill/DiscoverSkills/VerificationAgent 指导 |
| `memory` | 缓存 | `loadMemoryPrompt()` → 自动记忆系统行为指南 |
| `ant_model_override` | 缓存 | 内部模型覆盖的 system prompt 后缀 |
| `env_info_simple` | 缓存 | 工作目录、git 状态、平台、模型名、知识截止日期 |
| `language` | 缓存 | 用户语言偏好设置 |
| `output_style` | 缓存 | Explanatory/Learning 等输出风格 prompt |
| `mcp_instructions` | **不缓存** | MCP 服务器指令（连接可能每轮变化） |
| `scratchpad` | 缓存 | 临时目录指令 |
| `frc` | 缓存 | Function Result Clearing（microcompact 配置） |
| `summarize_tool_results` | 缓存 | 工具结果可能被清除的提醒 |
| `numeric_length_anchors` | 缓存 | Ant-only 数值长度限制 |
| `token_budget` | 缓存 | Token 预算模式指令 |
| `brief` | 缓存 | KAIROS brief 模式指令 |

---

## 4. systemPromptSection 缓存框架

文件：`src/constants/systemPromptSections.ts`

```ts
type SystemPromptSection = {
  name: string
  compute: () => string | null | Promise<string | null>
  cacheBreak: boolean   // true = 每轮重算，会打破 prompt cache
}
```

两种 section 类型：

```ts
// 缓存型：计算一次，缓存到 /clear 或 /compact
systemPromptSection('memory', () => loadMemoryPrompt())

// 非缓存型：每轮重算，变化时会打破 prompt cache
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns'  // 必须说明为什么需要破坏缓存
)
```

解析时先查缓存：

```ts
async function resolveSystemPromptSections(sections) {
  const cache = getSystemPromptSectionCache()
  return Promise.all(sections.map(async s => {
    if (!s.cacheBreak && cache.has(s.name)) {
      return cache.get(s.name) ?? null  // 命中缓存
    }
    const value = await s.compute()     // 重新计算
    setSystemPromptSectionCacheEntry(s.name, value)
    return value
  }))
}
```

清空时机：`/clear` 和 `/compact` 命令触发 `clearSystemPromptSections()`。

---

## 5. buildEffectiveSystemPrompt：优先级路由

文件：`src/utils/systemPrompt.ts`

这个函数决定"到底用哪套 prompt"。优先级从高到低：

```txt
0. overrideSystemPrompt    → 直接返回，忽略一切其他 prompt
1. Coordinator 模式        → 内部协调器 prompt
2. Proactive/KAIROS 模式   → 自主代理 prompt + agent 指令追加
3. Agent prompt            → 替换默认 prompt（主线程 agent）
4. customSystemPrompt      → --system-prompt 参数
5. defaultSystemPrompt     → getSystemPrompt() 的完整输出
+
appendSystemPrompt         → 始终追加到末尾（除 override 模式）
```

代码路径：

```ts
function buildEffectiveSystemPrompt({...}): SystemPrompt {
  // 0. Override：loop 模式等场景，完全替换
  if (overrideSystemPrompt) {
    return asSystemPrompt([overrideSystemPrompt])
  }

  // 1. Coordinator 模式：多代理协调器
  if (feature('COORDINATOR_MODE') && isEnvTruthy('CLAUDE_CODE_COORDINATOR_MODE')) {
    return asSystemPrompt([getCoordinatorSystemPrompt(), ...append])
  }

  // 2. Proactive 模式：agent 指令追加到默认 prompt（不替换）
  if (agentSystemPrompt && isProactiveActive()) {
    return asSystemPrompt([...defaultSystemPrompt, agentSystemPrompt, ...append])
  }

  // 3/4/5. 正常模式：agent prompt 替换默认，或 custom 替换，或用默认
  return asSystemPrompt([
    ...(agentSystemPrompt ?? customSystemPrompt ?? defaultSystemPrompt),
    ...append,
  ])
}
```

---

## 6. Proactive / KAIROS 模式的精简 prompt

当 `PROACTIVE` 或 `KAIROS` feature gate 开启且处于活跃状态时，`getSystemPrompt()` 走一条完全不同的路径：

```ts
// 只有极简的几行
return [
  "You are an autonomous agent. Use the available tools to do useful work.",
  getSystemRemindersSection(),
  await loadMemoryPrompt(),
  envInfo,
  getLanguageSection(settings.language),
  getMcpInstructionsSection(mcpClients),
  getScratchpadInstructions(),
  getFunctionResultClearingSection(model),
  SUMMARIZE_TOOL_RESULTS_SECTION,
  getProactiveSection(),   // <tick> 心跳、自主行为、偏行动
]
```

`getProactiveSection()` 有一套完整的自主代理行为规范：

```txt
- 接收 <tick> 心跳提示，当作"你醒了，做什么？"
- 用 SleepTool 控制节奏，5分钟 cache TTL 内平衡 API 开销
- 无事可做必须调 SleepTool，禁止输出 "still waiting"
- 首次唤醒简短问候，不主动探索
- 后续唤醒主动找事做，不重复问用户
- 偏向行动：读文件、搜代码、改代码、提交，不等确认
- terminalFocus 字段：用户不在→高度自主；用户在看→更协作
```

---

## 7. 环境信息 section

`computeSimpleEnvInfo()` 构建 `<env>` 块：

```txt
# Environment
You have been invoked in the following environment:
 - Primary working directory: /Users/zion/project
 - Is a git repository: true
 - Platform: darwin
 - Shell: zsh
 - OS Version: Darwin 25.3.0
 - You are powered by the model named Claude Sonnet 4.6. The exact model ID is claude-sonnet-4-6.
 - Assistant knowledge cutoff is August 2025.
 - The most recent Claude model family is Claude 4.5/4.6...
 - Claude Code is available as a CLI in the terminal, desktop app...
 - Fast mode for Claude Code uses the same Claude Opus 4.6 model with faster output...
```

**卧底模式**下模型信息全部抑制：

```ts
if (process.env.USER_TYPE === 'ant' && isUndercover()) {
  // 抑制所有模型名/ID、知识截止日期、fast mode 信息
}
```

知识截止日期按模型映射：

```ts
claude-sonnet-4-6 → 'August 2025'
claude-opus-4-6   → 'May 2025'
claude-haiku-4    → 'February 2025'
```

---

## 8. userContext：CLAUDE.md 文件发现与加载

文件：`src/utils/claudemd.ts`（1479 行），`src/context.ts`

### 8.1 文件发现顺序（优先级从低到高）

```txt
1. 托管记忆 (/etc/claude-code/CLAUDE.md)           — 全局管理员指令
2. 用户记忆 (~/.claude/CLAUDE.md)                   — 个人全局指令
3. 项目记忆 (CLAUDE.md / .claude/CLAUDE.md / .claude/rules/*.md) — 项目级
4. 本地记忆 (CLAUDE.local.md)                        — 私有项目指令（不提交）
```

加载从 cwd 向上遍历到根目录，**越近 cwd 的文件优先级越高（后加载）**：

```txt
/project/root/
  CLAUDE.md               ← 优先级低
  .claude/CLAUDE.md        ← 优先级中
  .claude/rules/*.md       ← 优先级中（每个 .md 文件）
  src/
    CLAUDE.md              ← 优先级高（更接近 cwd）
```

### 8.2 @include 指令

CLAUDE.md 文件支持 `@path` 语法包含其他文件：

```ts
// 语法：@path, @./relative/path, @~/home/path, @/absolute/path
// 只在叶文本节点生效（不在代码块内）
// 被包含文件作为独立条目插到包含文件之前
// 循环引用通过 processed files 集合防止
```

支持 80+ 种文件扩展名的 include。

### 8.3 上下文组装

```ts
// context.ts
getUserContext = memoize(async () => {
  const claudeMd = getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))
  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

`getUserContext` 是 **memoize** 的——每个会话只计算一次。结果被 `prependUserContext()` 插到 messages 数组的第一条用户消息之前。

### 8.4 systemContext

```ts
getSystemContext = memoize(async () => {
  const gitStatus = await getGitStatus()  // branch, main branch, status, recent 5 commits
  return {
    ...(gitStatus && { gitStatus }),
    ...(injection && { cacheBreaker: `[CACHE_BREAKER: ${injection}]` }),
  }
})
```

`getSystemContext` 同样是 memoize 的。结果被 `appendSystemContext()` 追加到 systemPrompt 末尾。

---

## 9. 自动记忆系统的 prompt

文件：`src/memdir/memdir.ts`

`loadMemoryPrompt()` 构建自动记忆行为指南，注入到动态层的 `memory` section：

```txt
# auto memory
You have a persistent, file-based memory system at `~/.claude/projects/.../memory/`.

## Types of memory
  user      — 用户角色、偏好、知识
  feedback  — 行为指导（做/不做）
  project   — 项目信息（不能从代码推导的）
  reference — 外部系统指针

## What NOT to save
  代码模式、架构、文件路径、git 历史、调试方案...（可从项目推导的）

## How to save memories
  Step 1 — 写记忆文件 (user_role.md) 带 frontmatter
  Step 2 — 在 MEMORY.md 索引中添加指针

## When to access memories
  相关时读取；用户要求时检查；用户说忽略时不使用

## Before recommending from memory
  验证记忆中引用的文件/函数是否仍然存在

## MEMORY.md
  (实际内容或 "Your MEMORY.md is currently empty.")
```

MEMORY.md 索引有硬限制：**200 行 / 25KB**，超出会截断并追加警告。

---

## 10. Prompt Cache 三层分割

文件：`src/utils/api.ts` → `splitSysPromptPrefix()`

这是 prompt cache 的核心——将 system prompt 数组分割成不同 cache scope 的 block：

```txt
┌─────────────────────────────────────────────────┐
│ attribution header (x-anthropic-billing-header)  │  cacheScope: null (不缓存)
├─────────────────────────────────────────────────┤
│ CLI_SYSPROMPT_PREFIXES                           │  cacheScope: null
├─────────────────────────────────────────────────┤
│ 静态 sections (join 所有 DYNAMIC_BOUNDARY 前的)   │  cacheScope: 'global' (跨组织缓存)
├─────────────────────────────────────────────────┤
│ 动态 sections (join 所有 DYNAMIC_BOUNDARY 后的)   │  cacheScope: null (不缓存)
└─────────────────────────────────────────────────┘
```

当全局缓存功能关闭（`shouldUseGlobalCacheScope() === false`）时：

```txt
┌─────────────────────────────────────────────────┐
│ attribution header                               │  cacheScope: null
├─────────────────────────────────────────────────┤
│ CLI_SYSPROMPT_PREFIXES                           │  cacheScope: 'org'
├─────────────────────────────────────────────────┤
│ 其余所有内容 (join)                               │  cacheScope: 'org'
└─────────────────────────────────────────────────┘
```

当有 MCP 工具时（每个用户的 MCP 不同），全局缓存被跳过，退化为 `org` scope：

```ts
if (useGlobalCacheFeature && options?.skipGlobalCacheForSystemPrompt) {
  // 不使用 global scope，全部用 org scope
}
```

---

## 11. 最终 API 调用形态

所有 section 在 `buildSystemPromptBlocks()` 中被转换为 Anthropic API 的 `TextBlockParam[]`：

```ts
function buildSystemPromptBlocks(systemPrompt, enablePromptCaching, options) {
  return splitSysPromptPrefix(systemPrompt, options).map(block => ({
    type: 'text',
    text: block.text,
    ...(enablePromptCaching && block.cacheScope !== null && {
      cache_control: getCacheControl({
        scope: block.cacheScope,        // 'global' | 'org'
        querySource: options?.querySource,
      }),
    }),
  }))
}
```

最终送达 API 的形状：

```json
{
  "system": [
    { "type": "text", "text": "attribution header..." },
    { "type": "text", "text": "CLI prefix..." },
    { "type": "text", "text": "静态行为指令...",
      "cache_control": { "scope": "global" } },
    { "type": "text", "text": "动态上下文 (env, memory, MCP)...",
      "cache_control": { "scope": "org" } }
  ],
  "messages": [
    { "role": "user", "content": "claudeMd: ...\ncurrentDate: ..." },
    { "role": "user", "content": "用户的实际输入" },
    ...
  ]
}
```

---

## 12. Subagent 的 prompt 路径

Subagent 不走 `getSystemPrompt()` 的完整流程。它通过 `enhanceSystemPromptWithEnvDetails()` 在 agent 的基础 prompt 上追加：

```ts
async function enhanceSystemPromptWithEnvDetails(
  existingSystemPrompt,  // agent 定义的 prompt（如 DEFAULT_AGENT_PROMPT）
  model,
  additionalWorkingDirectories,
  enabledToolNames,
) {
  return [
    ...existingSystemPrompt,
    notes,                    // 绝对路径、无 emoji、无冒号
    discoverSkillsGuidance,   // 技能发现指导（如果可用）
    envInfo,                  // 完整 <env> 块
  ]
}
```

Subagent 的 prompt 比 main session 精简得多——没有 "Doing tasks"、"Actions"、"Tone and style" 这些行为规范，只有身份声明 + 环境信息 + 基本注意事项。

---

## 13. 输出风格（Output Styles）

文件：`src/constants/outputStyles.ts`

用户可以通过设置选择不同的输出风格，每种风格注入不同的 prompt section：

| 风格 | 行为 |
|------|------|
| `default` | null（无额外 prompt） |
| `Explanatory` | 教育洞察 + 实现选择解释 + `keepCodingInstructions: true` |
| `Learning` | 暂停并让用户写代码练习 |

当 `outputStyleConfig !== null && keepCodingInstructions !== true` 时，"Doing tasks" section 被移除：

```ts
outputStyleConfig === null || outputStyleConfig.keepCodingInstructions === true
  ? getSimpleDoingTasksSection()   // 保留编码规范
  : null                           // 输出风格替换编码规范
```

---

## 14. 完整的 system prompt 生命周期

```txt
会话启动
  │
  ├── QueryEngine.ask()
  │     ├── fetchSystemPromptParts()
  │     │     ├── getSystemPrompt() ──── 计算静态+动态 sections
  │     │     ├── getUserContext()  ──── 读取 CLAUDE.md 文件链
  │     │     └── getSystemContext() ──── git status 快照
  │     │
  │     ├── buildEffectiveSystemPrompt()
  │     │     └── 按 override > coordinator > agent > custom > default 路由
  │     │
  │     └── asSystemPrompt([default/custom, memory, append])
  │
  ├── query() 首次循环
  │     ├── appendSystemContext(prompt, systemContext)
  │     ├── autocompact() 检查是否需要压缩
  │     ├── prependUserContext(messages, userContext)
  │     ├── splitSysPromptPrefix() ──── 分割 cache 层
  │     └── buildSystemPromptBlocks() ── 生成 API blocks
  │
  └── 后续循环
        ├── 动态 sections 查缓存（MCP 指令除外）
        ├── compact 后 clearSystemPromptSections() 清缓存
        └── 重新解析动态 sections
```

---

## 15. 关键设计决策总结

1. **数组而非字符串**：system prompt 是 `string[]` 而非单个 string，便于分段缓存和条件组装。

2. **DYNAMIC_BOUNDARY 分割**：静态内容跨组织缓存（`global` scope），动态内容不缓存或组织级缓存。这节省了大量重复 prompt 的 token 开销。

3. **内部/外部双轨 prompt**：`USER_TYPE === 'ant'` 条件编译让 Anthropic 内部用户获得更详细、更严格的 prompt（验证、忠实报告、沟通风格），外部用户获得更简洁的指令。这不是配置差异，而是编译时 DCE。

4. **memoize 的上下文**：`getUserContext` 和 `getSystemContext` 都是 lodash memoize 的，整个会话只计算一次。这意味着 CLAUDE.md 的修改需要新会话才能生效。

5. **section 缓存 vs 轮次重算**：大部分动态 section 缓存到 `/clear` 或 `/compact`，只有 MCP 指令每轮重算（`DANGEROUS_uncachedSystemPromptSection`）。

6. **卧底模式信息抑制**：当 Anthropic 员工在公开仓库工作时，模型名、知识截止日期、产品信息全部从 env section 移除，防止泄露到 commit 消息中。

7. **MEMORY.md 的硬限制**：200 行 / 25KB，超限截断。这是为了防止记忆系统无限膨胀吃掉上下文窗口。

8. **Subagent prompt 极简化**：子代理只获得身份 + 环境 + 基本注意事项，不继承主 session 的行为规范。这减少了子代理的 token 开销，但也意味着子代理的行为约束更少。
