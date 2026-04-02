# Claude Code 源码学习教程 00：总览与阅读地图

## 这份教程适合谁

这套教程面向两类读者：

1. 想系统理解 Claude Code 这类终端代理式 CLI 如何组织架构的人。
2. 想把当前仓库当作“源码分析样本”来学习启动流程、工具系统、任务系统、设置系统的人。

## 先说清楚：这是什么仓库

当前目录不是 Anthropic 官方公开维护的上游开发仓库，而是一个**已打包发布产物 + 恢复源码视图**的分析目录。这个定位在仓库自己的说明里已经写得很明确：`README.md:14` 把它定义为“已打包发布产物分析目录”，`README.md:118` 说明 `recovered_source/` 是从发布产物和映射信息中恢复出的源码视图。

因此，学习这个项目时要带着两个前提：

- 你看到的是**适合研究的恢复源码**，不是官方原始工程组织形态。
- 这里特别适合做**架构理解、运行机制拆解、工具能力观察**，不适合把它当作官方二次开发基座。

## 这个包的顶层结构

结合 `README.md:27`、`README.md:38` 和 `package.json:2` 可以把顶层目录先记成下面这张图：

```text
package/
├── cli.js                # 打包后的实际 CLI 入口
├── cli.js.map            # source map，帮助恢复源码
├── package.json          # npm 包元数据，声明包名/版本/bin
├── sdk-tools.d.ts        # 工具接口类型定义
├── vendor/               # 随包分发的原生/辅助二进制
└── recovered_source/     # 从发布物恢复出的源码主视图
```

其中最值得学习的几块是：

- `package.json:2-13`：确认这是 `@anthropic-ai/claude-code`，以及 CLI 的入口是 `cli.js`。
- `README.md:124-175`：已经给出了一版简明架构概览。
- `recovered_source/src/`：真正的学习主战场。

## 如何理解这个项目

如果只用一句话概括，可以把 Claude Code 看成：

> 一个基于终端交互界面运行的、由大模型驱动的“会话执行器”，它把命令系统、工具系统、权限系统、状态系统和子代理系统拼装在一起。

它不是普通的“一次性命令行工具”，而是更接近一个持续运行的交互式应用。仓库说明也已经点出了这一点：`README.md:165-175` 提到它具备 REPL 式对话、权限确认、会话恢复、技能/插件装载、MCP 集成、后台任务等能力。

## 推荐学习顺序

这套教程按照“从外到内”的顺序组织：

1. **启动层**：CLI 如何进入主程序。
2. **主循环层**：一次用户输入如何变成一轮模型-工具-状态循环。
3. **交互层**：slash commands、hooks、附件输入如何进入系统。
4. **工具层**：Claude Code 为什么能读文件、改文件、跑命令、开子代理。
5. **状态层**：任务、设置、记忆如何落盘与复用。

建议按下面顺序阅读：

- [01-bootstrap-and-startup.md](./01-bootstrap-and-startup.md)
- [02-query-engine-and-conversation-loop.md](./02-query-engine-and-conversation-loop.md)
- [03-input-commands-and-hooks.md](./03-input-commands-and-hooks.md)
- [04-tools-and-agent-execution.md](./04-tools-and-agent-execution.md)
- [05-state-tasks-settings-and-memory.md](./05-state-tasks-settings-and-memory.md)
- [99-reading-paths-and-next-steps.md](./99-reading-paths-and-next-steps.md)

## 整体心智模型

学习源码时，建议先记住这条主线：

```text
CLI 入口
  -> 主程序初始化
  -> 装配命令/工具/权限/上下文
  -> 接收用户输入
  -> 生成模型请求
  -> 执行工具调用
  -> 更新会话状态
  -> 继续下一轮交互
```

对应到关键文件：

- CLI 薄入口：`recovered_source/src/entrypoints/cli.tsx:28-217`
- 主装配入口：`recovered_source/src/main.tsx:585-719`
- 会话核心：`recovered_source/src/QueryEngine.ts:175-344`
- 交互预处理：`recovered_source/src/utils/processUserInput/processUserInput.ts:140-309`
- 命令注册：`recovered_source/src/commands.ts:353-509`
- 工具装配：`recovered_source/src/tools.ts:179-362`
- 任务系统：`recovered_source/src/utils/tasks.ts:199-438`
- 设置系统：`recovered_source/src/utils/settings/settings.ts:74-220`
- 会话记忆：`recovered_source/src/services/SessionMemory/sessionMemory.ts:134-319`

## 学习这份源码时最有价值的观察点

### 1. 它把“模型能力”严格收束到工具表面

不是让模型直接拥有系统权限，而是通过 `tools.ts` 组装一组可控工具，再通过权限模式和规则进一步筛选。

### 2. 它把“交互输入”当作一个单独的编排问题

用户发来的不一定只是普通文本，还可能是 slash command、附加上下文、图像、hook 插入内容、系统生成消息。

### 3. 它把“持续会话”当成核心而不是附属功能

`QueryEngine` 不是简单执行一次 API 请求，而是维护消息、状态、权限拒绝记录、文件读取缓存、预算等跨轮次数据。

### 4. 它把“工程安全边界”写进了工具实现里

例如：读文件工具会阻止危险设备文件；Bash 工具里有安全解析、只读约束、后台任务治理；Explore 这类子代理被显式限制为只读。

## 阅读方法建议

建议你读每个章节时都回答三个问题：

1. **这个文件在系统里的职责是什么？**
2. **它的输入从哪里来，输出交给谁？**
3. **它在保护什么边界？性能、权限、缓存、还是状态一致性？**

如果你带着这三个问题读，这份恢复源码会比“从头到尾浏览文件”更容易建立清晰模型。

## 下一章

继续阅读 [01-bootstrap-and-startup.md](./01-bootstrap-and-startup.md)。
