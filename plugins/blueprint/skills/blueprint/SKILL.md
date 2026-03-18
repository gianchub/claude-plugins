---
name: blueprint
description: >
  This skill should be used when the user asks to "create a blueprint",
  "blueprint this feature", "plan this implementation", "make a plan",
  "create an implementation plan", "design the architecture",
  "break this down into steps", or needs a structured plan with
  build-review-verify cycles. Produces plan artifacts with a 3-phase
  cycle per step.
---

# Blueprint Skill

## Purpose

Produce collaborative implementation plans as written artifacts, where every step follows a build-review-verify cycle. Transform vague feature requests, architectural changes, or refactoring goals into concrete, sequenced plans that a human or agent can execute step by step. Treat planning as a dialogue — explore the codebase, discover tooling, ask questions, assess complexity, then generate the plan.

## Effort Level

Apply maximum effort. Explore the codebase exhaustively before generating any plan. Read existing code, configuration, tests, and CI pipelines to understand the current architecture, conventions, and constraints. Never plan in a vacuum — every step must account for the real state of the project.

## Design Principles

### Collaborative: Ask, Don't Assume

Never guess at requirements, constraints, or preferences. When uncertainty exists, ask. Two to three clarifying questions per turn is fine — avoid overwhelming the user with a wall of questions, but also avoid proceeding with unvalidated assumptions. Common things to ask about:

- Scope boundaries: "Should this cover admin users too, or just regular users for now?"
- Existing patterns: "I see two different patterns for error handling in the codebase — which one to follow?"
- Priority tradeoffs: "This could be done with a simple approach now or a more flexible one that takes longer — which do you prefer?"
- Non-obvious constraints: "Is there a latency budget for this endpoint?"

Continue the clarification cycle until the task is solid enough to plan. State what is understood so far and what remains unclear. Do not proceed to plan generation while critical ambiguities remain.

### Prose Over Code

Plan steps describe intent, behavior, and constraints in prose. Do not include code blocks in plan steps except for the three permitted exceptions:

1. **Interface signatures** — when an exact function, method, or API signature is critical for cross-step compatibility.
2. **Exact config keys** — when a configuration shape must match a specific external contract.
3. **Schema shapes** — when a data schema (database columns, API response shape, message format) is the primary deliverable of the step.

Everything else stays in prose. If a step feels like it needs code to be clear, that signals the step is too large or too implementation-focused. Split it or raise the abstraction level.

### Adaptive Complexity

Not every plan needs the same structure. Apply these heuristics to choose the output format:

- **5 or fewer steps, single concern**: Generate a single plan document. No grouping or milestones needed.
- **6-8 steps, still a single concern**: Generate a single plan document with steps grouped under headings if natural groupings exist.
- **Distinct phases or more than 8 steps**: Generate a milestone folder with one file per milestone, each containing a subset of steps.
- **When in doubt**: Ask the user. Present the tradeoff: "This could be a single document with 7 steps or split into 2 milestones — preference?"

## Workflow

### 1. Understand the Task and Discover Tooling

Begin by reading the codebase broadly. Examine:

- Project structure (directories, modules, packages).
- Entry points (main files, route definitions, CLI commands).
- Data layer (models, schemas, migrations, repositories).
- Configuration (settings files, environment variables, feature flags).
- Existing tests (test structure, fixtures, factories, coverage).
- Documentation (architecture docs, ADRs, READMEs with setup instructions).

**While exploring, discover project tooling.** Tool discovery is not a separate phase — it happens naturally during codebase exploration. As configuration files, CI pipelines, and lock files are encountered, record the tools they imply:

1. **Scan config files**: `pyproject.toml` (`[tool.*]` sections), `package.json` (`scripts`, `devDependencies`), `Cargo.toml`, `go.mod`, `Makefile`/`justfile` targets. See `references/tool-discovery.md` for the full per-language lookup table.
2. **Check CI pipelines**: `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`. Extract the shell commands that validate code quality (test, lint, format, type-check). CI commands take precedence over config file commands when they conflict.
3. **Note lock files**: They confirm the package manager (`uv.lock` → uv, `yarn.lock` → yarn, etc.).
4. **Check for script conventions**: `npm scripts`, `Makefile` targets, `scripts/` directory executables.

