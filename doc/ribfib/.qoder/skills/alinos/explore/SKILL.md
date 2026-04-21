---
name: explore
description: "Use after initial proposal is generated, before implementation planning. Conducts Socratic self-examination of design decisions to surface and resolve all ambiguities, assumptions, and trade-offs."
model: ultimate
---

# Socratic Design Exploration

Systematically interrogate a design proposal through structured self-dialogue. Surface every assumption, challenge every decision, resolve every ambiguity — before a single line of code is planned.

<HARD-GATE>
Do NOT proceed to implementation planning or write any code until ALL identified design questions have explicit, reasoned resolutions documented in the output.
</HARD-GATE>

## Core Principle

**A design you haven't questioned is a design you don't understand.**

Every design contains implicit decisions. Implicit decisions are unexamined assumptions. Unexamined assumptions become bugs, rewrites, and scope creep. The purpose of this phase is to make every decision explicit and justified.

## When to Use

- After generating an initial proposal/design document
- Before entering implementation planning (brainstorming / writing-plans)
- When a design "feels right" but hasn't been stress-tested

## Input

- The initial proposal or design document from Stage 1
- Any project context (existing codebase, constraints, tech stack)

## Output

All exploration artifacts are written under the `docs/alinos/` directory tree:

- **Exploration report:** `docs/alinos/explorations/YYYY-MM-DD-<name>-exploration.md`
- **Refined design:** Update the existing `docs/alinos/specs/YYYY-MM-DD-<name>-design.md` in-place

## The Socratic Method for Design

Six categories of probing, applied systematically to every design decision:

| Category | Core Question | Purpose |
|----------|--------------|---------|
| **Clarification** | What exactly do we mean by X? | Eliminate vague terms |
| **Assumptions** | What are we taking for granted? | Surface hidden dependencies |
| **Evidence** | What supports this choice? | Distinguish reasoning from intuition |
| **Alternatives** | What other approaches exist? | Ensure we're not anchored to first idea |
| **Implications** | What follows if we do X? | Trace second and third-order effects |
| **Meta** | Is this the right problem to solve? | Challenge problem framing itself |

**Step 0 (project rules)**: Read AGENTS.md for project-specific rules bound to this skill. If rules exist, execute them BEFORE proceeding with the standard steps below. Project rules have HIGHER priority than default steps — they may add pre-steps, modify the workflow, or require additional output sections.

## The Process

### Phase 1: Extract Design Decisions

Read the initial proposal. Extract every design decision, both **explicit** (stated) and **implicit** (assumed).

For each, create a decision record:

```
Decision: [What was decided]
Type: explicit | implicit
Category: architecture | data model | API | behavior | integration | error handling | security | performance | UX
```

**Implicit decisions are more dangerous than explicit ones.** Look for:
- Technology choices not justified
- Data structures assumed but not discussed
- Error paths not mentioned
- Boundary conditions not defined
- Integration points glossed over
- "Obviously we'd do X" — why is it obvious?

### Phase 2: Socratic Examination

For EACH decision, conduct a structured dialogue. You play both roles:

**Questioner** (challenges): Asks probing questions, surfaces problems, demands justification.
**Defender** (justifies): Provides reasoning, evidence, trade-off analysis.

The dialogue follows this structure:

```
## Decision: [Name]

**Q: Clarification** — What exactly does [X] mean in this context?
**A:** [Precise definition, no ambiguity]

**Q: Assumptions** — What must be true for this to work?
**A:** [List every dependency and precondition]

**Q: Evidence** — Why this approach over alternatives?
**A:** [Concrete reasoning: performance data, existing patterns, constraints]

**Q: Alternatives** — What are 2-3 other ways to do this?
**A:** [Each alternative with trade-offs]
  - Alternative A: [pros/cons]
  - Alternative B: [pros/cons]
  - Chosen approach: [pros/cons and why it wins]

**Q: Implications** — What are the second-order effects?
**A:** [Impact on other components, future extensibility, operational burden]

**Q: Meta** — Are we solving the right problem?
**A:** [Confirm problem framing or reframe]

**Resolution:** [Final justified decision, 1-2 sentences]
```

Not every decision needs all six categories. Scale depth to the decision's impact:

