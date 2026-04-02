# Claude Code Source Learning Tutorial 07: Permissions and Security Boundaries

[中文版本](./07-permissions-and-security-boundaries.md)

## Questions this chapter answers

After reading this chapter, you should be able to answer:

- Why is the permission system a first-class architectural layer in Claude Code?
- How do permission modes and allow / ask / deny rules work together?
- Where do the real safety boundaries live for filesystem access, shell execution, plan mode, and sub-agents?

## Why the permission system is a first-class architectural layer

Earlier chapters showed that Claude Code is a tool-driven terminal agent. As soon as a system allows an LLM to read files, edit files, run shell commands, and spawn agents, it must answer two questions:

1. **What capabilities are exposed at all?**
2. **Under what conditions are those capabilities allowed to execute?**

That is why the permission system in Claude Code is not just a UI confirmation dialog. It is a control layer that cuts across:

- tool visibility
- pre-execution validation
- mode transitions
- filesystem path protection
- shell-risk reduction
- plan mode and sub-agent behavior

## 1. Permission modes: the system first decides the default operating posture

The first file to read is:

- `recovered_source/src/utils/permissions/PermissionMode.ts`

`PermissionMode.ts:42-90` shows the main modes in Claude Code:

- `default`
- `plan`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `auto` (when the feature is enabled)

That means Claude Code does not govern behavior only one request at a time through allow/ask/deny decisions. It first gives the session a broader **default permission posture**.

### What these modes mean in practice

A useful working interpretation is:

- **default**: normal mode, using the standard permission flow
- **plan**: prioritize exploration and planning before implementation
- **acceptEdits**: more permissive for a narrow set of edit-like filesystem commands
- **bypassPermissions**: a high-risk mode that skips most permission barriers
- **dontAsk**: minimizes prompting and relies on surrounding policy behavior
- **auto**: tied to classifier-mediated automation, including dangerous-rule stripping

The most important idea here is:

> a permission mode defines the runtime’s baseline behavior posture, not just one tool’s implementation detail

## 2. Dangerous-rule stripping: why allow rules themselves can be risky

The most important file here is:

- `recovered_source/src/utils/permissions/permissionSetup.ts`

### 1. Risk does not live only in deny or bypass logic

A common intuition is that only explicit bypass is dangerous. Claude Code’s implementation is more careful than that.

`permissionSetup.ts:472-503`, in `removeDangerousPermissions(...)`, and `permissionSetup.ts:510-552`, in `stripDangerousPermissionsForAutoMode(...)`, show that:

- some allow rules themselves are dangerous enough to bypass classifier-based safety
- those rules are stripped when entering auto mode
- stripped rules are stashed in `strippedDangerousRules`

So Claude Code is not asking only “is the current mode dangerous?” It is also asking:

> would a previously saved allow rule blow past the intended safety layer in this mode?

### 2. Those rules are restored when leaving auto mode

`permissionSetup.ts:561-579`, in `restoreDangerousPermissions(...)`, restores previously stashed rules when auto mode is left.

That means the system is not simply deleting rules forever. It is dynamically adjusting which rules may remain active under a given permission mode.

### 3. Mode transitions are treated like a state machine

`permissionSetup.ts:597-646`, in `transitionPermissionMode(...)`, is especially important because it centralizes side effects for:

- entering and exiting plan mode
- activating and deactivating auto mode
- stripping and restoring dangerous rules
- cleaning up plan-related state on exit

This shows that a permission mode is not merely a string flag. It is a **state transition system with side effects and restoration logic**.

## 3. Filesystem boundaries: some paths are intrinsically more sensitive

The most representative file here is:

- `recovered_source/src/utils/permissions/filesystem.ts`

### 1. Dangerous files and directories are explicitly listed

`filesystem.ts:53-79` defines:

- `DANGEROUS_FILES`
  - such as `.gitconfig`, `.bashrc`, `.zshrc`, `.mcp.json`, `.claude.json`
