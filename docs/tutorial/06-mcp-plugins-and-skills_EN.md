# Claude Code Source Learning Tutorial 06: MCP, Plugins, and Skills Deep Dive

[中文版本](./06-mcp-plugins-and-skills.md)

## Questions this chapter answers

After reading this chapter, you should be able to answer:

- How does Claude Code connect to external extension surfaces?
- What is the relationship between MCP, plugins, and skills?
- How do they ultimately feed into the command system, tool system, and startup flow?

## Start with the overall picture

The previous chapters explained how Claude Code starts, processes input, and organizes tools and state. This chapter focuses on how it **continues to extend itself**.

A useful first-pass view is:

- **MCP**: the protocol/runtime layer that connects external services and translates them into tools, commands, and resources.
- **Plugins**: a packaging and loading layer that can contribute commands, skills, and even MCP servers.
- **Skills**: a higher-level prompt-and-command abstraction used to expose reusable capabilities and local domain knowledge.

These are not three completely independent systems. They meet during startup and command/tool assembly.

## 1. How startup brings them into the runtime

### `main.tsx` registers built-in plugins and bundled skills first

`recovered_source/src/main.tsx:1919-1928` is a key place to read:

- it calls `initBuiltinPlugins()`
- then `initBundledSkills()`
- then calls `getCommands(preSetupCwd)`

That reflects an important design rule:

> register extension sources first, then assemble the current command set

If this order were reversed, command assembly could miss those sources entirely.

### MCP connection and resource prefetch are part of startup assembly too

`recovered_source/src/main.tsx:2704-2724` and the surrounding imports show that startup also relies on:

- `getMcpToolsCommandsAndResources(...)`
- `prefetchAllMcpResources(...)`

So MCP is not a peripheral “later feature.” It is part of the main runtime assembly path.

## 2. MCP: how external services become tools, commands, and resources

The two most important MCP files to study first are:

- configuration and policy: `recovered_source/src/services/mcp/config.ts`
- connection and capability extraction: `recovered_source/src/services/mcp/client.ts`

### 1. `config.ts` decides which MCP servers participate in the current runtime

`recovered_source/src/services/mcp/config.ts:1071-1250`, in `getClaudeCodeMcpConfigs(...)`, is one of the central configuration entry points.

It does much more than “load one config file”:

- loads enterprise / user / project / local scopes
- gives enterprise configs exclusive control when present
- filters servers by policy
- loads plugin-provided MCP servers: `config.ts:1114-1154`
- filters project servers through approval rules: `config.ts:1164-1170`
- deduplicates plugin servers against manually configured servers: `config.ts:1172-1229`
- merges the surviving configs with precedence, then applies policy filtering again: `config.ts:1231-1248`

That means MCP configuration in Claude Code includes:

- multi-scope merging
- plugin-source integration
- approval / policy filtering
- duplicate suppression

It is a real configuration-governance layer, not just a raw settings reader.

### 2. `parseMcpConfig(...)` converts external config into trusted internal structure

`recovered_source/src/services/mcp/config.ts:1297-1377`, in `parseMcpConfig(...)`, performs multiple normalization steps:

- schema validation
- environment-variable expansion
- warning generation for missing variables
- platform-specific checks such as Windows `npx` wrapping guidance

This is another example of a common Claude Code pattern: external input is not trusted just because it is configuration.

### 3. `client.ts` performs the real connections and extracts capabilities

`recovered_source/src/services/mcp/client.ts:2226-2403`, in `getMcpToolsCommandsAndResources(...)`, is the most important MCP runtime implementation to read.

It does the following:

- obtains all active MCP configuration entries: `client.ts:2237-2239`
- separates disabled servers first: `client.ts:2241-2254`
- calculates transport stats and partitions local vs. remote connections: `client.ts:2256-2271`
- connects to servers concurrently: `client.ts:2282-2402`
- for connected servers, extracts:
  - `fetchToolsForClient(client)`
  - `fetchCommandsForClient(client)`
  - skills/resources if the server supports them

At `client.ts:2344-2369`, you can see clearly that MCP can yield:

- tools
- commands
- skills (when resource-backed skill discovery is supported)
- resources

That is the real significance of MCP inside Claude Code:

> it turns external services into Claude Code-native tools, commands, and resource objects the runtime can consume directly

## 3. Plugins: how packaged extensions inject commands, skills, and MCP

The best plugin files to study first are:

- `recovered_source/src/utils/plugins/loadPluginCommands.ts`
- `recovered_source/src/plugins/bundled/index.ts`
- `recovered_source/src/services/mcp/config.ts` (for plugin MCP)

### 1. Built-in plugins currently exist as a startup extension hook

`recovered_source/src/plugins/bundled/index.ts:1-23` is short, but instructive.

It tells you that:

- built-in plugins are meant to be CLI-shipped features that appear in the `/plugin` system
- no built-in plugins are actively registered yet
- the startup hook `initBuiltinPlugins()` already exists as scaffolding

So the plugin system is not only for third parties. It is also a framework for future internal modularization.

### 2. How plugin commands are loaded

`recovered_source/src/utils/plugins/loadPluginCommands.ts:414-677`, in `getPluginCommands()`, shows the plugin-command loading pipeline:

