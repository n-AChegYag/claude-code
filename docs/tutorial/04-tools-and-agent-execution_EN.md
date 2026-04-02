# Claude Code Source Learning Tutorial 04: Tools and Agent Execution

[中文版本](./04-tools-and-agent-execution.md)

## Questions this chapter answers

After reading this chapter, you should be able to answer:

- Why does Claude Code need a tool system at all?
- How are tools registered, filtered, and exposed to the model?
- What is the relationship between sub-agents and ordinary tools?

## The tool system is Claude Code’s operational surface

Claude Code does not give the model direct file-system or terminal authority. Instead, it wraps capabilities inside explicit tools and exposes those tools under controlled runtime rules.

The most important entry file for learning this layer is:

- `recovered_source/src/tools.ts`

This file is not the implementation of one specific tool. It is the assembly point for the entire tool pool.

## `tools.ts` first answers: what tools can the system have?

### The base tool universe

`recovered_source/src/tools.ts:193-250`, in `getAllBaseTools()`, is the central source of truth for the built-in tool universe.

It assembles many capabilities into one list, including:

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
- the `Task*` tool family
- MCP-related tools
- multiple feature-gated runtime tools

That means Claude Code’s capability model is not “a few hardcoded functions.” It is a **tool pool assembled from environment conditions, feature gates, and runtime settings**.

## Why the tool pool is not fixed

From `tools.ts:193-250`, you can see at least three kinds of narrowing:

### 1. Feature-gate filtering

Some tools are only added when specific features are enabled, such as:

- `SleepTool`
- `RemoteTriggerTool`
- `WorkflowTool`
- `WebBrowserTool`
- `SnipTool`

### 2. Environment filtering

For example:

- certain tools only exist for `process.env.USER_TYPE === 'ant'`
- `ENABLE_LSP_TOOL` controls `LSPTool`
- `isWorktreeModeEnabled()` controls worktree tools

### 3. Permission-context filtering

`recovered_source/src/tools.ts:262-327`, through `filterToolsByDenyRules(...)` and `getTools(...)`, further narrows the tool set based on the active permission context.

So the real rule is:

> a tool being declared in the system does not mean it is automatically available in the current session

## Why `getTools()` matters

`recovered_source/src/tools.ts:271-327`, in `getTools(...)`, is one of the most important places to read.

It performs several layers of filtering:

1. handles simple mode: `tools.ts:271-298`
2. filters out denied/special tools: `tools.ts:300-310`
3. hides primitive tools when REPL mode wraps them: `tools.ts:312-323`
4. applies final `isEnabled()` filtering: `tools.ts:325-326`

That means the tool list seen by the model has already passed through:

- mode constraints
- permission constraints
- runtime enablement constraints

## A representative tool: `FileReadTool`

When learning the tool layer, do not try to read the entire `tools/` directory at once. The better strategy is to pick a few representative tools.

### Why `FileReadTool` is a good first example

It captures several key design principles at once:

- capability is explicit
- input/output are structured
- safety boundaries are enforced early
- UI rendering and model-facing result serialization are separate concerns

### `FileReadTool` is not just “read a file”

In `recovered_source/src/tools/FileReadTool/FileReadTool.ts:337-495`, the tool definition includes:

- `name`
- `description()` / `prompt()`
- input and output schemas
- activity descriptions
- read-only declaration: `isReadOnly()` at `FileReadTool.ts:376-378`
- permission checks
- UI rendering behavior
- input validation

So a tool implementation in Claude Code often has four responsibilities at once:

1. **the interface the model sees**
2. **the safety and permission rules the runtime sees**
3. **the display behavior the UI sees**
4. **the actual execution logic the runtime performs**

### Safety boundaries live inside the tool implementation

`FileReadTool` is especially valuable because it embeds several safety checks directly into the tool layer:

- blocks dangerous device-file paths: `FileReadTool.ts:98-127`
- rejects denied paths during validation: `FileReadTool.ts:442-458`
- rejects unsupported binary file reads: `FileReadTool.ts:469-482`
- explicitly blocks device files again during validation: `FileReadTool.ts:484-492`
- throws `MaxFileReadTokenExceededError` when token limits are exceeded: `FileReadTool.ts:175-183` and `FileReadTool.ts:770`

