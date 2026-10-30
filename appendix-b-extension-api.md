# 附录 B　扩展 API 与上下文速查

← 上一章：[附录 A 事件与钩子总表](appendix-a-events.md) ｜ [返回目录](README.md)

签名事实来源是 [packages/coding-agent/src/core/extensions/types.ts](file:///D:/workspace/harness/earendil/pi/packages/coding-agent/src/core/extensions/types.ts)（2314 行）与 [Extensions 文档](https://pi.dev/docs/latest/extensions)。本附录按"写扩展时的查找顺序"组织：先骨架，再注册方法，再 `ctx`，再工具定义，再 UI，最后可导入的辅助函数。

## B.1 骨架

```typescript
import { Type } from "typebox";
import { StringEnum } from "@earendil-works/pi-ai";
import { defineTool, isToolCallEventType, type ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.on("session_start", async (_event, ctx) => { /* 观察、启动会话级资源 */ });
  pi.on("tool_call", async (event, ctx) => { /* 门：block / 原地改 input */ });
  pi.registerCommand("name", { description: "...", handler: async (args, ctx) => {} });
  pi.registerShortcut("ctrl+shift+p", { description: "...", handler: async (ctx) => {} });
  pi.registerFlag("plan", { description: "...", type: "boolean", default: false });
  pi.registerTool(defineTool({ name: "my_tool", /* ... 见 B.4 */ }));
}
```

工厂可以是 `async`（Pi 会 await，因此早于 `session_start` 与 provider flush）。**不要在工厂里启动后台资源**（可能根本不开会话）。三种形态：单文件、`<dir>/index.ts`、带 `package.json`（`pi.extensions` 入口 + `dependencies` + `npm install`）。

## B.2 `pi.*` 注册与动作

| 方法 | 用途 | 关键语义 |
|---|---|---|
| `on(event, handler)` | 订阅 | 返回退订函数；加载顺序→注册顺序；不影响在途派发 |
| `registerTool(definition)` | 注册工具 | 加载期与运行期都可；新工具当场出现在 `getAllTools()`，无需 `/reload`。注册即激活，除非 `defaultActive: false` 或 exposure 非 `direct`/`model-only` |
| `registerCommand(name, options)` | `/name` | 可带 `getArgumentCompletions`（可返回 Promise）。同名多扩展全部保留并加数字后缀（`/review:1`） |
| `getCommands()` | 列出可调用命令 | 顺序：扩展 → 模板 → 技能（`skill:<name>`）；含 `source` 与 `sourceInfo`（权威来源字段）；**不含**内置交互命令（那份清单见 [Slash Commands](https://pi.dev/docs/latest/slash-commands)） |
| `registerShortcut(shortcut, options)` | 键位 | 格式见 [keybindings 文档](https://pi.dev/docs/latest/keybindings) |
| `registerFlag(name, options)` | CLI flag | `type: "boolean" \| "string"`；工厂里 `getFlag()` 返回默认值，取值要在 `session_start` 之后 |
| `getFlag(name)` | 读 flag | — |
| `sendMessage(message, options)` | 注入 custom 消息（**进模型**） | `deliverAs`: `steer`（默认）\| `followUp` \| `nextTurn`；`triggerTurn: true` 空闲时立即起一轮（对 `nextTurn` 无效）；流式中 `triggerTurn: false` 则在本回合后追加 |
| `sendUserMessage(content, options)` | 注入 user 消息 | 流式中必须给 `deliverAs`，否则报扩展错误。`expandPromptTemplates: true` 才走命令派发与技能/模板展开 |
| `appendEntry(customType, data?)` | 持久扩展数据（**不进模型**） | 常配 `registerEntryRenderer` |
| `setSessionName(name)` / `getSessionName()` | 显示名 | 触发 `session_info_changed` |
| `setLabel(entryId, label)` | 书签 | `undefined` 清除；`/tree` 里可见，`getLabel(id)` 读 |
| `registerMessageRenderer(customType, renderer)` | custom 消息渲染 | 消息进模型 |
| `registerEntryRenderer(customType, renderer)` | custom entry 渲染 | 不进模型 |
| `registerMarkdownTransformer(fn)` | 改写用户/助手/思考块的 Markdown | 显示层，按加载顺序链式；抛错则保留到此前结果并继续；原文与会话不变 |
| `registerToolRenderer(resolver)` | 为任意工具名选择渲染器（含尚未注册的工具） | 接收 `(toolName, next)`，`next() ?? mine` 只填缺 |
| `registerProvider(name, config)` / `registerProvider(provider)` | 注册或覆盖提供商 | 工厂内排队、之后立即生效；给 `models` 会替换该 provider 三种操作的全部模型 |
| `unregisterProvider(name)` | 移除并恢复被覆盖的内置模型 | 不需 `/reload` |
| `registerVirtualModel(model)` / `unregisterVirtualModel(provider, id)` | 路由模型 | 重复注册同 provider+id 即替换；`unregisterProvider` 不会移除它 |
| `registerMcpServer(name, config)` / `unregisterMcpServer(name)` / `getMcpServers()` | 会话级 MCP | 注册不持久，每次加载要重注册；`unregister` 关连接并使工具不可达 |
| `getActiveTools()` / `getAllTools()` / `setActiveTools(names)` | 工具集合 | `getAllTools()` 返回 `name`/`description`/`parameters`/`promptGuidelines`/`exposure`/`sourceInfo`，以及设置过的 `namespace`/`annotations`/`reachesTools`。`setActiveTools` 忽略未知与 `hidden` |
| `setModel(model)` | 换模型 | 记入会话历史并在 resume 时恢复，**不改** `defaultProvider`/`defaultModel`；无凭证返回 `false` |
| `getThinkingLevel()` / `setThinkingLevel(level)` | 思考等级 | `off`\|`minimal`\|`low`\|`medium`\|`high`\|`xhigh`\|`max`；会钳制，触发 `thinking_level_select` |
| `exec(command, args, options?)` | 无 shell 执行 | 返回 `{ stdout, stderr, code, killed, truncated }`；`signal`/`timeout`/`cwd`/`maxBuffer`（默认 256 MiB）；忽略 SIGTERM 者 5 秒后 SIGKILL |
| `events` | 扩展间事件总线 | `events.on(name, fn)` / `events.emit(name, data)` |
| `getSettings()` | 有效设置副本 | 已合并全局+项目+覆写 |

## B.3 `ctx` 的三种形状

| 字段/方法 | 事件 | 命令 | 工具 | 说明 |
|---|---|---|---|---|
| `ui` | 是 | 是 | 是 | 见 B.5 |
| `mode`: `tui`\|`rpc`\|`json`\|`print` | 是 | 是 | 是 | TUI 专属能力前先判 `mode === "tui"` |
| `hasUI` | 是 | 是 | 是 | TUI 与 RPC 为真；print/JSON 为假 |
| `cwd` | 是 | 是 | 是 | 拼项目配置用 `CONFIG_DIR_NAME`，别硬编码 `.pi` |
| `isProjectTrusted()` | 是 | 是 | 是 | 含临时信任与 CLI 覆写 |
| `sessionManager`（只读） | 是 | 是 | 是 | `getCwd`、`getSessionDir`、`getSessionId`、`getSessionFile`、`getSessionName`、`getHeader`、`getEntries`、`getEntry`、`getBranch`、`getTree`、`getLeafId`、`getLeafEntry`、`getLabel`、`buildContextEntries`、`buildSessionProjection` |
| `modelRegistry` / `model` / `thinkingLevel` / `scopedModels` | 是 | 是 | 是 | `streamSimple`/`complete`/`classify`/`generateImages`/`find`/`findOfType` 都从这里走；`getProviderAuth(id)` 不需要已加载模型；`scopedModels` 与 `/scoped-models` 同源 |
| `signal` | 活动回合 | 通常 `undefined` | 有 | 传给 `fetch`、模型调用、任何接受 `AbortSignal` 的 helper |
| `isIdle()` / `abort()` / `hasPendingMessages()` | 是 | 是 | 是 | `isIdle()` 在重试、自动压缩重试、排队续跑期间为假 |
| `shutdown()` | 是 | 是 | 是 | 各模式延后时机不同（9.5 节）；退出前会派发 `session_shutdown` |
| `getContextUsage()` / `compact()` / `getSystemPrompt()` | 是 | 是 | 是 | `compact()` 不等待完成，用 `onComplete`/`onError` |
| `getSystemPromptOptions()` | — | 是 | — | 提示词构建的基础输入（可能含完整上下文文件内容，属敏感数据） |
| `waitForIdle()` | — | 是 | — | 等待含重试与续跑的完全落定 |
| `newSession()` / `fork()` / `switchSession()` / `navigateTree()` / `reload()` | — | 是 | — | 只在命令里可用（事件里调用会死锁）；`withSession` 见 9.6 的四条约束 |
| `tools` / `executeTool()` | — | — | 是 | `ExtensionToolContext` 专属；嵌套调用语义见 8.8 |

## B.4 工具定义

| 字段 | 必填 | 说明 |
|---|---|---|
| `name` / `label` / `description` | 是 | `description` 给模型看 |
| `parameters` | 是 | TypeBox schema；字符串枚举用 `StringEnum`（产出各提供商都接受的 `enum`） |
| `execute(toolCallId, params, signal, onUpdate, ctx)` | 是 | 返回 `{ content, details, structuredContent?, usage?, isError?, terminate? }` |
| `promptSnippet` | 否 | `<tools>` 段的一行；缺省则工具**不进**该段 |
| `promptGuidelines` | 否 | 追加到 `<rules>` 的条目，仅当工具活动；**每条必须点名工具** |
| `prepareArguments(args)` | 否 | 校验前运行；用于兼容旧会话里的参数形状。保持公开 schema 严格，不要为兼容加废弃字段 |
| `outputSchema` | 否 | 声明后脚本侧拿 `structuredContent`；模型侧仍拿 `content` |
| `executionMode` | 否 | `"sequential"` 时该批整体串行（默认并行） |
| `defaultActive` | 否 | `false` 则注册但不活动 |
| `exposure` | 否 | `direct`（默认）\| `model-only` \| `codemode` \| `deferred` \| `hidden` |
| `namespace` / `annotations` / `reachesTools` / `prepareLoadout` | 否 | 分组、MCP 式提示、可达集、声明覆写（8.7） |
| `renderShell` / `renderCall` / `renderResult` | 否 | `renderShell: "self"` 时自己负责边框、padding、背景 |
| `constrainedSampling` | 否 | 请求提供商侧受约束采样 |

结果契约：每个结果（含 `onUpdate` 的部分结果）都要有 `content` 与 `details` 值，且 `details` 类型在同一工具内一致。报错方式：抛错，或返回 `isError: true`（后者保留 `details` 与 `structuredContent`）。`terminate: true` 只在整批都终止时生效。嵌套模型调用的 `usage` 要返回在 `usage` 字段。用 `defineTool()` 保住赋给变量后的参数类型推断。

## B.5 `ctx.ui` 与 RPC 支持度

| 组 | 方法 | RPC |
|---|---|---|
| 对话框 | `select(title, options, {timeout?, signal?})`、`confirm(title, message, opts?)`、`input(label, placeholder?, opts?)`、`editor(title, prefill?, opts?)` | 走 JSONL 子协议；超时/取消：前三者 `undefined`，`confirm` 为 `false` |
| 通知与状态 | `notify(text, "info"\|"warning"\|"error")`、`setStatus(key, text)`、`setTitle`、`setEditorText` / `getEditorText`、`pasteToEditor` | 通知有效；`getEditorText()` 返回 `""` |
| 工作指示 | `setWorkingMessage(text?)`、`setWorkingVisible(bool)`、`setWorkingIndicator({frames, intervalMs}?)` | no-op |
| 停靠内容 | `setWidget(key, lines \| factory, { placement?: "aboveEditor"\|"belowEditor" })`、`setFooter`、`setHeader`、`setHiddenThinkingLabel` | `setWidget` 仅接受 `string[]`，工厂被忽略；其余 no-op |
| 编辑器与补全 | `setEditorComponent(factory)`、`getEditorComponent()`、`addAutocompleteProvider((current) => provider)` | no-op / 返回 `undefined` |
| 组件级界面 | `custom<T>(factory, { overlay?, overlayOptions?, onHandle?, blocked? })`、`onTerminalInput(fn)` | `custom()` 返回 `undefined`；`onTerminalInput` no-op |
| 主题 | `theme.fg/bg/bold/italic/underline/strikethrough/style`、`colors`、`appearance`、`setTheme`、`getTheme`、`getAllThemes` | `setTheme` 返回 `{ success: false }`；`getAllThemes()` 返回 `[]` |
| 工具展开 | `getToolsExpanded()` / `setToolsExpanded(bool)` | 返回 `false` / no-op |

`addAutocompleteProvider` 的写法是"包装"：自定义触发用 `triggerCharacters`，不匹配时把 `getSuggestions` / `applyCompletion` / `shouldTriggerFileCompletion` 委托给 `current`。`custom()` 的四条纪律见 13.2 与 2.1 的示例（`render(width)` 行不超宽、`invalidate()` + `tui.requestRender()`、`keybindings.matches(data, "tui.select.*")`、只能通过 `done()` 结束）。等待用户输入的 inline 组件传 `blocked: { kind: "question"\|"permission"\|"auth", message }`。

## B.6 可导入的辅助

| 名称 | 用途 |
|---|---|
| `defineTool` | 保留参数类型推断的工具定义 |
| `isToolCallEventType` / `isBashToolResult` | 事件与结果类型收窄 |
| `truncateHead` / `truncateTail` / `truncateLine` / `formatSize` / `DEFAULT_MAX_BYTES` / `DEFAULT_MAX_LINES` | 截断（8.3） |
| `withFileMutationQueue` | 同文件串行（8.4） |
| `createReadTool` / `createBashTool` / `createPowerShellTool` / `create*ToolDefinition` | 内置工具工厂与完整定义 |
| `createLocalBashOperations` / `createLocalShellOperations` / `createLocalPowerShellOperations` | 复用 Pi 的本地 shell 后端 |
| `SessionManager` | 会话列表与构造（`list`、`listAll`、`inMemory`） |
| `convertToLlm` / `serializeConversation` | 角色折叠 / 摘要前的文本化（7.4） |
| `CONFIG_DIR_NAME` | 项目配置目录名（可被 fork 改写） |
| `CustomEditor` | 自定义主编辑器的基类（13.4） |
| `keyHint` / `keyText` / `rawKeyHint` | 按当前键位配置格式化提示 |
| `highlightCode` / `getLanguageFromPath` | 自定义渲染里的语法高亮 |
| `getCurrentSystemMessage` / `getCurrentSystemPrompt` / `getCurrentTools`（`@earendil-works/pi-ai`） | 从 transcript 反推状态（4.7） |
| `StringEnum`（`@earendil-works/pi-ai`） | 提供商通用的字符串枚举 schema |

## 相关页面

- 第 9 章（加载、分发、错误策略、模式差异）、第 10 章（六类模式的可复制写法）
- 附录 A（事件与可返回值）、附录 C（限额与默认值）
- 官方 [Extensions](https://pi.dev/docs/latest/extensions) 的 Examples Reference 表（78 行示例分类）

---

→ 下一章：[附录 C 数值、限额与配置速查](appendix-c-reference.md) ｜ [返回目录](README.md)
