# Code Audit

Language-agnostic codebase auditing with structured, severity-ranked reports.

This plugin performs thorough, systematic code audits that go beyond surface-level scanning. It reads every line of in-scope code, traces data flows across module boundaries, and produces a structured Markdown report with findings ranked by severity. It works with any language or framework, adapting its analysis checklists to the technologies present in your codebase.

The plugin ships two complementary skills: a general-purpose `audit` skill for code quality, and a deeper `security-audit` skill focused exclusively on security. They can be run independently or in sequence on the same codebase.

## Installation

Add the marketplace, then install:

```
/plugins marketplace add gianchub/claude-plugins
/plugins install code-audit
```

## Skills

### audit

Performs a comprehensive code-quality audit covering concurrency issues, dead code, anti-patterns, performance problems, correctness bugs, error handling gaps, and test quality. The skill intelligently selects which audit categories to apply based on your request and the codebase under review, confirms the plan with you, then executes a systematic two-phase analysis producing a structured `AUDIT-REPORT-YYYY-MM-DD.md` file at the project root.

Trigger with: "audit this codebase", "code audit", "check for bugs", "review code quality", "find dead code", "check for anti-patterns", "performance audit", "technical debt", "code health check", "review test quality", "check error handling"

### security-audit

Performs a security-focused audit using a threat-model-first, source-to-sink workflow. Covers the OWASP Top 10, OWASP API Top 10, CWE Top 25, language-specific footguns, secrets in current code and git history, dependency manifest review, IaC and container security, and CI/CD pipeline risks. Produces a structured `SECURITY-AUDIT-REPORT-YYYY-MM-DD.md` file with findings mapped to CWE and OWASP categories, severity scored by impact x exploitability x exposure, and concrete exploit scenarios for High and Critical findings.

Trigger with: "security audit", "find vulnerabilities", "check for security issues", "pentest this code", "OWASP audit", "find injection vulnerabilities", "check authentication", "check authorization", "find secrets", "review for XSS", "check for SSRF", "audit dependencies for CVEs", "review Dockerfile security", "review CI/CD security"

## How It Works

1. **Resolve scope** -- determines which files and directories to audit based on the request, excluding generated files, lock files, and vendored dependencies unless explicitly included. Re-audits default to the full original scope
2. **Select categories** -- analyzes the request and codebase to select relevant audit categories (e.g., concurrency checks are included automatically if async patterns are detected). For broad requests, defaults to all categories without asking
3. **Discover intent** -- scans the codebase for documented design decisions, trade-offs, conventions, and known limitations before analysis begins. Three parallel subagents scan documentation files, rationale comments, and git history to produce an Intent Brief that prevents flagging deliberate choices as findings
4. **Systematic analysis** -- two-phase analysis with automatic partitioning for large codebases (50+ files or 10,000+ lines, whichever applies first):
   - File-level: reads every line of every in-scope file, walking through category checklists item by item, cross-referencing potential findings against the Intent Brief
   - Cross-file: traces data flows from entry points through processing layers to terminal operations, evaluating validation, authorization, error handling, and resource cleanup across module boundaries
5. **Report generation** -- produces a structured `AUDIT-REPORT-YYYY-MM-DD.md` file with deduplicated findings ordered by severity, each including file location, category, description, impact assessment, and actionable recommendation. Includes a "Context & Intent" section documenting what design decisions were considered

## Audit Categories (audit skill)

1. **Race conditions and concurrency** -- shared mutable state, TOCTOU, missing locks, async hazards, transaction isolation, filesystem races
2. **Dead code** -- unreachable branches, unused imports/variables/functions, commented-out code, orphaned tests, stale feature flags
3. **Anti-patterns and code smells** -- god objects, deep nesting, magic numbers, excessive coupling, copy-paste duplication, SRP violations
4. **Performance** -- N+1 queries, unnecessary allocations, missing pagination, blocking in async contexts, algorithmic complexity, missing caching
5. **Correctness** -- off-by-one errors, null handling gaps, integer overflow, floating-point comparison, type coercion, boundary conditions
6. **Error handling gaps** -- swallowed exceptions, missing error paths, incomplete cleanup, leaked internals, missing retry/backoff, unhandled promise rejections
7. **Test quality** -- excessive mocking, vacuous assertions, missing failure path tests, test pollution, snapshot overuse, tests coupled to implementation, dead/skipped tests

## Security Audit Domains (security-audit skill)

1. **Authentication & sessions** -- password storage, MFA, session management, password reset, JWT, OAuth/OIDC, login enumeration
2. **Authorization** -- IDOR/BOLA, BFLA, mass assignment, tenant isolation, vertical and horizontal privilege escalation
3. **Injection** -- SQL/NoSQL, command, code, LDAP, XPath, template (SSTI), header (CRLF), log injection
4. **XSS, CSRF, and frontend security** -- stored/reflected/DOM XSS, CSP, CSRF, SameSite cookies, postMessage, security headers
5. **SSRF, redirects, URL handling** -- SSRF including cloud metadata, open redirect, URL parser confusion, DNS rebinding
6. **Cryptography** -- algorithm/mode/IV choice, KDFs, randomness, constant-time comparison, JWT alg confusion
7. **Deserialization** -- Python pickle, Java ObjectInputStream, PHP unserialize, YAML, .NET BinaryFormatter, gadget chains
8. **File handling** -- path traversal, zip slip, polyglots, image library vulnerabilities, upload validation
9. **Secrets and keys** -- hardcoded credentials in source and git history, secrets in logs, leaked env
10. **Error handling and logging** -- info disclosure, log injection, sensitive data in logs, audit-log gaps
11. **Business logic** -- workflow bypass, race-on-auth, double-spend, abuse flows, state machine flaws
12. **API security** -- OWASP API Top 10 (BOLA, BFLA, excessive data exposure, rate limiting, security misconfig)
13. **Dependencies** -- known-vulnerable versions, lock file integrity, typosquatting, install-time hooks
14. **Containers and IaC** -- Dockerfile, Kubernetes manifests, Terraform, cloud IAM, secret material in images
15. **CI/CD** -- GitHub Actions pwn-request, secrets in workflows, untrusted action versions, runner trust

## Severity Levels

The two skills use different severity models. The `audit` skill uses an impact-based scale; the `security-audit` skill uses an Impact x Exploitability x Exposure model. See each skill's `severity-guide.md` for the full criteria.

### audit skill (code quality)

- **Critical** -- direct paths to data loss or corruption, silently wrong results in critical business paths, resource exhaustion under realistic load, race conditions producing permanently inconsistent persistent state
- **High** -- correctness bugs that produce wrong results for common inputs, race conditions on production paths, error handling gaps that cause cascading failures, missing cleanup that leaks resources over time
- **Medium** -- performance issues under realistic load, anti-patterns that significantly degrade maintainability, missing validation with limited blast radius, test quality gaps on important paths
- **Low** -- dead code, unused imports, minor code smells, commented-out code, missing pagination on currently small datasets

### security-audit skill

Severity is composed from Impact (what an attacker achieves), Exploitability (how easy to trigger), and Exposure (whether the affected code is reachable by untrusted users / from the internet). The same weakness can be Critical when internet-facing and unauthenticated, and Medium when behind authentication and additional layers of defense. High and Critical findings are required to include a concrete exploit scenario; if the auditor cannot construct one despite the underlying weakness being clear, the finding still ships at its assessed severity, marked "Exploitability not confirmed" with the reasoning.

## Requirements

- Claude Code

## License

MIT