This is a strong hint that the tool layer itself is part of the system’s safety boundary, not just a thin feature wrapper.

## Another representative tool: `BashTool`

`recovered_source/src/tools/BashTool/BashTool.tsx` is a strong example of how Claude Code governs higher-risk tools.

From `BashTool.tsx:54-81`, you can see that it first defines several command-semantic categories:

- search commands
- read commands
- listing commands
- semantically neutral commands
- silent commands

That tells you Bash execution is not treated as a raw string passed into the shell. The runtime first classifies command intent so it can:

- present better UI summaries
- distinguish read-like operations from write-like operations
- enforce read-only constraints
- manage backgrounding and output handling more safely

So the hard part of BashTool is not just shell execution. It is **turning a dangerous capability into something structured, explainable, and governable**.

## Why sub-agents are also part of the tool story

It is easy to think of agents as something separate from tools, but in Claude Code they are tightly coupled to the same capability surface.

### `runAgent.ts` shows what a sub-agent really is

`recovered_source/src/tools/AgentTool/runAgent.ts:248-329` defines the main parameters for `runAgent(...)`. A sub-agent starts with inputs such as:

- the agent definition
- prompt messages
- tool-use context
- `canUseTool`
- `availableTools`
- `allowedTools`
- model choice
- `maxTurns`
- `worktreePath`
- `transcriptSubdir`

That means a sub-agent is not “some mysterious second model instance.” It is:

> a constrained execution unit launched under the parent session’s context, permissions, and tool-governance model

### Sub-agents inherit context, but not blindly

`runAgent.ts:380-410` is worth reading carefully:

- it resolves user/system context for the agent
- for read-only agents like Explore and Plan, it intentionally omits some broader parent context such as `claudeMd` and `gitStatus`

So the agent runtime tries to do two things at once:

1. preserve relevant continuity with the parent session
2. avoid copying unnecessary high-volume context into sub-agents

### Explore agent as a clean read-only example

`recovered_source/src/tools/AgentTool/built-in/exploreAgent.ts:24-83` is especially educational because it defines the role of the agent in very explicit terms:

- it is a file-search specialist
- it is explicitly READ-ONLY
- it forbids creation / modification / deletion
- it is told to focus on searching and analysis only
- `disallowedTools` excludes edit / write / notebook-edit / exit-plan-mode

This shows that Claude Code’s sub-agents are not designed under the rule “stronger is better.” They are **role-shaped and intentionally capability-constrained**.

## The deeper value of the tool system

By this point, the core engineering idea should be clear:

> model capability in Claude Code does not come from raw authority. It comes from toolization, permission filtering, and role shaping.

More concretely:

- toolization turns capabilities into structured interfaces
- permission filtering narrows the available surface for the session
- role shaping narrows that surface again for sub-agents

## Recommended deep-reading path

### Step 1: read tool-pool assembly first

- `recovered_source/src/tools.ts:193-327`

Question to answer: **what tools exist, and why are they not all exposed at once?**

### Step 2: read `FileReadTool`

- `recovered_source/src/tools/FileReadTool/FileReadTool.ts:337-495`

Question to answer: **how can one tool carry interface definition, validation, permissions, UI behavior, and execution logic at the same time?**

### Step 3: supplement with Bash and Agent

- `recovered_source/src/tools/BashTool/BashTool.tsx:54-81`
- `recovered_source/src/tools/AgentTool/runAgent.ts:248-410`
- `recovered_source/src/tools/AgentTool/built-in/exploreAgent.ts:24-83`

Question to answer: **how are high-risk operations and sub-agent roles both governed through the same tool-centric model?**

## Chapter summary

The most important conclusion from this chapter is:

> the tool system is the capability boundary of Claude Code.

The model can read files, edit files, run commands, and launch sub-agents not because it has direct raw access to those things, but because the system packages those abilities into tools and then exposes them through mode filtering, permission rules, and role-specific constraints.

## Next chapter

Continue to [05-state-tasks-settings-and-memory_EN.md](./05-state-tasks-settings-and-memory_EN.md).
