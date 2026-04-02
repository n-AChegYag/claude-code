# Claude Code Source Learning Tutorial 01: Bootstrap and Runtime Assembly

[中文版本](./01-bootstrap-and-startup.md)

## Questions this chapter answers

After reading this chapter, you should be able to answer:

- Where does the `claude` command actually begin execution?
- Why does the project have `cli.js`, `entrypoints/cli.tsx`, and `main.tsx` at the same time?
- Why is startup split into a thin bootstrap layer and a heavier runtime assembly layer?

## Start at the outermost layer: the package entry

`package.json:2-13` shows that this package is `@anthropic-ai/claude-code`, and that:

- `package.json:4-6` maps the `claude` binary to `cli.js`

So when a user types `claude` in the terminal, execution first enters the bundled artifact `cli.js`. For learning purposes, though, it is better to inspect the recovered source entry layers instead of the compressed bundle:

- `recovered_source/src/entrypoints/cli.tsx`
- `recovered_source/src/main.tsx`

## Layer 1: the lightweight dispatcher in `entrypoints/cli.tsx`

`recovered_source/src/entrypoints/cli.tsx:28-33` states its role directly:

> Bootstrap entrypoint - checks for special flags before loading the full CLI.

That line is important because it tells you:

- this is not the full application body
- its job is to **recognize special paths first** so the runtime does not eagerly load everything

### Why this thin layer exists

`recovered_source/src/entrypoints/cli.tsx:36-41` shows the simplest fast path:

- when the arguments are only `--version`, `-v`, or `-V`, it prints the version and returns immediately

That means a trivial command does not need to load the React UI, the tool system, the settings system, MCP, or the command registry. This is a classic CLI performance pattern:

- **keep simple requests cheap**
- **pay for full initialization only when needed**

### What kinds of special modes it handles

From `recovered_source/src/entrypoints/cli.tsx:50-217`, you can see that this bootstrap layer branches early for several cases, including:

- `--dump-system-prompt`
- `--claude-in-chrome-mcp` / `--chrome-native-host`
- `--daemon-worker`
- `remote-control` / `remote` / `sync` / `bridge`
- `daemon`
- `ps` / `logs` / `attach` / `kill` / `--bg`
- template job commands

### The key design idea in this file

The main keyword here is not “many features,” but **deferred heavyweight initialization**.

You can see many dynamic `import()` calls, for example at:

- `recovered_source/src/entrypoints/cli.tsx:45-48`
- `recovered_source/src/entrypoints/cli.tsx:55-67`
- `recovered_source/src/entrypoints/cli.tsx:101-104`
- `recovered_source/src/entrypoints/cli.tsx:191-207`

That tells you the file is doing two things:

1. reducing startup-time module evaluation cost
2. slicing functionality by execution path so only the needed modules load

## Layer 2: `main.tsx` assembles the full runtime

If `entrypoints/cli.tsx` is the routing layer, then `main.tsx` is the file that assembles the actual application.

`recovered_source/src/main.tsx:585` defines the main runtime entry `main()`. The file is large and import-heavy, so the right way to read it is not top-to-bottom first, but by asking what categories of startup work it performs.

## The four kinds of startup work in `main.tsx`

### 1. Process-level environment and safety setup

`recovered_source/src/main.tsx:588-607` is worth close attention:

- `NoDefaultCurrentDirectoryInExePath` is set to `1`
- warning handling is initialized early
- `exit` / `SIGINT` handlers are registered

That shows startup is not just about “open the UI.” It first handles:

- path-hijacking concerns
- terminal cleanup behavior on exit
- differences between interactive and print-mode interruption

### 2. Very early URL / deep-link / special-mode rewriting

`recovered_source/src/main.tsx:609-719` shows another important startup responsibility:

- parsing `cc://` and `cc+unix://` connection URLs
- handling `--handle-uri`
- supporting macOS URL handling
- rewriting flows like `assistant` and `ssh`

The idea here is:

> before the full runtime consumes argv, some argument forms must be rewritten into a normalized form the main command flow understands

So `main.tsx` does not merely consume argv. It normalizes heterogeneous entry paths into one broader runtime model.

### 3. Full initialization

`recovered_source/src/main.tsx:916` calls `await init()`.

That strongly suggests the system is now entering full runtime setup, including things like:

- configuration loading
- environment variable application
- auth / policy / managed-setting prefetch
- global runtime preparation

Even without unpacking `init()` yet, the architecture is already clear:

- `entrypoints/cli.tsx` avoids unnecessary initialization
- `main.tsx` performs the complete setup only after the request truly needs it

### 4. Command assembly and REPL entry

`recovered_source/src/main.tsx:1928` prepares `commandsPromise = getCommands(preSetupCwd)`.

That matters because it means:

- the command system is not hardcoded inline in the UI layer
- startup assembles built-in commands, skills, plugin commands, workflow commands, and more into a single current command set

You can also see several `launchRepl(...)` call sites, such as:

- `recovered_source/src/main.tsx:3134`
- `recovered_source/src/main.tsx:3176`
- `recovered_source/src/main.tsx:3242`

That shows the runtime ultimately enters a **long-lived interactive session**, not a one-shot execution path.

## Why this layering matters

A common first mistake when reading a codebase like this is to assume “the entry file is the business logic file.” In Claude Code, the better mental model is:

### Layer 1: startup dispatch

`entrypoints/cli.tsx`

Responsibilities:

- identify execution path quickly
- handle cheap commands early
- avoid heavy imports when possible
- route special modes to dedicated handlers

### Layer 2: runtime assembly

`main.tsx`

Responsibilities:

- initialize the runtime environment
- assemble permissions, settings, auth, telemetry, commands, plugins, and skills
- construct the terminal interaction context
- launch the REPL / main runtime loop

This is a very typical architecture for a high-complexity CLI:

- **thin entry layer for performance and routing**
- **thick runtime layer for assembly and long-lived execution**

## What code points deserve the closest reading

### Fast startup paths

- `recovered_source/src/entrypoints/cli.tsx:33-41`
- `recovered_source/src/entrypoints/cli.tsx:50-71`
- `recovered_source/src/entrypoints/cli.tsx:95-105`
- `recovered_source/src/entrypoints/cli.tsx:108-209`

These answer: **what cases do not need the full runtime?**

### Early environment governance in the main runtime

- `recovered_source/src/main.tsx:585-607`
- `recovered_source/src/main.tsx:609-719`

These answer: **what must be handled globally before the runtime enters main application behavior?**

### Assembly right before the interactive loop

- `recovered_source/src/main.tsx:916`
- `recovered_source/src/main.tsx:1928`
- `recovered_source/src/main.tsx:3134`

These answer: **when does the system truly become interactive?**

## Chapter summary

If you remember only one sentence from this chapter, let it be this:

> Claude Code startup is not “run one script,” but “route cheaply first, then assemble a full interactive runtime when necessary.”

That is one reason the project feels much more like a terminal application framework than a traditional small CLI utility.

## Next chapter

Continue to [02-query-engine-and-conversation-loop_EN.md](./02-query-engine-and-conversation-loop_EN.md).
