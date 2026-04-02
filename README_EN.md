# Claude Code Source Learning and Introductory Interpretation

[中文版本](./README.md)

> [!IMPORTANT]
> **This project is provided for learning, research, and technical analysis only.**
>
> **All Claude Code code and all related rights remain with Anthropic.**
>
> This project does not claim, transfer, or redistribute the original Claude Code codebase or its intellectual property.

## Project Positioning

This repository is a learning-oriented project centered on **studying and providing an introductory interpretation of the Claude Code source**.

It is not Anthropic’s official upstream development repository. Instead, it is a research-oriented workspace built around the currently visible packaged artifacts, a recovered source view, and tutorial documents derived from reading and organizing those materials.

## What you can find here

This repository currently includes:

- `cli.js`: the bundled CLI entry point
- `cli.js.map`: the source map
- `sdk-tools.d.ts`: built-in tool type definitions
- `recovered_source/`: the recovered source view
- `docs/tutorial/`: tutorial documents organized around source reading

## What this project focuses on

At the moment, the project mainly focuses on these Claude Code topics:

- CLI bootstrap and main runtime assembly
- QueryEngine and the conversation loop
- input handling, commands, and hooks
- the tool system and sub-agent execution
- persistent state systems such as tasks, settings, and session memory
- extension surfaces such as MCP, plugins, and skills
- the permission system and security boundaries

## Tutorial Entry Points

### Chinese tutorial

- [00 Overview and reading map](./docs/tutorial/00-overview.md)
- [01 Bootstrap and startup](./docs/tutorial/01-bootstrap-and-startup.md)
- [02 QueryEngine and conversation loop](./docs/tutorial/02-query-engine-and-conversation-loop.md)
- [03 Input, commands, and hooks](./docs/tutorial/03-input-commands-and-hooks.md)
- [04 Tools and agent execution](./docs/tutorial/04-tools-and-agent-execution.md)
- [05 Tasks, settings, and session memory](./docs/tutorial/05-state-tasks-settings-and-memory.md)
- [06 MCP, plugins, and skills](./docs/tutorial/06-mcp-plugins-and-skills.md)
- [07 Permissions and security boundaries](./docs/tutorial/07-permissions-and-security-boundaries.md)
- [99 Reading paths and next steps](./docs/tutorial/99-reading-paths-and-next-steps.md)

### English tutorial

- [00 Overview and reading map](./docs/tutorial/00-overview_EN.md)
- [01 Bootstrap and startup](./docs/tutorial/01-bootstrap-and-startup_EN.md)
- [02 QueryEngine and conversation loop](./docs/tutorial/02-query-engine-and-conversation-loop_EN.md)
- [03 Input, commands, and hooks](./docs/tutorial/03-input-commands-and-hooks_EN.md)
- [04 Tools and agent execution](./docs/tutorial/04-tools-and-agent-execution_EN.md)
- [05 Tasks, settings, and session memory](./docs/tutorial/05-state-tasks-settings-and-memory_EN.md)
- [06 MCP, plugins, and skills](./docs/tutorial/06-mcp-plugins-and-skills_EN.md)
- [07 Permissions and security boundaries](./docs/tutorial/07-permissions-and-security-boundaries_EN.md)
- [99 Reading paths and next steps](./docs/tutorial/99-reading-paths-and-next-steps_EN.md)

## Usage Boundary

Please note:

- this project is for learning, research, and technical analysis only
- this project does not represent Anthropic’s official implementation documentation
- all Claude Code code and related rights remain with **Anthropic**
- the explanations, interpretations, and tutorials in this repository do not constitute any ownership claim or official authorization

## Short Note

If you want to use this repository as a **Claude Code source-learning sample**, it is well suited to that purpose.

If your goal is to build a quicker introductory understanding of Claude Code’s startup flow, main loop, tool system, extension surfaces, and safety boundaries, the tutorial documents here are intended to support exactly that goal.
