# Claude Code 源码学习教程 04：工具系统与子代理执行

## 本章要解决的问题

读完这一章，你应该能回答：

- Claude Code 为什么必须有工具系统？
- 工具是如何被注册、筛选和暴露给模型的？
- 子代理（Agent）和普通工具之间是什么关系？

## 工具系统是 Claude Code 的“操作表面”

Claude Code 并不是让模型直接拥有文件系统和终端能力，而是通过一组明确注册的工具，把能力包装成可控接口再交给模型。

学习这一层最重要的入口是：

- `recovered_source/src/tools.ts`

这个文件不是某一个工具的实现，而是**整套工具池的装配中心**。

## `tools.ts` 先回答一个问题：系统可能有哪些工具？

### 基础工具全集

`recovered_source/src/tools.ts:193-250` 的 `getAllBaseTools()` 是工具总表的核心入口。

它会把很多能力拼成一个数组，包括：

- `AgentTool`
- `BashTool`
- `FileReadTool`
- `FileEditTool`
- `FileWriteTool`
- `GlobTool`
- `GrepTool`
- `WebFetchTool`
- `WebSearchTool`
- `AskUserQuestionTool`
- `EnterPlanModeTool`
- `Task*` 系列工具
- MCP 相关工具
- 多种 feature flag 控制下的额外工具

这说明 Claude Code 的能力模型不是“内置几个固定函数”，而是一个**按环境、feature gate、权限模式组装出来的工具池**。

## 为什么工具池不是固定的

从 `tools.ts:193-250` 能看出三种典型裁剪来源：

### 1. feature gate 裁剪

很多工具只有在特定 feature 开启时才会加入，例如：

- `SleepTool`
- `RemoteTriggerTool`
- `WorkflowTool`
- `WebBrowserTool`
- `SnipTool`

### 2. 环境裁剪

例如：

- `process.env.USER_TYPE === 'ant'` 才有的一些内部工具
- `ENABLE_LSP_TOOL` 控制 LSPTool
- `isWorktreeModeEnabled()` 控制 worktree 工具

### 3. 权限上下文裁剪

`recovered_source/src/tools.ts:262-327` 的 `filterToolsByDenyRules(...)` 与 `getTools(...)` 会根据 permission context 继续过滤工具。

这说明：

> 工具不是“注册完就一定能用”，而是“先声明可用集合，再根据当前会话安全边界收窄”。

## `getTools()` 的意义：把工具表变成当前会话可见工具

`recovered_source/src/tools.ts:271-327` 的 `getTools(...)` 是最值得精读的地方。

它做了几层工作：

1. 处理 simple mode：`tools.ts:271-298`
2. 过滤特殊工具和 deny rules：`tools.ts:300-310`
3. REPL 模式下隐藏 primitive tools：`tools.ts:312-323`
4. 最后再按 `isEnabled()` 做收尾过滤：`tools.ts:325-326`

这说明工具暴露给模型前，要经过多轮筛选：

- 模式约束
- 权限约束
- 运行时启用状态约束

## 一个代表性工具：`FileReadTool`

学习工具实现时，不要一次试图看完整个 `tools/` 目录。最好的方法是挑代表性工具。

### 为什么先看读文件工具

因为它最能体现 Claude Code 的设计原则：

- 能力明确
- 输入输出结构化
- 安全边界前置
- UI 展示与模型结果分离

### `FileReadTool` 关注的不是“怎么读文件”这么简单

看 `recovered_source/src/tools/FileReadTool/FileReadTool.ts:337-495`，你会发现它在定义工具时同时声明了：

- `name`
- `description()` / `prompt()`
- 输入输出 schema
- 活动描述
- 只读属性：`isReadOnly()`，见 `FileReadTool.ts:376-378`
- 权限检查
- UI 展示函数
- 输入校验

这说明工具实现其实包含了四层职责：

1. **给模型看的接口定义**
2. **给系统看的安全与权限约束**
3. **给 UI 看的展示规则**
4. **给运行时看的执行逻辑**

### 安全边界写在工具内部

`FileReadTool` 很值得学习的一点，是它把很多安全限制前置到了工具层：

- 阻止读取危险设备文件：`FileReadTool.ts:98-127`
- 输入校验阶段阻断：`FileReadTool.ts:418-495`
- 阻止被 deny rule 命中的路径：`FileReadTool.ts:442-458`
- 二进制文件类型限制：`FileReadTool.ts:469-482`
- 对设备文件再次做显式阻断：`FileReadTool.ts:484-492`
- 超出 token 限额会抛出 `MaxFileReadTokenExceededError`：`FileReadTool.ts:175-183`、`FileReadTool.ts:770`

