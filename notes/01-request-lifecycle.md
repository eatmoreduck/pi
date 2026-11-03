# 01 一条消息的完整旅程

从敲下回车，到工具结果回灌、agent 停止。这条链走通了，剩下的都是分支。

## 总图

```
argv
 └─ main.ts:574                装配：参数、设置、会话、模型、运行时
     └─ main.ts:997           选界面（961 rpc / 963 tui / 997 print）
         └─ session.prompt()   入口有三个：print-mode.ts:132、
            │                 interactive-mode.ts:1280、rpc 命令
            ├─ agent-session.ts:2113   扩展拦截、模板展开、队列、鉴权
            └─ _startRun:1894
                ├─ _prepareRun:1947   压缩检查、before_agent_start、拼消息
                └─ agent.prompt:382
                    └─ runAgentLoop:111 → runLoop:172
                        ├─ streamAssistantResponse:390
                        │    └─ streamFn（sdk.ts:412）
                        │        └─ ModelRuntime.streamSimple（model-runtime.ts:731）
                        │            └─ prepareRequest → provider.streamSimple → 具体 API 实现
                        └─ executeToolCalls:518
                             └─ tool.execute() → toolResult → 下一轮请求
```

## 分段看

### 1. 进程启动

`src/cli.ts:6` 只有两件事：`setupCli()` 装进程级兜底（信号、stdout 保护之类），然后 `main(argv)`。

构建产物入口是 `dist/bundle/cli.js`；源码里从 `src/cli.ts` 开始读即可。

### 2. 参数与装配

`main.ts:574` 是一个很长的函数，但它只做装配，不做业务：

- 解析参数、迁移旧配置、加载设置（`SettingsManager`）
- 解析项目信任（trust），决定要不要加载项目级扩展
- 选会话文件（新会话 / 继续 / fork）
- `createAgentSessionRuntime`（`agent-session-runtime.ts:420`）包住「换会话」的能力
- `createAgentSession`（`core/sdk.ts:191`）真正把 `Agent`、`AgentSession`、工具、系统提示词拼起来

**这是全项目最值得先读的函数**：谁依赖谁，看它的 import 和构造函数参数就够了。

### 3. 选界面

`main.ts:961/963/997` 三个分支分别进 RPC、交互式、print。它们拿到的是同一个 `runtimeHost`，里面是同一个 `AgentSession`。

### 4. 提交输入

三个入口最终都调用 `session.prompt(text, options)`：

| 界面 | 位置 |
|---|---|
| print | `modes/print-mode.ts:132` |
| interactive | `modes/interactive/interactive-mode.ts:1280`（另有 steer/followUp 分支在 3430/4580 附近） |
| rpc | `modes/rpc/` 里按命令分发 |

### 5. `AgentSession.prompt`

`agent-session.ts:2113`。顺序很重要：

1. 如果上一个 run 正在收尾（`_isEmittingAgentSettled`），把这次调用**延后**，不丢
2. `/` 开头的扩展命令先执行，命中就直接返回，不进模型
3. 压缩进行中则直接抛错
4. `_runInputHandlers`：扩展可以在这一步改写或吞掉输入
5. 展开 `/skill:` 和 prompt 模板
6. **如果正在流式输出**：按 `streamingBehavior` 进 `steer` 或 `followUp` 队列，然后返回
7. 校验模型与鉴权
8. `_startRun`

第 6 步是交互式体验的关键：`prompt()` 不是「阻塞到回答完」，而是「忙的时候只排队」。

### 6. `_startRun` 与 `_prepareRun`

`_startRun:1894`：

- `_prepareRun:1947` 先做**发送前压缩检查**（上一条 assistant 消息可能已经超窗）
- 触发 `before_agent_start` 扩展钩子，扩展可以改模型、改工具集、加消息
- 把用户消息、待处理的 nextTurn 消息、钩子返回的消息拼成一个数组
- `await this.agent.prompt(messages)`
- 之后是一个循环：`_handlePostAgentRun()` 返回 true 就 `agent.continue()`，再跑一轮
- `finally` 里发 `agent_settled`，释放等待者

