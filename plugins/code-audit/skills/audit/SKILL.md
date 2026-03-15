---
name: audit
description: >
  This skill should be used when the user asks to "audit this codebase",
  "audit this code", "security audit", "code audit", "find vulnerabilities",
  "check for bugs", "review code quality", "find dead code",
  "check for anti-patterns", "performance audit", or mentions wanting a
  comprehensive code quality analysis. Produces a structured severity-ranked
  report file.
---

# Code Audit Skill

## Purpose

Perform a thorough, language-agnostic audit of a codebase or subset of files, producing a structured report with findings ranked by severity. Cover security vulnerabilities, race conditions, dead code, anti-patterns, performance issues, correctness bugs, and error handling gaps. Deliver the report as a Markdown file at the project root following the template in `references/report-template.md`.

## Effort Level

Apply maximum effort throughout the audit. Read every line of in-scope code. Trace data flows from external inputs through processing layers to outputs and storage. Follow call chains across module boundaries to detect issues that only manifest through component interaction. Examine configuration files, build scripts, and infrastructure definitions alongside application code. When the scope is large enough that a single pass would be superficial, split the work across parallel subagents by module or directory, then merge and deduplicate findings before report generation.

## Workflow

### Step 1 — Resolve Scope

Determine the audit scope from the user's prompt. The user may specify:

- An entire repository or working directory.
- One or more directories (e.g., `src/`, `lib/api/`).
- A list of specific files.
- A functional area described in natural language (e.g., "the authentication flow").

If the prompt does not contain enough information to determine scope, ask a single clarifying question before proceeding. Do not guess at scope — an audit with unclear boundaries produces unreliable results.

Once scope is established, enumerate all files that fall within it. Exclude generated files (e.g., lock files, compiled output, vendored dependencies, minified bundles) unless the user explicitly includes them. State the resolved scope back to the user before continuing.

### Step 2 — Confirm Categories

Present the seven audit categories to the user:

1. Security vulnerabilities
2. Race conditions and concurrency
3. Dead code
4. Anti-patterns and code smells
5. Performance
6. Correctness
7. Error handling gaps

Allow the user to remove categories that are not relevant or add custom categories. If the user does not respond or confirms the defaults, proceed with all seven. Record the final category list for inclusion in the report summary.

Consult `references/categories.md` for the detailed checklist within each category. Use those checklists as the basis for systematic analysis. Skip individual checklist items that do not apply to the languages, frameworks, or paradigms present in the codebase.

### Step 3 — Systematic Analysis

Execute the audit in two phases: file-level analysis and cross-file analysis.

#### Phase A — File-Level Analysis

Iterate through every in-scope file. For each file:

- Read the file in full. Do not skim or sample.
- Walk through each checklist item from the confirmed categories, evaluating whether the code under inspection exhibits the described issue.
- Record each finding immediately with its file path, line number, category, a preliminary severity, a description of the problem, its impact, and a concrete recommendation.
- When a finding involves a data flow or call chain that exits the current file, mark it for cross-file follow-up in Phase B.

Avoid recording the same logical issue multiple times when it appears in multiple files due to shared patterns (e.g., a utility function used everywhere). Instead, record it once and note all affected locations.

#### Phase B — Cross-File Analysis

After completing the file-level pass, revisit findings marked for cross-file follow-up:

- Trace data from entry points (HTTP handlers, CLI parsers, message consumers, public API surfaces) through intermediate layers to terminal operations (database writes, file I/O, external API calls, responses to users).
- Evaluate whether input validation, authorization, error handling, or resource cleanup is missing at any point along the traced path, even if each individual file appears correct in isolation.
- Check for architectural issues: circular dependencies between modules, inconsistent error handling strategies across layers, mixed paradigms that introduce subtle bugs, and shared mutable state accessed from multiple modules.

Record any new findings and update severity assessments for file-level findings that turn out to be more or less severe in the broader context.

#### Deduplication

Before moving to report generation, deduplicate findings:

