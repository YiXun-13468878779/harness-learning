# DeepSeek Harness（dsh）深度架构解读报告

> **核心结论**：dsh 基于 vendored Cordis，把 agent 循环、工具注册表、会话日志、模型适配器全部做成插件；以能力接缝、append-only 会话日志 + 可执行不变量、平台沙箱链 + Landlock、类型图生成 RPC（Typert）实现可替换、可审计、可嵌入的运行时。

> 调研对象：`/Users/bytedance/deepseek-harness`（deepseek-ai/deepseek-harness 官方仓库完整源码）
> 调研方式：read/glob/grep 实际阅读源码（vendor/、packages/、apps/、docs/、python/、native/），所有结论均落到具体文件路径与导出符号；未贴大段源码，仅精炼引用。
> 用途：作为与 Claude Code 等主流 agent CLI 对比的素材。

> **读图提示**：如果你不熟悉 Cordis、事件溯源或 capability seam，建议先看 [04 可视化深度架构指南](04-visual-architecture-guide.md) 的 DeepSeek Harness 部分，再回到本文核对源码路径与实现细节。

---

## 0. 一句话概括

dsh 是一个 **基于 vendored Cordis 插件框架的、一切皆插件的 AI 编码代理运行时**：模型适配器、工具注册表、会话日志、agent 循环本身、沙箱、审批、子代理、工作流、技能、计划模式……全部是挂在一个 Cordis context 上的插件（函数/类/对象插件），通过服务依赖注入、类型化事件（emit/waterfall/parallel/serial/bail）与可逆 effect 组合成一颗插件树。**没有特权内核需要打补丁**——扩展 dsh 就是往配置树里加一行插件。

---

## 1. 总体架构与插件模型

### 1.1 Cordis 是什么

Cordis 是 Koishi 生态的插件框架（上游 `cordiverse/cordis`），dsh 把它**源码 vendor 进仓库**（`vendor/cordis/`，版本 4.0.0-rc.7，见 `vendor/README.md` 的 manifest 表，含上游 commit SHA），并统一 rescope 为 `@deepseek-ai/*` 命名空间（`cordis` → `@deepseek-ai/cordis`），所有 harness 包声明它为 peerDependency。vendoring 的理由：**完全拥有框架层**（可审计、可打补丁、可固定版本）。`vendor/README.md` 记录了 18 条本地修改（如 `fiber.ts` 生命周期加固、`include` 的 `applyEntryPatches` 导出、`disabled: !!js` 插值等）。

Cordis 核心机制（`vendor/cordis/src/`）：

| 概念 | 文件 | 关键符号 | 机制 |
|---|---|---|---|
| 上下文 | `context.ts` | `Context`（Proxy）、`extend()`、`isolate(name, label)`、`intercept(name, config)` | context 是**服务仓库**：属性读取走 Proxy 解析服务；`isolate` 给某个服务换独立作用域（preset 的 isolate realm 就用它）；`intercept` 给下游插件合并配置 |
| 服务 | `service.ts` | 抽象类 `Service`、`super(ctx, name)`、`[symbols.invoke]`、`[symbols.resolveConfig]` | 服务注册 `ctx.<key>`（如 `ctx.tools`、`ctx.llm`、`ctx.sessions`），随所属 fiber 自动卸载 |
| 事件 | `events.ts` | `ctx.on/once/emit/parallel/serial/bail/waterfall`，`DispatchMode = 'emit'\|'parallel'\|'serial'\|'bail'\|'waterfall'` | **类型化事件**通过 TypeScript declaration merging 声明在 `declare module '@deepseek-ai/cordis' { interface Events { ... } }` 里 |
| 插件注册 | `registry.ts` | `ctx.plugin(plugin, config)`、`ctx.inject(deps, cb)`、`Inject` 装饰器、`Plugin.Function/Constructor/Object` 三种形态 | 依赖注入驱动加载顺序（`inject` 数组/对象），服务未就绪就等待 |
| 可逆 effect | `fiber.ts` | `ctx.effect()`、Fiber、`FiberState` | **注册即 effect**：每个注册返回 disposer，fiber 卸载时按注册顺序逆序撤销 |

**Cordis 五种事件派发模式**（`docs/cordis-primer.md`）：`emit`（同步、不 await、无返回值）、`waterfall`（围绕中间件：`(...args, next)`，调 `next()` 委托，不调则短路）、`parallel`（并发 await 全部）、`serial`（按序 await 直到第一个 bail 值）、`bail`（同步短路）。dsh 事件 JSDoc 强制标注 `@mode`，由 `scripts/gen-cordis-catalog.ts` 生成目录并核对派发点与声明一致。

**waterfall 语义**（primer §"Cordis Waterfall Semantics"）：waterfall 是"around-middleware"，监听者必须调 `next()` 才委托给链上更内层（最终到内置行为）；`next()` 的返回值传播值；`prepend: true` 可插队。dsh 的 AGENTS.md 明确约定"Waterfall listeners MUST call `next()` to delegate; returning without it short-circuits the chain"。

### 1.2 "Everything is a plugin" 如何落地

`docs/architecture.md` 开宗明义：*"Every part of the product is a plugin, including the model adapter, the tool registry, the session log, and the agent loop itself, so every part is replaceable from configuration."*

证据链：

- **model adapter 是插件**：`packages/llm/llm-deepseek/src/adapter.ts` 的 `DeepSeekAdapter extends LlmAdapter`，经 `ctx.llm.registerAdapter(providers, adapter)` 注册；配置行在 `packages/bundle/base/cordis.patch.yml` 的 `- id: llm-deepseek`。
- **tool registry 是插件**：`packages/core/tools/src/index.ts` 的 `ToolRuntime extends Service`（`ctx.tools`），它自己也是 `- id: tools` 配置行；工具经 `ctx.tools.register(defineTool({...}))` 注册。
- **session log 是插件**：`packages/core/session/src/index.ts` 的 `SessionStore extends Service`（`ctx.sessions`）。
- **agent loop 本身是插件**：`packages/core/agent-loop/src/index.ts` 的 `AgentLoop extends Service implements AgentFactory`（`ctx.agentLoop`），`core/agent/src/index.ts` 的 `AgentRegistry extends Service`（`ctx.agents`）。
- **整个运行中的 dsh 是一棵插件树**，由 Loader（vendored `@deepseek-ai/cordis-plugin-loader`）从配置文件挂载。

### 1.3 Profiles 与 Bundles 的组装机制

**Profile**（`packages/boot/app-boot/src/profile.ts`）：`$DSH_HOME/profiles/<name>/` 目录，含 `package.json`（`dsh.profile.bundles` 有序 bundle 列表 + out-of-tree 插件依赖）与 `cordis.patch.yml`（用户补丁层）。`PROFILE_TEMPLATES = { web: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-web-app'], headless: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-headless'] }`。

**Bundle**（`packages/bundle/*/`）：一个 npm 包，`package.json` 里 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }` 声明自己导出的补丁层；`src/index.ts` 常是 `export {}`（纯配置包）。

**组装顺序**（`apps/cli/src/profile-boot.ts` 的 `composeProfile`/`composeLive`，`packages/boot/app-boot/src/profile.ts` 的 `composeEntries`）：**空根配置 → 每个 bundle 的补丁层（按 `dsh.profile.bundles` 顺序）→ profile 自己的 `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml`（机器级偏好）→ `--patch` overlays → telemetry 开关**。补丁按行 id 寻址，**整行替换 config**（不是 merge），因此同一行只在"一个 bundle 层 + 用户层"出现。`applyEntryPatches`（vendored include 的导出）是唯一补丁算法，`dsh --dump-config` 用它打印实际挂载的树。

`dsh-base` 的 `cordis.patch.yml`（451 行）就是**整个产品的清单**：timer、hmr、llm、session、typert 三件套、session-title、agent、agent-default-model（deepseek-official / deepseek-v4-flash）、jobs、llm-retry、settings、credentials、llm-pi-ai、session-persistence-jsonl、session-query-sqlite、sandbox、sandbox-policy（`DSH_PERMISSION_MODE ?? 'workspace-write'`）、bash/pwsh-sandbox、approval、permission-presets、tool-bash/tool-pwsh、fs-observation-policy、tool-fs、skill 全家、goal、plan-mode、token-meter、compaction、subagent 全家、workflow、tool-todo、tool-goal、tool-ralph、web、tools、system-prompt、agent-loop、fs-sandbox、llm-deepseek……可见**默认产品的每一个能力都是一行可补丁的插件**。

### 1.4 scoped context（每个 agent 一个世界）

`packages/core/scope/src/index.ts`：`createScope(ctx, key)` 铸造带 `kScope` 标签的子 context；`scopeOf(ctx)` 读最近标签；`scopeTarget(base, key)` 铸造**纯路由 carrier**（事件派发 `thisArg`，实现"agent 作用域的监听器只收到该 agent 的事件"）；`bindScopeParent`/`scopeChainOf` 建立 scope 父子链（**监听向祖先流动**：父 scope 的监听器接收所有后代 agent 事件）。`ScopedLayers`（`store.ts`）是注册表的层级容器：**子 scope 的注册遮蔽全局**，卸载即撤销。这是"per-agent 组合"（preset）与"tool 注册按 agent 隔离"的底层原语。

---

## 2. Agent 循环（core/agent + core/agent-loop）

### 2.1 词汇：turn 与 step

`docs/architecture.md` § Turn flow：**step = 一次模型请求 + 它调用的工具；turn = 零个或多个 step**，turn 在首个输入被 claim 前打开，在"无欠账"时关闭。对应时序图 `docs/agent-lifecycle.md`（mermaid，精确到事件签名）。

### 2.2 驱动流（ReactLoopAgent）

`packages/core/agent-loop/src/agent.ts` 的 `ReactLoopAgent implements Agent` 是默认驱动：

```
turn() 循环:
  turn/start 事件
  preStep(target, {turn, step})  → claim inbox + systemPrompt.assemble + agent/pre-step waterfall
    reject → turn 以 {kind:'blocked'} 关闭（不花任何 step）
  step/start 事件 → 把 claim 的消息逐个 append user/message
  step(assembly):
    buildRequest()  → request/header 事件（首次 initial / resume / change）+ request/context 事件
    llm.stream(request) → 逐 chunk append assistant/chunk → BlockAssembler 组装
    assistant/message 事件（usage、sourceEventSeqs 关联 chunk seq）
    finish.kind==='error'/'aborted' → agent/request-error waterfall（retry?）
    tool-call block → executeToolCalls() → 每个调用 append tool/call → 结果按模型序 append tool/result
  step/end 事件
  turn 结束条件（turnEnds && nextStep inbox 空）→ agent/turn-stopping (serial) → turn/end 事件
