---
name: audit
description: >
  This skill should be used when the user asks to "audit this codebase",
  "audit this code", "security audit", "code audit", "find vulnerabilities",
  "check for bugs", "review code quality", "find dead code",
  "check for anti-patterns", "performance audit", "check for code smells",
  "technical debt", or "code health check".
---

# Code Audit Skill

## Purpose

Perform a thorough, language-agnostic audit of a codebase or subset of files, producing a structured report with findings ranked by severity. Deliver the report as a Markdown file at the project root following the template in `references/report-template.md`.

## Effort Level

Read every line of in-scope code. Do not skim, sample, or rely on heuristics to skip files. Trace data flows from external inputs through processing layers to outputs and storage. Follow call chains across module boundaries to detect issues that only manifest through component interaction.

When the scope is large enough that thoroughness would suffer in a single pass, split work across parallel subagents by module or directory, then merge and deduplicate findings (see Step 4 for partitioning strategy).

When even partitioned analysis cannot cover every line — due to context limits, codebase size, or unavailable subagents — prioritize depth on security-critical and correctness-critical paths: entry points, authentication/authorization flows, data persistence, and external API boundaries. For anti-patterns and dead code categories, a single-pass identification is sufficient without tracing multi-hop call chains.

## Workflow

### Step 1 — Resolve Scope

Determine the audit scope from the user's prompt. The user may specify:

- An entire repository or working directory.
- One or more directories (e.g., `src/`, `lib/api/`).
- A list of specific files.
- A functional area described in natural language (e.g., "the authentication flow").

If the prompt does not contain enough information to determine scope, ask a single clarifying question before proceeding. Do not guess at scope — an audit with unclear boundaries produces unreliable results.

If a previous audit report exists in the project and the user requests a re-audit, default to the same scope as the original audit. Offer to narrow to only files with prior findings if the user wants a faster pass, but do not narrow silently — new issues can appear in files that were previously clean.

Once scope is established, enumerate all files that fall within it. Exclude generated files (e.g., lock files, compiled output, vendored dependencies, minified bundles) unless the user explicitly includes them. While generated and vendored code is excluded from file-level analysis, cross-file analysis in Phase B should trace data flows into excluded code to determine whether it provides expected validation or safety guarantees. Report findings at the boundary, not within the excluded code.

State the resolved scope back to the user. If the scope is unambiguous, proceed immediately without waiting for confirmation.

### Step 2 — Select and Confirm Categories

Analyze the user's request and the codebase to select the most relevant audit categories from the full list:

1. Security vulnerabilities
2. Race conditions and concurrency
3. Dead code
4. Anti-patterns and code smells
5. Performance
6. Correctness
7. Error handling gaps
8. Test quality

Consider both what the user explicitly asked for and what the code naturally warrants. For example, if the codebase uses async patterns, include the concurrency category even if the user didn't mention it.

For broad requests ("audit this codebase"), default to all categories and proceed without asking for confirmation. Only present the selected categories and ask for confirmation when the selection is non-obvious — when the agent has chosen a subset based on code analysis, or when the user's request is ambiguous about which categories apply.

Use the checklists in `references/categories.md` as the basis for systematic analysis. Skip individual checklist items that do not apply to the languages, frameworks, or paradigms present in the codebase.

### Step 3 — Discover Intent

Scan the codebase for documented intent — design decisions, trade-offs, conventions, known limitations — before analysis begins. This reduces false positives by providing context that distinguishes deliberate choices from genuine issues.

Run three subagents in parallel:

- **Documentation Scanner** — reads all documentation files project-wide (regardless of audit scope) and extracts stated decisions, trade-offs, constraints, and conventions.
- **Code Intent Scanner** — searches in-scope source files for rationale comments, suppression directives, and explanatory block comments that express the "why" behind code choices.
- **History Scanner** — extracts intent signals from git commit history for in-scope files, focusing on commits whose messages explain why a change was made.

This step always executes regardless of codebase size. For large codebases that use the partitioned strategy in Step 4, the same Intent Brief is shared with all partition subagents.

The output is a structured Intent Brief organized by theme (Architectural Decisions, Deliberate Trade-offs, Conventions & Standards, Known Limitations & Technical Debt, Suppressed Warnings & Intentional Deviations) — not by source type. Target no more than 100 entries, prioritized by relevance to the confirmed audit categories.

If intent discovery produces no entries, the Intent Brief is empty. Analysis proceeds normally without cross-referencing, and the report's "Context & Intent" section states that no documented intent signals were identified.

