# Claude Code 源码深度解读报告

> **核心结论**：Claude Code 是单 async generator（query()）驱动的终端产品，REPL 与 headless 共用一条循环；深度依赖 Anthropic 的 prompt cache 与流式 API，权限是规则集 + 工具自检 + AI 分类器 + 多路竞争，UI 是全自研 Ink fork + 原始终端栈。

> 仓库：`/Users/bytedance/Documents/harness/claude-code`（1884 个 TS 文件，Bun 运行时，React + 自研 Ink 终端 UI）
> 方法：实际 read/grep/glob 源码（非文件名臆测）；核心路径均标注「文件:行号/导出符号」。
> 说明：本快照含 ant-only（Anthropic 内部）代码，外部构建通过 `feature()` 编译期剔除；标注 [ant-only] 的能力外部不可见。

---

## 1. 入口与启动

### 1.1 进程入口（`src/entrypoints/cli.tsx`）

真正的 bin 入口。`main()` 先做零成本 fast-path：`--version/-v` 直接打印 `MACRO.VERSION`（构建期内联）不加载任何模块；随后 fast-path 拦截 `--dump-system-prompt`（输出渲染后的 system prompt 退出）、`--handle-uri`（deep link）、`remote-control` 命令（转 `bridge/bridgeMain.ts`）。`entrypoints/mcp.ts` 提供 `startMCPServer()`（`@modelcontextprotocol/sdk` 的 `Server` + `StdioServerTransport`），供 `claude mcp serve` 用。

### 1.2 启动副作用与性能剖析（`src/main.tsx`，4684 行）

main.tsx 顶部注释明示了启动工程学的三条铁律：

- **模块求值阶段的并行 I/O**：`profileCheckpoint('main_tsx_entry')` → `startMdmRawRead()`（plutil/reg query 子进程）→ `startKeychainPrefetch()`（macOS Keychain OAuth + 旧 API key 并行预读），全部在 import 期间以子进程并行跑，掩盖掉约 135ms 的模块加载时间。
- **编译期死代码消除**：`import { feature } from 'bun:bundle'`。`feature('COORDINATOR_MODE') ? require(...) : null` 这类条件 require 是 ant-only 代码的隔离墙——外部构建中不活跃分支连同其字符串一起被 Bun 树摇掉（源码多处注释提到 excluded-strings 检查）。
- **Commander 编排**：`run()` 构建 `new CommanderCommand()`，`preAction` hook 里串起 `ensureMdmSettingsLoaded() → init() → initSinks() → runMigrations() → loadRemoteManagedSettings()/loadPolicyLimits()`。注册约 50 个子命令（见 1.4），默认 action 处理交互/`-p` 两条主路径。

### 1.3 init() 与 setup()（`src/entrypoints/init.ts`、`src/setup.ts`）

`init()`（memoize 幂等）：`enableConfigs()`（放行配置系统）→ `applySafeConfigEnvironmentVariables()`（信任对话框**之前**只应用"安全"env）→ `setupGracefulShutdown()` → `configureGlobalMTLS()/configureGlobalAgents()`（代理）→ `preconnectAnthropicApi()`（预热 TCP+TLS 握手）→ 懒加载 OpenTelemetry（~400KB 延迟到遥测真正初始化时）。`initializeTelemetryAfterTrust()` 在信任确立后才初始化遥测，且远程托管设置用户要先等设置加载再应用 env。

`setup(cwd, permissionMode, ...)`：Node ≥18 校验 → UDS 消息服务（`startUdsMessaging`，agent 间进程通信）→ worktree 创建（`--worktree` 时 `process.chdir`，`saveWorktreeState`）→ `captureHooksConfigSnapshot()`（防 hook 配置被暗中修改）→ `initSessionMemory()` → `lockCurrentVersion()` → 发射 `tengu_started` beacon（会话成功率分母，越早越好）。**`--dangerously-skip-permissions` 安全检查**：root/sudo 拒绝、ant 用户必须在无网 Docker/沙箱容器中。

### 1.4 默认 action 的两条主路径（`main.tsx:1006` 起）

1. **非交互（`-p/--print`）**：`applyConfigEnvironmentVariables()`（print 模式视为已信任）→ 组装 `headlessStore`（`createStore(headlessInitialState, onChangeAppState)`）→ 逐个连接 MCP（`connectMcpBatch`，claude.ai connectors 5s 超时）→ `cli/print.ts` 的 `runHeadless()`。
2. **交互**：`createRoot(renderOptions)`（`ink.ts`）→ `showSetupScreens(root, ...)`（信任对话框 / onboarding / 登录 / resume 选择器）→ LSP manager 初始化（**必须在信任之后**，防止插件 LSP 在未信任目录执行代码）→ 按 `--continue` / `--resume` / `--teleport` / `--remote` / cc:// direct-connect / ssh / assistant / 普通会话 分支进入 `launchRepl(root, ...)` → `<App><REPL/></App>`（`replLauncher.tsx`）。

### 1.5 迁移（`src/migrations/`）

`CURRENT_MIGRATION_VERSION = 11`（`main.tsx:325`）。每次启动若版本不符，同步跑 10 个迁移：`migrateAutoUpdatesToSettings`、`migrateBypassPermissionsAcceptedToSettings`、`migrateEnableAllProjectMcpServersToSettings`、`resetProToOpusDefault`、`migrateSonnet1mToSonnet45`、`migrateLegacyOpusToCurrent`、`migrateSonnet45ToSonnet46`、`migrateOpusToOpus1m`、`migrateReplBridgeEnabledToRemoteControlAtStartup`；异步迁移 `migrateChangelogFromConfig` fire-and-forget。

---

## 2. 查询循环（Query Loop）

### 2.1 两个调用方共用同一个 `query()`

- **交互 REPL**：`screens/REPL.tsx:2793` 直接 `for await (const event of query({...}))`，回调 `setToolJSX`/`addNotification` 等 UI 钩子。
- **无头/SDK**：`QueryEngine.ts`（1295 行）——`class QueryEngine`，每个会话一个实例，`submitMessage()` 是 async generator，`ask()` 是便捷包装。它组装 system prompt（`fetchSystemPromptParts` → `getSystemPrompt(tools, model, dirs, mcpClients)`（`constants/prompts.ts`）+ `getUserContext()/getSystemContext()`（`context.ts`）），经 `processUserInput`（slash 命令处理/模型选择）后进入 `query()`，并把流事件翻译成 SDK 消息（`system_init`、`stream_event`、`result`、`compact_boundary`…），同时负责 `recordTranscript` 持久化、`maxBudgetUsd`/`maxTurns`/结构化输出重试上限的熔断。

### 2.2 上下文组装（`src/context.ts`）

- `getGitStatus()`（memoize）：并行跑 `git status --short`（截断 2000 字符）+ `git log -oneline -5` + `git config user.name`，注明"这是会话开始时的快照，不会更新"。
- `getSystemContext()`（memoize）：`{gitStatus, cacheBreaker?}`，作为 **system 层** append 到 system prompt。
- `getUserContext()`（memoize）：`{claudeMd, currentDate}` —— `getClaudeMds(getMemoryFiles())` 从 `~/.claude/CLAUDE.md`、`./CLAUDE.md` 逐级向上合并 + memory 文件。两者都 memoize 整个会话。
- 组装：`systemPrompt = asSystemPrompt([...defaultSystemPrompt, ...(appendSystemPrompt)])`；请求时 `prependUserContext(messages, userContext)` 把 userContext 插到第一条 user 消息前（prompt cache 前缀的一部分）。

### 2.3 主循环 `queryLoop`（`src/query.ts:241`，1729 行）

`while(true)` 状态机，`state`（messages/toolUseContext/turnCount/autoCompactTracking…）在 `continue` 点整体重建。每次迭代：

1. **上下文预算流水线**（顺序敏感）：`applyToolResultBudget`（工具结果大小预算）→ snip（`HISTORY_SNIP`）→ **microcompact**（`deps.microcompact`，去掉已读文件内容/陈旧附件）→ contextCollapse（`CONTEXT_COLLAPSE`）→ **autocompact**（`deps.autocompact`，见维度 10）。
2. **调用模型**：`deps.callModel({messages: prependUserContext(...), systemPrompt, tools, ...})` → `queryModelWithStreaming`（`services/api/claude.ts:752`）。流式返回 assistant 消息、`tool_use` block；`needsFollowUp = toolUseBlocks.length > 0` 是唯一的循环继续信号。
3. **工具执行**：`streamingToolExecutor ? executor.getRemainingResults() : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)`（`services/tools/toolOrchestration.ts:19`）。`runTools` 用 `partitionToolCalls` 把调用分成"并发安全的只读批"（并发跑，`getMaxToolUseConcurrency()` 默认 10）与"非只读单批"（串行）；`runToolUse`（`toolExecution.ts:337`）→ `streamedCheckPermissionsAndCallTool`（权限 + PreToolUse hooks + call + PostToolUse hooks + 工具结果消息）。
4. **结果回填**：每个工具的执行产物 yield 成 `user` 消息（`tool_result` content block），收集进 `toolResults`；**下一轮** `state.messages = [...messagesForQuery, ...assistantMessages, ...toolResults]` 再进 `callModel` —— 这就是"工具结果如何回到循环"：全部是消息数组拼接，无独立 tool loop。
5. **附件注入**：`getAttachmentMessages(...)`（记忆预取 `startRelevantMemoryPrefetch`、技能发现 `startSkillDiscoveryPrefetch`、队列命令 `getCommandsByMaxPriority`）→ yield 成 attachment 消息进 toolResults。
6. **终止**：无 tool_use → `handleStopHooks`（Stop hooks 可能阻塞/注入）→ 返回 `Terminal {reason}`。还有 maxTurns / max_output_tokens 恢复循环（`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT=3`）/ 模型 fallback（`FallbackTriggeredError` → 换 fallbackModel 重试）/ prompt-too-long 的 reactive compact 恢复。