- `DANGEROUS_DIRECTORIES`
  - such as `.git`, `.vscode`, `.idea`, `.claude`

These are sensitive because they can affect:

- shell behavior
- Git behavior
- Claude Code’s own configuration
- IDE configuration
- extension loading and connection behavior

In other words, this layer is not just protecting ordinary content files. It is protecting **files and directories that can reshape the runtime environment itself**.

### 2. Auto-edit behavior is explicitly constrained around those paths

`filesystem.ts:435-478`, in `isDangerousFilePathToAutoEdit(...)`, identifies paths that should not be auto-edited.

That shows Claude Code’s attitude is not “let the model edit first and notice later.” Instead, it proactively protects high-risk paths — especially against automatic approval flows.

### 3. `.claude` is treated with especially high sensitivity

From `filesystem.ts:224-241` and the comments around `:1242`, it is clear that Claude Code treats its own config area — including commands, agents, and skills directories — as an especially sensitive part of the filesystem.

That reflects an important architectural principle:

> the system must treat files that can modify the system’s own behavior as higher-sensitivity targets

## 4. Shell safety: why Bash permissions are not just string matching

Bash is one of the easiest places for boundaries to collapse, so it has one of the deepest safety layers. The key files are:

- `recovered_source/src/tools/BashTool/bashPermissions.ts`
- `recovered_source/src/tools/BashTool/modeValidation.ts`

### 1. `getSimpleCommandPrefix(...)` exists for safe matching, not convenience

`bashPermissions.ts:148-188`, in `getSimpleCommandPrefix(...)`, tries to extract a stable command prefix such as:

- `git commit -m ...` -> `git commit`
- `NODE_ENV=prod npm run build` -> `npm run`

But there is an important safety restriction:

- it strips leading env vars only if they are in `SAFE_ENV_VARS`
- the moment it encounters a non-safe env var, it falls back instead of pretending the command is safely normalizable

That means Claude Code is explicitly aware that:

> some environment variables are not cosmetic — they can fundamentally change how a command loads code, modules, or runtime behavior

### 2. Why `SAFE_ENV_VARS` matters

`bashPermissions.ts:368-430` defines `SAFE_ENV_VARS`, and the comments explicitly forbid adding variables like:

- `PATH`
- `LD_PRELOAD`
- `LD_LIBRARY_PATH`
- `PYTHONPATH`
- `NODE_OPTIONS`
- and similar variables that can affect code loading or execution

So shell permission matching is not only about command names. It also understands that some environment variables can completely change what “the same command” really does.

### 3. Why `BARE_SHELL_PREFIXES` is important

`bashPermissions.ts:196-226` defines `BARE_SHELL_PREFIXES`, including:

- `sh`
- `bash`
- `zsh`
- `cmd`
- `powershell`
- `env`
- `xargs`
- `nice`
- `nohup`
- `timeout`
- `sudo`
- `doas`
- `pkexec`

This means Claude Code explicitly treats these wrappers and shells as high-risk prefixes that should not be auto-suggested or safely normalized as if they were ordinary commands.

The reasoning is straightforward:

- `bash -c ...` can execute arbitrary code
- `env ...` can rewrite execution context
- `sudo` / `pkexec` introduce privilege escalation
- wrappers can obscure the real underlying command

### 4. `acceptEdits` is not broad shell bypass

`recovered_source/src/tools/BashTool/modeValidation.ts:7-15` defines the limited set of commands auto-allowed in `acceptEdits` mode:

- `mkdir`
- `touch`
- `rm`
- `rmdir`
- `mv`
- `cp`
- `sed`

And `modeValidation.ts:37-49` only auto-allows those commands when the current mode is `acceptEdits`.

So:

> `acceptEdits` does not mean “open Bash entirely.” It means “auto-approve a narrow, explicitly declared set of edit-like filesystem operations.”

