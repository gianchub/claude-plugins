---
name: execute-blueprint
description: >
  This skill should be used when the user asks to "execute this blueprint",
  "execute the blueprint", "run the blueprint", "run the plan", "execute
  the plan", "start building from the plan", "implement the blueprint",
  "implement the plan", "continue the plan", "resume execution", "execute
  01_milestone_name.md", "execute step 3", "run steps 3-5", or "skip to
  step 4".
---

# Execute Blueprint Skill

## Purpose

Execute blueprint plans by driving each step through a strict build, review, and verify cycle using dedicated subagents. The main conversation is a coordination hub between the user and the subagents — it asks the user the questions defined in this skill, dispatches subagents to do the work, reports results, and handles git according to the user's chosen mode. The main conversation never performs build, review, or verification work itself.

## Iron Laws

These rules govern every execution session. They override any local reasoning that suggests cutting corners.

<SUBAGENT-ONLY>
**The main conversation never performs build, review, or verification work itself.**

Every build, every review, and every verification is dispatched to a subagent via Claude Code's Agent tool. The main conversation only:

- Asks the user the gate questions defined below.
- Dispatches subagents using the templates in `references/subagent-prompts.md`.
- Reports results, blocking findings, and verification failures back to the user.
- Handles git according to the user's chosen mode.
- Marks progress in the plan file.

This rule applies regardless of plan size, step size, or apparent triviality. A one-line config change is still dispatched as a build subagent. The main conversation does not edit source files, run tests, or run linters directly during step execution. Tools the main conversation may use during execution are limited to: reading the plan, writing checkmarks back into the plan, dispatching agents, and creating git commits in skill-managed mode.
</SUBAGENT-ONLY>

<HARD-GATES>
**Phases 1–3 are sequential hard gates. Phase 4 (step execution) does not begin until phases 1, 2, and 3 are complete for this session.**

Each gate ends with a question to the user and waits for their answer. Do not bundle a gate question with any other output. Do not assume answers from a previous session — every new execution session re-asks the gate questions, even if the same plan was executed before.
</HARD-GATES>

## Effort Level

Treat each phase as the only chance to get it right. Builds should be complete implementations, not drafts. Reviews should be genuinely adversarial. Verifications must run every tool and inspect every result. After all three phases pass, the step's code should be production-ready.

## Execution Phases

Execution proceeds through these phases in strict order. Phases 1–3 are gates that must complete before phase 4 starts.

1. **Locate the Plan** — find or confirm the plan file, report progress so far, ask whether to proceed.
2. **Git Mode Gate** — ask the user to choose user-managed or skill-managed git handling. Hard gate.
3. **Cadence Gate** *(conditional)* — only if the user chose skill-managed AND the plan is complex (multi-milestone). Ask whether to pause at milestone boundaries or run full auto. Hard gate.
4. **Step Execution** — dispatch build, review, and verification subagents per step in batches; pause and commit per the chosen modes; mark progress in the plan file.

## Phase 1: Locate the Plan

Scan `docs/plans/` for plan files. If a single plan file exists, use it. If multiple plan files or milestone folders exist, present the list to the user and ask which to execute. For milestone folders containing numbered step files, start from the first step that is not marked with a checkmark. If the plan file is specified directly in the user's request (e.g., "execute 01_milestone_name.md"), use that file without asking.

Read the entire plan file into context before beginning execution. Identify all step headings, their phase instructions (Phase 1 build, Phase 2 review, Phase 3 verification), and any dependencies between steps. Count the total number of steps and report to the user how many steps the plan contains and how many are already marked complete before asking to proceed.

If the user requests executing only specific steps (e.g., "execute steps 3-5" or "skip step 2"), honor the request — execute only the specified steps in order, skipping others. Warn if skipped steps contain dependencies required by the requested steps.

Determine plan complexity from the plan structure:

- **Simple plan**: a single plan document.
- **Complex plan**: a milestone folder containing multiple milestone files.

The complexity classification is used by the Cadence Gate below.

## Phase 2: Git Mode Gate

<GIT-MODE-GATE>
**This phase ends with the git mode question — nothing else.** After locating the plan and reporting progress, ask the user which git mode to use and wait for their answer. Do not start step execution. Do not ask the cadence question yet — that comes after the user's git mode answer. Do not dispatch any subagent.

