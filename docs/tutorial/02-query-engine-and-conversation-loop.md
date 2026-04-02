# Claude Code 源码学习教程 02：QueryEngine 与会话主循环

## 本章要解决的问题

读完这一章，你应该能回答：

- Claude Code 为什么不是“收到输入 -> 调一次 API -> 结束”这么简单？
- `QueryEngine` 在整个系统里扮演什么角色？
- 一轮用户输入是如何进入模型、触发工具、再回到下一轮状态中的？

## 先记住核心定位

`recovered_source/src/QueryEngine.ts:175-183` 已经把 `QueryEngine` 的定位写得非常清楚：

- 它拥有一次会话的 query lifecycle
- 它维护 conversation 的 session state
- 一个会话对应一个 `QueryEngine`
- 每次 `submitMessage()` 是同一会话里的新一轮 turn

这意味着 Claude Code 的核心不是“函数调用式对话”，而是**长期持有状态的多轮执行器**。

## `QueryEngine` 真正在保存什么

看 `recovered_source/src/QueryEngine.ts:184-207`，你会发现这个类持有的不是单一 messages，而是一整套跨轮次状态：

- `mutableMessages`
- `abortController`
- `permissionDenials`
- `totalUsage`
- `readFileState`
- `discoveredSkillNames`
- `loadedNestedMemoryPaths`

这说明它解决的不是“怎么发请求”这么单薄的问题，而是：

1. 当前会话已经积累了哪些消息。
2. 这一轮工具权限有没有被拒绝。
3. 文件读取缓存如何复用。
4. token / usage 如何跨轮统计。
5. 当前会话中发现了哪些 skill 和记忆材料。

## 一轮 `submitMessage()` 做了什么

### 第一步：提取本轮配置并设置执行上下文

`recovered_source/src/QueryEngine.ts:209-240` 读取本轮使用的：

- 工作目录 `cwd`
- 命令集合 `commands`
- 工具集合 `tools`
- MCP clients
- budget / model / thinking 配置
- 状态读写函数

同时它会：

- 清空本轮 skill 发现集：`QueryEngine.ts:238`
- 把 Shell 当前目录切到会话 cwd：`QueryEngine.ts:239`

也就是说，`submitMessage()` 不是“只接 prompt”，而是把整轮执行的环境一起装配起来。

### 第二步：包装权限检查函数

`recovered_source/src/QueryEngine.ts:243-271` 的 `wrappedCanUseTool` 很关键。

它做的不只是调用权限判断，还会在拒绝时把信息记入 `permissionDenials`。这代表 `QueryEngine` 不只负责执行成功路径，也负责把**权限失败事件纳入会话状态**，供 SDK 或上层使用。

### 第三步：构造系统提示词与上下文

`recovered_source/src/QueryEngine.ts:284-325` 调用了 `fetchSystemPromptParts(...)`，拿到：

- `defaultSystemPrompt`
- `userContext`
- `systemContext`

接着又根据用户自定义 prompt、memory mechanics prompt、appendSystemPrompt 拼出最终 `systemPrompt`。

这里非常关键的一点是：

> Claude Code 不是把固定 system prompt 写死，而是根据工具、模型、工作目录、MCP、额外路径等条件**动态拼装系统上下文**。

这也是为什么这个项目的“提示词工程”实际上是运行时装配问题，而不是一个静态常量文件问题。

### 第四步：构造输入处理上下文

`recovered_source/src/QueryEngine.ts:335-344` 开始构造 `processUserInputContext`。

这一步意味着在真正进入模型前，Claude Code 先要解决：

- 当前消息数组是什么
- slash command 是否会改写消息
- 状态如何写回
- 工具调用环境如何传给输入处理阶段

也就是说，用户输入不是直接送去模型，而是先经过**会话级预处理层**。

### 第五步：调用 `processUserInput()`

`recovered_source/src/QueryEngine.ts:416` 真正把输入交给 `processUserInput(...)`。

这是一个分水岭：

- 如果输入其实是某个 slash command，本轮可能根本不会触发模型请求。
- 如果 hooks 阻止继续，本轮也会提前结束。
- 如果输入需要展开附件、图片、系统上下文，也都在这一层先处理。

所以 `QueryEngine` 和 `processUserInput` 的关系可以理解为：

- `QueryEngine` 负责**总控**
- `processUserInput` 负责**把原始输入正规化成可执行会话输入**

### 第六步：进入真正的 query loop

当输入处理完成后，`QueryEngine.ts:675-681` 会进入：

- `for await (const message of query({...}))`

这说明真正的模型交互不是一个普通 Promise，而是一个**流式异步生成器**。

Claude Code 可以边收到模型增量、边处理工具调用、边更新消息流，而不是等整轮结束再一次性落地。

## `query.ts` 为什么重要

如果说 `QueryEngine` 是“会话总导演”，那么 `query.ts` 更像“单轮执行引擎”。

`recovered_source/src/query.ts:219-239` 定义了 `query(params)`，再把工作委托给 `queryLoop(...)`。从导入和局部状态定义可以看出，这里负责的事情包括：

- 自动 compact
- prompt too long / max output tokens 恢复
- tool orchestration
- stop hooks
- token budget
- 命令队列通知
- 工具结果汇总

换句话说：

- `QueryEngine` 关心“会话级生命周期”
- `query.ts` 关心“单轮模型-工具循环怎么跑下去”

## 一个更实用的心智模型

你可以把一次会话想成下面这几层：

```text
用户输入
  -> QueryEngine.submitMessage()
  -> 组装 system prompt / user context / tool context
  -> processUserInput()
  -> query()
  -> 模型流式输出
  -> 工具调用与工具结果回填
  -> 新消息写回 mutableMessages
  -> 等待下一轮 submitMessage()
```

这条链路的关键不在“调用了哪些函数”，而在它体现了 Claude Code 的基本产品形态：

> 它是一个持续累积状态、持续裁剪上下文、持续执行工具的交互式代理系统。

## `QueryEngine` 为什么是学习核心

如果你只能精读一个文件，我会优先推荐 `recovered_source/src/QueryEngine.ts`，因为它把多个系统接到了一起：

- prompt 组装
- 权限包装
- 输入处理
- 工具系统
- 状态更新
- SDK 输出格式
- session persistence

也就是说，很多“局部文件”只有放回 `QueryEngine` 里才知道自己为什么存在。

## 阅读建议：本章该怎么精读源码

### 第一轮，只看结构

先读这些位置：

- `recovered_source/src/QueryEngine.ts:175-207`
- `recovered_source/src/QueryEngine.ts:209-325`
- `recovered_source/src/QueryEngine.ts:335-344`
- `recovered_source/src/QueryEngine.ts:416`
- `recovered_source/src/QueryEngine.ts:675-681`

目的是回答：**它的输入、状态、输出分别是什么？**

### 第二轮，再补 `query.ts`

读这些位置：

- `recovered_source/src/query.ts:181-217`
- `recovered_source/src/query.ts:219-260`

目的是回答：**单轮执行为什么需要一个流式状态机？**

## 本章总结

这一章最重要的结论是：

> Claude Code 的核心不是一次 API 调用，而是一个维护消息、权限、缓存、预算和工具执行结果的会话引擎。

而 `QueryEngine` 正是这个会话引擎最值得精读的主文件。

## 下一章

继续阅读 [03-input-commands-and-hooks.md](./03-input-commands-and-hooks.md)。
