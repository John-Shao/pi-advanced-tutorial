<p align="center">
  <img src="assets/pi-logo.svg" alt="Pi logo" width="96">
</p>

# Pi 高级教程

> 内部结构、工作原理、定制与扩展开发。面向已经会日常使用 Pi、现在要改它或接它的人。

Pi 是一个运行在终端里的可扩展智能体运行框架（agent harness），官方定位是"最小而可扩展"：内核只提供循环、模型抽象、会话与工具调度，其余能力以扩展形式存在。本教程讲这套机制的实现层——**代码放在哪里、行为为什么是这个顺序、你能在哪些边界上替换它**。

内容同时取材于两处一手资料，每个技术结论都标注出处：

- 官方文档：<https://pi.dev/docs/latest>（与仓库 `packages/coding-agent/docs/` 同源）
- 源码：Pi monorepo，版本基线 `@earendil-works/pi-coding-agent` **1.4.0**，commit **`f09d01b8`**（2026-10-30）

## 适用读者

- 用过 Pi：会启动、会提示、会用 `/model`、`/settings`、`/compact`，读过《Pi 入门教程》或官方 Quickstart。
- 能读 TypeScript。第 3、4、8、9 章需要跟着源码读；第 15 章含 Python 客户端示例。
- 目标之一是下列四项之一：写出可用扩展、接入新的模型服务、把 Pi 嵌进别的程序、在源码里定位并修复问题。

不重复入门教程的内容（安装、首次登录、终端基础操作、界面步骤），需要时给文字引用。

## 目录

### 第一部分　内部结构与工作原理

| 章节 | 内容 | 关键结论 |
|---|---|---|
| [前言：这份教程怎么用](00-preface.md) | 定位、事实来源、版本基线、阅读路线、正文约定 | 怎么读这本教程 |
| [第 1 章　架构总览](01-architecture-overview.md) | 三条塑形约束、11 个包与依赖方向、`coding-agent` 内部地图、四种运行形态 | 需求该落在哪一层 |
| [第 2 章　一次请求的完整生命周期](02-request-lifecycle.md) | 五段生命周期、输入处理顺序、请求装配、回合与工具、终态判定 | `agent_settled` 才是空闲信号 |
| [第 3 章　智能体循环内核](03-agent-loop-internals.md) | `Agent` 与 `AgentSession` 的分工、双层循环、队列、并行执行、abort 链、两层重试 | 扩展靠覆盖循环钩子字段生效 |
| [第 4 章　pi-ai：统一模型层](04-pi-ai-model-layer.md) | `Model`/`Provider`/`Api`、`stream` 与 `streamSimple`、12 个流事件、目录生成与运行时 overlay、凭证解析顺序 | 提示词与工具住在 transcript 里 |
| [第 5 章　上下文装配与系统提示词](05-context-assembly.md) | 段结构与合成规则、段补丁与缓存、上下文文件与技能、缓存预热经济学、钩子所有权矩阵 | 改段保住前缀，整段替换必未命中 |
| [第 6 章　会话树与存储](06-session-tree-and-storage.md) | JSONL entry 树、三段投影管线、系统消息作为提示词状态、分支与多根、扩展持久化选型 | 历史是树，上下文是投影 |
| [第 7 章　压缩与分支摘要](07-compaction-and-branch-summaries.md) | 触发公式与四处检查、切点与拆分用户消息段、摘要格式、溢出恢复落盘顺序、自定义摘要 | 压缩改未来，不改历史 |
| [第 8 章　工具系统内参](08-tool-system.md) | 工具的四重身份、内置工具与 `details`、截断、文件变更队列、进程树终止、`exposure` 五件套、嵌套调用 | 并行执行下 `details` 与队列是契约 |

### 第二部分　定制与扩展开发

| 章节 | 内容 | 关键结论 |
|---|---|---|
| [第 9 章　扩展运行时机制](09-extension-runtime.md) | jiti 加载与虚拟模块、发现优先级与信任门、两种分发通道、六种组合语义与错误策略、三个 context 形状、会话替换陷阱 | `tool_call` 抛错即阻塞 |
| [第 10 章　扩展开发实战](10-extension-development.md) | 开发回环、权限门、状态化工具、命令与界面、上下文注入、模型触发热重载、外部状态对齐、自检清单 | 六种可复制的模式 |
| [第 11 章　模型服务接入与虚拟模型](11-provider-integration.md) | 选型阶梯、`registerProvider` 两形态、`refreshModels` 与 publish、`compat` 与缓存、自定义 streaming 六条约束、路由模型 | 能力开关决定缓存行为 |
| [第 12 章　MCP 与 Codemode](12-mcp-and-codemode.md) | 三件套分工、传输与限额、连接与等待策略、暴露策略、QuickJS 沙箱与全局、BM25 检索 | 上百个工具压进一个可用上下文 |
| [第 13 章　TUI 与主题](13-tui-and-themes.md) | Component 契约、差分渲染、输入与焦点与 IME、图像协议与原生模块、主题 token 与颜色模型 | 渲染在关键路径上，宽度要量 |
| [第 14 章　资源定制与 Pi 包分发](14-resources-and-packaging.md) | 最轻机制选型、目录与命名、优先级与碰撞、技能与模板规范、包结构与筛选、依赖与身份、changelog | 宿主包走 `peerDependencies` |

