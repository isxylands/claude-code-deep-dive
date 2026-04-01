# Claude Code 2026-03-31 源码快速分析报告

## 1. 结论先行

这份 `extracted-source` 更像一套 **agent runtime / harness**，而不是一个“带 prompt 的 CLI”。

如果只盯着 system prompt、tools 列表、或者某个 agent prompt，会低估它真正的工程重点。源码里最关键的设计不是“模型怎么说话”，而是：

1. **如何把 prompt、tool、permission、hook、task、MCP、remote session 组装成一个可控执行面。**
2. **如何让实验开关、eval harness、ablation baseline、feature gate 在运行时可灰度、可回滚、可复现实验。**
3. **如何让同一套核心循环同时服务本地 REPL、SDK、bridge/remote-control、background task、remote agent。**

我对这份源码的核心判断是：

- 它的真正护城河不是单一 prompt，而是 **runtime governance plane**。
- “Harness Engineering” 在这里不是测试配套，而是 **模型外部、负责约束/增强/观测/调度模型行为的那一层系统工程**。
- 从实现风格上看，这个客户端已经接近一个 **Agent Operating System**：有状态机、有任务模型、有权限裁决层、有可插拔 MCP/skills/plugins、有本地与远程统一控制面。

## 2. 本次阅读范围与方法

### 2.1 样本边界

- 本次分析对象：`extracted-source`
- 总文件数：约 `1906`
- `src/` 顶层模块规模中，最重的不是 UI，而是：
  - `utils/` 564
  - `components/` 389
  - `commands/` 207
  - `tools/` 184
  - `services/` 130
  - `hooks/` 104
  - `ink/` 96

这说明两个事实：

1. 这是一个**高度产品化**的客户端，不只是推理循环。
2. 架构主导权不在 React/Ink 视图层，而在 `utils + tools + services + tasks + bridge` 这几个 runtime 模块。

### 2.2 重点阅读路径

本次没有试图逐文件通读，而是沿执行主链抓关键节点：

- 入口与启动：`src/entrypoints/cli.tsx`、`src/main.tsx`
- 核心执行循环：`src/QueryEngine.ts`、`src/query.ts`
- Prompt 装配：`src/constants/prompts.ts`、`src/utils/queryContext.ts`
- Tool 系统：`src/tools.ts`、`src/services/tools/toolExecution.ts`、`src/services/tools/toolOrchestration.ts`
- Agent / Task：`src/tools/AgentTool/*`、`src/tasks/*`
- Permission / Hook：`src/utils/permissions/*`、`src/utils/hooks.ts`
- MCP / 扩展面：`src/services/mcp/client.ts`、`src/commands.ts`
- Remote / Bridge：`src/bridge/*`、`src/remote/*`
- Memory / harness-only side channel：`src/memdir/memdir.ts`、`src/utils/claudeCodeHints.ts`

## 3. 架构主线

## 3.1 启动层不是单一路径，而是多入口统一 runtime

从 `src/entrypoints/cli.tsx` 和 `src/main.tsx` 可以看出，这个客户端不是单一 CLI：

- 有普通 CLI / REPL 启动路径
- 有 `--dump-system-prompt` 这种 eval/分析路径
- 有 `--claude-in-chrome-mcp`、`--chrome-native-host`
- 有 `remote-control / bridge` 路径
- 有 daemon / background sessions 路径

也就是说，Anthropic 在这里维护的是一套 **多表面共享的执行内核**，而不是一个“CLI UI + 若干特性”的程序。

一个容易被忽略的信号是 `src/entrypoints/cli.tsx` 顶部这段注释：

- `Harness-science L0 ablation baseline`
- 通过环境变量一次性关闭 thinking / compact / auto memory / background tasks

这说明启动层本身就是实验控制点，而不是纯粹的参数解析器。

## 3.2 QueryEngine + query loop 才是真正的核心

`src/QueryEngine.ts` 和 `src/query.ts` 构成了主循环：

- `QueryEngine.submitMessage()` 负责组织一次 turn 的上下文
- `fetchSystemPromptParts()` 负责拿到 systemPrompt / userContext / systemContext
- `query()` / `queryLoop()` 负责：
  - 发起模型请求
  - 接收流式结果
  - 识别 tool_use
  - 编排 tool 执行
  - 回收 tool_result
  - 触发 compact / stop hook / recovery 等逻辑