### 2.4 流式渲染（`services/api/claude.ts`）

`queryModel()` 用 **raw `Stream`**（非 BetaMessageStream，避免 O(n²) partial JSON 解析），SSE 事件 switch（约 L1979）：`message_start`（记 TTFT/usage）、`content_block_start`、`content_block_delta`（text/input_json_delta 累加 `partial_json`、signature_delta、thinking_delta）、`content_block_stop`（构造 AssistantMessage 并 yield）、`message_delta`（**把 usage/stop_reason 回写进已 yield 的消息**，并累计成本）。流失败自动降级 `executeNonStreamingRequest()`（max_tokens 压到 `MAX_NON_STREAMING_TOKENS=64_000`）。REPL 端 `useLogMessages` + `useDeferredValue` 消费这些增量事件刷新 UI。

---

## 3. 工具系统

### 3.1 Tool 类型（`src/Tool.ts:362`）

`Tool<Input, Output, P>` 关键字段：`name/aliases`、`inputSchema`（**Zod v4**）+ 可选的 `inputJSONSchema`（MCP/SyntheticOutput 直传 JSON Schema）、`call(args, ctx, canUseTool, parentMessage, onProgress)`、`description()/prompt()`（生成发给模型的描述）、`isEnabled()/isReadOnly(input)/isConcurrencySafe(input)/isDestructive?(input)`、`interruptBehavior?(): 'cancel'|'block'`、`validateInput?(input, ctx)`（执行前校验）、`checkPermissions(input, ctx): Promise<PermissionResult>`（allow/ask/deny/passthrough，`src/types/permissions.ts`）、`requiresUserInteraction?()`、`isOpenWorld?(input)`、`shouldDefer/alwaysLoad`（ToolSearch 工具集优化）、`maxResultSizeChars`（超限结果持久化到文件，只给模型路径）、`isMcp?/isLsp?/mcpInfo?`、`preparePermissionMatcher?`（hook 规则匹配）、`backfillObservableInput?`（观察者视角补字段，不改动 API 侧输入以保 prompt cache）、`toAutoClassifierInput`（auto 模式分类器输入）、以及整套 `renderToolUseMessage/renderToolResultMessage/renderToolUseProgressMessage/renderToolUseRejectedMessage/...` React 渲染钩子（UI 与逻辑同文件共存）。

`buildTool(def)` + `TOOL_DEFAULTS`（Tool.ts:757）：fail-closed 默认——`isEnabled→true`、`isConcurrencySafe→false`、`isReadOnly→false`、`isDestructive→false`、`checkPermissions→{behavior:'allow'}`。**没有** `needsPermissions` 字段（权限靠 checkPermissions），**没有** isStreamable（流式输出靠 `onProgress: ToolCallProgress<P>` 回调，progress 类型见 `src/types/tools.ts`）。

### 3.2 注册表（`src/tools.ts`）

- `getAllBaseTools()`（L193）：内置工具数组（AgentTool/BashTool/FileRead/Edit/Write/NotebookEdit/WebFetch/WebSearch/SkillTool/EnterPlanMode/ExitPlanModeV2/TodoWrite/AskUserQuestion/TaskStop…）；Glob/Grep 仅在 `!hasEmbeddedSearchTools()`（ant 构建内嵌 bfs/ugrep）时加入；TaskCreate/Get/Update/List 需 `isTodoV2Enabled()`；Team/SendMessage 需 `isAgentSwarmsEnabled()`；其余用 `feature('X')` 条件 require。
- `getTools(permissionContext)`（L271）：`CLAUDE_CODE_SIMPLE` 只留 Bash/Read/Edit → 剔除 3 个特殊工具 → `filterToolsByDenyRules`（deny 规则在**工具列表层面**预剔除，MCP 前缀规则 `mcp__server` 可整组屏蔽）→ `isEnabled()` 过滤。
- `assembleToolPool(permissionContext, mcpTools)`（L345）：内置 + MCP 各自按名排序后 `uniqBy('name')`（内置优先），排序是为了 **prompt cache 稳定**（全局 system cache policy 在最后一个内置工具后打断点）。
- 工具集是 **数组**（`Tools = readonly Tool[]`），不是 map。

### 3.3 关键工具实现要点

| 工具 | 位置 | 要点 |
|---|---|---|
| **BashTool** | `tools/BashTool/BashTool.tsx` + `utils/Shell.ts` | `inputSchema` 含 `command/timeout/description/run_in_background/dangerouslyDisableSandbox` + 内部 `_simulatedSedEdit`（模型侧被 omit）。执行走 `exec()`（Shell.ts:181，Node `child_process.spawn`），stdout/stderr 写 TaskOutput 文件 fd（O_APPEND 交错）或管道；`wrapSpawn` 处理超时/整树 kill/自动后台。权限核心 `bashToolHasPermission`（bashPermissions.ts:1663）：精确规则→前缀/通配→sed 约束→只读放行→沙箱自动允许→prompt 分类器。`isConcurrencySafe === isReadOnly` |
| **FileReadTool** | `tools/FileReadTool/FileReadTool.ts:337` | `{file_path, offset, limit, pages?}`；限额 `fileReadingLimits`（maxTokens/maxSizeBytes）；`maxResultSizeChars: Infinity`（防 Read→文件→Read 循环） |
| **FileWriteTool** | `tools/FileWriteTool/FileWriteTool.ts:56` | 原子写 + FileStateCache 陈旧检测 |
| **FileEditTool** | `tools/FileEditTool/` | `{file_path, old_string, new_string, replace_all?}`；`validateInput` 前置校验（old==new 拒绝、字符串必须唯一、`findActualString` 处理弯引号）；**无 postToolUse 复核机制**（grep 全库无此符号） |
| **Glob/Grep** | `tools/GlobTool`、`tools/GrepTool` | maxResults 默认 100；Grep 内部调 ripgrep（`utils/ripgrep.ts`），head_limit 默认 250 |
| **WebFetchTool** | `tools/WebFetchTool/WebFetchTool.ts:24` | 按 hostname 规则 + `isPreapprovedUrl`；重定向 3xx 返回提示让模型换 URL |
| **WebSearchTool** | `tools/WebSearchTool/WebSearchTool.ts:25` | **套娃实现**：call() 内再起一个 agent 循环（`queryModelWithStreaming`，querySource `web_search_tool`），解析子代理的 WebFetch 调用 |
| **AgentTool** | `tools/AgentTool/` | schema `{description, prompt, subagent_type?, model?, name?, team_name?, mode?, isolation?('worktree'|'remote'), run_in_background?, cwd?}`；checkPermissions 除 auto 外一律 allow；`runAgent()`（runAgent.ts:248）`createSubagentContext` 后 **`for await query()` 复用主循环**（不是独立 agent loop）；定义加载 `getAgentDefinitionsWithOverrides(cwd)`（loadAgentsDir.ts:296，memoize）＝内置 + plugin + `.claude/agents/*.md` |
| **SkillTool** | `tools/SkillTool/SkillTool.ts:291` | `{skill, args?}`；展开 slash 命令（`processPromptSlashCommand`），fork 型技能在子代理里跑 |
| **MCPTool** | `tools/MCPTool/MCPTool.ts:27`（桩） | 真实实例在 `services/mcp/client.ts:1766` 展开：`name='mcp__server__tool'`、`mcpInfo`、`inputJSONSchema=tool.inputSchema`、readOnly/concurrency 取 `annotations.readOnlyHint`、`alwaysLoad←_meta['anthropic/alwaysLoad']`；call 走 `callMCPToolWithUrlElicitationRetry` |
| **LSPTool** | `tools/LSPTool/LSPTool.ts:59` | `ENABLE_LSP_TOOL` 启用；operation 判别联合（definition/references/hover/documentSymbol/callHierarchy…）；`isLsp:true`、`shouldDefer:true` |
| **EnterPlanMode/ExitPlanModeV2** | `tools/EnterPlanModeTool`、`tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts:147` | 切换 `ToolPermissionContext.mode`；`prePlanMode` 记录原模式，exit 时恢复；exit 输入 `allowedPrompts`（Bash 规则列表），teammate 走 `plan_approval_request` 信箱 |
| **EnterWorktree/ExitWorktree** | `tools/EnterWorktreeTool` | `createWorktreeForSession` → `process.chdir` + `setCwd` + `saveWorktreeState` |
| **SendMessageTool/TeamCreateTool** | `tools/SendMessageTool`、`tools/TeamCreateTool` | swarm 协议：teammateMailbox / UDS / bridge 路由；TeamCreate 写 `.claude/teams/` TeamFile |
| **SyntheticOutputTool** | `tools/SyntheticOutputTool/SyntheticOutputTool.ts` | 名 `'StructuredOutput'`，仅非交互启用；`createSyntheticOutputTool(jsonSchema)` 用 Ajv 编译校验，`call` 校验模型输出不匹配抛 `TelemetrySafeError`；不进 `getTools()`，由 SDK 动态注入 |

### 3.4 错误处理

**没有** `PermissionError/InterruptedError` 类。权限拒绝以 `PermissionResult` 返回由权限层处理；执行异常统一在 `toolExecution.ts:469` 捕获，打包成 `<tool_use_error>Error calling tool (Name): msg</tool_use_error>`、`is_error: true` 的 tool_result 回喂模型；中断输出 `CANCEL_MESSAGE`。错误类型：`ClaudeError/AbortError/ConfigParseError/ShellError/TelemetrySafeError`（`utils/errors.ts`）。

---

## 4. 命令系统（Slash Commands）

### 4.1 Command 类型（`src/types/command.ts`）——三态联合

