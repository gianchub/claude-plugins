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

Execute blueprint plans by driving each step through a strict build, review, and verify cycle using dedicated subagents. When a review or a verification turns up a defect, a remediation subagent fixes it and the step is reviewed and verified again — the user is interrupted only for decisions that are genuinely theirs. The main conversation is a coordination hub between the user and the subagents: it asks the user the questions defined in this skill, dispatches subagents to do the work, reports results, and handles git according to the user's chosen mode. The main conversation never performs build, review, remediation, or verification work itself.

Pushing every build, review, remediation, and verification into a subagent is what keeps this skill viable across long, multi-milestone plans: the bulk of the token cost — reading source files, running tools, capturing full output — is spent inside subagents whose context is reclaimed the moment they return. The coordinator's context holds only the plan and the distilled summary each subagent reports back, never the file contents or tool output, so it stays lean no matter how large the plan grows.

## Iron Laws

These rules govern every execution session. They override any local reasoning that suggests cutting corners.

<SUBAGENT-ONLY>
**The main conversation never performs build, review, remediation, or verification work itself.**

Every build, every review, every remediation, and every verification is dispatched to a subagent via Claude Code's Agent tool. The main conversation only:

- Asks the user the gate questions defined below.
- Dispatches subagents using the templates in `references/subagent-prompts.md`.
- Reports results, findings, remediation activity, and escalations back to the user.
- Handles git according to the user's chosen mode.
- Marks progress in the plan file.

This rule applies regardless of plan size, step size, or apparent triviality. A one-line config change is still dispatched as a build subagent, and a one-character fix for a review finding is still dispatched as a remediation subagent. The main conversation does not edit source files, run tests, or run linters directly during step execution. Tools the main conversation may use during execution are limited to: reading the plan, writing progress markers back into the plan (the format-specific mutations defined in "Progress Tracking" below), dispatching agents, and creating git commits in skill-managed mode.

**Why this is absolute:** keeping all heavy work in subagents is the mechanism that keeps the coordinator's context window lean. If the main conversation reads source files or captures tool output directly, that material accumulates in its context and the skill degrades over a long plan. Subagent context is ephemeral — reclaimed the moment the subagent returns its summary — so the coordinator pays only for the distilled result, not for the work that produced it.
</SUBAGENT-ONLY>

<HARD-GATES>
**Phases 1–3 are sequential hard gates. Phase 4 (step execution) does not begin until phases 1, 2, and 3 are complete for this session.**

Each gate ends with a question to the user and waits for their answer. Do not bundle a gate question with any other output. Do not assume answers from a previous session — every new execution session re-asks the gate questions, even if the same plan was executed before.
</HARD-GATES>

<PASS-BEFORE-COMMIT>
**A step is committed only after its final state passes both review and verification.**

Remediation changes code, which makes the review and verification that preceded it stale. A step is complete only when a full review pass and a full verification pass both succeed against the code as it stands after the last change. The commit captures that state — never an intermediate attempt, never a step with an open finding, never a partially remediated working tree.

This holds in both git modes. In skill-managed mode, no automatic commit is created while a step is still being remediated or has an open escalation. In user-managed mode, the step is not presented to the user as ready to commit until it has passed cleanly.
</PASS-BEFORE-COMMIT>

## Effort Level

Treat each phase as the only chance to get it right. Builds should be complete implementations, not drafts. Reviews should be genuinely adversarial. Verifications must run every tool and inspect every result. Remediation is held to the same bar: fix the cause of a finding, never just enough to turn a check green. Once a step passes, its code should be production-ready.

## Execution Phases

Execution proceeds through these phases in strict order. Phases 1–3 are gates that must complete before phase 4 starts.

1. **Locate the Plan** — find or confirm the plan file, report progress so far, ask whether to proceed.
2. **Git Mode Gate** — ask the user to choose user-managed or skill-managed git handling. Hard gate.
3. **Cadence Gate** *(conditional)* — only if the user chose skill-managed AND the plan is complex (multi-milestone). Ask whether to pause at milestone boundaries or run full auto. Hard gate.
4. **Step Execution** — dispatch build, review, and verification subagents per step, strictly serially; remediate in-scope findings automatically and escalate the rest; pause and commit per the chosen modes; mark progress in the plan file.

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
- A step that is still being remediated, or that has an open escalation, is never committed. Commit only once review and verification both pass against the step's final state, per the `<PASS-BEFORE-COMMIT>` law. A partial step is never committed.
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
- Pass the captured build summary into `{{BUILD_SUMMARY}}`. On a re-review after remediation, pass the original build summary followed by every remediation summary produced for this step, in order.
- Pass the step's Phase 2 instructions verbatim from the plan into `{{PHASE_2_INSTRUCTIONS}}`.
- The review subagent reads files independently from disk — the build summary is provided for orientation and focus, not as a trusted source of truth.
- Parse the review response to identify blocking findings and the remediation class the reviewer assigned to each one.

