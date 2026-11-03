# 04 会话、分支与上下文

## 结论

会话文件是**只追加的 JSONL 树**。每条记录带 `id` 和 `parentId`，当前 `leafId` 决定活跃分支；模型上下文永远从活跃分支重新投影出来，而不是维护一个可变的消息数组。

官方格式文档：`packages/coding-agent/docs/session-format.md`（英文，比代码好读，建议对照）。

## 文件长什么样

位置：`~/.pi/agent/sessions/--<cwd>--/<timestamp>_<id>.jsonl`

第一行是 header，不进树：

```json
{"type":"session","version":3,"id":"uuid","timestamp":"...","cwd":"/path"}
```

后面每行一个 `SessionEntry`：

```json
{"type":"message","id":"a1b2c3d4","parentId":"...","timestamp":"...","message":{...}}
```

`version 3` 是当前版本；v1 是线性数组，加载时自动迁移（`session-manager.ts:286` 附近）。

## 条目类型

`session-manager.ts:183` 的联合类型：

| type | 作用 | 进模型上下文？ |
|---|---|---|
| `message` | user / assistant / toolResult / system | 是 |
| `model_change` | 记录模型切换 | 否，但影响后续请求 |
| `thinking_level_change` | 推理等级 | 否，影响后续请求 |
| `compaction` | 压缩摘要 | 是，替代 `firstKeptEntryId` 之前的消息 |
| `branch_summary` | 分支摘要 | 是（作为摘要消息） |
| `custom` | 扩展私有数据 | 否 |
| `custom_message` | 扩展注入的消息 | 是，转成 user 消息 |
| `context_edit` | 删除或替换某条消息的内容 | 是（投影时生效） |
| `label` | 给节点起名 | 否 |
| `session_info` | 会话显示名 | 否 |
| `usage` | 用量记录 | 否 |

注意 `custom` 和 `custom_message` 的区别：前者只是扩展存档，后者真的会进 prompt。

## 树与分支

- 每条记录一个 `parentId`，根节点为 `null`
- `leafId` 指向「当前所在位置」，不落盘，由加载时最后一条记录恢复
- 活跃分支 = 从 `leafId` 沿 `parentId` 一路走到根，再反转
- 切分支不需要复制文件：把 `leafId` 指到旧节点，下一条记录就接在它后面

所以在同一个文件里可以有多个分支，物理上就是一棵树。

`fork` / `clone` 是另一回事：它们创建**新文件**，header 里带 `parentSession`，内容复制选中的历史。

## 上下文投影

`buildSessionContext()`（`session-manager.ts:577`）负责：

1. 取活跃分支的条目
2. 过滤掉不进上下文的类型
3. 应用 `context_edit`（删除或替换）
4. 应用 `compaction`：`firstKeptEntryId` 之前的消息被摘要替代
5. 应用 system 消息的增量补丁：后出现的 system 消息按 section 名覆盖，`toolsAdded` / `toolsRemoved` 调整工具声明
6. 得到 `SessionContext { messages, thinkingLevel, model }`

关键点：**投影是纯函数**。同一份文件，同样的 leaf，永远得到同样的上下文。这让恢复、fork、压缩都好推理。

## 压缩

触发点三处：

- 手动 `/compact`
- 请求前检查（`agent-session.ts:3032` 的 `_checkCompaction`）
- 模型报上下文溢出后

产物是一条 `compaction` 条目：

```json
{"type":"compaction","summary":"...","firstKeptEntryId":"...","tokensBefore":12345}
```

原始消息**不删除**，只是投影时不再使用。所以压缩后仍可以回看原文，也可以从压缩点之前 fork。

## 不变量

1. 只追加，不修改已有行（迁移除外）
2. 每条记录都有 `id` 和 `parentId`；header 除外
3. `leafId` 永远指向文件里存在的记录
4. 投影只读；写上下文只能通过追加新条目
5. 工具调用和结果必须在同一分支上成对出现

## 恢复与 fork 时发生什么

- 继续：读文件 → 迁移 → 重建树 → `leafId` = 最后一条 → `buildSessionContext` → 作为 `Agent` 初始消息
- fork：新建文件 → 复制目标分支到分叉点 → 新 leaf 指向分叉点
- clone：复制整条活跃分支到新文件

对应 `main.ts` 里的 `openSessionOrExit`（`:338`）、`forkSessionOrExit`（`:348`）。

## 我的补充

<!-- 画一个三分支的会话树；记录你不确定的投影规则 -->
