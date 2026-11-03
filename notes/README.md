# Pi 源码阅读笔记

记录 Pi（`packages/` 下的 TypeScript monorepo）的阅读过程。目标不是逐行注释，而是搞清楚**一条请求从哪进来、经过谁、在哪停**，然后照着写一遍。

## 基线

- 代码版本：`@earendil-works/pi-coding-agent` 1.5.0
- 记录时间：2026-11
- 文中 `文件:行号` 只用于快速定位，升级后可能漂移，别当契约
- 主链代码约 1500 行；仓库总共约 2500 个 `.ts`，绝大部分是外围

## 目录

| 文件 | 内容 | 先读？ |
|---|---|---|
| `00-architecture.md` | 三层架构、包地图、五个关键接口 | 是 |
| `01-request-lifecycle.md` | 一条消息从回车到工具结果的完整链路 | 是 |
| `02-agent-loop.md` | 两层循环、steer/followUp 队列、终止与中止 | 是 |
| `03-ai-layer.md` | `StreamFn`、事件流、provider 分发 | 第二遍 |
| `04-session-context.md` | JSONL 会话树、分支、投影、压缩 | 第三遍 |
| `05-tools.md` | 工具的注册、执行、截断、并发 | 第三遍 |
| `06-ui-modes.md` | 四种界面如何共用同一个会话 | 选读 |
| `open-questions.md` | 待解决问题、下次验证点 | 每次结束看 |
| `daily/` | 每日记录：完成内容、卡点、明天从哪开始 | 每天 |
| `glossary.md` | 术语与 Java 概念对照 | 随时查 |
| `templates/mechanism-card.md` | 新机制笔记模板 | 复制用 |

## 阅读路线

**第一遍：骨架**（约 1 天）

- [ ] `00-architecture.md`
- [ ] `packages/coding-agent/docs/how-pi-works.zh-CN.md`（中文译本）
- [ ] `01-request-lifecycle.md`
- [ ] `02-agent-loop.md`
- [ ] 读完后合上笔记，手画一遍调用链

**第二遍：模型与流**

- [ ] `packages/agent/src/types.ts`（`AgentEvent`、`AgentContext`、`AgentLoopConfig`）
- [ ] `packages/ai/src/compat.ts` 的 `streamSimple`
- [ ] 一个具体 provider：`packages/ai/src/api/openai-completions.ts`
- [ ] `packages/ai/src/utils/event-stream.ts`
- [ ] `03-ai-layer.md`

**第三遍：状态**

- [ ] `packages/coding-agent/src/core/session-manager.ts`
- [ ] `agent-session.ts` 的 `_handleAgentEvent`
- [ ] `core/compaction/`
- [ ] `04-session-context.md`

**第四遍：工具与外围**

- [ ] `05-tools.md`
- [ ] `read.ts` → `bash.ts` → `output-accumulator.ts`
- [ ] `06-ui-modes.md`

## 每日记录

| 日期 | 完成 | 产出 | 遗留 |
|---|---|---|---|
| 2026-11-03 | 读完 `00`；弄清 `createRuntime` 工厂、类型别名与参数传递 | `00-architecture.md`、`how-pi-works.zh-CN.md` | Q1–Q5，见 `open-questions.md` |

每天结束时：在 `daily/` 下新建 `YYYY-MM-DD.md`，把当天读了什么、卡在哪、明天从哪开始写清楚，再在上表加一行。

## 使用方式

1. 每读一个机制，复制 `templates/mechanism-card.md` 建一张卡，文件名用机制名，例如 `cards/compaction.md`。
2. 每篇笔记末尾都有「我的补充」，只在那里写自己的话，避免把结论和过程混在一起。
3. 不确定的地方用 `???` 开头标注，之后回填；不要当场猜一个结论写死。
4. 手写实现时，在对应笔记里加一节「我实现的差异」，记录哪些复杂度是需求逼出来的，哪些只是原项目的选择。

## 约定

- 触发条件：谁在什么状态下调用它
- 状态变化：调用前后哪些字段变了
- 不变量：一旦破坏就会出 bug 的约束
- 失败路径：abort、超时、模型报错、工具抛异常
- 不要按文件记笔记，按机制记；一个机制可以横跨三个文件
