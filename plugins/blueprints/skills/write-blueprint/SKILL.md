---
name: write-blueprint
description: >
  This skill should be used when the user asks to "create a blueprint",
  "write a blueprint", "blueprint this feature", "plan this implementation",
  "make a plan", "create an implementation plan", "design the architecture",
  "design this feature", "break this down into steps",
  "plan this refactoring", or "help me plan".
---

# Write Blueprint Skill

## Purpose

Produce collaborative implementation plans as written artifacts, where every step follows a build-review-verify cycle. Transform vague feature requests, architectural changes, or refactoring goals into concrete, sequenced plans that a human or agent can execute step by step. Treat planning as a dialogue — explore the codebase, discover tooling, ask questions, compare approaches, assess complexity, then generate the plan.

## Effort Level

Scale exploration depth to task complexity, but always err toward more thoroughness. Read broadly before narrowing — the goal is a plan that surfaces zero surprises during execution.

## Plan Format

Plans are written as a self-contained file (or folder for milestone plans). The format determines the file extension and the step template used in Step 4. **HTML is the default** because it enables inline SVG dependency graphs for non-linear step dependencies, structured `data-status` attributes that `execute-blueprint` uses for precise progress tracking, and anchor cross-references between steps — affordances that materially improve a multi-step plan and cannot be expressed in Markdown.

Unlike the audit skills, the blueprint format is *not* a hard gate. Detect the user's preference from the launching message and respect it:

- HTML signals: `HTML`, `html`, "as a webpage", "browser-viewable" → use HTML.
- Markdown signals: `MD`, `md`, `Markdown`, `markdown`, "plain text plan", "as Markdown" → use MD.
- No signal: default to HTML silently. The file extension on disk (`.html` vs `.md`) is the canonical signal of which format was chosen — do not add a redundant "Format: HTML" line to the plan header just to restate what the extension already conveys.

If the launching message is genuinely ambiguous about format (e.g., conflicting signals, or the user asks "which format should I use?"), ask once and proceed. When you resolved real ambiguity with a default, briefly document the resolution in the plan header (e.g., "Format chosen: HTML — launching message had conflicting signals, defaulted to HTML"). Otherwise do not interrupt the planning dialogue with a format gate — the planning flow is already heavy with hard gates (tool chain, clarification, approach, plan review).

The chosen format is used consistently for the remainder of the run:

| Format | Step template | File extension |
|---|---|---|
| HTML | `references/step-template-html.md` | `.html` |
| MD | `references/step-template.md` | `.md` |

When writing an HTML plan, treat the HTML-specific affordances documented in `references/step-template-html.md` as tools to use when they add information — not decoration. Wrapping Markdown content in HTML tags without using those affordances defeats the point of choosing HTML.

## Anti-pattern: "Too Simple to Plan"

Even a one-line change carries assumptions about where it goes, what it affects, and how it gets verified. "Simple" tasks are precisely where unexamined assumptions cause wasted rework — the build-review-verify structure catches those before they compound. A plan can be a single step with one acceptance criterion; the fast-path for small plans (≤2 steps) already keeps overhead minimal. The anti-pattern is skipping planning entirely, not the plan's size.

## Design Principles

### Prose Over Code

Plan steps describe intent in prose. Do not include code blocks except for interface signatures, config keys, and schema shapes (full policy in `references/step-template.md`). Tool commands in Phase 3 checklists are operational instructions, not code — they are always permitted.

### Context Discipline (Subagent-Driven)

Planning is deliberately subagent-driven so the main conversation stays a lean coordinator. The two heaviest token consumers — bulk codebase reading during exploration (Step 1) and the adversarial plan review (Step 5) — are pushed into subagents whose context is reclaimed when they return. The main conversation keeps only distilled findings: a structural map, the confirmed tool chain, the resolved requirements, and the review report — not the raw contents of every file surveyed. This mirrors `execute-blueprint`, which keeps every build, review, and verification inside a subagent for the same reason. Keeping the context window lean is what lets both skills scale to large codebases and multi-milestone plans without degrading.

## Workflow

**Flow:** Explore codebase → Confirm tool chain (hard gate) → Clarify requirements (hard gate) → Propose approaches → Assess complexity → Generate plan → Adversarial review → Plan ready.

<PLANNING-GATE>
Two hard gates exist before plan generation:

1. **Tool confirmation gate (after Step 1)**: Present only the tool chain and wait for confirmation — do not combine this with clarifying questions or any other output.
2. **Readiness gate (after Step 1b)**: All critical ambiguities surfaced during clarification must be resolved. An ambiguity is critical if resolving it differently would change the plan's structure, step count, or chosen approach.

Proceeding without both produces plans built on guesswork.
</PLANNING-GATE>

