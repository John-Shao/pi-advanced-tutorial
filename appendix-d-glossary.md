# 附录 D　术语对照

← 上一章：[附录 C 数值、限额与配置速查](appendix-c-reference.md) ｜ [返回目录](README.md)

全书统一用下列译法。首次出现给"中文（英文）"，之后可只用中文；代码标识符、配置键、环境变量、文件名一律保留英文并加反引号。

| 英文 | 中文 | 定义 / 注意 |
|---|---|---|
| agent harness | 智能体运行框架 | Pi 的自我定位：最小内核 + 可扩展 |
| agent | 智能体 | |
| run | 运行 | 一次从输入到落定的工作；与"回合（turn）"区分 |
| turn | 回合 | 一次模型响应 + 该响应触发的全部工具调用 |
| low-level agent run | 低层运行 | `agent_start`…`agent_end` 之间；结束不代表无事可做 |
| settle | 落定 | `agent_before_settle`（可执行边界）→ `agent_settled`（终态） |
| continuation | 续跑 | 扩展在边界请求的再一次模型调用 |
| steering message | steering 消息 | 当前回合及其工具执行结束后插入（`Enter`） |
| follow-up message | follow-up 消息 | 全部待做工作之后插入（`Alt+Enter`） |
| queue mode | 队列模式 | `all` \| `one-at-a-time`（默认后者） |
| abort / cancellation | 中断 / 取消 | `AbortSignal` 链；一次 `Escape` 覆盖运行、重试、压缩、分支摘要 |
| provider | 提供商 | 模型服务方（`ProviderId`） |
| api | api（不译） | 分派用的字符串标识，如 `anthropic-messages`（4.2） |
| model | 模型 | `chat` \| `image` \| `classifier` 三类操作 |
| virtual model | 虚拟模型 | 可选条目，每次请求路由到物理模型（11.7） |
| thinking level | 思考等级 | `off`\|`minimal`\|`low`\|`medium`\|`high`\|`xhigh`\|`max` |
| `compat` | 兼容开关 | 描述**已验证**的差异，不要凭端点自称打开（11.5） |
| prompt cache | 提示词缓存 | 提供商侧；`cacheRead`/`cacheWrite`/`cacheWrite1h` |
| cache warming | 缓存预热 | 有成本模型（5.7），`kind: "cache_warm"` 的 usage entry |
| session | 会话（session） | JSONL entry 树，非"消息数组" |
| entry | entry（条目） | 树节点，11 种类型（附录 C.5） |
| projection | 投影 | `buildSessionProjection()` 的结果：条目 + 可见消息 |
| branch | 分支 | 树中一条到叶的路径；与 Git 分支区分时写"会话分支" |
| leaf | 叶 / 当前叶 | `getLeafId()`；上下文由叶向根构建 |
| compaction | 压缩 | 摘要旧消息以腾出上下文；原始条目不删 |
| branch summarization | 分支摘要 | `/tree` 导航时保留被放弃分支的语境 |
| `context_edit` | 上下文编辑 | append-only 覆写，只影响未来上下文与所在分支 |
| cut point | 切点 | 压缩保留边界的起点（7.3） |
| split user-message span | 拆分用户消息段 | 单段超预算时在段内切，产两摘要合并（7.3） |
| overflow recovery | 溢出恢复 | 省略失败尝试 → 压缩 → 重试一次（7.5） |
| tool | 工具 | 模型可调用单元；四重身份见 8.1 |
| tool call | 工具调用 | `ToolCall` 内容块 |
| `details` | `details`（不译） | 工具结果的结构化负载：渲染 + 状态重建 |
| structured output | 结构化输出 | `outputSchema` + `structuredContent`，给脚本而非模型 |
| exposure | 暴露（exposure） | `direct`\|`model-only`\|`codemode`\|`deferred`\|`hidden`（8.7） |
| loadout | 工具装载（loadout） | 请求携带的工具声明集合 |
| nested call | 嵌套调用 | `ctx.executeTool()`；带 `parentToolCallId`，不写 transcript |
| truncation | 截断 | 50 KB / 2000 行；`truncateHead`/`Tail`/`Line`/`Middle`（8.3） |
| file mutation queue | 文件变更队列 | `withFileMutationQueue()`，键为 realpath（8.4） |
| extension | 扩展（extension） | 进程内 TS 模块，同权限运行 |
| jiti | jiti（不译） | 直接求值 TS 的加载器（9.1） |
| virtual module | 虚拟模块 | 把指定包名映射到 Pi 自带副本（9.1） |
| handler composition | 处理器组合语义 | 六种：通知 / 原地修改 / 链式转换 / 门 / 边界 / 末位覆盖（9.4） |
| actionable boundary | 可执行边界 | 文档用语；代码是 `emitBoundary()`（9.3） |
| hot reload | 热重载 | `/reload`，仅对自动发现位置生效 |
| Pi package | Pi 包 | 不写"软件包"，以免与 npm 包混淆 |
| skill | 技能（skill） | `SKILL.md` 目录；描述常驻、正文按需读 |
| prompt template | 提示词模板 | 文件名即 `/command` |
| theme | 主题 | JSON；token 与 schema 同源（13.6） |
| context file | 上下文文件 | `AGENTS.md` 家族；不受信任门控 |
| project trust | 项目信任 | 控制加载哪些项目资源，**不是**沙箱（17.5） |
| agent directory | agent 目录 | 默认 `~/.pi/agent` |
| TUI | 终端界面（TUI） | `@earendil-works/pi-tui` |
| differential rendering | 差分渲染 | 按改动区间重写；条件见 13.3 |
| fullscreen / regular mode | 全屏 / 常规模式 | `tuiMode`；鼠标与 scrollback 归属不同 |
| keybinding id | 键位 id | `app.*`（coding-agent）与 `tui.*`（共享） |
| IME | 输入法 | 光标标记与 `focused` 传播（13.4） |
| print mode | 打印模式 | 只输出最终文本 |
| JSON mode | JSON 模式 | 单向 JSONL 事件流 |
| RPC mode | RPC 模式 | 双向 JSONL，长驻子进程 |
| disposition | 处置（`disposition`） | `prompt` 命令响应里的数据字段（15.3） |
| MCP server | MCP 服务器 | stdio / streamable HTTP（拒绝 SSE） |
| namespace | 命名空间 | 相关工具分组，MCP 服务器即一个命名空间 |
| codemode | Codemode | 保留英文，首字母大写；QuickJS 脚本编排 |
| sandbox | 沙箱 | 本文指 codemode 的 VM/Worker 隔离；OS 级隔离称"容器化/隔离"（17.5） |
| BM25 | BM25 | `tool_search` 的检索算法，非嵌入（12.7） |
| token | token | 不译为"令牌" |
| usage / cost | 用量 / 成本 | `Usage` 结构见附录 C.5 |
| replicated state | 复制状态 | chord 的快照 + delta 模型（16.2） |
| durable | 持久（durable） | 指 `pi-durable` 的"先提交后展示"，非会话持久化 |

## 相关页面

- [前言](00-preface.md) 的正文约定
- 第 1 章（分层与包名）、附录 A（事件）、附录 B（API）、附录 C（数值与配置）

---

→ 下一章：[目录索引](README.md) ｜ [返回目录](README.md)
