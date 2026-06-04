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

Pushing every build, review, and verification into a subagent is what keeps this skill viable across long, multi-milestone plans: the bulk of the token cost — reading source files, running tools, capturing full output — is spent inside subagents whose context is reclaimed the moment they return. The coordinator's context holds only the plan and the distilled summary each subagent reports back, never the file contents or tool output, so it stays lean no matter how large the plan grows.

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

This rule applies regardless of plan size, step size, or apparent triviality. A one-line config change is still dispatched as a build subagent. The main conversation does not edit source files, run tests, or run linters directly during step execution. Tools the main conversation may use during execution are limited to: reading the plan, writing progress markers back into the plan (the format-specific mutations defined in "Progress Tracking" below), dispatching agents, and creating git commits in skill-managed mode.

**Why this is absolute:** keeping all heavy work in subagents is the mechanism that keeps the coordinator's context window lean. If the main conversation reads source files or captures tool output directly, that material accumulates in its context and the skill degrades over a long plan. Subagent context is ephemeral — reclaimed the moment the subagent returns its summary — so the coordinator pays only for the distilled result, not for the work that produced it.
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
4. **Step Execution** — dispatch build, review, and verification subagents per step, strictly serially; pause and commit per the chosen modes; mark progress in the plan file.

## Phase 1: Locate the Plan

Scan `docs/plans/` for plan files. Plans may be written in either Markdown (`.md`) or HTML (`.html`). Search for both extensions and treat them equivalently — a single `.md` file, a single `.html` file, a milestone folder containing `.md` files, and a milestone folder containing `.html` files are all valid plan structures.

If a single plan file or folder exists, use it. If multiple plan files or folders exist, present the list to the user and ask which to execute. For milestone folders containing numbered step files, start from the first step that is not marked complete (see "Progress Tracking" below for how completion is encoded in each format). If the plan file is specified directly in the user's request (e.g., "execute 01_milestone_name.md" or "execute 01_milestone_name.html"), use that file without asking.

If a milestone folder mixes `.md` and `.html` step files, this is out-of-spec: `write-blueprint` never produces mixed folders. Warn the user, list the mixed files, and ask which format to use for the run. Do not silently pick one extension over the other.

**Detect the plan's format** by file extension. The format affects only how the plan is parsed and how progress is written back; it does not affect how subagents are dispatched. Cache the detected format for the rest of the session.

Read the entire plan file into context before beginning execution. Identify all step headings or step articles, their phase instructions (Phase 1 build, Phase 2 review, Phase 3 verification), and any dependencies between steps:

- For Markdown plans: step headings (`### Step N: ...`) delimit steps; phase headings (`#### Phase 1 — Build`, etc.) delimit phases.
- For HTML plans: `<article class="step" data-step="N">` elements delimit steps; `<section class="phase phase-build">`, `<section class="phase phase-review">`, `<section class="phase phase-verify">` delimit phases.

Count the total number of steps and report to the user how many steps the plan contains and how many are already marked complete before asking to proceed.

If the user requests executing only specific steps (e.g., "execute steps 3-5" or "skip step 2"), honor the request — execute only the specified steps in order, skipping others. Warn if skipped steps contain dependencies required by the requested steps.

Determine plan complexity from the plan structure:

- **Simple plan**: a single plan document (either `.md` or `.html`).
- **Complex plan**: a milestone folder containing multiple milestone files (either `.md` or `.html`).

The complexity classification is used by the Cadence Gate below. Format does not affect complexity.

## Phase 2: Git Mode Gate

<GIT-MODE-GATE>
**This phase ends with the git mode question — nothing else.** After locating the plan and reporting progress, ask the user which git mode to use and wait for their answer. Do not start step execution. Do not ask the cadence question yet — that comes after the user's git mode answer. Do not dispatch any subagent.

Present exactly two options to the user, with this wording or equivalent:

> How would you like git to be handled during execution?
>
> 1. **User-managed**: I pause after every step. You inspect, stage, and commit at your own pace.
> 2. **Skill-managed** *(default)*: I commit automatically after each step that passes all three phases.

Wait for the user's answer before continuing. Re-ask every new execution session — never carry the choice over from a previous session. Skill-managed is the default option presented, but the gate is still surfaced every session: do not silently start execution without the user choosing or pre-stating a mode.

**Pre-stated answer**: if the user's first execution message already specifies a mode unambiguously (e.g., "execute this plan in skill-managed mode", "use user-managed mode and execute"), accept it as the gate answer and proceed. Do not re-ask for ceremonial confirmation. The gate's purpose is to ensure the user has chosen, not to perform the question. If the wording is ambiguous (e.g., "manage it yourself"), ask the gate question.
</GIT-MODE-GATE>

