---
name: security-audit
description: >
  This skill should be used when the user asks for a security-focused review
  of a codebase. Triggers: "security audit", "find vulnerabilities",
  "check for security issues", "pentest this code", "security review",
  "secure code review", "vulnerability assessment", "threat model this code",
  "OWASP audit", "find injection vulnerabilities", "check authentication",
  "check authorization", "find secrets", "find hardcoded credentials",
  "check for XSS", "check for CSRF", "check for SSRF", "find IDOR",
  "find IDOR / BOLA", "check for SSTI", "review JWT handling", "review
  crypto", "find deserialization issues", "check for path traversal",
  "review for prototype pollution", "audit dependencies for CVEs", "supply
  chain audit", "review Dockerfile security", "review CI/CD security".
---

# Security Audit Skill

## Purpose

Perform a thorough, threat-model-first security audit of a codebase, producing a structured report with findings ranked by Impact x Exploitability x Exposure, mapped to CWE and OWASP categories, and (for High and Critical) supported by a concrete exploit scenario. Deliver the report as a self-contained file at the project root in the format the user chose (see "Report Format" below): HTML following `references/report-template-html.md`, or Markdown following `references/report-template.md`.

This skill is exclusively focused on security. For general code-quality auditing (concurrency, dead code, anti-patterns, performance, correctness, error handling, tests), use the sibling `code-audit` skill in the same plugin. The two skills can run independently or in sequence on the same codebase. If a non-security code-quality issue is encountered incidentally during this audit, note it briefly and recommend running the `code-audit` skill rather than analyzing it in depth here.

## Effort Level

Read every line of in-scope code. Do not skim, sample, or rely on heuristics to skip files. The defining technique of this audit is **source-to-sink dataflow** — trace untrusted input from every entry point through the call graph to every dangerous sink, and at each step evaluate validation, encoding, authorization, and trust assumptions. Pattern matching alone misses real vulnerabilities; dataflow tracing finds them. When the scope is too large for a single pass, split work across parallel subagents (see Phase 5 for partitioning strategy).

## Mindset

Audit the code as a pentester would: assume any untrusted input is hostile, assume any check can be bypassed unless proven otherwise, assume any boundary can be crossed unless validated. Defense-in-depth matters — when one layer fails, what's the next layer? Look for what is *missing* (a check that should exist), not only what is *present* (a check that's wrong). Most real-world breaches come from missing controls, mass-assignment-style overposting, broken object-level authorization, and trust placed in user-controlled fields, not from exotic exploits.

## Report Format

The audit produces a self-contained file at the project root. The format determines the file extension and the template used in Phase 6. **HTML is the default** because it enables inline SVG trust-boundary and dataflow diagrams, severity / CWE / OWASP tag styling, collapsible Source/Sink Map entries per service, and anchor cross-references between related findings — affordances that materially improve a long security report and cannot be expressed in Markdown.

<FORMAT-GATE>
**Determine the format before resolving scope.** This is a hard gate — do not start scope resolution, threat modeling, intent discovery, source/sink mapping, analysis, or any other work until the format is fixed for this run.

First, scan the user's launching message for an explicit format choice. Honor any of these:

- HTML signals: `HTML`, `html`, "as a webpage", "browser-viewable", "rendered report".
- Markdown signals: `MD`, `md`, `Markdown`, `markdown`, "plain text report", "as Markdown".

If the launching message contains a clear format signal, accept it as the gate answer and proceed — do not re-ask for ceremonial confirmation. State the chosen format briefly when you state the resolved scope.

If the launching message does not specify a format, ask exactly this question (or equivalent) and wait for the user's answer before proceeding:

> Which report format would you like? **HTML** (default, self-contained file with inline trust-boundary diagrams, CWE/OWASP tags, and severity badges) or **MD** (plain Markdown)?

Do not bundle this question with any other output. Do not assume a default without asking. The gate fires every time this skill runs.
</FORMAT-GATE>

Once the format is chosen, use it consistently for the remainder of the run:

| Format | Template | Filename |
|---|---|---|
| HTML | `references/report-template-html.md` | `SECURITY-AUDIT-REPORT-YYYY-MM-DD.html` |
| MD | `references/report-template.md` | `SECURITY-AUDIT-REPORT-YYYY-MM-DD.md` |

