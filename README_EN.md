# Claude Code Source Learning and Introductory Interpretation

[中文版本](./README.md)

> [!IMPORTANT]
> **This project is provided for learning, research, and technical analysis only.**
>
> **All ownership, copyright, and all other rights related to the code and associated materials remain with Anthropic.**
>
> This project does not claim, transfer, or redistribute the original Claude Code codebase or its intellectual property.

## Project Overview

The central theme of this repository is: **learning from, observing, and providing an introductory interpretation of the Claude Code source**.

More precisely, this is not Anthropic’s official public upstream development repository. It is a learning-oriented workspace organized around the packaged `@anthropic-ai/claude-code` distribution, a recovered source view, and a growing set of tutorial documents.

The main goals of this project are:

- to understand Claude Code at the architectural level
- to study its CLI bootstrap path and main runtime assembly
- to observe key subsystems such as QueryEngine, the command system, the tool system, the task system, and the settings system
- to provide an introductory breakdown of extension surfaces such as MCP, plugins, skills, the permission model, and security boundaries
- to turn that learning into structured tutorial documentation and reading paths

Because of that, this repository is best understood as:

> a learner-oriented repository for studying and introducing the Claude Code source.

## Research and Learning Boundaries

A few boundaries should be made explicit:

1. This project is intended only for **learning, research, technical analysis, and explanatory documentation**.
2. This project is not Anthropic’s official source repository and should not be treated as official implementation documentation.
3. All rights related to the code remain with **Anthropic**.
4. The interpretations, breakdowns, and tutorials in this repository are learning-oriented summaries based on the visible directory contents and do not constitute any ownership claim.

If your goal is to:

- study the overall structure of Claude Code
- build a working understanding of major subsystem responsibilities
- follow a guided reading path through the code
- use this repository as a sample for analyzing an agentic terminal-first CLI

then this project is well suited to that purpose.

If your goal is to treat this repository as Anthropic’s official upstream engineering base, it should not be used that way.

## What the Current Directory Contains

The current directory mainly consists of the following parts:

```text
.
├── cli.js
├── cli.js.map
├── package.json
├── sdk-tools.d.ts
├── README.md
├── README_EN.md
├── docs/
│   └── tutorial/
├── vendor/
└── recovered_source/
    └── src/
```

A simple way to think about these components is:

- `cli.js`: the bundled CLI entry point
- `cli.js.map`: the source map used to help reconstruct and locate source structure
- `sdk-tools.d.ts`: TypeScript definitions for Claude Code’s built-in tools
- `recovered_source/`: a recovered source view derived from the packaged artifact
- `docs/tutorial/`: tutorial documents organized around source reading

## Source Themes This Project Focuses On

At the moment, this repository focuses on these themes in its study and introductory interpretation of Claude Code:

- startup flow: how the CLI enters the main runtime
- the main loop: how QueryEngine organizes multi-turn sessions
- input handling: how commands, hooks, and attachments enter the system
- the tool system: how capabilities such as Read, Edit, Bash, and Agent are structured and constrained
- persistent state: how tasks, settings, and session memory support long-running workflows
- extension surfaces: how MCP, plugins, and skills connect to the main runtime
- safety boundaries: how permission modes, path protection, shell safety, and workflow boundaries interact

The goal here is not to “reconstruct official design documentation,” but to build a structured and readable learning path based on the code and materials visible in this directory.

## Tutorial Entry Points

This repository already includes a tutorial set that serves as a guided reading path for source learning.

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

## The Value of This Project

The value of this project does not come from replacing the official source repository. Its value comes from:

- offering a more accessible entry point for learning Claude Code
- turning a scattered recovered source view into a structured set of study themes
- helping readers build an initial understanding of major responsibilities, runtime flow, and system boundaries
- providing a tutorial and note-taking base for deeper future source analysis

## Statement

To restate the boundary clearly:

- this project is for learning, research, and technical analysis only
- the Claude Code code and all related rights remain with **Anthropic**
- the tutorials, explanations, and introductory interpretations in this repository do not constitute any ownership claim and do not imply any official authorization

## Closing Note

If you want to use this repository as a **Claude Code source-learning sample**, it is well suited for that purpose.

If your goal is to quickly build an introductory understanding of Claude Code’s startup flow, main loop, tool system, extension surfaces, and safety boundaries, then the tutorials and documentation in this repository are designed exactly for that purpose.