- load only enabled plugins: `loadPluginCommands.ts:422-429`
- process plugins in parallel: `loadPluginCommands.ts:431-433`
- load markdown commands from the default commands directory
- load commands from extra `commandsPaths`: `loadPluginCommands.ts:465-600`
- support inline command content from metadata: `loadPluginCommands.ts:603-667`

This means plugin commands in Claude Code are not fundamentally compiled code callbacks. They are heavily based on:

- markdown files
- frontmatter
- plugin manifest metadata
- directory scanning and path resolution

So the plugin-command model is largely a **content-driven command extension mechanism**.

### 3. Plugin skills are loaded in a similar way, but sit at a higher semantic layer

The same file also handles plugin skill loading. It recognizes:

- a direct skill directory containing `SKILL.md`
- nested subdirectories containing skill structures

This means plugins do not only add commands. They can also ship higher-level skill capabilities.

### 4. Plugins can also bring MCP servers into the runtime

Back in `recovered_source/src/services/mcp/config.ts:1114-1154`, the MCP configuration layer explicitly:

- loads enabled plugins through `loadAllPluginsCacheOnly()`
- collects plugin MCP servers
- feeds them into the same deduplication and policy pipeline as user/project/local/manual MCP config

So plugins in Claude Code can provide not only prompt-oriented or markdown-based capabilities, but also real external-service connections.

## 4. Skills: how high-level capability surfaces enter the command system

The most useful skill files to study are:

- `recovered_source/src/skills/bundledSkills.ts`
- `recovered_source/src/skills/loadSkillsDir.ts`
- `recovered_source/src/commands.ts`

### 1. Bundled skills are compiled into the CLI

`recovered_source/src/skills/bundledSkills.ts:11-108` explains the structure and registration flow for bundled skills:

- `BundledSkillDefinition` describes the skill shape
- `registerBundledSkill(...)` stores a skill in the internal registry
- `getBundledSkills()` returns the current bundled-skill set

An especially interesting detail appears in `bundledSkills.ts:30-36` and `:59-72`:

- a skill may include reference files
- those files may be extracted to disk lazily on first invocation
- the prompt gets a base-directory prefix so the model can read/grep those files

That means a skill is not just a prompt string. It can come with a small reference corpus that becomes part of its usable context surface.

### 2. `loadSkillsDir.ts` loads on-disk skills into the runtime

`recovered_source/src/skills/loadSkillsDir.ts:625-803`, in `getSkillDirCommands(cwd)`, is the main entry point for loading disk-based skills.

It gathers skills from multiple sources:

- managed skills directory
- user skills directory
- project skills directories
- explicitly added directories
- legacy commands directories

Then it performs:

- deduplication: `loadSkillsDir.ts:725-769`
- separation into unconditional vs. conditional skills: `loadSkillsDir.ts:771-803`

So skills are not just “scan one folder.” They go through **source aggregation, deduplication, and condition-activation preparation**.

### 3. Dynamic skill discovery

Even more interesting is `loadSkillsDir.ts:861-1038`:

- `discoverSkillDirsForPaths(...)` walks upward from file paths looking for `.claude/skills`
- `addSkillDirectories(...)` loads newly found skill directories into the current session
- `getDynamicSkills()` returns session-discovered dynamic skills
- `activateConditionalSkillsForPaths(...)` activates path-filtered skills based on touched files

That means the skills system is not “fully loaded once at startup and then frozen.” It can continue discovering and activating skills based on what files the current session interacts with.

## 5. Where all of this finally lands: the command system

`recovered_source/src/commands.ts:353-509` is where the story comes together.

The final command set can include:

- skill-directory commands
- plugin skills
- bundled skills
- built-in plugin skills
- plugin commands
- workflow commands
- built-in commands
- dynamic skills

So one major endpoint for MCP, plugins, and skills is the **current session command table** returned by `getCommands(cwd)`.

MCP also has a second, lower-level endpoint: it contributes tools and resources directly into runtime state.

## 6. A single combined picture

A useful simplified diagram is:

```text
startup/main.tsx
  -> initBuiltinPlugins()
  -> initBundledSkills()
  -> getCommands(cwd)
  -> getMcpToolsCommandsAndResources(...)

plugins
  -> plugin commands
  -> plugin skills
  -> plugin MCP servers

skills
  -> bundled skills
  -> user/project/managed skills
  -> dynamic/conditional skills

mcp
  -> external servers
  -> tools + commands + resources + mcp skills

all of the above
  -> flow into the command system / tool system / main session context
```

## Chapter summary

The most important conclusion from this chapter is:

> Claude Code’s extensibility is not implemented through one single plugin point. It is implemented through three interconnected systems — MCP, plugins, and skills — that collectively expand commands, tools, resources, and prompt-oriented capability surfaces.

More concretely:

- MCP is the runtime connection layer for external services
- plugins are the packaged distribution layer for extension content
- skills are the higher-level interaction and prompt-organization layer

All three eventually get absorbed into the runtime’s command table, tool pool, and context-assembly process.

## Next chapter

Continue to [07-permissions-and-security-boundaries_EN.md](./07-permissions-and-security-boundaries_EN.md).
