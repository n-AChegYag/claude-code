# Claude Code 源码学习教程 03：输入处理、命令系统与 Hooks

## 本章要解决的问题

读完这一章，你应该能回答：

- 为什么用户输入不能直接丢给模型？
- slash command 在哪里注册、怎么加载？
- hooks 为什么会改变一轮交互是否继续执行？

## 先看命令系统：`commands.ts`

Claude Code 的命令系统不是“几个 if/else 判断 `/help`、`/clear`”这么简单，而是一个聚合多个来源的命令注册表。

### 内建命令非常多

`recovered_source/src/commands.ts:2-188` 里可以看到大量内建命令导入，例如：

- `/help`
- `/login`
- `/logout`
- `/tasks`
- `/plan`
- `/permissions`
- `/model`
- `/status`
- `/memory`
- `/hooks`
- `/plugin`
- `/agents`

这说明 slash command 在 Claude Code 中不是边缘功能，而是第一等交互入口。

### 命令不只来自内建模块

更重要的是 `recovered_source/src/commands.ts:353-509`。这里展示了命令是如何从多个来源拼出来的：

- skill 目录命令：`commands.ts:353-386`
- plugin skills
- bundled skills
- builtin plugin skills
- plugin commands：`commands.ts:449-468`
- workflow commands
- 最后再拼上内建 `COMMANDS()`

`recovered_source/src/commands.ts:476-509` 的 `getCommands(cwd)` 则进一步做了：

- availability 过滤
- `isEnabled()` 过滤
- dynamic skills 注入
- 去重与插入位置控制

这说明 Claude Code 的命令系统并不是静态数组，而是一个**按 cwd、插件、技能、认证状态动态装配的命令集合**。

## 为什么命令系统要这样设计

因为 Claude Code 并不只服务一个固定的 CLI 产品壳，它还支持：

- 用户自定义 skill
- 插件扩展
- 运行时动态发现技能
- 基于用户状态/提供商状态进行可用性裁剪

所以命令系统必须是可组合的，而不能是简单硬编码。

## 再看输入预处理：`processUserInput.ts`

如果说 `commands.ts` 决定“系统有哪些命令可用”，那么 `processUserInput.ts` 决定“当前这次输入到底该怎么解释”。

### `processUserInput()` 是输入总入口

`recovered_source/src/utils/processUserInput/processUserInput.ts:85-140` 定义了主函数 `processUserInput(...)`。

它接收的并不只是字符串，还包括：

- `mode`
- `pastedContents`
- `ideSelection`
- 历史 `messages`
- `querySource`
- `canUseTool`
- `skipSlashCommands`
- `bridgeOrigin`
- `isMeta`
- `skipAttachments`

这说明 Claude Code 的“用户输入”是一个比纯文本复杂得多的对象。

## `processUserInput()` 的工作分两层

### 第一层：基础输入正规化

`recovered_source/src/utils/processUserInput/processUserInput.ts:153-171` 先调用 `processUserInputBase(...)`。

基础层会处理：

- 普通文本
- slash command 解析
- 粘贴内容扩展
- 图片/附件输入
- 是否应该真正进入 query

也就是说，这一层先决定：

> 这次输入究竟是一次“聊天请求”，还是一次“本地命令执行/系统内控制动作”。

### 第二层：UserPromptSubmit hooks

如果基础处理结果允许继续，`processUserInput.ts:178-263` 就会开始执行 `executeUserPromptSubmitHooks(...)`。

这是一个非常重要的扩展点。

从代码可以看到，hook 执行后可能发生几种结果：

1. **阻断并报错**：`processUserInput.ts:193-209`
2. **阻止继续但保留上下文**：`processUserInput.ts:211-224`
3. **附加上下文材料**：`processUserInput.ts:226-240`
4. **追加 hook 产生的消息**：`processUserInput.ts:242-262`

换句话说，hook 不是“边缘日志钩子”，而是能真正影响这一轮是否继续的控制层。

## 这说明了什么

这说明 Claude Code 在设计上把“用户输入”视为一条**可编排流水线**：

```text
原始输入
  -> 基础解析
  -> slash command / 附件 / 粘贴扩展
  -> hook 检查
  -> 附加上下文
  -> 决定是否进入模型 query
```

这和很多聊天应用的思路完全不同。很多系统会把 hooks 放在外围，但这里 hooks 已经进入了“主执行前置层”。

## slash command 与模型输入的关系

一个容易混淆的点是：slash command 并不总是等于“直接执行本地功能并结束”。

在 Claude Code 里，命令有不同类型：

- 有些命令会直接返回本地结果
- 有些命令会构造 prompt 再交给模型
- 有些命令会改写消息状态
- 有些命令只是在当前会话里切换模式/配置

这也是为什么 `QueryEngine` 在调用 `processUserInput()` 之前，必须把 commands、tool context、message context 都准备好。

## 为什么输入层这么复杂

因为 Claude Code 支持的不只是“用户敲一段自然语言”。它还支持：

- slash commands
- 带图像的多模态输入
- 粘贴内容展开
- IDE 选择区上下文
- 远程/bridge 来源输入
- 系统生成的 meta 输入
- hook 注入上下文

所以输入层本质上做的是：

> 把各种异构来源统一整理成“这轮会话真正该处理的消息集合”。

## 推荐精读片段

### 命令系统

- `recovered_source/src/commands.ts:353-398`
- `recovered_source/src/commands.ts:449-509`

要回答的问题：**命令是从哪些来源聚合出来的？**

### 输入处理主流程

- `recovered_source/src/utils/processUserInput/processUserInput.ts:140-171`
- `recovered_source/src/utils/processUserInput/processUserInput.ts:178-263`
- `recovered_source/src/utils/processUserInput/processUserInput.ts:281-309`

要回答的问题：**输入是如何被分层加工的？**

## 本章总结

这一章最重要的结论是：

> Claude Code 把“用户输入”当作一条可编排流水线，而不是一段直接送去模型的字符串。

命令系统决定系统有哪些交互入口；输入处理系统决定当前这次输入最终变成什么；hooks 则提供了在主循环前改写、阻断或补充上下文的能力。

## 下一章

继续阅读 [04-tools-and-agent-execution.md](./04-tools-and-agent-execution.md)。
