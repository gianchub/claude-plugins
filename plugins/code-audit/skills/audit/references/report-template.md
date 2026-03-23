# Audit Report Template

## Filename Convention

Save the report at the project root as:

```
AUDIT-REPORT-YYYY-MM-DD.md
```

If a file with that name already exists, append an incrementing numeric suffix:

```
AUDIT-REPORT-YYYY-MM-DD-2.md
AUDIT-REPORT-YYYY-MM-DD-3.md
```

Use the current date at the time of report generation.

---

## Template

```markdown
# Audit Report — YYYY-MM-DD

## Summary

| Field               | Value                        |
|---------------------|------------------------------|
| **Scope**           | <files, directories, or modules audited> |
| **Categories**      | <comma-separated list of audit categories applied> |
| **Total findings**  | <N>                          |
| **Critical**        | <N>                          |
| **High**            | <N>                          |
| **Medium**          | <N>                          |
| **Low**             | <N>                          |

---

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

---

## Critical

### AUDIT-001 — <Short title>

| Field            | Value |
|------------------|-------|
| **Category**     | <category name> |
| **Location**     | `<file>:<line>` |
| **Severity**     | Critical |

**Description**

<Clear explanation of the finding. State what the code does and why it is problematic.>

**Impact**

<Concrete consequences: data loss, unauthorized access, service disruption, etc.>

**Recommendation**

<Specific, actionable fix. Include a code snippet when it clarifies the recommendation.>

---

## High

### AUDIT-NNN — <Short title>

| Field            | Value |
|------------------|-------|
| **Category**     | <category name> |
| **Location**     | `<file>:<line>` |
| **Severity**     | High |

**Description**

<...>

**Impact**

<...>

**Recommendation**

<...>

---

## Medium

### AUDIT-NNN — <Short title>

| Field            | Value |
|------------------|-------|
| **Category**     | <category name> |
| **Location**     | `<file>:<line>` |
| **Severity**     | Medium |

**Description**

<...>

**Impact**

<...>

**Recommendation**

<...>

---

## Low

### AUDIT-NNN — <Short title>

| Field            | Value |
|------------------|-------|
| **Category**     | <category name> |
| **Location**     | `<file>:<line>` |
| **Severity**     | Low |

**Description**

<...>

**Impact**

<...>

**Recommendation**

<...>
```

---

## Identifier Numbering

Assign `AUDIT-NNN` identifiers sequentially starting at `AUDIT-001`, ordered by severity (Critical first, then High, Medium, Low). Within the same severity level, order findings by category, then by file path.

---

## Zero-Findings Report

When no issues are found, produce the full report structure with all counts set to 0. Replace each severity section body with:

```markdown
No issues found.
```

Example:

```markdown
## Summary

| Field               | Value                        |
|---------------------|------------------------------|
| **Scope**           | src/                         |
| **Categories**      | Security, Correctness, Error handling |
| **Total findings**  | 0                            |
| **Critical**        | 0                            |
| **High**            | 0                            |
| **Medium**          | 0                            |
| **Low**             | 0                            |

## Context & Intent

No documented intent signals were identified in the codebase.

## Critical

No issues found.

## High

No issues found.

## Medium

No issues found.

## Low

No issues found.
```

---

## Context & Intent Section

Empty subsections within Context & Intent are omitted from the report. If no intent signals were discovered, the entire section reads: "No documented intent signals were identified in the codebase."
