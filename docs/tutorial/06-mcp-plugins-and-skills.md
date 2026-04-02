# Claude Code 源码学习教程 06：MCP、插件与 Skills 深挖

[English Version](./06-mcp-plugins-and-skills_EN.md)

## 本章要解决的问题

读完这一章，你应该能回答：

- Claude Code 是如何接入外部扩展能力的？
- MCP、plugins、skills 三者分别是什么关系？
- 它们最终如何汇入命令系统、工具系统和启动主流程？

## 先建立总图

前面的章节主要解释了 Claude Code 自身如何启动、如何处理输入、如何组织工具与状态。这一章讨论的是它如何**继续扩展自己**。

你可以先把三者粗略理解成：

- **MCP**：把外部服务接入成工具、命令、资源的一层协议与运行时连接机制。
- **Plugins**：把第三方或本地扩展组织成一组可加载的命令、skills，甚至 MCP 服务器。
- **Skills**：把提示词化能力、命令式能力、目录内知识和上下文约束组织成一种高层交互入口。

它们不是彼此完全独立的三套系统，而是会在启动与命令装配阶段彼此连接。

## 一、启动阶段如何把它们接进来

### `main.tsx` 先注册内建 plugins 与 bundled skills

`recovered_source/src/main.tsx:1919-1928` 很关键：

- 它先执行 `initBuiltinPlugins()`
- 再执行 `initBundledSkills()`
- 然后才调用 `getCommands(preSetupCwd)`

这说明一个核心设计原则：

> 先把扩展来源注册好，再统一生成当前会话可见的命令表。

如果顺序反过来，命令装配就可能拿不到这些能力。

### MCP 连接与资源预取也在主装配期完成

`recovered_source/src/main.tsx:2704-2724` 以及相关导入显示，启动期还会调用：

- `getMcpToolsCommandsAndResources(...)`
- `prefetchAllMcpResources(...)`

这说明 MCP 不是一个“用户之后再手动触发的附属系统”，而是主程序装配流程中的正式组成部分。

## 二、MCP：外部能力如何变成工具、命令、资源

学习 MCP 最值得先看两组文件：

- 配置与策略：`recovered_source/src/services/mcp/config.ts`
- 连接与提取：`recovered_source/src/services/mcp/client.ts`

### 1. `config.ts` 负责“有哪些 MCP 服务器应该参与当前会话”

`recovered_source/src/services/mcp/config.ts:1071-1250` 的 `getClaudeCodeMcpConfigs(...)` 是核心入口之一。

它做的事情非常多：

- 读取 enterprise / user / project / local 等不同作用域配置
- 在 enterprise 配置存在时给予其排他优先级
- 根据 policy 限制过滤服务器
- 装载 plugin 提供的 MCP servers：`config.ts:1114-1154`
- 对 project servers 做审批过滤：`config.ts:1164-1170`
- 对 plugin servers 与手工配置 servers 做去重：`config.ts:1172-1229`
- 最后按优先级合并并再次做 policy filtering：`config.ts:1231-1248`

这说明 MCP 配置不是“读一个 JSON 然后连上去”这么简单，而是一个包含：

- 多作用域合并
- 插件来源接入
- 审批/策略过滤
- 重复服务器抑制

的完整配置治理层。

### 2. `parseMcpConfig(...)` 负责把外部配置变成可信内部结构

`recovered_source/src/services/mcp/config.ts:1297-1377` 的 `parseMcpConfig(...)` 会：

- 先跑 schema 校验
- 再做环境变量展开
- 记录缺失变量 warning
- 处理平台相关提示，例如 Windows 下 `npx` 包装

这说明 MCP 配置解析也是一个典型的“外部输入可信化”过程。

### 3. `client.ts` 负责真正连接 MCP 服务器并提取能力

`recovered_source/src/services/mcp/client.ts:2226-2403` 的 `getMcpToolsCommandsAndResources(...)` 是最值得精读的实现。

它会：

- 获取全部 MCP 配置：`client.ts:2237-2239`
- 先分离 disabled servers：`client.ts:2241-2254`
- 统计 transport 类型并区分 local / remote servers：`client.ts:2256-2271`
- 并发连接服务器：`client.ts:2282-2402`
- 针对已连接服务器提取：
  - `fetchToolsForClient(client)`
  - `fetchCommandsForClient(client)`
  - 资源支持下的 skills / resources

在 `client.ts:2344-2369`，你可以清楚看到 MCP 最终会产出：

- tools
- commands
- skills（如果 server 资源支持 skill 发现）
- resources

这就是 MCP 在 Claude Code 里的核心意义：

> 把外部能力规范化地翻译成 Claude Code 内部可以消费的工具、命令与资源对象。

## 三、Plugins：扩展包如何把命令、skills、MCP 带进系统

Plugin 相关最值得先看：

- `recovered_source/src/utils/plugins/loadPluginCommands.ts`
- `recovered_source/src/plugins/bundled/index.ts`
- `recovered_source/src/services/mcp/config.ts`（插件 MCP）

### 1. 当前 built-in plugins 是一个预留式启动点

`recovered_source/src/plugins/bundled/index.ts:1-23` 很短，但很重要。

它说明：

- built-in plugins 是 CLI 自带、可被注册到 `/plugin` 体系中的功能
- 目前还没有真正注册的内建插件
- 但框架已经预留好了启动入口 `initBuiltinPlugins()`

这意味着 plugin 机制在当前版本中不仅服务第三方，也在为未来内建功能模块化提供通道。

### 2. 插件命令是如何加载的

`recovered_source/src/utils/plugins/loadPluginCommands.ts:414-677` 的 `getPluginCommands()` 展示了 plugin command 的加载主流程：

