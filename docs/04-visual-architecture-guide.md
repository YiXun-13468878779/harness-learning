# DeepSeek Harness 与 Claude Code 可视化深度架构指南

> 这篇文档面向了解大模型、但不一定熟悉编译器、事件溯源或插件运行时的 AI 从业者。它不替代原有源码分析，而是先用图回答“系统由什么组成、一次请求怎样流动、哪里控制安全、状态如何恢复、能力怎样扩展”，再把每个结论链接回工程级文档。

## 0. 先建立一套读架构的方法

读 Agent 系统时，不要一上来追每个类和函数。先沿着下面五个问题建立主干：

```mermaid
flowchart LR
    Q1["1. 入口在哪里？"] --> Q2["2. 谁驱动模型循环？"]
    Q2 --> Q3["3. 工具如何执行并被授权？"]
    Q3 --> Q4["4. 对话状态存在哪里？"]
    Q4 --> Q5["5. 新能力从哪里接入？"]
```

两套系统都能归约成同一个最小闭环：

```mermaid
flowchart LR
    U["用户输入"] --> C["组装上下文"]
    C --> M["调用模型"]
    M --> D{"模型要调用工具吗？"}
    D -- "是" --> P["权限判断"]
    P --> T["执行工具"]
    T --> R["结果写回会话"]
    R --> C
    D -- "否" --> A["输出最终回答"]
```

真正的差异不在“有没有循环”，而在循环周围：DeepSeek Harness 把边界做成可替换的插件与能力接缝；Claude Code 把边界收进一个高度优化的产品控制流。

---

## 1. DeepSeek Harness：从插件树到一次完整执行

### 1.1 初学者总览：它不是一个固定 Agent，而是一套 Agent 装配系统

可以把 dsh 理解为五层。越靠上越接近用户，越靠下越接近基础设施；每层的大部分部件都能由 Cordis 插件替换。

```mermaid
flowchart TB
    subgraph L1["交互与接入层"]
        WEB["Web UI"]
        CLI["CLI / Headless"]
        SDK["TypeScript / Python SDK"]
        ACP["ACP 自动化客户端"]
    end
    subgraph L2["Agent 编排层"]
        LOOP["Agent Loop<br/>turn / step 状态机"]
        PROMPT["System Prompt 组装"]
        GOAL["Goal / Plan / Workflow"]
        SUB["Subagent Runtime"]
    end
    subgraph L3["能力层"]
        TOOLS["Tool Runtime"]
        LLM["LLM Runtime"]
        FS["FileSystem / Shell"]
        APPROVAL["Approval"]
        SANDBOX["Sandbox"]
        COMPACT["Compaction"]
    end
    subgraph L4["状态与协议层"]
        SESSION["Append-only Session Log"]
        PROJ["Projection / Query / Telemetry"]
        RPC["Typert RPC / JSON-RPC"]
    end
    subgraph L5["运行基础"]
        CORDIS["Cordis Context<br/>Service + Event + Effect"]
        STORE["JSONL / SQLite"]
        NATIVE["Landlock 等平台沙箱"]
    end
    WEB --> LOOP
    CLI --> LOOP
    SDK --> LOOP
    ACP --> LOOP
    LOOP --> PROMPT
    LOOP --> GOAL
    LOOP --> SUB
    LOOP --> TOOLS
    LOOP --> LLM
    TOOLS --> FS
    TOOLS --> APPROVAL
    FS --> SANDBOX
    LOOP --> COMPACT
    LOOP --> SESSION
    SESSION --> PROJ
    WEB --> RPC
    SDK --> RPC
    CORDIS --> LOOP
    CORDIS --> TOOLS
    CORDIS --> LLM
    CORDIS --> SESSION
    SESSION --> STORE
    SANDBOX --> NATIVE
```

这张图最重要的不是层数，而是箭头背后的控制权：`Agent Loop`、`Tool Runtime`、`LLM Runtime`、`Session Store` 都是挂到 `Cordis Context` 上的服务，不是不可替换的硬编码单例。