Present exactly two options to the user, with this wording or equivalent:

> How would you like git to be handled during execution?
>
> 1. **User-managed**: I pause after every step. You inspect, stage, and commit at your own pace.
> 2. **Skill-managed**: I commit automatically after each step that passes all three phases.

Wait for the user's answer before continuing. Re-ask every new execution session — never carry the choice over from a previous session, and never assume a default.

**Pre-stated answer**: if the user's first execution message already specifies a mode unambiguously (e.g., "execute this plan in skill-managed mode", "use user-managed mode and execute"), accept it as the gate answer and proceed. Do not re-ask for ceremonial confirmation. The gate's purpose is to ensure the user has chosen, not to perform the question. If the wording is ambiguous (e.g., "manage it yourself"), ask the gate question.
</GIT-MODE-GATE>

### User-managed mode

- Pause after each individual step that completes all three phases (build, review, verify).
- The user inspects changes, stages files, and commits at their discretion.
- Do not create any commits automatically.
- Do not resume execution until the user explicitly says to continue.
- Even when steps are batched, pause after each individual step within the batch. Present a batch summary after the final step in a batch as a cumulative checkpoint, but the user has already had the opportunity to review after each step.
- The Cadence Gate is skipped — user-managed mode always pauses per step regardless of plan complexity.

### Skill-managed mode

- After each individual step passes all three phases, create a commit automatically.
- If a step requires human intervention (blocking review findings or failed verification checks), do not commit any of that step's changes until the intervention is fully resolved and the step passes all remaining phases. A partial step is never committed.
- Include the plan file's progress updates (✅ heading prefix and ticked checkboxes) in the same commit as the step's implementation changes — do not create a separate commit for plan progress.
- Use a clear, descriptive commit message in conventional commits format referencing the step.
- Never include Claude Code or AI attribution in commit messages. No co-authored-by lines, no bot signatures, no AI references of any kind.
- Pause cadence is determined by the Cadence Gate below.

## Phase 3: Cadence Gate (skill-managed only)

This gate is **conditional**. Apply it only if both of the following are true:

- The user chose skill-managed mode in Phase 2.
- The plan is complex (multi-milestone folder).

If the user chose user-managed mode, **skip this gate entirely** and go to Phase 4 — user-managed always pauses per step.

If the user chose skill-managed mode and the plan is **simple** (single document), **skip this gate** and use continuous execution: run all steps without pausing and present a final summary when the plan is complete.

If the user chose skill-managed mode and the plan is **complex**, run the gate below:

<CADENCE-GATE>
**This phase ends with the cadence question — nothing else.** Ask the user which pause cadence to use and wait for their answer. Do not start step execution.

Present exactly two options to the user, with this wording or equivalent:

> The plan has multiple milestones. Which pause cadence?
>
> 1. **Milestone pauses** *(default)*: I pause after each milestone document and ask whether to continue to the next.
> 2. **Full auto**: I run the entire plan without pausing across milestone boundaries and present a final summary.

Wait for the user's answer before continuing.

**Pre-stated answer**: if the user's first execution message (or their answer to the Git Mode Gate) already specifies a cadence unambiguously (e.g., "skill-managed full auto", "milestone pauses"), accept it as the gate answer and proceed. Do not re-ask for ceremonial confirmation.
</CADENCE-GATE>

### Milestone pauses

- After every milestone document completes, present a milestone summary and ask whether to continue.
- Within a milestone, run all steps continuously without pausing at batch boundaries.

### Full auto

- Run all steps across all milestones continuously without pausing.
- Present a final summary when the plan is complete.

## Phase 4: Step Execution via Subagents

Once both gates have closed, begin step execution. All work in this phase is performed by subagents dispatched from the main conversation.

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

### Batching and Order

Group steps into batches of up to 3 steps maximum. Execute steps **strictly serially** within a batch: Step A must complete its full build, review, and verify cycle and pass all three phases before Step B begins its build phase. The execution order within a batch is always:

1. Step A: Build → Review → Verify (all must pass)
2. Step B: Build → Review → Verify (all must pass)
3. Step C: Build → Review → Verify (all must pass)

Never batch builds together. Never batch reviews together. Never batch verifications together. Never start Step B's build while Step A's review or verification is pending. Each phase gates the next phase. Each step gates the next step.

