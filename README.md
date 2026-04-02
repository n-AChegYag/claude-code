# Claude Code 源码学习与初步解读

[English Version](./README_EN.md)

> [!IMPORTANT]
> **本项目仅用于学习、研究与技术分析。**
>
> **Claude Code 相关代码及其一切权利仍归 Anthropic 公司所有。**
>
> 本项目不主张、不转移，也不重新分配 Claude Code 原始代码或相关知识产权。

## 项目定位

本项目是一个围绕 **Claude Code 源码学习与初步解读** 展开的学习型仓库。

它并不是 Anthropic 官方公开维护的上游开发仓库，而是基于当前可见目录内容，对 Claude Code 的发布产物、恢复源码视图与相关结构进行阅读、整理和教程化总结的研究型项目。

## 你可以在这里看到什么

当前仓库主要包含：

- `cli.js`：打包后的 CLI 入口
- `cli.js.map`：source map
- `sdk-tools.d.ts`：工具接口类型定义
- `recovered_source/`：恢复出的源码视图
- `docs/tutorial/`：围绕源码阅读整理的教程文档

## 这个项目主要关注什么

本项目目前主要关注 Claude Code 的这些主题：

- CLI 启动链路与主程序装配
- QueryEngine 与会话主循环
- 输入处理、命令系统与 hooks
- 工具系统与子代理执行
- tasks / settings / session memory 等持续状态系统
- MCP / plugins / skills 等扩展机制
- 权限系统与安全边界

## 教程入口

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

## 使用边界声明

请明确：

- 本项目仅用于学习、研究与技术分析
- 本项目不代表 Anthropic 官方实现文档
- Claude Code 代码及相应权利仍归 **Anthropic** 所有
- 本仓库中的说明、解读与教程不构成任何权利主张或官方授权说明

## 简短说明

如果你希望把这里当作一个 **Claude Code 源码学习样本** 来阅读，这个仓库是有价值的。

如果你希望通过更短路径建立对 Claude Code 启动流程、主循环、工具系统、扩展机制与安全边界的初步理解，那么这里的教程文档正是为这个目标服务的。
