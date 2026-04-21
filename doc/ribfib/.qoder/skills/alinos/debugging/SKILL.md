---
name: debugging
description: "Use when encountering any bug, test failure, or unexpected behavior, before proposing fixes. Enforces root cause investigation."
model: auto
---

# Systematic Debugging

## Overview

Random fixes waste time and create new bugs. Quick patches mask underlying issues.

**Core principle:** ALWAYS find root cause before attempting fixes. Symptom fixes are failure.

## The Iron Law

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

If you haven't completed Phase 1, you cannot propose fixes.

## When to Use

Use for ANY technical issue: test failures, bugs, unexpected behavior, performance problems, build failures, integration issues.

**Use this ESPECIALLY when:**
- Under time pressure (emergencies make guessing tempting)
- "Just one quick fix" seems obvious
- You've already tried multiple fixes
- Previous fix didn't work

## The Four Phases

### Phase 1: Root Cause Investigation

**BEFORE attempting ANY fix:**

1. **Read Error Messages Carefully** - Don't skip errors/warnings. Read stack traces completely. Note line numbers, file paths, error codes.
2. **Reproduce Consistently** - Can you trigger it reliably? If not reproducible, gather more data, don't guess.
3. **Check Recent Changes** - Git diff, recent commits, new dependencies, config changes.
4. **Gather Evidence in Multi-Component Systems** - Add diagnostic instrumentation at each component boundary. Run once to gather evidence showing WHERE it breaks.
5. **Trace Data Flow** - Where does bad value originate? Keep tracing up until you find the source. Fix at source, not at symptom.

### Phase 2: Pattern Analysis

1. **Find Working Examples** - Locate similar working code in same codebase
2. **Compare Against References** - Read reference implementation COMPLETELY
3. **Identify Differences** - List every difference, however small
4. **Understand Dependencies** - What other components does this need?

### Phase 3: Hypothesis and Testing

1. **Form Single Hypothesis** - "I think X is the root cause because Y"
2. **Test Minimally** - SMALLEST possible change, one variable at a time
3. **Verify Before Continuing** - Didn't work? Form NEW hypothesis. DON'T add more fixes on top.

### Phase 4: Implementation

1. **Create Failing Test Case** - Use TDD skill for writing proper failing tests
2. **Implement Single Fix** - ONE change at a time. No "while I'm here" improvements.
3. **Verify Fix** - Test passes? No other tests broken?
4. **If Fix Doesn't Work** - If < 3 attempts: Return to Phase 1. **If >= 3: STOP and question the architecture.**
5. **If 3+ Fixes Failed: Question Architecture** - Discuss with your human partner before attempting more fixes.

## Red Flags - STOP and Follow Process

- "Quick fix for now, investigate later"
- "Just try changing X and see if it works"
- "It's probably X, let me fix that"
- Proposing solutions before tracing data flow
- **"One more fix attempt" (when already tried 2+)**

**ALL of these mean: STOP. Return to Phase 1.**

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Issue is simple" | Simple issues have root causes too. |
| "Emergency, no time" | Systematic debugging is FASTER than guess-and-check. |
| "Just try this first" | First fix sets the pattern. Do it right. |
| "I see the problem" | Seeing symptoms != understanding root cause. |
| "One more fix attempt" (after 2+) | 3+ failures = architectural problem. |

## Quick Reference

| Phase | Key Activities | Success Criteria |
|-------|---------------|------------------|
| **1. Root Cause** | Read errors, reproduce, check changes | Understand WHAT and WHY |
| **2. Pattern** | Find working examples, compare | Identify differences |
| **3. Hypothesis** | Form theory, test minimally | Confirmed or new hypothesis |
| **4. Implementation** | Create test, fix, verify | Bug resolved, tests pass |

## Real-World Impact

- Systematic approach: 15-30 minutes to fix
- Random fixes approach: 2-3 hours of thrashing
- First-time fix rate: 95% vs 40%
