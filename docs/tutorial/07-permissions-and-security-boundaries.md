# Claude Code 源码学习教程 07：权限系统与安全边界深挖

[English Version](./07-permissions-and-security-boundaries_EN.md)

## 本章要解决的问题

读完这一章，你应该能回答：

- Claude Code 为什么把权限系统做成一个独立的架构层？
- permission mode、allow / ask / deny 规则是如何共同工作的？
- 文件系统、shell、plan mode、sub-agents 的安全边界分别落在哪里？

## 为什么权限系统是第一等架构层

前面的章节已经说明 Claude Code 是一个工具驱动型终端代理。只要系统允许模型读文件、改文件、跑 shell、开子代理，就一定要回答两个问题：

1. **什么能力可以被暴露给模型？**
2. **在什么条件下这些能力才允许执行？**

因此，权限系统在 Claude Code 里不是一个 UI 上的附属确认框，而是贯穿：

- 工具可见性
- 工具执行前校验
- 模式切换
- 文件路径保护
- shell 风险收敛
- plan mode 与 sub-agent 行为限制

的一整层控制体系。

## 一、Permission Mode：系统先决定“默认行为模式”

最先要看的文件是：

- `recovered_source/src/utils/permissions/PermissionMode.ts`

`PermissionMode.ts:42-90` 展示了 Claude Code 里几个核心模式：

- `default`
- `plan`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `auto`（在特性开启时出现）

这说明 Claude Code 不是只靠“每次都 ask / allow / deny”来治理行为，而是先给当前会话设定一种**整体权限模式**。

### 这些模式在语义上意味着什么

可以先这样理解：

- **default**：常规模式，按标准权限流程处理工具请求。
- **plan**：先规划、后执行，强调探索和设计，不应该直接进入实现修改。
- **acceptEdits**：对一部分典型文件系统修改命令给出更宽松的自动通过策略。
- **bypassPermissions**：跳过大部分权限阻拦，是高风险模式。
- **dontAsk**：尽量不弹出询问，依赖主流程中的策略处理。
- **auto**：和分类器/自动化判定相关联，进入时还要剥离危险规则。

这里最重要的点是：

> mode 决定的是系统面对工具请求时的“基准姿态”，而不是某个单一工具的实现细节。

## 二、危险规则剥离：为什么“允许规则”本身也可能有风险

这一层最值得看的是：

- `recovered_source/src/utils/permissions/permissionSetup.ts`

### 1. 危险规则不是只针对 deny，也针对 allow

很多人直觉会觉得只有“放开 bypass”才危险，但 Claude Code 的实现更细：

`permissionSetup.ts:472-503` 的 `removeDangerousPermissions(...)` 和 `permissionSetup.ts:510-552` 的 `stripDangerousPermissionsForAutoMode(...)` 表明：

- 某些 allow 规则本身就足以绕过分类器或过度放权
- 进入 auto mode 时，这些危险规则会先被剥离
- 剥离后的规则会先暂存到 `strippedDangerousRules`

也就是说，Claude Code 不仅问“当前是不是危险模式”，还会问：

> 你之前保存下来的允许规则，会不会在当前模式下直接把保护层打穿？

### 2. 离开 auto mode 时会恢复这些规则

`permissionSetup.ts:561-579` 的 `restoreDangerousPermissions(...)` 会在离开 auto mode 时把之前暂存的规则恢复回来。

这说明权限系统不是简单粗暴地永久删除规则，而是根据当前 mode 动态调整可生效的规则集合。

### 3. mode 切换本身就是一套状态机

`permissionSetup.ts:597-646` 的 `transitionPermissionMode(...)` 很关键。

这里把几类副作用集中处理：

- plan mode 进入 / 退出
- auto mode 激活 / 停用
- 危险规则剥离 / 恢复
- 退出 plan mode 时的状态清理

这说明 permission mode 不只是一个字符串，而是一组**伴随状态转换的行为规则**。

## 三、文件系统边界：哪些路径天然更敏感

权限/安全边界里最有代表性的文件是：

- `recovered_source/src/utils/permissions/filesystem.ts`

### 1. 敏感文件与敏感目录是显式列举的

`filesystem.ts:53-79` 直接定义了：

- `DANGEROUS_FILES`
  - 如 `.gitconfig`、`.bashrc`、`.zshrc`、`.mcp.json`、`.claude.json`