- `PromptCommand`（`type:'prompt'`）：`getPromptForCommand(args, context): Promise<ContentBlockParam[]>`，附 `progressMessage/contentLength/argNames/allowedTools/model/source/hooks/context:'inline'|'fork'/agent/effort/paths`。
- `LocalCommand`（`type:'local'`）：懒加载 `load(): Promise<{call(args, ctx): Promise<LocalCommandResult>}>`，结果 `{type:'text'|'compact'|'skip'}`。
- `LocalJSXCommand`（`type:'local-jsx'`）：`call(onDone, ctx, args)` 返回 ReactNode（渲染 Ink UI），`onDone(result, {display, shouldQuery, metaMessages})` 回调把结果回填成 `<local-command-stdout>` 消息。

### 4.2 注册表与分发（`src/commands.ts`）

`getCommands(cwd)`（L476）：`loadAllCommands(cwd)`（按 cwd memoize）拼接七类来源：`[...bundledSkills, ...builtinPluginSkills, ...skillDirCommands, ...workflowCommands, ...pluginCommands, ...pluginSkills, ...COMMANDS()]`。`COMMANDS()`（L258，memoize）是约 80 个内置命令的硬编码数组，部分按 `feature()` 条件注入。每次调用重跑 `meetsAvailabilityRequirement`（claude-ai/console 认证门槛）与 `isCommandEnabled`，所以 `/login` 后立即可见新命令。

**分发链**：`PromptInput.tsx onSubmit` → `utils/processUserInput/processUserInput.ts`（`/` 开头）→ `processSlashCommand.tsx`：`parseSlashCommand`（拆 `{commandName, args, isMcp}`，支持 `/mcp:tool (MCP) arg` 语法）→ 按 type 分发；`local-jsx` 渲染 UI、`local` 执行并回填、`prompt` 展开内容（`getMessagesForPromptSlashCommand`，fork 型走 `executeForkedSlashCommand` 子代理）→ `shouldQuery:true` 交给 query 引擎。**AI 侧共存**：模型通过 **SkillTool** 以工具调用触发同一批命令，与用户 `/xxx` 共用 `getPromptForCommand`；`userInvocable:false` 的命令仅限模型调用。

### 4.3 自定义 markdown 命令

`utils/markdownConfigLoader.ts` 的 `loadMarkdownFilesForSubdir` 合并三处来源：`~/.claude/<subdir>`（用户，全局）、受管策略目录、`getProjectDirsUpToHome`（从 cwd 向上到 git root 的所有 `.claude/<subdir>`，可逐级嵌套合并）。`skills/loadSkillsDir.ts` 的 `parseSkillFrontmatterFields`（L185）解析 frontmatter：`description/allowed-tools/argument-hint/arguments/model/when_to_use/version/user-invocable/disable-model-invocation/context:'fork'/agent/effort/hooks/paths/shell`；`createSkillCommand`（L270）组装 Command，`getPromptForCommand` 内做 `$ARGUMENTS`/`$0`/命名参数替换（`utils/argumentSubstitution.ts:94`）与 `!` 内联 bash 执行。

### 4.4 重点命令速查

| 命令 | 文件 | 职责 |
|---|---|---|
| /compact | commands/compact/compact.ts | 会话记忆压缩优先 → `compactConversation` 清理上下文 |
| /mcp | commands/mcp/mcp.tsx + main.tsx:3894 | local-jsx 管理 UI + CLI add/remove/list/get；写 `.mcp.json`/settings.json |
| /config | commands/config/config.tsx | 设置面板（别名 /settings） |
| /doctor | commands/doctor/ | 诊断安装与配置 |
| /login /logout | commands/login\|logout/ | ConsoleOAuthFlow 登录/登出 |
| /memory | commands/memory/ | 编辑记忆文件 |
| /skills | commands/skills/skills.tsx | 列出技能 |
| /tasks | commands/tasks/ | 后台任务管理（别名 /bashes） |
| /vim | commands/vim/vim.js | 切换 vim/普通编辑 |
| /diff | commands/diff/diff.js | 未提交变更与逐轮 diff |
| /cost | commands/cost/cost.ts | `cost-tracker.formatTotalCost` |
| /theme | commands/theme/theme.tsx | ThemePicker |
| /context | commands/context/ | 上下文占用可视化 |
| /resume /continue | commands/resume/resume.tsx | sessionStorage 列会话、loadFullLog 恢复 |
| /share | commands/share/index.js | 本快照为 isEnabled:false 的 stub [ant-only] |

---

## 5. 服务层（`src/services/`）

### 5.1 Anthropic API（`services/api/`）

- `client.ts`：`getAnthropicClient()` 按 env 分支动态 import Bedrock/Vertex/Foundry SDK；OAuth 用户用 authToken，API key 用户注入 `x-api-key`；默认 600s timeout。
- `claude.ts`（3419 行）：`queryModelWithStreaming`/`queryModel`（见 2.4）、`executeNonStreamingRequest`、`queryHaiku()`（压缩/分类器用的小模型）、`queryWithModel()`。
- `withRetry.ts`：`withRetry()` 手动重试——401/403/云凭据错误重建 client；529 连续达阈值抛 `FallbackTriggeredError`（触发模型 fallback）；`getRetryDelay()` = 500ms×2ⁿ + 25% jitter，优先 Retry-After 头；429 支持 persistent 分块模式。
- **Prompt caching**：`getCacheControl()`（claude.ts:358）`{type:'ephemeral', ttl:'1h'?, scope?}`；断点位置在 `userMessageToMessageParam`/`assistantMessageToMessageParam` 的**最后一块** content；`addCacheBreakpoints()`（L3063）**每次请求恰好一个 message-level cache_control marker**（skipCacheWrite 时移到倒数第二条）；system prompt 由 `buildSystemPromptBlocks()` 经 `splitSysPromptPrefix` 打断点。
- `bootstrap.ts` `fetchBootstrapData()`：拉 `client_data` + `additional_model_options` 写 global config（不变不落盘）。
- `filesApi.ts`：`downloadFile/uploadFile/downloadSessionFiles`，`buildDownloadPath` 防路径穿越。

### 5.2 Feature Flag：GrowthBook vs `feature()`

`analytics/growthbook.ts`：`initializeGrowthBook`（memoize，阻塞，`new GrowthBook({remoteEval:true})`，5s timeout，无信任时只读磁盘）；`getFeatureValue_CACHED_MAY_BE_STALE`（**同步**：env override → 内存 → 磁盘缓存）；`getDynamicConfig_BLOCKS_ON_INIT`（阻塞版）；`refreshGrowthBookAfterAuthChange`。**与 `bun:bundle` 的 `feature()` 分工**：`feature('X')` 是**编译期** flag（版本能力），GrowthBook 是**运行时** flag（本次运行开关）；外部构建把 ant-only 分支整个 DCE 掉。

### 5.3 MCP（`services/mcp/client.ts`）

`connectToServer`（memoize，L595）按配置建 transport：stdio（`StdioClientTransport`，spawn 用户配置命令，支持 `CLAUDE_CODE_SHELL_PREFIX`）、sse（`SSEClientTransport`+ClaudeAuthProvider）、http（StreamableHTTP）、ws/ws-ide/claudeai-proxy、进程内 InProcessTransport（Chrome/Computer Use）。`fetchToolsForClient`（L1743，memoizeWithLRU）发 `tools/list` 转成 `mcp__{server}__{tool}`；`getMcpToolsCommandsAndResources`（L2226）分批并发连接、按服务器并行取 tools/commands/skills/resources，首个支持 resources 的服务器附带 `ListMcpResourcesTool`/`ReadMcpResourceTool`。`callMCPTool`（L3029）→ `client.callTool(..., {signal, timeout, onprogress})`；-32042 elicitation 错误走 `callMCPToolWithUrlElicitationRetry`。`config.ts parseMcpConfig`（L1297）zod 校验 + env 展开；`officialRegistry.ts` 拉官方 MCP registry 判断 URL 是否官方。

### 5.4 OAuth（`services/oauth/`）

`OAuthService.startOAuthFlow()`：**PKCE**（`generateCodeVerifier/Challenge`）→ `AuthCodeListener.start()` 起 **localhost 随机端口**（捕获 `/callback?code&state`，校验 state 防 CSRF）→ `exchangeCodeForTokens`（authorization_code + code_verifier）→ `fetchProfileInfo`。`refreshOAuthToken`（refresh_token grant）；`isOAuthTokenExpired`（5min buffer）。存储：`utils/auth.ts saveOAuthTokensIfNeeded` 写 secureStorage（macOS Keychain）；`checkAndRefreshOAuthTokenIfNeeded` 自动刷新。

### 5.5 LSP（`services/lsp/`）

`initializeLspServerManager()`（manager.ts:145，`--bare` 跳过）；**LSP server 只来自插件 manifest**（`config.ts getAllLspServers`）；`LSPServerManager` 按扩展名路由，`ensureServerStarted` 懒启动；`LSPServerInstance.createLSPServerInstance()`（L90）状态机 stopped→starting→running→error，`sendRequest` 对 -32801 ContentModified 重试 3 次。工具化见 3.3（单个 `LSPTool`，无服务器时返回提示；**没有** ripgrep fallback——Grep 是独立工具）。

### 5.6 插件（`services/plugins/` + `utils/plugins/`）

插件是**声明式**的（无 JS main 执行）：`.claude-plugin/plugin.json` manifest（`pluginLoader.ts loadPluginManifest` L1147，`PluginManifestSchema` 校验 metadata/hooks/commands/agents/skills/mcp/lsp/settings），hooks 经 settings 机制执行，LSP/MCP 由 manifest 声明。安装管理在 `services/plugins/pluginOperations.ts`（install/uninstall/enable，scope 限 user/project/local）。版本化缓存：`copyPluginToVersionedCache`（utils/plugins/pluginLoader.ts:365）+ 孤儿清理。

### 5.7 压缩 / 记忆 / 成本

