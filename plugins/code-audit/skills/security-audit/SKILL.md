---
name: security-audit
description: >
  This skill should be used when the user asks to "security audit", "find
  vulnerabilities", "check for security issues", "pentest this code",
  "security review", "OWASP audit", "find injection vulnerabilities",
  "check authentication", "check authorization", "find secrets", "check
  for XSS", "check for SSRF", "review crypto", "audit dependencies for
  CVEs", "review Dockerfile security", "review CI/CD security", "find
  IDOR / BOLA", "check for SSTI", "review JWT handling", "find hardcoded
  credentials", "check for path traversal", "review for prototype
  pollution", "find deserialization issues", or otherwise asks for a
  security-focused review of a codebase.
---

# Security Audit Skill

## Purpose

Perform a thorough, threat-model-first security audit of a codebase, producing a structured report with findings ranked by Impact x Exploitability x Exposure, mapped to CWE and OWASP categories, and (for High and Critical) supported by a concrete exploit scenario. Deliver the report as a Markdown file at the project root following the template in `references/report-template.md`.

This skill is exclusively focused on security. For general code-quality auditing (concurrency, dead code, anti-patterns, performance, correctness, error handling, tests), use the sibling `audit` skill in the same plugin. The two skills can run independently or in sequence on the same codebase.

## Effort Level

Read every line of in-scope code. Do not skim, sample, or rely on heuristics to skip files. The defining technique of this audit is **source-to-sink dataflow** — trace untrusted input from every entry point through the call graph to every dangerous sink, and at each step evaluate validation, encoding, authorization, and trust assumptions. Pattern matching alone misses real vulnerabilities; dataflow tracing finds them. When the scope is too large for a single pass, split work across parallel subagents (see Phase 5 for partitioning strategy).

## Mindset

Audit the code as a pentester would: assume any untrusted input is hostile, assume any check can be bypassed unless proven otherwise, assume any boundary can be crossed unless validated. Defense-in-depth matters — when one layer fails, what's the next layer? Look for what is *missing* (a check that should exist), not only what is *present* (a check that's wrong). Most real-world breaches come from missing controls, mass-assignment-style overposting, broken object-level authorization, and trust placed in user-controlled fields, not from exotic exploits.

## Workflow

The workflow has six phases. Phases run sequentially; parallelism happens within phases (intent discovery subagents, partition subagents).

### Phase 1 — Resolve Scope

Determine the audit scope from the user's prompt. The user may specify:

- An entire repository or working directory.
- One or more directories (e.g., `src/`, `services/api/`).
- A list of specific files.
- A functional area described in natural language (e.g., "the authentication flow", "the file-upload endpoint").
- A specific function, class, or code region within a file. Restrict analysis to the specified code and its immediate dependencies.

Generated files (lock files, compiled output, vendored dependencies, minified bundles) are excluded by default. For source-committed generated code (protobuf stubs, OpenAPI clients, ORM models), check for modification markers — if files contain hand-written additions or "DO NOT EDIT" headers have been removed, treat them as in-scope. Cross-trust-boundary analysis in Phase 5 still traces flows *into* excluded code to determine whether it provides expected validation.

If the user specifies paths that do not exist, warn about missing paths and proceed with the rest. If none exist, stop and ask for clarification.

If the prompt does not contain enough information to determine scope, ask a single clarifying question before proceeding. An audit with unclear boundaries produces unreliable results.

If the user requests a re-audit, search for `SECURITY-AUDIT-REPORT-*.md` at the project root. If multiple reports exist, use the most recent. Default to the same scope as the prior audit. When a prior report exists, note in the new report which previous findings have been resolved and which persist, so the user gets a delta view.

State the resolved scope back to the user. If unambiguous, proceed immediately without waiting for confirmation.

### Phase 2 — Threat-Model the Application

Before reading the code in depth, build a Threat Model Brief that drives the rest of the audit. Without this step, the audit becomes an undifferentiated checklist scan; with it, the audit prioritizes the threats that actually apply to this application.

The Threat Model Brief identifies:

- **Application kind** — Web API, server-rendered web app, single-page app, CLI, library, mobile backend, desktop app, embedded, IoT, browser extension. Different kinds have radically different attack surfaces.
- **Exposure** — Internet-facing (anonymous traffic), authenticated public, internal-only (corporate network), localhost-only, single-tenant, multi-tenant. Same vulnerability has very different severity at different exposures.
- **Sensitive data classes** — PII, payment data, authentication credentials, session tokens, health records, financial records, IP/source code, customer-uploaded files. Determines which threats matter most.
- **Trust boundaries** — Where untrusted data enters (HTTP layer, message queues, file uploads, third-party API responses, environment), where privilege transitions occur (anonymous to authenticated, user to admin, tenant A to tenant B), where data crosses processes or services.
- **Authentication model** — Form login + session cookie, JWT bearer, OAuth/OIDC, mTLS, API keys, signed URLs, none. Pinpoints which parts of the auth checklist apply.
- **Authorization model** — RBAC, ABAC, ownership-based, none. Tells you what kind of authorization checks to look for.
- **External dependencies and integrations** — Databases, caches, queues, third-party APIs, payment processors, identity providers. Each is a potential trust boundary.
- **Deployment context** — Containers, serverless, VMs, on-prem; cloud provider; CI/CD platform. Drives the IaC/CI/CD checklists.