- `DANGEROUS_DIRECTORIES`
  - 如 `.git`、`.vscode`、`.idea`、`.claude`

这些路径之所以敏感，是因为它们可能影响：

- 用户 shell 行为
- Git 行为
- Claude Code 自身配置
- IDE 配置
- 外部连接与加载行为

也就是说，这一层在防的不是普通文本文件改动，而是**可以改变执行环境、配置环境、甚至扩展加载面的路径**。

### 2. 自动编辑会专门避开这些敏感路径

`filesystem.ts:435-478` 的 `isDangerousFilePathToAutoEdit(...)` 会判断一个路径是否属于不应被自动编辑的危险范围。

这说明 Claude Code 的态度不是“先让模型改，再让用户发现问题”，而是：

- 对高风险文件和目录提前做特殊保护
- 尤其防止自动通过的修改触碰这些区域

### 3. `.claude` 目录有特别高的防护优先级

从 `filesystem.ts:224-241`、`:1242` 附近的注释也能看出，Claude Code 对自己的配置目录和相关命令/agents/skills 区域会格外小心。

这是一个很重要的安全思想：

> 系统必须对“能改变系统自身行为的配置文件”保持更严格的权限边界。

## 四、Shell 安全：为什么 Bash 权限不是简单字符串匹配

Bash 是最容易把边界打穿的工具，所以它的权限实现也最复杂。最值得看的文件是：

- `recovered_source/src/tools/BashTool/bashPermissions.ts`
- `recovered_source/src/tools/BashTool/modeValidation.ts`

### 1. `getSimpleCommandPrefix(...)` 不是为了好看，而是为了安全匹配

`bashPermissions.ts:148-188` 的 `getSimpleCommandPrefix(...)` 会尝试把命令抽成更稳定的前缀，例如：

- `git commit -m ...` -> `git commit`
- `NODE_ENV=prod npm run build` -> `npm run`

但它有一个关键限制：

- 只有遇到 `SAFE_ENV_VARS` 里的环境变量，才允许忽略这些前缀
- 一旦遇到不安全的环境变量，就直接返回 `null`

这意味着 Claude Code 很明确地知道：

> 某些环境变量不是装饰性的，它们本身就可能改变代码加载、模块搜索、动态库注入甚至命令执行语义。

### 2. 为什么要维护 `SAFE_ENV_VARS`

`bashPermissions.ts:368-430` 定义了 `SAFE_ENV_VARS`，并在注释里明确禁止把这些变量加入白名单：

- `PATH`
- `LD_PRELOAD`
- `LD_LIBRARY_PATH`
- `PYTHONPATH`
- `NODE_OPTIONS`
- 等等

这说明 shell 权限匹配并不是只看命令名，还要考虑：

- 某些环境变量会把“看起来一样的命令”变成完全不同的执行结果

### 3. 为什么 `BARE_SHELL_PREFIXES` 很重要

`bashPermissions.ts:196-226` 定义了 `BARE_SHELL_PREFIXES`，其中包括：

- `sh`
- `bash`
- `zsh`
- `cmd`
- `powershell`
- `env`
- `xargs`
- `nice`
- `nohup`
- `timeout`
- `sudo`
- `doas`
- `pkexec`

这代表 Claude Code 明确把这些命令视为高风险 wrapper / shell / 提权入口，不允许简单地把它们当成“安全前缀”去自动匹配。

背后的安全逻辑很直接：

- `bash -c ...` 可以执行任意内容
- `env ...` 可以包裹执行上下文
- `sudo` / `pkexec` 涉及权限提升
- 某些 wrapper 会掩盖真正被执行的命令

### 4. acceptEdits mode 不是无限放权

`recovered_source/src/tools/BashTool/modeValidation.ts:7-15` 列出了 `acceptEdits` 模式下可自动通过的一小组文件系统命令：

- `mkdir`
- `touch`
- `rm`
- `rmdir`
- `mv`
- `cp`
- `sed`

而 `modeValidation.ts:37-49` 只在 `acceptEdits` 模式且命令属于这组范围时，才返回自动 allow。

这说明：

> acceptEdits 不是“让 Bash 全开”，而是只对一部分被明确认定为编辑型、文件系统型的命令做特判。

## 五、Plan mode：为什么“先计划”本身也是一种安全边界

计划模式看起来像流程设计，但其实也有明显的权限意义。关键文件是：

