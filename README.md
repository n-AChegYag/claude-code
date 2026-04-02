# Claude Code Package Research Notes

[English Version](./README_EN.md)

> [!IMPORTANT]
> **本仓库内容仅供学习、研究与技术分析使用。**
>
> **当前代码及其相关内容的所有权、著作权及其他一切权益均归 Anthropic 公司所有。**
>
> 本仓库不主张、不转移，也不重新分配 Claude Code 原始代码或相关知识产权。

## 概览

当前目录是 `@anthropic-ai/claude-code` 的一个**已打包发布产物分析目录**，并不是 Anthropic 官方公开维护的完整原始开发仓库。

从现有文件结构判断，这里主要用于：

- 研究 Claude Code 的发布包结构
- 分析 CLI 启动流程与模块组织
- 查阅工具接口与类型定义
- 理解其终端代理式工作模式
- 在恢复源码基础上编写学习教程与代码讲解文档

也就是说，这个仓库现在不仅是一个研究样本，也是一套围绕 Claude Code 架构的**教程型分析仓库**。

## 仓库定位

建议把这个项目理解成下面三者的结合：

1. **npm 发布产物研究样本**
2. **恢复源码结构分析目录**
3. **Claude Code 学习教程文档仓库**

如果你的目标是：

- 学习 Claude Code 的 CLI 启动方式
- 理解 QueryEngine、工具系统、命令系统、任务/设置/记忆系统
- 研究 MCP、插件、skills 这些扩展机制

那么这个仓库已经非常适合作为阅读材料。

如果你的目标是把这里当作 Anthropic 官方上游源码仓库继续维护，则不应这样理解它。

## 快速信息

| 项目 | 说明 |
| --- | --- |
| 包名 | `@anthropic-ai/claude-code` |
| 当前版本 | `2.1.88` |
| 运行环境 | Node.js `>=18.0.0` |
| 主入口 | `cli.js` |
| 调试映射 | `cli.js.map` |
| 工具类型定义 | `sdk-tools.d.ts` |
| 分析源码目录 | `recovered_source/` |
| 平台二进制 | `vendor/` |
| 教程目录 | `docs/tutorial/` |

## 目录结构

```text
.
├── cli.js
├── cli.js.map
├── package.json
├── sdk-tools.d.ts
├── bun.lock
├── LICENSE.md
├── README.md
├── README_EN.md
├── docs/
│   └── tutorial/
├── vendor/
│   ├── audio-capture/
│   └── ripgrep/
└── recovered_source/
    └── src/
```

## 这份代码是什么

这份目录更接近以下三者的结合：

1. **npm 发布产物**
2. **可执行 CLI 分发包**
3. **带 source map 与恢复源码的研究目录**

这也是为什么这里会同时出现：

- 体积很大的单文件 `cli.js`
- 更大的 `cli.js.map`
- 多平台二进制文件
- `recovered_source/` 这样的恢复源码目录

## 教程文档包含什么

仓库当前已经包含一套围绕源码阅读组织的教程文档，位于：

- 中文目录页：`docs/tutorial/00-overview.md`
- 英文目录页：`docs/tutorial/00-overview_EN.md`

教程内容主要覆盖以下主题：

1. **总览与阅读地图**
   - 仓库定位
   - 推荐阅读顺序
   - Claude Code 的整体心智模型
2. **启动链路与主程序装配**
   - `entrypoints/cli.tsx`
   - `main.tsx`
   - 启动快路径与主程序初始化
3. **QueryEngine 与会话主循环**
   - 多轮会话状态
   - system prompt / context 装配
   - query loop 与工具调用
4. **输入处理、命令系统与 hooks**
   - `commands.ts`
   - `processUserInput.ts`
   - slash command 与前置处理流水线
5. **工具系统与子代理执行**
   - `tools.ts`
   - FileRead / Bash / Agent 等代表性工具
   - 权限与角色化约束
6. **任务、设置与会话记忆**
   - `tasks.ts`
   - `settings.ts`
   - `sessionMemory.ts`
7. **MCP、插件与 skills 深挖篇**
   - MCP 配置与连接加载
   - plugin commands / plugin skills / plugin-provided MCP
   - bundled skills / skills 目录 / 动态技能发现
8. **继续深挖的阅读路径**
   - 面向不同目标的源码精读路线

## 教程入口

### 中文教程

- [00 总览与阅读地图](./docs/tutorial/00-overview.md)
- [01 启动链路与主程序装配](./docs/tutorial/01-bootstrap-and-startup.md)
- [02 QueryEngine 与会话主循环](./docs/tutorial/02-query-engine-and-conversation-loop.md)
- [03 输入处理、命令系统与 Hooks](./docs/tutorial/03-input-commands-and-hooks.md)
- [04 工具系统与子代理执行](./docs/tutorial/04-tools-and-agent-execution.md)
- [05 任务、设置与会话记忆](./docs/tutorial/05-state-tasks-settings-and-memory.md)
- [06 MCP、插件与 Skills 深挖](./docs/tutorial/06-mcp-plugins-and-skills.md)
- [99 继续深挖的阅读路径](./docs/tutorial/99-reading-paths-and-next-steps.md)

### English tutorial