Then ask clarifying questions. Focus on:

- What the user actually wants (not what they said — these sometimes differ).
- What constraints exist that are not visible in the code.
- What the definition of done looks like for this work.
- Whether there are related changes planned that this should accommodate.

Iterate on understanding. Summarize what has been gathered so far, identify gaps, and ask follow-up questions. Two to three rounds of clarification is normal for non-trivial plans. For simple, well-defined tasks, one round may suffice.

**Required deliverable before proceeding**: Present the discovered tool chain to the user for confirmation. Format it as a numbered list with the source of each discovery in parentheses. The user may confirm, add, remove, or reorder tools. Do not proceed to step 2 until the tool chain is confirmed. Example:

```
Discovered tools for this project:

1. Package manager: uv (from uv.lock)
2. Test runner: pytest (from pyproject.toml [tool.pytest])
3. Linter: ruff check . (from pyproject.toml [tool.ruff])
4. Formatter: ruff format --check . (from pyproject.toml [tool.ruff.format])
5. Type checker: mypy . (from pyproject.toml [tool.mypy])

Add, remove, or reorder? (or confirm to proceed)
```

### 2. Assess Complexity

With the task understood and tool chain confirmed, determine the plan's scope:

- Count the anticipated steps. Each step should represent one logical unit of work — something that can be built, reviewed, and verified independently.
- Evaluate whether natural milestones exist (e.g., "data layer first, then API, then UI").
- Apply the adaptive complexity heuristics from the Design Principles section to choose single-doc or milestone-folder format.
- If the choice is ambiguous, ask the user.

**Step sizing guidance**:

- A step should take between 15 minutes and 2 hours to execute. Shorter than 15 minutes means it should be merged with an adjacent step. Longer than 2 hours means it should be split.
- Each step must be independently verifiable — all its tests pass without depending on future steps being complete.
- Steps should build on each other sequentially. Later steps may depend on earlier steps, but not the reverse.
- Avoid steps that are purely structural ("set up the directory") unless the project has no existing structure. Structural work should be folded into the first functional step.

### 3. Generate the Plan

Write the plan artifact(s) following the structure defined in `references/step-template.md`. Every step includes all three phases: Build, Adversarial Review, and Verification.

**Plan header**: Include a title, date, summary of the goal, and the confirmed tool chain.

```markdown
# Plan: [Feature/Change Title]

**Date**: YYYY-MM-DD
**Goal**: [1-2 sentence summary of what this plan achieves]

## Tool Chain

| Category | Tool | Command |
|---|---|---|
| Test runner | pytest | `pytest tests/ -x -q` |
| Linter | ruff | `ruff check .` |
| Type checker | mypy | `mypy src/` |
| Formatter | ruff | `ruff format --check .` |

## Steps

[Steps follow here, each using the 3-phase template]
```

**Step generation rules**:

- Number steps sequentially starting from 1.
- Write clear, specific titles that describe the deliverable ("Add user authentication endpoint"), not the activity ("Work on authentication").
- Write acceptance criteria that are concrete and testable. Avoid vague criteria like "code is clean" or "performance is good." Use measurable conditions: "Response time under 200ms for 95th percentile," "All validation errors return 422 with field-level messages."
- In Phase 1 (Build), describe intent per the prose-first approach. Specify what to create, modify, and test. Reference existing code patterns where applicable ("Follow the same repository pattern used in `src/repos/product_repo.py`").
- In Phase 2 (Adversarial Review), write step-specific review questions. Target the most likely failure modes for what was just built. Do not use generic checklists — every review question should be relevant to the specific step.
- In Phase 3 (Verification), include the full checklist with tool commands from the confirmed tool chain. Add step-specific verification items beyond the standard checks.

**Dependency tracking**: If a step depends on artifacts from a previous step, state the dependency explicitly in the objective. Example: "Depends on Step 2 (user repository). Uses the `UserRepository` interface defined there."

**Writing the milestone folder** (when applicable):