### Fast-path Rules

Plans with ≤2 steps covering a narrowly scoped change (config change, single-file refactoring, straightforward addition) qualify for fast-path treatment:

- **Tool discovery (Step 1):** Present discovered tools inline with the generated plan instead of as a separate confirmation gate. Still include the tool chain table in the plan header.
- **Approach proposal (Step 2):** When only one credible strategy exists, state it briefly and move on — do not invent artificial alternatives.
- **Plan review (Step 5):** Skip the subagent dispatch. Perform a quick self-review checking for gaps in acceptance criteria, missing verification items, and dependency issues. Present the plan directly and ask if the user wants to start execution.

For plans with 3 or more steps, or any plan touching architecture, multiple modules, or cross-cutting concerns, follow the full workflow for all steps — no exceptions.

### 1. Explore Codebase and Discover Tooling

Begin by surveying the codebase broadly. For anything larger than a small project, **dispatch exploration subagents** (via Claude Code's Agent tool) to read areas in parallel and return distilled findings — a map of structure, entry points, data layer, conventions, and discovered tooling — rather than reading every file into the main conversation. The coordinator synthesizes those summaries and keeps only what the planning dialogue needs; this is what keeps the context window lean. Reserve direct reads in the main conversation for the few files you must see verbatim to ask precise clarifying questions.

Across the codebase, examine:

- Project structure (directories, modules, packages).
- Entry points (main files, route definitions, CLI commands).
- Data layer (models, schemas, migrations, repositories).
- Configuration (settings files, environment variables, feature flags).
- Existing tests (test structure, fixtures, factories, coverage).
- Documentation (architecture docs, ADRs, READMEs with setup instructions).

**While exploring, discover project tooling.** Tool discovery is not a separate phase — it happens naturally during codebase exploration. As configuration files, CI pipelines, and lock files are encountered, record the test runner, linter, formatter, type checker, and package manager they imply. See `references/tool-discovery.md` for the full per-language lookup table and detection methodology. When CI pipeline commands conflict with config file commands, prefer the CI commands — they reflect what actually runs.

<TOOL-CONFIRMATION-GATE>
**This step ends with tool confirmation — nothing else.** After exploration, present the discovered tool chain to the user and wait for their confirmation. Do not ask clarifying questions, propose approaches, or do any other work in this response. The tool chain confirmation is a hard gate: the user must confirm, add, remove, or reorder tools before you proceed to clarification.
</TOOL-CONFIRMATION-GATE>

Format the tool chain as a numbered list with the source of each discovery in parentheses. Example:

```
Discovered tools for this project:

1. Package manager: uv (from uv.lock)
2. Test runner: pytest (from pyproject.toml [tool.pytest])
3. Linter: ruff check . (from pyproject.toml [tool.ruff])
4. Formatter: ruff format --check . (from pyproject.toml [tool.ruff.format])
5. Type checker: mypy . (from pyproject.toml [tool.mypy])

Add, remove, or reorder? (or confirm to proceed)
```

**Fast-path:** For ≤2-step plans, present tools inline with the plan instead of as a separate confirmation gate (see Fast-path Rules).

### 1b. Clarify Requirements

Once the user has confirmed the tool chain, ask clarifying questions. Focus on:

- What the user actually wants (not what they said — these sometimes differ).
- What constraints exist that are not visible in the code.
- What the definition of done looks like for this work.
- Whether there are related changes planned that this should accommodate.

Iterate on understanding. Summarize what has been gathered so far, identify gaps, and ask follow-up questions. Two to three rounds of clarification is normal for non-trivial plans. For simple, well-defined tasks, one round may suffice.

When planning involves architectural decisions that benefit from diagrams or visual comparison of approaches, the superpowers plugin's visual companion can render these in a browser. This capability requires the superpowers plugin to be installed; no fallback is provided if it is absent.

### 2. Propose Approaches

Once the task is understood and tooling confirmed, outline 2-3 candidate implementation strategies before locking in a plan structure. For each, describe the approach in a sentence or two and call out its key trade-offs — what it optimizes for, what it sacrifices, and where it carries risk. Open with the strategy you recommend and explain the reasoning; then present the alternatives so the user can make an informed choice. Wait for the user to select an approach before moving to complexity assessment and plan generation.

**Fast-path:** If only one credible strategy exists, state it briefly and move on (see Fast-path Rules).

### 3. Assess Complexity

With the approach selected, determine the plan's scope:

- Count the anticipated steps. Each step should represent a single logical unit of work — larger than a trivial config change, smaller than a full feature. If it can be described in one sentence, merge it with an adjacent step. If it needs its own sub-plan, split it.
- Evaluate whether natural milestones exist (e.g., "data layer first, then API, then UI").
- Choose the output format based on step count:
  - **5 or fewer steps, single concern**: single plan document.
  - **6-8 steps, single concern**: single document with grouped headings.
  - **Distinct phases or more than 8 steps**: milestone folder with one file per milestone.
  - **Ambiguous**: ask the user.

**Step sizing guidance**:
- Each step must be independently verifiable — all its tests pass without depending on future steps being complete.
- Steps should build on each other sequentially. Later steps may depend on earlier steps, but not the reverse.
- Avoid steps that are purely structural ("set up the directory") unless the project has no existing structure. Structural work should be folded into the first functional step.

### 4. Generate the Plan

Write the plan artifact(s) following the template that matches the format chosen in the Plan Format section above: `references/step-template-html.md` for HTML output (default), or `references/step-template.md` for Markdown. Every step includes all three phases: Build, Adversarial Review, and Verification.

**Plan header**: Include a title, date, summary of the goal, and the confirmed tool chain. The exact rendering of the header lives in the chosen template's "Plan Document Structure" section. For Markdown, the header looks like:

```markdown
# Plan: [Feature/Change Title]

**Date**: YYYY-MM-DD
**Goal**: [1-2 sentence summary of what this plan achieves]

## Tool Chain

| Category | Tool | Command |
|---|---|---|
| Test runner | [discovered] | `[test command]` |
| Linter | [discovered] | `[lint command]` |
| Type checker | [discovered] | `[type-check command]` |
| Formatter | [discovered] | `[format command]` |

## Steps

[Steps follow here, each using the 3-phase template]
```

For HTML, the header uses the `<header class="plan-header">` and `<section id="tool-chain">` scaffold defined in `references/step-template-html.md`. The required IDs, classes, and `data-*` attributes on `<article class="step">` are not optional for HTML plans — `execute-blueprint` mutates those exact attributes to track progress.

**Step generation rules**:

- Number steps sequentially starting from 1.
- Write clear, specific titles that describe the deliverable ("Add user authentication endpoint"), not the activity ("Work on authentication").
- Write acceptance criteria that are concrete and testable. Avoid vague criteria like "code is clean" or "performance is good." Use measurable conditions: "Response time under 200ms for 95th percentile," "All validation errors return 422 with field-level messages."
- In Phase 1 (Build), describe intent per the prose-first approach. Specify what to create, modify, and test. Reference existing code patterns where applicable ("Follow the same repository pattern used in `src/repos/product_repo.py`").
- In Phase 2 (Adversarial Review), write step-specific review questions targeting the most likely failure modes, but also include broader integration questions: Does this change fit naturally in the existing codebase? Does it follow established conventions and patterns? Could it break or degrade anything outside its immediate scope? The review is a thorough, critical code review of the work done — not just an acceptance criteria checklist. The goal is to eliminate all issues introduced by the build phase before proceeding.
- In Phase 3 (Verification), include the full checklist with tool commands from the confirmed tool chain. Add step-specific verification items beyond the standard checks.
- If a step genuinely needs a person to look before execution can move on — a rendered UI, a migration against real data, an approval no tool can establish — say so explicitly in the step text. `execute-blueprint` fixes ordinary findings by itself and pauses only where the plan asks for a human, so an unstated checkpoint will not happen.

**Dependency tracking**: If a step depends on artifacts from a previous step, state the dependency explicitly in the objective. Example: "Depends on Step 2 (user repository). Uses the `UserRepository` interface defined there."

**Writing the milestone folder** (when applicable):

- Create one file per milestone using the chosen format's extension: `01_milestone-name.html`, `02_milestone-name.html`, etc. for HTML; `.md` for Markdown.
- Each milestone file follows the same structure (header, tool chain, steps) from the chosen template.
- Add a root `README` in the plan folder (`README.html` for HTML plans, `README.md` for Markdown plans) that lists milestones in order with one-sentence descriptions. The HTML variant uses the `<section id="milestones">` scaffold from `references/step-template-html.md`.
- Keep milestones to 3-5 steps each. If a milestone has more, split it.
- Do not mix formats within a single plan folder. All milestone files and the README share one extension.

### 5. Adversarial Plan Review

After writing the plan to disk, dispatch a subagent to perform an adversarial review of the entire plan. The subagent reads the plan fresh from disk with no anchoring to the planning context — it acts as a critical second pair of eyes whose sole purpose is to find weaknesses before execution begins.

**Why a subagent**: The agent that wrote the plan is anchored to its own reasoning. A fresh subagent without the full planning conversation history reads the plan as an executor would — spotting ambiguities, gaps, and logical flaws that the author is blind to. The subagent receives only a brief scope summary (see `{{PLANNING_CONTEXT}}` below) to verify the plan addresses the user's full intent, not the entire planning dialogue.

**Fast-path:** Plans with ≤2 steps skip the subagent — perform a quick self-review instead (see Fast-path Rules).

**Dispatching the review subagent**: Use Claude Code's Agent tool to dispatch the plan review subagent. Reference `references/plan-review-subagent.md` for the exact prompt template. Substitute placeholders before dispatching:

- `{{PLAN_PATH}}` — absolute path to the plan file or milestone folder.
- `{{PROJECT_ROOT}}` — absolute path to the project root.
- `{{PLANNING_CONTEXT}}` — compose a brief summary (5-10 sentences) of: what the user originally asked for, key constraints and decisions from the clarification rounds, agreed scope boundaries, and any explicit exclusions ("we agreed not to handle X"). This gives the subagent enough context to verify the plan addresses the user's full intent, not just what the Goal header captured.

The subagent prompt in `references/plan-review-subagent.md` contains the full review methodology covering completeness, dependencies, sizing, criteria quality, phase quality, prose compliance, architecture, and risk.

**After the subagent returns**:

- If the review finds **no issues**: Inform the user the plan passed adversarial review and ask if they want to start execution.
- If the review finds **issues**: Present all findings to the user with the subagent's full report. The user decides whether changes are needed or the plan is acceptable as-is.
  - If the user wants changes: make the requested modifications to the plan, then offer to re-run the adversarial review on the updated plan. The user may accept another review round or decline and proceed to execution. Repeat this review-modify cycle until the user is satisfied.
  - If the user says the plan is fine: proceed to ask if they want to start execution.

Do not skip the plan review (except via the fast-path above). Do not auto-resolve findings without user input. The plan review is a hard gate — the plan is not considered complete until it has passed this step.

**After approval**: **Commit the plan file to git** so it persists across sessions and supports checkmark-based progress tracking. Do not skip this step — without the commit, cross-session resumability breaks.

## Output Formats

See Step 3 (Assess Complexity) for which document layout to choose based on step count. See "Plan Format" near the top of this skill for the HTML vs Markdown decision. The two choices are orthogonal: any document layout can be HTML or Markdown.

### Single Document

**Path**: `docs/plans/YYYY-MM-DD-<topic>-plan.<ext>` where `<ext>` is `html` (default) or `md`.

Example (HTML): `docs/plans/2026-03-15-user-auth-plan.html`
Example (MD): `docs/plans/2026-03-15-user-auth-plan.md`

### Milestone Folder

**Path**: `docs/plans/YYYY-MM-DD-<topic>/`

Contents (HTML, the default):
```
docs/plans/2026-03-15-user-auth/
  README.html
  01_data-layer.html
  02_api-endpoints.html
  03_frontend-integration.html
```

Contents (Markdown):
```
docs/plans/2026-03-15-user-auth/
  README.md
  01_data-layer.md
  02_api-endpoints.md
  03_frontend-integration.md
```

The `README` (`.html` or `.md` matching the plan's format) provides an ordered list of milestones with summaries, the confirmed tool chain, and any cross-cutting concerns that apply to all milestones.

## Handling Plan Execution

When the user asks to execute, invoke `blueprints:execute-blueprint` — it provides full subagent orchestration with git handling and progress tracking, and handles both `.md` and `.html` plan formats. If unavailable, work through steps one at a time completing all three phases before advancing. Mark completed steps with a checkmark in the plan heading and tick Phase 3 checkboxes (or, for HTML plans, set `data-status="complete"` on the step's `<article>`, update the status badge text, prepend `✅ ` to the step heading, and add `checked` to verification checkboxes) for cross-session resumability. See `references/step-template.md` or `references/step-template-html.md` for what belongs in each phase of a plan step.

## Handling Scope Changes

- If scope changes during execution invalidate more than half the remaining plan, recommend starting fresh rather than patching a plan built on outdated assumptions.
- If new steps are needed during execution, propose them with the same 3-phase structure (build, adversarial review, verification) and insert them at the appropriate position in the plan.

## Additional Resources

Refer to the following reference files for detailed guidance:

- **`references/step-template.md`** — Markdown step template with phase-by-phase guidance and a complete example step. Use as the structural reference when the chosen plan format is Markdown.
- **`references/step-template-html.md`** — HTML step template with the required scaffold, structural contract (IDs/classes/`data-*` attributes), HTML-specific affordances guidance, and a complete example step. Use as the structural reference when the chosen plan format is HTML (default).
- **`references/tool-discovery.md`** — Per-language lookup tables for detecting project tooling across ecosystems. Use as a reference during codebase exploration in step 1.
- **`references/plan-review-subagent.md`** — Prompt template for the adversarial plan review subagent dispatched in step 5. Use this verbatim when dispatching the review subagent after plan generation.
