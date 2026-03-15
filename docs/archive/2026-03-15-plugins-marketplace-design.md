# Claude Plugins Marketplace — Design Spec

**Date**: 2026-03-15
**Status**: Draft

## Overview

This repository (`claude-plugins`) serves as a personal Claude Code plugins marketplace. It follows the same structure as `anthropics/claude-plugins-official` — a `plugins/` directory at the root containing independent, self-contained plugins. Once registered as a marketplace in Claude Code's `known_marketplaces.json`, plugins are discoverable, installable individually, and auto-updated whenever new commits are pushed to `main`.

## Scope

Two plugins for the initial release:

1. **code-audit** — Language-agnostic codebase auditing with structured report output
2. **blueprint** — Collaborative implementation planning with a 3-phase cycle per step, plus plan execution via subagents. Contains two skills: `blueprint` (planning) and `execute` (execution). These are bundled together because execute depends on blueprint's output format and they are always used together.

Codex compatibility is out of scope. Claude Code only.

## Marketplace Structure

```
claude-plugins/
├── README.md
├── LICENSE
├── plugins/
│   ├── code-audit/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   └── skills/
│   │       └── audit/
│   │           ├── SKILL.md
│   │           └── references/
│   │               ├── categories.md
│   │               └── report-template.md
│   └── blueprint/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           ├── blueprint/
│           │   ├── SKILL.md
│           │   └── references/
│           │       ├── step-template.md
│           │       └── tool-discovery.md
│           └── execute/
│               ├── SKILL.md
│               └── references/
│                   └── subagent-prompts.md
```

### Auto-Update Mechanism

Claude Code's marketplace system handles auto-updates natively. When registered via `known_marketplaces.json`, Claude Code:

1. Tracks the git commit SHA of installed plugins
2. On session start, checks for new commits on `main`
3. Pulls and caches the latest version automatically

No custom update mechanism is needed — pushing to `main` is the release trigger.

---

## Plugin 1: code-audit

### Purpose

Perform language-agnostic codebase auditing that produces a structured, severity-ranked report file. Covers security, correctness, performance, and code quality concerns.

### Effort Level

This skill operates at maximum effort. Every file in scope is analyzed thoroughly — no sampling, no shortcuts, no "this looks fine" skimming. The audit should read every line, trace data flows, follow call chains across files, and consider interactions between components. The goal is to find issues that a cursory review would miss. If the scope is large, the skill should use parallel subagents to divide the work without reducing depth.

### Audit Categories

- **Security vulnerabilities** — Injection flaws, auth issues, secrets in code, OWASP top 10
- **Race conditions and concurrency** — Shared state, missing locks, async hazards
- **Dead code** — Unreachable paths, unused imports/variables/functions
- **Anti-patterns and code smells** — Coupling, god objects, magic numbers, duplication
- **Performance** — N+1 queries, unnecessary allocations, algorithmic complexity
- **Correctness** — Off-by-one errors, null/undefined handling, edge cases, type mismatches
- **Error handling gaps** — Swallowed exceptions, missing error paths, incomplete cleanup

### Invocation Flow

1. **User triggers the skill** — e.g., "audit this codebase", "audit src/auth/", "audit for security issues"
2. **Scope resolution**:
   - If scope was specified in the prompt, use it directly. Scope is expressed in natural language (e.g., "the auth module", "src/api/", "all Python files"). The skill interprets the intent — no structured pattern syntax.
   - If no scope specified, ask: "Audit the entire repo or a specific path/module?"
3. **Category confirmation** — Present the list of audit categories that will be checked. Allow the user to add or remove categories before proceeding.
4. **Systematic analysis** — Scan files within scope, analyzing against confirmed categories
5. **Report generation** — Produce `AUDIT-REPORT-YYYY-MM-DD.md` at the project root. If a file with that name already exists, append an incrementing suffix (e.g., `AUDIT-REPORT-2026-03-15-2.md`). If zero findings are found, still produce the report with a clean bill of health (all counts at 0).

### Report Format

```markdown
# Audit Report — YYYY-MM-DD

## Summary
- **Scope**: [what was audited]
- **Categories**: [which categories were checked]
- **Findings**: X critical, Y high, Z medium, W low

## Critical

### [AUDIT-001] Finding title
- **Category**: Security
- **Location**: path/to/file.py:42-58
- **Description**: What the issue is
- **Impact**: What could go wrong
- **Recommendation**: How to fix it

## High
...

## Medium
...

## Low
...
```

Findings are deduplicated and grouped by severity (critical > high > medium > low), with each finding referencing its category and exact file location.

### Skill Files

- **SKILL.md** (~1,500-2,000 words) — Workflow instructions: scope negotiation, systematic scanning approach, deduplication strategy, report generation
- **references/categories.md** — Detailed checklist per audit category with specific things to look for in each
- **references/report-template.md** — The report structure template shown above

### plugin.json

```json
{
  "name": "code-audit",
  "description": "Language-agnostic codebase auditing with structured severity-ranked reports",
  "author": {
    "name": "fab"
  }
}
```

