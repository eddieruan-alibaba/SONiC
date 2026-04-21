---
description: "Use when you have a spec or requirements for a multi-step task, before touching code. Creates detailed implementation plans with bite-sized TDD steps."
model: ultimate
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context. Document everything: which files to touch, code, testing, docs. Bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

**Save plans to:** `docs/alinos/plans/YYYY-MM-DD-<feature-name>.md`

## Scope Check

If spec covers multiple independent subsystems, suggest breaking into separate plans. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified. Design units with clear boundaries. Prefer smaller, focused files. Files that change together should live together.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

Every plan MUST start with:

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** Use /alinos.subagent-dev (recommended) or /alinos.executing-plans to implement this plan task-by-task.

**Goal:** [One sentence]
**Architecture:** [2-3 sentences]
**Tech Stack:** [Key technologies]
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**
- [ ] **Step 2: Run test to verify it fails**
- [ ] **Step 3: Write minimal implementation**
- [ ] **Step 4: Run test to verify it passes**
- [ ] **Step 5: Commit**
````

## No Placeholders

Every step must contain actual content. These are **plan failures**:
- "TBD", "TODO", "implement later"
- "Add appropriate error handling"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code)

## Self-Review

After writing the complete plan:
1. **Spec coverage:** Can you point to a task for each spec requirement?
2. **Placeholder scan:** Any red flags from "No Placeholders" section?
3. **Type consistency:** Do types/method names match across tasks?

## Multi-Person Development

After Self-Review, **must ask the user:**

> "本次开发是否需要多人协作？如果是，请告知参与的开发人员数目和各自名称/代号。"

### If single person (or user declines)

Skip this section, proceed to Execution Handoff.

### If multiple persons

Based on the number of developers provided, restructure the plan into **per-person assignment**:

1. **Dependency analysis:** Identify task dependencies and parallelizable groups
2. **Person assignment:** Assign tasks to each developer, principles:
   - Minimize cross-person dependencies (reduce blocking)
   - Group related tasks to the same person (reduce context switching)
   - Balance workload across developers
   - Mark inter-person handoff points clearly
3. **Pair programming pairing:** For tasks with high complexity or cross-component coupling, designate pair programming sessions:
   - Mark which developer drives and which reviews
   - Identify shared interface tasks that require synchronization

### Per-Person Plan Format

````markdown
## Developer Assignment

### [Person A Name]

**Responsible tasks:** Task 1, Task 3, Task 5
**Pairs with [Person B] on:** Task 2 (interface definition)

| Task | Dependencies | Status |
|------|-------------|--------|
| Task 1: [name] | None (can start immediately) | ⬜ |
| Task 3: [name] | Waits for Task 2 interface | ⬜ |
| Task 5: [name] | After Task 1 | ⬜ |

### [Person B Name]

**Responsible tasks:** Task 2, Task 4, Task 6
**Pairs with [Person A] on:** Task 2 (interface definition)

| Task | Dependencies | Status |
|------|-------------|--------|
| Task 2: [name] | None (can start immediately) | ⬜ |
| Task 4: [name] | After Task 2 | ⬜ |
| Task 6: [name] | Waits for Task 5 output | ⬜ |

### Synchronization Points

- [ ] **Sync 1:** After Task 2 — A and B align on interface contract
- [ ] **Sync 2:** After Task 5 + Task 4 — Integration test together
````

### Assignment Principles

- **Interface tasks** (shared types, API contracts): Pair programming, define together first
- **Leaf tasks** (no downstream dependents): Freely parallelizable, assign to balance load
- **Sequential chains**: Assign entire chain to one person to avoid handoff overhead
- **Integration tasks**: Schedule after all dependent tasks, assign to the person with most context

## Execution Handoff

After saving the plan:

> "Plan complete. Two execution options:
> 1. **Subagent-Driven (recommended)** - `/alinos.subagent-dev`
> 2. **Inline Execution** - `/alinos.executing-plans`
> Which approach?"
