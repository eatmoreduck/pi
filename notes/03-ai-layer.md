# 03 模型层与事件流

## 结论

`ai` 包把「每家 provider 的 HTTP/SSE」压成一种东西：一个会吐 `AssistantMessageEvent` 的异步流。上层只认这个流，不认 provider。

## 两个类型

```ts
// agent/src/types.ts:33
type StreamFn = (
  model: Model<Api>,
  context: TranscriptContext,
  options?: SimpleStreamOptions,
) => AssistantMessageEventStream | Promise<AssistantMessageEventStream>;
```

`AssistantMessageEvent`（`ai/src/types.ts:823`）的成员：

| 事件 | 用途 |
|---|---|
| `start` | 给出 partial 消息，上层先占位 |
| `text_start/delta/end` | 正文增量，按 `contentIndex` 定位 |
| `thinking_start/delta/end` | 推理内容增量 |
| `toolcall_start/delta/end` | 工具调用参数增量；`end` 时才拿到完整 `toolCall` |
| `done` | 终态：`stop` / `length` / `toolUse` / `deferred` |
| `error` | `aborted` / `error`，带一条完整的错误消息 |

每个事件都带 `partial`：当前已经拼到一半的 `AssistantMessage`。上层不用自己拼字符串，替换整条消息即可。

## EventStream 的双重身份

`ai/src/utils/event-stream.ts:26` 的 `EventStream<T, R>` 同时是：

- `AsyncIterable<T>`：调用方 `for await` 逐事件消费
- 一个带 `.result(): Promise<R>` 的对象：想要最终结果的调用方直接 await

因此同一个请求既可以流式渲染，也可以 `completeSimple()` 一次拿结果。实现上是两个 FIFO 队列：一个放待发事件，一个放等待中的消费者。

这是个很值得抄的设计：**生产者推、消费者拉、结果单独收口**，不需要回调，也不需要把流读完才能拿结果。

## 请求怎么落到具体 provider

调用链（coding-agent 路径）：

```
Agent 构造时注入 streamFn            sdk.ts:412
  → ModelRuntime.streamSimple        model-runtime.ts:731
      → prepareRequest               解析鉴权、请求头、maxTokens、thinking
      → provider.streamSimple        具体 API 实现
```

`ai/src/compat.ts:278` 的 `streamSimple` 是**旧的全局注册表入口**，按 `model.api` 找 provider。SDK 用户可能走它，但 `coding-agent` 自己走 `ModelRuntime`。读代码时别把两条路混了。

`ModelRuntime` 还负责虚拟模型：如果一个 model 是虚拟的，先把请求路由到真实模型，再发出去（`model-runtime.ts:731` 的 `isVirtualModel` 分支）。

## provider 实现长什么样

以 `ai/src/api/openai-completions.ts` 为例，一个 provider 只做三件事：

1. 把统一的 `Context`（system / user / assistant / toolResult）翻译成该家的请求体
2. 发 HTTP，解析 SSE
3. 边解析边发 `text_delta` / `toolcall_delta`，最后发 `done` 或 `error`

难点不在协议，而在：

- 增量 JSON 的拼接（工具参数是一段一段来的），见 `ai/src/utils/streaming-json.ts`
- 各家 stop reason 到 `stop | length | toolUse` 的映射
- thinking / reasoning 字段的兼容
- 错误归类：可重试（429、5xx）vs 不可重试（400）

## 鉴权

- `getApiKey` 每次请求都调用，因为 OAuth token 会过期
- `ModelRuntime` 持有凭据存储，provider 只拿解析好的 key
- 环境变量兜底在 `ai/src/env-api-keys.ts`

## 重试与溢出

- `ai/src/utils/retry.ts`：延迟与可重试判定
- `ai/src/utils/overflow.ts`：判断这次失败是不是上下文超限
- 循环层看到 `stopReason === "error"` 就结束本轮；重试由 `AgentSession` 的 `auto_retry_*` 逻辑接管，不在 `agent` 包里

## 自己写一个 provider 要做什么

最小清单：

1. 实现 `streamSimple(model, context, options)`
2. 返回一个 `EventStream`，`isComplete = e => e.type === "done" || e.type === "error"`
3. 每收到一段文本，更新 partial 消息并 emit `text_delta`
4. 工具调用结束时 emit `toolcall_end`，附完整 `ToolCall`
5. 确保任何异常都变成 `error` 事件，而不是抛出去

只要满足这五条，上层循环不用改一行。

## 我的补充

<!-- 记录你实际调试 SSE 时踩的坑；哪家的字段最别扭 -->
