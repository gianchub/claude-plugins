# Intent Discovery for Code Audit Skill

**Date**: 2026-03-23
**Status**: Draft
**Plugin**: code-audit
**Skill**: audit

## Problem

The audit skill sometimes flags code patterns as issues when they are actually intentional design decisions, documented trade-offs, or deliberate conventions. This happens because the analysis phases have no awareness of the project's documented intent — READMEs, architecture docs, rationale comments, git history, and other sources that explain *why* things are the way they are. The result is false positives that erode trust in the audit report.

## Solution

Add a new **Step 3 — Discover Intent** to the audit workflow, positioned after scope and category confirmation but before any code analysis begins. Three parallel subagents scan the codebase for intent signals, and their findings are compiled into an **Intent Brief** — a structured document that the analysis phases use to distinguish intentional decisions from genuine issues.

## Design Decisions

1. **Position in workflow**: Between Step 2 (Select & Confirm Categories) and the current Step 3 (Systematic Analysis). Both Phase A and Phase B benefit from having the full Intent Brief from the start.

2. **Parallel subagents by source type**: Three specialized subagents run concurrently, each focused on a different source type (documentation, code comments, git history). This plays to the existing large-codebase strategy pattern and keeps each subagent focused.

3. **Discovery by pattern, not by path**: Subagents find documentation by scanning for file types, name patterns, and content indicators — not by looking for hardcoded directory names like `docs/`. Different projects organize documentation differently.

4. **Compiled Intent Brief**: Subagent findings are merged into a single themed document (organized by decision type, not source type) that is passed to all analysis subagents as context.

5. **Single discovery pass for large codebases**: Intent discovery runs once for the whole repo before partitioning. The same Intent Brief is shared with all analysis partition subagents.

6. **Intent Brief included in report**: A "Context & Intent" section in the audit report provides transparency into what was considered, placed between the Summary table and the first severity section.

## Step 3 — Discover Intent

### Overview

Scan the codebase for documented intent — the reasons behind design decisions, architectural trade-offs, known limitations, and deliberate choices. Produce an Intent Brief that the analysis phases use to reduce false positives.

### Three Discovery Subagents

All three run in parallel after scope and categories are confirmed.

#### Subagent A — Documentation Scanner

Discovers and reads documentation files across the audit scope and repo root. Rather than looking for hardcoded paths, it:

1. Scans the repo for all markdown (`.md`), text (`.txt`), reStructuredText (`.rst`), and AsciiDoc (`.adoc`) files.
2. Identifies files whose names or content suggest documentation: anything containing words like "readme", "architecture", "design", "decision", "adr", "contributing", "changelog", "history", "conventions", "standards", "guide", "rationale".
3. Checks for agent instruction files: `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `CURSORRULES`, `.windsurfrules`, and similar.
4. Reads each discovered file and extracts intent signals: stated design decisions, trade-offs, constraints, conventions, deliberate limitations.

Reports a list of intent entries, each with: the source file, a summary of the decision/convention, and a direct quote where useful.

#### Subagent B — Code Intent Scanner

Searches in-scope source files for comments that express rationale:

- **Rationale markers**: `NOTE`, `HACK`, `WHY`, `DESIGN`, `TRADE-OFF`, `INTENTIONAL`, `DELIBERATE`, `BY DESIGN`, `RATIONALE`, `REASON`.
- **Suppression markers**: `nolint`, `noqa`, `@SuppressWarnings`, `eslint-disable`, `type: ignore` — these signal intentional deviations.
- **Explanatory block comments**: Comments longer than one line containing words like "because", "trade-off", "instead of", "chosen", "deliberately", "intentionally" — comments that explain "why" rather than "what".
- **Config file comments**: Rationale in `.env.example`, `Dockerfile`, CI configs, `Makefile`, etc.

Reports each finding with: file path, line number, comment text, and a brief interpretation of the intent.

#### Subagent C — History Scanner

Examines git history for in-scope files to surface design-shaping decisions:

1. Runs `git log` on in-scope files to extract commit messages.
2. Filters for commits with substantive messages that explain *why* — excluding noise like "fix typo", "bump version".
3. Identifies large refactors, architectural changes, or deliberate removals (commits that delete code with an explanation).
4. Checks merge/PR commit messages, which often contain richer context.

Reports significant history entries with: commit hash (short), date, message summary, and affected files.

### The Intent Brief

After all three subagents complete, their findings are merged into a single Intent Brief organized by theme:

```markdown
## Intent Brief

