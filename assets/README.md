# `assets/` 素材说明

| 文件 | 来源 | 用途 | 注意 |
|---|---|---|---|
| `architecture-layers.svg` | 本教程绘制 | 第 1 章的分层与内部依赖图 | 依赖边与"依赖："文字取自各包 `package.json` 的 `dependencies`（基线 commit `f09d01b8`）。构建拓扑序取自根 `package.json` 的 `build` 脚本。Pi 版本变化后需按新 `package.json` 重绘 |
| `interactive-mode.png` | Pi 仓库 `packages/coding-agent/docs/images/interactive-mode.png` | 第 13 章的界面形态参照 | 官方截图，界面随版本变化 |
| `tree-view.png` | Pi 仓库 `packages/coding-agent/docs/images/tree-view.png` | 第 6 章的会话树视图参照 | 同上 |
| `doom-extension.png` | Pi 仓库 `packages/coding-agent/docs/images/doom-extension.png` | 第 13 章 overlay 能力的示例 | 官方示例扩展的截图 |
| `pi-logo.svg` | pi.dev | README 与前言的标识 | 仅用于标识 Pi |

正文中的其余图示都是 Mermaid 代码块（结构图、时序图、状态图、决策图），由 `.md` 文件直接渲染，不依赖本目录。

Pi 与 pi.dev 的素材版权归 Earendil Inc. 及社区所有，本目录内容仅用于介绍与教学。
