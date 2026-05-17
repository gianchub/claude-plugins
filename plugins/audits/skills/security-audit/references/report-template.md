# Security Audit Report Template — Markdown

Use this template when the user has chosen the **Markdown** report format. For the HTML format (the default), use `report-template-html.md` instead.

## Filename Convention

Save the report at the project root as:

```
SECURITY-AUDIT-REPORT-YYYY-MM-DD.md
```

If a file with that name already exists, append an incrementing numeric suffix:

```
SECURITY-AUDIT-REPORT-YYYY-MM-DD-2.md
SECURITY-AUDIT-REPORT-YYYY-MM-DD-3.md
```

Use the current date at the time of report generation.

---

## Template

```markdown
# Security Audit Report — YYYY-MM-DD

## Summary

| Field                       | Value |
|-----------------------------|-------|
| **Scope**                   | <files, directories, or services audited> |
| **Application kind**        | <from Threat Model Brief> |
| **Exposure**                | <from Threat Model Brief> |
| **Domains audited**         | <comma-separated list of checklist domains applied> |
| **Languages analyzed**      | <comma-separated list of languages with footgun checks applied> |
| **Total findings**          | <N> |
| **Critical**                | <N> |
| **High**                    | <N> (of which <M> Exploitability Not Confirmed) |
| **Medium**                  | <N> |
| **Low**                     | <N> |

---

## Threat Model

<Insert the Threat Model Brief from Phase 2 verbatim or summarized. Include at minimum: application kind, exposure, sensitive data classes, authentication model, authorization model, severity-modifier notes.>

---

## Documented Security Posture

<Insert the Security Intent Brief from Phase 3. Omit subsections that have no entries. If no intent signals were discovered, state: "No documented security intent was identified in the codebase. All findings are evaluated without intent-based adjustments.">

---

## Source/Sink Map (Appendix-Linked)

<One-paragraph summary of the source/sink enumeration; full map in Appendix A. State: total sources enumerated, total sinks enumerated, services covered, any sinks marked unreachable from any in-scope source.>

---

## Critical

### SEC-001 — <Short title>

| Field            | Value |
|------------------|-------|
| **Domain**       | <domain category, e.g., Authorization, Injection, Crypto> |
| **CWE**          | CWE-<id> — <CWE name> |
| **OWASP**        | A<NN>:2021 — <OWASP Top 10 title> |
| **OWASP API**    | API<N>:2023 — <OWASP API Top 10 title> (omit if N/A) |
| **Location**     | `<file>:<line>` (or list for multi-location) |
| **Severity**     | Critical (Impact: <score>, Exploitability: <score>, Exposure: <score>) |
| **Modifier**     | <if applied, name and source; otherwise omit> |

**Description**

<What the code does, why it is unsafe, and the dataflow path from source to sink in summary. Cite file:line for each step.>

**Impact**

<Concrete consequences of exploitation: data accessed, integrity violated, availability impacted. Specific to this application's data classes.>

**Exploit Scenario**

<Per the `references/exploit-scenarios.md` template: starting position, action, path, outcome. Or use the "Exploit Scenario — Not Confirmed" structure when a scenario cannot be constructed.>

**Recommendation**

<Specific, actionable fix. Include code snippets when they clarify. Mention defense-in-depth options where relevant.>

---

## High

### SEC-NNN — <Short title>

[Same structure as Critical.]

---

## Medium

### SEC-NNN — <Short title>

| Field            | Value |
|------------------|-------|
| **Domain**       | <domain> |
| **CWE**          | CWE-<id> |
| **OWASP**        | <category> (omit if not applicable) |
| **Location**     | `<file>:<line>` |
| **Severity**     | Medium (Impact: <score>, Exploitability: <score>, Exposure: <score>) |

**Description**

<...>

**Impact**

<...>

**Exploit Scenario** (optional)

<Include only if it clarifies the impact; otherwise omit.>

**Recommendation**

<...>

---

## Low

### SEC-NNN — <Short title>

| Field            | Value |
|------------------|-------|
| **Domain**       | <domain> |
| **CWE**          | CWE-<id> |
| **OWASP**        | <category> (omit if not applicable) |
| **Location**     | `<file>:<line>` |
| **Severity**     | Low (Impact: <score>, Exploitability: <score>, Exposure: <score>) |

**Description**

<...>

**Impact**

<...>

**Recommendation**

<...>

---

## Appendix A — Source/Sink Map

<Full enumeration from Phase 4. Format:

### Sources
- `<file>:<line>` — <source class> (<input shape>): <reachable sinks>

### Sinks
- `<file>:<line>` — <sink class>: <reachable sources>

For monorepos, group by service.>

---

## Appendix B — Coverage and Limitations

<State explicitly:
- Which files / paths were in scope and read in full.
- Which files / paths were excluded (generated, vendored, lock files) and why.
- Any analysis the auditor was unable to perform and the reason (e.g., "could not enumerate dependency CVEs without a populated lock file").
- Any High/Critical finding with "Exploitability Not Confirmed" and what would be needed to confirm.
- Any threat-model assumption made due to ambiguity, and how it could change findings if wrong.>

---

## Appendix C — Re-Audit Delta (only if a prior report exists)

<If a prior SECURITY-AUDIT-REPORT-*.md was found in Phase 1:

### Resolved since prior report
- <SEC-NNN-prior> (<title>) — fix verified at <file>:<line>

### Persistent from prior report
- <SEC-NNN-prior> (<title>) — still present; reassigned <SEC-NNN-current> in this report.

### New in this report
- All findings not in the prior report.

If no prior report exists, omit this appendix.>
```

---

## Identifier Numbering

Assign `SEC-NNN` identifiers sequentially starting at `SEC-001`, ordered by severity (Critical first, then High, Medium, Low). Within the same severity level, order findings by domain category, then by file path.

When a re-audit produces persistent findings from a prior report, assign new `SEC-NNN` identifiers in the current report and cross-reference the prior identifier in the Re-Audit Delta appendix. Do not reuse prior identifiers; identifiers are scoped to a single report.

---

## Multi-Location Findings

When a finding affects multiple locations (same root cause across files), use a comma-separated list in the Location field for up to 5 locations. Beyond 5, use the primary location and list the full set in the Description body:

```markdown
| **Location** | `src/api/users.py:42` (primary; full list in Description) |
```

---

## Severity Field Format

Always include the three component scores:

```
Severity: <level> (Impact: <Severe|High|Moderate|Low>, Exploitability: <Trivial|Easy|Multistep|Hard|Not Confirmed>, Exposure: <Public|Authenticated public|Authenticated restricted|Internal|Local>)
```

Note that Exploitability uses **Multistep** (not "Moderate") to keep its scale visually distinct from Impact's Moderate level.

When a Threat-Model modifier was applied, add a `Modifier:` row in the table citing the modifier name and source.

---

## Exploit Scenario Field Rules

- **Critical, High** — required. If exploitation cannot be constructed, use the "Exploit Scenario — Not Confirmed" format from `references/exploit-scenarios.md`.
- **Medium** — optional; include when it materially clarifies impact.
- **Low** — typically omitted.

---

## Zero-Findings Report

When no issues are found, produce the full report structure with all counts set to 0. Replace each severity section body with:

```markdown
No issues found at this severity level.
```

Still include Threat Model and Documented Security Posture sections; they document what was considered. Include Appendices A and B; they document scope and coverage.
