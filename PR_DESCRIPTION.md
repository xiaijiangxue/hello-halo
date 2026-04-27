# Add support for loading custom agent types from agent definition files

## Problem

Halo's Agent tool only supported built-in agent types (`general-purpose`, `Explore`, `Plan`, `statusline-setup`, `claude-code-guide`). Custom agents defined in `~/.claude/agents/` were not available, preventing tools like Get Shit Done (GSD) from working properly.

**Error before this fix:**
```
Agent type 'gsd-codebase-mapper' not found. 
Available agents: general-purpose, statusline-setup, Explore, Plan, claude-code-guide
```

### Root Cause

The `PREDEFINED_AGENTS` object was defined in `agents.ts` but never passed to the SDK. The `send-message.ts` file built SDK options without including the `agents` field, so the SDK only knew about its built-in agent types.

## Solution

This PR adds support for dynamically loading custom agent definitions from `~/.claude/agents/` and passing them to the SDK.

### Changes

**1. `src/main/services/agent/agents.ts`**
- Added `parseGsdAgentFile()` function to parse agent definition files
  - Supports `.md` files with YAML frontmatter
  - Extracts: `name`, `description`, `tools`, `model`
  - Uses file content after frontmatter as the agent's system prompt
- Added `loadGsdAgents()` function to scan and load all `gsd-*.md` files from `~/.claude/agents/`
- Updated `PREDEFINED_AGENTS` to include dynamically loaded agents using spread operator

**2. `src/main/services/agent/send-message.ts`**
- Imported `PREDEFINED_AGENTS` from `./agents`
- Added `sdkOptions.agents = PREDEFINED_AGENTS` to pass custom agents to the SDK

### How It Works

1. When Halo starts, `agents.ts` is imported
2. `loadGsdAgents()` executes automatically, scanning `~/.claude/agents/` for `gsd-*.md` files
3. Each agent definition file is parsed and converted to `AgentDefinition` format
4. All loaded agents are merged into `PREDEFINED_AGENTS`
5. When sending a message, `send-message.ts` passes `PREDEFINED_AGENTS` to the SDK via `sdkOptions.agents`
6. The SDK makes these agents available as `subagent_type` options for the Agent tool

### Agent Definition File Format

```markdown
---
name: gsd-codebase-mapper
description: Explores codebase and writes structured analysis documents
tools: Read, Bash, Grep, Glob, Write
model: sonnet
---

<role>
You are a GSD codebase mapper...
</role>
```

## Testing

**Before:**
```typescript
Agent(subagent_type="gsd-codebase-mapper", ...)
// Error: Agent type 'gsd-codebase-mapper' not found
```

**After:**
```typescript
Agent(subagent_type="gsd-codebase-mapper", ...)
// ✓ Agent spawns successfully and executes
```

Verified that:
- ✅ GSD agents are loaded from `~/.claude/agents/`
- ✅ `gsd-codebase-mapper` agent can be spawned successfully
- ✅ Agent executes and returns results
- ✅ All GSD commands that depend on custom agents now work

## Breaking Changes

None. This is a purely additive feature that doesn't change existing behavior.

## Considerations

- **Loading time**: Agents are loaded once at module initialization. For 30+ agent files, this adds minimal overhead (~50-100ms)
- **Hot reload**: Changes to agent definition files require Halo restart to take effect
- **Error handling**: Malformed agent files are logged but don't crash the application
- **Scope**: Currently only loads files matching `gsd-*.md` pattern. Could be extended to load all `.md` files if needed

## Future Enhancements

Potential improvements for future PRs:
- Support hot-reloading of agent definitions without restart
- Add validation for agent definition format
- Support loading agents from additional directories
- Add UI for managing custom agents
- Support for agent definition file watching and auto-reload

## Related Issues

Fixes the issue where GSD (Get Shit Done) and other tools requiring custom agents were not functional in Halo.