```

**inbox 双队列**（`core/agent/src/inbox.ts` 的 `Inbox`）：`next-turn`（普通提示队列）与 `next-step`（steering/injected context）。`claim(target, turn)` 一次取走全部 next-step + （target=next-turn 时）一条 next-turn。inbox 的每次变更都是**持久化事件** `agent/inbox/spliced`（`core/agent/src/types.ts` 里 SessionEventMap 扩展），live 派发先于投影变更——**inbox 本身就是日志的可重放投影**。

### 2.3 四种消息入口（`Agent` 接口，`core/agent/src/runtime-types.ts`）

- `followup(content)`：排入 next-turn 并唤醒驱动（普通用户追问）。
- `steer(content)`：排入 next-step 并唤醒（运行中在最近 step 边界消费）。
- `inject(content)`：排入 next-step 不唤醒（模型可见上下文，等下一个唤醒消费）。
- `send(message, target, wakeup)`：统一原语。
- `cancel(cause, {keepInbox})`、`whenIdle()`、`runMaintenance(task)`：取消/静默/维护任务。

### 2.4 agent/* 事件域（`core/agent/src/runtime-types.ts` 的 `declare module Events`）

| 事件 | @mode | 职责 |
|---|---|---|
| `agent/created` / `agent/disposed` | emit | 生命周期（disposed 在驱动静默后、scope 撤销后发） |
| `agent/status` | emit | idle ⇄ running |
| `agent/inbox/inserted` / `claimed` / `discarded` | emit | inbox 变更通知 |
| `agent/session-start` | emit | 会话生命周期开始（startup/resume/clear/compact），**首个注入点** |
| `agent/pre-step` | **waterfall** | 决定模型看到什么：reject 或重写进入的 messages |
| `agent/request` | **waterfall** | 替换冻结的请求配置（provider/model/maxTokens/…） |
| `agent/request-error` | **waterfall** | 处理失败请求（llm-retry 与 compaction 都挂这里） |
| `agent/turn-stopping` | **serial** | turn 关闭前检查点；监听者可 steer 追加一步 |
| `agent/error` | emit | 错误通知 |

三道/四道 waterfall 汇总：**pre-step（进入）、request（请求构造）、request-error（错误恢复）、llm/stream（流拦截）**——加上 tools 三道（pre-execute/execute/post-execute），即架构文档说的"pre-step/request/llm-stream/tools 三道 waterfall"（request-error 是 agent 域第四道）。

### 2.5 initiator scope（进程内因果链）

`core/agent/src/index.ts` 的 `AgentRegistry` 用 `AsyncLocalStorage` 维护 `withInitiator(agent, op)`：驱动链上所有异步操作都能用 `ctx.agents.requireInitiator()` 找回发起 agent（tool-calls.ts 用它拿 `agent.session` 记日志）。这是**进程内因果归属**，不是授权（文档明示 "presence is neither liveness proof nor authorization"）。

### 2.6 turn-stopping 与工具循环

`agent/turn-stopping` 是 serial 检查点：模型不再欠响应（无活工具调用、无新 steering）时在 turn/end 提交前等待；监听者若 objection 就 `agent.steer()`，机器重读 inbox——**fresh steering 多跑一步，没有则关 turn**。反向控制是数据：工具结果带 `concludesTurn: true` 时该 step 结束即关 turn（`ToolRunContext.concludeTurn()`）。

---

## 3. 会话日志（Session log）

### 3.1 事件源模型

`packages/core/session/src/index.ts` 的 `Session`（普通类，非 Service）：**append-only 的 `SessionEvent[]` 日志是会话唯一事实源**。`append(type, data, opts)` 是唯一写入入口：`snapshotJsonValue` 单遍 lossless-JSON 校验+深拷贝（拒绝 BigInt/函数/循环/稀疏数组/Map/Set/Date……），坏事件在 append 现场失败；`deepFreeze` 冻结事件与嵌套数据；热路径不阻塞 I/O，持久化是异步订阅者的事。事件信封固定 `{type, seq, time, data, surfaceOp?, sourceEventSeqs?, ignorable?}`，**`seq = log.length` 连续契约**是全局依赖。

### 3.2 事件词汇表 `SessionEventMap`（`src/types.ts`）

核心事件（插件经 declaration merging 扩展，仓库共 45 种）：`turn/start`、`turn/end`（`{turn, reason: TurnEndReason}`）、`step/start`、`step/end`、`user/message`、`assistant/chunk`（token 级重放保真）、`assistant/message`（+usage）、`tool/call`（arguments 为模型原始 JSON 字符串）、`tool/result`（+error/meta）、`request/header`、`request/context`、`session/end-seed`。三类**surface 事件**：`user/message`、`assistant/message`、`tool/result`（`SurfaceEventType`），它们必须带 `surfaceOp` 标记，是唯一能派生模型历史的来源。

### 3.3 deriveMessages() 与 surface

- `deriveMessages()`（`index.ts:726`）：遍历 `surface.nodes`（seq 数组），逐节点调纯函数 `deriveEventMessage`（`surface.ts:83`：user→原样、assistant→消息（**空 content 跳过**，它只是 max-tokens step 的 usage 宿主）、tool/result→带 tool-result 块的 user 消息、其余→null），带缓存（每节点投影一次，`replaceGeneration` 变化整体重建）。
- **surface 机制**（`surface.ts`）：`SurfaceOp = 'append' | { op: 'replace', start, end }`。append 尾部追加；replace 遮蔽 [start,end] 区间（compaction 用），其 `sourceEventSeqs` 必须覆盖每个被遮蔽节点（`assertProvenance`）；`SurfaceManager` 增量折叠，`foldSurface` 全量重放。**人类可读 transcript 读 append-origin 事件**（`isAppendSurfaceEvent`），被替换的区间对模型隐藏但对人可见。

### 3.4 "model-visible ⟺ logged" 不变量

三重执行（AGENTS.md 约定：*anything that reaches a model request must be reconstructable from the session log*）：

1. **agent-loop invariant**（`core/agent-loop/src/invariant.ts`，经 `ctx.invariants.register` 挂到 `llm/stream` prepend）：断言 loop 请求 frozen、带 sessionId、日志有 step/start 与 request/header、**`JSON.stringify(options.messages) === JSON.stringify(session.deriveMessages())`**、请求头与 `foldRequestHeader(events)` 逐字段一致。
2. **checkpoint policy**（`session/session-checkpoint-policy/src/index.ts`）：`llm/stream` 派发前 `await ctx.sessions.flush(session)`、`tools/execute` 顶层调用前 flush、`agent/pre-step` 请求前 flush——**持久化先于可见，失败即不派发**。
3. **session invariant**（`core/session/src/invariant.ts`）：维护 trace（lastSeq/openTurn/openStep/pendingCalls），断言 seq 递增、turn/step 编号连续、执行事件 turn 封闭、tool/result 必须同 step 有 tool/call。

### 3.5 持久化（JSONL + SQLite）

`packages/session/session-persistence/`：`SessionPersistence` 抽象 + `PersistenceCoordinator`（后端无关编排：`session/event` 入队 → `SessionWriteBehind` 默认 200ms 批窗口 → 每 id 串行链 → `appendCore` 校验 seq 连续 → `backend.appendBatch`；崩溃修复 `prepareCore` 用 `interruptedTurnClosers`（`core/session/src/repair.ts`）合成 `TOOL_NOT_STARTED`/`TOOL_OUTCOME_UNKNOWN` 错误结果 + step/end + interrupted turn/end）。

- **JSONL 后端**（`session-persistence-jsonl/`）：每会话一个文件 `<root>/--<projectKey(cwd)>--/<encodeSegment(id)>/session.jsonl[.zstd]`；首行 header record（`toHeaderLine`）；事件行默认 `packChunks: true`，`packChunkRuns`（`core/session/src/chunk-rows.ts`）把 ≥3 个连续同块 delta chunk 打包成 `text-chunks`/`reasoning-chunks`/`tool-call-chunks` 一行（~56× 体积下降，无损、布局无关解码）；zstd 用 checksummed 帧容器（`zstd.ts`），可追加、可 torn-frame 恢复；materialize 用 temp 写+fsync+`link()+unlink()` 发布（防并发覆盖）。
- **SQLite 后端**（`session-persistence-sqlite/`，`SCHEMA_VERSION = 15`，application_id `0x44534850`）：`sessions` 表（materialization 信号）+ `events` 表（`PRIMARY KEY(session_id, seq)`），journal WAL，**版本不匹配拒绝而非迁移**（与 SESSION_FORMAT_VERSION 立场一致）。

### 3.6 格式版本：`SESSION_FORMAT_VERSION`

`core/session/src/types.ts`：`SESSION_FORMAT_VERSION = 0`（单一整数、无 major/minor；预发布期钉 0，**不兼容日志直接拒绝，无迁移承诺**）。**谁决定 bump：写者**——仅结构性变化（header 形状、事件信封、核心事件语义、surface 机制）才 bump；普通新增事件类型不 bump，交给 per-event `ignorable: true` 标记（未知类型无标记=required，必须拒绝重构）。`KNOWN_SESSION_EVENT_TYPES` 由 `scripts/gen-persistence-catalog.ts` 从所有 SessionEventMap merge 生成，`verify-persistence-catalog` 保鲜。方向化拒绝：新于读者报"请升级 harness"，旧于读者报"无升级路径"。

### 3.7 投影 / 查询 / 标题 / 遥测

- **Projection**（`session-projection/`）：`ProjectionDefinition<K, S>`（纯同步 `init/apply/view` + zod schema + `stateVersion`），**whole-value 规则**（携带状态的事件必须存完整后状态，如 `todo/write`、`session/title`）；`SessionProjectionRegistry`（`ctx.sessionProjections`）订阅一次 `session/event`，引用变化才通知；`session-projection-cache` 落 `session_projcache` KvTable，带身份绑定防 id 复用。
- **Query**（`session-query/`）：`SessionQueryEngine`（live-preferred 逻辑语料）+ `SqliteSessionQueryEngine`（**可丢弃的派生全文索引**：FTS5 unicode61 + `live_sessions`/`persisted_docs` 代数式 `_reconcile`，`SESSION_QUERY_SQLITE_SCHEMA_VERSION = 8`）；`tool-session-query` 暴露 5 个模型操作（搜索/追踪/读取，带 workspace-access 过滤）。
- **Stats/Title/Telemetry**：`session-stats` 投影（以 step/end 为步计数）；`session-title` 标题本身入日志 `session/title` 事件；`session-telemetry` 捕获 + `session-telemetry-otel` OTLP 导出（默认 DISABLED，`DSH_TELEMETRY_MODE` 显式开启）。

---

## 4. 能力接缝（Capability Seams）

### 4.1 三角色模式

`docs/glossary.md` + Agent Note `2026-06-13-capability-seams.md`：**seam = Service Definition（Cordis `Service`，拥有 `ctx.<key>` 与词汇类型，是抽象类或具体 registry，绝不是 TS interface）+ Service Provider + Consumer（通常是模型面对的工具）**。"one role alone is not a seam"——dsh 的 AGENTS.md 强制"capability seam comprises Service Definition / Service Provider / Consumer roles… complete, never one role"。`packages/shell` 是规范模板（`dsh-shell` / `dsh-bash-local`+`dsh-bash-sandbox` / `dsh-tool-bash`）。三角色因**变化速率不同**而拆包；"provider 交换不触及模型可见 schema"是核心收益；**LLM seam 是例外**（Definition+Consumer 折叠在 `dsh-llm`，因为 Consumer 就是 loop 本身）。全量图谱见生成文件 `docs/capability-seams.md`（每 `ctx.*` 的 Role/Owner/Implementations/Direct consumers/Companion plugins）。

**关键后果（"一个 provider 交换改变整个产品"）**：fs 与 subprocess provider 共享一个执行世界，把两者指向远程沙箱（E2B），Bash、PTY、LSP 全部跟着换，**无 provider 分支**。`packages/e2b/`（E2B POC：`fs-e2b extends FileSystem` + `subprocess-e2b extends SubprocessRuntime` 都 await 同一 `ctx.e2b.getSandbox()`，文件与进程操作住在同一远程 Linux 世界）是"整 seam 兄弟实现"的证据——**容器/microVM/远程执行器不是 `ctx.sandbox` 的后端，而是以"环境一致组"整体替换 `ctx.shell`/`ctx.fs` 的 Service Provider**。

### 4.2 实例

| Seam | Service Definition | Providers | Consumers | 关键机制 |
|---|---|---|---|---|
| **fs** | `packages/fs/fs/src/index.ts`：抽象类 `FileSystem`（`ctx.fs`），事件 `fs/write-intent`、`fs/edit-intent`（单槽 waterfall 决策）、`fs/observed`（同步记录） | `fs-local`（`LocalFileSystem`）、`fs-sandbox`（`SandboxedFileSystem extends LocalFileSystem`——**继承者替换**，"loading it INSTEAD OF dsh-fs-local is the whole swap"） | `tool-fs`（read/write/edit/read_image）、`tool-fs-search`（经 ctx.subprocess 跑 ripgrep：glob/grep）、`tool-str-replace-editor` | 无 ReadOptions 对象，是 `expected?/signal?/sandboxPolicy?` 分离参数；`processPath()/fileUrl()/contains()` 让 subprocess 与 fs 共享执行世界 |
| **shell** | `packages/shell/shell/src/index.ts`：抽象类 `ShellExecutor`（`ctx.shell`），**`resolve(request): ShellExecSpec` 显式默认化**（仓库"explicit > implicit"模板） | `bash-local`、`bash-sandbox`（**bash-local 的沙箱子类**，经 `ctx.sandbox.confine(['bash','-c',command])` 包装 argv）、`pwsh-local`、`pwsh-sandbox` | `tool-bash`、`tool-pwsh`、`tool-bash-persistent`、`shell-env` | 工具层只读能力事实 `ctx.shell.sandboxMode`（undefined 则不广告 `sandbox_permissions` 升级字段） |
| **subprocess** | `packages/subprocess/subprocess/src/index.ts`：抽象类 `SubprocessRuntime`（`ctx.subprocess`）：`spawn(spec)`、`spawnTerminal(spec)`；`scrubbedParentEnv()`（`SENSITIVE_ENV_PATTERN = /KEY|PASSWORD|SECRET|TOKEN/i` 清洗环境） | `subprocess-local` | shell provider、tool-bash、workflow worker、subagent 子进程 provider | `done` 在进程关闭时 resolve，spawn 级失败才 reject |
| **sandbox** | `packages/sandbox/sandbox/src/index.ts`：抽象类 `SandboxProvider`（`ctx.sandbox`）：`confine(argv, policy): ConfinedArgv` | `sandbox-local`（平台链：linux bwrap→landlock、darwin seatbelt、win32 windows-acl，`PLATFORM_CHAINS`）、`sandbox-windows-acl`、`sandbox-policy`（把 session 的 mode+cwd 解析成 `SandboxExecutionPolicy`） | tool-bash、fs-sandbox、guard | 见 §7 |
| **subagent** | `packages/subagent/subagent/src/index.ts`：`SubagentRuntime`（`ctx.subagents`）——**多 provider 共存按名注册**（仿 LLM adapter registry，与 shell 单执行器相反） | in-process：`subagent-spawn-in-process`（全新子 agent）、`subagent-fork-in-process`（父会话平衡前缀 seed）；out-of-process：`subagent-acp`、`subagent-claude-code`、`subagent-codex`、`subagent-dsh-sdk` | `tool-subagent`、`tool-subagent-control`（send_message/interrupt_agent）、`tool-subagent-report` | 见 §6.1 |
| **skill** | `packages/skill/skill/src/index.ts`：`SkillProvider` 接口（`list()/get()`）+ registry（`ctx.skills`） | `skill-filesystem`（本地 SKILL.md 目录）、`skill-badge` | `tool-skill`（加载 + `<available_skills>` 会话目录注入） | 见 §6.4 |
| **web** | `packages/web/web/src/index.ts` | `web-search-deepseek`、fetch provider（默认不挂） | `tool-web` | 事件 `web/search-result` 等 |

注意：shell/subprocess/sandbox/web/terminal/lsp 均为**纯方法服务**（无自定义事件）；只有 fs（`fs/*`）与 skill（`skills/change`）声明了事件——事件是"策略附着点"，不是每个服务的标配。

### 4.3 交叉机制（每 seam 的贯穿约定）

- **resolve(request): Spec 分离**：`ShellExecutor.resolve`（request 可选字段 → spec 全必填）与 subprocess（"this seam applies no defaults"）是模板；fs/lsp 因无实现默认值而无 resolve 步。
- **错误即契约**：每 seam 的 `XxxError extends HarnessError` + 稳定 code（`FsErrorCode` 13 码含 `FS_SANDBOX_DENIED`/`FS_STALE_VERSION`/`FS_NOT_OBSERVED`、`SANDBOX_UNAVAILABLE`、`WEB_DUPLICATE_PROVIDER`、`WEB_PROVIDER_AMBIGUOUS`…），工具注册表保留 `{name, code}` 供重试/权限/UI 分流。
- **能力门控 schema**：`sandboxMode` 能力事实决定工具 schema 是否出现升级字段（bash/fs 两个族一致）；`register*` 一律返回 disposer（HMR 安全），provider 同层重名即抛。
- **策略单一归属**：`ctx.sandboxPolicy.resolve()` 是唯一 home，bash-sandbox/fs-sandbox/terminal-bash 三个执行族读同一个 resolve——**bash 与 fs 不可能 confine 到不同根**。
- **terminal/lsp 跟随执行世界**：`terminal-bash` 的 PTY 经 `ctx.subprocess.spawnTerminal`（node-pty），且 `spawnArgv()` 里 `sandbox.confine(argv, {...policy, mode}).argv`（**PTY 会话本身也进沙箱**）；`lsp-stdio` 读源码走 `ctx.fs`、启服务器走 `ctx.subprocess`（"both local and remote implementations share one host"）——两者天然跟随 fs/subprocess 的 provider 交换。

---

## 5. 工具系统与执行管线（core/tools）

### 5.1 scoped tool registry

`packages/core/tools/src/index.ts` 的 `ToolRuntime extends Service`（`ctx.tools`）：

- `register(definition: ToolDefinition): () => void`——返回**精确 effect disposer**；scoped 注册（经 `agent.ctx`）遮蔽全局。
- `restrict({allow, deny})`——per-scope 全局工具掩码（空过滤器/未知名/保留名报错）。
- `guard(guard)`——**单调守卫**：注册在 pre-execute 之后、工具体之前；任何守卫返回 reason 即拒绝，**没有守卫能"放行"另一个守卫拒绝的调用**（listener 顺序不能把拒绝变回允许）。
- `presentAs(mode)`：`'native' | 'code' | 'both'` 工具呈现模式。**Code Mode** 是独特设计：只给模型一个 `run_code` 工具 + 从工具 schema 生成的 **TypeScript/Python SDK 提示**（`ts-types.ts` 的 `renderToolsSdk` / `py-types.ts` 的 `renderToolsSdkPy`），模型在程序内调工具，原生工具名被拒为 `UNKNOWN_TOOL`（附带"只能从 run_code 内调用"的路由提示）。`run_code` 是保留名，不可注册/遮蔽。
- `ToolDefinition`：`schema + output{schema, render, presentationMeta} + execute(args, exec) + finalizeContent? + timeoutMs? + isConcurrencySafe? + presentCall?/presentResult?`。**工具必须声明 canonical JSON output**（`ToolOutputError` 校验）；`presentCall/presentResult` 是纯函数 UI 渲染意图（`generic|terminal|diff|read|search` 卡片，从 args 派生，可重放）。

### 5.2 执行管线（pre-execute → execute → post-execute）

完整流程图见 `docs/tool-execution-pipeline.md`：

```
tool/call 事件（执行前记录）
→ tools/pre-execute waterfall（hooks/权限/沙箱）→ allow/deny/ask
  → ask 经 ctx.approval（allowed-once 唯一放行，否则 fail-closed deny）
→ 单调 guards（deny 或 abstain）
→ tools/execute waterfall（timeout/retry/metrics 的 around-dispatch 包装）
  → 工具体 tool.execute()
    →（tool-fs 变更额外走 fs/write-intent / fs/edit-intent 门）
    →（工具自有事件：todo/write、fs/observed、hook/invoked、tool/code-dispatch…）
→ tools/post-execute waterfall（accept 替换 content/value、block 转错误、附加 additionalContexts）
→ ToolDefinition.finalizeContent（同步内容最后变换，快照于执行开始）
→ tools/result（同步只读通知，frozen 权威结果）
→ tool/result 事件（单一模型可见结果）+ additionalContexts 注入 next-step inbox
```

**并行调度**（`core/agent-loop/src/tool-calls.ts` 的 `executeToolCalls`）：`executionMode` 分类 `exclusive`（形成屏障）与 `parallel`（有界滚动池，`maxParallelToolCalls` 默认见 `constants.ts`）；**policy/结果/上下文保持模型序**（ordered commit），dispatch 可重叠；abort 给未开始调用记录合成错误结果（`TOOL_ABORTED_BEFORE_DISPATCH`）保重放有效；内部调度失败不伪造结果。`TOOL_RUNTIME_SCHEDULER` 符号是 registry 的分阶段视图（prepare/dispatch/finalize/finish）。

### 5.3 具体工具实现样例

- **BashTool**（`packages/shell/tool-bash/src/index.ts`）：`defineTool({name:'bash', ...})`；`resolveWorkdir`（sandbox-policy root 优先）；`approveBashEscalation`（`sandbox_permissions`/`justification` 升级经 `approveEscalation`→`ctx.approval`）；`run_in_background` 经 `ctx.jobs.start`（返回 jobId，输出经 job_output/job_kill 收集）；foreground 走 `ctx.shell.run(ctx.shell.resolve({command, workdir, timeoutMs, dshEnv, sandboxPolicy, signal}))`；描述文本就是模型侧操作规范（含 `[exit code: N]` 标记、`[sandbox: ...]` 拒绝标记的语义）。**注意：dsh 的 bash 工具不使用持久 shell（`bash -c` 每次新 shell），持久终端是单独的 `tool-bash-persistent`/`terminal` seam。**
- **read**（`packages/fs/tool-fs/src/read.ts`）：一次 stat 决定类型/路由/观测版本；≥10MB 流式读（`STREAM_MIN_SIZE`）；`buildWindow` 限行限字节；读后 `ctx.emit('fs/observed', target, {kind:'present', version}, exec)`（观测策略输入）；`presentCall/presentResult` 纯函数渲染 read 卡片（带 lang 高亮、行号）。
- **write/edit**（`write.ts`/`edit.ts`）：经 `fs/write-intent`/`fs/edit-intent` 门；edit 是版本 CAS（`replaceIfVersion`）而非朴素字符串替换。
- **glob/grep**（`tool-fs-search`）：经 `ctx.subprocess` 跑真实 ripgrep，输出结构化匹配 + `locations` 渲染意图。

---

## 6. 子代理 / 工作流 / 目标 / 技能 / 计划 / todo

### 6.1 subagent：`ctx.subagents`（`packages/subagent/subagent/`）

- **Service Definition**：`SubagentRuntime extends Service`，多 provider 共存、按名注册（`registerProvider`/`getProvider`/`start`/`startContinuable`/`followup`/`interrupt`/`reportFrom`/`listChildren`），`subagent/provider-added`、`subagent/start`、`subagent/end` 事件。
- **Provider 契约**（`types.ts`）：`SubagentProvider { name, capabilities: {outputSchema, depthLimit, toolFilter, persona}, inheritsParentContext, start(request): Promise<SubagentRun> }`。`SubagentRun.result` **永不 reject**（失败以 `stopReason: 'completed'|'aborted'|'error'|'max-tokens'|'refusal'` 解析）。
- **两种 in-process provider**（注意：**都不是 worker_thread/child_process**，是同进程同 context 的子 Agent）：
  - `subagent-spawn-in-process`：全新子 agent（`inheritsParentContext=false`），共享驱动 `subagent-in-process-driver/src/index.ts` 的 `startInProcessRun()`（`ctx.agents.create` + setup + `whenIdle` + 从日志折叠 stopReason）。
  - `subagent-fork-in-process`：**会话 fork**——取父会话到最后一个 `turn/end` 的平衡前缀作 seed（in-flight turn 不可重放），`inheritsParentContext=true`，工具描述说"继承本对话"。
- **四种 out-of-process provider**（经 `ctx.subprocess.spawn` 真子进程）：ACP（`subagent-acp`）、Claude Code（`subagent-claude-code`）、Codex（`subagent-codex`）、dsh-sdk（`subagent-dsh-sdk`——子进程是完整 dsh runtime，stdio JSON-RPC）。共享词汇 `out-of-process.ts`（`NO_START_CAPABILITIES`、`settleRunResult`）。
- **Continuable 后台子代理**（`continuation.ts`）：`SubagentContinuationManager`——一个持久 Session 至多一个进程内 Activation，可跑多个 FIFO turn；`startContinuable` 返回 `{childId, messageId}`（inbox 接受即成功）；`followup` 按 Activation 状态路由（running 入队/waiting 唤醒/无则 `coldResume`）；`interrupt` 只发 `Agent.cancel(cause, {keepInbox:true})`；授权分 user/ancestor。这就是我（当前会话）作为子代理时 `send_message`/`interrupt_agent` 背后的机制。
- **Consumer**：`tool-subagent`（`backgroundMode: 'one-shot'|'continuable'`，`Config.provider` 点名 spawn/fork/acp…，provider 生命周期事件驱动工具挂载）、`tool-subagent-control`（全局 `send_message`、`interrupt_agent`）、`tool-subagent-report`（把 `report` 工具装进每个 continuable 子代理的 unpublished scope，经 `registerContinuableSetup`）。

### 6.2 workflow：`ctx.workflowEngine`（`packages/workflow/`）

- `WorkflowEngine extends Service`（`workflow/src/index.ts`），**单引擎**（与 subagent 相反）；`workflow/start|phase|log|agent-start|agent-end|end` 事件；`WorkflowRun.result` 永不 reject。
- **worker-thread provider**（`workflow-worker-thread/`）：每次 run 一个 `node:worker_threads.Worker`，脚本在 worker 内 vm context 执行（`runtime.ts` 的 `WorkflowExecution`：`new vm.Script('(async()=>{…})()')`）；**隔离阶梯**：vm（脚本 containment，非安全边界）→ worker_thread（可强制终止）→ child_process。编排原语注入为 frozen globals：`agent(prompt, opts?)`（仅 label/phase/schema/provider/model，其余选项拒绝为 `UNSUPPORTED_OPTION`；失败→null）、`parallel(thunks)`（barrier）、`pipeline(items, ...stages)`（**stage 间无 barrier**，item 失败→null）、`phase/log/args`。`tool-workflow` 是 Consumer；`tool-ralph` 用固定脚本跑 fresh-child Ralph 循环（structured report schema 校验 continue/complete/blocked）。

### 6.3 goal：同会话长期目标（`packages/goal/`）

- 事件源：`goal/change`（整快照变更，`goal/src/fold.ts` 严格 replay fold：revision 恰好 +1、phase 迁移合法、round 连续）。`GoalPhase = active|paused|blocked|complete`。
- `GoalService`（`ctx.goals`，TypertRemoteService 暴露给 wire）：create/edit/pause/resume/complete/block/clear；**resume/fork 后目标 disarmed**，需人工授权 resume 重新武装（`agent/session-start` 重置 activation）。
- `goal-round-driver`：agent idle + goal active+armed → 用 `<goal_round>` prompt 块构造 round 消息（`source: {kind:'goal', goalId, revision, round}`）→ `agent.followup` 入队；pre-step 围栏 `validReservation` 校验消息与 reservation 一致，失败 `block(code:'prompt-rejected')`；round 上限到则 `block(code:'round-limit')`。
- `tool-goal`：`get_goal/create_goal/update_goal`（authority：`requireDirectHuman`——root agent 当前 turn 内有 `source.kind==='user'` 消息；blocked 需 ≥3 连续轮）；`command-goal`：`/goal` 命令。

### 6.4 skill：`ctx.skills`（`packages/skill/`）

`SkillProvider { list(options), get(candidate, options) }` + registry（按 name 合并目录、rank 低者胜）；`tool-skill` 的 `skill` 工具加载 SKILL.md；会话技能目录：每个 pre-step 把 `<available_skills>` catalog 以 `skill-catalog` source 的 user message 注入（digest 去重、replacement 语义）；`/name` 用户手势把技能内容作为 instructions 注入。**注意**：dsh 的 skill 系统与当前会话的"可用技能目录"机制同源——`tool-skill` 把技能列表做成日志里的 user message 目录，模型按目录调用 `skill` 工具。

### 6.5 plan-mode：`ctx.planMode`（`packages/plan/plan-mode/`）

**Plan mode 是 logged per-agent 协作状态**：`plan/mode {active: boolean}` 事件（log-only、whole-value-replace、永不进模型 transcript），`foldPlanMode` last-wins 折叠——resume/fork/compaction 都能恢复，无 live mirror。写入时机刻意放在"下一个被接受的 in-turn pre-step"（session 事件必须 turn-enclosed）。`exit_plan_mode` 工具校验 markdown 以 `#` 开头 → user-questions 审批渠道（Approve/Keep planning）。prompt section `plan:policy` 按折叠状态渲染部署文案。

### 6.6 todo：`todo_write`（`packages/todo/tool-todo/`）

整表替换工具：`session.append('todo/write', {todos})`（**每次调用必须发全量列表**，无部分更新）；会话投影 `todos`（`todo/write`→整表、`turn/start`→null——turn 结束清单保留可见、下轮开始清空）；`allowParallelInProgress` 部署策略。

### 6.7 preset：`ctx.agentPresets`（`packages/preset/agent-presets/`）

**per-session 插件组合**：每个 preset 从自己的 `cordis.yml` 组合插件；**standing mount**（每个 preset 只 mount 一次，agent 经 `bindScopeParent(agentKey, standing.key)` 挂到 standing mount 之下——mount 的注册对 agent 可见、mount 的监听收到 agent 事件）；preset 发布的 service 放在 **isolate realm**（组外不可见，外部经 `serviceFor(agent, name)` 按 agent 读）；子 agent 同步继承父 agent 的同一 generation；`recompose` 仅限"零产出"agent。

---

## 7. 权限与沙箱

### 7.1 审批（`packages/interaction/`）

`user-approval/src/index.ts`：`ApprovalService`（`ctx.approval`），`ApprovalOutcome = 'allowed-once'|'rejected'|'cancelled'|'unavailable'` 封闭四值，**唯一授权是 allowed-once，其余全 fail-closed**。流程：`approval/request`（agent 作用域 waterfall）→ answerer（Web 端在 `host/apiproxy/src/api-proxy.ts:1422`——先反查日志定位自己的 `approval/asked` id 防并行工具调用偷配对，经 mux 流发帧、`POST /api/respond` 结算、断连存活；ACP 在 `acp/src/index.ts:215`——`requestPermission` 只做一次性选择）→ 审计对 `approval/asked`/`approval/decided` 落 session log（**必须包在 open turn 内**，`hasOpenTurn` 前置守卫，防跨 turn 裸事件被当崩溃尾巴丢弃）。`decide()` 的判定顺序：signal aborted→`'cancelled'`；策略 `'never'`→**在 service 内部**确定性拒绝（与监听器注册顺序无关）；否则 scoped `approval/request` waterfall，无应答兜底 `'unavailable'`；abort 与答案竞速时 abort 赢、晚到答案构造性丢弃。会话策略 `approval/policy {policy: 'ask'|'never'}`（log-only、可回放）。`tool-ask-user`（`ctx.userQuestions.ask`，前置守卫：调用者必须是 live runtime root——`CALLER_NOT_LIVE`、被别的 agent 拥有的子代理禁止问人——`DELEGATED_CALLER`）、`commands`（`CommandRuntime`，`command/run`+`command/done` 审计，**不经模型 turn**）、`permission-presets`（`read-only`/`workspace-write`/`danger-full-access` 三预设，每个 = `SandboxMode` × `ApprovalPolicy` 两个旋钮的捆绑，写路径经规范 setter 落 `sandbox/mode`+`approval/policy`+`permission/preset` 三个事件）。

### 7.2 沙箱（`packages/sandbox/` + `native/`）

- **`SandboxProvider.confine(argv, policy): ConfinedArgv`**（`sandbox/src/index.ts`）：统一形态 `[runner, profileArgs..., '--', ...argv]`，**必须返回强制 argv 或 fail-closed**（`SandboxUnavailableError`，code `SANDBOX_UNAVAILABLE`），静默无限制透传被禁止。`SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access'`；`SandboxEnforcement = 'full'|'partial'`；每个后端带 `denialSignatures`（EROFS/EACCES/EPERM 方言）与 `runnerFailureRules`（区分"命令没跑起来"与"沙箱拒绝了"）。`SandboxPolicy` 是 **per-call** 携带（`{mode, workspaceRoot, sessionId?}`），不是 provider 固定配置；`escalation.ts` 提供 `WIDER_MODES` 严格更宽阶梯（read-only→[workspace-write, danger-full-access]）、`validateEscalationArgs`（`sandbox_permissions` 与 `justification` 必须成对）、`approveEscalation`（**任何东西执行前**先做执行期严格更宽检查，非更宽请求绝不弹人；`allowed-once` 只盖章到这一次调用）。
- **`sandbox-local`**（`sandbox-local/src/index.ts`）：`PLATFORM_CHAINS = { linux: ['bwrap','landlock'], darwin: ['seatbelt'], win32: ['windows-acl'] }`，多候选时探测仲裁（bwrap 用 `--ro-bind / / ... -- true` 探测、landlock 用 `--probe`）；**Linux 优先 bubblewrap（mount namespace：`--ro-bind / /` + workspace-write 追加 `--tmpfs /tmp` + `--bind <ws> <ws>`），Landlock 兜底（allow-list：`--ro /`、`--rw /dev/null`，workspace-write 追加 `--rw /tmp`、`--rw <ws>`）**；macOS 用 sandbox-exec（SBPL deny-then-allow profile，root 列表来自共享 `writableRoots`）；Windows 用 ACL restricted token + 能力 SID（`workspaceWriteSid` 按 workspace 派生、ACE 常驻复用；每个 live session 获得随机私有临时目录 + 独立 temp SID，dispose 吊销，enforcement 诚实标 partial）。
- **`native/landlock-run/`**：C11 源码（`packages/entry/src/main.c`，约 300 行、静态 musl、**直接调 raw Landlock UAPI syscall 444/445/446，零第三方库**）：`landlock-run [--ro <path>]... [--rw <path>]... -- <argv>` 或 `--probe`；self-restrict-then-exec（`PR_SET_NO_NEW_PRIVS` 中和 setuid/setgid 提权 → 建 ruleset → `landlock_restrict_self` → `execvp`，**ruleset 跨 execve 继承**，命令及全部子孙被约束、调用方进程不受影响）；fail-closed：内核无 Landlock → exit 125 不 exec；老 ABI 部分执行可接受但 stderr 如实上报 `partial enforcement`。Node 侧 `packages/entry/src/index.ts`：`launcherPath()`（按平台包解析，**刻意无环境变量覆盖**）、`probe()` → `'full'|'partial'|'unusable'`、`LAUNCHER_FAILURE_EXIT = 125`。三包 npm 家族（entry/linux-x64/linux-arm64），"Landlock Run" CI 构建 + "Landlock Run Release" 发布。
- **`fs-sandbox`**（`fs/fs-sandbox/src/`）：`SandboxedFileSystem extends LocalFileSystem`——进程内文件写拦截（`FS_SANDBOX_DENIED`），`containment.ts` 的 `isPathUnder`（lexical 快路径 + dev+ino 祖先身份比对保守回退，防 Windows 8.3/大小写别名逃逸）；read 放行，write/edit 按 mode 拒绝；workspace-write 分支**此刻重新 resolve 规范化**（捕捉并发换掉的 symlink 祖先）并用 fresh target 委托写入（消除 check-here-write-there TOCTOU）。注意 fence 是 trusted 代码对模型控制路径的 containment，不是内核边界（内核级隔离是 `ctx.shell` 的事）。
- **关键澄清**：`fs-observation-policy`（`fs/fs-observation-policy/`）**不触发审批**——它是"读先于写/版本 CAS"守卫（`createIfAbsent` vs `replaceIfVersion`、`FS_NOT_OBSERVED`），只由 `fs/observed` 事件喂状态；审批只由沙箱升级重试（`FS_SANDBOX_DENIED` → 带 `sandbox_permissions`/`justification` 重试，经 `approveEscalation`）或 `tools/pre-execute` 的 `ask` 门触发。
- **local vs sandbox 切换点**：组合层决定谁注册 `ctx.shell`/`ctx.fs`（重复注册抛错）；工具层只读能力事实（`shell.sandboxMode`、`fs.sandboxMode`——undefined 则不广告升级字段、不 resolve 策略）；`bash-sandbox` 是 `bash-local` 的子类，仅替换 argv 并做 denial/runner-failure 分类（spawn 拒绝且能归因 runner → `SandboxUnavailableError`）。**shell 工具在沙箱内的执行方式**：`ctx.sandbox.confine(['bash','-c',command], policy)` → spawn 包装后的 argv；`danger-full-access` 直通不包装。bash-local 本身：默认超时 120s/上限 600s、stdout 64KB 超限转 spill 文件、进程组 SIGTERM→3s→SIGKILL。
- **guard 插件**：`guard/timeout-policy`（`tools/execute` around 包装，读 `ToolDefinition.timeoutMs`，`deadline(exec.signal, timeoutMs, TOOL_TIMEOUT)` 派生超时 signal 替换 exec.signal、finally 恢复上游 signal；超时结果结构化 `TOOL_TIMEOUT` isError；**协作式**——不竞速不弃跑工具 promise）、`guard/repeat-tool-reminder`（连续重复工具调用链检测，参数 canonical 化，阈值 [3,5,8]；`tools/post-execute` 上 observe-and-enrich 绝不 veto，`agent/pre-step` 见 user 消息重置）。
- **hooks**（`packages/hooks/`）：`hook-protocol/runner.ts` 的 `runHook`（经 `ctx.shell` 跑 hook 进程、stdin JSON payload、默认 600s 超时、解码结构化输出含 exit 2 deny/additionalContext）；`hooks-claude-code` 桥 7 个点（SessionStart/UserPromptSubmit/PreToolUse deny|ask/PostToolUse block|context/Stop block→`agent.steer` 强制续跑/SubagentStart|Stop，CC 方言 payload + `${CLAUDE_PLUGIN_ROOT}` 替换，每次调用写 `hook/invoked`+`hook/result` 审计对）；`hooks-codex` 桥 5 个点（matcher 一律正则、snake_case payload、只兑现 blocking 决策）。
- **defensive-patterns**（`docs/defensive-patterns.md`）：独立上报正交结果（`timedOut`/`signal`/`exitCode` 可同时成立）；dispose 必须 kill→await done→先关监听再杀（防孤儿与迟到完成）；**不把非受信输出交给环境或可预测路径**——spawn 环境剥离 `*KEY*/*SECRET*/*TOKEN*/*PASSWORD*`、临时/spill 文件用私有 0700 目录+随机名+`'wx'`/`0o600` 独占打开；contain callback 异常；unlink link-shaped 路径（lstatSync isSymbolicLink + unlinkSync，避免跟随链接）。

---

## 8. LLM 层（packages/llm）

### 8.1 Service Definition：`LlmRuntime`（`packages/llm/llm/src/index.ts`）

- 词汇表（`types.ts`/`message.ts`/`content.ts`）：`Message`、`ContentBlockMap`（text/tool-call/tool-result/image…）、`StreamChunk`（text-delta/reasoning-delta/tool-call-delta/block-start/block-end/usage/finish）、`LlmFailure`（可序列化错误事实）。**唯一词汇表，provider 必须翻译到它**。
- `abstract class LlmAdapter`：`providerInfo/providerRetryPolicy/listModels/resolveModel/stream(options): AsyncIterable<StreamChunk>`；`ctx.llm.registerAdapter(providers, adapter)` 返回 `AdapterRegistrationHandle`（disposer + `replace()` 原子路由替换，配置热更新靠它）。
- **`llm/stream` 是 waterfall**（`index.ts:64`）：监听者可 `next()` 直达 adapter 或短路（retry/replay/routing 的拦截位）。
- `PreparedLlmCall`（`prepareCall(config, signal)`）：config + retryPolicy + context + `stream()`——一次 adapter 调用 = 一次 provider 尝试（pi-ai 显式 `maxRetries: 0`），**所有可见重试归 agent 恢复层**。
- `BlockAssembler`（`assembler.ts`）：把 chunk 流组装成完整 assistant 消息（loop 用它），带 `replayState`/usage。
- `LlmError extends HarnessError`（code：`AUTH`/`RATE_LIMIT`/`NO_ADAPTER`/`TRANSPORT`/`TIMEOUT`/`ABORTED`…），`failure` 可序列化。
- `attribution.ts`：`attributionHeaders()`（每次 provider HTTP 请求必须带，标识 harness 流量）。

### 8.2 DeepSeek provider（`packages/llm/llm-deepseek/`）

- `DeepSeekAdapter`（`adapter.ts`）：直连 `fetch(baseURL + '/chat/completions')`，SSE 用 `eventsource-parser`（`sse.ts` 的 `parseSse`，`[DONE]` 终止、流提前关闭报 `STREAM_CLOSED`）；`reasoning` 默认配置（thinking disabled 时 `OFF_ONLY_REASONING_EFFORTS`）；每次请求带 `attributionHeaders()` + `x-deepseek-harness-user-id` + 可选 `x-deepseek-harness-session-id`/`x-deepseek-harness-compact`；流空闲看门狗（`streamIdleTimeoutMs` 默认 300s，`TIMEOUT`）；**一次 resolveModel 冻结 connection 快照 + credential**（端点与密钥不可能来自不同配置代）。
- `llm-pi-ai`：pi-ai 库封装的多 provider 孪生（settings 节提供 provider profiles 才激活路由）。
- `llm-retry`（`llm-retry/src/index.ts`）：**挂在 `agent/request-error` waterfall**（`core/agent-loop/src/agent.ts:353-371`）：检查 provider 的 `retryPolicy.retryableCodes`、每次调度先 `session.append('llm/retry', ...)`（持久化再等待）再 `cancellableDelay`（有界指数退避 + 对称抖动，`providerRetryAfterMs` 优先），返回 `{kind:'retry'}` 驱动 loop 重发；`llm/retry` + `llm/retry-started` 事件（durable、replayable）。
- `token-meter`（`token-meter/src/index.ts`）：`TokenMeter`（`ctx.tokenMeter`）——"provider usage 锚点 vs 启发式估价"双 baseline 重放计量（`estimateMessage/estimateHeader/ROLE_OVERHEAD`），从 session 事件折叠。
- **compaction**（`packages/compaction/`）：`CompactionEngine` 抽象（`ctx.compaction`）+ `compaction-basic`：两个触发点——`agent/pre-step`（上下文压力）与 `agent/request-error`（`CONTEXT_WINDOW_EXCEEDED`）；流程：`compaction/start`（锁）→ 工具结果修剪（`compaction-tool-result-pruner`，阈值 8192 字符）→ LLM 摘要调用（`purpose: 'compaction'`，`summarizer.ts` 的 `summarizeWithLlm`）→ `user/message` surface **replace** → `compaction/end`；成功返回 `{kind:'retry'}` 与 llm-retry 在同一 waterfall 组合。

---

## 9. 客户端 / Web UI / RPC（host/client 分离）

### 9.1 Typert：类型图生成 RPC（`packages/typert/`）

把 Host 侧 TypeScript 类型"编译"成跨进程 RPC 契约，四个子包：

- **protocol**（`protocol/src/types.ts`）：merge-extensible 类型图——`TypertLookup<Host, Wire>`（Host 对象与 wire 身份类型级关联，如 `Agent ↔ agentId`，注册处见 `core/agent/src/index.ts` 的 `typert.lookups.register('agent', ...)`）、`TypertContext`、`TypertRemoteMap`（生成器填充的远端方法签名表 → `ctx.remote.<ns>`，**无 Proxy，全是具体函数**）；`InvocationDescriptor`（service/namespace/method、参数 codec json|lookup、cancellation 参数）；`Remote()`/`RemoteScope()` 装饰器 + `TypertRemoteService` 基类。
- **generator**（`generator/src/`）：`ts.Program → 编译器无关 model → 产物`；`WorkspaceAnalyzer`（host/client 双 face 分析）、`FaceModelEmitter.emit()`（js+dts，host face 含 invocation 时附加 remote 贡献）、`tsdown-plugin.ts`（`typertPlugin()` rolldown 插件在构建期写 `lib/typert.host.*`）。
- **loader**（`loader/src/index.ts`）：监听 `internal/plugin` 增量扫描 → 解析 `exports["./typert"]` → import `TYPERT` manifest → `ctx.typert.register()`。
- **registry**（`registry/src/service.ts`）：`TypertRegistry`（`ctx.typert`）——local/remote 两张 endpoint 表（endpoint = `<ns>/<method>`）、LookupStore/ContextStore（registerHost/registerClient）。

### 9.2 API gateway（`packages/api/`）

`gateway/src/index.ts`：`TypertGatewayService`（`ctx.typertGateway`）构造时 `ctx.connection.rpc.intercept('/api', ...)` 挂到共享 `/api` 通道。`invoke()`：descriptor → 精确参数校验（`assertExactArguments`，缺/多字段即错）→ receiver 解析（direct→root ctx；context→provider.resolve）→ 逐参数 decode（strict codec `schema.parse` + `assertJsonValue` 拒绝循环/稀疏数组）→ lookup 参数还原 Host 对象 → `Reflect.apply` → 结果 decode；错误封闭在 `TypertGatewayError`（17 个稳定码）。**SRC 开发回退**：Host 以 tsx 源码启动时不跑编译器插件，Gateway 用 `Function.prototype.toString` 解析参数名构造弱 descriptor（`src-json` codec，无 schema 校验），Client 拒绝挂载 SRC 贡献。`remotes/`：Host BFF 身份策略（`agent-lookup.ts` 的 `createApiRemoteAgentResolver`——复用活 Agent、自动冷恢复、拒绝 subagent 所有权身份）与 `API_REMOTE_FORWARDED_EVENTS` 白名单（11 个事件逐字转发：`commands/change`、`credentials/updated`、`llm/adapters-updated`、`settings/document-updated` 等）。

### 9.3 host/client 通信（`packages/client/connection/` + `packages/host/`）

- **自定义四象限消息模型**（`host/apiproxy/src/api/rpc.ts`）：`RpcMessage = ClientRequest | ServerResponse | ServerRequest | ClientResponse`；上行 `POST /api/<method>`（信任栅栏：DNS-rebinding 防御 + `trustedHosts`；`PRIVILEGED_METHODS` 集合——settings/credentials/agentPreset 读写、host.pickDirectory、llm.discoverModels——**强制 loopback**），下行两条单向 WebSocket（`/api/events.mux`、`/api/events.host`，客户端上行消息是协议违规）。两层解析纪律：消息层 zod schema + 业务 payload 按 method 二次 parse。
- **浏览器半**：`web-api-client.ts` 的 `WebApiClient`（unary fetch POST + 两条只读 WS）；`ConnectionController`（generation 循环：严格握手 `host.describe` 单播可达 + 双流 onOpen 后才 connected，断线指数退避 500ms→2x→10s cap 重连）；`createWebConnectionRpc().call(channel, endpoint, payload, signal)` 的 rpcId echo 校验。
- **调用路径示例**：`ctx.remote.goals.create(agentId, {...})` → `POST /api/goals/create` → 信任检查 → Gateway 认领 → descriptor+lookup 解析 → Host GoalService → 结果 decode 回 `RpcResult`。下行事件（approval/question/session event）走 `/api/events.mux`、`/api/events.host` 两条 WebSocket 的 `server-request` 帧，客户端经 `POST /api/respond` 应答。
- **Host 侧**（`packages/host/`）：`webserver/` 的 `WebServer`（`ctx.webServer`）是纯路由注册载体（exact/prefix/upgrade 路由、单一 fallback seat、index taps、SSE 持有响应），不服务文件不打印 URL；`frontend-static/` 是 SPA dist 服务器（越界 403、miss 回退 index.html 200，`__DSH_BOOT__` 在这里经 `applyIndexTaps` 进页面）；`apiproxy/` 的 `ApiProxy`（~3700 行）是业务域全集（sessions/workspaces/skills/settings/credentials/llm/goals/approvals/questions/subagents/jobs/host/events），作为 `/api` 的 **fallback**（无 Remote descriptor 的 endpoint 走它），`api/` 子目录是零 node 依赖的浏览器安全契约层；另有 `directory-picker-*`（win32 对话框 worker）、`plugin-inventory`（loader 条目只读投影）。

### 9.4 Web UI（`apps/web` + `packages/client/` + `packages/bundle/web-app/`）

- `apps/web/src/main.ts`：找 `#root` → `new AppWebEntry(el).run()`；`vite.config.ts` **拒绝 standalone serve**（无 `__DSH_BOOT__` 不启动——不是独立应用，必须由 dsh host 注入引导）；shell 包（`dsh-client-web` 等）alias 指向 src，插件包绝不进 shell bundle。
- `packages/client/web/src/boot.tsx` 的 `AppWebEntry.run()` 启动链：① `parseBootManifest(window.__DSH_BOOT__)` → ② `ClientModuleSystem`（懒 CJS 模块表：bundle 执行只注册 factory，副作用在首次 import 才 materialize）+ `registerStatic` shell 自带模块（app-shell）+ `window.__DSH_MODULES__` 槽 → ③ 渲染加载页 + 预取 `immediately` 层 → ④ `new Context()` + 挂 vendored Loader（`loader.internal = modules`，浏览器禁止裸 import 回退）→ ⑤ 创建条目 `[modules, ...插件, app-shell]` → `loader.await()` → `assertEntriesActive()` 全量 fiber 扫描（非 ACTIVE 即 fail-loud 列失败清单）→ ⑥ `ctx.slots.renderSlot('root', {})`（ui-layout 在 root 槽注册 AppFrame）。
- **UI 模块体系**：每个 `ui-*` 是 dual-face 包（Node 半空 apply、浏览器半经 `exports["./client"]` + `dsh.client` 声明被发现，带 `inject` 依赖边与可选 `immediately` 首屏预取）；`SlotRegistry`/`renderSlot('root')` 组装，`packages/client/web-react/` 的 `bindSnapshotSelector` 把快照选择器绑成 React hook。
- **`window.__DSH_BOOT__` 注入**（`client/modules/src/index.ts:170`）：Host 半 `ClientModuleRegistry` 增量扫描 `dsh.client` 声明 → 组装 `WebBootGraph` → `injectBootManifest()` 注入 `<script>window.__DSH_BOOT__ = {...}</script>` → 服务 `/plugins/<id>/client.js?rev=`；`rebuilt(id)` 重哈希是 HMR 唯一入口。`client/hmr/`：Node 半 500ms stat 轮询 + `/plugins/events` SSE 广播，浏览器半收帧 → invalidate→prefetch→registry-first teardown→entry.refresh()，级联刷新靠 fiber `_refresh` epoch。
- `bundle/web-app/cordis.patch.yml`：web profile 完整组装（webserver 默认 127.0.0.1:3080、connection、api-gateway、api-remotes、client-runtime、modules、client-hmr、全部 ui-*），并把 base 的进程级 agent 平面（tool-bash/tool-fs/tool-skill/tool-goal/tool-subagent/workflow 等）全部 `disabled: true`，改为 agent-presets 每 session 挂载——**配置即组合**：profile = 空 `cordis.yml` + 有序 patch 层。
- **三套 RPC 并存**：浏览器↔Host 自定义四象限（HTTP POST + WS downlink）；SDK 为 NDJSON JSON-RPC 2.0 over stdio；ACP 为 ACP over JSON-RPC stdio。SDK 直接 `session/prompt` 进 agent 队列（无 UI），ACP 只给自动化客户端 committed text + 一次性权限。

