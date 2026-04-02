# Claude Code 源码学习教程 99：继续深挖的阅读路径

[English Version](./99-reading-paths-and-next-steps_EN.md)

如果你已经读完前面的章节，这里给你几条不同目的的继续学习路径。

## 路线一：想理解“Claude Code 为什么像一个操作系统壳层”

按这个顺序继续精读：

1. `recovered_source/src/main.tsx`
2. `recovered_source/src/QueryEngine.ts`
3. `recovered_source/src/query.ts`
4. `recovered_source/src/tools.ts`

重点关注：

- 启动时装配了哪些系统
- QueryEngine 如何持有会话状态
- query loop 如何处理工具与 token 预算
- 工具池如何根据权限和 feature gate 收缩

## 路线二：想理解“交互层为什么这么复杂”

按这个顺序继续精读：

1. `recovered_source/src/commands.ts`
2. `recovered_source/src/utils/processUserInput/processUserInput.ts`
3. `recovered_source/src/utils/messages.ts`
4. `recovered_source/src/utils/attachments.ts`

重点关注：

- slash command 如何聚合
- hooks 如何拦截或补充上下文
- 输入是如何被正规化成消息的
- 附件与多模态输入如何进入主循环

## 路线三：想理解“安全边界到底落在哪里”

按这个顺序继续精读：

1. `recovered_source/src/tools/FileReadTool/FileReadTool.ts`
2. `recovered_source/src/tools/BashTool/BashTool.tsx`
3. `recovered_source/src/tools/AgentTool/built-in/exploreAgent.ts`
4. `recovered_source/src/tools/AgentTool/runAgent.ts`
5. `recovered_source/src/utils/permissions/` 目录

重点关注：

- 输入校验发生在工具的哪一层
- 只读/高风险工具如何做限制
- 子代理为什么也要被角色化裁剪
- 权限模式如何影响最终可用能力

## 路线四：想理解“长期状态与协作能力”

按这个顺序继续精读：

1. `recovered_source/src/utils/tasks.ts`
2. `recovered_source/src/utils/settings/settings.ts`
3. `recovered_source/src/services/SessionMemory/sessionMemory.ts`
4. `recovered_source/src/utils/sessionStorage.ts`

重点关注：

- 为什么任务系统要落盘并加锁
- 配置为什么要多来源合并与缓存
- 记忆系统为什么不等同于 transcript
- 会话恢复和长期状态如何配合

## 路线五：想理解“Agent 为什么不是简单并发调用”

按这个顺序继续精读：

1. `recovered_source/src/tools/AgentTool/runAgent.ts`
2. `recovered_source/src/tools/AgentTool/built-in/exploreAgent.ts`
3. `recovered_source/src/tools/AgentTool/built-in/planAgent.ts`
4. `recovered_source/src/tools/AgentTool/loadAgentsDir.ts`
5. `recovered_source/src/tools.ts`

重点关注：

- agent 拿到的是怎样的工具池
- 父会话上下文如何被裁剪后传给子代理
- built-in agent 如何体现角色约束
- agent 与普通工具为什么共享同一能力治理体系

## 路线六：想理解“Claude Code 如何扩展自己”

按这个顺序继续精读：

1. `recovered_source/src/main.tsx`
2. `recovered_source/src/services/mcp/config.ts`
3. `recovered_source/src/services/mcp/client.ts`
4. `recovered_source/src/utils/plugins/loadPluginCommands.ts`
5. `recovered_source/src/skills/bundledSkills.ts`
6. `recovered_source/src/skills/loadSkillsDir.ts`
7. `recovered_source/src/commands.ts`

重点关注：

- MCP 服务器如何进入当前会话
- plugin commands / plugin skills 如何被加载
- plugin-provided MCP 如何并入统一配置流程
- bundled skills 与磁盘 skills 如何装配
- dynamic skills 如何随文件路径被发现和激活
- 这些扩展最终如何汇入命令表与工具池

## 推荐的二刷方法

第一遍看结构，第二遍看边界。

### 第一遍看结构

回答这些问题：

- 这个文件的职责是什么？
- 上游输入从哪里来？
- 下游输出交给谁？

### 第二遍看边界

回答这些问题：

- 它限制了什么？
- 它缓存了什么？
- 它在并发、权限、性能上保护了什么？

## 你现在应该已经建立的总体理解

如果这套教程起作用，你现在应该能用一句话描述这个项目：

> Claude Code 是一个以 QueryEngine 为会话核心、以工具系统为能力边界、以命令/输入系统为交互入口、以任务/设置/记忆系统为持续状态支撑、并通过 MCP / plugins / skills 持续扩展能力面的终端代理应用。

## 如果你要继续扩展这套教程

后续还可以继续补写这些专题：

- session restore / transcript 体系
- REPL UI 层与 Ink 组件树
- analytics / telemetry 与启动期预热机制
- worktree / background sessions / remote-control 相关能力
- 权限系统中的 hook / classifier / policy 交互细节

这些都值得单独拆成深挖章节。

## 返回目录

回到 [00-overview.md](./00-overview.md)。
