---
description: "Use when creating new skills, editing existing skills, or verifying skills work before deployment."
model: ultimate
---

# Writing Skills

## Overview

**Writing skills IS Test-Driven Development applied to process documentation.**

You write test cases (pressure scenarios with subagents), watch them fail (baseline behavior), write the skill (documentation), watch tests pass (agents comply), and refactor (close loopholes).

**Core principle:** If you didn't watch an agent fail without the skill, you don't know if the skill teaches the right thing.

## What is a Skill?

A **skill** is a reference guide for proven techniques, patterns, or tools.

**Skills are:** Reusable techniques, patterns, tools, reference guides
**Skills are NOT:** Narratives about how you solved a problem once

## When to Create a Skill

**Create when:**
- Technique wasn't intuitively obvious
- You'd reference this again across projects
- Pattern applies broadly
- Others would benefit

**Don't create for:**
- One-off solutions
- Standard practices well-documented elsewhere
- Project-specific conventions (put in AGENTS.md)

## SKILL.md / Command Structure

```markdown
---
description: "Use when [specific triggering conditions]"
---

# Skill Name

## Overview
Core principle in 1-2 sentences.

## When to Use
Symptoms and use cases. When NOT to use.

## Core Pattern
Before/after comparison.

## Quick Reference
Table for scanning.

## Common Mistakes
What goes wrong + fixes.
```

## The Iron Law (Same as TDD)

```
NO SKILL WITHOUT A FAILING TEST FIRST
```

## RED-GREEN-REFACTOR for Skills

### RED: Write Failing Test (Baseline)
Run pressure scenario WITHOUT the skill. Document exact behavior.

### GREEN: Write Minimal Skill
Write skill addressing those specific rationalizations. Verify agents now comply.

### REFACTOR: Close Loopholes
Found new rationalization? Add explicit counter. Re-test until bulletproof.

## Skill Creation Checklist

**RED Phase:**
- [ ] Create pressure scenarios
- [ ] Run scenarios WITHOUT skill - document baseline
- [ ] Identify patterns in failures

**GREEN Phase:**
- [ ] Description starts with "Use when..."
- [ ] Clear overview with core principle
- [ ] Address specific baseline failures
- [ ] Verify agents now comply

**REFACTOR Phase:**
- [ ] Identify new rationalizations
- [ ] Add explicit counters
- [ ] Re-test until bulletproof