- 见维度 10。
- `extractMemories/extractMemories.ts`：`initExtractMemories()`（L296，闭包状态：`lastMemoryMessageUuid` 游标 + 节流计数）；`executeExtractMemories()`（L598）由 `handleStopHooks` 在**每次查询循环结束**触发（仅主代理、GB gate、每 N 轮默认 1、主代理已直接写 memory 则跳过）；执行 `runForkedAgent` 完美 fork（共享 prompt cache，maxTurns 5），`createAutoMemCanUseTool` 限制只读工具 + 仅 memory 目录内 Edit/Write；写入 `~/.claude/projects/<path>/memory/`。
- `tokenEstimation.ts`：`countMessagesTokensWithAPI`（`anthropic.beta.messages.countTokens`，Bedrock 走 `countTokensWithBedrock`）；`countTokensViaHaikuFallback`；`roughTokenCountEstimation`（字符/4）。
- `policyLimits/`：`isPolicyAllowed(policy)` + `loadPolicyLimits`（fail-open、ETag、1h 轮询）；`remoteManagedSettings/`：企业托管设置拉取/校验/热更新。

---

## 6. 权限系统

### 6.1 模式与上下文

模式（`utils/permissions/PermissionMode.ts` + `src/types/permissions.ts`）：`default / plan / acceptEdits / bypassPermissions / dontAsk / auto`（auto 为 ant-only，`PERMISSION_MODES` 数组）。

`ToolPermissionContext`（`Tool.ts:123`，DeepImmutable）：`{mode, additionalWorkingDirectories: Map, alwaysAllowRules, alwaysDenyRules, alwaysAskRules, isBypassPermissionsModeAvailable, shouldAvoidPermissionPrompts, awaitAutomatedChecksBeforeDialog, prePlanMode}`。规则来源加载：`utils/permissions/permissionsLoader.ts loadAllPermissionRulesFromDisk`（settings.json 的 `permissions.allow/deny/ask` + additionalDirectories），启动时 `initializeToolPermissionContext`（`permissionSetup.ts:872`）合并 CLI 的 `--allowedTools/--disallowedTools/--permission-mode`，并做危险规则剥离（`removeDangerousPermissions`/`stripDangerousPermissionsForAutoMode`）。

### 6.2 决策管线 `hasPermissionsToUseTool`（`utils/permissions/permissions.ts:473`）

`hasPermissionsToUseToolInner` 步骤（1a→1g→2a→2b→3）：

1. **1a** 整工具 deny 规则 → deny；**1b** 整工具 ask 规则 → ask（Bash 沙箱自动放行例外）；**1c** `tool.checkPermissions(parsedInput, context)`（工具自检，如 Bash 子命令规则）；**1d** 工具自检 deny；**1e** `requiresUserInteraction()` 即使 bypass 也 ask；**1f** 内容级 ask 规则（如 `Bash(npm publish:*)`）**优先级高于 bypassPermissions**；**1g** `safetyCheck`（`.git/`、`.claude/`、`.vscode/`、shell 配置等敏感路径，`checkPathSafetyForAutoEdit`）**bypass-immune**，必须弹窗。
2. **2a** bypass 判定（`mode==='bypassPermissions'` 或 plan+`isBypassPermissionsModeAvailable`）→ allow；**2b** `toolAlwaysAllowedRule`（整工具 allow 规则）→ allow。
3. **3** passthrough 转 ask → 进入交互层。

外层（L473 起）追加模式转换：`dontAsk` 把 ask 转 deny；**auto 模式**先试 acceptEdits 快路径（`tool.checkPermissions` 在 acceptEdits 模式下 allow 则放行，Agent/REPL 工具除外）→ 安全工具 allowlist（`isAutoModeAllowlistedTool`）→ **YOLO 分类器** `classifyYoloAction`（`yoloClassifier.ts`，把当前动作+最近消息发小模型裁决，含 usage/成本遥测）；分类器不可用时按 `tengu_iron_gate_closed` 决定 fail-closed/fail-open；连续 deny 达到阈值回退人工确认（`denialTracking`）。

### 6.3 交互层（`hooks/useCanUseTool.tsx` + `hooks/toolPermission/`）

`useCanUseTool` 是 REPL 提供的 `CanUseToolFn`。决策结果：

- allow → `ctx.buildAllow(result.updatedInput ?? input)`，直接放行。
- deny → 记日志/通知（auto 模式 `recordAutoModeDenial`）。
- ask → 依次尝试：`handleCoordinatorPermission`（coordinator worker 自动裁决）→ `handleSwarmWorkerPermission`（swarm 工人把请求转发给 leader）→ Bash 投机分类器 `peekSpeculativeClassifierCheck`（Bash 执行期间**并行**跑分类器，2s 竞速）→ `handleInteractivePermission`（`interactiveHandler.ts:57`）。

`handleInteractivePermission` 通过 `createPermissionContext`（`PermissionContext.ts`，388 行）把确认请求推入队列（`pushToQueue`，UI 渲染 `PermissionRequest.tsx` 弹窗），注册 `onAllow/onReject/onAbort/onUserInteraction/recheckPermission` 回调，并用 **`resolveOnce`/`claim()` 原子竞争模型**：本地用户按键、hook、分类器、CCR bridge 响应（`bridgeCallbacks.sendRequest` 转发到手机/网页端）、channel（飞书等）谁先到谁赢。允许时还可带 `PermissionUpdate[]`（如"总是允许此规则"写回 settings）。`recheckPermission` 支持规则热更新后重查。

### 6.4 Hook 机制（`utils/hooks.ts`，5022 行）

约 25 种 hook 事件，全部有对应执行器：`executePreToolUseHooks`（L3394）、`executePostToolUseHooks`（L3450）、`executePostToolUseFailureHooks`、`executePermissionDeniedHooks`、`executeStopHooks`（L3639，可阻止继续/注入消息）、`executeSessionStartHooks`（L3867）、`executeSessionEndHooks`、`executeSubagentStartHooks`、`executePreCompactHooks`（L3961）/`executePostCompactHooks`、`executePermissionRequestHooks`（L4157）、`executeConfigChangeHooks`、`executeCwdChangedHooks`、`executeFileChangedHooks`、`executeStatusLineCommand`（StatusLine 可自定义状态栏）、`executeFileSuggestionCommand`、`executeWorktreeCreateHook` 等。`getMatchingHooks`（L1603）按 hook 事件 + `if` 条件（含工具名/`Bash(git *)` 风格匹配）筛选；执行经 `executeHooks` 派生（外部命令 /plugin hook）。工具执行管线（`toolExecution.ts`）在 `runToolUse` 里串起 PreToolUse → 权限 → call → PostToolUse，hook 可改输入（updatedInput）、可阻断（blocking error 触发 query loop 的 `stop_hook_blocking` 重试）。

---

## 7. Bridge 与多代理

### 7.1 IDE / 远程双向通信（`src/bridge/`，31 文件）

- `bridgeMain.ts`：`bridgeMain(args)`（L1980）/`runBridgeLoop`（L141）——`claude remote-control`/`claude rc` 的常驻循环：环境注册（codeSessionApi）、**入站 WebSocket**（`remote/SessionsWebSocket.ts`）接收手机/网页端消息，`replBridge.ts`/`replBridgeTransport.ts` 维护连接状态机，`initReplBridge.ts` 在 REPL 内装配（AppState 里 `replBridgeEnabled/Connected/SessionActive` 等字段）。
- 权限双向：`bridgePermissionCallbacks.ts` 把工具确认请求经 `sendRequest/sendResponse` 转发到 CCR（见 6.3 竞争模型）。
- `trustedDevice.ts`：可信设备 token 注册；`workSecret.ts`：工作区密钥。
- 另有 `capacityWake.ts`（会话唤醒）、`sessionRunner.ts`、`pollConfig.ts`（轮询配置）。

### 7.2 远程会话（`src/remote/`）

`RemoteSessionManager`（L95 类）：`createRemoteSessionConfig(sessionId, getAccessToken, orgUUID, hasInitialPrompt)`（L329）；`sdkMessageAdapter.ts` 把 CLI 内部消息翻译成远程 SDK 消息；`remotePermissionBridge.ts` 处理远程权限响应。`--remote`/`--teleport`（CCR）流程在 main.tsx：`teleportToRemoteWithErrorHandling` → `checkOutTeleportedSessionBranch` → `processMessagesForTeleportResume`（`utils/teleport.ts`），policy 检查 `isPolicyAllowed('allow_remote_sessions')`。

### 7.3 Server 模式（`src/server/`）

`claude server`（main.tsx:3962 注册，DIRECT_CONNECT gate）：`startServer`（HTTP/Unix socket）、`SessionManager`（idle-timeout 600s、max-sessions 32）、`DangerousBackend`、lockfile 防多实例。`createDirectConnectSession` + `parseConnectUrl`（`cc://` URL 解析）；交互模式 cc:// URL 在 main() 里 argv 重写（把 `cc://` 剥离、走默认 action 拿完整 TUI），headless 走 `server/connectHeadless.ts`。

### 7.4 Coordinator（`src/coordinator/coordinatorMode.ts`）

`isCoordinatorMode()` = `CLAUDE_CODE_COORDINATOR_MODE` env（feature 门控）。coordinator 模式只暴露 `Task/TeamCreate/TeamDelete/SendMessage/SyntheticOutput` 等编排工具（`INTERNAL_WORKER_TOOLS` + 过滤），workers 只剩 Bash/Read/Edit；`getCoordinatorUserContext` 注入 scratchpad 上下文；权限走 `coordinatorHandler.ts`（`awaitAutomatedChecksBeforeDialog` 先跑自动化检查再弹窗）。

### 7.5 子代理与团队（Task/teammate/swarm）

