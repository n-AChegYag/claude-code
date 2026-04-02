# Claude Code Source Learning Tutorial 05: Tasks, Settings, and Session Memory

[中文版本](./05-state-tasks-settings-and-memory.md)

## Questions this chapter answers

After reading this chapter, you should be able to answer:

- Where does Claude Code persist state beyond the message list?
- Why is the task system not just an in-memory array?
- Why does the settings system include parsing, caching, merging, and validation layers?
- How is “memory” automatically extracted in this project?

## Why this layer matters

The earlier chapters mostly explain how the system runs. This chapter focuses on how the system **remembers, organizes, and carries work forward over time**.

In a long-lived CLI like Claude Code, the quality of the experience often depends not just on the model’s current output, but on these durable subsystems:

- the task system
- the settings system
- the session memory system

## 1. The task system: `utils/tasks.ts`

### It is not just UI support — it is a small persistence layer

At first glance, `recovered_source/src/utils/tasks.ts` might look like a helper module, but it is much closer to a lightweight persistence layer.

From `recovered_source/src/utils/tasks.ts:69-89`, the task schema includes:

- `id`
- `subject`
- `description`
- `activeForm`
- `owner`
- `status`
- `blocks`
- `blockedBy`
- `metadata`

That means tasks are not just display rows. They are structured entities with status transitions and dependency relationships.

### Tasks are persisted by directory and file

`recovered_source/src/utils/tasks.ts:221-240` defines:

- `getTasksDir(taskListId)`
- `getTaskPath(taskListId, taskId)`
- `ensureTasksDir(taskListId)`

So task state is written into Claude’s config area rather than only living inside process memory. That matters because it supports:

- persistence across turns and refreshes
- multi-process / multi-agent collaboration
- external watching and shared task-list behavior

### Task ID allocation is designed with concurrency in mind

`recovered_source/src/utils/tasks.ts:91-130` and `:267-307` are especially important.

They include:

- a `.highwatermark` file
- lockfile retry / backoff configuration
- “lock first, read highest ID second, write third” creation flow

That tells you the task system is not designed only for toy single-threaded cases. It is designed for:

> multiple Claudes, sub-agents, or cooperating processes modifying the same task list safely

### Task list identity also understands team/session context

`recovered_source/src/utils/tasks.ts:190-210`, in `getTaskListId()`, chooses task-list ownership by priority, including:

- explicit task list ID
- teammate / team name
- session ID

So the task system naturally supports both “shared by team” and “scoped to a session” collaboration modes.

### CRUD logic also protects consistency

The main task entry points are:

- create: `tasks.ts:284-307`
- read: `tasks.ts:310-349`
- update: `tasks.ts:370-391`
- delete: `tasks.ts:393-438`

Deletion is particularly worth reading because it does more than remove a JSON file. It also updates the high-water mark and removes references from other tasks that point to the deleted task. That is proper persistence-layer consistency logic, not just file deletion.

## 2. The settings system: `utils/settings/settings.ts`

If the task system answers “what work exists,” the settings system answers “what behavior is allowed and where those rules come from.”

### It is fundamentally a multi-source merge system

`recovered_source/src/utils/settings/settings.ts:74-120`, in `loadManagedFileSettings()`, shows one of the most important configuration patterns in the repository:

- load `managed-settings.json` first
- then load `managed-settings.d/*.json`
- apply sorted drop-ins on top

This is a mature configuration design pattern similar to systemd or sudoers drop-ins. Its value is that it lets:

- baseline policy live separately from customizations
- multiple teams ship independent configuration fragments
- administrators avoid editing one giant monolithic file forever

### Settings parsing includes caching

`recovered_source/src/utils/settings/settings.ts:178-199`, in `parseSettingsFile()`, first checks cache and only then calls `parseSettingsFileUncached()`.

That shows the settings layer is thinking about more than simple file access:

- settings may be read frequently
- parsed results should be cached
- callers should get clones so they cannot mutate cache state accidentally

That is a strong sign of careful engineering around configuration as shared runtime state.