When generating an HTML report, treat the HTML-specific affordances documented in `references/report-template-html.md` as tools to use when they add information — not decoration. Wrapping Markdown content in HTML tags without using those affordances defeats the point of choosing HTML.

## Workflow

The workflow has six phases. Phase 1 must precede everything else; Phases 2 and 3 may run in parallel (they have no data dependency); Phase 4 depends on Phase 2; Phases 5 and 6 depend on all earlier phases. Parallelism within phases is encouraged where useful — Phase 3 always uses subagents (one per scanner); Phase 5 uses subagents to partition large scopes.

### Phase 1 — Resolve Scope

Determine the audit scope from the user's prompt. The user may specify:

- An entire repository or working directory.
- One or more directories (e.g., `src/`, `services/api/`).
- A list of specific files.
- A functional area described in natural language (e.g., "the authentication flow", "the file-upload endpoint").
- A specific function, class, or code region within a file. Restrict analysis to the specified code and its immediate dependencies.

Generated files (lock files, compiled output, vendored dependencies, minified bundles) are excluded by default. For source-committed generated code (protobuf stubs, OpenAPI clients, ORM models), check for modification markers — if files contain hand-written additions or "DO NOT EDIT" headers have been removed, treat them as in-scope. Phase 5 Phase B (cross-trust-boundary dataflow) still traces flows *into* excluded code to determine whether it provides expected validation.

If the user specifies paths that do not exist, warn about missing paths and proceed with the rest. If none exist, stop and ask for clarification.

If the prompt does not contain enough information to determine scope, ask a single clarifying question before proceeding. An audit with unclear boundaries produces unreliable results.

If the user requests a re-audit, search for `SECURITY-AUDIT-REPORT-*.md` and `SECURITY-AUDIT-REPORT-*.html` at the project root. If multiple reports exist, use the most recent by file modification time regardless of format. Default to the same scope as the prior audit. When a prior report exists, note in the new report which previous findings have been resolved and which persist, so the user gets a delta view. The new report's format follows the gate decision for the current run; it does not have to match the prior report's format.

State the resolved scope to the user and proceed immediately; only stop for confirmation if scope is genuinely ambiguous.

### Phase 2 — Threat-Model the Application

Before reading the code in depth, build a Threat Model Brief that drives the rest of the audit. Without this step, the audit becomes an undifferentiated checklist scan; with it, the audit prioritizes the threats that actually apply to this application.

The Threat Model Brief identifies:

- **Application kind** — Web API, server-rendered web app, single-page app, CLI, library, mobile backend, desktop app, embedded, IoT, browser extension.
- **Exposure** — Internet-facing (anonymous traffic), authenticated public, internal-only (corporate network), localhost-only, single-tenant, multi-tenant (mutually trusted vs. mutually hostile).
- **Sensitive data classes** — PII, payment data, authentication credentials, session tokens, health records, financial records, IP/source code, customer-uploaded files.
- **Trust boundaries** — Where untrusted data enters, where privilege transitions occur (anonymous-to-authenticated, user-to-admin, tenant-to-tenant), where data crosses processes or services.
- **Authentication model** — Form login + session cookie, JWT bearer, OAuth/OIDC, mTLS, API keys, signed URLs, none. Determines which sections of `references/checklists/auth-and-session.md` apply.
- **Authorization model** — RBAC, ABAC, ownership-based, tenancy-scoped, none. Determines which authorization checks the audit must verify.
- **External dependencies and integrations** — Databases, caches, queues, third-party APIs, payment processors, identity providers; for each, the trust assumption.
- **Deployment context** — Containers, serverless, VMs, on-prem; cloud provider; CI/CD platform. Drives the IaC and CI/CD checklists.
- **Applicable checklists** — The subset of `references/checklists/*.md` to load in Phase 5, derived from the above signals (e.g., a CLI tool skips XSS/CSRF; a public API includes API security).
- **Severity-modifier notes** — Application-specific factors that should raise or lower default severities (e.g., PHI handling, hostile multi-tenancy, behind-WAF mitigations).

Detect this from `README`, `package.json` / `pyproject.toml` / `go.mod`, `Dockerfile`, `docker-compose.yml`, framework signals (Django settings, Express middleware, Spring annotations, Rails routes), entry-point files, and the directory structure. Ask a single clarifying question only if the application kind or exposure is genuinely ambiguous from the code; otherwise infer and document the inference so the report consumer can correct it.