**Verification subagent dispatch:**
- Pass the step's Phase 3 checklist verbatim from the plan into `{{PHASE_3_CHECKLIST}}`.
- Pass the project's tool chain configuration (test runner, linter, type checker commands) into `{{TOOL_CHAIN_CONFIG}}`. Extract the tool chain from the plan's Tool Chain table — it was confirmed by the user during planning and is the authoritative source. For HTML plans, the table lives inside `<section id="tool-chain">`. If the plan has no Tool Chain table, detect from the project's config files (pyproject.toml, package.json, Makefile, etc.) as a fallback. Cache the resolved configuration and reuse it for subsequent verification dispatches unless the plan explicitly changes tool chain requirements.
- The verification subagent runs actual tools and reports pass/fail per checklist item.
- Parse the verification response to identify any failures. A single failed check means the entire verification phase fails.

**Remediation subagent dispatch:**
- Dispatch only when the "Failure Handling: Remediate or Escalate" rules below allow it.
- Pass the step's acceptance criteria list verbatim from the plan into `{{ACCEPTANCE_CRITERIA}}` and its Phase 1 instructions verbatim into `{{PHASE_1_INSTRUCTIONS}}` — together they define the step's intended scope.
- Pass the file list from the build summary (plus any files changed by earlier remediation cycles for this step) into `{{STEP_SCOPE}}`.
- Pass the exact defects into `{{FINDINGS_TO_FIX}}`: blocking review findings with their `file:line` references, or failed verification checks with their full error output, verbatim and untruncated.
- The remediation subagent has full filesystem access. It fixes what it can within scope and returns anything it cannot as an escalation.
- Capture the structured remediation summary and read its verdict before deciding what to dispatch next.

### Execution Order

Execute steps **strictly serially**, one at a time. Step A must complete its full cycle — build, review, verify, plus any remediation those phases call for — and pass before Step B begins its build phase. The order for every step is always:

1. **Build** subagent → 2. **Review** subagent → 3. **Verify** subagent.

When review or verification fails, a **remediation** subagent fixes the findings and review and verification both run again — see "Failure Handling: Remediate or Escalate" below. A step is complete only when a full review pass and a full verification pass both succeed against its final state.

Never start a step's build while the previous step's review, remediation, or verification is still pending. Never run the builds (or reviews, or verifications) of multiple steps together. Each phase gates the next phase within a step; each step gates the next step. The coordinator dispatches exactly one subagent at a time and retains only the structured summary it returns — never the file contents or tool output — which is what keeps its context lean across a long plan.

After a step passes all three phases, handle git according to the chosen mode and mark progress in the plan file, then decide whether to pause:

- **User-managed mode**: pause after every step for the user to inspect and commit.
- **Skill-managed mode**: commit automatically and continue to the next step without pausing — unless a milestone boundary has been reached under the milestone-pauses cadence, in which case pause and ask. Simple plans and full auto run straight through to the end.

Pausing during normal progress is governed entirely by git mode and milestones; there is no fixed-size step grouping. Findings do not pause the run by default — they are remediated in place. While an escalation is open, do not start the next step's build.

### Failure Handling: Remediate or Escalate

When a review returns blocking findings or a verification check fails, the step is not finished — but it is not automatically a question for the user either. Most findings are ordinary defects in work this run has just produced, and the run fixes them itself. The user is interrupted only when the decision is genuinely theirs to make.

**Default: remediate.** Blocking review findings and failed verification checks are fixed automatically by a remediation subagent, after which the step is reviewed and verified again. Do not ask the user for permission to fix a defect the run created.

**Exception: escalate.** Stop and hand the decision to the user when a finding carries an escalation class, when a step exhausts its remediation budget, when one subsystem keeps failing, or when the plan asks for a human checkpoint.

This policy is the same in both git modes. User-managed mode pauses after every step so the user can inspect and commit; it does not turn every finding into a question.

#### Escalation classes

| Class | Escalate when | Example |
|-------|---------------|---------|
| `design-decision` | The plan does not determine what the correct behaviour is. Two defensible fixes exist and choosing between them is a product or design call. | An acceptance criterion is silent on expired tokens: reject, refresh, or re-authenticate? |
| `scope-expansion` | The fix cannot stay inside the step's stated scope. It needs a new dependency, a change to a public interface, schema, or configuration contract, or edits to modules this step was never meant to touch. | The handler bug is only fixable by changing a shared serializer used by four other modules. |
| `destructive-or-external` | The fix would alter state that is not this step's to alter: existing data, migrations that drop or rewrite columns, files the step did not create, git history, remote or production systems, credentials, or anything with side effects outside the working tree. | The failing test passes only after a migration that drops an existing column. |
| `environment` | A verification check failed because a tool, service, or credential is missing or misconfigured. The code is not what is wrong. | `pytest: command not found`; the type checker is not installed; a required service is unreachable. |