- `src/Task.ts`：任务类型 `local_bash/local_agent/remote_agent/in_process_teammate/local_workflow/monitor_mcp/dream`，`generateTaskId`（前缀 `b/a/r/t/w/m/d` + 8 位字母数字，防 symlink 暴力猜解）、`createTaskStateBase`；`src/tasks.ts` 注册表。
- AgentTool 子代理：`runAgent`（runAgent.ts:248）→ `createSubagentContext`（带 `agentId`，权限/记忆/工具独立）→ **复用 `query()`**；后台 agent 产出 Task 状态，UI 有任务面板。
- Swarm（`utils/swarm/`）：tmux 队友（`--agent-id/--agent-name/--team-name` CLI 选项 + `teammateModeSnapshot`）、in-process 队友（`InProcessTeammateTask`，`injectUserMessageToTeammate`）、`teammateMailbox` 消息路由、`leaderPermissionBridge`（worker 权限请求转发 leader）。`utils/teammate.js`/`swarm/reconnection.ts` 处理重连。
- Fork 子代理：`tools/AgentTool/forkSubagent.ts`（`buildForkedMessages` 共享父 prompt cache）。

---

## 8. 持久化与状态

### 8.1 双层状态

- **模块级全局状态** `bootstrap/state.ts`（1758 行）：无框架的 getter/setter 池——`getSessionId()/switchSession()`、成本/耗时计数器（`getTotalCostUSD/getTotalInputTokens/getTotalAPIDuration`…）、`setCwdState/getCwdState`、模型（`setMainLoopModelOverride/getMainLoopModel`）、`setIsRemoteMode` 等。被 services 和 utils 大量引用，避免 prop-drilling。
- **React Store** `state/AppStateStore.ts`：`AppState` 类型（DeepImmutable：settings/mode/toolPermissionContext/mcp/plugins/tasks/agentDefinitions/remote 状态…）+ `getDefaultAppState()` + `createStore`（`state/store.ts`，zustand 风格）+ `onChangeAppState`。REPL 通过 `getAppState/setAppState` 读写。

### 8.2 会话转录（`utils/sessionStorage.ts`，5105 行）

会话 = `~/.claude/projects/<项目路径编码>/<sessionId>.jsonl`（`getTranscriptPath`），**JSONL 即真相**：`recordTranscript(messages)` 增量 append（100ms 懒序列化写队列），`flushSessionStorage` 强制落盘；`sessionIdExists`；子代理走 **sidechain transcript**（`recordSidechainTranscript`，`getAgentTranscriptPath`，按 agentId 独立文件）。恢复：`loadConversationForResume`/`loadTranscriptFile` → `processResumedConversation`（`utils/sessionRestore.ts`）重建消息树、恢复 agent 定义/文件快照（`fileHistoryMakeSnapshot`）/成本（`restoreCostStateForSession`）。`getLastSessionLog` 供 `/continue`。`--no-session-persistence` 可关。

### 8.3 输入历史（`src/history.ts`）

`~/.claude/history.jsonl`（`addToHistory`，文件锁 + 幂等 flush）；**粘贴内容引用机制**：大段粘贴不直接进历史，替换为 `[Pasted text #N +10 lines]` / `[Image #N]` 引用（`formatPastedTextRef`/`parseReferences`/`expandPastedTextRefs`），内容哈希存 paste store，按项目/会话过滤回显（`getHistory`）。

### 8.4 配置（`utils/config.ts`）

`getGlobalConfig/saveGlobalConfig`（`~/.claude.json`：migrationVersion、theme、autoCompactEnabled、cachedGrowthBookFeatures、lastReleaseNotesSeen、clientDataCache…）+ `getCurrentProjectConfig/saveCurrentProjectConfig`（`<cwd>/.claude.json`：lastCost/lastSessionId 等会话退出指标）。`enableConfigs()` 之前配置系统不可读（防止未信任目录读配置执行代码）。设置文件：`utils/settings/settings.ts getSettingsForSource`（policySettings/userSettings/projectSettings/localSettings/flags，`settingsMergeCustomizer` 深度合并），Zod 校验（`utils/settings/validation.ts`），schema 集中在 `schemas/hooks.ts`（hook 设置 schema）+ `utils/settings/types.ts`。

### 8.5 记忆（`src/memdir/`）与技能（`src/skills/`）

- `memdir/memdir.ts`：`loadMemoryPrompt()`（L419）、`ensureMemoryDirExists`、`buildMemoryLines/buildMemoryPrompt`（MEMORY.md 入口 ≤200 行/25KB + 主题文件 + "搜索过去上下文"指导）；`paths.ts getAutoMemPath()` = `~/.claude/projects/<path>/memory/`；`findRelevantMemories.ts findRelevantMemories()`（按查询找相关记忆文件）；`teamMemPaths/teamMemPrompts`（团队记忆）。
- `skills/loadSkillsDir.ts`：`getSkillDirCommands(cwd)`（`name/SKILL.md` 目录格式）、`parseSkillFrontmatterFields`、`createSkillCommand`；`bundledSkills.ts` + `bundled/`（内置技能注册）。

---

## 9. 终端 UI

### 9.1 Ink 架构（`src/ink.ts` + `src/ink/` 96 文件）

`ink.ts` 是**自研 Ink fork** 的再包装：`createRoot/render` 全部包一层 `ThemeProvider`（`components/design-system/ThemeProvider.tsx`），导出 `ThemedBox/ThemedText/Ansi/Button/Link/Spacer`、`useInput/useStdin/useApp/useSelection/useTabStatus/useTerminalFocus/useTerminalTitle/useTerminalViewport/useAnimationFrame`、事件系统（`ink/events/input-event.ts` 等）、`FocusManager`、`wrapText`。底层 `ink/root.ts → ink/ink.tsx`（Yoga flex 布局），`termio/` 实现原始终端：`stdin.setRawMode(true)`、Kitty 键盘协议白名单、OSC 52 剪贴板、终端尺寸检测、非 TTY（管道）fallback（`termio/` + `Ansi.ts`）。

### 9.2 REPL 组件树（`screens/REPL.tsx`，5005 行）

单棵 `<App>`（Provider 外壳）→ `<REPL>` 巨型组件：`FullscreenLayout` + `ScrollBox` 实现"输出区 flexGrow 滚动、输入区 flexShrink=0 固定"。流式渲染：REPL 直接 `for await (const event of query({...}))`，`handleMessageFromStream` → `setMessages`，输入框用 `useDeferredValue` 保流畅。`useLogMessages` 管理消息数组。子组件 ~389 个：`PromptInput`（多行输入、`/` 命令补全、文件路径补全）、`VirtualMessageList`（虚拟滚动 + 搜索高亮）、`MessageSelector`（消息过滤）、`TodoPanel`、`AgentPanel`、`Statusline`、`CostThresholdDialog`、`IdleReturnDialog` 等。

### 9.3 权限弹窗（`components/permissions/PermissionRequest.tsx` + `interactiveHandler.ts`）

按工具分派渲染组件（Bash 弹窗显示命令+建议规则，Edit 显示 diff，WebFetch 显示 URL…），y/n 走 `confirm:yes/no` 键位，支持"总是允许此规则"（写回 settings）与"拒绝并反馈"；队列式（一次一个确认）；Esc 中断。交互期间 Bash 分类器可在后台并行预判，用户 200ms 宽限期内按键不打断。

### 9.4 Diff / Vim / Voice / Keybindings

- **diff**：`color-diff-napi`（原生） + `diff` 库，`components/StructuredDiff/`（colorDiff.ts）+ `Fallback.tsx`；diff 高亮 + 每轮文件变更面板。
- **vim**：**纯自研状态机**（`src/vim/transitions.ts` + motions/operators/textObjects），`useVimInput` 驱动 `VimTextInput`，`/vim` 命令切换。
- **voice**：WebSocket 连 Anthropic voice_stream（Deepgram STT），原生音频模块录音，`voice:pushToTalk` 键位（默认空格）。
- **keybindings**：`src/keybindings/`（14 文件）默认键位表 + 用户 `~/.claude/keybindings.json` 自定义；`useGlobalKeybindings`/`useCommandKeybindings` 装配。

---

## 10. 上下文与成本

### 10.1 成本追踪（`src/cost-tracker.ts`）

`addToTotalSessionCost(cost, usage, model)`：按模型聚合 token 用量（input/output/cacheRead/cacheCreation/webSearch）+ `calculateUSDCost` 换算金额 + OTel 计数器。`formatTotalCost()` 输出"Total cost / API duration / 增删行数 / 按模型 Usage"。**会话恢复**：`saveCurrentSessionCosts()` 把 `lastCost/lastSessionId/lastModelUsage` 等写进 `.claude.json`（`utils/config.ts` 的 `saveCurrentProjectConfig`），`restoreCostStateForSession(sessionId)` 在 `/resume` 时恢复，保证跨会话成本连续。`/cost` 命令直接调它；`--max-budget-usd` 在 QueryEngine 每消息熔断。

### 10.2 Token 估算（`services/tokenEstimation.ts` + `utils/tokens.ts`）

精确路径 `countMessagesTokensWithAPI`（`anthropic.beta.messages.countTokens`）；`countTokensViaHaikuFallback`（Haiku 实测，thinking 块换 Sonnet）；快速路径 `roughTokenCountEstimation`（字符/4，JSON 2 字节/token，图片/文档固定 2000）。`utils/tokens.ts` 的 `tokenCountWithEstimation` 优先用 API 返回的真实 usage（`finalContextTokensFromLastResponse`），缺失才估算——这是 autocompact 判定依据。

### 10.3 压缩体系（多级、`services/compact/`）

触发阈值（`autoCompact.ts`）：`getEffectiveContextWindowSize` = 上下文窗口 − 预留 20K（summary 输出 p99.99）；`getAutoCompactThreshold` = 有效窗口 − `AUTOCOMPACT_BUFFER_TOKENS(13_000)`；警告/错误阈值再各留 20K；手动 `MANUAL_COMPACT_BUFFER_TOKENS = 3_000`（手动压缩前还能撑一点）。`calculateTokenWarningState` 输出 percentLeft/warning/error/autoCompact/blocking 五档，REPL 顶部显示百分比。

