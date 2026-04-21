---
description: "Use when starting feature work that needs isolation from current workspace or before executing implementation plans."
model: auto
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously.

**Core principle:** Systematic directory selection + safety verification = reliable isolation.

## Directory Selection Process

### 1. Check Existing Directories
```bash
ls -d .worktrees 2>/dev/null     # Preferred (hidden)
ls -d worktrees 2>/dev/null      # Alternative
```
If found, use that directory. If both exist, `.worktrees` wins.

### 2. Check AGENTS.md
```bash
grep -i "worktree.*director" AGENTS.md 2>/dev/null
```
If preference specified, use it without asking.

### 3. Ask User
```
No worktree directory found. Where should I create worktrees?
1. .worktrees/ (project-local, hidden)
2. Custom location
```

## Safety Verification

**For project-local directories:**

```bash
git check-ignore -q .worktrees 2>/dev/null
```

If NOT ignored: add to .gitignore, commit, then proceed.

## Creation Steps

1. **Detect project name:** `basename "$(git rev-parse --show-toplevel)"`
2. **Create worktree:** `git worktree add "$path" -b "$BRANCH_NAME"`
3. **Run project setup:** Auto-detect (package.json, requirements.txt, etc.)
4. **Verify clean baseline:** Run tests
5. **Report location and test status**

## Quick Reference

| Situation | Action |
|-----------|--------|
| `.worktrees/` exists | Use it (verify ignored) |
| `worktrees/` exists | Use it (verify ignored) |
| Both exist | Use `.worktrees/` |
| Neither exists | Check AGENTS.md -> Ask user |
| Directory not ignored | Add to .gitignore + commit |
| Tests fail during baseline | Report failures + ask |

## Red Flags

**Never:**
- Create worktree without verifying it's ignored (project-local)
- Skip baseline test verification
- Proceed with failing tests without asking
- Assume directory location when ambiguous

## Integration

- `/alinos.subagent-dev` and `/alinos.executing-plans` - REQUIRED before executing tasks
- `/alinos.finish-branch` - REQUIRED for cleanup after work complete
