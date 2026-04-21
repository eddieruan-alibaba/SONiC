---
name: alinos-spec-propose
description: "[project] Propose a new change by generating initial proposal and design artifacts via openspec. Use when the user wants to describe what they want to build and get a proposal framework as the starting point."
license: MIT
compatibility: Requires openspec CLI.
model: ultimate
metadata:
  author: alinos-spec
  version: "2.0"
---

Propose a new change — generate the initial proposal and design framework via openspec.

This skill handles the first step of a three-stage process:

1. **`/alinos.propose` (this skill)** — Generate proposal.md + design.md
2. **`/alinos.explore`** — Socratic self-examination to refine the design
3. **`/alinos.brainstorming`** -> **`/alinos.writing-plans`** — Implementation details and development plan

---

**Input**: The user's request should include a change name (kebab-case) OR a description of what they want to build.

---

**Step 0 (project rules)**: Read AGENTS.md for project-specific rules bound to this skill. If rules exist, execute them BEFORE proceeding with the standard steps below. Project rules have HIGHER priority than default steps — they may add pre-steps, modify the workflow, or require additional output sections.

## Steps

1. **If no clear input provided, ask what they want to build**

   Use the **AskUserQuestion tool** (open-ended, no preset options) to ask:
   > "What change do you want to work on? Describe what you want to build or fix."

   From their description, derive a kebab-case name (e.g., "add user authentication" -> `add-user-auth`).

   **IMPORTANT**: Do NOT proceed without understanding what the user wants to build.

2. **Create the change (openspec scaffolding only)**
   ```bash
   openspec new change "<name>"
   ```
   This creates openspec metadata at `openspec/changes/<name>/`. Openspec is used only for scaffolding and status tracking — **all artifact files are written to `docs/alinos/`**, not to the openspec directory.

3. **Get the artifact build order**
   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to get:
   - `applyRequires`: array of artifact IDs needed before implementation (e.g., `["tasks"]`)
   - `artifacts`: list of all artifacts with their status and dependencies

4. **Create proposal and design artifacts only**

   <IMPORTANT>
   **Output path rule**: ALL artifact files MUST be written under `docs/alinos/`. IGNORE the `outputPath` returned by `openspec instructions` — that points to the openspec internal directory which is NOT the correct location.
   </IMPORTANT>

   Artifact output paths:
   - Proposal: `docs/alinos/proposals/YYYY-MM-DD-<name>-proposal.md`
   - Design: `docs/alinos/specs/YYYY-MM-DD-<name>-design.md`

   Use the **TodoWrite tool** to track progress through the artifacts.

   Loop through artifacts in dependency order, creating **proposal.md** and **design.md** only (stop before tasks):

   a. **For each artifact that is `ready` (dependencies satisfied)**:
      - Get instructions:
        ```bash
        openspec instructions <artifact-id> --change "<name>" --json
        ```
      - From the instructions JSON, use ONLY these fields:
        - `context`: Project background (constraints for you - do NOT include in output)
        - `rules`: Artifact-specific rules (constraints for you - do NOT include in output)
        - `template`: The structure to use for your output file
        - `instruction`: Schema-specific guidance for this artifact type
        - `dependencies`: Completed artifacts to read for context
      - **IGNORE `outputPath`** — write to the `docs/alinos/` paths listed above
      - Read any completed dependency files for context
      - Create the artifact file using `template` as the structure
      - Apply `context` and `rules` as constraints - but do NOT copy them into the file
      - Show brief progress: "Created <artifact-id> at docs/alinos/..."

   b. **Stop after design.md is created** — do NOT generate tasks.md yet

   c. **If an artifact requires user input** (unclear context):
      - Use **AskUserQuestion tool** to clarify
      - Then continue with creation

5. **Update Consolidated Design Document**

   After creating the feature design spec, update the project-level consolidated design document at `docs/alinos/<name>_design.md` (where `<name>` is the same kebab-case change name used throughout this skill). This document represents the **current state of the entire project's design** — a single source of truth that integrates all feature designs.

   <IMPORTANT>
   **命名规范**: 文件名必须包含项目名，格式为 `docs/alinos/<name>_design.md`，例如 `docs/alinos/fuzzy-test_design.md`。
   **语言规范**: 该文件必须全中文撰写，包括标题、正文、变更记录。指令中的英文仅用于描述流程，产出物内容一律使用中文。
   </IMPORTANT>

   **Update process:**
   a. **Read existing `docs/alinos/<name>_design.md`** (if it exists)
   b. **Read the feature design spec** just created (`docs/alinos/specs/YYYY-MM-DD-<name>-design.md`)
   c. **Merge into the consolidated document:**
      - If the file doesn't exist yet: Create it with the structure below and the current feature as the first section
      - If the feature already has a section: Replace that section with the updated content
      - If the feature is new: Add a new section in the appropriate location
   d. **Document structure (Chinese):**
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
   e. **Organization rules:**
      - Each feature section contains the **distilled design** — key decisions, architecture, interfaces, constraints — not a full copy of the spec
      - Order sections logically by component/module relationship, not by creation date
   f. **Write the updated `docs/alinos/<name>_design.md`**

   **Content guidelines:**
   - **提炼而非复制**: 提取核心设计要素 — 架构决策、数据模型、接口定义、约束条件
   - **维护交叉引用**: 功能间有交互时，注明依赖关系
   - **保持最新**: 只反映最新决策，不保留历史版本
   - **清除过时内容**: 设计决策被替代时，直接更新，不保留旧内容

6. **Show status and guide next step**
   ```bash
   openspec status --change "<name>"
   ```

7. **Clean up openspec scaffolding directory**
   ```bash
   rm -rf openspec/
   ```
   The `openspec/` directory is only needed during artifact generation. All artifacts are already written to `docs/alinos/`, so the scaffolding directory should be removed to keep the project clean.

   Present to user:
   > "Initial proposal and design framework generated."
   >
   > **Created artifacts:**
   > - proposal.md — What & why
   > - design.md — Initial design framework
   > - docs/alinos/<name>_design.md — Consolidated design updated
   >
   > **Recommended next step:** Run `/alinos.explore` to stress-test the design through Socratic examination before planning implementation.
   >
   > **Full workflow:**
   > 1. `/alinos.explore` — Examine and refine the design (resolve all design ambiguities)
   > 2. `/alinos.brainstorming` — Discuss implementation details
   > 3. `/alinos.writing-plans` — Generate development plan

---

## Artifact Creation Guidelines

- Follow the `instruction` field from `openspec instructions` for each artifact type
- The schema defines what each artifact should contain - follow it
- Read dependency artifacts for context before creating new ones
- Use `template` as the structure for your output file - fill in its sections
- **IMPORTANT**: `context` and `rules` are constraints for YOU, not content for the file
  - Do NOT copy `<context>`, `<rules>`, `<project_context>` blocks into the artifact
  - These guide what you write, but should never appear in the output

## Guardrails

- **All artifacts write to `docs/alinos/`** — NEVER write to `openspec/changes/` or any openspec internal path
- Only generate proposal.md and design.md — do NOT generate tasks.md or invoke other skills
- Always read dependency artifacts before creating a new one
- If context is critically unclear, ask the user — but prefer making reasonable decisions to keep momentum
- If a change with that name already exists, ask if user wants to continue it or create a new one
- Verify each artifact file exists after writing before proceeding to next
