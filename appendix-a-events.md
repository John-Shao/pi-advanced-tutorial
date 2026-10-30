# 附录 A　事件与钩子总表

← 上一章：[第 18 章 源码阅读地图与进阶路线](18-source-map-and-roadmap.md) ｜ [返回目录](README.md)

Pi 的扩展事件在代码里是 `ExtensionEvent` 判别联合（[packages/coding-agent/src/core/extensions/types.ts:1411](file:///D:/workspace/harness/earendil/pi/packages/coding-agent/src/core/extensions/types.ts#L1411)），展开 `SessionEvent` 后共 **41 个**具体事件。本附录按运行阶段分组，"类别"取第 9.4 节的六种组合语义（通知 / 原地修改 / 链式转换 / 门 / 边界 / 末位覆盖），"可返回"只列处理器**被读取**的返回值。

处理器一律按"扩展加载顺序 → 该扩展内注册顺序"执行；退订不影响在途派发；所有处理器被 `await`，因此在流式高频事件里做慢活会拖慢整条流。

## A.1 启动与资源（3）

| 事件 | 类别 | 可返回 | 载荷要点 |
|---|---|---|---|
| `project_trust` | 门 | `{ trusted: "yes" \| "no" \| "undecided", remember? }` | `cwd`。只有用户/全局与 `-e` 扩展参与；首个 yes/no 拥有决策并抑制内置提示；`remember` 落 `trust.json`，否则仅本进程 |
| `resources_discover` | 贡献值 | `{ skillPaths?, promptPaths?, themePaths? }` | `cwd`、`reason`（`startup` \| `reload`）。在 `session_start` 之后 |
| `mcp_servers_change` | 通知（有副作用） | — | `servers` 为变更后全量。处理它即把本扩展标记为"负责连接注册服务器"的一方 |

## A.2 会话生命周期（10）

| 事件 | 类别 | 可返回 | 载荷要点 |
|---|---|---|---|
| `session_start` | 通知 | — | `reason`（`startup` \| `reload` \| `new` \| `resume` \| `fork`）、`previousSessionFile`（new/resume/fork 时有） |
| `session_info_changed` | 通知 | — | `name`（清除时为 `undefined`） |
| `session_before_switch` | 门 | `{ cancel: true }` | `reason`（`new` \| `resume`）、`targetSessionFile`（仅 resume） |
| `session_before_fork` | 门 | `{ cancel: true }` \| `{ skipConversationRestore: true }`（保留） | `entryId`、`position`（`before` = `/fork`，`at` = `/clone`） |
| `session_before_compact` | 门 / 替换 | `{ cancel: true }` \| `{ compaction: { summary, firstKeptEntryId, tokensBefore, usage?, details? } }` | `preparation`（`messagesToSummarize`、`turnPrefixMessages`、`previousSummary`、`fileOps`、`tokensBefore`、`firstKeptEntryId`、`settings`）、`branchEntries`、`customInstructions`、`reason`（`manual` \| `threshold` \| `overflow`）、`willRetry`、`signal` |
| `session_compact` | 通知 | — | `compactionEntry`、`fromExtension`、`reason`、`willRetry` |
| `session_compact_failed` | 通知 | — | `reason`、`errorMessage`、`aborted`、`willRetry`、`fromExtension`。应与 `session_before_compact` 配对上报 |
| `session_before_tree` | 门 / 替换 | `{ cancel: true }` \| `{ summary: { summary, usage?, details? } }` \| `{ customInstructions, replaceInstructions, label }` | `preparation`（`targetId`、`oldLeafId`、`commonAncestorId`、`entriesToSummarize`、`userWantsSummary`）。总会被触发，无论用户是否选摘要 |
| `session_tree` | 通知 | — | `newLeafId`、`oldLeafId`、`summaryEntry`、`fromExtension`。**分支状态重建的第二入口**（与 `session_start` 并列） |
| `session_shutdown` | 通知 | — | `reason`（`quit` \| `reload` \| `new` \| `resume` \| `fork`）、`targetSessionFile`。清理要写成幂等 |

## A.3 输入（1）

| 事件 | 类别 | 可返回 | 载荷要点 |
|---|---|---|---|
| `input` | 门 / 链式转换 | `{ action: "continue" }` \| `{ action: "transform", text?, images? }` \| `{ action: "handled" }` | `text`（**原始**，技能与模板未展开）、`images`、`source`（`interactive` \| `rpc` \| `extension`）、`streamingBehavior`（`steer` \| `followUp` \| `undefined`）。扩展命令命中时本事件不触发 |

## A.4 运行与回合（7）

| 事件 | 类别 | 可返回 | 载荷要点 |
|---|---|---|---|
| `before_agent_start` | 链式转换 | `{ message?: { customType, content, display }, systemPrompt? }` | `messageType`（`user` \| `custom`）、`prompt`、`images`、`message`、`systemPrompt`（当前链上值）、`systemPromptOptions`（`customPrompt`、`forceSystemPrompt`、`selectedTools`、`hiddenTools`、`toolSnippets`、`toolGuidelines`、`promptGuidelines`、`sections`、`appendSystemPrompt`、`cwd`、`contextFiles`、`skills`） |
| `agent_start` | 通知 | — | 低层运行开始 |
| `turn_start` | 通知 | — | `turnIndex`、`timestamp` |
| `turn_end` | **边界** | `{ entries?, continue? }` | `turnIndex`、`message`、`messageEntryId`、`toolResults`、`outcome`、`entries`（此前提案）、`context`（据提案重建）。允许草稿类型：`custom`、`custom_message`、`context_edit`、`compaction` |
| `agent_end` | 通知 | — | `messages`、`willRetry`。**不是终态** |
| `agent_before_settle` | **边界** | `{ entries?, continue? }` | 最后一个可执行边界；中断时有效条目仍提交，但续跑被抑制 |
| `agent_settled` | 通知 | — | `aborted`。此后 `ctx.isIdle()` 为真；此处请求的工作延到全部处理器结束 |

## A.5 消息（3）

| 事件 | 类别 | 可返回 | 载荷要点 |
|---|---|---|---|
| `message_start` | 通知 | — | `message`。所有消息（user/assistant/toolResult/custom）都触发 |
| `message_update` | 通知 | — | `message`、`assistantMessageEvent`（逐 token 流事件，12 个成员，见 4.4）。高频路径，勿做慢活 |
| `message_end` | 链式转换 | `{ message }`（**必须同 `role`**，否则记为扩展错误并跳过该处理器） | 权威完成消息。示例：改写 `usage.cost.total` |

## A.6 工具执行（5）

| 事件 | 类别 | 可返回 | 载荷要点 |
|---|---|---|---|
| `tool_execution_start` | 通知 | — | `toolCallId`、`toolName`、`args`、`parentToolCallId`。并行模式下按助手源顺序在预检阶段发出 |
| `tool_call` | **门** / 原地修改 | `{ block: true, reason?, terminate? }` | `toolName`、`toolCallId`、`input`（**可原地改，无二次校验**）。用 `isToolCallEventType(name, event)` 收窄。抛错即阻塞（fail-closed） |
| `tool_execution_update` | 通知 | — | 加 `partialResult`。并行时可能交错 |
| `tool_result` | 链式转换 | 部分补丁：`{ content?, details?, structuredContent?, isError?, usage? }` | 未返回的字段保持原值；替换 `content` 而不返回 `structuredContent` 会丢弃结构化内容。嵌套调用在此带 `parentToolCallId` |
| `tool_execution_end` | 通知 | — | `result`、`isError`、`durationMs`（未执行时缺省）。**每个定型的调用都会触发**，包括未知工具、校验失败、被阻塞、被中断 |

## A.7 上下文与请求（6）

| 事件 | 类别 | 可返回 | 载荷要点 |
|---|---|---|---|
| `context` | 链式转换 | `{ messages }` | **不含系统消息**的深拷贝。改动后 Pi 会把当前提示词段与工具声明重放进头部一条系统消息，因此无法丢掉提示词；未改动时保留会话中途的系统消息 |
| `context_with_system` | 链式转换 | `{ messages }` | 完整 transcript（含头部系统消息与段/工具补丁）。**你拥有本次请求的提示词与工具声明**：索引 0 必须有系统消息（丢失则 Pi 报错）；删除某条系统消息会连带删除它携带的补丁与工具声明 |
| `before_provider_headers` | 原地修改 | — | `headers`。置字符串为新增/覆盖，置 `null` 为删除。每请求一次；重试复用同一批头 |
| `before_provider_request` | 链式转换 | 替换后的 payload，或 `undefined` 保持原值 | provider 专属 payload。可改写提供商级系统指令，但 `ctx.getSystemPrompt()` 看不到这些改动 |
| `after_provider_response` | 通知 | — | `status`、`headers`（在消费流之前）。抽象掉 HTTP 的 provider 可能不给头 |
| `provider_stream_event` | 通知（只读） | — | `provider`、`api`、`model`、`data`（Pi 能拿到的最早结构化值，**不是原始字节**）。按流顺序 await，改动可能影响归一化；不持久化 |

## A.8 模型、思考与缓存（3）

| 事件 | 类别 | 可返回 | 载荷要点 |
|---|---|---|---|
| `model_select` | 通知 | — | `model`、`previousModel`、`source`（`set` \| `cycle`）。**恢复会话不触发**，模型相关初始化放在 `session_start` 读 `ctx.model` |
| `thinking_level_select` | 通知 | — | `level`、`previousLevel`。设置键、快捷键、`pi.setThinkingLevel()` 都会触发 |
| `cache_warming_decision` | 末位覆盖 | `{ action: "warm" \| "stop" }` | 只带 Pi 的估计：`warmCost`、`missCost`、`continuationProbability`、`action`。其余从 `ctx.model`、`ctx.isIdle()`、`ctx.getContextUsage()` 取；`"stop"` 结束预热直到下一次真实请求 |

## A.9 用户 shell 与交互等待（3）

| 事件 | 类别 | 可返回 | 载荷要点 |
|---|---|---|---|
| `user_bash` | 门 | `{ operations }`（换后端执行）\| `{ result }`（不执行直接记账）\| `undefined` 交给下一处理器/本地执行 | `command`、`excludeFromContext`（`!!` 前缀）、`cwd`。抛错即阻塞命令，**不回落到本地执行** |
| `ui_prompt_start` | 通知（不等待） | — | `reason: "ui_prompt"`、`kind`（`select` \| `confirm` \| `input` \| `editor` \| `custom`）、`title`。嵌套/重叠提示被合并成一个外层等待区间 |
| `ui_prompt_end` | 通知（不等待） | — | 该等待区间结束 |

## A.10 顺序速查

处理器按固定顺序插入一次运行，这张顺序表把 A.1-A.9 串起来：

```text
启动    project_trust → session_start → resources_discover
输入    [扩展命令] → input → [skill/模板展开] → before_agent_start
运行    agent_start → turn_start
请求    context → context_with_system → before_provider_headers
        → before_provider_request → after_provider_response
        → provider_stream_event × n（伴随 message_start / message_update / message_end）
工具    tool_execution_start → tool_call → execute（tool_execution_update × n）
        → tool_result → tool_execution_end
回合    turn_end（边界）→ 下一 turn_start 或 agent_end
收尾    agent_end →（重试 / 溢出压缩 / 队列）→ agent_before_settle（边界）→ agent_settled
替换    session_before_switch|_fork|_compact|_tree → session_shutdown → 重载 → session_start
```

同一批工具调用的三个易错点：`tool_call` 之前 Pi 已把 `ctx.sessionManager` 同步到当前 assistant 消息，但**并行模式下兄弟调用的结果不保证可见**；`tool_execution_end` 按完成顺序、`toolResult` 消息按助手源顺序；未执行的调用（未知工具、校验失败、被阻塞、被中断）跳过 `tool_call` 与 `tool_result`，但仍有 `tool_execution_end`。

## 相关页面

- 机制解释：第 9 章（分发通道、组合语义、错误策略）
- 生命周期图：第 2 章（五段划分）、第 3 章（循环内部顺序）
- 边界事件的草稿类型与校验：7.5 节（`context_edit` 与恢复顺序）、第 6 章（entry 类型）
- 处理器写法：第 10 章（六类模式）、附录 B（API 与 `ctx` 形状）

---

→ 下一章：[附录 B 扩展 API 与上下文速查](appendix-b-extension-api.md) ｜ [返回目录](README.md)