The Threat Model Brief is a short structured document (typically 200–400 words for a single service). For monorepos with multiple distinct services or application kinds, produce one brief per service or one combined brief that names each kind's threats explicitly; the per-service form scales better. The brief is regenerated on each audit run; never persist it across runs.

See `references/threat-modeling.md` for the full template, the inference heuristics, and worked examples for common application kinds.

### Phase 3 — Discover Security Intent

Scan for documented security decisions, threat models, and acknowledged limitations before analysis begins. This reduces false positives by distinguishing deliberate trade-offs from genuine issues.

Run three subagents in parallel:

- **Documentation Scanner** — reads project-wide documentation, with extra weight on security-specific files (`SECURITY.md`, `THREAT_MODEL.md`, `.well-known/security.txt`, `docs/security/`, ADRs, compliance documents, pentest reports). Full file-discovery list is in `references/intent-discovery.md`.
- **Code Intent Scanner** — searches in-scope source for security suppression markers (across SAST tools, secret scanners, and language linters) and rationale comments containing security keywords. Full marker and keyword lists are in `references/intent-discovery.md`.
- **History Scanner** — extracts intent signals from git history, prioritizing commits whose messages mention security keywords (CVE, vulnerability, auth, escalation, disclosure, harden, bypass, exploit, pentest, etc.) and commits that touched security-sensitive files. Full keyword list and pattern guidance are in `references/intent-discovery.md`.

This step always executes regardless of codebase size. The output is a structured Security Intent Brief organized by theme (Documented Threat Model, Acknowledged Risks & Trade-offs, Security Conventions, Suppressed Findings & Rationale, Historical Security Fixes). Target no more than 100 entries.

If intent discovery produces no entries, the brief is empty. Analysis proceeds normally; the report's "Documented Security Posture" section states that no documented intent signals were identified.

See `references/intent-discovery.md` for detailed subagent prompts and the brief template.

### Phase 4 — Build the Source/Sink Map

Before flagging vulnerabilities, enumerate the application's untrusted-input sources and dangerous sinks. The map drives the rest of the audit: every High/Critical injection-class finding traces back to a documented source and a documented sink.

**Sources** include HTTP entry points (body, query, path, headers, cookies, multipart files), authentication artifacts treated as input (JWT/OAuth/SSO claims after decode), CLI inputs (argv, stdin, env), message/event inputs (queue consumers, webhook receivers, scheduled-job payloads), responses from external calls treated as trusted, and storage reads of previously-untrusted data ("second-order" sources).

**Sinks** include database query construction, shell and process execution, code execution (`eval`/`Function`/dynamic `import`), deserialization, filesystem operations, HTTP egress (SSRF surface), template rendering, HTTP response writing (HTML / redirect / `Set-Cookie` / custom headers), logging (sensitive-data leak and log injection), cryptographic operations, and authentication/authorization decision points.

The map does not need to be exhaustive prose; a list of `source@file:line` and `sink@file:line` entries with the input shape and the operation suffices. For monorepos, build the map per service.

See `references/source-sink-mapping.md` for the full enumeration of source classes and sink categories, framework-specific patterns (Express, Flask, FastAPI, Django, Rails, Spring, Gin, ASP.NET, etc.), heuristics for building the map, and worked examples.

### Phase 5 — Systematic Analysis

#### Large Codebase Partitioning

When scope exceeds 50 files or 10,000 lines of code (either threshold alone is sufficient), use parallel subagents to avoid superficial analysis. The thresholds are intentionally disjunctive.

- Partition by service first in monorepos; within a service, prefer architectural-layer boundaries (data access, business logic, API handlers, auth middleware).
- Each subagent receives the Threat Model Brief, the Security Intent Brief, the Source/Sink Map, and the applicable checklists for its partition.
- Each subagent performs Phase A (file-level analysis) on its partition independently, following the same checklists.
- After all subagents complete, perform Phase B (cross-trust-boundary dataflow) on the merged set, paying special attention to data crossing partition boundaries — this is where missed authorization checks and tainted-data injection commonly hide.
- Deduplicate findings from overlapping discoveries; consolidate to highest severity and list all locations.

If subagents are unavailable or scope is small, perform all phases sequentially. When even partitioned analysis cannot cover every line, prioritize entry points, authentication and authorization code, sinks identified in Phase 4, secrets handling, and any code referenced in the Security Intent Brief's "Suppressed Findings" section.

