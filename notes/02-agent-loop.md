# 02 Agent 循环

## 结论

`runLoop` 是两层 `while`：

- **内层**：只要「还有工具调用」或「还有 steer 消息」，就继续请求模型
- **外层**：内层要停时，再看有没有 followUp 消息；有就回到内层

一个 turn = 一次 assistant 响应 + 它触发的全部工具执行与结果。

## 伪代码

```ts
runLoop(context, newMessages, config, signal, emit, streamFn) {
  pending = config.getSteeringMessages()

  while (true) {                      // 外层：followUp
    hasMoreToolCalls = true

    while (hasMoreToolCalls || pending.length > 0) {   // 内层：工具 + steer
      if (lastCompletedTurn) {
        snapshot = config.prepareNextTurn(lastCompletedTurn)  // 可换模型、可压缩
        pending ||= config.getSteeringMessages()
        emit(turn_start)
      }

      for (m of declareToolChanges(context, prepared + pending)) emit(message_start/end)
      pending = []

      context = config.prepareRequest(context)       // 每轮重建请求视图
      message = streamAssistantResponse(context)      // 一次模型请求
      newMessages.push(message)

      if (message.stopReason === "error" || "aborted") {
        finishTurn(...); emit(turn_end); emit(agent_end); return
      }

      toolCalls = message.content.filter(toolCall)
      hasMoreToolCalls = false
      if (toolCalls.length) {
        results = executeToolCalls(...)               // 并行或串行
        hasMoreToolCalls = !results.terminate
        context.messages.push(...results)
      }

      decision = finishTurn(...)
      emit(turn_end)
      if (decision === "end") { emit(agent_end); return }

      pending = config.getSteeringMessages()
    }

    followUps = config.getFollowUpMessages()
    if (followUps.length) { pending = followUps; continue }   // 回到内层
    if (explicitContinuation) { ...; continue }
    break
  }
  emit(agent_end)
}
```

对应 `packages/agent/src/agent-loop.ts:172`（`runLoop`），入口 `:111`（`runAgentLoop`）、`:137`（`runAgentLoopContinue`）。

## 为什么是两层

三种「用户又说了一句」的时机需要不同处理：

| 队列 | 入队 API | 语义 | 消费点 |
|---|---|---|---|
| steering | `agent.steer()`（`agent.ts:308`） | 当前 assistant 轮次一结束就插入，保留它的工具结果 | 内层每轮开始前 `getSteeringMessages` |
| followUp | `agent.followUp()`（`:313`） | agent 本来要停时才插入，让模型再答一次 | 内层退出后 `getFollowUpMessages` |

如果只有一层，followUp 会被当成普通消息立刻发送，打断正在进行的工具批次。反过来，如果 steer 走外层，用户就会发现「我明明打断它了，它还在把活干完」——这恰恰是 steer 想要的行为。

`QueueMode` 控制每次取一条还是全部（`one-at-a-time` / `all`），在 `agent-session.ts` 里由设置同步。

## 上下文是怎么组装的

每轮请求前有两次机会改上下文：

1. `prepareNextTurn`：上一轮结束后、下一轮开始前。用于压缩、换模型、注入消息。**可以很慢**，所以取 steer 消息放在它之后再次轮询。
2. `prepareRequest`：真正发请求前。返回新的 context / model / thinkingLevel。

`declareToolChanges`（`agent-loop.ts:342`）负责把「可用工具集」和「transcript 里声明的工具集」的差异，写成一条 system 消息插进去。原因是会话恢复后，模型看到的 transcript 必须自解释：它当时能用哪些工具。

## 事件顺序

一次典型的「工具调用」run：

```
agent_start
turn_start
message_start (user) / message_end (user)
message_start (assistant, partial)
message_update × N
message_end (assistant)
tool_execution_start
tool_execution_update × N
tool_execution_end
message_start (toolResult) / message_end
turn_end
turn_start                       ← 第二轮
...
agent_end
```

`AgentEvent` 定义在 `agent/src/types.ts:528`；会话级事件（`agent_settled`、`queue_update`、`compaction_*`、`auto_retry_*`）定义在 `agent-session.ts:209`。两级别混用，读代码时注意 `emit` 的是哪一个。

## 终止条件

同时满足才停：

- 没有工具调用（或工具批次返回 `terminate: true`）
- 没有 steer 消息
- 没有 followUp 消息
- `finishTurn` 没有要求 `continue`

任一不满足就再来一轮。注意 `finishTurn` 的 `end` 是**立即结束**，不会再查 followUp。

## 中止

- `Agent.abort()`（`agent.ts:350`）abort 当前 `AbortController`
- `signal` 一路传到 `streamFunction` 和 `tool.execute`，由它们自己响应
- 串行工具批次每执行完一个检查一次 `signal.aborted`，然后 break
- `AgentSession` 在 `agent_start` 时若发现「启动期间已被要求中止」，立即补一次 abort

中止不等于回滚：已经落盘的消息留在会话里，状态由 `stopReason: "aborted"` 表达。

## 不变量

1. 每个 `toolCall` 最终必须对应一个 `toolResult`，即使失败也要有
2. `message_update` 只对 assistant 消息发出；partial 消息始终占据 `context.messages` 的同一个位置
3. 工具批次要么全部串行，要么全部并行，不会混合
4. `agent_end` 只发一次，且在 `turn_end` 之后
5. 循环本身不落盘、不读文件；它只操作内存里的 `AgentContext`

## 阅读时的疑问

- 为什么 `pending` 在处理完 prepared 消息后要清空？→ 避免同一条 steer 在下一轮被重复消费
- `prepareNextTurn` 慢操作期间进来的 steer 会不会丢？→ 处理完快照后会再轮询一次，但只在第一次没取到消息时轮询（防止 `one-at-a-time` 一次取两条）
- `explicitContinuation` 是什么？→ `finishTurn` 返回 `continue` 但既不因工具、也不因队列而要再请求时，补一个只有上下文的 turn

## 我的补充

<!-- 画一遍状态机；记录你没想明白的分支 -->