### 9.5 SDK / ACP / CLI

- **`packages/sdk/`**：`protocol/src/transport.ts` 的 `JsonRpcLineTransport`（NDJSON；缺 handler→-32601）；`HarnessSdkRequestMap`（initialize/session/prompt/shutdown）+ `HarnessSdkNotificationMap`（session.event/session.status/subagent.started/finished）；`HarnessSdkJsonRpcServer`（订阅 session/agent/subagent 事件转通知，`session/prompt` 懒创建 agent）。
- **`packages/acp/`**：`acp` 插件用 `@agentclientprotocol/sdk` 的 `AgentSideConnection` over JSON-RPC stdio 实现 ACP；**只流已提交的 assistant 文本**（`agent_message_chunk`），reasoning/tools/plan 不上自动化线；`approval/request`→`requestPermission`。
- **`apps/cli/`**：`args.ts` 的 `parseDshArgs`——launcher 只解析 `--profile/--patch/--dump-config`，**第一个不认识的 token 起全部透传**；`web` 是 `--profile web` 硬别名；`bin.ts` 按 mode 动态 import 分发；`profile-boot.ts` 的 `runProfile` 完成 patch 栈组装 + `provideCmdline` + SIGTERM=0/SIGINT=130 + 用户 patch 文件 config-only HMR。