源码入口可对照 [02 DeepSeek Harness 架构解读](02-dsh-architecture.md) 的第 1、2、3、5、8、9 节。

### 1.2 启动阶段：配置不是参数集合，而是产品装配图

普通应用的配置常用于改端口或开关；dsh 的配置决定“系统里实际安装哪些部件”。

```mermaid
flowchart TB
    NAME["选择 profile<br/>web / headless / 自定义"] --> DISCOVER["读取 profile.package.json"]
    DISCOVER --> BUNDLES["按顺序加载 bundles"]
    BUNDLES --> PPATCH["叠加 profile cordis.patch.yml"]
    PPATCH --> MPATCH["叠加机器级 patch"]
    MPATCH --> CPATCH["叠加命令行 --patch"]
    CPATCH --> TREE["得到最终 Cordis 插件树"]
    TREE --> RESOLVE["按服务依赖解析加载顺序"]
    RESOLVE --> EFFECT["注册 Service / Event / Tool / Effect"]
    EFFECT --> READY["Web、CLI 或 SDK 开始接收请求"]
    OVERRIDE["同 row id 的上层配置"] -.->|"整行替换"| TREE
```

因此，在 dsh 里“换模型”“换沙箱”“换持久化后端”首先是一个装配问题，而不一定需要修改 Agent 循环。`profile → bundle → patch` 是理解整个系统的第一把钥匙。

### 1.3 运行阶段：一次用户请求如何穿过 turn 与 step

