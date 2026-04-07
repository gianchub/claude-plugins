---
name: execute
description: >
  This skill should be used when the user asks to "execute this blueprint",
  "run the plan", "execute the plan", "start building from the plan",
  "implement the blueprint", "implement the plan", "continue the plan",
  "resume execution", or "execute 01_milestone_name.md".
---

# Execute

## Purpose

Execute blueprint plans by driving each step through a strict build, review, and verify cycle using dedicated subagents. Keep the main conversation lean and focused on orchestration and user decisions while subagents handle the heavy implementation, review, and verification work. Ensure every step meets its acceptance criteria before marking it complete and moving on.

## Effort Level

Treat each phase as the only chance to get it right. Builds should be complete implementations, not drafts. Reviews should be genuinely adversarial. Verifications must run every tool and inspect every result. After all three phases pass, the step's code should be production-ready.

## Workflow

### Locate the Plan

Scan `docs/plans/` for plan files. If a single plan file exists, use it. If multiple plan files or milestone folders exist, present the list to the user and ask which to execute. For milestone folders containing numbered step files, start from the first step that is not marked with a checkmark. If the plan file is specified directly in the user's request (e.g., "execute 01_milestone_name.md"), use that file without asking.

Read the entire plan file into context before beginning execution. Identify all step headings, their phase instructions (Phase 1 build, Phase 2 review, Phase 3 verification), and any dependencies between steps. Count the total number of steps and report to the user how many steps the plan contains and how many are already marked complete before asking to proceed.

If the user requests executing only specific steps (e.g., "execute steps 3-5" or "skip step 2"), honor the request — execute only the specified steps in order, skipping others. Warn if skipped steps contain dependencies required by the requested steps.

### Git Handling

Before executing the first step, ask the user which git mode to use. Present exactly two options:

**User-managed mode:**
- Pause after each individual step completes all three phases (build, review, verify).
- The user inspects changes, stages files, and commits at their discretion.
- Do not create any commits automatically.
- Do not resume execution until the user explicitly says to continue.
- Even when steps are batched, pause after each individual step within the batch. The batch boundary serves as a cumulative progress checkpoint — present a batch summary after the final step in the batch, but the user has already had the opportunity to review after each step.

**Skill-managed mode:**
- After each individual step passes all three phases, create a commit automatically.
- Include the plan file's progress updates (✅ heading prefix and ticked checkboxes) in the same commit as the step's implementation changes — do not create a separate commit for plan progress.
- Use a clear, descriptive commit message in conventional commits format referencing the step.
- Never include Claude Code or AI attribution in commit messages. No co-authored-by lines, no bot signatures, no AI references of any kind.
- Pause cadence depends on plan complexity:
  - **Simple plan (single document):** Execute all steps continuously without pausing. Present a final summary when the plan is complete.
  - **Complex plan (multiple milestone documents):** After the user selects skill-managed mode, ask which pause cadence to use:
    - **Milestone pauses** (default): Pause after each milestone document completes. Present a milestone summary and ask whether to continue to the next milestone. Within a milestone, execute all steps continuously without pausing at batch boundaries.
    - **Full auto**: Execute the entire plan without pausing, even across milestone boundaries. Present a final summary when the plan is complete.

Store the user's choices (git mode and, for skill-managed complex plans, pause cadence) and apply them consistently for the remainder of the execution session. If the user wants to switch modes mid-session, they must say so explicitly. Default to asking every time — never assume a mode or pause cadence from a previous session.

### Step Execution

Group steps into batches of up to 3 steps maximum. Execute steps **strictly serially** within a batch: Step A must complete its full build, review, and verify cycle and pass all three phases before Step B begins its build phase. The execution order within a batch is always:

1. Step A: Build → Review → Verify (all must pass)
2. Step B: Build → Review → Verify (all must pass)
3. Step C: Build → Review → Verify (all must pass)