- Create one file per milestone: `01_milestone-name.md`, `02_milestone-name.md`, etc.
- Each milestone file follows the same structure (header, tool chain, steps).
- Add a root `README.md` in the plan folder that lists milestones in order with one-sentence descriptions.
- Keep milestones to 3-5 steps each. If a milestone has more, split it.

## Output Formats

### Single Document

Use for plans with 5 or fewer steps, or up to 8 steps with a single concern.

**Path**: `docs/plans/YYYY-MM-DD-<topic>-plan.md`

Example: `docs/plans/2026-03-15-user-auth-plan.md`

### Milestone Folder

Use for plans with distinct phases or more than 8 steps.

**Path**: `docs/plans/YYYY-MM-DD-<topic>/`

Contents:
```
docs/plans/2026-03-15-user-auth/
  README.md
  01_data-layer.md
  02_api-endpoints.md
  03_frontend-integration.md
```

The `README.md` provides an ordered list of milestones with summaries, the confirmed tool chain, and any cross-cutting concerns that apply to all milestones.

## What Belongs in a Plan Step

Include:

- **Prose intent**: What to build and why, described at the right abstraction level.
- **Acceptance criteria**: Concrete, testable conditions for "done."
- **What to test**: Which test cases to write, covering both happy paths and failure modes.
- **What to review**: Step-specific review questions targeting likely failure modes.
- **Verification commands**: Exact tool commands to run, populated from the discovered tool chain.
- **Dependencies**: Which previous steps this one builds on and what artifacts it uses.

Do not include:

- **Code blocks** (except the three permitted exceptions: interface signatures, config keys, schema shapes).
- **Implementation details that go stale**: Algorithm pseudocode, variable names, internal data structure choices. These belong in the code, not in the plan.
- **Generic advice**: "Write clean code," "follow best practices," "handle errors properly." Every instruction must be specific to the step.
- **Premature optimization notes**: Unless performance is an acceptance criterion for the step, defer optimization concerns.

## Handling Plan Execution

When the user asks to execute a plan (or begins working through steps), shift into execution mode. **Preferred**: use the `blueprint:execute` skill (it ships with this plugin) — it provides full subagent-based orchestration with batching, git handling, and progress tracking. Always try to invoke `blueprint:execute` first.

If the execute skill is not available, follow this fallback:

- Work through one step at a time, completing all three phases before moving to the next. Never proceed to the next step until the current step's verification passes.
- After Phase 3 verification passes, summarize what was completed and confirm readiness to proceed to the next step. Include a brief list of files changed and tests added.
- If Phase 2 review or Phase 3 verification reveals issues, fix them within the current step before moving on. Surface blocking issues to the user rather than silently resolving them.
- If execution reveals that a future step needs modification (scope changed, new constraint discovered), note the required adjustment and confirm with the user before modifying the plan.
- For multi-session execution, mark completed steps with a ✅ checkmark in the plan file (e.g., `### Step 1: Auth` → `### ✅ Step 1: Auth`) and tick all markdown checkboxes within the completed step (`- [ ]` → `- [x]`). On resume, find the first unmarked step and continue from there.

## Handling Ambiguity and Scope Changes

- If the user's request is too vague to plan ("make the app better"), ask for specifics. Do not generate a plan from vague input. Push back respectfully — a clear problem statement is a prerequisite for a useful plan.
- If the user changes scope mid-planning, acknowledge the change, assess its impact on the current plan state, and either adjust or restart as appropriate. If the change invalidates more than half the existing plan, recommend starting fresh rather than patching.
- If a step proves unnecessary during execution, skip it explicitly — do not silently omit it. Note why it was skipped and confirm with the user.
- If new steps are needed during execution, propose them with the same 3-phase structure and get user confirmation before adding them to the plan.
- If conflicting requirements surface during planning, flag the conflict immediately. Present the tradeoff to the user with a recommended resolution rather than silently choosing one interpretation.

## Additional Resources

Refer to the following reference files for detailed guidance:

- **`references/step-template.md`** — Full step template with phase-by-phase guidance and a complete example step. Use this as the structural reference for every step in every plan.
- **`references/tool-discovery.md`** — Per-language lookup tables for detecting project tooling across ecosystems. Use as a reference during codebase exploration in step 1.