### Architectural Decisions
- <decision summary> (source: <file or commit>)

### Deliberate Trade-offs
- <trade-off summary> (source: <file or commit>)

### Conventions & Standards
- <convention summary> (source: <file or commit>)

### Known Limitations & Technical Debt
- <limitation summary> (source: <file or commit>)

### Suppressed Warnings & Intentional Deviations
- <deviation summary> (source: <file or commit>)
```

Empty sections are omitted.

### How the Intent Brief Is Used

- **Passed as context** to all analysis subagents (Phase A and Phase B) alongside the category checklists.
- **During analysis**: Before recording a finding, the agent checks the Intent Brief for relevant entries. If a potential finding matches a documented intentional decision, it is either skipped entirely or downgraded with a note referencing the source.
- **During deduplication**: The brief serves as an additional filter — findings that conflict with documented intent are flagged for review rather than reported as issues.

## Changes to Existing Steps

### Step Renumbering

| Current | New | Name |
|---------|-----|------|
| Step 1 | Step 1 | Resolve Scope |
| Step 2 | Step 2 | Select & Confirm Categories |
| — | **Step 3** | **Discover Intent** (new) |
| Step 3 | Step 4 | Systematic Analysis |
| Step 4 | Step 5 | Large Codebase Strategy |
| Step 5 | Step 6 | Report Generation |

### Step 4 — Systematic Analysis (modifications)

- **Phase A**: Before recording a finding, cross-reference against the Intent Brief. If a potential issue matches a documented decision, skip it or note it as "acknowledged — see Intent Brief".
- **Phase B**: Same cross-referencing during cross-file analysis.
- **Deduplication**: Add intent-based filtering as a deduplication criterion alongside same-root-cause merging.

### Step 5 — Large Codebase Strategy (modifications)

- Intent discovery runs once before partitioning.
- The Intent Brief is passed to each partition's analysis subagent alongside the architecture description and checklists.

### Step 6 — Report Generation (modifications)

- New "Context & Intent" section added to the report, between the Summary table and the Critical severity section.
- `references/report-template.md` updated with the new section template.

## New Files

### `references/intent-discovery.md`

Detailed subagent prompts for all three discovery subagents and the Intent Brief template. Keeps SKILL.md lean by moving the detailed instructions out of the main workflow.

### Report Template Addition

New section in `references/report-template.md`:

```markdown
## Context & Intent

The following documented decisions, trade-offs, and conventions were identified during the intent discovery phase and taken into account during analysis:

### Architectural Decisions
- <decision summary> (source: <file or commit>)

### Deliberate Trade-offs
- <trade-off summary> (source: <file or commit>)

### Conventions & Standards
- <convention summary> (source: <file or commit>)

### Known Limitations & Technical Debt
- <limitation summary> (source: <file or commit>)

### Suppressed Warnings & Intentional Deviations
- <deviation summary> (source: <file or commit>)
```

Empty sections are omitted from the report. If no intent signals are discovered, the section reads: "No documented intent signals were identified in the codebase."

## Acceptance Criteria

1. The audit skill SKILL.md contains a Step 3 — Discover Intent between category confirmation and systematic analysis.
2. Three subagent types are defined: Documentation Scanner, Code Intent Scanner, History Scanner.
3. Subagents discover documentation by pattern/content, not by hardcoded directory paths.
4. Subagent findings are merged into a themed Intent Brief.
5. The Intent Brief is passed as context to all analysis subagents (Phase A, Phase B, and large-codebase partitions).
6. Analysis phases cross-reference findings against the Intent Brief before recording.
7. The report template includes a "Context & Intent" section between Summary and Critical.
8. A new `references/intent-discovery.md` file contains the detailed subagent prompts and Intent Brief template.
9. For large codebases, intent discovery runs once before partitioning; the brief is shared with all partition subagents.
10. The SKILL.md remains lean — detailed subagent instructions live in the reference file.