Detect this from `README`, `package.json`/`pyproject.toml`/`go.mod`, `Dockerfile`, `docker-compose.yml`, framework signals (Django settings, Express middleware, Spring annotations, Rails routes), entry-point files, and the directory structure. Ask a single clarifying question only if the kind of application or exposure is genuinely ambiguous from the code; otherwise infer and state your inference.

The Threat Model Brief is a short structured document (typically 200–400 words) consumed in every later phase. It is regenerated on each audit run; never persist it across runs.

See `references/threat-modeling.md` for the full template, the inference heuristics, and worked examples for common application kinds.

### Phase 3 — Discover Security Intent

Scan for documented security decisions, threat models, and acknowledged limitations before analysis begins. This reduces false positives by distinguishing deliberate trade-offs from genuine issues.

Run three subagents in parallel:

- **Documentation Scanner** — reads project-wide documentation, with extra weight on `SECURITY.md`, `THREAT_MODEL.md`, `THREAT-MODEL.md`, `.well-known/security.txt`, `docs/security/`, `SECURITY-POLICY.md`, ADRs that mention security, compliance documents, penetration test reports.
- **Code Intent Scanner** — searches in-scope source for security suppressions (`# nosec`, `# noqa: S*`, `# semgrepignore`, `// codeql[*]`, `# bandit:skip`, `// CodeQL`, `// lgtm`, security-related `eslint-disable` comments) and rationale comments mentioning `security`, `auth`, `CSRF`, `XSS`, `injection`, `sandbox`, `untrusted`, `trusted`, `hardened`, `threat`, `attacker`, `risk`, `mitig`.
- **History Scanner** — extracts intent signals from git history, prioritizing commits whose messages mention `security`, `CVE`, `vulnerability`, `auth`, `permission`, `escalation`, `disclosure`, `harden`, `mitigat`, `patch`, `bypass`, `exploit`, `pentest`. Includes commits that touched security-sensitive files (auth code, crypto, secrets handling, deserialization).

This step always executes regardless of codebase size. The output is a structured Security Intent Brief organized by theme (Documented Threat Model, Acknowledged Risks & Trade-offs, Security Conventions, Suppressed Findings & Rationale, Historical Security Fixes). Target no more than 100 entries.

If intent discovery produces no entries, the brief is empty. Analysis proceeds normally; the report's "Documented Security Posture" section states that no documented intent signals were identified.

See `references/intent-discovery.md` for detailed subagent prompts and the brief template.

### Phase 4 — Build the Source/Sink Map

Before flagging vulnerabilities, enumerate the application's untrusted-input sources and dangerous sinks. The map drives the rest of the audit: every High/Critical injection-class finding traces back to a documented source and a documented sink.

**Sources** to enumerate:

- HTTP entry points: route handlers, request body, query string, path params, headers (User-Agent, Referer, X-Forwarded-For, custom), cookies, form fields, multipart files
- Authentication artifacts treated as input: JWT claims, OAuth state, SSO assertions
- CLI entry points: argv, stdin
- File and message inputs: file uploads, message queue consumers (Kafka/SQS/RabbitMQ), webhook receivers, scheduled job payloads
- External calls treated as trusted: third-party API responses, identity provider responses, payment provider callbacks
- Environment: env vars (especially when injected from user-controlled sources like Helm values), config files
- Storage read after writing untrusted data: rows previously written from untrusted input ("second-order" injection)

**Sinks** to enumerate:

- Database query construction (SQL string-building, raw queries, ORM `.raw()` / `.query()`, NoSQL `$where`, dynamic `find()`)
- Shell execution (`exec`, `system`, `popen`, `subprocess` with `shell=True`, `child_process.exec`, `Runtime.exec`, backticks)
- Code execution (`eval`, `Function`, `setTimeout(string)`, `vm.runInThisContext`, `exec` in Python, `instance_eval` in Ruby)
- Deserialization sinks (pickle, ObjectInputStream, unserialize, YAML.load, JSON.parse with reviver, BinaryFormatter)
- Filesystem (`open`, `read`, `write`, `unlink`, `Path` joining; archive extraction)
- HTTP egress (`requests`, `fetch`, `axios`, `http.Client`, `urllib`) — SSRF risk
- Template rendering with user input (Jinja2, Twig, EJS, Handlebars `{{{ }}}`, ERB, Thymeleaf)
- HTTP response writing (HTML, JSON serialization, redirect headers, Set-Cookie, Location)
- Logging (sensitive data leak, log injection)
- Cryptographic operations (key handling, signing, verification)
- Authentication/authorization decision points (any code that grants access)