它不是“发一次请求然后等结果”的简单 loop，而是一个多阶段状态机，里面混合了：

- token budget 管理
- compact / microcompact / reactive compact
- stop hooks
- tool result budget
- withheld max_output_tokens recovery
- 命令生命周期通知

所以 Claude Code 的“智能性”并不只来自模型，而是来自这条循环把各种治理逻辑嵌进去。

## 3.3 Prompt 不是文案，而是 assembly pipeline

`src/constants/prompts.ts` 的价值不是一段长 prompt，而是一个 **system prompt assembly architecture**。

我认为这里最重要的点有三个：

1. Prompt 被切分成静态段和动态段，并用 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 做边界。
2. Prompt 装配会读取工具、模型、language、output style、memory、MCP instructions、hooks 说明等上下文。
3. Prompt 与 cache-key 绑定非常紧，`src/utils/queryContext.ts` 明确把 `systemPrompt + userContext + systemContext` 作为 API cache-key prefix 的核心材料。

这意味着 Claude Code 并不是“先写个 prompt，再想办法省 token”，而是一开始就按 **缓存友好、动态注入、可灰度调整** 的思路设计 prompt 层。

## 3.4 Tool 系统是模型执行面的抽象边界

`src/tools.ts` 是整个执行面的注册表。这里的重点不是工具数量，而是它体现了一个统一抽象：

- 工具有 schema
- 工具有 enable/gate
- 工具可按 feature flag / user type / experimental gate 挂载
- 工具池并不是固定的，而是随：
  - 用户类型
  - feature flag
  - MCP 连接
  - worktree / plan mode / swarm 模式
  - ToolSearch
  动态变化

再往下看 `src/services/tools/toolExecution.ts` 与 `toolOrchestration.ts`，可以看到工具调用不是直接执行，而是一个 pipeline：

1. 校验输入
2. 权限裁决
3. pre-tool hooks
4. tool execution
5. post-tool hooks / failure hooks
6. telemetry / tracing / progress
7. tool result storage / truncation / persistence

这就是典型的 harness 思维：**模型请求工具只是发起意图，真正的执行由 harness 接管。**

## 3.5 Task 模型把“异步执行”正式化了

`src/Task.ts` 与 `src/tasks/*` 说明 Claude Code 并没有把 agent、shell、remote work 当成临时异步操作，而是建成了统一 task framework。

支持的任务类型包括：

- `local_bash`
- `local_agent`
- `remote_agent`
- `in_process_teammate`
- `local_workflow`
- `monitor_mcp`
- `dream`

这个建模非常关键，因为它让系统可以统一处理：

- 生命周期
- 中断与 kill
- 输出文件
- 通知
- 后台运行
- 任务恢复
- UI 展示

如果没有这层任务抽象，多 agent、多后台、多远程的体验会迅速失控。

## 3.6 Agent 体系已经不是“递归调用自己”这么简单

`src/tools/AgentTool/AgentTool.tsx`、`runAgent.ts`、`forkSubagent.ts` 展示了三种不同的 agent 执行模式：

- **普通 local agent**
- **fork subagent**
- **remote / worktree / in-process teammate**

这里最有价值的不是“能开子 agent”，而是它把子 agent 的隔离方式也工程化了：

- 继承上下文但保留缓存一致性的 fork
- 通过 worktree 做文件隔离
- 通过 remote session 做环境隔离
- 通过 AsyncLocalStorage 做 in-process teammate 身份隔离

这说明它不是粗暴地“多开几个 LLM call”，而是在认真处理：

- 上下文继承边界
- 权限边界
- 文件系统边界
- 会话与转录边界
- 任务通知边界

## 3.7 扩展体系是 product surface，不是附属插件

`src/commands.ts`、`src/services/mcp/client.ts`、skills/plugins 相关模块共同说明了一件事：

Claude Code 把扩展能力分成了几层不同的 surface：

- Commands：用户显式控制面
- Tools：模型可调用执行面
- MCP：外部能力接入面
- Skills：prompt-native workflow 封装
- Plugins：更高层的能力分发与策略面

这几个层面不是重复设计，而是职责拆分：

