---
description: "Use when facing 2+ independent tasks that can be worked on without shared state or sequential dependencies."
model: auto
---

# Dispatching Parallel Agents

## Overview

When you have multiple unrelated failures (different test files, different subsystems, different bugs), investigating them sequentially wastes time.

**Core principle:** Dispatch one agent per independent problem domain. Let them work concurrently.

## When to Use

**Use when:**
- 3+ test files failing with different root causes
- Multiple subsystems broken independently
- Each problem can be understood without context from others
- No shared state between investigations

**Don't use when:**
- Failures are related (fix one might fix others)
- Need to understand full system state
- Agents would interfere with each other

## The Pattern

### 1. Identify Independent Domains

Group failures by what's broken. Each domain is independent.

### 2. Create Focused Agent Tasks

Each agent gets:
- **Specific scope:** One test file or subsystem
- **Clear goal:** Make these tests pass
- **Constraints:** Don't change other code
- **Expected output:** Summary of what you found and fixed

### 3. Dispatch in Parallel

Use the Task tool to dispatch multiple agents simultaneously:
```
Task("Fix component-A failures")
Task("Fix component-B failures")
Task("Fix component-C failures")
```

### 4. Review and Integrate

- Read each summary
- Verify fixes don't conflict
- Run full test suite
- Integrate all changes

## Agent Prompt Structure

Good agent prompts are:
1. **Focused** - One clear problem domain
2. **Self-contained** - All context needed
3. **Specific about output** - What should the agent return?

## Common Mistakes

- **Too broad:** "Fix all the tests" -> agent gets lost
- **No context:** Paste error messages and test names
- **No constraints:** Agent might refactor everything
- **Vague output:** "Fix it" -> you don't know what changed

## Verification

After agents return:
1. Review each summary
2. Check for conflicts
3. Run full suite
4. Spot check - agents can make systematic errors
