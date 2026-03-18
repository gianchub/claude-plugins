# Skill Audit Improvements

**Date**: 2026-03-18
**Scope**: All 3 skills across 2 plugins (blueprint:blueprint, blueprint:execute, code-audit:audit)

---

## 1. Remove workflow summaries from all 3 descriptions (CSO fix)

**Why**: Descriptions that summarize workflow cause Claude to follow the description as a shortcut instead of reading the full skill body. All three skills end their descriptions with a workflow summary that should be removed.

**Files to edit**:
- `plugins/blueprint/skills/blueprint/SKILL.md` — remove "Produces plan artifacts with a 3-phase cycle per step."
- `plugins/blueprint/skills/execute/SKILL.md` — remove "Orchestrates plan execution using subagents for each phase."
- `plugins/code-audit/skills/audit/SKILL.md` — remove "Produces a structured severity-ranked report file."

The trigger phrases in each description are already strong and sufficient for discovery.

---

## 2. Replace "Apply maximum effort" with contextual guidance that pushes for exceptional quality

**Why**: "Apply maximum effort" is vague and unscalable. Replace with guidance that scales to the task but still pushes the agent to go well beyond the minimum — the goal is spectacularly good plans and audits, not just adequate ones.

**Files to edit**:
- `plugins/blueprint/skills/blueprint/SKILL.md` — Effort Level section
- `plugins/blueprint/skills/execute/SKILL.md` — Effort Level section
- `plugins/code-audit/skills/audit/SKILL.md` — Effort Level section

**Replacement guidance per skill**:

### Blueprint
Replace the current Effort Level section with guidance that says: Scale exploration depth to the complexity of the task, but always err on the side of more thoroughness, not less. Even for a seemingly simple change, investigate its context — what it touches, what depends on it, what patterns surround it. A small refactoring doesn't need full-project archaeology, but it does need enough context to produce a plan that accounts for ripple effects. The goal is a plan so thorough that execution surfaces zero surprises. Read broadly before narrowing. Understand the surrounding architecture before planning a change to one piece of it. When in doubt, explore more rather than less.

### Execute
Replace the current Effort Level section with guidance that says: Treat each phase as if it were the only chance to get it right. Builds should be complete, well-tested implementations — not drafts to be cleaned up later. Reviews should be genuinely adversarial — actively searching for flaws, not confirming success. Verifications should run every tool and inspect every result. Never rubber-stamp a phase. The standard is: after all three phases pass, the step's code should be production-ready with no known issues.

### Audit
Replace the current Effort Level section with guidance that says: Read every line of in-scope code. Do not skim, sample, or rely on heuristics to skip files. Trace data flows from external inputs through processing layers to outputs and storage. Follow call chains across module boundaries to detect issues that only manifest through component interaction. When the scope is large enough that thoroughness would suffer in a single pass, split work across parallel subagents by module or directory, then merge and deduplicate findings. The goal is zero missed findings within the confirmed categories — the report should be comprehensive enough that a second audit would find nothing new.

---

## 3. No AI attribution in commits — confirm no conflict

**Status**: User has updated `~/.claude/CLAUDE.md` to remove the conflict. The execute skill's rule (no co-authored-by lines, no bot signatures, no AI references) is the intended behavior. No change needed to the skill. If a conflict is still detected at implementation time, the skill's rule wins: no attribution to Claude Code in commits.

**Files to edit**: None (verify only).

---

## 4. Add fast-path for tool discovery on narrowly-scoped plans

**Why**: The current workflow requires discovering, presenting, and confirming the tool chain before generating any plan — even for trivial 2-step plans. This adds friction without value for small tasks.

**File to edit**: `plugins/blueprint/skills/blueprint/SKILL.md` — Workflow section, Step 1

**Change**: Add a fast-path rule: If the user's request is narrowly scoped (e.g., a config change, single-file refactoring, or other small task), present the discovered tools inline with the plan rather than as a separate confirmation gate. Still discover tools, but fold the confirmation into plan presentation rather than blocking on it. For multi-step or architecturally significant plans, keep the current separate confirmation step.

