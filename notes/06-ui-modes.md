# 06 四种界面

## 结论

四种界面共用同一个 `AgentSession`，区别只在「输入从哪来、事件往哪写」。写自己的前端不需要碰 agent 包。

| 模式 | 入口 | 输入 | 输出 | 用途 |
|---|---|---|---|---|
| interactive | `modes/interactive/interactive-mode.ts` | 终端编辑器 | TUI 渲染 | 日常使用 |
| print | `modes/print-mode.ts` | 参数 | 最终文本 / stdout | 脚本、一次性任务 |
| json | 同上，`mode: "json"` | 参数 | 每行一个事件的 JSONL | 程序消费 |
| rpc | `modes/rpc/rpc-mode.ts` | stdin 的 JSONL 命令 | stdout 的 JSONL 响应与事件 | 外部进程控制 |
| SDK | `core/sdk.ts` | 直接调用 | 事件回调 | 嵌入应用 |

## 共同骨架

```
createAgentSessionRuntime()
  → runtimeHost.session
  → session.subscribe(handler)     // AgentSessionEvent
  → session.prompt(text)           // 或 prompt + streamingBehavior
```

`print-mode.ts` 是最短、最完整的例子（173 行），建议先读它：

- `session.subscribe` 里，json 模式把事件转成 JSONL
- text 模式在 `prompt()` 返回后，从 `session.state.messages` 取最后一条 assistant，打印其中的文本
- 出错时把 exit code 设为 1
- `finally` 里释放 runtime、flush stdout

## interactive 模式

7200 行，但结构不复杂：

- 编辑器、按键、粘贴、历史
- 订阅会话事件，把消息和工具进度渲染成终端组件
- slash 命令（`/model`、`/resume`、`/compact`……）
- 流式期间允许输入，按 steer / followUp 排队

不要把时间花在主题、快捷键、渲染细节上。只读两个地方：

1. 用户提交时调用 `session.prompt` 的位置（`interactive-mode.ts:1280` 附近）
2. 事件如何被转成 UI 状态

`tui` 包只提供渲染原语（布局、文本、输入框），不含任何 agent 逻辑。想换前端，替换这一层即可。

## rpc 模式

协议是「一行一个 JSON 对象」：命令进 stdin，响应和事件出 stdout。类型在 `modes/rpc/rpc-types.ts`。

适合：编辑器插件、GUI 外壳、远程控制。进程隔离带来的好处是崩了不影响宿主。

## json 模式

事件映射在 `modes/json-event.ts`，是 `AgentSessionEvent` 的稳定序列化。适合 CI、日志管道。

注意它输出的是**会话级事件**，不是原始 `AgentEvent`，所以会包含 `queue_update`、`compaction_*` 等。

## 写自己的界面时

1. 用 `createAgentSession`（SDK），不要自己拼 `Agent`
2. 先只订阅 `message_end`，能把对话显示出来
3. 再加 `tool_execution_start/update/end`
4. 最后处理 `queue_update`、`compaction_*`、`auto_retry_*`
5. 中止用 `session.abort()`；等待空闲用 `session.waitForIdle()`

## 我的补充

<!-- 记录你实际想做的界面形态；哪些事件是必须的，哪些可以忽略 -->
