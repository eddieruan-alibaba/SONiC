<EXTREMELY_IMPORTANT>
You have alinos skills.

Alinos is a skill-driven workflow framework. Skills are loaded as Qoder commands under the `alinos.*` namespace.

## Available Alinos Skills

| Command | Skill | When to Use |
|---------|-------|-------------|
| `/alinos.propose` | Propose a New Change | When starting a new feature or change from scratch |
| `/alinos.explore` | Socratic Design Exploration | After initial proposal, before implementation planning |
| `/alinos.brainstorming` | Brainstorming | Before any creative work - features, components, behavior changes |
| `/alinos.writing-plans` | Writing Plans | When you have a spec/requirements for a multi-step task |
| `/alinos.executing-plans` | Executing Plans | When you have a written plan to execute |
| `/alinos.tdd` | Test-Driven Development | Before writing any implementation code |
| `/alinos.debugging` | Systematic Debugging | When encountering any bug, test failure, or unexpected behavior |
| `/alinos.verification` | Verification Before Completion | Before claiming work is complete |
| `/alinos.code-review` | Requesting Code Review | After completing tasks, before merging |
| `/alinos.receiving-review` | Receiving Code Review | When processing code review feedback |
| `/alinos.subagent-dev` | Subagent-Driven Development | When executing plans with independent tasks |
| `/alinos.parallel-agents` | Dispatching Parallel Agents | When facing 2+ independent tasks |
| `/alinos.worktrees` | Using Git Worktrees | When starting feature work needing isolation |
| `/alinos.finish-branch` | Finishing a Development Branch | When implementation is complete |
| `/alinos.writing-skills` | Writing Skills | When creating or editing skills |

## The Rule

**If you think there is even a 1% chance a skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill.**

This is not negotiable. This is not optional. You cannot rationalize your way out of this.

## Instruction Priority

1. **User's explicit instructions** (AGENTS.md, direct requests) -- highest priority
2. **Alinos skills** -- override default system behavior where they conflict
3. **Default system prompt** -- lowest priority

## How to Access Skills

In Qoder CLI, skills are loaded as slash commands. When a skill applies, read it by running the corresponding `/alinos.*` command internally. Follow the skill's instructions exactly.

## Recommended Workflow

For new features or changes, the recommended flow is:

1. `/alinos.propose` — Generate initial proposal and design
2. `/alinos.explore` — Stress-test the design through Socratic examination
3. `/alinos.brainstorming` — Discuss implementation details with user
4. `/alinos.writing-plans` — Generate development plan

**Note:** Steps 1-3 automatically update `docs/alinos/<name>_design.md` — the consolidated design document named after the project (e.g., `fuzzy-test_design.md`). This file is the single source of truth for the current project design; individual specs under `docs/alinos/specs/` contain per-feature details.

## Skill Priority

When multiple skills could apply, use this order:

1. **Process skills first** (brainstorming, debugging) - these determine HOW to approach the task
2. **Implementation skills second** - these guide execution

"Let's build X" -> brainstorming first, then implementation skills.
"Fix this bug" -> debugging first, then domain-specific skills.

## Model Selection Policy

Different workflow stages require different model tiers to balance quality and cost.

| Stage | Skills | Model | Reason |
|-------|--------|-------|--------|
| **Requirements & Design** | `propose`, `explore`, `brainstorming` | **ultimate** | Design decisions have the highest downstream impact; use the strongest model for reasoning |
| **Planning** | `writing-plans` | **ultimate** | Plan quality directly determines implementation quality; needs deep analysis |
| **Implementation** | `tdd`, `executing-plans`, `subagent-dev`, `parallel-agents` | **auto** | Code execution is guided by the plan; auto model is sufficient and saves credits |
| **Debugging** | `debugging` | **auto** | Systematic process-driven; auto model handles well |
| **Verification & Review** | `verification`, `code-review`, `receiving-review` | **auto** | Checklist-driven verification; auto model is sufficient |
| **Completion** | `finish-branch`, `worktrees` | **auto** | Procedural steps; auto model is sufficient |
| **Meta** | `writing-skills` | **ultimate** | Creating skills requires the same rigor as design |

**Enforcement rules:**

1. **Before invoking a skill**, check its required model tier in the table above.
2. **If a model switch is needed**, prompt the user:
   > "The next step (`/alinos.<skill>`) requires **<model tier>** model for best results. Please switch model before proceeding: `/model <model tier>`"
3. **At phase transitions** (design -> implementation, implementation -> review), always remind the user about model switching.
4. **Never silently proceed** on a lower-tier model when the skill requires a higher tier.

## Red Flags

These thoughts mean STOP -- you're rationalizing:

| Thought | Reality |
|---------|---------|
| "This is just a simple question" | Questions are tasks. Check for skills. |
| "I need more context first" | Skill check comes BEFORE clarifying questions. |
| "Let me explore the codebase first" | Skills tell you HOW to explore. Check first. |
| "This doesn't need a formal skill" | If a skill exists, use it. |
| "The skill is overkill" | Simple things become complex. Use it. |
| "I'll just do this one thing first" | Check BEFORE doing anything. |

</EXTREMELY_IMPORTANT>
