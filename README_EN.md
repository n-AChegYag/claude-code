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

It should not be treated as an ownership claim or as the canonical upstream source repository.

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

In practice, this directory is best viewed as a **Claude Code package research repository**. Its main value lies in helping readers:

- understand how the Claude Code CLI is packaged and bootstrapped
- inspect its tool-driven agent architecture
- review its tool interfaces and distribution layout

If your goal is to understand the product structure, this repository is informative. If your goal is to work from Anthropic's official upstream source, this should not be treated as the canonical development repository.