管线（query.ts 每次迭代顺序）：**snip**（HISTORY_SNIP，删中间段落保留头尾）→ **microcompact**（`microCompact.ts`，去已读文件内容/陈旧附件，CACHED_MICROCOMPACT 版本用 API 的 cache_edits 服务端删）→ **contextCollapse**（CONTEXT_COLLAPSE，提交日志投影，保留细粒度上下文）→ **autocompact** `autoCompactIfNeeded`（L241）：先试 `trySessionMemoryCompaction`（会话记忆优先，只压缩记忆相关段）→ `compactConversation`（`compact.ts:387`）：PreCompact hooks → `streamCompactSummary`（整个对话 + `<analysis>+<summary>` 提示词，禁止调工具，预留 20K 输出）→ 清 readFileState → 并行生成附件（恢复 ≤5 个文件/50K token、plan、skills、agent delta）→ 产出 `compact_boundary` 系统消息（带 preCompactTokenCount）→ summary 作为 `isCompactSummary:true` 的 user 消息。**Reactive compact**（REACTIVE_COMPACT）：API 413 prompt-too-long 时**响应式**压缩重试（`tryReactiveCompact`）；失败 3 次熔断（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）。System prompt 本身不压缩（每次重建，靠 prompt cache）。

### 10.4 上下文窗口管理

`utils/context.ts`：`getContextWindowForModel(model, betas)`（200K/1M 档 + betas）；plan 模式超 200K 时模型降级（`doesMostRecentAssistantMessageExceed200k` → `getRuntimeMainLoopModel`）；blocking limit 检查（`isAtBlockingLimit`，保留 3K 给手动 /compact）；max_output_tokens 恢复循环与 `ESCALATED_MAX_TOKENS`（64K 重试）；`query/tokenBudget.ts`（TOKEN_BUDGET，长输出回合自动续）。

---

## 附：Claude Code 最独特的设计决策（20 条）

1. **编译期 + 运行时双轨 feature flag**：`bun:bundle` 的 `feature('X')` 让 ant-only 代码连同字符串在外部构建中被整个树摇掉；GrowthBook 负责运行时开关。能力边界在构建期、灰度在运行期。
2. **整个 agent 循环是 async generator 单控制流**：`query()` 一个 `while(true)` + 状态机，模型流、工具执行、hook、压缩全部 yield 进同一条事件流；工具结果靠"消息数组拼接"回填，没有独立的 tool loop。
3. **REPL 与 headless 共用同一个 `query()`**：`screens/REPL.tsx:2793` 和 `QueryEngine.ts` 只是同一循环的两个消费端（UI 回调 vs SDK 消息翻译）。
4. **极致的启动性能工程**：模块求值期并行启动 MDM/keychain 子进程；`startDeferredPrefetches()` 把所有可延迟工作藏到首帧之后；`profileCheckpoint` 遍布全路径，启动基准有专门 env 开关。
5. **工具 schema 双轨制**：Zod v4（开发体验）+ `inputJSONSchema`（MCP/结构化输出直通），`zodToJsonSchema` 只在两者都缺时兜底。
6. **工具描述即缓存键**：工具按名排序、`--settings` 用内容哈希路径、`assembleToolPool` 内置优先去重——一切都是为了 **prompt cache 前缀稳定**（一次 cache miss 是 12x 输入 token 成本）。
7. **权限 = 规则集 + 工具自检 + 模式状态机**，且 deny 规则在"工具列表"层面预剔除（`filterToolsByDenyRules`），模型根本看不到被禁工具。
8. **bypass 也有不可绕过的红线**：`safetyCheck`（`.git/`、`.claude/`、shell 配置）和内容级 ask 规则在 bypassPermissions 模式下仍然弹窗。
9. **auto 模式的 YOLO 分类器**：每次 ask 用一个小模型裁决，带 acceptEdits 快路径 + 安全工具 allowlist + 连续拒绝熔断 + fail-open/fail-closed 双模式（iron gate）。
10. **权限决策是"多路竞争"**：本地按键 / hook / 分类器 / CCR 手机端 / 频道，`resolveOnce/claim()` 谁先到谁赢——CLI 不是唯一裁决者。
11. **多级上下文压缩流水线**：snip → microcompact（服务端 cache_edits 删 token）→ contextCollapse（保留细粒度）→ autocompact（总结）→ reactive compact（413 响应式），每级都有熔断。
12. **prompt cache 与压缩联动**：microcompact 用 `cache_edits`/`cache_reference` 让**服务端**删 token，压缩边界消息携带 token 计数，缓存断点检测防误报。
13. **会话 JSONL 即真相 + sidechain**：主线程转录与子代理 sidechain 分离，任何时刻 kill -9 后都能 `/resume`；转录写入在用户消息落库前就发生（防"无会话可恢复"）。
14. **粘贴引用机制**：大段粘贴存哈希，历史里只留 `[Pasted text #N]` 引用，Ctrl+R/↑ 时按需还原。
15. **AgentTool 复用 query() 而非独立循环**：子代理 = 新 `ToolUseContext`（agentId/权限/记忆隔离）+ 同一个 query()，天然获得压缩、hook、fallback 全部能力。
16. **WebSearchTool 套娃**：搜索 = 再起一个 agent 循环，让子代理用 WebFetch 自主探索——"搜索"本身是 agentic 的。
17. **Hook 系统是扩展点之王**：25+ 事件（Pre/PostToolUse、Stop、SessionStart/End、SubagentStart、Pre/PostCompact、StatusLine、FileChanged、WorktreeCreate…），外部命令或 JS 均可，hook 可以改工具输入、阻断循环、注入消息。
18. **插件是纯声明式的**：manifest 声明 hooks/commands/agents/skills/MCP/LSP，无 JS main 入口——安全模型简单，能力全靠声明 + hook 机制。
19. **会话指标跨会话持久化**：成本/时长/增删行数/token 写进 `.claude.json`，resume 时恢复成本状态；退出时 `tengu_exit` 事件回传上一会话指标做健康度分母。
20. **自研终端栈**：Ink fork（Yoga 布局）+ 原始终端层（kitty 键盘/OSC52）+ 自研 vim 状态机 + Deepgram 语音——UI 全栈自控，不依赖上游 Ink。

---

## 关键文件速查表

| 路径 | 一句话职责 |
|---|---|
| `src/entrypoints/cli.tsx` | 进程 bin 入口；`--version` 等 fast-path 拦截 |
| `src/main.tsx` | Commander CLI 编排（4684 行）：全部选项/子命令、权限初始化、MCP/插件装配、REPL 各分支 |
| `src/entrypoints/init.ts` | `init()`：配置系统/安全 env/代理/预连接/懒遥测 |
| `src/setup.ts` | `setup()`：node 校验、UDS、worktree、hook 快照、bypass 安全检查、`tengu_started` beacon |
| `src/bootstrap/state.ts` | 模块级全局状态池（sessionId/成本/token/模型…1758 行） |
| `src/replLauncher.tsx` | `<App><REPL/></App>` 装配 |
| `src/query.ts` | 查询主循环（1729 行）：压缩流水线→流式模型→工具执行→结果回填 |
| `src/QueryEngine.ts` | headless/SDK 会话引擎：system prompt 组装、SDK 消息翻译、转录/预算熔断 |
| `src/context.ts` | git status / CLAUDE.md / 日期 上下文（memoize） |
| `src/Tool.ts` | Tool 类型、ToolUseContext、buildTool 默认实现 |
| `src/tools.ts` | 工具注册表：getAllBaseTools/getTools/assembleToolPool |
| `src/tools/BashTool/` | Bash 执行 + bashPermissions 规则引擎 |
| `src/tools/AgentTool/runAgent.ts` | 子代理执行（createSubagentContext + 复用 query） |
| `src/tools/AgentTool/loadAgentsDir.ts` | agent 定义加载（内置+plugin+`.claude/agents/*.md`） |
| `src/tools/SyntheticOutputTool/` | StructuredOutput 工具（Ajv 结构化输出校验） |
| `src/commands.ts` | slash 命令注册表（内置 + skills/plugins/workflows 合并） |
| `src/types/command.ts` | Command 三态类型（prompt/local/local-jsx） |
| `src/utils/processUserInput/processSlashCommand.tsx` | slash 命令分发执行 |
| `src/services/api/claude.ts` | 流式 API 客户端：SSE switch、cache_control、fallback、非流式降级 |
| `src/services/api/withRetry.ts` | 重试/退避/529 fallback 触发 |
| `src/services/analytics/growthbook.ts` | 运行时 feature flag（remoteEval GrowthBook） |
| `src/services/mcp/client.ts` | MCP 连接/工具/资源；mcp__ 工具实例化（L1766） |
| `src/services/mcp/config.ts` | MCP 配置解析/增删（写 .mcp.json/settings.json） |
| `src/services/oauth/` | PKCE OAuth 流程 + localhost 回调 + 刷新 |
| `src/services/lsp/manager.ts` | LSP server manager（插件声明式 server） |
| `src/services/compact/compact.ts` | compactConversation：总结压缩 + 附件重建 + boundary |
| `src/services/compact/autoCompact.ts` | 自动压缩阈值/判定/熔断 |
| `src/services/extractMemories/extractMemories.ts` | 自动记忆提取（fork agent 只读扫描 → memory/） |
| `src/services/tokenEstimation.ts` | API 计数 + 字符估算 |
| `src/utils/permissions/permissions.ts` | hasPermissionsToUseTool 决策管线（1a–3 步骤） |
| `src/utils/permissions/permissionSetup.ts` | 启动时权限上下文初始化、危险规则剥离 |
| `src/utils/permissions/yoloClassifier.ts` | auto 模式 AI 分类器 |
| `src/hooks/useCanUseTool.tsx` | CanUseToolFn：allow/deny/ask 分派 |
| `src/hooks/toolPermission/handlers/interactiveHandler.ts` | 权限弹窗队列 + 多路竞争裁决 |
| `src/utils/hooks.ts` | 25+ hook 事件执行器（5022 行） |
| `src/utils/sessionStorage.ts` | 会话转录 JSONL 读写/resume/侧链（5105 行） |
| `src/history.ts` | 输入历史 + 粘贴引用机制 |
| `src/memdir/memdir.ts` | 记忆目录：MEMORY.md 入口、记忆 prompt |
| `src/skills/loadSkillsDir.ts` | 技能目录发现 + frontmatter 解析 |
| `src/state/AppStateStore.ts` | AppState 类型 + getDefaultAppState |
| `src/cost-tracker.ts` | 成本聚合/展示/会话恢复 |
| `src/screens/REPL.tsx` | 交互式 REPL 巨型组件（5005 行，query() 消费者） |
| `src/ink.ts` / `src/ink/` | 自研 Ink：raw mode、termio、组件库 |
| `src/vim/` | 自研 vim 状态机 |
| `src/bridge/bridgeMain.ts` | remote-control 常驻循环（CCR 双向） |
| `src/remote/RemoteSessionManager.ts` | 远程会话配置/管理 |
| `src/coordinator/coordinatorMode.ts` | coordinator 模式工具过滤 |
| `src/Task.ts` | 任务类型/ID 生成（多代理状态） |
| `src/migrations/` | 11 个配置迁移（CURRENT_MIGRATION_VERSION=11） |

