# Claude Code Source Learning Tutorial 00: Overview and Reading Map

[中文版本](./00-overview.md)

## Who this tutorial is for

This tutorial series is aimed at two kinds of readers:

1. People who want to understand how a terminal-first agentic CLI like Claude Code is structured.
2. People who want to use this repository as a source-analysis sample to study startup flow, the tool system, the task system, the settings system, and the extension surfaces.

## First, what kind of repository is this?

This directory is not Anthropic’s public upstream development repository. It is an **analysis directory built from a packaged release artifact plus a recovered source view**. The repository itself already states this clearly: `README_EN.md:12-23` frames it as a research and tutorial-oriented analysis repository, and `README_EN.md:153-156` explains that `recovered_source/` is a recovered source view derived from the packaged artifact and mapping information.

So you should read this project with two assumptions in mind:

- What you see is a **recovered source view suitable for research**, not the original upstream development layout.
- It is especially valuable for **architecture understanding, runtime flow analysis, tool behavior analysis, and extension-system learning**, not for treating it as the official development base.

## The top-level package layout

Combining `README_EN.md:37-58` with `package.json:2-13`, the top-level layout can be remembered like this:

```text
package/
├── cli.js                # bundled runtime CLI entry point
├── cli.js.map            # source map used to recover source structure
├── package.json          # npm package metadata and bin definition
├── sdk-tools.d.ts        # tool interface type definitions
├── docs/tutorial/        # tutorial documentation
├── vendor/               # packaged native/helper binaries
└── recovered_source/     # recovered source view derived from the bundle
```

The most useful areas for learning are:

- `package.json:2-13`: confirms this is `@anthropic-ai/claude-code` and that `cli.js` is the binary entry.
- `README_EN.md`: repository positioning, research boundary, and tutorial entry points.
- `recovered_source/src/`: the main study surface.

## How to think about this project

If you summarize Claude Code in one sentence, a good mental model is:

> A terminal-first, LLM-driven session executor that assembles command routing, tools, permissions, persistence, and sub-agents into one interactive runtime, while continuing to expand through MCP, plugins, and skills.

It is not just a one-shot command-line tool. It behaves much more like a long-running interactive application. The repository description already hints at this: `README_EN.md:204-212` describes REPL-style interaction, permission prompts, session recovery, skill/plugin loading, MCP integration, and background task features.

## Recommended reading order

This tutorial is organized from the outside inward:

1. **Startup layer**: how the CLI enters the main runtime.
2. **Main loop layer**: how one user input becomes a model/tool/state cycle.
3. **Interaction layer**: how slash commands, hooks, and attachments enter the system.
4. **Tool layer**: why Claude Code can read files, edit files, run commands, and spawn sub-agents.
5. **Persistence layer**: how tasks, settings, and memory are stored and reused.
6. **Extension layer**: how MCP, plugins, and skills plug into Claude Code.
7. **Security layer**: how permission modes, path protection, shell safety, and workflow boundaries work together.

Recommended chapter order:

- [01-bootstrap-and-startup_EN.md](./01-bootstrap-and-startup_EN.md)
- [02-query-engine-and-conversation-loop_EN.md](./02-query-engine-and-conversation-loop_EN.md)
- [03-input-commands-and-hooks_EN.md](./03-input-commands-and-hooks_EN.md)
- [04-tools-and-agent-execution_EN.md](./04-tools-and-agent-execution_EN.md)
- [05-state-tasks-settings-and-memory_EN.md](./05-state-tasks-settings-and-memory_EN.md)
- [06-mcp-plugins-and-skills_EN.md](./06-mcp-plugins-and-skills_EN.md)
- [07-permissions-and-security-boundaries_EN.md](./07-permissions-and-security-boundaries_EN.md)
- [99-reading-paths-and-next-steps_EN.md](./99-reading-paths-and-next-steps_EN.md)

## A practical mental model

When reading the source, it helps to keep this main thread in mind:

```text
CLI entry
  -> main runtime initialization
  -> command/tool/permission/context assembly
  -> accept user input
  -> produce model request
  -> execute tool calls
  -> update session state
  -> extend capabilities through MCP / plugins / skills
  -> continue the next turn
```

Mapped to key files:

- Thin CLI entry: `recovered_source/src/entrypoints/cli.tsx:28-217`
- Main assembly entry: `recovered_source/src/main.tsx:585-719`
- Conversation core: `recovered_source/src/QueryEngine.ts:175-344`
- Input preprocessing: `recovered_source/src/utils/processUserInput/processUserInput.ts:140-309`
- Command assembly: `recovered_source/src/commands.ts:353-509`
- Tool assembly: `recovered_source/src/tools.ts:179-362`
- Task system: `recovered_source/src/utils/tasks.ts:199-438`
- Settings system: `recovered_source/src/utils/settings/settings.ts:74-220`
- Session memory: `recovered_source/src/services/SessionMemory/sessionMemory.ts:134-319`
- MCP / plugins / skills: see `recovered_source/src/services/mcp/`, `recovered_source/src/utils/plugins/`, and `recovered_source/src/skills/`

## The most valuable things to observe while studying the code

### 1. Model capability is tightly constrained at the tool surface

The model is not granted direct system authority. Instead, `tools.ts` assembles a controlled tool set, and permission rules narrow it further.

### 2. User interaction is treated as a separate orchestration problem

User input is not always plain text. It may also be a slash command, additional context, an image, hook-injected content, or a system-generated message.

### 3. Persistent conversation state is a core feature, not an add-on

`QueryEngine` does not simply execute a single API call. It maintains messages, cached file reads, permission denials, budgets, and other turn-spanning state.

### 4. Engineering safety boundaries are embedded in tool implementations

For example, the file-reading tool blocks unsafe device files, Bash contains semantic classification and read-only constraints, and Explore-style agents are explicitly limited to read-only work.

### 5. Extensibility exists in multiple layers

Claude Code does not rely only on built-in commands and tools. MCP, plugins, and skills continue to expand the available commands, tools, resources, and prompt surfaces.

## Suggested reading method

For each chapter, try answering these three questions:

1. **What is this file responsible for in the system?**
2. **Where does its input come from, and where does its output go?**
3. **What boundary is it protecting: performance, permissions, caching, or state consistency?**

If you read with those questions in mind, this recovered source view becomes much easier to turn into a coherent mental model than simply browsing files top to bottom.

## Next chapter

Continue to [01-bootstrap-and-startup_EN.md](./01-bootstrap-and-startup_EN.md).