See `references/intent-discovery.md` for detailed subagent prompts, extraction rules, and the Intent Brief template.

### Step 4 — Systematic Analysis

#### Large Codebase Partitioning

When the scope exceeds either 50 files or 10,000 lines of code (whichever is reached first), use parallel subagents to avoid superficial analysis:

- Partition the scope into logical modules or directory subtrees. Aim for partitions of roughly equal size. Prefer boundaries that align with architectural layers (e.g., data access, business logic, API handlers) rather than arbitrary file count splits.
- Assign one subagent per partition. Each subagent performs Phase A (file-level analysis) on its partition independently, following the same checklist and recording format. Provide each subagent with a brief description of the overall codebase architecture, the Intent Brief from Step 3, and the applicable checklists.
- After all subagents complete, perform Phase B (cross-file analysis) on the merged set of findings, focusing on interactions between partitions. Pay special attention to trust boundaries — data flowing from one partition to another is a common source of missed validation and injection vulnerabilities.
- Deduplicate findings that were independently discovered by multiple subagents operating on shared or overlapping code. Consolidate into the highest-severity version and list all affected locations.

If subagents are not available or the scope is small enough, perform all phases sequentially as a single agent.

#### Phase A — File-Level Analysis

Iterate through every in-scope file. For each file:

- Read the file in full. Do not skim or sample.
- Walk through each checklist item from the confirmed categories, evaluating whether the code under inspection exhibits the described issue.
- Before recording a finding, cross-reference it against the Intent Brief. Skip when the brief directly and explicitly addresses the exact pattern flagged at the exact location. Downgrade (report at reduced severity with a note citing the intent source) when the brief provides general context but not per-instance acknowledgment.
- Record each finding immediately with its file path, line number, category, a preliminary severity (per `references/severity-guide.md`), a description of the problem, its impact, and a concrete recommendation.
- When a finding involves a data flow or call chain that exits the current file, mark it for cross-file follow-up in Phase B.

Avoid recording the same logical issue multiple times when it appears in multiple files due to shared patterns (e.g., a utility function used everywhere). Instead, record it once and note all affected locations.

Prioritize depth on security-critical and correctness-critical paths. For anti-patterns and dead code categories, a single-pass identification is sufficient — do not trace multi-hop call chains for Low-severity categories.

#### Phase B — Cross-File Analysis

After completing the file-level pass, revisit findings marked for cross-file follow-up:

- Trace data from entry points (HTTP handlers, CLI parsers, message consumers, public API surfaces) through intermediate layers to terminal operations (database writes, file I/O, external API calls, responses to users).
- Evaluate whether input validation, authorization, error handling, or resource cleanup is missing at any point along the traced path, even if each individual file appears correct in isolation.
- Check for architectural issues: circular dependencies between modules, inconsistent error handling strategies across layers, mixed paradigms that introduce subtle bugs, and shared mutable state accessed from multiple modules.
- Cross-reference new cross-file findings against the Intent Brief using the same skip/downgrade rules as Phase A.

Record any new findings and update severity assessments for file-level findings that turn out to be more or less severe in the broader context.

#### Deduplication

Before moving to report generation, deduplicate findings:

- Merge findings that describe the same root cause manifesting in multiple locations into a single finding. List all affected locations in that finding.
- Remove false positives identified during cross-file analysis (e.g., input that appeared unvalidated in one file but is validated by a middleware layer discovered later).
- Consolidate findings that stem from the same documented trade-off in the Intent Brief into a single finding referencing the intent entry and listing all affected locations.
- Reconcile conflicting severity assessments by considering the worst realistic impact.

### Step 5 — Report Generation

Generate the final report following the structure and formatting rules defined in `references/report-template.md`. Specifically:

- Set the report date to the current date.
- Populate the summary table with the resolved scope, confirmed categories, and finding counts by severity.
- Include a "Context & Intent" section between the Summary table and the Critical severity section. This section summarizes the Intent Brief and lists key documented decisions that influenced the audit.
- Assign sequential `AUDIT-NNN` identifiers starting at `AUDIT-001`. Order findings by severity (Critical first), then by category, then by file path within the same severity and category.
- Write each finding with all required fields: identifier, short title, category, location (`file:line`), severity, description, impact, and recommendation.
- When the audit produces zero findings, generate the full report structure with all counts at 0 and "No issues found." in each severity section.
- Save the report at the project root using the filename convention in the template (`AUDIT-REPORT-YYYY-MM-DD.md`), incrementing the suffix if a file with that name already exists.

After saving the report, state the file path and a brief summary of the results to the user.
