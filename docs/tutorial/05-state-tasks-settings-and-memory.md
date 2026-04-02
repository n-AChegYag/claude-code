# Claude Code 源码学习教程 05：任务、设置与会话记忆

## 本章要解决的问题

读完这一章，你应该能回答：

- Claude Code 的状态除了 messages 之外，还落在哪里？
- 任务系统为什么不是一个内存数组？
- 设置系统为什么要有解析、缓存、合并和校验多层逻辑？
- “记忆”在这个项目里是如何自动提炼的？

## 为什么要学这一层

前几章主要在看“系统如何跑起来”，这一章看的是“系统如何记住东西、组织东西、延续东西”。

在 Claude Code 这类长会话 CLI 中，真正决定体验上限的，往往不是单轮模型输出，而是这些持续状态系统：

- 任务系统
- 设置系统
- 会话记忆系统

## 一、任务系统：`utils/tasks.ts`

### 它不是 UI 附属功能，而是一个落盘数据层

`recovered_source/src/utils/tasks.ts` 第一眼看起来像工具辅助模块，但其实它已经接近一个小型持久化层。

从 `recovered_source/src/utils/tasks.ts:69-89` 可以看到任务 schema：

- `id`
- `subject`
- `description`
- `activeForm`
- `owner`
- `status`
- `blocks`
- `blockedBy`
- `metadata`

这意味着任务不是“只给界面显示的列表项”，而是有完整结构、依赖关系和状态流转的实体。

### 任务是按目录落盘的

`recovered_source/src/utils/tasks.ts:221-240` 定义了：

- `getTasksDir(taskListId)`
- `getTaskPath(taskListId, taskId)`
- `ensureTasksDir(taskListId)`

这说明任务会落到 Claude 配置目录下，而不是仅存在当前进程内存里。这样做的价值是：

- 可以跨刷新/跨轮次保留
- 可以支持多进程/多 agent 协作
- 可以被 watcher 或其他会话看到

### 任务 ID 设计考虑了并发

`recovered_source/src/utils/tasks.ts:91-130` 和 `:267-307` 很值得认真看。

这里有：

- 高水位标记 `.highwatermark`
- lockfile 重试与退避配置
- 创建任务时先持锁、再读最高 ID、再写入

这说明任务系统不是为“单线程玩具场景”写的，而是为：

> 多个 Claude/子代理/协作进程同时改任务列表

这种情况准备的。

### 任务列表 ID 还支持团队/会话上下文

`recovered_source/src/utils/tasks.ts:190-210` 的 `getTaskListId()` 会按优先级决定任务列表归属：

- 显式 task list id
- teammate/team name
- session id

这进一步说明任务系统天然支持“同一会话”与“同一团队”两种协作维度。

### 增删改查都考虑了一致性

核心入口分别在：

- 创建：`tasks.ts:284-307`
- 读取：`tasks.ts:310-349`
- 更新：`tasks.ts:370-391`
- 删除：`tasks.ts:393-438`

尤其删除逻辑不仅删文件，还会更新高水位并移除其他任务对它的依赖引用。这是典型的数据层一致性处理，而不是简单删个 JSON 文件了事。

## 二、设置系统：`utils/settings/settings.ts`

如果任务系统解决的是“做什么”，那设置系统解决的是“允许怎么做、从哪里读取这些规则”。

### 设置系统首先是多来源合并系统

`recovered_source/src/utils/settings/settings.ts:74-120` 的 `loadManagedFileSettings()` 展示了很重要的设计：

- 先读 `managed-settings.json`
- 再读 `managed-settings.d/*.json`
- drop-in 文件按名称排序后叠加

这是一种非常成熟的配置设计思路，类似 systemd/sudoers 的 drop-in 机制。它的价值在于：

- 基础配置与增量配置分离
- 多团队可以独立下发策略片段
- 管理侧不必永远改同一个大文件

### 配置解析自带缓存

`recovered_source/src/utils/settings/settings.ts:178-199` 的 `parseSettingsFile()` 先查缓存，再调用 `parseSettingsFileUncached()`。

这说明设置系统考虑的不是“能不能读到文件”，而是：

- 配置读取可能很频繁
- 解析结果需要缓存
- 返回前还要 clone，防止调用方污染缓存内容

这体现了很强的工程化意识。