Never batch builds together. Never batch reviews together. Never batch verifications together. Never start Step B's build while Step A's review or verification is pending. Each phase gates the next phase. Each step gates the next step.

After a batch completes (all steps in the batch passed all phases), handle git according to the chosen mode, then present a batch summary to the user showing which steps completed. In user-managed mode, the user has already been paused after each step. In skill-managed mode, continue to the next batch without pausing unless a pause cadence boundary has been reached (milestone boundary for milestone pauses, or end of plan for simple plans and full auto).

### Subagent Dispatch

Use Claude Code's Agent tool to dispatch subagents. Reference `references/subagent-prompts.md` for the exact prompt templates. Substitute placeholders with actual values before dispatching.

**Build subagent dispatch:**
- Pass the step's objective text verbatim from the plan into `{{STEP_OBJECTIVE}}`.
- Pass the step's acceptance criteria list verbatim from the plan into `{{ACCEPTANCE_CRITERIA}}`.
- Pass the step's Phase 1 instructions verbatim from the plan into `{{PHASE_1_INSTRUCTIONS}}`.
- The build subagent has full filesystem access to read and write files.
- Capture the structured build summary from the subagent's response.

**Review subagent dispatch:**
- Pass the step's acceptance criteria list verbatim from the plan into `{{ACCEPTANCE_CRITERIA}}`.
- Pass the captured build summary into `{{BUILD_SUMMARY}}`.
- Pass the step's Phase 2 instructions verbatim from the plan into `{{PHASE_2_INSTRUCTIONS}}`.
- The review subagent reads files independently from disk — the build summary is provided for orientation and focus, not as a trusted source of truth.
- Parse the review response to identify any blocking findings.

**Verification subagent dispatch:**
- Pass the step's Phase 3 checklist verbatim from the plan into `{{PHASE_3_CHECKLIST}}`.
- Pass the project's tool chain configuration (test runner, linter, type checker commands) into `{{TOOL_CHAIN_CONFIG}}`. Extract the tool chain from the plan's Tool Chain table — it was confirmed by the user during planning and is the authoritative source. If the plan has no Tool Chain table, detect from the project's config files (pyproject.toml, package.json, Makefile, etc.) as a fallback. Cache the resolved configuration and reuse it for subsequent verification dispatches unless the plan explicitly changes tool chain requirements.
- The verification subagent runs actual tools and reports pass/fail per checklist item.
- Parse the verification response to identify any failures. A single failed check means the entire verification phase fails.

### Failure Handling

If the review subagent returns any **blocking** finding, stop immediately. Do not proceed to verification. Do not attempt to auto-fix. Do not retry the build. Present the full blocking findings to the user with file:line references and explanations. Wait for the user to provide guidance on how to proceed. The user may choose to:
- Fix the issues manually and ask to re-run the review.
- Ask the skill to re-run the build with additional instructions.
- Skip the step (mark it as skipped, not as complete).
- Abort execution entirely.

If the verification subagent returns any **failed** check, stop immediately. Do not proceed to the next step. Do not attempt to auto-fix. Present the full failure details to the user including complete error output. Wait for the user to provide guidance. The same options apply as with blocking review findings.

Advisory review findings do not block execution. Present them to the user as informational notes after the step passes all phases. The user may choose to address them later or ignore them.

Steps that have already passed all three phases and been marked complete retain their checkmarks regardless of subsequent failures in other steps. A failure in Step B does not roll back Step A.

### Progress Tracking

Immediately after a step passes all three phases (build, review, verify), mark it complete in the plan file with two updates:

1. Prepend a checkmark to the step heading. For example, transform `### Step 1: Auth middleware` into `### ✅ Step 1: Auth middleware`.
2. Tick all markdown checkboxes within the completed step by changing `- [ ]` to `- [x]` for every checkbox in that step's Phase 3 verification checklist (and any other checkboxes within the step).