- `recovered_source/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`
- `recovered_source/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`

### 1. EnterPlanMode 明确把流程切到“先探索、后实现”

`EnterPlanModeTool.ts:77-101` 会在进入 plan mode 时：

- 拒绝在 agent context 内使用
- 调用 `prepareContextForPlanMode(...)`
- 把 permission mode 改成 `plan`

而它返回给模型的指令还会明确强调：

- 此阶段要以探索和设计为主
- 不应该立即写/改文件

这说明 plan mode 不是单纯的 UX 约束，而是把当前会话推入一种**更保守、更偏分析的操作状态**。

### 2. ExitPlanMode 会要求用户确认并恢复模式状态

`ExitPlanModeV2Tool.ts:195-238` 展示了退出 plan mode 时的几个关键限制：

- 只有在当前确实处于 plan mode 时才允许退出
- 非 teammate 情况下要向用户请求确认：`ExitPlanModeV2Tool.ts:233-238`

而 `ExitPlanModeV2Tool.ts:318-345` 又处理了：

- 退出时如何恢复 mode
- 如果 auto gate 不再可用，如何回退到更安全的默认状态

这说明 plan mode 的退出也不是“点一下就结束”，而是一个受控的状态恢复过程。

## 六、Sub-agents：权限规则连 agent 自己也能过滤

如果你以为权限系统只限制 Bash / Read / Edit 这类工具，那 `recovered_source/src/utils/permissions/permissions.ts` 会告诉你：agent 本身也在规则体系内。

### 1. 可以直接 deny 某种 agent 类型

`permissions.ts:305-320` 的 `getDenyRuleForAgent(...)` 允许通过 `Agent(Explore)` 这类规则直接拒绝某种 agent。

### 2. 系统会在 agent 列表层面做过滤

`permissions.ts:323-343` 的 `filterDeniedAgents(...)` 会把被 deny 的 agent type 从候选列表中移掉。

这说明子代理不是“额外隐形能力”，而是：

- 同样要经过 permission rule 过滤
- 同样属于系统能力表面的一部分

### 3. headless / async agents 还有 hook 参与

`permissions.ts:392-419` 的 `runPermissionRequestHooksForHeadlessAgent(...)` 表明：

- 对无法显示权限 UI 的 agent，系统会先给 hooks 一个决策机会
- 如果 hooks 没有明确决策，才进入后续自动拒绝路径

这说明 Claude Code 还考虑到了：

> 在没有本地交互界面的 agent 环境里，权限请求应该如何被优雅处理。

## 七、把权限系统放回整体架构里看

你可以把 Claude Code 的权限 / 安全边界理解成下面几层：

```text
permission mode
  -> 决定当前会话的默认行为姿态

permission rules
  -> allow / ask / deny 规则进一步收窄或放开能力

filesystem boundaries
  -> 保护危险文件、危险目录、Claude 自身配置区域

shell safety
  -> 限制环境变量剥离、wrapper 命令、提权入口、模式特判

workflow boundaries
  -> plan mode / sub-agents / headless hooks 控制复杂流程中的权限行为
```

这几层叠加起来，Claude Code 才能在“强能力工具”与“可控风险”之间找到平衡。

## 推荐精读路径

如果你想继续深挖权限系统，建议按下面顺序看：

1. `recovered_source/src/utils/permissions/PermissionMode.ts`
2. `recovered_source/src/utils/permissions/permissionSetup.ts`
3. `recovered_source/src/utils/permissions/filesystem.ts`
4. `recovered_source/src/tools/BashTool/bashPermissions.ts`
5. `recovered_source/src/tools/BashTool/modeValidation.ts`
6. `recovered_source/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`
7. `recovered_source/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`
8. `recovered_source/src/utils/permissions/permissions.ts`

## 本章总结

这一章最重要的结论是：

> Claude Code 的权限系统不是一个弹窗层，而是一套贯穿模式、规则、路径校验、shell 安全和多阶段工作流的架构性控制系统。

它真正保护的，不只是“某次工具调用能不能执行”，而是：

- 当前会话能暴露多大能力面
- 哪些路径不该被自动触碰
- 哪些 shell 规则可能绕过边界
- 复杂工作流是否还维持在安全的默认姿态下

## 下一章

继续阅读 [99-reading-paths-and-next-steps.md](./99-reading-paths-and-next-steps.md)。