### It is not “read JSON and trust it”

`recovered_source/src/utils/settings/settings.ts:201-220` shows the uncached parsing flow:

1. safely resolve the path
2. read file contents
3. return empty settings for empty files
4. parse JSON
5. filter invalid permission rules first
6. then run `SettingsSchema` validation

This means the real job of the settings system is not simply “load a file.” It is to **turn unreliable external configuration input into trusted internal structures**.

## 3. The session memory system: `services/SessionMemory/sessionMemory.ts`

### Memory is not the same thing as chat history

`recovered_source/src/services/SessionMemory/sessionMemory.ts:1-5` states its purpose directly:

- automatically maintain a markdown memory file
- run periodically in the background
- extract key information using a forked sub-agent
- avoid interrupting the main conversation flow

That means “memory” here is not the transcript and not the raw message list. It is **a distilled knowledge artifact derived from the session**.

### When memory extraction happens

`recovered_source/src/services/SessionMemory/sessionMemory.ts:134-181`, in `shouldExtractMemory(messages)`, is the core trigger function.

It considers multiple conditions:

- total token count
- whether initialization threshold has been reached
- whether enough tokens have accumulated since the last extraction
- whether enough tool calls have happened
- whether the last assistant turn still contains tool calls

So the memory system is not summarizing constantly. It is looking for a better trade-off point between value and cost.

### How the session memory file is established

`recovered_source/src/services/SessionMemory/sessionMemory.ts:183-233`, in `setupSessionMemoryFile(...)`, is especially worth reading.

It does the following:

- creates the memory directory
- initializes a template if the file does not yet exist
- clears read cache so it does not get a `file_unchanged` stub
- reads current memory content by calling `FileReadTool.call(...)`

That means even an internal background subsystem tries to reuse the existing tool model rather than silently bypassing it with a completely separate file protocol.

### Extraction is performed through a forked agent

`recovered_source/src/services/SessionMemory/sessionMemory.ts:272-319` shows the extraction path:

- check the feature gate
- check whether extraction conditions are met
- create an isolated context
- build the update prompt
- call `runForkedAgent(...)`

This reflects another important architectural pattern in Claude Code:

> even internal maintenance behavior tries to reuse the shared agent + tool machinery instead of inventing a totally separate execution path

## How these three subsystems fit together

They may look separate, but they are all answering the same higher-level question:

> in a long-running session, what kinds of information should persist beyond a single turn?

- the task system keeps “what work exists right now”
- the settings system keeps “what behavior is allowed and how it should be configured”
- the session memory system keeps “what knowledge from this conversation is worth carrying forward”

Together, they help turn Claude Code from an instant-response chat tool into a durable terminal agent.

## Recommended deep-reading path

### Task system

- `recovered_source/src/utils/tasks.ts:69-89`
- `recovered_source/src/utils/tasks.ts:190-307`
- `recovered_source/src/utils/tasks.ts:370-438`

Question to answer: **why does a task system need file locks, a high-water mark, and dependency cleanup logic?**

### Settings system

- `recovered_source/src/utils/settings/settings.ts:74-120`
- `recovered_source/src/utils/settings/settings.ts:178-220`

Question to answer: **why is configuration loading more than `JSON.parse(readFile())`?**

### Session memory system

- `recovered_source/src/services/SessionMemory/sessionMemory.ts:134-181`
- `recovered_source/src/services/SessionMemory/sessionMemory.ts:183-233`
- `recovered_source/src/services/SessionMemory/sessionMemory.ts:272-319`

Question to answer: **why does memory extraction need its own timing logic and a forked agent?**

## Chapter summary

The most important conclusion from this chapter is:

> Claude Code’s long-lived working behavior depends not only on QueryEngine, but also on a set of deeply engineered persistence and continuity subsystems.

Tasks make work decomposable and collaborative. Settings make behavior controllable. Session memory reduces the need to rely entirely on the active message window.

## Next chapter

Continue to [06-mcp-plugins-and-skills_EN.md](./06-mcp-plugins-and-skills_EN.md).