### 第三部分　集成与工程

| 章节 | 内容 | 关键结论 |
|---|---|---|
| [第 15 章　可编程接口：print、JSON、RPC、SDK](15-programmatic-interfaces.md) | 边界选择、print/JSON 退出码与重建规则、RPC 帧与 33 条命令、扩展 UI 三档、SDK 组合根与替换、`RpcClient` | 分帧只按 LF；订阅早于提示 |
| [第 16 章　周边运行时：chord、durable、env、telemetry、evals](16-peripheral-runtimes.md) | 依赖分界、chord 的对象与复制状态、`pi-durable` 的先提交后展示、`pi-env` 的远程执行、telemetry 与 evals | monorepo 里有两条产品线 |
| [第 17 章　工程实践：调试、测试、性能与贡献](17-engineering-practice.md) | 开发回环与仓库语法约束、诊断工具箱、测试分层与 faux provider、性能阈值、隔离与信任边界、贡献与发布链条 | 没有内置权限系统，边界要靠环境 |
| [第 18 章　源码阅读地图与进阶路线](18-source-map-and-roadmap.md) | 三圈阅读法、症状索引、端到端追 bug、八个进阶项目、跟上演进 | 先读契约，再倒推编排 |

### 附录

| 附录 | 内容 |
|---|---|
| [附录 A　事件与钩子总表](appendix-a-events.md) | 41 个扩展事件按阶段分组，标注语义类别、可返回什么、关键载荷、代码出处 |
| [附录 B　扩展 API 与上下文速查](appendix-b-extension-api.md) | `pi.*` 全方法、`ctx` 三种形状、工具定义字段、渲染与 UI 方法 |
| [附录 C　数值、限额与配置速查](appendix-c-reference.md) | 默认值与限额（含出处文件）、设置键分组、环境变量、RPC 命令分组、entry 类型 |
| [附录 D　术语对照](appendix-d-glossary.md) | 全书术语的中英文对照 |

## 阅读路线

四条推荐路径（改行为 / 接模型 / 做集成 / 读源码）列在[前言：这份教程怎么用](00-preface.md)的"阅读路线"一节，本处不重复，以免两处不同步。每章都可独立阅读：开头给出学习目标与对应官方文档，结尾有小结、带验收标准的练习与延伸阅读。

## 正文约定（摘要）

- 每章标题下方有"上一章"链接、文末有"下一章"链接（顺序：前言 → 第 1 至 18 章 → 附录 A 至 D），两者都附返回目录链接。
- 命令可直接复制执行；输出示例标注"示例输出"；数值一律给实际数字并注明出处文件。
- "源码位置"为仓库相对路径加行号，链接指向本机 checkout；行号取自基线 commit，代码移动后以符号名为准。
- 图表：Mermaid 为主（GitHub 与 VS Code 可渲染），总体架构图另有 `assets/architecture-layers.svg`；终端截图来自 Pi 官方仓库，界面随版本变化。
- 术语中英对照见附录 D；跨章引用写作"见第 N 章 / 见 N.x 节"。

详见[前言：这份教程怎么用](00-preface.md)的完整约定与事实来源说明。

## 素材来源

| 文件 | 来源 |
|---|---|
| `assets/interactive-mode.png`、`tree-view.png`、`doom-extension.png` | Pi 仓库 `packages/coding-agent/docs/images/` |
| `assets/architecture-layers.svg` | 本教程绘制，依赖边取自各包 `package.json` |
| `assets/pi-logo.svg` | pi.dev |

## 许可与致谢

本教程基于 Pi 官方文档（MIT 许可）与 Pi 源码整理、改写与补充，示例命令与配置均按仓库代码核对。终端截图与 Pi 标志来自 Pi 官方仓库与 pi.dev，仅用于介绍 Pi。Pi 由 Earendil Inc. 及社区开发，项目主页 <https://pi.dev>。