Write both changes to the plan file on disk so progress persists across sessions. On resume, scan the plan file for the first step heading that does not have a ✅ prefix and begin execution from that step.

If the user previously paused execution (user-managed git mode), re-read the entire plan file before resuming. The user may have edited the plan during the pause — added steps, removed steps, reordered steps, or modified instructions. Honor whatever the plan file contains at resume time.

### Plan Modifications on Resume

Always re-read the plan file from disk when resuming execution. Never rely on a cached version of the plan from a previous session or earlier in the conversation. The user may have:
- Edited step instructions based on review findings.
- Added new steps between existing steps.
- Removed steps that are no longer needed.
- Reordered steps for dependency reasons.
- Modified acceptance criteria or verification checklists.

Accept the plan as-is at resume time. Do not warn about or question changes unless a step's dependencies appear to be broken (e.g., a step references artifacts from a removed step).

## Subagent Architecture

| Phase | Receives | Does |
|-------|----------|------|
| Build | Phase 1 instructions (verbatim from plan); full filesystem access | Read context, implement changes, write tests, return structured summary listing files changed with intent, key decisions, and any deviations |
| Review | Acceptance criteria (verbatim from plan) + build summary (structured, for orientation) + Phase 2 instructions (verbatim from plan) | Read all changed files fresh from disk, verify each acceptance criterion is met, perform adversarial review, return findings categorized as blocking or advisory with file:line references |
| Verification | Phase 3 checklist (verbatim from plan, includes acceptance criteria) + tool chain configuration | Execute each check using actual tools, return pass/fail per checklist item with full error output on failure |

The review subagent reads files independently from disk. The build summary tells it where to look and what was intended, but the review subagent verifies everything by reading actual file contents. This prevents a build subagent from misrepresenting its own output.

The verification subagent runs real commands. It does not estimate whether tests pass. It does not guess linter output. It executes the tools and reports what happened.

## Batching Rules

- Maximum batch size: 3 steps.
- Execution within a batch: strictly serial. Step N must pass build, review, and verify before Step N+1 begins.
- After a batch completes:
  - **User-managed git mode**: the user has already paused after each step within the batch. Present the batch summary as cumulative context. The user commits at their discretion.
  - **Skill-managed git mode**: commits happen after each step within the batch. After the batch, present the batch summary. Pause behavior depends on the pause cadence:
    - Simple plan (single document): never pause at batch boundaries. Continue until the plan is complete.
    - Milestone pauses: pause only when the completed batch is the last batch in a milestone. Continue without pausing for mid-milestone batch boundaries.
    - Full auto: never pause at batch boundaries. Continue until the entire plan is complete.
- If a step fails within a batch, the remaining steps in that batch do not execute. Present the failure and wait for guidance.
- A batch never crosses milestone boundaries. If the current milestone has 2 remaining steps and the next milestone has steps, the current batch contains only those 2 steps.
- If the plan has fewer remaining steps than the batch size, the batch contains only the remaining steps.

## Step Reporting

After each step completes all three phases, present a step report. The level of detail depends on the execution mode:

- **User-managed mode** (pauses after each step): Present a full step report containing the step label and status, count of files changed, count of blocking and advisory findings (list advisory findings briefly if any exist), verification result summary (N/N checks passed), and the git action taken (paused for user).
- **Skill-managed continuous modes** (simple plans, or full auto): Condense each step report to a single line — step label, status, files changed, verification result, and commit hash. Present a comprehensive summary at the end of the run.
- **Skill-managed milestone pauses**: Use the condensed single-line format for steps within a milestone. Present a full milestone summary at each pause point.

After a batch completes, present a batch summary listing all steps that completed in the batch and cumulative statistics.

## Additional Resources

Refer to `references/subagent-prompts.md` for the complete prompt templates used when dispatching build, review, and verification subagents. These templates contain the exact system context, process instructions, constraints, and return format specifications for each subagent type. Substitute all placeholders with actual values before dispatching.
