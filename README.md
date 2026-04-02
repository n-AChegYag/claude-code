# Claude Code 源码学习与初步解读

[English Version](./README_EN.md)

> [!IMPORTANT]
> **本项目仅用于学习、研究与技术分析。**
>
> **当前代码及其相关内容的所有权、著作权及其他一切权利，仍归 Anthropic 公司所有。**
>
> 本项目不主张、不转移，也不重新分配 Claude Code 原始代码或相关知识产权。

## 项目说明

本项目的主题是：**对 Claude Code 源码进行学习、观察与初步解读**。

更准确地说，这里并不是 Anthropic 官方公开维护的上游开发仓库，而是一个围绕 `@anthropic-ai/claude-code` 发布产物、恢复源码视图以及教程文档组织起来的学习型目录。

本项目希望完成的事情主要包括：

- 从整体上理解 Claude Code 的架构轮廓
- 学习其 CLI 启动链路与主程序装配方式
- 观察 QueryEngine、命令系统、工具系统、任务系统、设置系统等核心模块
- 初步拆解其 MCP、plugins、skills、权限系统与安全边界等扩展机制
- 以教程文档的方式沉淀阅读路径与源码理解

因此，这个项目更适合被理解为：

> 一个面向学习者的 Claude Code 源码学习与初步解读仓库。

## 研究与学习边界

需要明确几点边界：

1. 本项目用途仅限于**学习、研究、技术分析与文档讲解**。
2. 本项目并不是 Anthropic 官方源码仓库，也不代表官方实现文档。
3. 本项目中的代码相关权利仍归 **Anthropic** 所有。
4. 本仓库中的解读、拆解和教程，属于基于现有目录内容进行的学习性整理，不构成任何权利声明。

如果你的目标是：

- 学习 Claude Code 的整体结构
- 建立对关键模块职责的理解
- 跟随教程逐步阅读源码
- 把这里作为一个分析样本来研究终端代理型 CLI 的实现思路

那么这个项目是合适的。

如果你的目标是把这里当作 Anthropic 官方上游工程继续维护，则不应这样使用。

## 当前目录主要包含什么

当前目录主要由以下部分组成：

```text
.
├── cli.js
├── cli.js.map
├── package.json
├── sdk-tools.d.ts
├── README.md
├── README_EN.md
├── docs/
│   └── tutorial/
├── vendor/
└── recovered_source/
    └── src/
```

可以简单理解为：

- `cli.js`：打包后的 CLI 入口
- `cli.js.map`：source map，用于辅助恢复与定位源码结构
- `sdk-tools.d.ts`：Claude Code 工具接口类型定义
- `recovered_source/`：从发布产物中恢复出的源码视图
- `docs/tutorial/`：围绕源码阅读整理出的教程文档

## 本项目当前关注的源码主题

围绕 Claude Code 的学习与初步解读，目前主要关注这些主题：

- 启动流程：CLI 如何进入主程序
- 主循环：QueryEngine 如何组织多轮会话
- 输入处理：commands、hooks、附件输入如何汇入系统
- 工具系统：Read / Edit / Bash / Agent 等能力如何被组织与约束
- 持久状态：tasks、settings、session memory 如何支撑长期工作流
- 扩展能力：MCP、plugins、skills 如何接入主程序
- 安全边界：权限模式、路径保护、shell 安全与工作流边界如何协同工作

这些内容并不是要“复刻官方设计文档”，而是以当前可见目录为基础，建立一种适合学习者理解的结构化阅读方式。

## 教程文档入口

当前仓库已经整理出一套教程文档，作为源码学习的阅读路径。

### 中文教程

- [00 总览与阅读地图](./docs/tutorial/00-overview.md)
- [01 启动链路与主程序装配](./docs/tutorial/01-bootstrap-and-startup.md)
- [02 QueryEngine 与会话主循环](./docs/tutorial/02-query-engine-and-conversation-loop.md)
- [03 输入处理、命令系统与 Hooks](./docs/tutorial/03-input-commands-and-hooks.md)
- [04 工具系统与子代理执行](./docs/tutorial/04-tools-and-agent-execution.md)
- [05 任务、设置与会话记忆](./docs/tutorial/05-state-tasks-settings-and-memory.md)
- [06 MCP、插件与 Skills 深挖](./docs/tutorial/06-mcp-plugins-and-skills.md)
- [07 权限系统与安全边界深挖](./docs/tutorial/07-permissions-and-security-boundaries.md)
- [99 继续深挖的阅读路径](./docs/tutorial/99-reading-paths-and-next-steps.md)

### English tutorial

- [00 Overview and reading map](./docs/tutorial/00-overview_EN.md)
- [01 Bootstrap and startup](./docs/tutorial/01-bootstrap-and-startup_EN.md)
- [02 QueryEngine and conversation loop](./docs/tutorial/02-query-engine-and-conversation-loop_EN.md)
- [03 Input, commands, and hooks](./docs/tutorial/03-input-commands-and-hooks_EN.md)
- [04 Tools and agent execution](./docs/tutorial/04-tools-and-agent-execution_EN.md)
- [05 Tasks, settings, and session memory](./docs/tutorial/05-state-tasks-settings-and-memory_EN.md)
- [06 MCP, plugins, and skills](./docs/tutorial/06-mcp-plugins-and-skills_EN.md)
- [07 Permissions and security boundaries](./docs/tutorial/07-permissions-and-security-boundaries_EN.md)
- [99 Reading paths and next steps](./docs/tutorial/99-reading-paths-and-next-steps_EN.md)

## 本项目的价值

本项目的价值不在于替代官方源码仓库，而在于：

- 为学习 Claude Code 提供一个更容易进入的阅读入口
- 把分散的恢复源码整理成可以逐步理解的主题
- 帮助读者建立对关键模块职责、运行流程和系统边界的初步认识
- 为后续更深入的源码研究提供结构化笔记与教程基础

## 声明

再次声明：

- 本项目仅用于学习、研究与技术分析用途
- Claude Code 相关代码及相应权利仍归 **Anthropic** 公司所有
- 本项目中的教程、说明和初步解读，不构成对原始代码权利的主张，也不构成任何官方授权说明

## 结语

如果你希望把这个仓库当作一个**Claude Code 源码学习样本**来阅读，那么它是有价值的。

如果你希望通过它快速建立对 Claude Code 启动流程、主循环、工具体系、扩展机制与安全边界的初步理解，那么这里的教程和整理内容正是为这个目标服务的。