- Merge findings that describe the same root cause manifesting in multiple locations into a single finding. List all affected locations in that finding.
- Remove false positives identified during cross-file analysis (e.g., input that appeared unvalidated in one file but is validated by a middleware layer discovered later).
- Reconcile conflicting severity assessments by considering the worst realistic impact.

### Step 4 — Large Codebase Strategy

When the scope contains more than roughly 50 files or 10,000 lines of code, use parallel subagents to avoid superficial analysis:

- Partition the scope into logical modules or directory subtrees. Aim for partitions of roughly equal size.
- Assign one subagent per partition. Each subagent performs Phase A (file-level analysis) on its partition independently, following the same checklist and recording format.
- After all subagents complete, perform Phase B (cross-file analysis) on the merged set of findings, focusing on interactions between partitions.
- Deduplicate findings that were independently discovered by multiple subagents operating on shared or overlapping code.

If subagents are not available or the scope is small enough, perform all phases sequentially as a single agent.

### Step 5 — Report Generation

Generate the final report following the structure and formatting rules defined in `references/report-template.md`. Specifically:

- Set the report date to the current date.
- Populate the summary table with the resolved scope, confirmed categories, and finding counts by severity.
- Assign sequential `AUDIT-NNN` identifiers starting at `AUDIT-001`. Order findings by severity (Critical first), then by category, then by file path within the same severity and category.
- Write each finding with all required fields: identifier, short title, category, location (`file:line`), severity, description, impact, and recommendation.
- When the audit produces zero findings, generate the full report structure with all counts at 0 and "No issues found." in each severity section, as shown in the template.
- Save the report at the project root using the filename convention in the template (`AUDIT-REPORT-YYYY-MM-DD.md`), incrementing the suffix if a file with that name already exists.

After saving the report, state the file path and a brief summary of the results to the user.

## Severity Classification Guide

Use the following criteria to assign severity levels. When a finding could fit multiple levels, choose the higher severity and note the reasoning.

### Critical

Reserve for findings that represent an immediate, exploitable threat or a near-certain path to significant damage:

- Remotely exploitable security vulnerabilities (injection, auth bypass, SSRF with internal network access).
- Direct paths to data loss or corruption (unprotected destructive operations, missing transaction safety on critical writes).
- Hardcoded production credentials or secrets committed to version control.
- Privilege escalation that grants administrative access to unauthorized users.

### High

Assign to findings that are likely to cause real-world bugs, outages, or security degradation under normal operating conditions:

- Correctness bugs that produce wrong results or crash the application for common inputs.
- Race conditions on data structures or resources accessed in production paths.
- Missing authorization checks on sensitive but non-critical endpoints.
- Error handling gaps that cause cascading failures (e.g., unhandled exceptions in request middleware that crash the entire process).

### Medium

Assign to findings that degrade quality, maintainability, or performance but are unlikely to cause immediate failures:

- Performance issues that cause slowdowns under realistic load (N+1 queries, blocking in async contexts, O(n^2) algorithms on growing datasets).
- Anti-patterns that make the code significantly harder to maintain or extend (god objects, deep nesting, SRP violations).
- Missing input validation on internal APIs where the blast radius is limited.
- Insecure defaults that are partially mitigated by other layers (e.g., missing CORS headers behind an API gateway that enforces its own).

### Low

Assign to findings that represent minor quality issues with limited practical impact:

- Dead code (unused imports, unreachable branches, orphaned tests) that adds noise but does not affect runtime behavior.
- Minor code smells (magic numbers in non-critical paths, slightly duplicated code blocks).
- Commented-out code that should be cleaned up.
- Missing pagination on endpoints with currently small datasets but potential future growth.

## Additional Resources

Refer to the following reference files during the audit:

- **`references/categories.md`** — Detailed per-category checklists. Use these as the systematic basis for file-level analysis in Step 3, Phase A. Skip items that do not apply to the languages and frameworks in scope.
- **`references/report-template.md`** — Report structure, filename conventions, identifier numbering, and zero-findings format. Follow this template exactly when generating the output report in Step 5.