---

## Plugin 2: blueprint

### Purpose

Produce implementation plan artifacts through collaborative dialogue. Every step in the plan includes a mandatory 3-phase cycle: build, adversarial review, and verification. Plans describe *what* to do and *why* — not *how* at the code level.

### Effort Level

This skill operates at maximum effort. The planning process is exhaustive — every requirement is explored, every edge case is considered, every dependency is mapped. The skill should deeply understand the codebase before proposing a plan: read relevant files, trace architectures, understand existing patterns. The resulting plan should be comprehensive enough that execution requires no guesswork. Cutting corners or producing a superficial plan is never acceptable.

### Design Principles

- **Collaborative, not assumptive** — Ask clarifying questions before committing to a plan structure. Do not make assumptions about requirements, constraints, or approach.
- **Prose over code** — Plans describe intent, architecture, and acceptance criteria in natural language. Minimal to no code snippets, because plans change and code in plan documents gets stale quickly. The only acceptable exceptions: interface/contract signatures when they define a boundary, configuration keys that must be exact, or schema shapes critical to understanding the step. Even then, keep them short.
- **Adaptive complexity** — Simple tasks get one plan file; complex tasks get a numbered milestone folder. Heuristics: use a single file when the plan has ~5 or fewer steps with no natural grouping; switch to milestones when there are distinct phases (e.g., "data layer, then API, then UI"), cross-cutting concerns, or more than ~8 steps. When in doubt, ask the user.

### Invocation Flow

1. **User triggers the skill** — e.g., "create a blueprint for X", "blueprint this feature"
2. **Understand the task** — Read the codebase for context. Ask clarifying questions iteratively — avoid dumping a large list, but batching 2-3 related questions per turn is fine. Do not proceed until the task is well-understood.
3. **Tool discovery** — Detect the project's linter, type checker, test runner, formatter, build tool, etc. Present the discovered tool chain to the user for confirmation. The user may add, remove, or reorder tools.
4. **Assess complexity** — Determine single-doc vs multi-milestone based on scope.
5. **Generate the plan** — Write the plan artifact(s) with the 3-phase cycle baked into every step.

### Output Formats

**Simple plan** (single document):
```
docs/plans/YYYY-MM-DD-<topic>-plan.md
```

**Complex plan** (milestone folder):
```
docs/plans/YYYY-MM-DD-<topic>/
├── 01_milestone_name.md
├── 02_milestone_name.md
├── 03_milestone_name.md
└── ...
```

### Step Structure (3-Phase Cycle)

Every step in the plan follows this structure:

```markdown
### Step N: [Step Title]

**Objective**: What this step achieves and why.

**Acceptance criteria**:
- [Criterion 1]
- [Criterion 2]

#### Phase 1 — Build

[Prose instructions for what to implement. Describes intent, components to create/modify, data flow, and integration points. Must include what tests to write — every piece of new or changed functionality gets tested. No code snippets unless absolutely necessary for clarity.]

#### Phase 2 — Adversarial Review

[Instructions for a thorough, adversarial review of the Phase 1 output. What to look for: correctness, edge cases, security implications, performance concerns, error handling, adherence to acceptance criteria. The review should actively try to find flaws.]

#### Phase 3 — Verification

[Run the confirmed tool chain. Specific checks to perform:]
- [ ] New or modified code has corresponding tests (unit, integration, or both as appropriate)
- [ ] All tests pass (new and existing)
- [ ] Test coverage is adequate — no untested happy paths, edge cases, or error paths
- [ ] Linter passes
- [ ] Type checker passes (if applicable)
- [ ] All acceptance criteria from this step are met
- [ ] [Any step-specific verification items]
```

### Tool Discovery

The skill detects project tooling by examining:
- Config files (pyproject.toml, package.json, Makefile, Cargo.toml, etc.)
- Lock files (uv.lock, package-lock.json, Cargo.lock, etc.)
- CI configuration (.github/workflows/, .gitlab-ci.yml, etc.)
- Existing scripts (Makefile targets, npm scripts, etc.)

Detected tools are presented to the user for confirmation before being embedded in the plan's verification phases.

### Skill Files

- **SKILL.md** (~2,000-3,000 words) — Workflow instructions: collaborative questioning approach, tool discovery process, complexity assessment, plan generation with 3-phase cycle. May run slightly longer than typical skills due to behavioral complexity.
- **references/step-template.md** — The 3-phase cycle template with detailed guidance on what each phase should contain
- **references/tool-discovery.md** — How to detect project tooling across languages and frameworks

### plugin.json

```json
{
  "name": "blueprint",
  "description": "Collaborative implementation planning and execution with build-review-verify cycles per step",
  "author": {
    "name": "fab"
  }
}
```

---

## Execute Skill (within blueprint plugin)

### Purpose

Execute blueprint plans step-by-step, driving each step through its 3-phase cycle (build, adversarial review, verification) using dedicated subagents. The main conversation orchestrates and stays lightweight; heavy work happens in subagents to avoid context window exhaustion.