`turn` 是用户感知的一轮任务，可能包含多次模型请求；每一次“模型请求 + 工具执行”叫一个 `step`。

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户或客户端
    participant I as Agent Inbox
    participant A as ReactLoopAgent
    participant S as Session Log
    participant L as LLM Runtime
    participant T as Tool Runtime
    U->>I: followup / steer / inject
    I->>S: 记录 inbox 变化
    A->>I: claim 下一批输入
    A->>S: append turn/start
    loop 一个 turn 中的每个 step
        A->>A: assemble prompt + tool schemas
        A->>S: append step/start 与 user/message
        A->>S: deriveMessages()
        A->>L: llm/stream（消息必须可由日志重建）
        L-->>A: assistant chunks / message / tool calls
        A->>S: append assistant/* 与 tool/call
        A->>T: executeToolCalls()
        T-->>A: canonical JSON result
        A->>S: append tool/result 与 step/end
    end
    A->>A: agent/turn-stopping 检查欠账
    A->>S: append turn/end
    A-->>U: 已提交的回答与状态
```

这里有三个容易混淆的点：

1. `followup` 进入下一轮，`steer` 尽快在下一个 step 边界进入，`inject` 只排队但不主动唤醒。
2. 模型看到的历史不是任意内存数组，而是 `session.deriveMessages()` 从日志投影出的结果。
3. 工具结果先成为会话事件，下一步再从会话投影回模型上下文。

### 1.4 控制面：插件怎样在不改循环的情况下改变行为

dsh 把关键节点做成 waterfall。waterfall 不是“广播后不管”，而是 around-middleware：监听者调用 `next()` 才把控制权交给下一层，也可以短路或改写结果。

```mermaid
flowchart LR
    INPUT["进入一个 step"] --> PRE["agent/pre-step"]
    PRE --> REQUEST["agent/request"]
    REQUEST --> STREAM["llm/stream"]
    STREAM --> TCALL["模型产生 tool call"]
    TCALL --> TPRE["tools/pre-execute"]
    TPRE --> TEXE["tools/execute"]
    TEXE --> TPOST["tools/post-execute"]
    TPOST --> NEXT["结果入日志，进入下一 step"]
    POLICY["策略插件"] -.->|"允许 / 拒绝 / 改写"| PRE
    RETRY["重试或模型路由插件"] -.->|"包裹请求"| REQUEST
    OBS["遥测或流拦截插件"] -.->|"观察 / 包裹"| STREAM
    SECURITY["权限、超时、重试插件"] -.->|"包裹工具"| TPRE
    SECURITY -.-> TEXE
    SECURITY -.-> TPOST
```

这解释了“一切皆插件”的实际价值：扩展行为不必不断给 `while` 循环添加 `if/else`，而是挂到有明确语义的控制点上。

### 1.5 数据面：为什么说 Session Log 是唯一事实源

```mermaid
flowchart TB
    EVENTS["turn / step / user / assistant / tool 等事件"] --> APPEND["Session.append()"]
    APPEND --> LOG["不可变、连续 seq 的事件日志"]
    LOG --> SURFACE["Surface 折叠<br/>append / replace"]
    SURFACE --> MODEL["deriveMessages()<br/>模型可见历史"]
    LOG --> TRANSCRIPT["人类可读 transcript"]
    LOG --> RESUME["resume / fork / crash repair"]
    LOG --> PROJECTION["todo / goal / title / stats 等投影"]
    LOG --> TELEMETRY["telemetry / query index"]
    INVARIANT["运行时不变量"] -.->|"断言 messages 等于日志投影"| MODEL
    CHECKPOINT["checkpoint policy"] -.->|"持久化成功后才允许外部可见"| APPEND
```

对初学者来说，可以把它类比成银行流水：余额、账单、统计图都能从流水计算，不能让某个页面偷偷维护另一份“真正余额”。压缩也不会删除历史，而是用带来源关系的 `replace` surface 事件改变模型当前看到的窗口。

### 1.6 工具与安全：授权、执行和展示为什么分开

```mermaid
flowchart TB
    CALL["模型提交 tool call"] --> SCHEMA["参数解析与 schema 校验"]
    SCHEMA --> PRE["tools/pre-execute"]
    PRE --> DECISION{"allow / deny / ask"}
    DECISION -- "deny" --> DENIED["结构化拒绝结果"]
    DECISION -- "ask" --> APPROVAL["ApprovalService<br/>仅本次授权"]
    APPROVAL -->|"拒绝"| DENIED
    APPROVAL -->|"allowed-once"| GUARD["scope guard / restrict"]
    DECISION -- "allow" --> GUARD
    GUARD --> EXEC["tools/execute<br/>超时 / 重试 / 指标可包裹"]
    EXEC --> BODY["工具实现"]
    BODY --> SANDBOX["FileSystem / Shell / Sandbox seam"]
    SANDBOX --> OUTPUT["canonical JSON output"]
    OUTPUT --> POST["tools/post-execute"]
    POST --> RENDER["纯函数 render / presentation"]
    RENDER --> LOG["tool/result 写入 Session Log"]
```

拆分后的收益：安全策略不需要知道 UI 怎样画，UI 重放不需要重新执行工具，工具实现也不需要直接 import React。对于文件和命令执行，沙箱是独立能力边界；如果安全后端不可用，设计目标是显式失败或声明降级，而不是悄悄裸跑。

### 1.7 子代理：同一接口背后可以是完全不同的执行引擎

```mermaid
flowchart LR
    PARENT["父 Agent"] --> API["SubagentRuntime<br/>统一生命周期 API"]
    API --> INPROC["spawn in-process<br/>新上下文"]
    API --> FORK["fork in-process<br/>继承已完成历史"]
    API --> ACP["ACP<br/>进程外"]
    API --> CODEX["Codex app-server"]
    API --> CLAUDE["Claude Agent SDK"]
    API --> DSHSDK["dsh SDK 子进程"]
    API --> DESC["durable descriptor"]
    DESC --> COLD["followup / interrupt / cold resume"]
    INPROC --> REPORT["统一报告回父 Agent"]
    FORK --> REPORT
    ACP --> REPORT
    CODEX --> REPORT
    CLAUDE --> REPORT
    DSHSDK --> REPORT
```

这不是简单的“开几个线程”：`SubagentRuntime` 把“谁来执行”与“父 Agent 怎样管理生命周期”分开，因此同一编排逻辑可以切换本地子 Agent、Codex 或 Claude Code。

### 1.8 dsh 的架构取舍

| 得到什么 | 付出什么 |
|---|---|
| 任意核心能力可替换，适合作为平台或嵌入式运行时 | 插件、scope、事件模式和配置装配的学习成本更高 |
| 会话可审计、可重放，崩溃恢复边界清晰 | 所有模型可见状态都要遵守日志与不变量纪律 |
| 安全、工具、UI 解耦，便于多端复用 | 跨插件追踪一次调用比读单体函数更费力 |
| 多 provider 子代理能接入异构 Agent | provider 生命周期、权限和恢复语义需要统一约束 |

---

## 2. Claude Code：从单循环到产品级控制系统

### 2.1 初学者总览：围绕 query() 打磨出的终端产品

Claude Code 的中心不是插件树，而是 `query()` 这条 async generator 控制流。交互 TUI 与 headless SDK 都消费它产生的事件。

```mermaid
flowchart TB
    subgraph UX["用户与接入层"]
        TUI["Terminal TUI<br/>App / REPL / Ink"]
        HEADLESS["Headless / SDK<br/>QueryEngine"]
        IDE["IDE / Remote / Server Bridge"]
    end
    subgraph CORE["核心控制流"]
        QUERY["query()<br/>async generator"]
        CONTEXT["System + User Context"]
        MODEL["Anthropic Streaming API"]
        ORCH["Tool Orchestration"]
        STOP["Stop / Fallback / Recovery"]
    end
    subgraph CAP["产品能力"]
        TOOLS["Built-in Tools"]
        PERM["Permission Engine"]
        HOOKS["25+ Hooks"]
        MCP["MCP / LSP / Skills / Plugins"]
        AGENT["AgentTool / Tasks / Swarm"]
        COMPACT["Context Compression"]
    end
    subgraph STATE["状态与基础设施"]
        TRANSCRIPT["JSONL Transcript / Sidechain"]
        APPSTATE["Global State + React Store"]
        CACHE["Prompt Cache / Cost / Token"]
        TERM["Raw Terminal / Yoga / Ink fork"]
    end
    TUI --> QUERY
    HEADLESS --> QUERY
    IDE --> QUERY
    QUERY --> CONTEXT
    QUERY --> MODEL
    QUERY --> ORCH
    QUERY --> STOP
    ORCH --> TOOLS
    ORCH --> PERM
    PERM --> HOOKS
    TOOLS --> MCP
    TOOLS --> AGENT
    QUERY --> COMPACT
    QUERY --> TRANSCRIPT
    TUI --> APPSTATE
    QUERY --> CACHE
    TUI --> TERM
```

它也有模块化扩展，但目标不是让 `query()` 本身被替换，而是在固定产品骨架上通过 hooks、MCP、skills、agents 和声明式插件增加能力。

源码入口可对照 [03 Claude Code 源码解读](03-claude-code.md) 的第 1、2、3、5、6、7、8、9、10 节。

### 2.2 启动阶段：为什么“进入 REPL 前”就有大量架构工作

```mermaid
flowchart TB
    BIN["cli.tsx 进程入口"] --> FAST{"fast path?"}
    FAST -- "--version 等" --> EXIT["直接输出并退出"]
    FAST -- "普通启动" --> PARALLEL["模块加载期并行预取<br/>MDM / Keychain"]
    PARALLEL --> INIT["init()<br/>安全环境变量 / 代理 / API 预连接"]
    INIT --> TRUST{"工作目录已信任？"}
    TRUST -- "否" --> DIALOG["onboarding / trust / login"]
    DIALOG --> TRUST
    TRUST -- "是" --> SETUP["setup()<br/>UDS / worktree / hook 快照 / session memory"]
    SETUP --> MODE{"运行模式"}
    MODE -- "-p" --> QE["QueryEngine / headless"]
    MODE -- "interactive" --> REPL["App + REPL + Ink"]
    QE --> QUERY["query()"]
    REPL --> QUERY
```

这条启动链体现的是产品安全与性能：不可信目录不能提前触发 LSP 等可能执行代码的能力；昂贵但安全的准备工作尽量并行或推迟，以缩短用户看到首屏的时间。

### 2.3 运行阶段：query() 如何同时服务 TUI 与 SDK

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant C as REPL 或 QueryEngine
    participant Q as query()
    participant M as Anthropic API
    participant O as Tool Orchestration
    participant P as Permission + Hooks
    participant J as JSONL Transcript
    U->>C: 输入任务
    C->>J: 记录用户消息
    C->>Q: system prompt + messages + tools
    loop queryLoop while(true)
        Q->>Q: 预算检查与多级压缩
        Q->>M: callModel（流式）
        M-->>Q: text / thinking / tool_use / usage
        Q-->>C: yield 流事件用于 UI 或 SDK
        Q->>J: 追加 assistant 消息
        alt 有 tool_use
            Q->>O: runTools()
            O->>P: 权限判断与 hooks
            P-->>O: allow / ask / deny
            O-->>Q: tool_result 消息
            Q->>J: 追加 tool_result
            Q->>Q: 拼接 messages 后进入下一轮
        else 没有 tool_use
            Q->>P: Stop hooks
            Q-->>C: Terminal reason / final result
        end
    end
```

与 dsh 最大的直观差别是：Claude Code 直接重建 `state.messages = [...旧消息, ...assistantMessages, ...toolResults]`；没有单独的事件溯源 surface 再投影一遍。它用更直接的消息控制流换取实现紧凑和 prompt cache 前缀可控。

### 2.4 工具执行：工具在模型看到之前就开始受策略影响

```mermaid
flowchart TB
    BASE["内置工具 + MCP 工具"] --> FILTER["按 deny 规则与 isEnabled 过滤"]
    FILTER --> SORT["排序、去重、稳定 schema 前缀"]
    SORT --> MODEL["把可见工具发给模型"]
    MODEL --> CALL["模型返回 tool_use"]
    CALL --> PARTITION["只读并发安全批 / 非只读串行批"]
    PARTITION --> VALIDATE["Zod / JSON Schema / validateInput"]
    VALIDATE --> PERM["checkPermissions + 规则与安全检查"]
    PERM --> HOOK["PreToolUse hook"]
    HOOK --> ASK{"需要用户交互？"}
    ASK -- "是" --> RACE["本地 / hook / 分类器 / 远程端<br/>多路裁决"]
    ASK -- "否" --> EXEC["tool.call()"]
    RACE -->|"允许"| EXEC
    RACE -->|"拒绝"| ERROR["is_error tool_result"]
    EXEC --> POST["PostToolUse hook"]
    POST --> RESULT["tool_result 拼回 messages"]
```

注意两层防线：第一层在工具池组装时就让模型看不到被禁止的工具；第二层在实际调用时根据参数内容、模式和安全规则重新判断。

### 2.5 权限决策：不是一个 yes/no 弹窗，而是一套优先级系统

下面是便于理解的概念顺序；具体分支与模式见 [03 文档第 6 节](03-claude-code.md)。

```mermaid
flowchart TB
    REQ["工具 + 具体参数"] --> DENY{"命中 deny 规则？"}
    DENY -- "是" --> NO["拒绝"]
    DENY -- "否" --> SELF["工具自身 checkPermissions"]
    SELF --> REDLINE{"命中不可绕过的安全红线？"}
    REDLINE -- "是" --> ASK["必须询问"]
    REDLINE -- "否" --> MODE{"当前权限模式"}
    MODE -- "规则允许" --> YES["允许"]
    MODE -- "需要确认" --> AUTO{"auto 分类器或自动检查可裁决？"}
    MODE -- "规则拒绝" --> NO
    AUTO -- "允许" --> YES
    AUTO -- "拒绝或不确定" --> ASK
    ASK --> CHANNELS["本地键盘 / hook / CCR / 频道"]
    CHANNELS --> FINAL{"最先有效的裁决"}
    FINAL -- "allow" --> YES
    FINAL -- "deny" --> NO
```

产品层面的好处是普通用户只感知到“该问时问、能自动时自动”，代价是权限语义分散在规则、工具自检、模式、hooks、分类器和交互层之间。

### 2.6 上下文管理：压缩不是一次总结，而是逐级减压

```mermaid
flowchart LR
    FULL["当前消息上下文"] --> BUDGET["工具结果预算"]
    BUDGET --> SNIP["snip<br/>裁掉历史中段"]
    SNIP --> MICRO["microcompact<br/>去旧文件内容与附件"]
    MICRO --> COLLAPSE["contextCollapse<br/>保留细粒度状态"]
    COLLAPSE --> AUTO{"仍超过阈值？"}
    AUTO -- "是" --> SUMMARY["autocompact<br/>模型生成总结 + 重建附件"]
    AUTO -- "否" --> REQUEST["发送模型请求"]
    SUMMARY --> REQUEST
    REQUEST --> APIERR{"API 返回 prompt too long？"}
    APIERR -- "是" --> REACTIVE["reactive compact<br/>有限次恢复"]
    REACTIVE --> REQUEST
    APIERR -- "否" --> OUTPUT["继续流式处理"]
    CACHE["prompt cache / cache_edits"] -.->|"降低重发稳定前缀的成本"| MICRO
    CACHE -.-> REQUEST
```

这个设计与 Anthropic prompt cache 深度耦合：工具排序、稳定 system prompt 前缀和 cache breakpoint 都是架构的一部分，而不只是费用优化小技巧。

### 2.7 持久化与恢复：主线程、子代理和产品状态怎样拼回去

```mermaid
flowchart TB
    MAIN["主 query 消息"] --> JSONL["主 session JSONL"]
    CHILD["AgentTool 子代理消息"] --> SIDE["sidechain transcript"]
    INPUT["终端输入历史"] --> HISTORY["history.jsonl<br/>大粘贴只存引用"]
    METRIC["成本 / token / 时长 / 文件变化"] --> CONFIG["项目与全局配置状态"]
    JSONL --> RESUME["/resume 或 --continue"]
    SIDE --> RESUME
    CONFIG --> RESUME
    RESUME --> TREE["重建消息树、agent 定义、文件快照与成本"]
    TREE --> QUERY["重新进入 query()"]
    JSONL --> TUI["REPL 消息列表"]
    HISTORY --> TUI
```

Claude Code 的持久化更像“产品运行记录”：消息转录、sidechain、输入历史、成本状态各有存储，再由 resume 流程组合恢复。

### 2.8 子代理与团队：复用同一循环，而不是换一个运行时

```mermaid
flowchart TB
    PARENT["主 query()"] --> AGENTTOOL["AgentTool"]
    AGENTTOOL --> CTX["createSubagentContext<br/>agentId / 工具 / 权限 / 记忆隔离"]
    CTX --> CHILDQ["再次调用同一个 query()"]
    CHILDQ --> RESULT["前台结果或后台 Task 状态"]
    RESULT --> PARENT
    PARENT --> TASKS["Task Registry"]
    TASKS --> LOCAL["local_agent / local_bash"]
    TASKS --> REMOTE["remote_agent"]
    TASKS --> TEAM["in-process 或 tmux teammate"]
    TEAM --> MAIL["teammateMailbox / UDS / Bridge"]
    MAIL --> LEADER["Leader / Coordinator"]
    LEADER --> PERM["权限请求转发与统一编排"]
```

复用 `query()` 的优势是子代理天然继承压缩、hooks、fallback、流式处理和工具编排；限制是所有子代理仍然围绕 Claude Code 的同一产品内核运行，不像 dsh provider 那样天然抽象成异构引擎。

### 2.9 扩展边界：什么能扩展，什么仍由产品内核掌握

```mermaid
flowchart LR
    EXT["外部扩展"] --> PLUGIN["声明式 Plugin"]
    EXT --> MCP["MCP"]
    EXT --> SKILL["Skills / Commands / Agents"]
    EXT --> HOOKS["Hooks"]
    PLUGIN --> HOOKS
    PLUGIN --> MCP
    PLUGIN --> SKILL
    HOOKS --> EVENTS["PreToolUse / Stop / Compact / Session 等节点"]
    MCP --> TOOLPOOL["进入 Tool Pool"]
    SKILL --> PROMPT["展开为 prompt、命令或子代理"]
    KERNEL["产品内核掌握"] --> QUERY["query() 控制流"]
    KERNEL --> CACHE["Anthropic API 与 prompt cache"]
    KERNEL --> TUI["权限 UI 与终端体验"]
    KERNEL --> STATE["会话与全局状态"]
```

Claude Code 的插件没有任意 JavaScript `main` 去替换内核服务；它通过声明式资源与 hooks 在产品预留的节点上工作。这降低了扩展带来的不可控性，也明确了“可扩展”不等于“核心可替换”。

### 2.10 Claude Code 的架构取舍

| 得到什么 | 付出什么 |
|---|---|
| 单一 `query()` 让交互、headless、子代理共享成熟能力 | 核心循环与 Anthropic 产品假设更难替换 |
| prompt cache、压缩、启动和 TUI 被统一优化 | 优化点跨工具排序、上下文、API 与 UI，耦合较深 |
| 权限体验覆盖规则、分类器、本地与远程裁决 | 想完整理解一次授权需要跨多个模块追踪 |
| hooks、MCP、skills 足以覆盖大量用户扩展 | 扩展只能进入预留边界，不能重组整个运行时 |

---

## 3. 把两套系统放到同一张图上

### 3.1 同一个任务，两条不同的控制路径

```mermaid
flowchart TB
    TASK["用户任务"] --> DENTRY["dsh Client / Host"]
    TASK --> CENTRY["Claude Code REPL / QueryEngine"]
    subgraph DSH["DeepSeek Harness"]
        DENTRY --> DLOOP["可替换 Agent Loop 插件"]
        DLOOP --> DLOG["先写 Session Event Log"]
        DLOG --> DLLM["可替换 LLM Runtime"]
        DLLM --> DTOOL["Tool Runtime waterfalls"]
        DTOOL --> DSEAM["FS / Shell / Sandbox / Approval seams"]
        DSEAM --> DLOG
    end
    subgraph CC["Claude Code"]
        CENTRY --> CQUERY["固定 query() 控制流"]
        CQUERY --> CMSG["维护 messages 数组"]
        CMSG --> CAPI["Anthropic Streaming API"]
        CAPI --> CTOOL["权限 + hooks + tool.call"]
        CTOOL --> CMSG
    end
    DLOG --> DOUT["投影后输出到 Web / CLI / SDK"]
    CMSG --> COUT["yield 到 TUI / SDK，并写 JSONL"]
```

### 3.2 核心概念映射

| 你要找的东西 | DeepSeek Harness | Claude Code | 本质差异 |
|---|---|---|---|
| 主循环 | `ReactLoopAgent` 的 turn / step | `queryLoop while(true)` | 事件化可替换状态机 vs 单控制流产品内核 |
| 模型接口 | `LlmRuntime` + adapter provider | Anthropic API 客户端及云 provider 适配 | 通用能力接缝 vs 深度绑定统一 API 语义 |
| 工具注册 | scoped `ToolRuntime` | 排序后的 `Tool[]` | 作用域服务注册表 vs 稳定 prompt cache 工具池 |
| 工具结果 | canonical JSON → render → session event | `ToolResult` → `tool_result` message | 可重放数据与展示分离 vs 直接消息回填 |
| 权限 | pre-execute waterfall + approval seam | 规则 + 工具自检 + hooks + 分类器 + UI | 框架边界 vs 产品级多层策略 |
| 会话真相 | append-only event log + surface projection | JSONL transcript + 内存 messages | 事件溯源模型 vs 消息转录模型 |
| 子代理 | 多 provider `SubagentRuntime` | AgentTool 复用 `query()` | 异构运行时抽象 vs 同构能力复用 |
| 扩展 | Cordis 插件、service、event、patch | hooks、MCP、skills、声明式 plugin | 可重组内核 vs 扩展固定内核 |
| UI | Host / Client + Typert RPC，Web-first | Ink fork + Raw Terminal，Terminal-first | 多端嵌入 vs 单端体验深挖 |

### 3.3 为什么它们会做出不同选择

```mermaid
flowchart LR
    DGOAL["目标：成为可嵌入 Agent 运行时"] --> DCHOICE["选择：插件树、能力接缝、事件日志、类型化 RPC"]
    DCHOICE --> DRESULT["结果：可替换、可审计、多端，但概念更多"]
    CGOAL["目标：成为开箱即用的终端编码产品"] --> CCHOICE["选择：单 query()、Anthropic cache、深权限、全栈 TUI"]
    CCHOICE --> CRESULT["结果：体验一致、性能集中优化，但内核更封闭"]
```

不能只问“哪种架构更先进”。更有效的问题是：

- 如果你在做 Agent 平台、需要换模型/沙箱/子代理后端，dsh 的接缝思路更值得借鉴。
- 如果你在做单一终端产品、想统一优化启动、缓存、权限和交互，Claude Code 的单控制流更直接。
- 如果你只想加一个外部数据源，两者都不需要动主循环：dsh 注册工具或 provider；Claude Code 接 MCP 或 plugin。

---

## 4. 四个具体场景，帮助你把图落回工程

### 场景 A：替换模型供应商

- dsh：实现或选择 `LlmAdapter` provider，在 profile / patch 中替换模型相关插件行；Agent Loop 不需要知道供应商细节。
- Claude Code：通常仍走产品支持的 Anthropic API 兼容路径、Bedrock / Vertex / Foundry 配置；不是替换 `query()` 的插件。

### 场景 B：增加一个文件处理工具

- dsh：定义 canonical output schema、执行函数和纯展示函数，注册进 agent scope；权限与沙箱从工具管线和能力 seam 复用。
- Claude Code：实现 `Tool` 的 Zod schema、`call`、权限检查、并发/只读属性与 React 渲染钩子，或通过 MCP 暴露工具。

### 场景 C：对危险命令增加公司审批

- dsh：在 `tools/pre-execute` 做策略判断，把 `ask` 交给可替换 `ApprovalService`；命令仍经 Sandbox seam 执行。
- Claude Code：组合 permission rules、PreToolUse hook 与交互处理；产品内置的安全红线继续拥有更高优先级。

### 场景 D：进程崩溃后恢复任务

- dsh：恢复 append-only 日志，修补未闭合的 tool / step / turn 事件，再从日志投影模型历史与状态。
- Claude Code：加载主 JSONL 和 sidechain，重建消息树、agent / 文件快照与成本状态，再回到 `query()`。

---

## 5. 推荐阅读路线

### 只想建立直觉（约 20 分钟）

1. [00 给 AI 学习者的导读](00-ai-learning-guide.md)
2. 本文第 1.1、1.3、2.1、2.3、3.1 节
3. [01 深度对比](01-comparison.md) 的结论与架构图附录

### 想自己设计 Agent（约 1–2 小时）

1. 本文完整阅读
2. [02 DeepSeek Harness 架构解读](02-dsh-architecture.md) 第 1–7 节
3. [03 Claude Code 源码解读](03-claude-code.md) 第 2、3、6、7、8、10 节

### 准备改源码

1. 从本文对应场景确定边界
2. 去 02 / 03 的“关键文件速查表”找入口
3. 沿一次真实请求只追一条调用链
4. 最后再展开旁路：缓存、遥测、UI、迁移和 feature flag

> 图负责建立关系，源码负责确认细节。遇到图与未来版本源码不一致时，以对应版本的源码、测试和运行行为为准。