### User-managed mode

- Pause after each individual step that completes all three phases (build, review, verify).
- The user inspects changes, stages files, and commits at their discretion.
- Do not create any commits automatically.
- Do not resume execution until the user explicitly says to continue.
- The Cadence Gate is skipped — user-managed mode always pauses per step regardless of plan complexity.

### Skill-managed mode

- After each individual step passes all three phases, create a commit automatically.
- If a step requires human intervention (blocking review findings or failed verification checks), do not commit any of that step's changes until the intervention is fully resolved and the step passes all remaining phases. A partial step is never committed.
- Include the plan file's progress updates (✅ heading prefix and ticked checkboxes for Markdown plans; `data-status="complete"`, badge text, ✅ heading prefix, and `checked` attributes for HTML plans) in the same commit as the step's implementation changes — do not create a separate commit for plan progress.
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
- Within a milestone, run all steps continuously, pausing only at the milestone boundary.

### Full auto

- Run all steps across all milestones continuously without pausing.
- Present a final summary when the plan is complete.

## Phase 4: Step Execution via Subagents

Once both gates have closed, begin step execution. All work in this phase is performed by subagents dispatched from the main conversation.

### Subagent Dispatch

Use Claude Code's Agent tool to dispatch subagents. Reference `references/subagent-prompts.md` for the exact prompt templates. Substitute placeholders with actual values before dispatching.

**Extracting placeholder content from the plan**: For Markdown plans, copy the content beneath the matching heading verbatim. For HTML plans, extract the inner content of the matching section (`<p class="step-objective">` for objective, `<section class="acceptance-criteria">` for criteria, `<section class="phase phase-build">` for Phase 1 instructions, `<section class="phase phase-review">` for Phase 2, `<section class="phase phase-verify">` for Phase 3). Preserve semantic markup (`<ul>`, `<ol>`, `<pre>`, `<code>`, `<p>`, `<a>`) but strip the outer wrapper tag. The subagent reads HTML natively; do not flatten lists or code blocks to plain text.

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
- Pass the project's tool chain configuration (test runner, linter, type checker commands) into `{{TOOL_CHAIN_CONFIG}}`. Extract the tool chain from the plan's Tool Chain table — it was confirmed by the user during planning and is the authoritative source. For HTML plans, the table lives inside `<section id="tool-chain">`. If the plan has no Tool Chain table, detect from the project's config files (pyproject.toml, package.json, Makefile, etc.) as a fallback. Cache the resolved configuration and reuse it for subsequent verification dispatches unless the plan explicitly changes tool chain requirements.
- The verification subagent runs actual tools and reports pass/fail per checklist item.
- Parse the verification response to identify any failures. A single failed check means the entire verification phase fails.

### Execution Order

Execute steps **strictly serially**, one at a time. Step A must complete its full build, review, and verify cycle and pass all three phases before Step B begins its build phase. The order for every step is always:

1. **Build** subagent → 2. **Review** subagent → 3. **Verify** subagent — all three must pass before the step is complete.

Never start a step's build while the previous step's review or verification is still pending. Never run the builds (or reviews, or verifications) of multiple steps together. Each phase gates the next phase within a step; each step gates the next step. The coordinator dispatches exactly one subagent at a time and retains only the structured summary it returns — never the file contents or tool output — which is what keeps its context lean across a long plan.

After a step passes all three phases, handle git according to the chosen mode and mark progress in the plan file, then decide whether to pause:

- **User-managed mode**: pause after every step for the user to inspect and commit.
- **Skill-managed mode**: commit automatically and continue to the next step without pausing — unless a milestone boundary has been reached under the milestone-pauses cadence, in which case pause and ask. Simple plans and full auto run straight through to the end.

Pausing is governed entirely by git mode and milestones; there is no fixed-size step grouping. If a step fails — a blocking review finding or a failed verification check — stop immediately and do not execute any subsequent step. Present the failure and wait for guidance.

### Failure Handling

If the review subagent returns any **blocking** finding, stop immediately. Do not proceed to verification. Do not attempt to auto-fix. Do not retry the build. Do not commit any changes from this step. Present the full blocking findings to the user with file:line references and explanations. Wait for the user to provide guidance on how to proceed. The user may choose to:
- Fix the issues manually and ask to re-run the review.
- Ask the skill to re-run the build with additional instructions.
- Skip the step (mark it as skipped, not as complete).
- Abort execution entirely.

If the verification subagent returns any **failed** check, stop immediately. Do not proceed to the next step. Do not attempt to auto-fix. Do not commit any changes from this step. Present the full failure details to the user including complete error output. Wait for the user to provide guidance. The same options apply as with blocking review findings.