## 5. Plan mode: why “plan first” is also a safety boundary

Plan mode may look like a workflow feature, but it has a clear permission meaning too. The key files are:

- `recovered_source/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`
- `recovered_source/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`

### 1. Entering plan mode pushes the runtime into an exploration-first posture

`EnterPlanModeTool.ts:77-101` shows that entering plan mode:

- is rejected inside agent contexts
- calls `prepareContextForPlanMode(...)`
- updates the session permission mode to `plan`

The tool result also explicitly instructs the model to focus on exploration and design rather than writing or editing files immediately.

So plan mode is not just a UX nicety. It pushes the conversation into a **more conservative, analysis-oriented operational posture**.

### 2. Exiting plan mode requires confirmation and mode restoration

`ExitPlanModeV2Tool.ts:195-238` shows several important constraints:

- it validates that the current mode is actually `plan`
- for non-teammates, it requires explicit user confirmation to exit plan mode

Then `ExitPlanModeV2Tool.ts:318-345` handles:

- restoring mode state on exit
- falling back safely when auto-mode gates are unavailable

So leaving plan mode is also a controlled state transition, not just a casual toggle.

## 6. Sub-agents: the permission system can filter agents too

If you think the permission system only governs Bash, Read, and Edit, `recovered_source/src/utils/permissions/permissions.ts` shows otherwise.

### 1. A specific agent type can be denied directly

`permissions.ts:305-320`, in `getDenyRuleForAgent(...)`, allows rules like `Agent(Explore)` to explicitly deny a certain agent type.

### 2. Agent lists are filtered before use

`permissions.ts:323-343`, in `filterDeniedAgents(...)`, removes denied agent types from the candidate agent set.

So sub-agents are not treated as a hidden backdoor capability. They are part of the same governed capability surface.

### 3. Headless/async agents still go through permission hooks

`permissions.ts:392-419`, in `runPermissionRequestHooksForHeadlessAgent(...)`, shows that:

- if an agent cannot show local permission UI, hooks still get a chance to decide
- only if no hook decision is made does the system fall through to later denial behavior

That means Claude Code also has a plan for handling permissions in environments without interactive approval dialogs.

## 7. Put the whole permission system back into the broader architecture

A useful layered model is:

```text
permission mode
  -> defines the session’s default operating posture

permission rules
  -> allow / ask / deny rules narrow or widen specific behavior

filesystem boundaries
  -> protect dangerous files, dangerous directories, and Claude’s own config areas

shell safety
  -> constrain env-var stripping, wrapper handling, privilege boundaries, and mode-specific exceptions

workflow boundaries
  -> plan mode, sub-agents, and headless hook handling control complex multi-stage behavior
```

These layers work together so that Claude Code can expose powerful tools while still trying to keep the risk surface understandable and constrained.

## Recommended reading path

If you want to keep digging into the permission system, read in this order:

1. `recovered_source/src/utils/permissions/PermissionMode.ts`
2. `recovered_source/src/utils/permissions/permissionSetup.ts`
3. `recovered_source/src/utils/permissions/filesystem.ts`
4. `recovered_source/src/tools/BashTool/bashPermissions.ts`
5. `recovered_source/src/tools/BashTool/modeValidation.ts`
6. `recovered_source/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`
7. `recovered_source/src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`
8. `recovered_source/src/utils/permissions/permissions.ts`

## Chapter summary

The most important conclusion from this chapter is:

> Claude Code’s permission system is not just a prompt layer. It is an architectural control system spanning modes, rules, path validation, shell safety, and multi-stage workflow boundaries.

What it protects is not only whether one tool call may execute, but also:

- how much capability is exposed in the current session
- which paths should never be auto-touched casually
- which shell patterns could bypass intended restrictions
- whether complex workflows are still operating under safe default assumptions

## Next chapter

Continue to [99-reading-paths-and-next-steps_EN.md](./99-reading-paths-and-next-steps_EN.md).
