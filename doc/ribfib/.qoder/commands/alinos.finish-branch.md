---
description: "Use when implementation is complete, all tests pass, and you need to decide how to integrate the work."
model: auto
---

# Finishing a Development Branch

## Overview

Guide completion of development work by generating a summary, verifying tests, and handling integration.

**Core principle:** Summary -> Verify tests -> Commit check -> Present options -> Execute choice -> Clean up.

## The Process

### Step 1: Generate Development Summary

Before any verification or integration, generate a summary document capturing the full picture of what was designed and built.

**Save to:** `docs/alinos/summary/<feature_name>_YYYY_MM_DD_summary.md`

The summary **must** include these sections:

```markdown
# <Feature Name> Summary

## 设计概要
- 设计动机和目标
- 关键设计决策及其理由
- 架构选择（选了什么、为什么、否决了什么）

## 实现概要
- 新增/修改的文件清单（附简要说明）
- 核心实现逻辑（按模块/组件组织）
- 与现有系统的集成方式

## 接口说明
- 对外接口（API、CLI、配置项、DB schema 等）
- 对内接口（模块间调用关系、数据流）
- 接口变更（如有，说明向后兼容策略）

## 测试覆盖
- 测试类型和覆盖范围
- 关键测试场景

## 已知限制与后续事项
- 当前实现的已知限制
- 未来可能的改进方向
```

**Summary 生成规则：**
- 回顾 `docs/alinos/<name>_design.md`（项目整体设计文档，`<name>` 为项目名）获取全局设计上下文
- 回顾 `docs/alinos/` 下的 proposal、design spec、exploration、plan 文档获取当前 feature 的详细设计上下文
- 回顾实际代码变更（git diff against base branch）获取实现细节
- 接口说明必须具体到函数签名、命令格式、schema 字段，不得用泛化描述
- 如有信息不确定，标记 `[待确认]` 而非跳过

### Step 2: Verify Tests

```bash
# Run project's test suite
pytest / npm test / cargo test / go test ./...
```

**If tests fail:** Report failures. Stop. Don't proceed to Step 3.
**If tests pass:** Continue to Step 3.

### Step 3: Commit Check

Check if there are uncommitted changes (staged or unstaged):

```bash
git status --porcelain
```

**If there are uncommitted changes**, ask the user:

```
Uncommitted changes detected. Would you like to consolidate all changes into a single commit before proceeding?

1. Yes, create a commit now (will use /commit-smart)
2. No, I'll handle commits myself later
```

- **Option 1:** Invoke `/commit-smart` to stage all changes and create a semantic commit. After the commit is created, continue to Step 4.
- **Option 2:** Continue to Step 4 without committing. Warn the user that uncommitted changes may be lost depending on the integration choice.

**If there are no uncommitted changes**, skip this step and continue to Step 4.

### Step 4: Determine Base Branch

```bash
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

### Step 5: Present Options

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

### Step 6: Execute Choice

**Option 1: Merge Locally** - checkout base, pull, merge, verify tests, delete branch
**Option 2: Push and Create PR** - push with -u, create PR via gh
**Option 3: Keep As-Is** - report branch name and location
**Option 4: Discard** - confirm first, require typed 'discard', then force delete

### Step 7: Cleanup Worktree

For Options 1, 2, 4: `git worktree remove <path>`
For Option 3: Keep worktree.

## Red Flags

**Never:**
- Proceed with failing tests
- Merge without verifying tests on result
- Delete work without confirmation
- Force-push without explicit request

## Integration

Called by `/alinos.subagent-dev` and `/alinos.executing-plans` after all tasks complete.
Pairs with `/alinos.worktrees` for cleanup.