---

# Claude Code 工具系统源码笔记（src/ 快照）

> 所有引用均为实际读到的代码。核心路径：`src/Tool.ts`、`src/tools.ts`、`src/tools/**`、`src/services/mcp/client.ts`、`src/services/tools/toolExecution.ts`、`src/utils/Shell.ts`。

## 1. Tool 类型（src/Tool.ts）

- **schema 是 Zod（v4），非手写 JSON Schema**：`Tool<Input, Output, P>` 的 `readonly inputSchema: Input`（`Input extends AnyObject = z.ZodType<{...}>`）。MCP 工具例外：可选字段 `inputJSONSchema?: ToolInputJSONSchema`（`{type:'object', properties}`）直接携带 JSON Schema（`src/services/mcp/client.ts:1813` 注入）。
- **关键字段**：`name`、`aliases?`、`call()`、`description()`、`prompt()`、`outputSchema?`、`isConcurrencySafe(input)`、`isEnabled()`、`isReadOnly(input)`、`isDestructive?(input)`、`interruptBehavior?(): 'cancel'|'block'`、`isSearchOrReadCommand?`、`isOpenWorld?`、`requiresUserInteraction?`、`isMcp?`、`isLsp?`、`shouldDefer?`、`alwaysLoad?`、`mcpInfo?`、`maxResultSizeChars`、`strict?`、`checkPermissions(input, context)`、`validateInput?(input, context): Promise<ValidationResult>`、`getPath?`、`preparePermissionMatcher?`、`backfillObservableInput?`、`toAutoClassifierInput`、`mapToolResultToToolResultBlockParam` 及一整套 `renderToolUseMessage/renderToolResultMessage/...` React 渲染钩子。
- **权限声明方式**：没有 `needsPermissions` 字段；权限靠 `checkPermissions(): Promise<PermissionResult>`（`behavior: 'allow'|'ask'|'deny'|'passthrough'`，见 `src/types/permissions.ts`）+ `validateInput` 前置校验。`src/tools.ts` 里还有按 deny 规则**预先剔除**工具：`filterToolsByDenyRules()`（`getDenyRuleForTool` 匹配 `mcp__server` 前缀规则）。
- **默认值**：`buildTool(def)`（Tool.ts:783）用 `TOOL_DEFAULTS` 补齐——`isEnabled→true`、`isConcurrencySafe→false`、`isReadOnly→false`、`isDestructive→false`、`checkPermissions→{behavior:'allow', updatedInput}`、`userFacingName→name`。`ToolDef` = 可省略这些默认方法。
- **流式输出**：无 `isStreamable` 字段。流式通过 `call(..., onProgress?: ToolCallProgress<P>)` 回调（`ToolProgress<ToolProgressData>`：`bash_progress`/`mcp_progress`/`web_search_progress` 等，类型在 `src/types/tools.ts`）；工具结果通过 `ToolResult<T>` 的 `newMessages`（User/Assistant/System 消息数组）多轮展开。
- **ToolResult<T>**：`{ data, newMessages?, contextModifier?, mcpMeta? }`。

## 2. 注册表（src/tools.ts）

- `getAllBaseTools(): Tools`（:193）返回全部内置工具数组（AgentTool、BashTool、FileRead/Edit/Write、NotebookEdit、WebFetch/WebSearch、SkillTool、EnterPlanMode、ExitPlanModeV2、Glob/Grep 仅在 `!hasEmbeddedSearchTools()` 时加入、LSPTool 需 `ENABLE_LSP_TOOL`、Enter/ExitWorktree 需 worktree 模式、TaskCreate/Get/Update/List 需 `isTodoV2Enabled()`、TeamCreate/Delete 需 `isAgentSwarmsEnabled()`，其余靠 `feature('X')` 条件 require 做死代码消除）。
- `getTools(permissionContext)`（:271）：`CLAUDE_CODE_SIMPLE` 时只留 Bash/Read/Edit（REPL 模式替换为 REPLTool）；否则 `getAllBaseTools()` 去掉 `ListMcpResourcesTool/ReadMcpResourceTool/StructuredOutput` 三个特殊工具 → `filterToolsByDenyRules` → REPL 模式隐藏 `REPL_ONLY_TOOLS` → `isEnabled()` 过滤。
- `assembleToolPool(permissionContext, mcpTools)`（:345）：内置 + MCP 各自按名排序后 `uniqBy('name')` 去重（内置优先），保证 prompt 缓存稳定。`getMergedTools`（:383）简单拼接。
- **返回值结构**：`Tools = readonly Tool[]`（数组，非 map）。

## 3. BashTool（src/tools/BashTool/）

- `inputSchema`（BashTool.tsx:227）：`command`、`timeout`（`semanticNumber`，max 由 `getMaxTimeoutMs()`）、`description`、`run_in_background`、`dangerouslyDisableSandbox`、内部 `_simulatedSedEdit`（**模型侧被 `.omit()` 掉**，防止绕过权限/沙箱）。
- **执行**：`call()` 消费 `runShellCommand` async generator → `exec()`（`src/utils/Shell.ts:181`）→ `spawn(spawnBinary, shellArgs)`（Node `child_process`，shell 为 bash/zsh/pwsh；`provider.getSpawnArgs` 来自 `bashProvider.ts`）。stdout/stderr 走 TaskOutput 文件 fd（O_APPEND 交错）或 onStdout pipe 模式；超时/整树 kill/自动后台由 `wrapSpawn` 处理。CWD 用 `utils/cwd.ts` 的 `pwd()/getCwd()`，`setCwd`（Shell.ts:447）；子代理 `preventCwdChanges=true`。
- **权限**：`checkPermissions → bashToolHasPermission()`（`bashPermissions.ts:1663`，核心 `bashToolCheckPermission` 流程：精确匹配 deny/ask/allow 规则 → 前缀/通配规则 → sed 约束 → 只读命令 allow；另有 sandbox 自动允许、Bash prompt deny/ask 分类器、`checkCommandAndSuggestRules`）。`preparePermissionMatcher` 用 `parseForSecurity` 拆复合命令逐子命令匹配 hook 规则。
- `isReadOnly` = `checkReadOnlyConstraints` 通过；`isConcurrencySafe` = `isReadOnly`。错误走 `ShellError`，`is_error: true`；图片输出检测 `isImageOutput`；大输出持久化 `persistedOutputPath`（>30K `maxResultSizeChars`）。

## 4. FileRead / FileWrite / FileEdit / Glob / Grep

- **FileReadTool**（FileReadTool.ts:337）：`{file_path, offset, limit, pages?}`；`isReadOnly/isConcurrencySafe→true`；限额来自 `toolUseContext.fileReadingLimits ?? getDefaultFileReadingLimits()`（`limits.ts`：maxTokens/maxSizeBytes），超限报错并提示用 offset/limit；`maxResultSizeChars: Infinity`（防 Read→文件→Read 循环）；支持 notebook/图片（token 预算内压缩）。
- **FileWriteTool**（FileWriteTool.ts:56）：`{file_path, content}`；`checkPermissions → checkWritePermissionForTool`；原子写（`writeTextContent`，备份 + 读状态过期检测 `FileStateCache`）。
- **FileEditTool**（FileEditTool/types.ts:6）：`{file_path, old_string, new_string, replace_all?}`；`validateInput` 做**编辑前校验**（old==new 拒绝、deny 规则、>1GiB 拒绝、字符串不存在/多处匹配未开 replace_all 报错、`findActualString` 处理弯引号）；`checkPermissions → checkWritePermissionForTool(FileEditTool,...)`；`call` 用 `getPatchForEdit` 生成结构化 patch。**无 checkEditResult/postToolUse 复核机制**（全库 grep 无此符号）——靠 validateInput 前置匹配 + 读状态缓存；`backfillObservableInput` 把 file_path 展开为绝对路径供 hook 匹配。
- **GlobTool**（GlobTool.ts:26）：`{pattern, path?}`；maxResults 默认 100（`globLimits`）；相对路径化省 token。
- **GrepTool**（GrepTool.ts:33）：`{pattern, path?, glob?, output_mode('content'|'files_with_matches'|'count'), -A/-B/-C, -n, head_limit(默认250), multiline}`；内部调 ripgrep（`src/utils/ripgrep.ts`）；绝对路径转相对。

## 5. WebFetch / WebSearch

