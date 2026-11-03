# 00 架构总览

## 一句话

Pi 是「会话状态」和「模型循环」分离的 agent harness：`coding-agent` 管装配、落盘和界面，`agent` 管循环，`ai` 管 provider 适配。三者之间只靠 `AgentEvent` 和 `StreamFn` 两个接口耦合。

## 三层与依赖方向

```
coding-agent ──▶ agent ──▶ ai
      │
      ├──▶ tui                     终端渲染原语（只有 interactive 用）
      ├──▶ mcp / codemode
      └──▶ schema / env / telemetry
```

依赖是单向的：`agent` 不认识 `coding-agent`，也不认识任何具体 provider。这是整个设计里最重要的一条边界，手写时优先保住它。

| 层 | 包 | 职责 | 不该做的事 |
|---|---|---|---|
| 外壳 | `packages/coding-agent` | CLI 参数、配置、会话文件、系统提示词、内置工具、四种界面、扩展加载 | 不实现循环本身 |
| 循环 | `packages/agent` | 组装请求、消费流式响应、执行工具、管理 steer/followUp 队列 | 不知道文件系统，不读配置 |
| 适配 | `packages/ai` | 模型目录、鉴权、provider 请求、统一事件流 | 不关心会话和工具 |

## 包地图

| 包 | 行数（src，不含生成文件） | 是否主链 | 说明 |
|---|---|---|---|
| `coding-agent` | ~77k | 是 | CLI 与全部运行时装配 |
| `agent` | ~2.5k | 是 | 循环，最值得逐行读 |
| `ai` | ~28k | 是 | provider 与流式协议 |
| `tui` | ~20k | 否 | 终端渲染原语，只被 interactive 模式使用 |
| `chord` | 未统计 | 否 | 独立的对象运行时，与 agent 主链无关 |
| `durable` | 未统计 | 否 | 持久化执行相关，独立 |
| `codemode` / `mcp` / `schema` / `env` / `telemetry` / `evals` | 小 | 否 | 按需再读 |

注：`ai/src/models.generated.ts` 是生成的数据文件，不要读。

## 装配顺序（谁 new 了谁）

```
src/cli.ts:6                  main(argv)
  └─ src/main.ts:574          main()
       ├─ parseArgs           参数
       ├─ SessionManager      会话文件
       ├─ createAgentSessionRuntime  agent-session-runtime.ts:420
       └─ createAgentSession  core/sdk.ts:191
            ├─ ModelRuntime        模型目录 + 鉴权 + 请求路由
            ├─ SettingsManager     设置
            ├─ DefaultResourceLoader  系统提示词、技能、扩展
            ├─ new Agent(...)      sdk.ts:403
            └─ new AgentSession(...)  sdk.ts:455
```

关键点：`Agent` 在 `sdk.ts` 里被显式注入了一个 `streamFn`（`sdk.ts:412`），内部转给 `ModelRuntime.streamSimple`。所以 `agent` 包本身没有任何 provider 依赖，`getDefaultStreamFn` 只是兜底。

## 五个关键接口

| 接口 | 位置 | 作用 |
|---|---|---|
| `AgentEvent` | `agent/src/types.ts:528` | 循环向外广播的唯一通道：message / turn / tool 生命周期 |
| `AgentSessionEvent` | `coding-agent/src/core/agent-session.ts:209` | 在 `AgentEvent` 之上加会话级事件：队列、压缩、重试、落盘 |
| `StreamFn` | `agent/src/types.ts` | `(model, context, options) => AssistantMessageEventStream` |
| `AgentTool` | `agent/src/types.ts` | 循环唯一认识的工具形态：名字、参数、`execute()` |
| `SessionEntry` | `coding-agent/src/core/session-manager.ts:183` | 会话树节点，全部落盘类型的联合 |

扩展（extension）不是接口，而是一组钩子：`before_agent_start`、context 变换、工具注册、UI 渲染。它挂在 `AgentSession` 上，不进 `agent` 包。

## 数据流

```
用户输入
  → AgentSession.prompt()        加扩展钩子、模板展开、鉴权检查
  → Agent.prompt()               组装 AgentContext（system + messages + tools）
  → runAgentLoop()               进入循环
  → streamAssistantResponse()    一次模型请求
  → StreamFn                     provider SSE
  → AssistantMessageEvent        text/toolcall 增量
  → AgentEvent                   循环级广播
  → AgentSession._handleAgentEvent   ① 落盘 ② 转发扩展 ③ 转发界面
  → 有 toolCall？执行，追加 toolResult，再请求
  → 无 toolCall 且无排队消息：agent_end
```

## 为什么这么切

问题：既要支持终端、print/JSON、RPC、SDK 几种入口，又要支持多家模型。

做法：把「会话」放在外层，把「循环」做成纯函数式内核，模型只以一个函数注入。于是 `agent` 包可以单独发布、单独测试，provider 增减不影响循环。

对照 Java：`agent` 相当于一个不依赖 Spring 的领域服务；`coding-agent` 是装配它的 Spring 配置 + Controller；`ai` 是 JDBC 驱动。会话落盘类似 event sourcing，但只追加、不回放副作用。

## 阅读时先别管

- `extensions/` 的具体实现（先知道有钩子即可）
- `tui` 的渲染细节
- `chord`、`durable`
- 模型目录与鉴权的细节（知道 `ModelRuntime` 负责即可）

## 我的补充

<!-- 在这里写：哪一层和你的预期不一样；如果重写会怎么分层 -->