---

## 10. Python SDK 与 native

- **`python/sdk`**（dist `deepseek-harness-sdk` / 模块 `deepseek_harness`）：高层 turns API + 低层 JSON-RPC client；`HarnessClient`（spawn 捆绑 runtime、EOF→SIGTERM→SIGKILL 拆除梯、`NotificationSubscription`）；`DeepSeekHarnessConfig`（provider 默认 deepseek-official / deepseek-v4-flash）；`session_prompt`/`request`/`next_notification` 等。与 `packages/sdk/client` 设计孪生。
- **`python/sdk-runtime`**（dist `deepseek-harness-runtime-bin` / 模块 `deepseek_harness_runtime`）：捆绑的 runtime 二进制 + 默认 agent 配置（`cordis.yml`），`hatch_build.py`/`platforms.json` 定义平台分发；`scripts/build-exe-for-python-sdk.ts`、`scripts/build-python-release.py` 负责构建。
- **`native/landlock-run`**：见 §7.2——C11 静态 musl 二进制（self-restrict-then-exec），entry + linux-x64 + linux-arm64 三包 npm 家族，属于仓库根 pnpm workspace（开发/CI 用 workspace entry 包，发布经 "Landlock Run Release" workflow 组装平台包）。

---

## 11. 工程实践

### 11.1 TypeScript / 模块 / monorepo