### 配置不是盲读 JSON，而是“解析 + 过滤 + 校验”

`recovered_source/src/utils/settings/settings.ts:201-220` 可以看到未缓存解析流程：

1. 安全解析路径
2. 读取文件内容
3. 空文件返回空设置
4. JSON 解析
5. 先过滤无效 permission rules
6. 再进入 `SettingsSchema` 校验

这说明设置系统最重要的工作不是“读文件”，而是**把不可靠外部输入变成可信内部结构**。

## 三、会话记忆系统：`services/SessionMemory/sessionMemory.ts`

### 记忆不是聊天历史本身，而是提炼出来的会话笔记

`recovered_source/src/services/SessionMemory/sessionMemory.ts:1-5` 的文件头就直接写出目标：

- 自动维护 markdown 记忆文件
- 周期性后台运行
- 用 forked subagent 提取关键信息
- 不阻塞主对话流程

这说明这里的“memory”不是 transcript，也不是原始 message 列表，而是**从会话中抽取出的摘要性知识**。

### 什么时候触发记忆提炼

`recovered_source/src/services/SessionMemory/sessionMemory.ts:134-181` 的 `shouldExtractMemory(messages)` 是核心判断函数。

它会综合考虑：

- 当前 token 总量
- 是否已达到初始化阈值
- 距离上次提炼是否积累了足够 token
- 工具调用次数是否达到阈值
- 最近一轮 assistant 是否还包含 tool calls

这说明记忆系统并不是“每轮都总结”，而是在找一个成本和收益更平衡的触发点。

### 会话记忆文件如何建立

`recovered_source/src/services/SessionMemory/sessionMemory.ts:183-233` 的 `setupSessionMemoryFile(...)` 很值得看。

这里会：

- 创建记忆目录
- 如果文件不存在就初始化模板
- 清掉 read cache，避免拿到 `file_unchanged` stub
- 再通过 `FileReadTool.call(...)` 读取当前记忆内容

这说明记忆系统虽然是后台能力，但仍然尽量复用已有工具体系，而不是偷偷绕开工具层另开一套文件协议。

### 记忆提炼本身由 forked agent 完成

`recovered_source/src/services/SessionMemory/sessionMemory.ts:272-319` 显示：

- 先检查 gate
- 再检查是否满足提炼条件
- 然后创建隔离上下文
- 构造更新 prompt
- 最后通过 `runForkedAgent(...)` 运行提炼

这说明 Claude Code 的一个重要架构原则是：

> 即使是“系统内部维护能力”，也尽量复用 agent + tool 这套统一机制。

## 把三者串起来理解

这三套系统看起来分散，其实都在回答同一个问题：

> 一次长会话里，哪些东西应该持续存在？

- 任务系统保留“当前要做什么”
- 设置系统保留“当前允许怎么做、默认怎么做”
- 会话记忆保留“当前会话里有哪些长期有价值的信息”

它们一起把 Claude Code 从“即时对话器”提升成“可持续工作的终端代理”。

## 推荐精读路径

### 任务系统

- `recovered_source/src/utils/tasks.ts:69-89`
- `recovered_source/src/utils/tasks.ts:190-307`
- `recovered_source/src/utils/tasks.ts:370-438`

回答问题：**为什么任务需要文件锁、高水位和依赖回写？**

### 设置系统

- `recovered_source/src/utils/settings/settings.ts:74-120`
- `recovered_source/src/utils/settings/settings.ts:178-220`

回答问题：**为什么配置读取不是简单 `JSON.parse(readFile())`？**

### 会话记忆系统

- `recovered_source/src/services/SessionMemory/sessionMemory.ts:134-181`
- `recovered_source/src/services/SessionMemory/sessionMemory.ts:183-233`
- `recovered_source/src/services/SessionMemory/sessionMemory.ts:272-319`

回答问题：**为什么记忆提炼要单独找触发时机，还要用 forked agent？**

## 本章总结

这一章最重要的结论是：

> Claude Code 的长期工作能力，不只来自 QueryEngine，也来自一组被工程化得很深的持续状态系统。

任务让工作可以分解和协作，设置让行为可控，记忆让长期上下文不会完全依赖当前窗口消息。

## 下一章

继续阅读 [06-mcp-plugins-and-skills.md](./06-mcp-plugins-and-skills.md)。