---

## 5. Clarify that tool commands in Phase 3 are not prose-over-code violations

**Why**: The prose-over-code principle says "do not include code blocks in plan steps except for the three permitted exceptions." But Phase 3 verification checklists contain tool commands like `pytest tests/ -x -q` which are technically code. This needs a carve-out.

**File to edit**: `plugins/blueprint/skills/blueprint/SKILL.md` — Design Principles, "Prose Over Code" section

**Change**: Add a fourth permitted exception or a clarifying note: Tool commands in Phase 3 verification checklists (e.g., `pytest`, `ruff check .`) are not violations of prose-over-code — they are operational instructions, not implementation details.

---

## 6. Give the build subagent the full step context (objective + acceptance criteria + Phase 1)

**Why**: The current build subagent prompt only receives Phase 1 instructions. But acceptance criteria and the objective sit above Phase 1 in the step template. The builder should know what "done" looks like, not just what to build.

**File to edit**: `plugins/blueprint/skills/execute/references/subagent-prompts.md` — Build Subagent section

**Changes**:
- Add a `{{STEP_OBJECTIVE}}` placeholder for the step's objective text.
- Add a `{{ACCEPTANCE_CRITERIA}}` placeholder for the step's acceptance criteria list.
- Update the prompt template to include these before the Phase 1 instructions, so the builder has full context.
- Update the PROCESS section to include: "Before starting implementation, read the acceptance criteria. Every criterion must be met by the end of this build."

Also update `plugins/blueprint/skills/execute/SKILL.md` — Subagent Dispatch section to reflect the new placeholders.

---

## 7. Git mode asked every session — no change needed

**Status**: User confirms git mode must be asked every session. The current behavior ("Default to asking every time — never assume a mode from a previous session") is correct. No change needed.

**Files to edit**: None.

---

## 8. Audit: Smart category selection with user confirmation

**Why**: If the user says "audit the auth module for security issues," presenting all 7 categories for confirmation is unnecessary friction. But the skill should still confirm its choice.

**File to edit**: `plugins/code-audit/skills/audit/SKILL.md` — Step 2 (Confirm Categories)

**Change**: Replace the current "present all 7 and let the user remove/add" with: Analyze the user's request and the codebase to select the most relevant categories. Consider both what the user explicitly asked for and what the code naturally warrants (e.g., async code warrants the concurrency category even if the user didn't mention it). Present the selected categories to the user with a brief rationale for each inclusion, and ask if they want to add or remove any before proceeding. For broad requests ("audit this codebase"), default to all categories.

---

## 9. Add test quality to audit categories

**Why**: The current categories focus on application code. Test code can have its own significant issues — excessive mocking that hides real bugs, tests that don't actually assert anything meaningful, missing edge case coverage, test pollution, etc.

**File to edit**: `plugins/code-audit/skills/audit/references/categories.md`

**Change**: Add a new category "8. Test Quality" with checklist items including:
- Excessive mocking — tests that mock so much they're testing the mocks, not the code. Tests where the real dependencies could be used but mocks are used for convenience.
- Vacuous assertions — tests that assert on truthy values, check that "no error occurred" instead of verifying actual output, or use overly broad matchers.
- Missing failure path tests — test suites that only cover happy paths without testing error conditions, edge cases, or boundary values.
- Test pollution — shared mutable state between tests, test order dependencies, missing teardown/cleanup that causes flaky runs.
- Snapshot overuse — snapshot tests used as a substitute for behavioral assertions, making it easy to blindly update snapshots when behavior changes.
- Duplicated test setup — repeated setup code across test files that should be extracted into fixtures or factories.
- Tests that test implementation, not behavior — tests coupled to internal implementation details (private methods, internal state) rather than observable behavior, making refactoring unnecessarily difficult.
- Dead or skipped tests — tests marked as skip/pending/disabled that have been left in that state indefinitely.
- Missing integration tests — unit tests exist but no tests verify that components work together correctly.

