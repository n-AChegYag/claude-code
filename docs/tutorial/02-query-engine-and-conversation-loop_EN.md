# Claude Code Source Learning Tutorial 02: QueryEngine and the Conversation Loop

[中文版本](./02-query-engine-and-conversation-loop.md)

## Questions this chapter answers

After reading this chapter, you should be able to answer:

- Why is Claude Code much more than “receive input -> call the API once -> stop”?
- What role does `QueryEngine` play in the system?
- How does one user turn enter the model, trigger tools, and return as new session state?

## Start with the core positioning

`recovered_source/src/QueryEngine.ts:175-183` states the role of `QueryEngine` very clearly:

- it owns the query lifecycle of a conversation
- it owns conversation session state
- one conversation gets one `QueryEngine`
- each `submitMessage()` call is a new turn inside the same conversation

That means the core of Claude Code is not a stateless request handler. It is a **multi-turn executor that deliberately holds ongoing state**.

## What `QueryEngine` actually keeps

Look at `recovered_source/src/QueryEngine.ts:184-207`, and you can see that the class stores much more than just messages:

- `mutableMessages`
- `abortController`
- `permissionDenials`
- `totalUsage`
- `readFileState`
- `discoveredSkillNames`
- `loadedNestedMemoryPaths`

That tells you the problem it solves is not merely “how to issue a model request.” It also manages:

1. what messages the current session has accumulated
2. whether tool permissions were denied during the current turn
3. how file-read cache state is reused across turns
4. how token / usage accounting survives multiple turns
5. what skills and memory materials were discovered during the session

## What happens inside one `submitMessage()` call

### Step 1: pull turn-level config and set execution context

`recovered_source/src/QueryEngine.ts:209-240` reads the turn-level execution environment, including:

- working directory `cwd`
- command list `commands`
- tool list `tools`
- MCP clients
- budget / model / thinking configuration
- state read/write hooks

At the same time it also:

- clears the turn-local discovered-skill set: `QueryEngine.ts:238`
- moves the shell context to the conversation cwd: `QueryEngine.ts:239`

So `submitMessage()` is not just “accept a prompt.” It assembles the execution environment for the whole turn.

### Step 2: wrap permission checking

`recovered_source/src/QueryEngine.ts:243-271` defines `wrappedCanUseTool`.

This wrapper does more than forward permission checks. When a tool use is denied, it records the denial into `permissionDenials`. That means `QueryEngine` does not only manage the success path; it also folds **permission failures into session-level state** for SDK or upper-layer consumers.

### Step 3: assemble the system prompt and context

`recovered_source/src/QueryEngine.ts:284-325` calls `fetchSystemPromptParts(...)`, which returns:

- `defaultSystemPrompt`
- `userContext`
- `systemContext`

Then it constructs the final `systemPrompt` from custom prompt pieces, memory-mechanics prompt injection, and append-only prompt additions.

The key point is this:

> Claude Code does not treat the system prompt as a fixed constant. It assembles runtime context dynamically from tools, model choice, working directories, MCP, and other execution details.

That is why prompt engineering in this codebase is really a runtime assembly problem, not just a static-string problem.

### Step 4: build input-processing context

`recovered_source/src/QueryEngine.ts:335-344` begins constructing `processUserInputContext`.

This tells you that before the model ever sees the input, Claude Code must first resolve things like:

- what the current message array is
- whether slash commands might rewrite that array
- how state mutations should be written back
- how tool-use context should be handed to the input-processing layer

So the raw user input does not go straight to the model. It first passes through a **conversation-aware preprocessing stage**.

### Step 5: call `processUserInput()`

`recovered_source/src/QueryEngine.ts:416` is where the input is handed to `processUserInput(...)`.

This is a major boundary in the flow:

- if the input is actually a slash command, the turn may never invoke the model at all
- if hooks block continuation, the turn may stop early
- if attachments, images, or other context need to be normalized, they are handled here first

That is why the relationship between `QueryEngine` and `processUserInput` is best understood as:

- `QueryEngine` is the **top-level orchestrator**
- `processUserInput` turns raw incoming input into a form suitable for the turn execution pipeline

### Step 6: enter the actual query loop

After input preprocessing completes, `QueryEngine.ts:675-681` enters:

- `for await (const message of query({...}))`

That means the model interaction is not a single promise-returning function call. It is a **streaming async generator**.

Claude Code can therefore stream model output, execute tools, and update the message flow incrementally instead of waiting for a whole turn to finish before doing anything useful.

## Why `query.ts` matters

If `QueryEngine` is the conversation-level director, then `query.ts` is closer to the single-turn execution engine.

`recovered_source/src/query.ts:219-239` defines `query(params)` and forwards into `queryLoop(...)`. From the imports and loop state, you can see that this file is responsible for things like:

- automatic compaction
- prompt-too-long / max-output-token recovery
- tool orchestration
- stop hooks
- token budgets
- command queue lifecycle notifications
- tool result summarization

So the distinction is:

- `QueryEngine` cares about the **conversation lifecycle**
- `query.ts` cares about **how one model/tool turn keeps running correctly**

## A more practical mental model

You can think of a conversation turn like this:

```text
user input
  -> QueryEngine.submitMessage()
  -> assemble system prompt / user context / tool context
  -> processUserInput()
  -> query()
  -> streaming model output
  -> tool execution and tool result reinjection
  -> write new messages back into mutableMessages
  -> wait for the next submitMessage()
```

The important part is not the exact call graph, but what it says about the product:

> Claude Code is an interactive agent system that continuously accumulates state, trims context, executes tools, and carries the conversation forward.

## Why `QueryEngine` is the best learning anchor

If you could deeply read only one file, `recovered_source/src/QueryEngine.ts` would be one of the strongest candidates because it connects multiple subsystems together:

- prompt assembly
- permission wrapping
- input preprocessing
- the tool layer
- state updates
- SDK output handling
- session persistence behavior

In other words, many seemingly local files only make full sense once you understand how `QueryEngine` uses them.

## Suggested reading strategy for this chapter

### First pass: read the structure only

Focus on:

- `recovered_source/src/QueryEngine.ts:175-207`
- `recovered_source/src/QueryEngine.ts:209-325`
- `recovered_source/src/QueryEngine.ts:335-344`
- `recovered_source/src/QueryEngine.ts:416`
- `recovered_source/src/QueryEngine.ts:675-681`

The goal is to answer: **what are its inputs, its state, and its outputs?**

### Second pass: supplement with `query.ts`

Read:

- `recovered_source/src/query.ts:181-217`
- `recovered_source/src/query.ts:219-260`

The goal is to answer: **why does a single turn need a streaming state machine?**

## Chapter summary

The most important conclusion from this chapter is:

> The core of Claude Code is not a single API call, but a session engine that maintains messages, permissions, caches, budgets, and tool execution results across turns.

And `QueryEngine` is the most valuable central file for understanding that engine.

## Next chapter

Continue to [03-input-commands-and-hooks_EN.md](./03-input-commands-and-hooks_EN.md).