The map does not need to be exhaustive prose; a list of `source@file:line` and `sink@file:line` entries with the input shape and the operation suffices. For monorepos, build the map per service.

See `references/source-sink-mapping.md` for enumeration heuristics, framework-specific source/sink patterns, and worked examples.

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
- Walk through each applicable checklist from `references/checklists/`. The Threat Model Brief tells you which checklists are applicable: authn/authz/sessions for any auth code, injection for any sink-adjacent code, crypto for any code that calls crypto APIs, file handling for upload/download endpoints, error/logging across the board, etc. Skip checklists that don't apply to the application kind (e.g., XSS/CSRF on a CLI tool).
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

For every finding tentatively scored High or Critical, attempt to construct a concrete exploit scenario:

- Specify the attacker's starting position (anonymous internet user, authenticated low-privilege user, authenticated high-privilege user adjacent tenant, internal user, etc.).
- Provide a concrete input or sequence of requests, including HTTP method, path, headers, and body where relevant. Use placeholder values where credentials would be needed.
- Trace the dataflow from input to outcome, citing file:line for each step.
- State the achieved outcome (data read, data written, command executed, account taken over, privilege escalated).

If a concrete scenario can be constructed, include it in the finding under "Exploit Scenario."

If a scenario cannot be constructed despite the underlying weakness being clear (defense-in-depth gap, missing-but-not-yet-exploitable check, second-layer protection that mitigates in practice but not in design, ambiguous trust boundary), keep the finding at its assessed severity and label the section "Exploit Scenario — Not Confirmed" with an explanation: what would be needed to exploit, why the construction failed, and the residual risk. Do not silently downgrade or drop a High/Critical solely because the scenario is hard to construct — defenders need to know what's questionable.

For Medium and Low findings, an exploit scenario is optional. Include one when it materially clarifies impact; omit it when the weakness is self-explanatory.

See `references/exploit-scenarios.md` for templates, examples for each major weakness class, and rules for what counts as a valid scenario.

#### Deduplication

Before report generation:

- Merge findings with the same root cause manifesting in multiple locations into a single finding listing all affected locations.
- Remove false positives discovered during Phase B (e.g., input that appeared unvalidated in one file but is validated by middleware discovered later — verify the middleware actually applies to the route in question).
- Consolidate findings derived from the same documented trade-off in the Security Intent Brief into a single finding referencing the intent entry.
- Reconcile conflicting severity assessments by considering worst realistic Impact × Exploitability × Exposure.

### Phase 6 — Report Generation

Generate the final report following `references/report-template.md`. Specifically:

- Set the report date to the current date.
- Populate the summary table: scope, application kind and exposure (from Threat Model Brief), domains audited, finding counts by severity.
- Include "Threat Model" and "Documented Security Posture" sections summarizing Phase 2 and Phase 3 outputs.
- Include a "Source/Sink Map" appendix summarizing the Phase 4 enumeration so the reader can see what was analyzed.
- Assign sequential `SEC-NNN` identifiers starting at `SEC-001`. Order findings by severity (Critical first), then by domain category, then by file path.
- Each finding contains: identifier, short title, domain category, CWE ID(s), OWASP Top 10 / API Top 10 mapping, location (`file:line`), severity (with the Impact/Exploitability/Exposure breakdown), description, impact, exploit scenario (or "Not Confirmed" explanation for High/Critical), and recommendation.
- Save the report at the project root using the filename convention in the template (`SECURITY-AUDIT-REPORT-YYYY-MM-DD.md`), incrementing the suffix if the file already exists.

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
- `references/report-template.md` — Phase 6 report format.

**Domain checklists** (load only those flagged applicable by the Threat Model Brief):

- `references/checklists/auth-and-session.md`
- `references/checklists/authorization.md`
- `references/checklists/injection.md`
- `references/checklists/xss-csrf-frontend.md`
- `references/checklists/ssrf-redirect-url.md`
- `references/checklists/crypto.md`
- `references/checklists/deserialization.md`
- `references/checklists/file-handling.md`
- `references/checklists/secrets-and-keys.md`
- `references/checklists/error-and-logging.md`
- `references/checklists/business-logic.md`
- `references/checklists/api-security.md`
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