After a batch completes (all steps in the batch passed all phases), handle git according to the chosen mode, then present a batch summary to the user showing which steps completed. In user-managed mode, the user has already been paused after each step. In skill-managed mode, continue to the next batch without pausing unless a pause cadence boundary has been reached (milestone boundary for milestone pauses, or end of plan for simple plans and full auto).

A batch never crosses milestone boundaries. If the current milestone has 2 remaining steps and the next milestone has steps, the current batch contains only those 2 steps. If the plan has fewer remaining steps than the batch size, the batch contains only the remaining steps. If a step fails within a batch, the remaining steps in that batch do not execute — present the failure and wait for guidance.

### Failure Handling

If the review subagent returns any **blocking** finding, stop immediately. Do not proceed to verification. Do not attempt to auto-fix. Do not retry the build. Do not commit any changes from this step. Present the full blocking findings to the user with file:line references and explanations. Wait for the user to provide guidance on how to proceed. The user may choose to:
- Fix the issues manually and ask to re-run the review.
- Ask the skill to re-run the build with additional instructions.
- Skip the step (mark it as skipped, not as complete).
- Abort execution entirely.

If the verification subagent returns any **failed** check, stop immediately. Do not proceed to the next step. Do not attempt to auto-fix. Do not commit any changes from this step. Present the full failure details to the user including complete error output. Wait for the user to provide guidance. The same options apply as with blocking review findings.

After the user resolves the intervention (manual fix, re-run build, etc.), re-run the remaining phases from the point of failure. Only commit the step once it passes all three phases cleanly — the commit captures the final resolved state, not intermediate attempts.

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

A new execution session also re-runs the gate phases (Phase 2, and Phase 3 if applicable). Resuming an in-progress plan does not skip the gates — the user's git mode and cadence preferences are session-scoped, not plan-scoped.

## Step Reporting

After each step completes all three phases, present a step report. The level of detail depends on the execution mode:

- **User-managed mode** (pauses after each step): Present a full step report containing the step label and status, count of files changed, count of blocking and advisory findings (list advisory findings briefly if any exist), verification result summary (N/N checks passed), and the git action taken (paused for user).
- **Skill-managed continuous modes** (simple plans, or full auto): Condense each step report to a single line — step label, status, files changed, verification result, and commit hash. Present a comprehensive summary at the end of the run.
- **Skill-managed milestone pauses**: Use the condensed single-line format for steps within a milestone. Present a full milestone summary at each pause point.

After a batch completes, present a batch summary listing all steps that completed in the batch and cumulative statistics.

## Red Flags — STOP and re-read this skill

If any of these thoughts appear, the skill is about to be violated. Stop and correct course before continuing.

| Thought | Reality |
|---------|---------|
| "This plan is short, I'll skip the git question." | Phase 2 runs every session regardless of plan size. Ask. |
| "User just said 'go' — I'll assume skill-managed." | Never assume. The user's word for "go" is their answer to a specific gate question. If the gate has not been asked, ask it. |
| "The user picked skill-managed last time, I'll reuse it." | Mode choices are session-scoped, not plan-scoped. Re-ask every new session. |
| "This is a simple one-line edit, I'll just do it inline." | No. Dispatch a build subagent. The Subagent-Only law has no size exception. |
| "I'll run the tests myself to confirm — it's faster." | No. Verification runs in a verification subagent. Main chat does not run tests during step execution. |
| "The build subagent's summary is enough — I'll skip the review subagent." | No. Every step runs all three phases. Reviews read files from disk independently. |
| "I'll batch all the builds first, then all the reviews." | No. Steps run strictly serially within a batch: Build → Review → Verify per step before moving to the next step. |
| "The plan only has 2 milestones, I'll skip the cadence question." | The Cadence Gate fires for any complex (multi-milestone) plan in skill-managed mode. Two milestones still counts. |
| "User-managed mode in a milestone plan — I should ask the cadence too." | No. Cadence Gate is conditional on skill-managed mode. User-managed always pauses per step. |
| "I'll squash the plan progress into a separate commit so the diff is cleaner." | No. Plan progress (checkmarks) goes in the same commit as the step's implementation. |
| "Step B failed verification, but Step A passed — I'll roll back A." | No. Completed steps keep their checkmarks. Failures stop forward progress; they do not undo prior progress. |

All of these mean: stop, re-read the relevant phase or law, and follow it as written.
