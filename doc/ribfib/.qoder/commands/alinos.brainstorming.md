---
description: "Use before any creative work - creating features, building components, adding functionality, or modifying behavior. Explores intent, requirements and design before implementation."
model: ultimate
---

# Brainstorming Ideas Into Designs

Help turn ideas into fully formed designs and specs through natural collaborative dialogue.

<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it.
</HARD-GATE>

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** -- check files, docs, recent commits
2. **Ask clarifying questions** -- one at a time, understand purpose/constraints/success criteria
3. **Propose 2-3 approaches** -- with trade-offs and your recommendation
4. **Present design** -- in sections scaled to their complexity, get user approval after each section
5. **Write design doc** -- save to `docs/alinos/specs/YYYY-MM-DD-<topic>-design.md` and commit
6. **Spec self-review** -- check for placeholders, contradictions, ambiguity, scope
7. **User reviews written spec** -- ask user to review the spec file before proceeding
8. **Update consolidated design** -- merge into `docs/alinos/<topic>_design.md` (see below)
9. **Transition to implementation** -- invoke `/alinos.writing-plans` to create implementation plan

**Step 0 (project rules)**: Read AGENTS.md for project-specific rules bound to this skill. If rules exist, execute them BEFORE proceeding with the standard steps below. Project rules have HIGHER priority than default steps — they may add pre-steps, modify the workflow, or require additional output sections.

## The Process

**Understanding the idea:**
- Check out the current project state first (files, docs, recent commits)
- Before asking detailed questions, assess scope: if request describes multiple independent subsystems, flag this immediately
- Ask questions one at a time, prefer multiple choice when possible
- Focus on: purpose, constraints, success criteria

**Exploring approaches:**
- Propose 2-3 different approaches with trade-offs
- Lead with your recommended option and explain why

**Presenting the design:**
- Scale each section to its complexity
- Ask after each section whether it looks right
- Cover: architecture, components, data flow, error handling, testing

**Design for isolation and clarity:**
- Break into smaller units with one clear purpose
- Well-defined interfaces, testable independently

**Working in existing codebases:**
- Follow existing patterns
- Include targeted improvements where existing code has problems affecting the work
- Don't propose unrelated refactoring

## After the Design

**Documentation:**
- Write validated design to `docs/alinos/specs/YYYY-MM-DD-<topic>-design.md`
- Commit the design document to git

**Spec Self-Review:**
1. Placeholder scan: Any "TBD", "TODO", incomplete sections?
2. Internal consistency: Do sections contradict each other?
3. Scope check: Focused enough for a single implementation plan?
4. Ambiguity check: Could any requirement be interpreted two ways?

**User Review Gate:**
> "Spec written and committed. Please review and let me know if you want changes before we write the implementation plan."

**Update Consolidated Design Document:**

After user approves the spec, update the project-level consolidated design document at `docs/alinos/<topic>_design.md` (where `<topic>` is the same name used in the spec filename `YYYY-MM-DD-<topic>-design.md`). This document represents the **current state of the entire project's design** — a single source of truth that integrates all feature designs.

<IMPORTANT>
**命名规范**: 文件名必须包含项目名，格式为 `docs/alinos/<topic>_design.md`，例如 `docs/alinos/fuzzy-test_design.md`。
**语言规范**: 该文件必须全中文撰写，包括标题、正文、变更记录。指令中的英文仅用于描述流程，产出物内容一律使用中文。
</IMPORTANT>

1. **Read existing `docs/alinos/<topic>_design.md`** (if it exists)
2. **Read the feature design spec** just approved (`docs/alinos/specs/YYYY-MM-DD-<topic>-design.md`)
3. **Merge into the consolidated document:**
   - If the file doesn't exist yet: Create it with the structure below and the current feature as the first section
   - If the feature already has a section: Replace that section with the updated content
   - If the feature is new: Add a new section in the appropriate location
4. **Document structure (Chinese):**
   ```markdown
   # 项目设计

   [项目概述段落]

   ## [功能名称 A]
   [该功能的核心设计：架构决策、数据模型、接口、约束]

   ## [功能名称 B]
   ...

   ## 变更记录
   | 日期 | 功能 | 变更说明 |
   |------|------|---------|
   | YYYY-MM-DD | 功能名称 | 新增/更新说明 |
   ```
5. **Organization rules:**
   - Each feature section contains the **distilled design** — key decisions, architecture, interfaces, constraints — not a full copy of the spec
   - Order sections logically by component/module relationship, not by creation date
6. **Write the updated `docs/alinos/<topic>_design.md`**

**Content guidelines:**
- **提炼而非复制**: 提取核心设计要素 — 架构决策、数据模型、接口定义、约束条件
- **维护交叉引用**: 功能间有交互时，注明依赖关系
- **保持最新**: 只反映最新决策，不保留历史版本
- **清除过时内容**: 设计决策被替代时，直接更新，不保留旧内容

**Implementation:**
- Invoke `/alinos.writing-plans` to create implementation plan
- Do NOT invoke any other skill. writing-plans is the next step.

## Key Principles

- **One question at a time** - Don't overwhelm
- **Multiple choice preferred** - Easier to answer
- **YAGNI ruthlessly** - Remove unnecessary features
- **Explore alternatives** - Always 2-3 approaches
- **Incremental validation** - Present design, get approval