- [00 Overview and reading map](./docs/tutorial/00-overview_EN.md)
- [01 Bootstrap and startup](./docs/tutorial/01-bootstrap-and-startup_EN.md)
- [02 QueryEngine and conversation loop](./docs/tutorial/02-query-engine-and-conversation-loop_EN.md)
- [03 Input, commands, and hooks](./docs/tutorial/03-input-commands-and-hooks_EN.md)
- [04 Tools and agent execution](./docs/tutorial/04-tools-and-agent-execution_EN.md)
- [05 Tasks, settings, and session memory](./docs/tutorial/05-state-tasks-settings-and-memory_EN.md)
- [06 MCP, plugins, and skills](./docs/tutorial/06-mcp-plugins-and-skills_EN.md)
- [99 Reading paths and next steps](./docs/tutorial/99-reading-paths-and-next-steps_EN.md)

## 关键组成

### `cli.js`

发布给终端用户执行的单文件 CLI 入口，已经过打包压缩，通常通过：

```bash
claude
```

或：

```bash
node cli.js --version
```

进行调用。

### `cli.js.map`

source map 文件，用于把打包后的代码映射回原始源码位置，便于调试和分析。

### `sdk-tools.d.ts`

Claude Code 内置工具的 TypeScript 类型定义，可以直接看到它暴露出的工具输入输出模型，例如：

- `Agent`
- `Bash`
- `Read`
- `Write`
- `Edit`
- `Glob`
- `Grep`
- `WebFetch`
- `WebSearch`
- `AskUserQuestion`
- `EnterWorktree` / `ExitWorktree`
- `ExitPlanMode`

### `vendor/`

发布包附带的多平台二进制资源：

- `vendor/ripgrep/`：内置 `rg`，用于高性能全文搜索
- `vendor/audio-capture/`：原生 `.node` 模块，说明其支持音频采集相关能力

### `recovered_source/`

从发布产物和映射信息中恢复出的源码视图，适合做结构研究和行为分析。

## 代码架构分析

### 1. CLI 启动入口

- `recovered_source/src/entrypoints/cli.tsx`
- `recovered_source/src/main.tsx`

`entrypoints/cli.tsx` 是较轻量的入口，负责处理若干快速路径，例如：

- `--version`
- 后台会话 / 守护进程相关命令
- remote-control / bridge 模式
- 某些 MCP 或浏览器集成入口

随后再按需加载完整主程序。

`main.tsx` 则负责主程序装配，包括：

- 初始化配置
- 解析 CLI 参数
- 初始化终端 UI
- 装载命令系统、技能系统、插件系统与 MCP 支持
- 初始化权限、任务、恢复会话等运行机制

### 2. QueryEngine 会话核心

- `recovered_source/src/QueryEngine.ts`

`QueryEngine` 是 Claude Code 对话执行模型的重要核心之一，负责：

- 管理用户消息与会话状态
- 组织系统提示词与上下文
- 协调工具调用和权限判断
- 跟踪用量、预算、文件状态与中断控制

其核心运行逻辑可以概括为：

1. 接收用户输入
2. 组装系统上下文
3. 调用模型推理
4. 执行工具调用
5. 更新状态并进入下一轮

### 3. 终端 UI 形态

从 `main.tsx` 的导入关系可以看出，这个 CLI 不只是普通命令脚本，而是更接近一个基于 React / Ink 思路构建的交互式终端应用，支持：

- REPL 式对话
- 会话恢复
- 权限确认
- 技能与插件装载
- MCP 集成
- 后台任务与多会话管理
- IDE、浏览器与远程能力集成

## 可以确认的主要能力

基于当前目录与恢复源码，可以较明确地判断 Claude Code 具备以下能力：

- 在终端中进行自然语言编程协助
- 读取、编辑、写入本地文件
- 执行 shell / bash 命令
- 搜索代码与发现文件
- 管理任务与后台会话
- 进行分步计划与执行
- 支持 Git / review / PR 相关流程
- 通过 MCP 扩展能力
- 通过 skills / plugins 扩展交互模式
- 支持会话恢复、远程连接和桥接模式

## 为什么这不是标准源码仓库

有几个非常明显的判断依据：

- 主执行逻辑被打包成单文件 `cli.js`
- 存在体积很大的 `cli.js.map`
- 包中携带多平台原生二进制
- `package.json` 更像发布包元数据，而不是开发仓库配置
- 目录中包含 `recovered_source/` 这类研究型恢复源码内容

因此，这里更适合作为：

- 发布包研究样本
- CLI 架构分析材料
- 工具接口参考目录
- 启动流程与执行模型的学习资料
- 教程型源码阅读仓库

## 关键文件索引

- `package.json`：包名、版本、Node 要求、二进制入口
- `cli.js`：发布后的实际 CLI 入口
- `cli.js.map`：source map
- `sdk-tools.d.ts`：工具接口类型定义
- `recovered_source/src/entrypoints/cli.tsx`：启动分发入口
- `recovered_source/src/main.tsx`：主程序装配入口
- `recovered_source/src/QueryEngine.ts`：对话执行核心

## 使用与研究边界

再次强调：

- 本仓库内容仅供学习、研究与技术分析
- 相关代码与内容权益归 **Anthropic** 所有
- 本 README 旨在描述结构与用途，不构成任何权利声明或许可授予

## 结论

这个目录本质上是一个 **Claude Code 发布包研究仓库 + 教程文档仓库**。它最有价值的部分在于：

- 能帮助理解 Claude Code 的 CLI 启动方式
- 能帮助分析其工具驱动型代理架构
- 能帮助系统化学习其命令、工具、状态与扩展机制
- 能帮助作为教程材料开展中英双语源码阅读

如果你的目标是理解产品结构，这个仓库已经足够有参考价值；如果你的目标是作为 Anthropic 官方上游源码继续维护，则不应把这里视为标准开发仓库。