- `strict: true` + `noImplicitAny`；任何残余 `any` 必须解释；`verify-export-jsdoc` 强制每个导出有 JSDoc（`@param`/`@returns`）。
- **ESM everywhere**（`"type": "module"`）；本地相对导入显式 `.ts` 后缀；`dsh` CLI 源码启动经 `node --import tsx/esm`。
- pnpm workspaces（node `^22.19 || >=24`）；每个 npm 包 `@deepseek-ai/dsh-<name>`；**双编译面**：`tsconfig.host.json` 与 `tsconfig.client.json` 两个隔离 aggregate（Context 声明合并冲突的解法），构建顺序 `build:lib:host → build:lib:client → build:web`，Typert 只在 Host tsdown 跑。
- 强类型跨边界：跨边界的 id 都 **branded**（`Branded<B>`，如 `SessionId`/`CallId`/`GoalId`）；封闭联合以 `assertNever` 结尾；merge-extensible 联合有文档化 default。

### 11.2 测试策略（`docs/testing.md`）

- **Unit**（`pnpm run test`）：vitest，测试与代码同包 `tests/`；每个 registry 有 HMR-safety 测试（dispose fiber 断言清理）。
- **Coverage 门禁**（`pnpm run test:coverage`）：**per-file 100%** on `packages/*/*/src`（CI 门禁；未覆盖行常是应删的死代码）。
- **Real-API e2e**（`test:e2e`）：带 key 测试（DEEPSEEK_API_KEY 等），无 key 自跳过（"我们是 DeepSeek——不要吝啬真实 API 测试"）。
- **Snapshot**（`test:snapshot`）：**keyless 可重放**——ACP 启动真实 automation-server 例子、重放记录会话、diff 规范化 JSON-RPC + 重持久化日志；`test:snapshot:record` 需要 key 重录。产品级行为变更必须带 snapshot。
- **Web 浏览器 snapshot**（`test:web`）：Chromium 对比 `apps/web/tests/snapshots/`；CI 强制 `DSH_SNAPSHOT=replay` 只读。
- 原则：**"验证世界，不验证自我报告"**（e2e 断言重新执行命令/重读文件）；**mock 只 mock 昂贵的/非确定的边界**（LLM adapter/network/clock），其余用真实实现；**测试真实入口路径**（built `lib/bin.js` 在 plain node 下跑，暴露 tsx 掩盖的问题）；source plane 与 artifact plane 严格分离。

