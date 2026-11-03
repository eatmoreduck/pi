# 05 工具

## 结论

循环只认识 `AgentTool`：一个名字、一个 JSON Schema、一个 `execute()`。具体工具怎么实现、怎么渲染，循环不关心。

## 两种「工具」

| 类型 | 位置 | 谁用 |
|---|---|---|
| `AgentTool` | `agent/src/types.ts:478` | 循环执行用 |
| `ToolDefinition` | `coding-agent/src/core/extensions/types.ts` | 扩展注册、UI 渲染、生成给模型的 schema |

`tool-definition-wrapper.ts` 把 `ToolDefinition` 包成 `AgentTool`。这么分层是因为扩展既想控制执行，也想控制终端里怎么画。

## 内置工具

`core/tools/index.ts` 的 `ToolName`：

`read`、`bash`、`powershell`、`edit`、`write`、`grep`、`find`、`ls`

默认编码工具集是 `read + bash + edit + write`（`createCodingTools`），只读模式是 `read + grep + find + ls`。

## 一次调用经过什么

`agent-loop.ts` 的 `prepareToolCall:718`：

1. 按名字在 `context.tools` 里找；找不到 → 直接造一个错误结果（不抛）
2. `prepareArguments`（可选兼容层）
3. 按 schema 校验参数；不合法 → 错误结果
4. `beforeToolCall` 钩子：可以 block，也可以改参数
5. 检查 abort
6. `tool.execute(id, args, signal, onUpdate)`
7. `afterToolCall` 钩子：可以改结果
8. 包装成 `toolResult` 消息

任何一步抛异常都在这里被捕获并转成 `isError: true` 的结果。**工具失败不会中断循环**，模型会看到错误并决定下一步。

## 并发模型

- 默认并行：所有调用先按顺序 prepare，再同时 `execute`
- 只要有任一工具声明 `executionMode: "sequential"`，整批串行
- 串行批次每完成一个检查 `signal.aborted`

并行只对「读」类工具安全；`edit` / `write` 靠 `file-mutation-queue.ts` 按真实路径串行化，避免同一个文件被并发改坏。

## 输出控制

所有工具共用 `truncate.ts` 的默认值：

- 最多 2000 行
- 最多 50 KB

`read` 用 `truncateHead`（保留开头）；`bash` 经 `OutputAccumulator` 用 `truncateTail`（保留结尾，因为错误通常在后面）。

`bash` 还用了 `OutputAccumulator`：流式输出只保留一个有界尾部，超过限制时把完整内容写临时文件，结果里给路径。这样内存不会因为一条 `yes` 命令爆掉。

`bash` 的 `structuredContent` 另有一个 1 MB 上限，供程序化调用方（codemode）使用。

## 权限

- 工具以 Pi 进程的操作系统权限运行
- 没有沙箱；project trust 只决定「加载哪些项目资源」，不限制工具
- 所以安全边界是「你信不信这个仓库」，不是「工具能不能越权」

这一条在 `docs/security.md` 里写得很直接，手写实现时不要误以为有隔离。

## 自己实现工具时

最小要求：

1. 参数用 JSON Schema 描述清楚，模型才能正确调用
2. 失败时抛异常或返回 `isError: true`，不要只在正文里说「失败了」
3. 长输出必须截断，并在结果里告诉模型被截断了
4. 写文件类工具要处理并发（同一路径串行）
5. 支持 `signal`，否则用户中止后进程仍会被子进程拖住

## 我的补充

<!-- 对比 read.ts 和 bash.ts 的 execute 签名；记录你觉得多余或必要的参数 -->