#### Phase A — File-Level Analysis

Iterate through every in-scope file. For each file:

- Read the file in full.
- Walk through each applicable checklist from `references/checklists/`. The Threat Model Brief's "Applicable checklists" entry identifies which to load: authn/authz/sessions for any auth code, injection for any sink-adjacent code, crypto for any code that calls crypto APIs, file handling for upload/download endpoints, error/logging across the board, etc. Skip checklists that don't apply to the application kind (e.g., XSS/CSRF on a CLI tool).
- Load language-footgun references (`references/language-footguns/<lang>.md`) for each language present in the partition. Apply language-specific checks alongside the domain checklists.
- Cross-reference candidate findings against the Security Intent Brief. Skip when the brief explicitly addresses the exact pattern at the exact location. Downgrade by one severity level when the brief provides general context but not per-instance acknowledgment, and cite the intent source.
- Record each finding with file path, line number, domain category, CWE ID(s), OWASP mapping (Top 10 and/or API Top 10), preliminary severity, the weakness, the impact, and a concrete recommendation. Mark findings that involve cross-file flows for Phase B follow-up.

Avoid recording the same logical issue multiple times when it manifests in many files due to a shared utility function. Record it once at the root cause and list all affected locations.

#### Phase B — Cross-Trust-Boundary Dataflow

After file-level analysis, walk every source-sink pair from the Phase 4 map:

- For each source, trace its dataflow through validators, sanitizers, encoders, ORM layers, framework abstractions, and intermediate functions to every sink it can reach. Where the path is long, follow the call graph rather than guessing.
- At each step, evaluate: is the data validated for type/length/format/range? Is it sanitized for the eventual sink (HTML-encoded, SQL-parameterized, shell-quoted, path-canonicalized)? Are authorization checks present at the right boundary (typically before any state change, not after)? Is the trust transition explicit (e.g., a function named `sanitize`, `escape`, `parameterize`) or implicit (assumed)?
- Pay particular attention to **second-order taint**: data read from the database that was originally user-controlled (stored XSS, second-order SQL injection, log injection from stored content).
- Pay particular attention to **trust laundering**: a check on the source value but use of a derived value that bypasses the check (e.g., normalize-then-check vs. check-then-normalize, header parsing inconsistencies).
- Check for **authorization on every protected operation**, not just on the canonical endpoint — sibling endpoints, batch endpoints, GraphQL field resolvers, and admin tools commonly miss the same check.
- Apply business-logic checks (`references/checklists/business-logic.md`): state-machine bypasses, race conditions on auth, double-spend, insufficient atomicity on financial operations.
- Check for **defense-in-depth gaps**: a single check protecting a critical operation, with no second layer if it fails.

Record new findings; update severity for file-level findings whose impact changes when seen in cross-file context.

#### Exploit Scenario Construction

For every finding tentatively scored High or Critical, attempt to construct a concrete exploit scenario answering: starting position (who is the attacker), action (what concrete request/input/sequence), path (file:line steps from source to outcome), and outcome (specific data accessed, command executed, privilege gained).

If the scenario can be constructed, include it under "Exploit Scenario." If the underlying weakness is clear but a scenario cannot be constructed (defense-in-depth gap, second-layer mitigation, ambiguous trust boundary, missing precondition the auditor cannot fabricate), keep the finding at its assessed severity and use the "Exploit Scenario — Not Confirmed" structure. **Do not silently downgrade or drop a High/Critical solely because the scenario is hard to construct.** For Medium and Low findings, an exploit scenario is optional.

See `references/exploit-scenarios.md` for the full 4-question rule, scenario templates by weakness class, the "Not Confirmed" template, and anti-patterns to avoid.

#### Deduplication

Before report generation:

- Merge findings with the same root cause manifesting in multiple locations into a single finding listing all affected locations.
- Remove false positives discovered during Phase B (e.g., input that appeared unvalidated in one file but is validated by middleware discovered later — verify the middleware actually applies to the route in question).
- Consolidate findings derived from the same documented trade-off in the Security Intent Brief into a single finding referencing the intent entry.
- Reconcile conflicting severity assessments by considering worst realistic Impact × Exploitability × Exposure. See `references/severity-guide.md` for the scoring matrix, threat-model modifiers, and worked examples.

### Phase 6 — Report Generation