### 11.3 大量 verify-* 门禁脚本（`scripts/`，145 个文件）

`pnpm run hygiene` = rescope-vendor:check + knip + publint + constraints + verify-dsh-package-licenses + verify-package-invariants + verify-built-package-invariants + verify-cordis-config + verify-node-next-types + verify-runtime-closure + verify-vendored-links。另有 `verify-export-jsdoc`、`verify-type-equiv`（docs 里的 type-equiv 代码块与源码逐字一致）、`verify-doc-budgets`、`verify-md-links/md-wrap/mermaid`、`verify-translation-pairing`（i18n 配对）、`verify-agent-note-*`、`run-gates.ts`（CI 门禁矩阵编排）、`gen-*` 系列生成器（gen-cordis-catalog、gen-config-catalog、gen-tool-catalog、gen-module-graph、gen-persistence-catalog、gen-scoped-events）。lefthook 提供 pre-commit/pre-push 本地检查点（翻译配对、归档 notes、staged lint、空白、vendor manifest 守卫）。

### 11.4 AGENTS.md 约定（面向 agent 的仓库宪法）

- 每个非平凡变更必须附带 **Agent Note**（`.agents/notes/`，按 implemented/archived 分类，归档即冻结）。
- **Runtime invariants assert owned relationships**：每个包必须有自己的 `./invariant`（`ctx.invariants.register` 挂到事件流上断言关系，`verify-package-invariants` 强制），不变量检查权威事件流/可变数据，不检查"服务/方法存在性"。
- "Plugins, not loop changes"：新行为上文档化扩展点；改 `agent-loop` 必须同步更新 `docs/architecture.md`。
- "Model-visible ⟺ logged"（§3.4）；"Registrations are effects"；"Misconfiguration fails loud"；"Explicit > implicit at package boundaries"（`resolve(request): Spec` 模板）。
- 双语文档纪律（docs 中英配对、`doc-sync` 门禁）、`dsh-prose-standard` 措辞标准。

