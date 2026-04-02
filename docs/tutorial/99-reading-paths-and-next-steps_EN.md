# Claude Code Source Learning Tutorial 99: Reading Paths and Next Steps

[中文版本](./99-reading-paths-and-next-steps.md)

If you have already finished the earlier chapters, here are several deeper reading paths depending on what you want to understand next.

## Path 1: understand why Claude Code feels like a terminal application shell

Read in this order:

1. `recovered_source/src/main.tsx`
2. `recovered_source/src/QueryEngine.ts`
3. `recovered_source/src/query.ts`
4. `recovered_source/src/tools.ts`

Focus on:

- what systems are assembled during startup
- how QueryEngine holds conversation state
- how the query loop handles tools and token budgets
- how the tool pool shrinks under permission and feature constraints

## Path 2: understand why the interaction layer is this complex

Read in this order:

1. `recovered_source/src/commands.ts`
2. `recovered_source/src/utils/processUserInput/processUserInput.ts`
3. `recovered_source/src/utils/messages.ts`
4. `recovered_source/src/utils/attachments.ts`

Focus on:

- how slash commands are aggregated
- how hooks intercept or enrich context
- how input is normalized into messages
- how attachments and multimodal input enter the main loop

## Path 3: understand where the safety boundaries actually live

Read in this order:

1. `recovered_source/src/tools/FileReadTool/FileReadTool.ts`
2. `recovered_source/src/tools/BashTool/BashTool.tsx`
3. `recovered_source/src/tools/AgentTool/built-in/exploreAgent.ts`
4. `recovered_source/src/tools/AgentTool/runAgent.ts`
5. `recovered_source/src/utils/permissions/`

Focus on:

- where input validation happens inside tools
- how read-only and higher-risk tools are constrained
- why sub-agents are also role-shaped
- how permission modes affect the final available capability set

## Path 4: understand persistence and collaborative continuity

Read in this order:

1. `recovered_source/src/utils/tasks.ts`
2. `recovered_source/src/utils/settings/settings.ts`
3. `recovered_source/src/services/SessionMemory/sessionMemory.ts`
4. `recovered_source/src/utils/sessionStorage.ts`

Focus on:

- why the task system persists to disk and uses locks
- why configuration supports multi-source merge and caching
- why session memory is not the same thing as transcript history
- how session restore and longer-lived state reinforce each other

## Path 5: understand why agents are not just “parallel calls”

Read in this order:

1. `recovered_source/src/tools/AgentTool/runAgent.ts`
2. `recovered_source/src/tools/AgentTool/built-in/exploreAgent.ts`
3. `recovered_source/src/tools/AgentTool/built-in/planAgent.ts`
4. `recovered_source/src/tools/AgentTool/loadAgentsDir.ts`
5. `recovered_source/src/tools.ts`

Focus on:

- what kind of tool pool an agent receives
- how parent context is selectively inherited
- how built-in agent roles are constrained
- why agents and ordinary tools share the same capability-governance model

## Path 6: understand how Claude Code extends itself

Read in this order:

1. `recovered_source/src/main.tsx`
2. `recovered_source/src/services/mcp/config.ts`
3. `recovered_source/src/services/mcp/client.ts`
4. `recovered_source/src/utils/plugins/loadPluginCommands.ts`
5. `recovered_source/src/skills/bundledSkills.ts`
6. `recovered_source/src/skills/loadSkillsDir.ts`
7. `recovered_source/src/commands.ts`

Focus on:

- how MCP servers enter the current runtime
- how plugin commands and plugin skills are loaded
- how plugin-provided MCP participates in the same config pipeline
- how bundled and on-disk skills are assembled
- how dynamic skills are discovered and activated from file-path interaction
- how these extension surfaces ultimately feed into commands and tools

## A good second-pass reading method

First pass: structure. Second pass: boundaries.

### First pass: read for structure

Try answering:

- what is this file responsible for?
- where does its input come from?
- where does its output go?

### Second pass: read for boundaries

Try answering:

- what does it constrain?
- what does it cache?
- what does it protect in terms of concurrency, permissions, or performance?

## The overall understanding you should now have

If this tutorial has done its job, you should now be able to describe the project like this:

> Claude Code is a terminal agent application whose conversation core is QueryEngine, whose capability boundary is the tool system, whose interaction entry surface is the command/input pipeline, whose long-lived continuity is supported by tasks/settings/memory, and whose capability surface keeps expanding through MCP, plugins, and skills.

## If you want to continue expanding this tutorial set

Good future deep-dive topics include:

- the permission system and security policy model
- session restore / transcript mechanisms
- the REPL UI layer and Ink component tree
- analytics / telemetry and startup prefetch behavior
- worktree / background sessions / remote-control features

Each of those would support a dedicated chapter.

## Return to the index

Go back to [00-overview_EN.md](./00-overview_EN.md).