这说明工具层承担的是**系统安全边界的一部分**，不是单纯的功能封装。

## 再看一个代表性工具：`BashTool`

`recovered_source/src/tools/BashTool/BashTool.tsx` 更能体现 Claude Code 的高风险工具治理思路。

从 `BashTool.tsx:54-81` 可以看到它先定义了多组命令语义分类：

- 搜索类命令
- 读取类命令
- 列目录类命令
- 语义中性命令
- 静默命令

这说明 Bash 并不是“把一串命令交给 shell”这么粗暴，而是先对命令语义进行分类，以便：

- UI 更好展示
- 区分读操作与写操作
- 做只读模式限制
- 做后台任务和结果摘要治理

也就是说，BashTool 的真正难点不是 shell 执行，而是**让一个危险能力被纳入可解释、可限制的框架**。

## 子代理为什么也是工具系统的一部分

很多人会把 Agent 理解成独立于工具系统之外的能力，但在 Claude Code 的实现中，它本质上也是一种工具入口。

### `runAgent.ts` 展示了子代理的真实角色

`recovered_source/src/tools/AgentTool/runAgent.ts:248-329` 定义了 `runAgent(...)` 的主要参数。你会看到子代理启动时需要拿到：

- agent 定义
- prompt messages
- toolUseContext
- canUseTool
- availableTools
- allowedTools
- model
- maxTurns
- worktreePath
- transcriptSubdir

这意味着子代理并不是“另一个神秘模型实例”，而是：

> 在父会话提供的上下文、权限、工具池和状态约束下，启动出的一个受控执行单元。

### 子代理会继承，但不会无限继承

`runAgent.ts:380-410` 这段很值得看：

- 它会解析 user/system context
- 对 Explore/Plan 这类只读 agent，刻意省掉 `claudeMd` 和 `gitStatus` 之类不必要上下文

这说明子代理系统也在做两件事：

1. 保持和父会话的上下文关联
2. 避免把不必要的大上下文无差别复制进去

### Explore agent 是一个很好的“只读子代理”样本

`recovered_source/src/tools/AgentTool/built-in/exploreAgent.ts:24-83` 非常有教学价值，因为它把子代理的角色定义写得极其直接：

- 这是 file search specialist
- 明确是 READ-ONLY exploration task
- 禁止创建/修改/删除文件
- 强调只做搜索与分析
- 通过 `disallowedTools` 禁掉 edit / write / notebook edit / exit plan mode

这说明 Claude Code 的子代理不是“能力越强越好”，而是根据任务角色精确裁剪工具面。

## 工具系统的真正价值

读到这里，你应该能理解 Claude Code 的一个核心工程思想：

> 模型能力不是靠“放权”获得，而是靠“工具化 + 权限化 + 角色化”获得。

其中：

- 工具化：把能力收束成结构化接口
- 权限化：按会话规则裁剪工具表
- 角色化：对子代理再做一次工具限制和上下文瘦身

## 推荐精读路径

### 第一步：先看工具池装配

- `recovered_source/src/tools.ts:193-327`

回答问题：**系统有哪些工具，它们为什么不会同时全部暴露？**

### 第二步：再看读文件工具

- `recovered_source/src/tools/FileReadTool/FileReadTool.ts:337-495`

回答问题：**一个工具如何同时承担接口定义、安全校验、权限检查和 UI 呈现？**

### 第三步：补看 Bash 与 Agent

- `recovered_source/src/tools/BashTool/BashTool.tsx:54-81`
- `recovered_source/src/tools/AgentTool/runAgent.ts:248-410`
- `recovered_source/src/tools/AgentTool/built-in/exploreAgent.ts:24-83`

回答问题：**高风险能力与子代理能力是如何被收束的？**

## 本章总结

这一章最重要的结论是：

> 工具系统就是 Claude Code 的能力边界本身。

模型之所以能“读文件、改文件、跑命令、开子代理”，不是因为模型天然会这些事，而是因为系统把这些能力包装成结构化工具，再通过权限、模式、角色三层过滤后暴露出来。

## 下一章

继续阅读 [05-state-tasks-settings-and-memory.md](./05-state-tasks-settings-and-memory.md)。