---

## 12. dsh 最独特的设计决策（15 条，与主流 agent CLI 对比）

1. **没有特权内核**：连 agent loop 本身都是插件（`ctx.agentLoop` 由配置行挂载）。Claude Code 有不可替换的内核，dsh 的全部可替换——扩展 = 加一行 `cordis.patch.yml`。
2. **事件溯源会话**：会话不是 transcript 快照，而是 **append-only SessionEvent 日志**（`seq=log.length`）；模型历史由 `deriveMessages()` 派生，**永不单独存储**。Claude Code 的 transcript 是产物，dsh 的日志是唯一事实源。
3. **"模型可见 ⟺ logged" 作为运行时不变量**：`agent-loop/invariant` 在 `llm/stream` 上断言 `messages === deriveMessages()`（JSON 相等），checkpoint policy 保证"持久化先于可见"。主流 CLI 没有这种可执行的重构一致性证明。
4. **turn/step 双边界 + 双队列 inbox**：`next-turn`/`next-step` 分离普通追问与 steering/injected context；inbox 变更本身是日志事件（`agent/inbox/spliced`），可重放。Claude Code 用单一 message 队列。
5. **四道 agent 域 waterfall + 三道 tools 域 waterfall**：pre-step / request / request-error / llm-stream / pre-execute / execute / post-execute——全部是 Cordis waterfall（around-middleware，`next()` 委托）。策略、重试、压缩、超时都以监听者身份附着，**不修改 loop**。
6. **工具必须声明 canonical JSON output + 纯函数 UI 呈现**（`presentCall/presentResult`）：模型只见渲染文本，UI 卡片从日志 `meta` 重放重建。工具设计把"UI 渲染意图"作为一等公民。
7. **沙箱是 argv 包装 seam**：`SandboxProvider.confine(argv, policy)`，平台链 bwrap→landlock / seatbelt / windows-acl；**静默无限制透传被禁止**（fail-closed）。Claude Code 靠 OS 级 sandbox-exec；dsh 把它做成可插拔、可诚实降级（enforcement: full/partial）的服务。
8. **per-call 沙箱策略 + 升级审批流**：模式 `read-only|workspace-write|danger-full-access` 每次调用解析（两个消费者可同时处于不同模式）；被拒后唯一补救是带 `sandbox_permissions`+`justification` 的一次性升级重试（经 `ctx.approval`）。
9. **read-before-write 观测策略**：fs 读操作发 `fs/observed` 事件喂状态，写/edit 必须 CAS（`replaceIfVersion`/`createIfAbsent`），未经观察的编辑被拒（`FS_NOT_OBSERVED`）——把 Claude Code 的"read-then-edit 提醒"升级成强制机制。
10. **Code Mode：只暴露一个 run_code 工具 + 生成式 SDK 提示**（TS/Python），模型在程序内调工具——比"给模型几十个原生工具"更省 token、schema 更稳定。这是"工具呈现"可插拔的直接体现。
11. **子代理有六种 provider**（spawn/fork 同进程 + acp/claude-code/codex/dsh-sdk 子进程），`subagent_fork` 是真正的**会话 fork**（父日志平衡前缀 seed），`send_message` 续跑是持久 session 上的 Activation。Claude Code 的子代理不可续跑/不可 fork。
12. **goal 是同会话持久目标**（`goal/change` 事件源 + round driver 自动续跑 + 直接人类授权围栏），workflow 是 worker-thread + vm 的脚本编排（`pipeline` 无 barrier、`parallel` barrier、失败→null），Ralph 是 fresh-child 循环——三种"多 agent"模式并存且全是插件。
13. **plan mode 是 logged state**（`plan/mode` 事件 + fold），resume/fork/compaction 后状态自动恢复；`exit_plan_mode` 走审批渠道（Approve/Keep planning），反馈作为工具错误返回让模型修订。
14. **Typert：类型图生成 RPC**——Host 的 TS 类型（含 `@Remote` 装饰器）在构建期编译成跨进程契约，浏览器端得到**无 Proxy、全具体函数**的 `ctx.remote.<ns>`；事件转发靠 11 条白名单逐字转发。主流 CLI 的 UI 层没有这种类型级 RPC。
15. **双编译面（host/client）+ window.__DSH_BOOT__ 引导图**：每个 UI 模块是 dual-face 包；Web 应用不是独立 Vite 应用，必须由 host 注入引导清单（拒绝 standalone serve）。
16. **per-file 100% 覆盖率门禁 + keyless snapshot 回放 + "mock 只 mock 边界"**：测试哲学是"验证世界，不验证自我报告"，产品级变更必须带可重放的组装应用转录。
17. **版本演进极简主义**：`SESSION_FORMAT_VERSION = 0` 单一整数、写者决定 bump、per-event `ignorable` 处理词汇增长、**拒绝而非迁移**；SQLite `SCHEMA_VERSION` 同立场。未发布期零兼容承诺（AGENTS.md 明示"foundation over blast radius"）。
18. **面向 agent 的工程宪法**：AGENTS.md + Agent Notes + 每个包 runtime invariant + 大量 verify-* 门禁 + 双语文档生成/校验——仓库本身设计成"可被 agent 安全修改"（包括自举 demo：`pnpm run demo:cordis` 让 agent 修改自己的运行时）。

