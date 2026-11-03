# 待解决问题

规则：每次阅读结束把没想通的问题记在这里，标上代码位置；第二天先从这里开始。解决后把状态改成「已解决」，并补一句结论。

| 编号 | 问题 | 状态 |
|---|---|---|
| Q1 | `AgentSession` 和 `Agent` 是什么关系 | 已简答，待验证 |
| Q2 | `AgentSession.prompt` 和 `Agent.prompt` 的区别 | 已简答，待验证 |
| Q3 | `AgentEvent` 到底干什么，是不是别的 Agent 发事件进来 | 已纠正，待验证 |
| Q4 | `AssistantMessageEvent` 和 `AgentEvent` 的区别 | 已简答，待验证 |
| Q5 | 工具调用之后还会再跑一遍 loop 吗 | 已简答，待验证 |

---

## Q1 `AgentSession` 和 `Agent` 是什么关系

**简答**：组合，不是继承。`AgentSession` 的构造参数第一个就是 `agent`（`agent-session.ts:265` 的 `AgentSessionConfig`，实例化在 `sdk.ts:455`）。

分工：

- `Agent`：循环内核。只管消息、模型、工具、steer/followUp 队列
- `AgentSession`：应用层。持有 `Agent`，负责落盘、压缩、扩展、事件分发、模型与设置

**下次验证**：读 `AgentSession` 构造函数（`agent-session.ts:498`）和字段声明，列出「Session 有而 Agent 没有」的职责。

---

## Q2 `AgentSession.prompt` 和 `Agent.prompt`

**简答**：调用与被调用，不在同一层。

```
AgentSession.prompt (2113)   扩展命令、输入钩子、模板展开、忙时排队、鉴权
  └─ _startRun (1894)
       └─ _prepareRun (1947)  压缩检查、before_agent_start、拼 user 消息
            └─ Agent.prompt (382)   只接收已就绪的消息，启动循环
```

区别：

- Session 层会「决定要不要发、发什么」；Agent 层只负责「已经决定要发，那就跑」
- 正在流式时，`Agent.prompt` 会直接抛错，排队是 Session 层做的

**下次验证**：在 `agent-session.ts:2113` 和 `agent.ts:382` 各打一条日志，观察一次提问的调用顺序。

---

## Q3 `AgentEvent` 的作用

**原来的猜测**：其他 Agent 处理完，把事件传进当前 Agent。

**纠正**：不是。这里没有 Agent 之间的通信。`AgentEvent` 是**一个 Agent 向外广播自己的生命周期**：

- 谁发：`agent-loop.ts` 里的 `emit(...)`
- 谁收：`AgentSession` 在构造时订阅（`agent-session.ts:524`），再转发给扩展和界面
- 用途：落盘（`message_end`）、渲染（`message_update`）、工具进度（`tool_execution_*`）

一个会话只有一个 Agent；事件是「内 → 外」，不是「外 → 内」。

**下次验证**：搜 `agent.subscribe`，确认所有订阅者；再看 `_handleAgentEvent` 怎么处理每种事件。

---

## Q4 `AssistantMessageEvent` 和 `AgentEvent`

**简答**：层级不同，前者被后者包着。

| | AssistantMessageEvent | AgentEvent |
|---|---|---|
| 来源 | provider 的流式响应 | 循环的生命周期 |
| 典型事件 | `text_delta`、`toolcall_delta`、`done` | `turn_start`、`message_start`、`tool_execution_end` |
| 定义 | `ai/src/types.ts:823` | `agent/src/types.ts:528` |

`AgentEvent` 的 `message_update` 里带着 `assistantMessageEvent`：

```ts
{ type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
```

一句话：provider 吐的是「字」，循环吐的是「阶段」。

**下次验证**：在 `streamAssistantResponse` 里打印 provider 事件类型，再在 `_handleAgentEvent` 打印循环事件类型，对比两层。

---

## Q5 工具调用之后还会再跑 loop 吗

**简答**：会。这正是 agent 和普通聊天接口的区别。

流程：

1. assistant 消息里带 `toolCall`
2. `executeToolCalls` 执行，产出 `toolResult` 消息
3. 结果 push 进 `context.messages`，`hasMoreToolCalls = true`
4. 内层 `while` 再走一轮，带上工具结果重新请求模型
5. 直到没有工具调用、没有 steer/followUp 消息，才 `agent_end`

**下次验证**：问一个需要读文件的问题，数 `turn_start` 出现几次；再对照 `agent-loop.ts:277` 附近的 `hasMoreToolCalls` 赋值。

---

## 已解决

（解决后把问题挪到这里，写一句结论和代码位置）
