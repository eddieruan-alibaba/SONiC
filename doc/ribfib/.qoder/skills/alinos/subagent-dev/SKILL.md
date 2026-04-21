---
name: subagent-dev
description: "Use when executing implementation plans with independent tasks in the current session. Dispatches fresh subagent per task with two-stage review."
model: auto
---

# Subagent-Driven Development

Execute plan by dispatching fresh subagent per task, with two-stage review after each: spec compliance review first, then code quality review.

**Core principle:** Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration

## When to Use

- Have implementation plan with mostly independent tasks
- Want to stay in this session
- Alternative: `/alinos.executing-plans` for parallel session

## The Process

1. **Read plan, extract all tasks** - Note context, create TodoWrite
2. **Per task:**
   - Dispatch implementer subagent (Task tool)
   - Handle questions if any
   - Implementer implements, tests, commits, self-reviews
   - Dispatch spec reviewer subagent -> confirm code matches spec
   - If spec issues: implementer fixes, re-review
   - Dispatch code quality reviewer subagent -> approve quality
   - If quality issues: implementer fixes, re-review
   - Mark task complete
3. **After all tasks** - Dispatch final code reviewer for entire implementation
4. **Generate summary** - Use `/alinos.finish-branch` Step 1 to generate `docs/alinos/summary/` document
5. **Use `/alinos.finish-branch`** to complete work

## Handling Implementer Status

- **DONE:** Proceed to spec compliance review
- **DONE_WITH_CONCERNS:** Read concerns before proceeding
- **NEEDS_CONTEXT:** Provide missing context and re-dispatch
- **BLOCKED:** Assess blocker - provide more context, try more capable model, break task smaller, or escalate to human

## Red Flags

**Never:**
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed issues
- Dispatch multiple implementation subagents in parallel (conflicts)
- Make subagent read plan file (provide full text instead)
- **Start code quality review before spec compliance passes**
- Move to next task while either review has open issues

**If subagent asks questions:** Answer clearly and completely
**If reviewer finds issues:** Implementer fixes, reviewer reviews again, repeat until approved
**If subagent fails task:** Dispatch fix subagent with specific instructions

## Integration

- `/alinos.writing-plans` - Creates the plan
- `/alinos.code-review` - Code review template for reviewers
- `/alinos.finish-branch` - Complete development after all tasks
- `/alinos.tdd` - Subagents follow TDD for each task
