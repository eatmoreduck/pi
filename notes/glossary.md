# 术语表

按「代码里的词」查，不按中文查。Java 对照只是帮助理解，不是等价物。

| 术语 | 含义 | 代码位置 | Java 对照 |
|---|---|---|---|
| harness | 把模型、工具、上下文、会话串起来的运行时 | 整个 `coding-agent` | 应用框架 + 装配 |
| Agent | 循环本体：持有消息、模型、工具队列 | `agent/src/agent.ts:194` | 领域服务 |
| AgentSession | 在 Agent 外面包一层：落盘、压缩、扩展、事件分发 | `coding-agent/src/core/agent-session.ts:498` | 应用服务 / 事务边界 |
| AgentContext | 一次请求的输入快照：system + messages + tools | `agent/src/types.ts` | 方法参数对象 |
| AgentMessage | 循环内部的消息联合类型，比 LLM 消息多 `custom` 等 | `agent/src/types.ts` | 领域模型 |
| `Message` | 真正发给模型的消息：system/user/assistant/toolResult | `ai/src/types.ts` | DTO |
| message | 会话中的一条记录（`type: "message"`） | `session-manager.ts` | 事件日志中的一条 |
| turn | 一次 assistant 响应 + 它触发的工具执行 | `agent-loop.ts` 的 `turn_start` | 一次事务 |
| run | 一次 `prompt()` 到 agent 停止，可含多个 turn | `AgentSession._startRun` | 一次请求处理 |
| session | 对话的完整记录，落盘为 JSONL | `session-manager.ts` | 聚合根 |
| entry | 会话文件里的一行（消息、模型切换、压缩……） | `session-manager.ts:183` | 领域事件 |
| branch | 从根到某个 entry 的路径 | `getBranch()` | 快照 / 版本 |
| leaf | 当前所在 entry，决定活跃分支 | `leafId` | 当前版本指针 |
| projection | 从分支重建模型上下文的过程 | `buildSessionContext():577` | 读模型（CQRS 的查询侧） |
| compaction | 用摘要替代较早消息，原文保留 | `core/compaction/` | 快照压缩 |
| branch summary | 切分支时对旧分支的摘要 | `BranchSummaryEntry` | 归档说明 |
| steering | 当前轮次结束后插入的消息 | `agent.ts:308` | 插队消息 |
| follow-up | agent 将停时插入的消息 | `agent.ts:313` | 追加任务 |
| StreamFn | 请求模型的函数签名 | `agent/src/types.ts:33` | 函数式接口 |
| AssistantMessageEvent | provider 流式输出的统一事件 | `ai/src/types.ts:823` | 流式响应事件 |
| EventStream | 同时支持 `for await` 和 `.result()` 的流 | `ai/src/utils/event-stream.ts:26` | `Flux` + `CompletableFuture` |
| provider | 模型服务接入方 | `ai/src/api/*` | 驱动实现 |
| Tool | 循环可执行的工具：schema + `execute()` | `agent/src/types.ts:478` | 策略实现 |
| ToolDefinition | 扩展注册的工具：额外带渲染信息 | `coding-agent/.../extensions/types.ts` | 带视图的插件 |
| extension | 加载进进程的 TS 模块，注册钩子 | `core/extensions/` | 插件 + 拦截器 |
| skill | 按需加载的指令与文件 | `core/skills.ts` | 资源包 |
| prompt template | 提交前展开的文本模板 | `core/prompt-templates.ts` | 模板片段 |
| project trust | 是否加载项目级设置与资源 | `core/project-trust.ts` | 配置信任域 |
| SDK | 进程内创建并控制会话的 API | `core/sdk.ts` | 内嵌 API |
| RPC | 通过 stdin/stdout 的 JSONL 控制协议 | `modes/rpc/` | 进程间接口 |

## 容易混的三组

1. **Agent vs AgentSession**：前者只管循环，后者管「循环之外的一切」。看到落盘、压缩、扩展，基本都在 Session。
2. **AgentMessage vs Message**：前者是循环内部类型，含扩展自定义消息；后者是发给模型的四种角色。`convertToLlm` 负责转换。
3. **turn vs run**：一次提问可能包含多个 turn（每次工具调用后都会再请求一次）。日志里 `turn_start` 出现多次是正常的。

## 我的补充

<!-- 遇到新词就加一行，别另开文件 -->