### Effort Level

This skill operates at maximum effort. Each subagent phase is thorough — the build subagent implements completely, the review subagent scrutinizes aggressively, and the verification subagent validates rigorously. No phase is a rubber stamp.

### Invocation Flow

1. **User triggers the skill** — e.g., "execute this blueprint", "run the plan", "execute 01_milestone_name.md"
2. **Locate the plan** — Find the plan file(s). If multiple plans exist, ask the user which one. If a milestone folder, present the milestones and ask which to start with (or start from the first unmarked one).
3. **Git handling** — Ask the user: "Do you want to handle git commits and branches yourself, or should I manage them?"
   - **User-managed git**: Pause after each individual step for the user to review and commit. Batching still applies for execution grouping, but the user gets a pause point after every step within the batch. Wait for explicit go-ahead before proceeding.
   - **Skill-managed git**: Commit after each individual step automatically. Never include any mention of Claude Code, Claude, AI, or co-authorship in commit messages.
4. **Step execution** — Process steps in batches of up to 3 (never more, even if steps are small). For each step, dispatch subagents via Claude Code's Agent tool:
   - **Build subagent**: Receives the step's Phase 1 instructions. Has full filesystem access to discover and read relevant context. Implements the changes, writes tests, and returns a structured summary: files changed, intent per file, key decisions made.
   - **Review subagent**: Receives the build subagent's structured summary (intent context) AND the step's Phase 2 instructions (review checklist). Does its own fresh read of the actual files. Returns findings categorized as **blocking** (must fix before proceeding) or **advisory** (noted for awareness, does not block).
   - **Verification subagent**: Receives the Phase 3 checklist (which embeds the step's acceptance criteria) and the tool chain configuration from the plan. Runs tools against actual file state. Reports pass/fail for each check.
5. **Failure handling** — If the review subagent reports any **blocking** findings, or if any verification check fails, stop immediately and surface the problem to the user with full details. Do not retry automatically. Wait for user guidance. If a step fails mid-batch, already-completed steps in that batch retain their changes and checkmarks — no rollback. The failed step remains unmarked.
6. **Progress tracking** — Mark each step done immediately after it passes all three phases, by updating the plan file header (e.g., `### Step 1: Auth System` → `### ✅ Step 1: Auth System`). This makes it easy to stop and resume later — the skill reads the plan, finds the first unmarked step, and continues from there.
7. **Plan modifications** — During pauses (user-managed git or failure stops), the user may edit the plan. On resume, the skill re-reads the plan file to pick up any changes before continuing.

### Subagent Architecture

Each phase runs in its own subagent to keep the main conversation lean:

| Phase | Subagent receives | Subagent does |
|-------|-------------------|---------------|
| **Build** | Step's Phase 1 instructions; full filesystem access | Reads relevant files, implements changes, writes tests, returns structured summary (files changed, intent per file, key decisions) |
| **Review** | Build subagent's structured summary + step's Phase 2 instructions | Fresh read of actual files, adversarial review per Phase 2 checklist, returns findings as blocking or advisory |
| **Verification** | Step's Phase 3 checklist (includes acceptance criteria) + tool chain config from plan | Runs tools against actual file state, checks acceptance criteria, returns pass/fail per item |

All subagents are dispatched via Claude Code's Agent tool. The review subagent must form its own independent judgment by reading the code directly — it uses the build summary only to know what to focus on, not as a source of truth. The tool chain configuration (which linter, type checker, test runner, etc.) is embedded in the plan's Phase 3 sections by the blueprint skill during planning.

### Batching Rules

- Maximum 3 steps per batch, even if steps are trivially small
- Steps within a batch are executed strictly serially: Step A (build → review → verify) must fully pass before Step B (build → review → verify) begins. Each phase gates the next — never run multiple steps' build phases together, and never batch reviews or verifications across steps.
- After all steps in a batch pass, pause for user review/commit (user-managed git) or commit and continue (skill-managed git)

### Design Principles

- **User stays in control** — The main conversation orchestrates and communicates. No silent loops or retries.
- **Context preservation** — Subagents do the heavy lifting. The main conversation tracks progress and communicates results without filling up with implementation details.
- **Resumable** — Green checkmarks in plan files mean execution can stop and resume across sessions. The skill reads the plan, finds the first unmarked step, and continues from there.
- **No Claude Code attribution** — Commit messages never mention Claude Code, Claude, AI, or co-authorship.

### Skill Files

- **SKILL.md** (~2,000-2,500 words) — Orchestration workflow: plan location, git handling, batching rules, subagent dispatch, failure handling, progress marking
- **references/subagent-prompts.md** — Detailed prompt templates for build, review, and verification subagents

Note: This skill lives under `plugins/blueprint/skills/execute/`, not as a standalone plugin. It shares the blueprint plugin's `plugin.json`.

---

## What Is NOT In Scope

- Codex CLI compatibility
- Custom update mechanisms (Claude Code's native marketplace auto-update suffices)
- Any web UI or dashboard — this is a git-based marketplace only