- 只加载 enabled plugins：`loadPluginCommands.ts:422-429`
- 按 plugin 并行处理：`loadPluginCommands.ts:431-433`
- 从插件默认 commands 目录读取 markdown 命令
- 从 `commandsPaths` 指定的额外路径读取命令：`loadPluginCommands.ts:465-600`
- 支持直接从 metadata 提供 inline content：`loadPluginCommands.ts:603-667`

这说明 plugin command 的本质不是编译时函数，而是大量依赖：

- markdown 文件
- frontmatter
- manifest metadata
- 目录扫描与路径解析

换句话说，plugin command 更像是一种**内容驱动的命令扩展模型**。

### 3. 插件 skills 的加载与 plugin commands 类似，但语义更高层

同一个文件里，后半部分还负责 plugin skills 的加载。它会识别：

- 直接 skill 目录中的 `SKILL.md`
- 子目录里的 skill 结构

这说明在 Claude Code 里，plugin 不只是“加几个命令”，也可以直接把更高层的 skill 能力带进系统。

### 4. 插件还可以自带 MCP servers

回到 `recovered_source/src/services/mcp/config.ts:1114-1154`。

这里明确显示：

- 在收集 MCP 配置时，会先运行 `loadAllPluginsCacheOnly()`
- 然后从 enabled plugins 中收集 plugin MCP servers
- 将它们与手工配置、project/local/user 配置一起进入同一套 dedup / policy 流程

这说明 plugin 在 Claude Code 中不仅能提供“静态文档型命令/skills”，还能提供真正的外部连接型能力。

## 四、Skills：高层提示词能力如何进入命令系统

Skills 相关最值得看的文件是：

- `recovered_source/src/skills/bundledSkills.ts`
- `recovered_source/src/skills/loadSkillsDir.ts`
- `recovered_source/src/commands.ts`

### 1. Bundled skills：编译进 CLI 的 skills

`recovered_source/src/skills/bundledSkills.ts:11-108` 说明了 bundled skill 的定义与注册机制：

- `BundledSkillDefinition` 描述 skill 的字段
- `registerBundledSkill(...)` 把 skill 注册进内部数组
- `getBundledSkills()` 返回当前 bundled skills 集合

更有意思的是 `bundledSkills.ts:30-36` 与 `:59-72`：

- skill 可以自带 reference files
- 首次调用时可被提取到磁盘
- prompt 会自动加上 base directory 提示

这说明 skill 不是纯 prompt 字符串，而是可以附带一小组“可供 Read/Grep 的参考材料”。

### 2. `loadSkillsDir.ts`：磁盘上的 skills 如何装配进系统

`recovered_source/src/skills/loadSkillsDir.ts:625-803` 的 `getSkillDirCommands(cwd)` 是技能目录装配的关键入口。

它会从多种来源加载 skills：

- managed skills 目录
- user skills 目录
- project skills 目录
- 额外添加的目录
- legacy commands 目录

然后继续做：

- 去重：`loadSkillsDir.ts:725-769`
- 区分 unconditional / conditional skills：`loadSkillsDir.ts:771-803`

这说明 skills 不是单纯扫描目录，而是要经历一轮完整的**来源合并 + 去重 + 条件激活准备**。

### 3. 动态 skill 发现

更值得注意的是 `loadSkillsDir.ts:861-1038`：

- `discoverSkillDirsForPaths(...)` 会沿文件路径向上查找 `.claude/skills`
- `addSkillDirectories(...)` 会把这些目录中的 skills 动态装入当前会话
- `getDynamicSkills()` 返回当前会话中动态发现的技能
- `activateConditionalSkillsForPaths(...)` 会按 paths frontmatter 激活条件技能

这说明 Claude Code 的 skills 系统不是“启动时一次性装好就结束”，而是会随着当前会话接触到的文件路径继续发现和激活。

## 五、它们最后如何汇入命令系统

`recovered_source/src/commands.ts:353-509` 把整个故事真正串起来了。

这里的命令最终来自多种来源：

- skill 目录命令
- plugin skills
- bundled skills
- builtin plugin skills
- plugin commands
- workflow commands
- 内建 commands
- dynamic skills

也就是说，MCP、plugins、skills 最终的落点之一，就是 `getCommands(cwd)` 返回的**当前会话命令表**。

而 MCP 另一条更底层的落点，则是生成工具与资源并汇入主程序运行时。

## 六、把三者放在同一个图里看

你可以用下面这张简化图理解：

```text
startup/main.tsx
  -> initBuiltinPlugins()
  -> initBundledSkills()
  -> getCommands(cwd)
  -> getMcpToolsCommandsAndResources(...)

plugins
  -> plugin commands
  -> plugin skills
  -> plugin MCP servers

skills
  -> bundled skills
  -> user/project/managed skills
  -> dynamic/conditional skills

mcp
  -> external servers
  -> tools + commands + resources + mcp skills

all of the above
  -> 汇入命令系统 / 工具系统 / 主会话上下文
```

## 本章总结

这一章最重要的结论是：

> Claude Code 的扩展能力不是靠单一插件点完成的，而是通过 MCP、plugins、skills 三套互相连接的机制，共同扩展命令、工具、资源与提示词能力。

其中：

- MCP 更偏外部服务接入与运行时连接
- Plugins 更偏打包式扩展分发
- Skills 更偏高层交互与提示词化能力组织

三者最终都被主程序吸纳进当前会话的命令表、工具池与上下文装配流程中。

## 下一章

继续阅读 [99-reading-paths-and-next-steps.md](./99-reading-paths-and-next-steps.md)。