- Commands 面向人
- Tools 面向模型
- MCP 面向外部系统
- Skills 面向工作流复用
- Plugins 面向分发、信任与政策

这是非常成熟的产品架构，不是边写边堆的临时集成。

## 3.8 Remote / Bridge 是第二套执行平面

`src/bridge/bridgeMain.ts`、`src/bridge/sessionRunner.ts`、`src/remote/RemoteSessionManager.ts` 表明系统已经具备一套完整的 remote control / remote execution 平面：

- 本地桥接进程负责拉取 work、spawn child session、heartbeat、token refresh
- child CLI 通过 NDJSON / control_request 暴露权限请求
- remote session manager 负责 WebSocket 收消息、HTTP 发消息、回传 permission response

这意味着 Claude Code 不是“把本地 CLI 搬上云”，而是把 **本地与远程统一成同一种 session protocol**。

这点很重要，因为它解释了为什么 background tasks、ultraplan、ultrareview 这些能力能够在产品上自然落地。

## 4. Harness Engineering 机制拆解

## 4.1 Harness 的真正定义

从源码用词来看，这里的 harness 不是 test harness 的狭义概念，而是：

> 模型之外、负责环境准备、行为约束、执行代理、权限裁决、实验控制、信息侧信道、持久化与观测的那一层系统。

证据非常直接：

- `update-config` skill 明写：`the harness executes these, not Claude`
- `memdir.ts` 反复写：`Harness guarantees the directory exists`
- `claudeCodeHints.ts` 写明：`hints are a harness-only side channel`
- `cli.tsx` 直接出现 `Harness-science L0 ablation baseline`
- `growthbook.ts` 多次提到 `eval harnesses`

所以如果要研究 Claude Code 的能力来源，最应该研究的是 harness，而不是单看模型输出。

## 4.2 实验、灰度、ablation、determinism 是一等公民

这是我认为最值得重视的工程特征。

`src/services/analytics/growthbook.ts` 明确支持：

- remote eval
- experiment exposure logging
- env var override
- config override
- 缓存兜底
- auth change 后重建 client

关键点在于：

1. **实验不只是开关，而是可记录曝光、可回放、可覆盖。**
2. **env override 优先于 remote eval，用于保证 eval harness 的确定性。**
3. **启动阶段直接支持 ablation baseline，一次性关闭一组高级能力。**

这套设计说明 Anthropic 在持续做两件事：

- 功能灰度
- 行为科学实验

换句话说，Claude Code 不是“先做功能，后补分析”，而是从一开始就把 **可实验、可比较、可回滚** 作为产品机制写进去了。

## 4.3 Prompt cache 与行为一致性是硬约束

很多地方都能看到他们对 prompt cache 和 byte-level consistency 的执念，最典型的是：

- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`
- `queryContext.ts` 对 cache-key prefix 的抽象
- `forkSubagent.ts` 明确要求 fork child 复用 byte-identical API prefix

特别是 fork subagent 那段设计很说明问题：

- 不重新生成 system prompt
- 直接传递已渲染的 prompt bytes
- tool_result 也尽量使用固定占位内容

这不是小优化，而是一个架构原则：

> 只要一个设计会破坏 prompt cache 命中或导致实验噪音，它就会被视为架构问题。

这类思路很“Harness Engineering”，因为它关注的是系统行为的稳定性，而不是某次推理是否凑巧有效。

## 4.4 权限系统不是一层 if/else，而是裁决引擎

`src/utils/permissions/permissionSetup.ts` 与 Bash/PowerShell 相关权限文件非常值得重视。

核心特征：

- 有 permission mode
- 有 rule source 区分
- 有危险 rule 检测
- 有 classifier
- 有 auto mode gate
- 有 working directory / internal path / sandbox 等不同来源的裁决理由

尤其值得注意的是：

1. 它显式防止“过宽规则”绕过 classifier，比如 `python:*`、`bash:*` 这类解释器级别放行。
2. 它把 **internal harness paths** 单独列入白名单读取面，说明系统内部文件和用户工作区文件被区别对待。
3. Bash 权限逻辑里大量注释都在讨论 feature flag、DCE cliff、classifier coverage、复合命令安全等问题，说明这不是表面权限，而是不断被攻击面和误判率打磨过的系统。

我的判断是：权限层是 Claude Code 最大的工程投入之一，也是它能从“会调用工具”进化到“敢默认执行工具”的前提。

## 4.5 Hooks 是 harness 级自动化，不是记忆增强

`src/utils/hooks.ts` 和 bundled `update-config` skill 对 hooks 的定义非常清晰：

- hooks 是 shell command
- 在 session/tool/task/permission 等生命周期节点触发
- 真正执行者是 harness，不是模型
- 可同步、可异步、可 background

这里有一个很容易误解的点：

> “以后每次 X 都做 Y” 这类能力，真正的落点不是 memory，而是 hooks + settings。

这实际上把 Claude Code 从“会聊天的 agent”推进到“可配置自动化 runtime”。这也是我建议后续重点深挖的方向，因为它决定了产品从一次性问答走向长期工作流的能力上限。

## 4.6 Memory 机制也被 harness 化了

`src/memdir/memdir.ts` 里有一个特别关键的细节：系统会先 `ensureMemoryDirExists()`，然后 prompt 明示“目录已经存在，直接写，不要 mkdir，不要检查”。

这类设计很值得注意，因为它不是功能性创新，而是 **降低模型在环境交互上的无谓试探**。

换句话说，Harness 在这里做的是：

- 预先创建环境
- 在 prompt 中告知保证条件
- 让模型跳过低价值探测动作

这类看似细碎的“环境保障”实际上很重要，因为 agent 系统的大量损耗不是来自大决策，而是来自无意义的 existence check、重复探测、错误的环境假设。

## 4.7 Harness-only side channel 是产品化信号

`src/utils/claudeCodeHints.ts` 很有代表性：

- 外部 CLI/SDK 可以在 stderr 中输出 `<claude-code-hint />`
- harness 会扫描并剥离这些 tag
- 模型看不到这些 tag
- UI 只把它们转成用户安装提示

这说明系统已经开始建立 **模型不可见、只供 runtime/UI 使用的协议层**。

这是一个非常成熟的产品化信号，因为它意味着：

- 不是所有信息都要经过模型
- 某些交互由 harness 直接处理更稳定
- 模型在系统里不是唯一控制器

我认为这类 side channel 未来只会越来越多。

## 4.8 多种隔离形态共同构成 harness 执行边界

Claude Code 目前至少有四种隔离方式：

1. **同会话本地执行**
2. **worktree 隔离**
3. **remote session 隔离**
4. **in-process teammate 身份隔离**

这说明 Anthropic 并不相信“一个 agent + 一个 cwd”能覆盖所有场景，而是把隔离形态做成了 runtime primitive。

这是正确方向。因为真正难的不是生成一个子任务，而是：

- 子任务是否污染父上下文
- 子任务是否污染工作区
- 子任务权限是否独立
- 子任务能否恢复/通知/中止

这正是 harness 在解决的问题。

## 5. 我对当前架构的评价

## 5.1 最强的地方

### A. 它把 agent 能力做成了工程系统，而不是 prompt 花活

这点最关键。

很多产品表面上像 agent，实质还是“LLM + function calling”。Claude Code 已经明显超过这个阶段，因为它把：

- prompt
- task
- permission
- hooks
- MCP
- telemetry
- remote execution
- recovery / compact

全部纳入了一个一致的运行模型。

### B. 它非常重视实验确定性和缓存稳定性

这不是“优化细节”，而是平台成熟度的信号。

只要一个系统开始主动处理：

- env override for eval harness
- ablation baseline
- prompt prefix byte stability
- experiment exposure logging

就说明团队已经在做系统性迭代，而不是靠人工感受判断模型是否“更好了”。

### C. 它把“模型应该少做什么”想得很清楚

例如：

- harness 先保证 memory 目录存在
- hints 直接由 harness 拦截，不让模型介入
- internal paths 走专门的权限白名单
- hook 自动化明确归 harness，不归 memory

这些都体现了一种成熟判断：

> 不要把所有事情都交给模型。让模型只做适合它做的部分。

## 5.2 我认为存在的风险与复杂度代价

### A. feature flag + DCE + lazy require 组合已经很重

源码里大量存在：

- `feature('...')`
- dead code elimination 注释
- lazy require
- ant-only / external build 分歧

这对实验速度很有帮助，但长期看会带来两个风险：

1. 模块边界被构建策略污染
2. 某些行为只在特定组合下出现，测试矩阵会膨胀

换句话说，这套系统很强，但也已经进入“需要持续压复杂度”的阶段了。

### B. query loop 的责任继续膨胀会有维护风险

`query.ts` 已经承载了很多横切逻辑：

- compact
- stop hooks
- tool orchestration
- retry / recovery
- token budget
- command lifecycle

如果继续把新特性直接挂进 query loop，而不是外提成更稳定的 phase/pipeline abstraction，后续理解成本和回归风险会继续上升。

### C. 扩展面很多，信任边界管理会越来越难

它同时支持：

- MCP
- plugin
- skill
- hook
- remote
- teammate

这意味着未来最难的不是“再加能力”，而是：

- 谁能带来什么能力
- 谁能修改什么 prompt / context
- 谁能访问什么路径
- 谁的错误会污染主会话

所以从中长期看，Claude Code 的核心竞争力很可能会逐渐从 agent 编排，转向 **trust and policy architecture**。

## 6. 下一步最值得继续深挖的方向

如果你下一轮还要继续研究，我建议不要平均用力，而是优先深挖下面四块：

1. **Permission classifier 全链路**
   - Bash / PowerShell classifier
   - dangerous rule 检测
   - auto mode 与 classifier 的交互

2. **AgentTool 调度与多隔离模式**
   - fork / worktree / remote / in-process teammate
   - 子 agent prompt/cache 继承边界

3. **Hooks + settings 组成的长期自动化层**
   - 这是 Claude Code 从“助手”变成“工作流 runtime”的关键

4. **Bridge / Remote session protocol**
   - 这是 background tasks、remote review、ultraplan 的基础设施层

## 7. 关键文件索引

- 启动与入口
  - `extracted-source/src/entrypoints/cli.tsx`
  - `extracted-source/src/main.tsx`
- 核心执行循环
  - `extracted-source/src/QueryEngine.ts`
  - `extracted-source/src/query.ts`
  - `extracted-source/src/utils/queryContext.ts`
- Prompt 装配
  - `extracted-source/src/constants/prompts.ts`
- Tool / 执行编排
  - `extracted-source/src/tools.ts`
  - `extracted-source/src/services/tools/toolExecution.ts`
  - `extracted-source/src/services/tools/toolOrchestration.ts`
- Agent / Task
  - `extracted-source/src/tools/AgentTool/AgentTool.tsx`
  - `extracted-source/src/tools/AgentTool/runAgent.ts`
  - `extracted-source/src/tools/AgentTool/forkSubagent.ts`
  - `extracted-source/src/tasks/LocalAgentTask/LocalAgentTask.tsx`
  - `extracted-source/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`
  - `extracted-source/src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`
- Permission / Hook / Memory
  - `extracted-source/src/utils/permissions/permissionSetup.ts`
  - `extracted-source/src/tools/BashTool/bashPermissions.ts`
  - `extracted-source/src/utils/permissions/filesystem.ts`
  - `extracted-source/src/utils/hooks.ts`
  - `extracted-source/src/memdir/memdir.ts`
- Harness / 实验 / 侧信道
  - `extracted-source/src/services/analytics/growthbook.ts`
  - `extracted-source/src/utils/claudeCodeHints.ts`
  - `extracted-source/src/skills/bundled/updateConfig.ts`
- 远程执行
  - `extracted-source/src/bridge/bridgeMain.ts`
  - `extracted-source/src/bridge/sessionRunner.ts`
  - `extracted-source/src/remote/RemoteSessionManager.ts`
  - `extracted-source/src/services/mcp/client.ts`

## 8. 最后判断

如果只用一句话概括这份源码：

> Claude Code 的本质不是“一个会调用工具的模型客户端”，而是一套把模型包进可实验、可治理、可扩展、可远程执行的 runtime harness。

如果再往前推一步，我会给出一个更尖锐的判断：

> 真正值得学习的，不是它“让模型更像工程师”，而是它如何用 Harness Engineering 让一个本来不可靠的模型，在产品层面表现得越来越像一个可控的软件系统。
