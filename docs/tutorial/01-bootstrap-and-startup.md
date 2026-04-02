# Claude Code 源码学习教程 01：启动链路与主程序装配

## 本章要解决的问题

读完这一章，你应该能回答：

- `claude` 命令到底从哪里开始执行？
- 为什么项目同时有 `cli.js`、`entrypoints/cli.tsx` 和 `main.tsx`？
- 启动阶段为什么要拆成“薄入口 + 重装配”两层？

## 先看最外层：npm 包入口

`package.json:2-13` 说明这个包名是 `@anthropic-ai/claude-code`，并且通过：

- `package.json:4-6` 把二进制命令 `claude` 指向 `cli.js`

也就是说，用户在终端里敲 `claude`，首先进入的是打包产物 `cli.js`。但学习时我们不直接分析压缩产物，而是看恢复源码里的两层入口：

- `recovered_source/src/entrypoints/cli.tsx`
- `recovered_source/src/main.tsx`

## 第一层入口：轻量分发器 `entrypoints/cli.tsx`

`recovered_source/src/entrypoints/cli.tsx:28-33` 直接给出了它的定位：

> Bootstrap entrypoint - checks for special flags before loading the full CLI.

这句话非常重要。它告诉你：

- 这里不是完整应用逻辑。
- 它的任务是**先识别特殊路径**，避免一上来就把整套主程序全部加载进去。

### 为什么要有这层薄入口

从 `recovered_source/src/entrypoints/cli.tsx:36-41` 可以看到最典型的快路径：

- 只有 `--version` / `-v` / `-V` 时，直接输出版本并返回。

这意味着最简单的命令不必加载 React UI、工具系统、设置系统、MCP、命令系统等重模块。这是标准的 CLI 性能优化思路：

- **简单请求尽量零成本返回**
- **复杂路径再进入完整初始化**

### 它识别了哪些特殊模式

从 `recovered_source/src/entrypoints/cli.tsx:50-217` 可以看出，薄入口会提前分流多类场景，例如：

- `--dump-system-prompt`：导出系统提示词
- `--claude-in-chrome-mcp` / `--chrome-native-host`
- `--daemon-worker`
- `remote-control` / `remote` / `sync` / `bridge`
- `daemon`
- `ps` / `logs` / `attach` / `kill` / `--bg`
- 模板任务相关命令

### 这一层的设计重点

这里的关键词不是“功能多”，而是“**把昂贵初始化延后**”。

你会发现它大量使用动态 `import()`，例如：

- `recovered_source/src/entrypoints/cli.tsx:45-48`
- `recovered_source/src/entrypoints/cli.tsx:55-67`
- `recovered_source/src/entrypoints/cli.tsx:101-104`
- `recovered_source/src/entrypoints/cli.tsx:191-207`

这说明它在做两件事：

1. 降低启动时的模块求值成本。
2. 把功能按运行路径切分，只有命中某路径才加载相应模块。

## 第二层入口：`main.tsx` 负责完整主程序装配

如果说 `entrypoints/cli.tsx` 负责“先分流”，那 `main.tsx` 就负责“把真正的应用组装起来”。

`recovered_source/src/main.tsx:585` 定义了主入口 `main()`。这个文件体量很大，导入也很多，所以不要试图一次吃透；正确读法是先看它在启动阶段做了哪些类别的事情。

## `main.tsx` 启动时做的四类工作

### 1. 先处理全局运行环境与进程级安全

`recovered_source/src/main.tsx:588-607` 很值得注意：

- `NoDefaultCurrentDirectoryInExePath` 被设置为 `1`
- 早期初始化 warning handler
- 注册 `exit` / `SIGINT` 处理

这说明项目在启动时不只是“开 UI”，还会先处理：

- 路径劫持风险
- 终端退出时的清理动作
- 交互式和 print 模式的中断差异

### 2. 处理极早期的 URL / 深链 / 特殊模式改写

`recovered_source/src/main.tsx:609-719` 展示了第二类工作：

- 解析 `cc://`、`cc+unix://` 等连接 URL
- 处理 `--handle-uri`
- 兼容 macOS URL handler
- 处理 `assistant` / `ssh` 等特殊子路径

这一层体现的是：

> 主程序启动前，参数本身可能需要被“重写”成统一主流程能消费的形式。

也就是说，`main.tsx` 不是简单消费 argv，而是在把各种异构入口转换为统一运行模型。

### 3. 执行完整初始化

`recovered_source/src/main.tsx:916` 调用了 `await init()`。

这通常意味着：

- 配置读取
- 环境变量应用
- 认证/策略/远程设置预热
- 启动所需的全局状态准备

虽然我们这一章不展开 `init()` 内部，但你要记住：

- `entrypoints/cli.tsx` 负责避免不必要的初始化
- `main.tsx` 真正进入完整模式后，才做系统级准备

### 4. 装配命令并进入 REPL / 交互运行

`recovered_source/src/main.tsx:1928` 提前准备 `commandsPromise = getCommands(preSetupCwd)`。

这一步很关键，因为它意味着：

- 命令系统不是硬编码死在 UI 里的
- 启动时会把内建命令、技能命令、插件命令、工作流命令等聚合出来

随后你还能看到多处 `launchRepl(...)`，例如：

- `recovered_source/src/main.tsx:3134`
- `recovered_source/src/main.tsx:3176`
- `recovered_source/src/main.tsx:3242`

这说明主程序最终会进入一个**长期运行的交互会话**，而不是一次性执行后退出。

## 为什么这套分层设计很重要

很多人第一次看这类项目时，会误以为“入口文件就是主业务文件”。但在 Claude Code 里，更准确的理解是：

### 层 1：启动分发层

`entrypoints/cli.tsx`

职责：

- 快速判断路径
- 提前处理廉价命令
- 减少重模块加载
- 把特殊模式送往各自 handler

### 层 2：应用装配层

`main.tsx`

职责：

- 初始化运行时环境
- 建立权限、设置、认证、遥测、命令、插件、技能等系统
- 构造终端交互上下文
- 启动 REPL / 主循环

这是一种很典型的高复杂度 CLI 架构：

- **薄入口负责性能和分发**
- **厚入口负责组装和长期运行**

## 你应该重点盯住哪些代码点

### 启动快路径

- `recovered_source/src/entrypoints/cli.tsx:33-41`
- `recovered_source/src/entrypoints/cli.tsx:50-71`
- `recovered_source/src/entrypoints/cli.tsx:95-105`
- `recovered_source/src/entrypoints/cli.tsx:108-209`

它们回答的是：**什么情况不需要完整主程序？**

### 主程序早期环境治理

- `recovered_source/src/main.tsx:585-607`
- `recovered_source/src/main.tsx:609-719`

它们回答的是：**完整主程序在真正进入业务前要先做哪些全局准备？**

### 主循环启动前的装配

- `recovered_source/src/main.tsx:916`
- `recovered_source/src/main.tsx:1928`
- `recovered_source/src/main.tsx:3134`

它们回答的是：**什么时候系统才算真正进入交互态？**

## 本章总结

这一章你只要记住一句话就够了：

> Claude Code 的启动不是“执行一个脚本”，而是“先经过轻量路由，再进入完整交互应用的装配流程”。

这也是为什么这个项目更像一个“终端应用框架”，而不是传统 CLI 小工具。

## 下一章

继续阅读 [02-query-engine-and-conversation-loop.md](./02-query-engine-and-conversation-loop.md)。