Everything else is `in-scope` and is remediated without asking.

**Who assigns the class.** The review subagent labels every blocking finding with one of the classes above or with `in-scope`. Classify verification failures from the reported output: treat them as `in-scope` unless the output plainly shows a missing tool, an unavailable service, or a credential problem, which is `environment`. The remediation subagent has the final say — it has read the code, and any finding it returns as escalated is escalated regardless of the label the finding started with.

**If any finding in the set carries an escalation class, stop before remediating anything.** Do not fix the in-scope findings first. A design decision can change what the right fix is, and a half-remediated tree is harder for the user to reason about than an untouched one.

#### The remediation cycle

When every finding in the set is `in-scope` and no budget or checkpoint rule applies:

1. Dispatch a **remediation subagent** with the exact defects to fix — blocking findings with their `file:line` references, or failed checks with their full error output.
2. If it returns `ESCALATE`, stop. Report which findings it fixed, which it escalated, and why.
3. Otherwise re-run the **review** subagent, then the **verification** subagent, both in full. Remediation changes the code the review passed, and a fix can regress a check that was green. Never re-run only the failed check.
4. If both pass, the step is complete. Continue to git handling and progress tracking as normal.
5. If either fails again, return to step 1 — subject to the budgets below.

#### Budgets

Before dispatching any remediation subagent, apply these checks in order. The first one that fires wins.

1. **Escalation class** — any finding in the set is not `in-scope` → stop and escalate.
2. **Step budget** — this step has already used **two** remediation cycles → stop and escalate. Two focused attempts by agents that read the code did not clear the step; a third rarely does, and the real problem is usually not the local defect it appears to be.
3. **Subsystem budget** — the subsystem these findings belong to has already absorbed **two** remediation cycles this session, contributed by **two different steps** → stop and ask for an **architectural decision**, not for permission to fix. Name the subsystem, show the pattern across steps, and say what the repeated findings have in common. The two-step condition is what makes this a pattern rather than one stubborn step: a single step thrashing in one place is already covered by the step budget.

If none fires, dispatch.

**Tracking the subsystem counter.** Key it on the nearest common parent directory, below the project root, of the files cited in the findings (`src/auth/`, `app/models/`). If the cited files share no parent below the root, key it on the directory of the first blocking finding. Increment it once per remediation cycle dispatched, and record which step each cycle came from — the two-step condition depends on it. The counter is a signal that a human should look at the design, not a correctness mechanism, so approximate keying is fine.

Both budgets are session-scoped. They reset at the start of a new execution session, and the relevant counter resets once the user has resolved an escalation and asked to continue: their direction is new information, and the attempts that preceded it no longer count against the step or the subsystem.

#### Manual checkpoints

Pause when the plan itself asks a human to look — a step whose text explicitly calls for confirmation, sign-off, approval, or manual verification that cannot be settled by reading code or running a tool. Present exactly what needs confirming and wait. This is independent of git mode, and separate from user-managed mode's pause after every step.

#### Presenting an escalation

Every escalation tells the user the same four things:

- The step, and which phase failed.
- Every finding, with `file:line` references, and the full error output for verification failures — do not truncate.
- Why the run stopped instead of fixing it: the escalation class, the exhausted budget, the subsystem pattern, or the checkpoint.
- What remediation already changed on disk during this step, if anything, so the state of the working tree is clear.

Then wait. The user may:

- Give a decision or direction and ask for remediation to re-run with it.
- Fix the issue themselves and ask for the review and verification to re-run.
- Amend the plan and re-run the step.
- Skip the step (mark it skipped, not complete).
- Abort execution.

If the user chooses to continue the step by any route, it still ends with a full review pass and a full verification pass against its final state before it can be marked complete or committed. Never commit and never advance to the next step on a partial pass.

#### Advisory findings

Advisory findings never block and are never remediated. Fixing them is scope creep dressed up as diligence. Report them with the step; the user decides whether they matter and when.

A finding the remediation subagent **disputes** — one it argues is not a real defect — is not a failure either. The next review is the arbiter: if a fresh reviewer raises the same finding again, the cycle proceeds normally and the budgets apply. Surface disputes in the step report even when the step passes, so the user sees what was argued away.

#### Failures do not roll back

Steps that have already passed and been marked complete keep their completion markers regardless of what happens later. A failure in Step B does not undo Step A. If a step ends the session with an escalation still open, record it as `failed` (HTML plans) or leave it unmarked (Markdown plans) — never as complete.