| Impact Level | Scope | Depth |
|-------------|-------|-------|
| **High** | Architecture, data model, core abstractions | All 6 categories, thorough |
| **Medium** | API design, error handling, integration | 3-4 most relevant categories |
| **Low** | Naming, file structure, minor implementation details | 1-2 categories, brief |

### Phase 3: Cross-Cutting Concerns

After examining individual decisions, examine how they interact:

1. **Consistency check** — Do decisions contradict each other?
2. **Dependency map** — Which decisions depend on others? Are there circular dependencies?
3. **Risk concentration** — Are multiple high-risk decisions stacked in one area?
4. **Gap analysis** — What questions does the design NOT answer that it should?

For each issue found, return to Phase 2 and examine it.

### Phase 4: Resolution Synthesis

Produce a **Design Exploration Report** that contains:

```markdown
# Design Exploration: [Feature Name]

## Decisions Examined
[Total count, breakdown by impact level]

## Key Findings

### Confirmed Decisions
[Decisions that survived questioning — with brief justification]

### Revised Decisions
[Decisions that changed during exploration — what changed and why]

### New Decisions
[Questions that emerged and were resolved during exploration]

### Open Questions (if any)
[Questions that genuinely require user input — not laziness]

## Refined Design
[Updated design incorporating all resolutions. This replaces the initial proposal as the source of truth.]

## Risk Register
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| ...  | ...       | ...    | ...        |
```

## Quality Gates

Before declaring exploration complete:

- [ ] Every explicit design decision has been examined
- [ ] At least 3 implicit decisions have been surfaced and examined
- [ ] Every High-impact decision has alternatives documented
- [ ] No decision justified only by "it's obvious" or "it's standard"
- [ ] Cross-cutting consistency check passed
- [ ] Gap analysis found no unanswered structural questions
- [ ] Refined design is self-consistent and complete

**Cannot check all boxes? Keep exploring.**

## Anti-Patterns

| Anti-Pattern | What It Looks Like | Fix |
|-------------|-------------------|-----|
| **Rubber-stamp dialogue** | Q and A that always agree | Questioner MUST find at least one real concern per High-impact decision |
| **Shallow alternatives** | "We could also use [obviously worse thing]" | Alternatives must be genuinely viable |
| **Question avoidance** | Skipping categories for "simplicity" | Scale by impact level, but don't skip High-impact examination |
| **Premature closure** | "This is fine, let's move on" | Resolution must include explicit reasoning |
| **Analysis paralysis** | 50 questions about button color | Scale depth to impact. Low-impact = brief |

## Update Consolidated Design Document

After completing the exploration and updating the refined design spec, update the project-level consolidated design document at `docs/alinos/<name>_design.md` (where `<name>` is extracted from the spec filename `YYYY-MM-DD-<name>-design.md`). This document represents the **current state of the entire project's design** — a single source of truth that integrates all feature designs.

<IMPORTANT>
**命名规范**: 文件名必须包含项目名，格式为 `docs/alinos/<name>_design.md`，例如 `docs/alinos/fuzzy-test_design.md`。
**语言规范**: 该文件必须全中文撰写，包括标题、正文、变更记录。指令中的英文仅用于描述流程，产出物内容一律使用中文。
</IMPORTANT>

**Update process:**
1. **Read existing `docs/alinos/<name>_design.md`** (if it exists)
2. **Read the refined design spec** (`docs/alinos/specs/YYYY-MM-DD-<name>-design.md`)
3. **Merge into the consolidated document:**
   - If the file doesn't exist yet: Create it with the structure below and the current feature as the first section
   - If the feature already has a section: Replace that section with the updated content from the refined design
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
6. **Write the updated `docs/alinos/<name>_design.md`**

**Content guidelines:**
- **提炼而非复制**: 提取核心设计要素 — 架构决策、数据模型、接口定义、约束条件
- **维护交叉引用**: 功能间有交互时，注明依赖关系
- **保持最新**: 只反映最新决策（探索后的修正），不保留历史版本
- **清除过时内容**: 探索中修正的设计决策，只保留最终结论

## The Bottom Line

**Every unasked question becomes a surprise during implementation.**

Ask now. Answer now. Surprise later = never.
