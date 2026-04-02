# Claude Code Package Research Notes

[中文版本](./README.md)

> [!IMPORTANT]
> **The contents of this repository are provided for study, research, and technical analysis only.**
>
> **All ownership, copyright, and all other rights related to the code and associated materials belong to Anthropic.**
>
> This repository does not claim, transfer, or redistribute the original Claude Code codebase or its intellectual property.

## Overview

This directory is best understood as a **research snapshot of the packaged `@anthropic-ai/claude-code` distribution**, not as the complete official upstream development repository maintained by Anthropic.

Based on the current file layout, it is primarily useful for:

- studying the Claude Code package structure
- analyzing CLI startup and module organization
- reviewing tool interfaces and type definitions
- understanding its terminal-first agent workflow
- learning the system through repository-local tutorial documentation

In other words, this repository now serves not only as a research sample, but also as a **tutorial-oriented analysis repository** built on top of the recovered source view.

## Repository Positioning

It is best to think of this project as a combination of:

1. **an npm release artifact research sample**
2. **a recovered-source analysis directory**
3. **a Claude Code learning and tutorial documentation repository**

If your goal is to:

- learn how Claude Code boots and initializes
- understand QueryEngine, the tool system, the command system, and the persistence layers
- study extension surfaces such as MCP, plugins, and skills

then this repository is already a very useful reading base.

If your goal is to work from Anthropic’s official upstream source repository, this should not be treated that way.

## Quick Facts

| Item | Details |
| --- | --- |
| Package name | `@anthropic-ai/claude-code` |
| Version | `2.1.88` |
| Runtime | Node.js `>=18.0.0` |
| Main entry | `cli.js` |
| Source map | `cli.js.map` |
| Tool definitions | `sdk-tools.d.ts` |
| Recovered source view | `recovered_source/` |
| Platform binaries | `vendor/` |
| Tutorial docs | `docs/tutorial/` |

## Repository Layout

```text
.
├── cli.js
├── cli.js.map
├── package.json
├── sdk-tools.d.ts
├── bun.lock
├── LICENSE.md
├── README.md
├── README_EN.md
├── docs/
│   └── tutorial/
├── vendor/
│   ├── audio-capture/
│   └── ripgrep/
└── recovered_source/
    └── src/
```

## What This Code Represents

This directory is effectively a combination of:

1. **an npm release artifact**
2. **a packaged CLI distribution**
3. **a research-oriented snapshot with source maps and recovered source**

That is why it contains:

- a large bundled `cli.js`
- an even larger `cli.js.map`
- platform-specific binaries
- a `recovered_source/` directory intended for analysis

## What the Tutorial Covers

The repository now includes a source-learning tutorial series under:

- Chinese index: `docs/tutorial/00-overview.md`
- English index: `docs/tutorial/00-overview_EN.md`

The tutorial currently covers the following topics:

1. **Overview and reading map**
   - repository positioning
   - recommended reading order
   - a high-level mental model for Claude Code
2. **Bootstrap and startup**
   - `entrypoints/cli.tsx`
   - `main.tsx`
   - fast paths and runtime assembly
3. **QueryEngine and the conversation loop**
   - multi-turn session state
   - system prompt / context assembly
   - query loop and tool execution
4. **Input processing, commands, and hooks**
   - `commands.ts`
   - `processUserInput.ts`
   - slash commands and pre-query processing
5. **Tools and agent execution**
   - `tools.ts`
   - representative tools such as FileRead, Bash, and Agent
   - permissions and role-based constraints
6. **Tasks, settings, and session memory**
   - `tasks.ts`
   - `settings.ts`
   - `sessionMemory.ts`
7. **MCP, plugins, and skills deep dive**
   - MCP configuration and connection loading
   - plugin commands / plugin skills / plugin-provided MCP
   - bundled skills / on-disk skills / dynamic skill discovery
8. **Reading paths and next steps**
   - guided deep-reading routes for different learning goals

## Tutorial Entry Points

### Chinese tutorial

- [00 Overview and reading map](./docs/tutorial/00-overview.md)
- [01 Bootstrap and startup](./docs/tutorial/01-bootstrap-and-startup.md)
- [02 QueryEngine and conversation loop](./docs/tutorial/02-query-engine-and-conversation-loop.md)
- [03 Input processing, commands, and hooks](./docs/tutorial/03-input-commands-and-hooks.md)
- [04 Tools and agent execution](./docs/tutorial/04-tools-and-agent-execution.md)
- [05 Tasks, settings, and session memory](./docs/tutorial/05-state-tasks-settings-and-memory.md)
- [06 MCP, plugins, and skills](./docs/tutorial/06-mcp-plugins-and-skills.md)
- [99 Reading paths and next steps](./docs/tutorial/99-reading-paths-and-next-steps.md)

### English tutorial