---

## 附：关键文件速查表

| 路径 | 一句话职责 |
|---|---|
| `vendor/cordis/src/{context,service,events,registry,fiber}.ts` | vendored Cordis 核心：context Proxy、Service 基类、五模式事件总线、插件注册、可逆 effect |
| `vendor/README.md` | vendoring 清单（上游 SHA）+ 18 条本地修改日志 |
| `packages/boot/app-boot/src/profile.ts` | profile/bundle 发现、补丁层组合（`composeEntries`）、模块双锚点解析 |
| `packages/bundle/base/cordis.patch.yml` | 默认产品完整插件清单（451 行补丁） |
| `apps/cli/src/profile-boot.ts` | `runProfile`：patch 栈组装 + 引导 + 信号处理 + 用户 patch HMR |
| `packages/core/agent/src/index.ts` | `AgentRegistry`（ctx.agents）、initiator scope、`Agent` 接口 |
| `packages/core/agent/src/inbox.ts` | `Inbox` 双队列（next-turn/next-step），日志可重放投影 |
| `packages/core/agent/src/runtime-types.ts` | `agent/*` 事件词汇表（含四道 waterfall 签名） |
| `packages/core/agent-loop/src/agent.ts` | `ReactLoopAgent`：turn/step 驱动、buildRequest、事件闭环 |
| `packages/core/agent-loop/src/tool-calls.ts` | 工具并行调度器（exclusive 屏障 / parallel 滚动池，模型序提交） |
| `packages/core/agent-loop/src/index.ts` | `AgentLoop`：工厂注册、create/resume 事务、配置 agent |
| `packages/core/agent-loop/src/invariant.ts` | "model-visible ⟺ logged" 运行时不变量（llm/stream 断言） |
| `packages/core/session/src/index.ts` | `Session`（append-only 日志 + deriveMessages）+ `SessionStore` |
| `packages/core/session/src/types.ts` | `SessionEventMap` 词汇表、`SESSION_FORMAT_VERSION = 0` |
| `packages/core/session/src/surface.ts` | surface 机制（append/replace）、`deriveEventMessage` |
| `packages/core/session/src/chunk-rows.ts` | assistant/chunk delta run 打包（~56× 压缩，无损） |
| `packages/core/session/src/repair.ts` | `interruptedTurnClosers`：崩溃合成 tool/result + turn/end |
| `packages/session/session-persistence/src/coordinator.ts` | 持久化编排（write-behind、串行链、崩溃修复、版本拒绝） |
| `packages/session/session-persistence-jsonl/src/{format,zstd}.ts` | JSONL + zstd 帧容器磁盘布局 |
| `packages/session/session-persistence-sqlite/src/schema.ts` | SQLite schema（SCHEMA_VERSION=15，WAL，拒绝迁移） |
| `packages/session/session-projection/src/index.ts` | `SessionProjectionRegistry`：日志→读模型的 whole-value 投影 |
| `packages/session-query/session-query-sqlite/src/` | FTS5 派生全文索引 + SQL 查询构建 |
| `packages/core/tools/src/index.ts` | `ToolRuntime`：scoped 注册表、restrict/guard、pre/execute/post 管线、approval 门、Code Mode |
| `packages/core/system-prompt/src/index.ts` | `SystemPrompt`：section/context/variable/tool 注册与组装 |
| `packages/core/scope/src/index.ts` | `createScope/scopeOf/scopeTarget`：per-agent 作用域原语 |
| `packages/llm/llm/src/index.ts` | `LlmRuntime` + `LlmAdapter` 抽象 + `llm/stream` waterfall |
| `packages/llm/llm-deepseek/src/adapter.ts` | `DeepSeekAdapter`：直连 chat/completions + SSE + 看门狗 |
| `packages/llm/llm-retry/src/index.ts` | agent/request-error 上的有界退避重试（持久化再等待） |
| `packages/llm/token-meter/src/index.ts` | `TokenMeter`：usage 锚点 + 启发式估价双 baseline |
| `packages/compaction/compaction-basic/src/index.ts` | 压缩引擎：pre-step 压力 + request-error 触发，surface replace |
| `packages/fs/fs/src/index.ts` | `FileSystem` 抽象 + fs/write-intent、fs/edit-intent、fs/observed 事件 |
| `packages/fs/fs-sandbox/src/index.ts` | `SandboxedFileSystem`：进程内写拦截（FS_SANDBOX_DENIED） |
| `packages/fs/fs-observation-policy/src/index.ts` | read-before-write CAS 守卫（事件门，不触发审批） |
| `packages/fs/tool-fs/src/read.ts` | read 工具实现（流式、窗口、观测、presentCall/Result） |
| `packages/shell/shell/src/index.ts` | `ShellExecutor` 抽象（resolve(request): Spec 模板） |
| `packages/shell/tool-bash/src/index.ts` | bash 工具（后台 job、沙箱升级、退出码标记） |
| `packages/subprocess/subprocess/src/index.ts` | `SubprocessRuntime` 抽象 + 环境清洗 |
| `packages/sandbox/sandbox/src/index.ts` | `SandboxProvider` 抽象（confine→ConfinedArgv，fail-closed） |
| `packages/sandbox/sandbox-local/src/index.ts` | 平台沙箱链（bwrap/landlock/seatbelt/windows-acl）+ 探测 |
| `native/landlock-run/` | C11 Landlock self-restrict-then-exec 启动器（三包 npm 家族） |
| `packages/interaction/user-approval/src/index.ts` | `ApprovalService`：allowed-once 唯一授权、审计事件、策略事件 |
| `packages/subagent/subagent/src/index.ts` | `SubagentRuntime`：多 provider 注册表 + 生命周期事件 |
| `packages/subagent/subagent/src/continuation.ts` | continuable 子代理 Activation（followup/interrupt/coldResume） |
| `packages/subagent/subagent-fork-in-process/src/index.ts` | 会话 fork provider（父日志平衡前缀 seed） |
| `packages/workflow/workflow-worker-thread/src/` | worker-thread + vm 脚本编排（agent/parallel/pipeline/phase） |
| `packages/goal/goal/src/index.ts` | `GoalService`（goal/change 事件源、revision CAS） |
| `packages/goal/goal-round-driver/src/index.ts` | 目标轮次驱动（reservation、pre-step 围栏、round-limit） |
| `packages/skill/skill/src/index.ts` | `SkillProvider` + `ctx.skills` registry |
| `packages/skill/tool-skill/src/index.ts` | skill 加载工具 + `<available_skills>` 会话目录 |
| `packages/plan/plan-mode/src/index.ts` | plan mode（plan/mode 日志事件 + exit_plan_mode + 审批） |
| `packages/todo/tool-todo/src/index.ts` | todo_write（整表替换、turn 边界清空） |
| `packages/preset/agent-presets/src/index.ts` | 每会话 preset 组合（standing mount + isolate realm） |
| `packages/typert/{protocol,generator,loader,registry}/` | 类型图生成 RPC 四件套 |
| `packages/api/gateway/src/index.ts` | `TypertGatewayService`（/api RPC 网关、17 错误码） |
| `packages/api/remotes/src/remote-events.ts` | 11 条 Host 事件转发白名单 |
| `packages/client/connection/src/` | 浏览器↔Host 四象限 RPC + 双 WS 下行 |
| `packages/client/modules/src/client/system.ts` | `ClientModuleSystem`：懒 CJS 模块表（__DSH_MODULES__） |
| `packages/client/modules/src/index.ts` | `window.__DSH_BOOT__` 引导图注入 |
| `apps/web/src/main.ts` | Web 入口（new AppWebEntry(el).run()） |
| `packages/sdk/{protocol,server,client}/` | NDJSON JSON-RPC 2.0 SDK（stdio） |
| `packages/acp/acp/src/index.ts` | ACP 自动化服务器（只流已提交 assistant 文本） |
| `packages/host/webserver|frontend-static|apiproxy/` | 路由/SPA 静态服务/业务域 API 代理 |
| `python/sdk/src/deepseek_harness/client.py` | Python SDK `HarnessClient`（JSON-RPC over stdio） |
| `python/sdk-runtime/` | 捆绑 runtime 二进制 + cordis.yml 默认配置 |
| `scripts/`（145 文件） | verify-*/gen-* 门禁与生成器、run-gates CI 矩阵 |
| `docs/architecture.md` | 架构总纲（turn flow、事件域、扩展点表） |
| `docs/cordis-primer.md` | Cordis 五想法 + 派发模式 + waterfall 语义 |
| `docs/agent-lifecycle.md` | turn/step 时序图（精确到事件签名） |
| `docs/tool-execution-pipeline.md` | 工具管线流程图 |
| `AGENTS.md` / `packages/AGENTS.md` | 面向 agent 的仓库宪法与包级规范 |