`_prepareRun` 还会做图片规范化；它不负责读 `@file`——文件内容在 CLI 参数处理（`main.ts:223`）或界面层就已经变成消息内容了。

### 7. 进入循环

`agent.ts:382` → `runPromptMessages:441` → `runAgentLoop:111`。

`runPromptMessages` 用 `runWithLifecycle` 包了一层，负责创建 `AbortController`、把事件转给 `processEvents`、在结束时清理。真正的循环是 `agent-loop.ts` 里的纯函数。

### 8. 一次模型请求

`streamAssistantResponse:390`：

1. `transformContext`（扩展可以改历史）
2. `convertToLlm`（`AgentMessage[]` → 模型消息）
3. 解析 API key（可能是会过期的 OAuth token，所以每次请求都解析）
4. 调 `streamFunction`
5. 逐事件消费：`start` 时先塞一条 partial 消息，`text_delta`/`toolcall_delta` 时原地替换，`done`/`error` 时落最终消息

partial 消息会被放进 `context.messages`，所以订阅者能实时看到半成品；最终消息替换同一个位置。

### 9. 工具执行

`executeToolCalls:518`：

- 默认并行（`executeToolCallsParallel:596`）
- 只要有一个工具的 `executionMode === "sequential"`，或配置要求串行，就整体串行（`:540`）
- 每个调用：`tool_execution_start` → 执行 → `tool_execution_end` → 生成 `toolResult` 消息 → `message_start/end`
- `stopReason === "length"` 时不执行工具，直接给每个调用造一个失败结果，因为参数很可能被截断了

### 10. 结果回灌与落盘

工具结果被 push 进 `currentContext.messages`，`hasMoreToolCalls` 为真，内层循环继续 → 再发一次请求。

与此同时，所有事件都通过 `Agent.subscribe` 传到 `AgentSession._handleAgentEvent`（`agent-session.ts:1118`）：

- `message_end` → `SessionManager.appendMessage`（`:1215`）落盘
- 转发给扩展
- `_emit` 给界面

**对话消息的落盘只走这一条路径**；模型切换、压缩、label 等另有专门的 append 方法，入口同样都在 `AgentSession`。要找「什么时候写文件」，先来这里。

## 三个容易看漏的点

1. **`prompt()` 可能根本没跑**：扩展命令、输入被吞、忙时排队，都会让它提前返回。
2. **一次 `prompt()` 可能包含多次模型请求**：工具调用、followUp 队列、自动重试、压缩重试都能再开一轮。
3. **事件是唯一的旁路**：界面不直接读 `Agent` 状态，扩展也不直接改 transcript，全部走事件和钩子。

## 失败路径

| 情况 | 表现 | 位置 |
|---|---|---|
| 模型未配置 / 鉴权失败 | `prompt()` 抛错，不进 run | `agent-session.ts:2180` 附近 |
| 用户中止 | `AbortController` 取消，当前 assistant 消息标记 aborted，队列消息退回 | `agent.ts:350` |
| 模型返回 error | `stopReason === "error"`，结束本轮并发 `agent_end` | `agent-loop.ts:255` |
| 输出被截断（length） | 所有工具调用判失败，不执行 | `agent-loop.ts:279` |
| 上下文溢出 | `_checkCompaction` 触发压缩，然后重试 | `agent-session.ts:3032` |
| 工具抛异常 | 包装成 `isError: true` 的 toolResult，循环继续 | `agent-loop.ts:869` |

## 实验建议

- 在 `agent-session.ts:1894`、`agent-loop.ts:172`、`streamAssistantResponse` 各打一条日志，跑 `pi -p "hi"`，看顺序
- 跑一次带工具的提问，数一数 `turn_start` 出现了几次
- 流式过程中再输入一句话，看它进的是 steer 还是 followUp

## 我的补充

<!-- 写下你打断点观察到的实际顺序、和你预期不符的地方 -->