Generate the final report following the template that matches the format chosen in the Format Gate: `references/report-template-html.md` for HTML output, `references/report-template.md` for Markdown output. The content rules below apply to both formats; the templates define the structural rendering.

- Set the report date to the current date.
- Populate the summary table: scope, application kind and exposure (from Threat Model Brief), domains audited, finding counts by severity.
- Include "Threat Model" and "Documented Security Posture" sections summarizing Phase 2 and Phase 3 outputs.
- Include a "Source/Sink Map" appendix summarizing the Phase 4 enumeration so the reader can see what was analyzed.
- Assign sequential `SEC-NNN` identifiers starting at `SEC-001`. Order findings by severity (Critical first), then by domain category in the order listed in the Reference Index below, then by file path within the same severity and category.
- Each finding contains: identifier, short title, domain category, CWE ID(s), OWASP Top 10 / API Top 10 mapping, location (`file:line`), severity (with the Impact/Exploitability/Exposure breakdown), description, impact, exploit scenario (or "Not Confirmed" explanation for High/Critical), and recommendation. Use `references/owasp-cwe-mapping.md` for CWE/OWASP lookup; pick the most specific CWE that fits.
- Save the report at the project root following the filename convention from the chosen template (`SECURITY-AUDIT-REPORT-YYYY-MM-DD.html` for HTML, `SECURITY-AUDIT-REPORT-YYYY-MM-DD.md` for Markdown), incrementing the numeric suffix if the file already exists.

When the chosen format is HTML, additionally:

- Follow the structural contract in `references/report-template-html.md` exactly: required IDs, classes, and `data-*` attributes are not optional. The scaffold is what makes the report parseable and the re-audit delta possible.
- Use HTML-specific affordances when they add information: inline SVG trust-boundary diagrams in the Threat Model section for multi-service or multi-tenant applications; SVG source-to-sink dataflow diagrams for High/Critical injection findings whose path spans many files; collapsible Security Intent Brief themes; anchor cross-references between related findings (shared root cause, chained exploit prerequisites). Skip them when prose already conveys the relationship cleanly.

After saving, state the file path and a brief summary of the results to the user, including the number of Critical findings (if any), the application kind and exposure used in scoring, and any High findings whose exploitability was not confirmed.

## Reference Index

Load these references when their phase is active. The SKILL.md body intentionally does not duplicate their content.

**Workflow references** (load when entering the relevant phase):

- `references/threat-modeling.md` — Phase 2 brief template and inference heuristics.
- `references/intent-discovery.md` — Phase 3 subagent prompts and brief template.
- `references/source-sink-mapping.md` — Phase 4 enumeration heuristics.
- `references/exploit-scenarios.md` — Phase 5 exploit-scenario construction.
- `references/severity-guide.md` — Impact x Exploitability x Exposure scoring.
- `references/owasp-cwe-mapping.md` — quick lookup for OWASP and CWE tags.
- `references/report-template.md` — Phase 6 report format when the user chose Markdown.
- `references/report-template-html.md` — Phase 6 report format when the user chose HTML (default).

**Domain checklists** (load only those flagged applicable by the Threat Model Brief). Grouped to aid selection and to define the canonical domain-category sort order for the report:

*Identity & access:*

- `references/checklists/auth-and-session.md`
- `references/checklists/authorization.md`
- `references/checklists/secrets-and-keys.md`

*Input handling & injection:*

- `references/checklists/injection.md`
- `references/checklists/xss-csrf-frontend.md`
- `references/checklists/ssrf-redirect-url.md`
- `references/checklists/deserialization.md`
- `references/checklists/file-handling.md`

*Cross-cutting concerns:*

- `references/checklists/crypto.md`
- `references/checklists/error-and-logging.md`
- `references/checklists/business-logic.md`
- `references/checklists/api-security.md`

*Infrastructure & supply chain:*

- `references/checklists/dependencies.md`
- `references/checklists/containers-iac.md`
- `references/checklists/cicd.md`

**Language footguns** (load one per language detected in scope):

- `references/language-footguns/python.md`
- `references/language-footguns/javascript-typescript.md`
- `references/language-footguns/java-kotlin.md`
- `references/language-footguns/go.md`
- `references/language-footguns/ruby.md`
- `references/language-footguns/php.md`
- `references/language-footguns/csharp-dotnet.md`
- `references/language-footguns/rust.md`
- `references/language-footguns/c-cpp.md`