Also update `plugins/code-audit/skills/audit/SKILL.md` — Step 2 to list 8 categories instead of 7.

---

## 10. Reframe step sizing without time estimates

**Why**: The current guidance uses "15 minutes to 2 hours" which introduces time estimates. Reframe as complexity/granularity.

**File to edit**: `plugins/blueprint/skills/blueprint/SKILL.md` — Step sizing guidance in "Assess Complexity"

**Change**: Replace the time-based sizing with: "A step should represent a single logical unit of work — larger than a trivial config change, smaller than a full feature. If it can be described in one sentence, merge it with an adjacent step. If it needs its own sub-plan, split it. Each step must be independently verifiable — all its tests pass without depending on future steps being complete."

Keep the other sizing rules (sequential dependencies, avoid purely structural steps).

---

## 11. (Reference files are solid — no changes needed)

---

## 12. Strengthen adversarial review to be a thorough, context-aware code review

**Why**: The adversarial review should not just check that the build phase met its acceptance criteria. It should be a proper, thorough, critical code review that also evaluates whether the built code fits well within the existing application. A perfectly built component that doesn't integrate well with the surrounding codebase is still a problem.

**Files to edit**:
- `plugins/blueprint/skills/blueprint/references/step-template.md` — Phase 2 guidance
- `plugins/blueprint/skills/blueprint/SKILL.md` — step generation rules for Phase 2
- `plugins/blueprint/skills/execute/references/subagent-prompts.md` — Review Subagent prompt
- `plugins/blueprint/skills/execute/SKILL.md` — Design Principles, "Adversarial review" section

**Changes**:

### step-template.md — Phase 2 guidance
Expand the guidance to make the review broader than just the step's acceptance criteria. Add:
- **Codebase integration**: Does the new code follow the conventions, patterns, and idioms already established in the codebase? Does it integrate naturally with surrounding modules, or does it introduce a different style, structure, or approach that creates inconsistency?
- **Architectural fit**: Does the implementation sit at the right abstraction layer? Does it respect existing boundaries (service layers, repository patterns, API contracts)? Would the change surprise a developer familiar with the codebase?
- **Ripple effects**: Could this change break or degrade anything outside its immediate scope? Check imports, shared utilities, configuration, and any module that depends on modified interfaces.
- **Goal**: The review should aim to eliminate all issues introduced by the build phase before proceeding. Think of it as the last gate — anything that passes review should be genuinely production-ready, not just "meets acceptance criteria."

### SKILL.md (blueprint) — step generation rules
Update the Phase 2 guidance in "Step generation rules" to say: "In Phase 2 (Adversarial Review), write step-specific review questions targeting the most likely failure modes, but also include broader integration questions: Does this change fit naturally in the existing codebase? Does it follow established patterns? Could it break anything outside its immediate scope? The review is a full critical code review, not just an acceptance criteria checklist."

### subagent-prompts.md — Review Subagent prompt
Expand the PROCESS section to include:
- Step 5: Evaluate codebase integration. Read surrounding files beyond the changed files to assess whether the new code follows existing conventions, patterns, and idioms. Flag inconsistencies in style, structure, naming, error handling approach, or architectural patterns compared to the rest of the codebase.
- Step 6: Assess ripple effects. Check whether the changes could break or degrade anything outside the step's immediate scope — shared utilities, dependent modules, configuration, imports.
- Update the classification rules: blocking findings should include "integration issues that would require rework if discovered later" and "pattern violations that create inconsistency with the established codebase."

### SKILL.md (execute) — Design Principles
Update the "Adversarial review" principle to: "The review phase is a thorough, critical code review — not just a check that acceptance criteria were met. The review subagent actively tries to find flaws in correctness, security, test quality, and codebase integration. It reads files beyond the immediate changes to assess whether the new code fits naturally within the existing application. A change that meets its acceptance criteria but clashes with established patterns, breaks surrounding code, or introduces inconsistency is a blocking finding. The goal is to eliminate all issues introduced by the build phase before proceeding."