After the user resolves the intervention (manual fix, re-run build, etc.), re-run the remaining phases from the point of failure. Only commit the step once it passes all three phases cleanly — the commit captures the final resolved state, not intermediate attempts.

Advisory review findings do not block execution. Present them to the user as informational notes after the step passes all phases. The user may choose to address them later or ignore them.

Steps that have already passed all three phases and been marked complete retain their completion markers regardless of subsequent failures in other steps. A failure in Step B does not roll back Step A.

### Progress Tracking

Immediately after a step passes all three phases (build, review, verify), mark it complete in the plan file. The exact mutation depends on the plan format detected in Phase 1.

**Markdown plans (`.md`)**: Apply two updates:

1. Prepend a checkmark to the step heading. For example, transform `### Step 1: Auth middleware` into `### ✅ Step 1: Auth middleware`.
2. Tick all markdown checkboxes within the completed step by changing `- [ ]` to `- [x]` for every checkbox in that step's Phase 3 verification checklist (and any other checkboxes within the step).

**HTML plans (`.html`)**: Apply three updates to the step's `<article class="step" data-step="N">` element:

1. Change the `data-status` attribute from its current value (typically `pending`) to `complete`. This is the canonical machine-readable state and the basis for resume detection.
2. Prepend `✅ ` to the inner text of the step's `<h3>` element inside `<header class="step-header">`. For example, `<h3>Step 1: Auth middleware</h3>` becomes `<h3>✅ Step 1: Auth middleware</h3>`. This is the human-visible signal.
3. Update the status badge text and add the `checked` attribute to verification checkboxes:
   - Change `<span class="status-badge">Pending</span>` to `<span class="status-badge">Complete</span>` (the badge color updates automatically via the inline CSS).
   - For every `<input type="checkbox" disabled>` inside that step's `<section class="phase phase-verify">`, add the `checked` attribute → `<input type="checkbox" disabled checked>`. Do not modify checkboxes belonging to other steps.

Use `Edit` (or equivalent surgical string replacement) for these mutations — do not rewrite the entire file. The HTML scaffold defined in `references/step-template-html.md` of the `write-blueprint` skill uses stable IDs (`id="step-N"`) and class names that make targeted edits straightforward.

Write all changes to the plan file on disk so progress persists across sessions. On resume:

- For Markdown plans, scan for the first step heading without a ✅ prefix.
- For HTML plans, scan for the first `<article class="step">` whose `data-status` is not `complete` (typically `pending`, or `failed`/`skipped` if a prior session encountered issues).

Other terminal statuses (`failed`, `skipped`) follow the same mutation pattern with the appropriate `data-status` value and badge text. Do not prepend `✅ ` for non-complete terminal states.

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

## Red Flags — STOP and re-read this skill

If any of these thoughts appear, the skill is about to be violated. Stop and correct course before continuing.

| Thought | Reality |
|---------|---------|
| "This plan is short, I'll skip the git question." | Phase 2 runs every session regardless of plan size. Ask. |
| "User just said 'go' — I'll assume skill-managed." | The gate is still presented every session. Skill-managed is the default option, but "go" is not an answer to a gate the user has not seen — if the gate has not been asked, ask it. |
| "The user picked skill-managed last time, I'll reuse it." | Mode choices are session-scoped, not plan-scoped. Re-ask every new session. |
| "This is a simple one-line edit, I'll just do it inline." | No. Dispatch a build subagent. The Subagent-Only law has no size exception. |
| "I'll run the tests myself to confirm — it's faster." | No. Verification runs in a verification subagent. Main chat does not run tests during step execution. |
| "The build subagent's summary is enough — I'll skip the review subagent." | No. Every step runs all three phases. Reviews read files from disk independently. |
| "I'll batch all the builds first, then all the reviews." | No. Steps run strictly serially: Build → Review → Verify per step, fully, before the next step's build begins. |
| "The plan only has 2 milestones, I'll skip the cadence question." | The Cadence Gate fires for any complex (multi-milestone) plan in skill-managed mode. Two milestones still counts. |
| "User-managed mode in a milestone plan — I should ask the cadence too." | No. Cadence Gate is conditional on skill-managed mode. User-managed always pauses per step. |
| "I'll squash the plan progress into a separate commit so the diff is cleaner." | No. Plan progress markers (Markdown checkboxes or HTML `data-status` / badge / checkbox attributes) go in the same commit as the step's implementation. |
| "Step B failed verification, but Step A passed — I'll roll back A." | No. Completed steps keep their completion markers. Failures stop forward progress; they do not undo prior progress. |

All of these mean: stop, re-read the relevant phase or law, and follow it as written.