### Progress Tracking

Immediately after a step's final state passes both review and verification, mark it complete in the plan file. The exact mutation depends on the plan format detected in Phase 1.

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

After each step completes, present a step report. Automatic remediation must stay visible: a user who never sees what was auto-fixed cannot judge whether to trust it. The level of detail depends on the execution mode:

- **User-managed mode** (pauses after each step): Present a full step report containing the step label and status, count of files changed, count of blocking and advisory findings (list advisory findings briefly if any exist), the remediation cycles run and what each one fixed, any disputed findings, the verification result summary (N/N checks passed), and the git action taken (paused for user).
- **Skill-managed continuous modes** (simple plans, or full auto): Condense each step report to a single line — step label, status, files changed, remediation cycle count, verification result, and commit hash. Present a comprehensive summary at the end of the run.
- **Skill-managed milestone pauses**: Use the condensed single-line format for steps within a milestone. Present a full milestone summary at each pause point.

Every milestone summary and end-of-run summary includes a remediation roll-up: which steps needed remediation, what each cycle fixed, and any disputed findings — so the auto-fixed work can be reviewed in one place rather than reconstructed from the step lines.

## Red Flags — STOP and re-read this skill

If any of these thoughts appear, the skill is about to be violated. Stop and correct course before continuing.

| Thought | Reality |
|---------|---------|
| "This plan is short, I'll skip the git question." | Phase 2 runs every session regardless of plan size. Ask. |
| "User just said 'go' — I'll assume skill-managed." | The gate is still presented every session. Skill-managed is the default option, but "go" is not an answer to a gate the user has not seen — if the gate has not been asked, ask it. |
| "The user picked skill-managed last time, I'll reuse it." | Mode choices are session-scoped, not plan-scoped. Re-ask every new session. |
| "This is a simple one-line edit, I'll just do it inline." | No. Dispatch a build subagent — or a remediation subagent if it is a fix for a finding. The Subagent-Only law has no size exception. |
| "I'll run the tests myself to confirm — it's faster." | No. Verification runs in a verification subagent. Main chat does not run tests during step execution. |
| "The build subagent's summary is enough — I'll skip the review subagent." | No. Every step runs all three phases. Reviews read files from disk independently. |
| "I'll batch all the builds first, then all the reviews." | No. Steps run strictly serially: Build → Review → Verify per step, fully, before the next step's build begins. |
| "The plan only has 2 milestones, I'll skip the cadence question." | The Cadence Gate fires for any complex (multi-milestone) plan in skill-managed mode. Two milestones still counts. |
| "User-managed mode in a milestone plan — I should ask the cadence too." | No. Cadence Gate is conditional on skill-managed mode. User-managed always pauses per step. |
| "I'll squash the plan progress into a separate commit so the diff is cleaner." | No. Plan progress markers (Markdown checkboxes or HTML `data-status` / badge / checkbox attributes) go in the same commit as the step's implementation. |
| "Step B failed verification, but Step A passed — I'll roll back A." | No. Completed steps keep their completion markers. Failures stop forward progress; they do not undo prior progress. |
| "The review found a real bug — I'll ask the user how they want to handle it." | No. An in-scope defect this run created is remediated automatically. Ask only for an escalation class, an exhausted budget, a subsystem pattern, or a plan checkpoint. |
| "Two findings are escalation-class, but the other three are easy — I'll fix those first." | No. If any finding in the set escalates, stop before remediating. The user's decision can change what the right fix is. |
| "Remediation fixed it and verification passes — I'll skip the re-review." | No. Remediation changed the code the review passed. Re-run review and verification, both in full. |
| "Only check #4 failed, so I'll re-run just check #4." | No. Fixes regress other checks. Re-run the whole verification checklist every time. |
| "This test is wrong — I'll mark it xfail so the step can pass." | Never. Weakening a test, loosening an assertion, or relaxing a lint rule to go green is a failed step wearing a pass. Escalate instead. |
| "The fix needs a column dropped from the users table, but it's the only way." | Escalate. Destructive or external state is never remediated automatically, however obvious the fix looks. |
| "Third remediation cycle in `src/auth` — one more attempt should do it." | No. The third cycle in one subsystem is an architectural decision, not another patch. Stop and present the pattern. |
| "The remediation subagent escalated, but I can see the answer — I'll direct it myself." | No. The escalation classes are the user's call by definition. Present it and wait. |
| "Advisory findings are cheap to fix while I'm in there." | No. Advisory findings are never remediated. Report them and move on. |
| "The step is basically done — I'll commit now and clean up in the next step." | No. `<PASS-BEFORE-COMMIT>`: the commit captures a state that passed review and verification in full, nothing less. |

All of these mean: stop, re-read the relevant phase or law, and follow it as written.
