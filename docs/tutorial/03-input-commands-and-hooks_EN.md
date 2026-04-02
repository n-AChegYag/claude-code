# Claude Code Source Learning Tutorial 03: Input Processing, Commands, and Hooks

[中文版本](./03-input-commands-and-hooks.md)

## Questions this chapter answers

After reading this chapter, you should be able to answer:

- Why can’t user input just be sent directly to the model?
- Where are slash commands registered, and how are they loaded?
- Why can hooks decide whether a turn continues at all?

## Start with the command system: `commands.ts`

Claude Code’s command system is not just a few `if/else` checks for `/help` or `/clear`. It is a registry that aggregates commands from multiple sources.

### There are many built-in commands

In `recovered_source/src/commands.ts:2-188`, you can see many built-in command imports, including:

- `/help`
- `/login`
- `/logout`
- `/tasks`
- `/plan`
- `/permissions`
- `/model`
- `/status`
- `/memory`
- `/hooks`
- `/plugin`
- `/agents`

That already tells you slash commands are not a side feature in Claude Code. They are a first-class interaction surface.

### Commands do not come only from built-ins

The more important part is `recovered_source/src/commands.ts:353-509`. This shows how command sources are assembled together:

- skill-directory commands: `commands.ts:353-386`
- plugin skills
- bundled skills
- built-in plugin skills
- plugin commands: `commands.ts:449-468`
- workflow commands
- finally the built-in `COMMANDS()` list

Then `recovered_source/src/commands.ts:476-509`, through `getCommands(cwd)`, applies additional processing:

- availability filtering
- `isEnabled()` filtering
- dynamic skill injection
- deduplication and insertion-position logic

So the command system is not a static array. It is a **command set dynamically assembled from cwd, plugins, skills, and current runtime conditions**.

## Why the command system is designed this way

Claude Code is not just one fixed CLI shell. It also supports:

- user-defined skills
- plugin-based extension
- runtime skill discovery
- availability filtering based on auth/provider state

That means the command layer must be composable rather than hardcoded.

## Now look at input preprocessing: `processUserInput.ts`

If `commands.ts` answers “what commands exist in the system,” then `processUserInput.ts` answers “what does this specific input actually mean?”

### `processUserInput()` is the main input gateway

`recovered_source/src/utils/processUserInput/processUserInput.ts:85-140` defines `processUserInput(...)`.

It accepts much more than just a plain string. Its inputs include:

- `mode`
- `pastedContents`
- `ideSelection`
- historical `messages`
- `querySource`
- `canUseTool`
- `skipSlashCommands`
- `bridgeOrigin`
- `isMeta`
- `skipAttachments`

That tells you “user input” in Claude Code is a much richer object than plain text.

## `processUserInput()` has two major stages

### Stage 1: normalize the base input

`recovered_source/src/utils/processUserInput/processUserInput.ts:153-171` first calls `processUserInputBase(...)`.

The base stage handles things like:

- ordinary text
- slash command parsing
- pasted-content expansion
- image / attachment handling
- deciding whether the turn should actually proceed into query execution

So the first question it answers is:

> is this input really a chat request, or is it some other local control action inside the runtime?

### Stage 2: run UserPromptSubmit hooks

If the base stage says the turn should continue, `processUserInput.ts:178-263` begins iterating over `executeUserPromptSubmitHooks(...)`.

This is an extremely important extension point.

From the code, hooks can produce several outcomes:

1. **block and return an error**: `processUserInput.ts:193-209`
2. **stop continuation while preserving the original prompt in context**: `processUserInput.ts:211-224`
3. **attach additional context material**: `processUserInput.ts:226-240`
4. **append hook-generated messages**: `processUserInput.ts:242-262`

So hooks are not just side-band logging. They participate in whether the turn is allowed to proceed at all.

## What this tells you about Claude Code’s interaction model

It means Claude Code treats user input as a **pipeline that can be transformed and gated**:

```text
raw input
  -> base parsing
  -> slash command / attachment / pasted-content expansion
  -> hook checks
  -> additional context injection
  -> decide whether to enter model query execution
```

That is quite different from many chat systems, where hooks are peripheral. Here, hooks sit directly in the pre-query control path.

## The relationship between slash commands and model input

One easy misunderstanding is to assume every slash command means “run something locally and stop.” In Claude Code, commands can behave in several different ways:

- some commands return local results immediately
- some commands generate prompts and then invoke the model
- some commands mutate conversation state
- some commands switch runtime modes or configuration

That is one reason `QueryEngine` must prepare commands, tool context, and message context before calling `processUserInput()`.

## Why the input layer is this complex

Because Claude Code supports much more than “the user typed a sentence.” It also supports:

- slash commands
- multimodal inputs with images
- pasted text expansion
- IDE selection context
- remote / bridge-origin inputs
- system-generated meta inputs
- hook-injected context

So the real job of the input layer is:

> to normalize multiple heterogeneous sources into the message set that the turn should actually execute on

## Recommended reading slices

### Command system

- `recovered_source/src/commands.ts:353-398`
- `recovered_source/src/commands.ts:449-509`

Main question: **what sources are commands assembled from?**

### Input-processing main flow

- `recovered_source/src/utils/processUserInput/processUserInput.ts:140-171`
- `recovered_source/src/utils/processUserInput/processUserInput.ts:178-263`
- `recovered_source/src/utils/processUserInput/processUserInput.ts:281-309`

Main question: **how is input processed in layers before query execution?**

## Chapter summary

The most important conclusion from this chapter is:

> Claude Code treats user input as a composable preprocessing pipeline, not as a raw string that goes directly to the model.

The command system defines the available interaction entry points. The input-processing layer decides what this specific turn becomes. Hooks can block, enrich, or reshape that flow before the main query loop begins.

## Next chapter

Continue to [04-tools-and-agent-execution_EN.md](./04-tools-and-agent-execution_EN.md).
