# 第 18 章 源码阅读地图与进阶路线

← 上一章：[第 17 章 工程实践：调试、测试、性能与贡献](17-engineering-practice.md) ｜ [返回目录](README.md)

> **本章你将学会**：用"三圈法"分阶段建立对 Pi 源码的可 working 模型（内圈一天、中圈两天量级）；按症状直接跳到该看的文件；用非交互手段端到端追一个 bug；选一个能真正练到多章知识的进阶项目；跟踪 Pi 的演进。
>
> **对应官方文档**：[Overview](https://pi.dev/docs/latest) · 仓库 `CONTRIBUTING.md`、`AGENTS.md`、`packages/coding-agent/docs/`

本教程前面的章节按知识体系组织，本章按"你要做事"组织。它也是全书的收尾：给一条从读代码到改代码再到影响上游的路线。

## 18.1 三圈阅读法

```mermaid
flowchart LR
    A["内圈：契约与格式<br/>约 3000 行<br/>读完能说出系统边界"] --> B["中圈：编排<br/>关键四文件约 3200 行<br/>读完能预测时序"]
    B --> C["外圈：实现与策略<br/>其余全部<br/>按需读，不要通读"]
```

| 圈 | 文件（行数取自仓库实测） | 读它的收益 |
|---|---|---|
| 内圈 | `packages/agent/src/types.ts`（539，`AgentMessage`、`AgentEvent`、`QueueMode`、`ToolExecutionMode`） | 循环层的全部类型都定义在这里，且不依赖实现 |
| 内圈 | `packages/ai/src/types.ts` 的前 500 行（共 915；内容块、`Usage`、`SystemMessage`、`Message`、`AssistantMessageEvent`） | 提供商边界与流协议 |
| 内圈 | `packages/coding-agent/src/core/messages.ts`（196）+ `session-manager.ts` 的类型段（前 200 行，共 2028） | entry 树与角色折叠 |
| 内圈 | `packages/coding-agent/src/core/system-prompt.ts`（229，可整读） | 提示词的段结构与 diff |
| 内圈 | `packages/coding-agent/src/core/extensions/types.ts`（2314，只读判别联合与 `ExtensionAPI`/`ExtensionContext` 接口两段） | 扩展的全部可行动作 |
| 中圈 | `packages/agent/src/agent-loop.ts`（959，务必通读 `runLoop`） | 时序的唯一真相 |
| 中圈 | `packages/agent/src/agent.ts`（623） | 队列、abort、钩子字段 |
| 中圈 | `packages/coding-agent/src/core/extensions/runner.ts`（1565） | 分发语义与错误策略 |
| 中圈 | `packages/coding-agent/src/core/sdk.ts`（481）+ `agent-session-services.ts`（233）+ `agent-session-runtime.ts`（447） | 组合根与替换语义 |
| 中圈按需 | `resource-loader.ts`（1319）、`package-manager.ts`（2895）、`main.ts`（1010） | 发现/安装/启动三处的顺序 |
| 外圈 | `packages/coding-agent/src/core/agent-session.ts`（4479） | **只按需要读**：用 18.2 的索引跳转，别通读 |
| 外圈 | `packages/ai/src/api/*.ts`、`core/tools/*`、`core/compaction/*`、`modes/*`、`packages/tui/src/*` | 具体实现与策略 |

一条经验：Pi 的行为差异几乎总能在**内圈的格式定义**里找到解释。想不通"为什么提示词是段而不是字符串"，就去看 `SystemMessage.sections`；想不通"为什么压缩后旧消息还在文件里"，就去看 `CompactionEntry.firstKeptEntryId`。

## 18.2 症状索引

| 症状/问题 | 先看 | 章节 |
|---|---|---|
| 提示词里没有我的工具说明 | `promptSnippet` 是否定义、工具是否 active、是否被 `prepareLoadout` 隐藏 | 5.2、8.7 |
| 技能/AGENTS.md 没生效 | `resource-loader.ts` 的发现顺序与 `formatSkillsForPrompt` 的可读书写工具判定 | 5.5、14.4 |
| 缓存每轮都未命中 | `compat.supportsMidConvoSystemMessages`、是否用了 `forceSystemPrompt`、段变化频率 | 5.6、11.5 |
| 上下文莫名溢出 | 自定义工具是否截断（50 KB/2000 行）、`nestedCalls` 是否无界 | 8.3、8.8 |
| 压缩行为怪 | `prepareCompaction` 的切点规则、`firstKeptEntryId`、`modelOverrides` 合并顺序 | 7.3、7.7 |
| `Escape` 停不下某件事 | `AgentSession.abort()` 聚合的四类工作；扩展里是否忽略了 `ctx.signal` | 3.7、9.5 |
| 工具调用顺序/结果错位 | `executeToolCallsParallel` 的三段式与 `withFileMutationQueue` | 3.6、8.4 |
| 扩展没被加载 | rank 表与同名 first-wins、项目信任、`pi config` 状态 | 9.2、14.3 |
| 扩展 handler 抛错后行为 | `emitToolCall`（无 try/catch）vs 其他链式事件（catch 后继续） | 9.4 |
| 重载后状态丢失 | `session_shutdown` 与 `session_start` 的重建职责、`withSession` 的旧闭包 | 9.6 |
| MCP 工具不可调用 | exposure 值、codemode/tool_search 是否活动、`autoEnableCodemode` | 12.4 |
| 脚本 OOM 或超时 | 256 MB VM、16 Mi 输出、worker 池与 60 秒等待 | 12.6 |
| TUI 破版/残色 | 每行宽度量、每行重贴样式、`invalidate()` 是否清缓存 | 13.2、13.6 |
| 交互模式渲染卡顿 | 16 ms 合并、full redraw 触发条件、流式事件里的慢活 | 13.3、17.4 |
| RPC 客户端丢消息 | 按 LF 分帧（不是 `readline`）、订阅早于 prompt、等 `agent_settled` | 15.2、15.3 |
| SDK 注入的历史不生效 | `SessionManager` 是权威，赋值 `state.messages` 无效 | 15.5 |
| 模型列表缺项 | 目录 overlay 的 `lastModified > generatedAt`、`enabledModels` 作用域、刷新契约差异 | 4.5、11.3 |
| 凭证在 A shell 可用 B 不可用 | 存储类型不匹配不回落 env、`AuthContext` 注入、`$ENV`/`!cmd` 在配置层 | 4.6 |

## 18.3 端到端追一个 bug

以"扩展改了 `tool_call.input` 但工具拿到的是原值"为例，走一遍标准流程：

```bash
# 1. 最小复现：不用交互模式，直接看事件与结果
pi --mode json -e ./repro.ts "让它跑一条 bash" 2>/dev/null | jq -c 'select(.type=="tool_execution_start" or .type=="tool_call")'

# 2. 定位处理器是否被调用、调用顺序对不对
grep -n "emitToolCall" -A 20 packages/coding-agent/src/core/extensions/runner.ts

# 3. 看工具参数从哪儿取：预检阶段还是执行阶段
grep -n "prepareToolCall" -A 30 packages/agent/src/agent-loop.ts

# 4. 需要断点级证据时，在扩展里打印 event.input 与 execute 收到的 params
```

这套流程的可移植性在于：Pi 的四层结构（分发 → 循环 → 注册表 → 实现）在每一层都有**唯一入口函数**，所以"症状 → 层 → 函数 → 代码"是三步可达的。追 bug 时优先用 `--mode json`（有完整事件、无 TUI 干扰）而不是交互模式；要驱动状态变化用 `--mode rpc`；完全不涉及 UI 时写一次性脚本走 SDK。测试落点按 17.3 走 faux provider，避免消耗真实 token。

## 18.4 进阶项目

按"要串起的章节数"排序，每个都能独立验证：

| 项目 | 用到的机制 | 验收方式 |
|---|---|---|
| 权限层：路径白名单 + 命令审批 + 审计到会话 | `tool_call`（gate + mutate）、`annotations`、`appendEntry`、`registerEntryRenderer` | 10.8 清单全过；审计条目在 `/tree` 后仍属正确分支 |
| 结构化输出 agent：终止于一次带 schema 的工具调用 | `outputSchema`/`structuredContent`、`terminate`、faux provider 测试 | `pi --mode json` 里最后一条是工具结果而非助手文本 |
| 新提供商接入（含自定义协议流式） | `createProvider`、`streamSimple` 六条约束、`compat`、`refreshModels` | 11.6 的测试矩阵逐项通过；借用 `packages/ai/test` 套件 |
| 成本路由器：按任务复杂度选模型并保住缓存 | `registerVirtualModel`、`reason` 四值、`state` 分支语义 | 跨物理模型的 assistant `model` 字段与页脚显示一致；无谓的缓存未命中为零 |
| IDE 客户端 | RPC 帧与 33 条命令、`disposition`、扩展 UI 三档 | 能显示流式文本、工具状态、并在扩展请求确认时弹出对话框 |
| 计划模式（Pi 刻意不内置的功能） | `registerFlag`、`registerShortcut`、`setStatus`/`setWidget`、`sendMessage`、`setActiveTools` | 参考 `examples/extensions/plan-mode/`，自己实现一遍才算读懂 |
| 远程执行后端 | `operations` 接口、`createLocalBashOperations()` 复用、`spawnHook` | 内置工具在远端跑，本地只有 Pi 与凭证（Gondolin 模式） |
| 观测扩展：把 Pi 自己的性能画出来 | `model_select`、`message_update` 累计 usage、`registerEntryRenderer` | 参考 `.pi/extensions/tps.ts` 与 `redraws.ts` |

## 18.5 跟上演进

三个信息源，按更新频率排序：

1. **包的 `CHANGELOG.md`**（`packages/*/CHANGELOG.md`，`## [Unreleased]` 下有 `Added/Changed/Fixed/Removed/Breaking Changes`）。Pi 迭代快，任何"本教程说 X 这样"的结论若与新版本行为不符，先查这里（写作约定见 README 的正文约定）。
2. **RFCs**：<https://rfc.earendil.com/keyword/pi/>，长期计划在此，也是判断"某功能是设计还是偶然"的依据。
3. **文档站与 docs 目录**：`https://pi.dev/docs/latest/<slug>` 与仓库 `packages/coding-agent/docs/*.md` 同源（分工见前言）。其中 `models-configuration.md` 由 `npm run generate:references` 从 TypeScript 源生成（文件头注明 "Do not edit it manually"），`models` / `settings` / `keybindings` / `theme` 四份 JSON Schema 由 `packages/coding-agent/scripts/generate-schemas.ts` 从同一批 TypeBox 定义发布——这几处是**不会漂移**的事实来源。其余文档页是手写的，因此扩展 API 有疑义时的权威顺序是：`core/extensions/types.ts` > 生成的 schema 与 `models-configuration.md` > 手写文档页 > 本教程。

维护本教程的建议：每章末尾的"延伸阅读"已经列出该章的事实来源；Pi 升级后只需重跑这些来源的比对（尤其是 `settings-defaults.ts` 的默认值、`truncate.ts` 的限额、`codemode` 的常量、`rpc-types.ts` 的命令联合），不必全书重读。

## 小结

- 内圈读契约与格式（约 3000 行）就能建立系统边界模型；中圈先读四个编排文件（约 3200 行）掌握时序；外圈按需读，`agent-session.ts` 永远不要通读。
- 行为异常几乎都能在内圈的类型定义里找到解释（`sections`、`firstKeptEntryId`、`details`、`exposure`）。
- 追 bug 的标准顺序是：`--mode json` 复现 → 症状索引定位层 → 该层的唯一入口函数 → 加钩子取证 → faux provider 固化成测试。
- 八个进阶项目各覆盖一组章节，其中"权限层、新提供商、成本路由器"三项能检验你是否真的理解了第 5、7、11 章。
- 事实来源优先级：源码与生成的参考页 > changelog > 本教程。

## 练习

1. 用 18.1 的内圈清单在两天内读完五个文件，然后不看资料画出"一次带两次工具调用的回合"的完整时序。验收标准：与第 2 章与第 3 章的图对照，事件名与顺序全对。
2. 从 18.2 里任选三行，验证症状索引仍然有效（构造反例或复现现象）。验收标准：给出文件与行号级证据。
3. 完成"权限层"项目并写成 Pi 包（14.6）。验收标准：`pi install ./pkg` 后在两个不同项目里表现出不同策略（项目信任 + `ctx.isProjectTrusted()`）。
4. 跑一次 `cd packages/coding-agent && npm run generate:references`，再看 `docs/models-configuration.md` 与 `scripts/generate-schemas.ts` 的输出。验收标准：说出哪一份文档与哪四份 schema 是从 TypeScript 定义生成的，并解释为什么 `settings.md` 的**默认值**仍然要以 `core/settings-defaults.ts` 为准（17.1 的命令链）。

## 延伸阅读

- 第 1 章（地图）、第 2 章（动态路径）、第 17 章（工程门禁与贡献流程）
- 附录 A（事件与钩子总表）、附录 B（扩展 API 速查）、附录 C（常量与设置速查）
- `packages/coding-agent/examples/`（79 个扩展示例条目 + 14 个 SDK 示例）与 `packages/coding-agent/.pi/skills/`（Pi 自己的工作流资产）

---

→ 下一章：[附录 A 事件与钩子总表](appendix-a-events.md) ｜ [返回目录](README.md)
