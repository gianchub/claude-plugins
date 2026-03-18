# Code Audit

Language-agnostic codebase auditing with structured, severity-ranked reports.

This plugin performs thorough, systematic code audits that go beyond surface-level scanning. It reads every line of in-scope code, traces data flows across module boundaries, and produces a structured Markdown report with findings ranked by severity. It works with any language or framework, adapting its analysis checklists to the technologies present in your codebase.

## Installation

Add the marketplace, then install:

```
/plugins marketplace add gianchub/claude-plugins
/plugins install code-audit
```

## Skills

### audit

Performs a comprehensive codebase audit covering security vulnerabilities, concurrency issues, dead code, anti-patterns, performance problems, correctness bugs, error handling gaps, and test quality. The skill intelligently selects which audit categories to apply based on your request and the codebase under review, confirms the plan with you, then executes a systematic two-phase analysis producing a structured `AUDIT-REPORT-YYYY-MM-DD.md` file at the project root.

Trigger with: "audit this codebase", "security audit", "code audit", "find vulnerabilities", "check for bugs", "review code quality", "find dead code", "check for anti-patterns", "performance audit"

## How It Works

1. **Resolve scope** -- determines which files and directories to audit based on the request, excluding generated files, lock files, and vendored dependencies unless explicitly included
2. **Select categories** -- analyzes the request and codebase to select relevant audit categories (e.g., concurrency checks are included automatically if async patterns are detected), then confirms the selection with the user
3. **Systematic analysis** -- two-phase analysis:
   - File-level: reads every line of every in-scope file, walking through category checklists item by item
   - Cross-file: traces data flows from entry points through processing layers to terminal operations, evaluating validation, authorization, error handling, and resource cleanup across module boundaries
4. **Large codebase strategy** -- partitions work across parallel subagents for codebases with 50+ files or 10,000+ lines of code, splitting along architectural boundaries for meaningful cross-file analysis within each partition
5. **Report generation** -- produces a structured `AUDIT-REPORT-YYYY-MM-DD.md` file with deduplicated findings ordered by severity, each including file location, category, description, impact assessment, and actionable recommendation

## Audit Categories

1. **Security vulnerabilities** -- injection, authentication bypasses, access control, hardcoded secrets, SSRF, insecure deserialization, cryptography issues, CORS misconfiguration
2. **Race conditions and concurrency** -- shared mutable state, TOCTOU, missing locks, async hazards, transaction isolation, filesystem races
3. **Dead code** -- unreachable branches, unused imports/variables/functions, commented-out code, orphaned tests, stale feature flags
4. **Anti-patterns and code smells** -- god objects, deep nesting, magic numbers, excessive coupling, copy-paste duplication, SRP violations
5. **Performance** -- N+1 queries, unnecessary allocations, missing pagination, blocking in async contexts, algorithmic complexity, missing caching
6. **Correctness** -- off-by-one errors, null handling gaps, integer overflow, floating-point comparison, type coercion, boundary conditions
7. **Error handling gaps** -- swallowed exceptions, missing error paths, incomplete cleanup, leaked internals, missing retry/backoff, unhandled promise rejections
8. **Test quality** -- excessive mocking, vacuous assertions, missing failure path tests, test pollution, snapshot overuse, tests coupled to implementation, dead/skipped tests

## Severity Levels

- **Critical** -- immediate exploitable threats, direct paths to data loss or corruption, hardcoded production credentials, privilege escalation to admin access
- **High** -- correctness bugs that produce wrong results for common inputs, race conditions on production paths, missing authorization checks, error handling gaps that cause cascading failures
- **Medium** -- performance issues under realistic load (N+1 queries, blocking in async contexts), anti-patterns that significantly degrade maintainability, missing validation with limited blast radius
- **Low** -- dead code, unused imports, minor code smells, commented-out code, missing pagination on currently small datasets

## Requirements

- Claude Code

## License

MIT
