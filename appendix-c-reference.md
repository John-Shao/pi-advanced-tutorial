# 附录 C　数值、限额与配置速查

← 上一章：[附录 B 扩展 API 与上下文速查](appendix-b-extension-api.md) ｜ [返回目录](README.md)

所有数值取自基线 commit `f09d01b8` 的源码与文档，括号内是出处。设置键的完整语义见 [Settings](https://pi.dev/docs/latest/settings)；默认值以 [core/settings-defaults.ts](file:///D:/workspace/harness/earendil/pi/packages/coding-agent/src/core/settings-defaults.ts) 为准，校验器由 `generate-schemas.ts` 从同一批 TypeBox 定义发布为 `settings.schema.json`。

## C.1 默认值（设置键）

| 组 | 键 = 默认值 |
|---|---|
| 模型与思考 | `defaultProvider`/`defaultModel` = 自动 · `defaultThinkingLevel` = `medium` · `modelThinkingLevels` = 无 · `enabledModels` = 全部 · `hideThinkingBlock` = `false` · `showCacheMissNotices` = `false` · `cacheWarming` = `streaming` |
| 交互 | `steeringMode` = `one-at-a-time` · `followUpMode` = `one-at-a-time` · `externalEditor` = `$VISUAL`→`$EDITOR`→平台默认 · `doubleEscapeAction` = `tree` · `treeFilterMode` = `default` · `defaultProjectTrust` = `ask`（**仅 agent 目录可设**） |
| 工具 | `defaultTools` = `read`,`bash`,`edit`,`write` · `codemode.mode` = `on` · `codemode.inlineBudget` = `3000` |
| 会话与上下文 | `sessionDir` = agent 会话目录 · `sessionShareDestination` = `auto` · `compaction.enabled` = `true` · `compaction.reserveTokens` = `16384` · `compaction.keepRecentTokens` = `20000` · `branchSummary.reserveTokens` = `16384` · `branchSummary.skipPrompt` = `false` |
| 终端与显示 | `theme` = `system` · `quietStartup` = `false` · `tuiMode` = `fullscreen` · `fullscreenExitOutput` = `transcript` · `fullscreenScrollbar` = `auto` · `fullscreenCopyOnSelect` = `true` · `fullscreenWheelScrollLines` = `auto` · `editorPaddingX` = `0` · `outputPad` = `1` · `autocompleteMaxVisible` = `5` · `showHardwareCursor` = `false` · `terminal.showImages` = `true` · `terminal.imageWidthCells` = `60` · `terminal.clearOnShrink` = `false` · `terminal.showTerminalProgress` = `false` · `terminal.hyperlinks`/`images`/`trueColor` = `auto` · `images.autoResize` = `true`（上限 2000×2000） · `images.blockImages` = `false` · `markdown.codeBlockIndent` = `"  "` · `markdown.mermaid` = `streaming` |
| 网络与重试 | `transport` = `auto` · `httpIdleTimeoutMs` = `300000` · `websocketConnectTimeoutMs` = `15000` · `retry.enabled` = `true` · `retry.maxRetries` = `3` · `retry.baseDelayMs` = `2000` · `retry.maxAgentDelayMs` = `60000` · `retry.provider.maxRetries` = `0` · `retry.provider.timeoutMs` = 取 `httpIdleTimeoutMs` · `retry.provider.maxRetryDelayMs` = `60000` · `httpProxy` = 无（**仅 agent 目录可设**） |
| Shell | `shellPath` = 平台默认 · `shellCommandPrefix` = 无 · `npmCommand` = `["npm"]` |
| 资源 | `packages`/`extensions`/`skills`/`prompts`/`themes` = `[]` · `enableSkillCommands` = `true` |
| 更新与遥测 | `collapseChangelog` = `false` · `enableInstallTelemetry` = `true` · `enableAnalytics` = `false` · `warnings.anthropicExtraUsage` = `true` |

## C.2 代码里的常量与限额

| 主题 | 数值 | 出处 |
|---|---|---|
| **工具截断** | 2000 行 / 50 KB（先到者胜）；grep 单行 500 字符 | `core/tools/truncate.ts:11-13` |
| 工具默认上限 | grep `DEFAULT_LIMIT` 100 · find 1000 · ls 500 | `core/tools/{grep,find,ls}.ts` |
| bash 结构化输出 | 1 MiB（超出保留首尾各 512 KiB） | `core/tools/bash.ts:24` |
| bash 超时上限 | `MAX_TIMEOUT_MS = 2_147_483_647`；schema 无默认超时 | `core/tools/bash.ts:22` |
| 进程树终止 | bash：非 Windows 对进程组立即 SIGKILL（macOS 先 SIGSTOP）；Windows `taskkill /F /T` | `utils/shell.ts:220` |
| `pi.exec()` | `maxBuffer` 默认 256 MiB；忽略 SIGTERM 者 5 秒后 SIGKILL | `core/exec.ts` |
| 嵌套调用记录 | 最多 256 条；单调用参数 8 KiB；单结果参数合计 32 KiB；超出置 `complete: false` | `core/nested-tool-calls.ts:27-29` |
| **压缩** | `reserveTokens` 16384 / `keepRecentTokens` 20000；token 估算 = 字符/4，图片按 4800 字符 | `core/compaction/compaction.ts:126,276` |
| 摘要序列化 | 工具结果截到 2000 字符；thinking 块省略 | `core/compaction/utils.ts` |
| 溢出恢复 | 每轮运行最多一次 | `core/agent-session.ts:3100` |
| **缓存预热** | 期望节省 ≥ $0.05 才预热；空闲续接概率 0.15；在 TTL 的 90% 处发（TTL ≤ 10 秒不预热）；流式上限 1 小时、空闲上限 30 分钟 | `core/cache-warmer.ts:15-29,386-395` |
| **重试** | 回合层：3 次、基础 2000 ms、指数 `base*2^(n-1)`、封顶 60000 ms。提供商层：默认 0 次；`retryProviderRequest` 指数 `0.5*2^i` 封顶 8 秒 | `settings-defaults.ts:18-26`、`ai/src/utils/retry.ts:146`、`ai/src/utils/provider-retry.ts:67` |
| **codemode** | VM 内存 256 MB；输出 16 Mi 字符 / 100000 条目；默认输出预算 10000 token；`store` 单值 262144 / 合计 1048576 字符；进程内 2 个 worker（`PI_CODEMODE_MAX_WORKERS`）、等不到 60 秒失败；单脚本并发模型调用 4；声明目录预算 3000 token；QuickJS 为 `quickjs-wasi@3.6.2` | `extensions/codemode/execute.ts:56-62`、`codemode/src/runtime/prelude-source.ts:34-43` |
| **MCP** | 单消息 16 MB；客户端请求 30 秒；配置 `timeout` 默认 60 秒；首个 prompt 只等 `direct`，上限 10 秒；结果文本 20 KB 后去中段；工具名 ≤ 64 字符（截断加 sha256 前 8 位）；协议版本 `2025-11-25`；`mcp.log` 超 5 MB 轮转；HTTP 408/429/5xx 重试两次 | `mcp/src/transports/transport.ts:3`、`mcp/src/client.ts:38`、`extensions/mcp/{runtime,tools,index}.ts` |
| 检索 | BM25 `k1 = 1.2`、`b = 0.75`；默认返回 8 | `extensions/tool-search/tool.ts:144` |
| **会话文件** | `CURRENT_SESSION_VERSION = 3`；entry id 为 8 位十六进制（碰撞重试 100 次）；会话 id 为 `uuidv7`；header 扫描上限 1 MiB；读缓冲 1 MiB | `core/session-manager.ts:41,264-284,604-607` |
| **模型目录** | 远端 overlay 刷新窗口 4 小时；默认目录基址 `https://pi.dev`；采用条件 `lastModified > 内置 generatedAt`；合并键 `type\0id` | `core/remote-catalog-provider.ts:14-16,32-71` |
| OAuth 刷新 | 提前 5 分钟刷新；超时 15 秒；在 `credentials.modify` 锁内落盘 | `ai/src/auth/resolve.ts:102-119` |
| **TUI** | 渲染合并 16 ms；单次写入分块 1 MiB；输入序列超时 50 ms、ESC 超时 10 ms（SSH 下 100 ms）；图像协议仅 `kitty`/`iterm2`；默认图片宽 60 列 | `tui/src/tui.ts:524`、`tui-main-screen.ts:12`、`stdin-buffer.ts:23-24`、`terminal-image.ts:7` |
| 内置工具集 | 默认进入提示词的 `selectedTools` = `read`,`bash`,`edit`,`write` | `core/system-prompt.ts:64` |

## C.3 环境变量

**Pi 进程配置**：`PI_CODING_AGENT_DIR`（默认 `~/.pi/agent`）、`PI_CODING_AGENT_SESSION_DIR`（被 `--session-dir` 覆写）、`PI_PACKAGE_DIR`、`PI_OFFLINE`、`PI_SKIP_VERSION_CHECK`、`PI_TELEMETRY`、`PI_CODEMODE_MAX_WORKERS`（默认 2）、`PI_CACHE_RETENTION`（`long`）、`PI_SHARE_VIEWER_URL`、`PI_RADIUS_GATEWAY`、`PI_CLEAR_ON_SHRINK`、`PI_HARDWARE_CURSOR`、`PI_CLIPBOARD_OSC52`、`PI_HYPERLINKS`（`1`/`0`/`auto`）、`PI_PROGRAM_STATUS`、`PI_IMAGE_PROTOCOL`（`kitty`/`iterm2`/`none`/`auto`）、`PI_TRUE_COLOR`、`PI_TUI_ESC_TIMEOUT`、`PI_TUI_WRITE_LOG`、`VISUAL`/`EDITOR`、`HTTP_PROXY`/`HTTPS_PROXY`。派生项目（fork 改名后前缀随名字变，见 1.1 与 15.6）：`piConfig.name`。

**进程标记**（子进程继承；经 SDK 嵌入时不自动设置）：`AI_AGENT=pi`、`PI_CODING_AGENT=true`。

**shell 工具会话环境**（注入 `bash`/`powershell` 工具，不注入用户 `!`/`!!`；`exposeSessionEnvironment: false` 时 Pi 主动清除继承值）：`PI_SESSION_ID`、`PI_SESSION_FILE`、`PI_PROVIDER`、`PI_MODEL`、`PI_REASONING_LEVEL`（`off`\|`minimal`\|`low`\|`medium`\|`high`\|`xhigh`\|`max`）。虚拟模型下 `PI_MODEL` 是**选择**的模型而非路由后的物理模型（11.7）。

**测试/门禁相关**：`PI_ALLOW_LOCKFILE_CHANGE=1`（允许提交 lockfile，17.6）。

## C.4 命令行要点

权威清单是 [Command Line](https://pi.dev/docs/latest/cli) 与 `pi --help`。日常高频：

| 用途 | 参数 |
|---|---|
| 模式 | `--mode json` \| `--mode rpc` \| 交互（默认，stdin/stdout 为 TTY 时）；`-p`/`--print` |
| 会话 | `--continue`、`--session-id`、`--name`/`-n`、`--no-session`、`--session-dir`、fork 相关校验见 `main.ts` |
| 模型 | `--provider`、`--model`、`--models`（作用域模式，minimatch `provider/modelId`）、`--list-models` |
| 工具 | `--tools`（`+name`/`-name` 为增量，纯名字为替换；`mcp__` 前缀才影响 MCP 工具）、`--exclude-tools`、`--no-tools`、`--no-builtin-tools`、`--no-mcp` |
| 资源 | `-e`/`--extension`、`--no-extensions`/`-ne`、`--no-skills`/`-ns`、`--no-prompt-templates`/`-np`、`--no-themes`、`--no-context-files`/`-nc` |
| 提示词 | `--system-prompt`、`--append-system-prompt`（可多次） |
| 信任 | `--approve` / `--no-approve`（非交互模式唯一的一次性决策手段） |
| 外观 | `--use-theme`、`--tui-mode` |
| 包 | `pi install`（`--local`/`-l`）、`pi remove`、`pi list`、`pi update --extensions`/`--self`、`pi config`（`--local`） |
| MCP | `pi mcp add`/`remove`/`list`/`login`/`logout`（**不加载扩展**，只看文件配置） |

## C.5 会话 entry 类型与消息角色

11 类 entry（[core/session-manager.ts:183](file:///D:/workspace/harness/earendil/pi/packages/coding-agent/src/core/session-manager.ts#L183)）：`message`、`thinking_level_change`、`model_change`、`usage`、`compaction`、`branch_summary`、`custom`、`custom_message`、`context_edit`、`label`、`session_info`；文件首行 `type: "session"` 是 header，不入树。

8 类角色（`AgentMessage` 联合，[docs/message-types.md](https://pi.dev/docs/latest/message-types)）：`system`、`user`、`assistant`、`toolResult`、`bashExecution`、`custom`、`branchSummary`、`compactionSummary`。`stopReason` 取值：`pending` \| `stop` \| `length` \| `toolUse` \| `error` \| `aborted` \| `deferred`（`pending` 不会持久化）。

`Usage`：`input`、`output`、`cacheRead`、`cacheWrite`、`cacheWrite1h?`、`reasoning?`（已含在 `output` 内）、`totalTokens`、`cost{input,output,cacheRead,cacheWrite,total,totalFromProvider?}`。

## C.6 RPC 命令（33）与事件

命令分组见[第 15.3 节](15-programmatic-interfaces.md)，逐命令字段见 [RPC Commands](https://pi.dev/docs/latest/rpc-commands)。命令联合定义在 [packages/coding-agent/src/modes/rpc/rpc-types.ts:20](file:///D:/workspace/harness/earendil/pi/packages/coding-agent/src/modes/rpc/rpc-types.ts#L20)（`RpcCommand` 的 33 个成员与该页条目数一致，两者可互相校验）。

JSON 与 RPC 共享的事件（`AgentSessionEvent` 的 wire 形态）：`agent_start`、`agent_end`（`messages`、`willRetry`）、`agent_settled`（`aborted`）、`turn_start`、`turn_end`、`message_start`、`message_update`（delta-only + `usage`）、`message_end`、`tool_execution_start`、`tool_execution_update`、`tool_execution_end`、`queue_update`、`entry_appended`、`session_info_changed`、`thinking_level_changed`、`compaction_start`、`compaction_end`、`auto_retry_start`、`auto_retry_end`、`summarization_retry_scheduled`、`summarization_retry_attempt_start`、`summarization_retry_finished`。RPC 专属：`bash_execution_update`、`extension_error`，以及 `extension_ui_request` / `extension_ui_response` 子协议。

## 相关页面

- 第 2 章（生命周期）与第 15 章（协议与终态）
- 附录 A（41 个扩展事件）、附录 B（`pi.*` 与 `ctx`）
- 第 17 章（这些数值如何被测试与门禁覆盖）

---

→ 下一章：[附录 D 术语对照](appendix-d-glossary.md) ｜ [返回目录](README.md)
