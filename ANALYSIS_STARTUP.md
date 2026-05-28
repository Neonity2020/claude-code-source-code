# Claude Code 源码解剖：启动链路

## 1. 总入口：`src/entrypoints/cli.tsx`

这是最外层 bootstrap，目标是 **先处理快路径，避免加载完整 CLI**。

关键逻辑：

```ts
async function main(): Promise<void> {
  const args = process.argv.slice(2)

  if (args.length === 1 && (args[0] === '--version' ...)) {
    console.log(`${MACRO.VERSION} (Claude Code)`)
    return
  }

  // 各种 fast-path:
  // --dump-system-prompt
  // --claude-in-chrome-mcp
  // --chrome-native-host
  // remote-control / bridge
  // daemon
  // bg sessions
  // templates
  // worktree + tmux
  // ...

  startCapturingEarlyInput()
  const { main: cliMain } = await import('../main.js')
  await cliMain()
}
```

### 观察

Claude Code 很重，所以它做了大量 **动态 import 延迟加载**：

- `--version`：几乎零依赖启动
- `remote-control`、`daemon`、`bg`、`chrome`、`mcp` 等单独快路径
- 普通交互模式才进入 `src/main.tsx`

也就是说：

```txt
bin/cli
  ↓
src/entrypoints/cli.tsx
  ↓ 动态 import
src/main.tsx
```

---

## 2. 主 CLI：`src/main.tsx`

这是最核心的启动编排文件，非常大，约 4k+ 行。

主函数：

```ts
export async function main() {
  process.env.NoDefaultCurrentDirectoryInExePath = '1'

  initializeWarningHandler()

  // 处理 direct connect / assistant / ssh 等特殊 argv rewrite

  const cliArgs = process.argv.slice(2)
  const hasPrintFlag = cliArgs.includes('-p') || cliArgs.includes('--print')
  const isNonInteractive = hasPrintFlag || ... || !process.stdout.isTTY

  setIsInteractive(!isNonInteractive)
  initializeEntrypoint(isNonInteractive)

  eagerLoadSettings()

  await run()
}
```

### 重要点

它先判断当前是：

- 交互 TUI：`claude`
- 非交互/headless：`claude -p "..."` 或 stdout 非 TTY
- SDK 模式
- remote / ssh / assistant 等特殊模式

然后进入 `run()`，用 Commander 注册所有命令和参数。

---

## 3. Commander 注册和默认 action

`run()` 里创建 Commander：

```ts
const program = new CommanderCommand()
```

然后注册主命令：

```ts
program
  .name('claude')
  .description('Claude Code - starts an interactive session by default...')
  .argument('[prompt]', 'Your prompt', String)
  .option('-p, --print', ...)
  .option('--output-format <format>', ...)
  .option('--allowed-tools <tools...>', ...)
  .option('--permission-mode <mode>', ...)
  .option('--model <model>', ...)
  .option('--mcp-config <configs...>', ...)
  ...
  .action(async (prompt, options) => {
    ...
  })
```

也就是说默认用法：

```bash
claude
claude "fix this bug"
claude -p "explain this file"
```

都走这个 `.action(...)`。

---

## 4. 启动前置 hook：`program.hook('preAction')`

在真正进入 action 之前，会执行：

```ts
program.hook('preAction', async thisCommand => {
  await Promise.all([
    ensureMdmSettingsLoaded(),
    ensureKeychainPrefetchCompleted()
  ])

  await init()

  initSinks()
  runMigrations()

  void loadRemoteManagedSettings()
  void loadPolicyLimits()
})
```

这里完成：

- MDM 设置读取
- Keychain 预取
- `init()`
- 日志/遥测 sinks
- 配置迁移
- 远程 managed settings
- policy limits

---

## 5. 默认 action 内部主分叉

action 里处理大量参数后，核心分成两条路：

### A. Headless / `--print` 路径

判断：

```ts
if (isNonInteractiveSession) {
  ...
  const { runHeadless } = await import('src/cli/print.js')

  void runHeadless(
    inputPrompt,
    () => headlessStore.getState(),
    headlessStore.setState,
    commandsHeadless,
    tools,
    sdkMcpConfigs,
    ...
  )

  return
}
```

即：

```txt
claude -p "..."
  ↓
main.tsx
  ↓
src/cli/print.ts
  ↓
QueryEngine / query
```

### B. 交互 REPL 路径

普通 `claude` 会走：

```ts
await launchRepl(root, {
  getFpsMetrics,
  stats,
  initialState
}, {
  ...sessionConfig,
  initialMessages,
  pendingHookMessages
}, renderAndRun)
```

对应文件：

```ts
src/replLauncher.tsx
```

内容很简单：

```tsx
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')
  const { REPL } = await import('./screens/REPL.js')

  await renderAndRun(root,
    <App {...appProps}>
      <REPL {...replProps} />
    </App>
  )
}
```

所以交互链路是：

```txt
main.tsx
  ↓
launchRepl()
  ↓
components/App.tsx
  ↓
screens/REPL.tsx
```

---

## 6. 初步总启动图

```txt
src/entrypoints/cli.tsx
  ├─ --version / daemon / bridge / bg / chrome / etc 快路径
  └─ import ../main.js
       ↓
src/main.tsx
  ├─ main()
  │   ├─ 判断 print / interactive / sdk / ssh / remote
  │   ├─ eagerLoadSettings()
  │   └─ run()
  │
  └─ run()
      ├─ Commander 注册参数、子命令
      ├─ preAction: init / sinks / migrations / policy / remote settings
      └─ default action
          ├─ setup()
          ├─ showSetupScreens()
          ├─ MCP / tools / commands / agents 初始化
          ├─ if --print:
          │     └─ src/cli/print.ts → runHeadless()
          └─ else:
                └─ launchRepl()
                    └─ App + REPL
```

---

## 7. 第一层结论

Claude Code 的启动设计有几个明显特点：

1. **极端重视启动性能**  
   大量 dynamic import、prefetch、fast-path、profile checkpoint。

2. **交互和 headless 是两套入口，但共享底层 query 逻辑**  
   - 交互：`REPL.tsx`
   - headless：`cli/print.ts`
   - 底层：`query.ts` / `QueryEngine.ts`

3. **启动过程中就做大量策略控制**  
   包括：
   - 权限模式
   - 工作区 trust
   - managed settings
   - policy limits
   - GrowthBook feature flags
   - MCP
   - agents
   - plugins
   - skills
   - telemetry

4. **`main.tsx` 是启动总调度器，不是业务核心**  
   真正的模型调用和工具循环在下一层：  
   `REPL.tsx → query.ts → tools`

---

## 8. 下一步

进入第二层：**用户输入如何从 REPL 进入 query 主循环**。

重点文件：

- `src/screens/REPL.tsx`
- `src/utils/handlePromptSubmit.ts`
- `src/query.ts`
- `src/QueryEngine.ts`