- [00 Overview and reading map](./docs/tutorial/00-overview_EN.md)
- [01 Bootstrap and startup](./docs/tutorial/01-bootstrap-and-startup_EN.md)
- [02 QueryEngine and conversation loop](./docs/tutorial/02-query-engine-and-conversation-loop_EN.md)
- [03 Input, commands, and hooks](./docs/tutorial/03-input-commands-and-hooks_EN.md)
- [04 Tools and agent execution](./docs/tutorial/04-tools-and-agent-execution_EN.md)
- [05 Tasks, settings, and session memory](./docs/tutorial/05-state-tasks-settings-and-memory_EN.md)
- [06 MCP, plugins, and skills](./docs/tutorial/06-mcp-plugins-and-skills_EN.md)
- [99 Reading paths and next steps](./docs/tutorial/99-reading-paths-and-next-steps_EN.md)

## Key Components

### `cli.js`

This is the bundled single-file CLI entry point shipped to users. It has already been compiled and compressed, and is typically invoked through:

```bash
claude
```

or:

```bash
node cli.js --version
```

### `cli.js.map`

This is the source map used to map bundled code back to original source locations for debugging and analysis.

### `sdk-tools.d.ts`

This file exposes TypeScript type definitions for Claude Code's built-in tools. From it, you can directly infer tool families such as:

- `Agent`
- `Bash`
- `Read`
- `Write`
- `Edit`
- `Glob`
- `Grep`
- `WebFetch`
- `WebSearch`
- `AskUserQuestion`
- `EnterWorktree` / `ExitWorktree`
- `ExitPlanMode`

### `vendor/`

This directory contains packaged multi-platform binaries:

- `vendor/ripgrep/`: bundled `rg` executables for high-performance code search
- `vendor/audio-capture/`: native `.node` modules indicating audio capture support

### `recovered_source/`

This is a recovered source view derived from the packaged artifact and mapping information, useful for structural research and implementation analysis.

## Architecture Notes

### 1. CLI Startup Path

- `recovered_source/src/entrypoints/cli.tsx`
- `recovered_source/src/main.tsx`

`entrypoints/cli.tsx` acts as a lightweight bootstrap layer that handles fast paths such as:

- `--version`
- background session / daemon-related commands
- remote-control / bridge modes
- certain MCP or browser integration entry points

It then lazily loads the full application when needed.

`main.tsx` assembles the main runtime, including:

- configuration initialization
- CLI argument parsing
- terminal UI setup
- command, skill, plugin, and MCP loading
- permission, task, and session restore mechanisms

### 2. QueryEngine as Conversation Core

- `recovered_source/src/QueryEngine.ts`

`QueryEngine` is one of the key cores of Claude Code's conversation runtime. It is responsible for:

- tracking messages and session state
- assembling system prompt context
- coordinating tool execution and permission checks
- tracking usage, budgets, file state, and interruption flow

Its main operating loop can be summarized as:

1. accept user input
2. build context
3. run model inference
4. execute tool calls
5. update state and continue

### 3. Terminal UI Model

From the import graph in `main.tsx`, the CLI appears to be much closer to an interactive terminal application built with a React / Ink style architecture than to a simple shell script. It supports:

- REPL-style interaction
- session recovery
- permission prompts
- skill and plugin loading
- MCP integration
- background tasks and multi-session management
- IDE, browser, and remote integration

## Capabilities That Can Be Inferred

From the current directory and recovered source view, the following capabilities can be inferred with reasonable confidence:

- natural-language coding assistance in the terminal
- reading, editing, and writing local files
- executing shell / bash commands
- code search and file discovery
- task and background session management
- step-by-step planning and execution
- Git / review / PR related workflows
- MCP-based extensibility
- skills / plugins based extension points
- session restore, remote, and bridge functionality

## Why This Is Not a Standard Source Repository

There are several strong indicators:

- the primary executable logic is bundled into a single `cli.js`
- a very large `cli.js.map` is included
- the package ships platform-specific native binaries
- `package.json` looks like release metadata rather than a typical development repo manifest
- the directory includes a research-oriented `recovered_source/` tree

Because of that, this repository is better suited for:

- release artifact research
- CLI architecture analysis
- tool interface reference
- learning the startup and execution model
- tutorial-based source reading

## Key File Index

- `package.json`: package metadata, version, Node requirement, binary entry
- `cli.js`: bundled CLI entry point
- `cli.js.map`: source map
- `sdk-tools.d.ts`: tool interface definitions
- `recovered_source/src/entrypoints/cli.tsx`: startup dispatch layer
- `recovered_source/src/main.tsx`: main runtime assembly
- `recovered_source/src/QueryEngine.ts`: conversation execution core

## Research-Only Boundary

To restate the boundary clearly:

- this repository is for study, research, and technical analysis only
- rights related to the code and associated materials belong to **Anthropic**
- this README is descriptive only and does not grant or imply any license or ownership claim

## Conclusion

In practice, this directory is best viewed as a **Claude Code package research repository plus tutorial documentation repository**. Its main value lies in helping readers:

- understand how the Claude Code CLI is packaged and bootstrapped
- inspect its tool-driven agent architecture
- systematically learn its command, tool, persistence, and extension layers
- study the codebase through a bilingual tutorial set

If your goal is to understand the product structure, this repository is informative. If your goal is to work from Anthropic's official upstream source, this should not be treated as the canonical development repository.