- **WebFetchTool**（WebFetchTool.ts:24）：`{url, prompt}`（prompt 用于对抓取内容提问）；`checkPermissions` 按 hostname 走规则+`isPreapprovedUrl`（preapproved.ts）；`getURLMarkdownContent` 抓取；重定向 3xx 时返回让模型用新 URL 再调。
- **WebSearchTool**（WebSearchTool.ts:25）：`{query, ...}`；**实现方式奇特**：`call()` 内用 `queryModelWithStreaming`（`services/api/claude.js`）**再起一次 agent 循环**（`querySource: 'web_search_tool'`），从流中解析子代理的 WebFetch 调用并透传进度（`query_update`），最终汇总为搜索结果文本。

## 6. AgentTool（src/tools/AgentTool/）

- **schema**（AgentTool.tsx:82/110）：`{description, prompt, subagent_type?, model?('sonnet'|'opus'|'haiku'), name?, team_name?, mode?(permissionModeSchema: default/acceptEdits/plan/bypassPermissions/auto), isolation?('worktree'|'remote'), run_in_background?, cwd?}`；输出 `{status:'completed'|'async_launched'|'teammate_spawned', ...}`。
- **checkPermissions**：除 auto 模式外一律 `allow`（权限下放给子代理自己的工具）。
- **代理定义加载**：`getAgentDefinitionsWithOverrides(cwd)`（loadAgentsDir.ts:296，memoize）＝内置（`builtInAgents.ts`）+ plugin 代理 + `.claude/agents/*.md`（`parseAgentFromMarkdown` 解析 frontmatter：name/description/tools/model/permission-mode/hooks/mcp 等）；`getActiveAgentsFromList` 过滤（MCP 依赖 `filterAgentsByMcpRequirements`）。颜色：`agentColorManager.ts` 的 `setAgentColor/getAgentColor`（存 `bootstrap/state.ts` 的 agentColorMap，UI 用 AGENT_COLORS 主题色）。
- **执行**：`runAgent()`（runAgent.ts:248，async generator）——`createSubagentContext(toolUseContext,{agentId, agentType})` 构造子上下文，然后 **`for await (const message of query({...}))` 复用主循环**（`src/query.ts`，非独立 mainAgentLoop）；`permissionMode` 继承/覆盖、`isolation:'worktree'` 创建临时 git worktree；清理：`killShellTasksForAgent`、hook 清理、todos 移除。另有 fork 路径（`forkSubagent.ts`）、`resumeAgent.ts` 恢复、Task 状态注册（`createTask`/`utils/tasks.ts`，AgentTool.tsx:759/1046 注册进度）。
- Task.ts 只是任务状态类型/工厂（`isTerminalTaskStatus`、`generateTaskId`、`createTaskStateBase`）。

## 7. SkillTool / MCPTool / LSPTool

- **SkillTool**（SkillTool.ts:291）：`{skill, args?}`；validateInput 校验技能存在且为 prompt 型；checkPermissions 按 `getRuleByContentsForTool('deny'/'allow')` + `skillHasOnlySafeProperties` 自动允许；`call()` 展开 slash-command（`processPromptSlashCommand`），fork 型技能在**子代理**里跑（`runAgent`），返回 `{success, commandName, agentId?, result}`。
- **MCPTool**（MCPTool/MCPTool.ts:27）：**只是一个模板桩**——`name:'mcp'`、`inputSchema: z.object({}).passthrough()`、`checkPermissions→passthrough`；真实实例在 `src/services/mcp/client.ts:1766` 用 `{...MCPTool, name: 'mcp__server__tool', mcpInfo, inputJSONSchema: tool.inputSchema, isReadOnly/isConcurrencySafe←annotations.readOnlyHint, isDestructive←destructiveHint, alwaysLoad←_meta['anthropic/alwaysLoad']}` 构造，`call()` 走 `callMCPToolWithUrlElicitationRetry`（-32042 elicitation 重试）。
- **LSPTool**（LSPTool/LSPTool.ts:59）：`ENABLE_LSP_TOOL` 启用；operation 判别联合（`textDocument/definition|references|hover|documentSymbol|callHierarchy...`）；`validateInput` 校验 operation；只读；`call` 经 `getLspServerManager()` 发 JSON-RPC 请求，结果按 operation 过滤/格式化（`formatters.ts`、`symbolContext.ts`）。

## 8. Plan/Worktree/多代理/任务工具

- **EnterPlanModeTool**（EnterPlanModeTool.ts:36）：无参数；`call()` 禁用于 agent 上下文；`handlePlanModeTransition(mode,'plan')` + `applyPermissionUpdate(prepareContextForPlanMode(...), {type:'setMode',mode:'plan'})` 把 `ToolPermissionContext.mode` 切到 `'plan'`（`prePlanMode` 记录原模式）。`shouldDefer: true`。
- **ExitPlanModeV2Tool**（ExitPlanModeV2Tool.ts:147）：输入 `{allowedPrompts?: {tool:'Bash', prompt}[]}`（SDK 侧由 normalizeToolInput 注入 `plan/planFilePath`）；`validateInput` 非 plan 模式拒绝；非 teammate `checkPermissions→ask('Exit plan mode?')`，teammate 走 `plan_approval_request` 信箱；`call()` 把 mode 恢复为 `prePlanMode ?? 'default'`（含 auto 模式 gate 熔断回退、dangerous rules 恢复）。
- **EnterWorktreeTool**（EnterWorktreeTool.ts:52）：`{reason, name?}`；`createWorktreeForSession` 后 `process.chdir` + `setCwd`，`saveWorktreeState`；ExitWorktreeTool 反向。
- **SendMessageTool**（SendMessageTool.ts:67）：`{to('*'广播/teammate名/uds:/bridge:), message: string|structured, summary?}`；structured 消息类型含 `shutdown_request/approval`、`plan_approval`；走 `teammateMailbox` 或 UDS；跨机器 bridge 消息 `checkPermissions→ask`。
- **TeamCreateTool**（TeamCreateTool.ts:37）：`{team_name, description?, agent_type?}`；写 TeamFile（`.claude/teams/`）、重置任务列表、AppState.teamContext 注册 lead；`isAgentSwarmsEnabled()` 门控。TaskCreateTool（`:18`）：`{subject, description, activeForm?, metadata?}` → `createTask`；TaskUpdateTool 更新任务状态。

## 9. SyntheticOutputTool 与错误处理

- **SyntheticOutputTool**（SyntheticOutputTool/SyntheticOutputTool.ts:28）：名 `'StructuredOutput'`（`SYNTHETIC_OUTPUT_TOOL_NAME`），仅非交互会话启用；`inputSchema: z.object({}).passthrough()`；`call()` 原样返回输入作结构化输出。真正用法是 `createSyntheticOutputTool(jsonSchema)`（:116，WeakMap 缓存 Ajv）：校验用户 JSON Schema（`ajv.validateSchema`）后编译，把 `inputJSONSchema` 注入工具并让 call 用 `ajv.compile` 校验模型输出，不匹配抛 `TelemetrySafeError`。此工具不进 `getTools()`（tools.ts:304 specialTools 剔除），由 SDK/QueryEngine 动态创建注入。
- **错误处理**：本快照**没有** `PermissionError`/`InterruptedError` 类（全库 grep 无）。权限拒绝以 `PermissionResult`（deny/ask）返回，由权限层展示 UI；执行异常统一在 `toolExecution.ts` 的 `streamedCheckPermissionsAndCallTool` 外层 catch（:469）打包成 `tool_result`：`content: '<tool_use_error>Error calling tool (Name): msg</tool_use_error>'`、`is_error: true` 回喂模型；中断/取消输出 `CANCEL_MESSAGE`；Shell 失败抛 `ShellError`（utils/errors.ts:51）。错误类：`ClaudeError/AbortError/ConfigParseError/ShellError/TelemetrySafeError`。
- **Schema 生成**：`src/utils/zodToJsonSchema.ts:17` 用 Zod v4 原生 `toJSONSchema` + WeakMap 按 identity 缓存；`src/utils/api.ts:160/541` 组装 API 工具时优先用 `tool.inputJSONSchema`（MCP/StructuredOutput），否则 `zodToJsonSchema(tool.inputSchema)`，`strict:true` 工具加 `strict` 标志，`shouldDefer` 工具加 `defer_loading`。

## 工具系统速查表

| 项 | 位置 | 要点 |
|---|---|---|
| Tool 接口 | src/Tool.ts:362 | Zod inputSchema + inputJSONSchema 双轨；isReadOnly/isConcurrencySafe/isEnabled/isDestructive/checkPermissions/validateInput |
| 默认实现 | Tool.ts:757 `TOOL_DEFAULTS` / `buildTool` | 默认 enabled、非并发安全、非只读、权限 allow |
| 注册表 | tools.ts `getAllBaseTools`/`getTools`/`assembleToolPool` | 数组型 Tools；deny 规则预剔除；内置优先去重 |
| Bash 执行 | BashTool.tsx → Shell.ts:316 `spawn` | shell 子进程、TaskOutput 文件/管道、wrapSpawn 超时+树 kill |
| Bash 权限 | bashPermissions.ts `bashToolHasPermission` | 精确→前缀/通配→sed→只读；分类器+沙箱自动允许 |
| Edit 校验 | FileEditTool/types.ts + validateInput | 前置匹配校验；无 postToolUse 复核 |
| Agent 加载 | loadAgentsDir.ts:296 | 内置+plugin+.claude/agents/*.md；memoize |
| Agent 运行 | runAgent.ts:248 → query.ts | createSubagentContext + 复用 query() 主循环 |
| MCP 实例化 | services/mcp/client.ts:1766 | 展开 MCPTool 桩：inputJSONSchema/annotations/mcpInfo |
| 流式 | Tool.ts `ToolCallProgress` + types/tools.ts | 按工具类型的 progress 事件 |
| 错误回喂 | toolExecution.ts:469 | `<tool_use_error>` + is_error:true |
| Schema 转换 | utils/zodToJsonSchema.ts | Zod v4 toJSONSchema + WeakMap 缓存 |
