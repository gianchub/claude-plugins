# Severity Classification Guide

Use the following criteria to assign severity levels. When a finding could fit multiple levels, choose the higher severity and note the reasoning. This guide covers code-quality severity only; the `security-audit` skill defines its own security-tuned severity model that incorporates exploitability and exposure.

**Intent-based downgrades:** When the Intent Brief provides general context justifying a pattern (but not an explicit per-instance acknowledgment), reduce severity by one level (e.g., High → Medium) and cite the intent source in the finding. Never downgrade Critical findings below High — Critical severity indicates risk significant enough that even documented intent warrants attention.

### Critical

Reserve for findings that represent a near-certain path to significant damage in normal operation:

- Direct paths to data loss or corruption (unprotected destructive operations, missing transaction safety on critical writes).
- Correctness bugs that produce silently wrong results on common inputs in critical business paths (financial calculations, billing, audit-relevant state).
- Resource leaks or unbounded growth that will exhaust memory, file handles, or connections under realistic load.
- Race conditions on persistent state that produce permanently inconsistent or corrupt data.

### High

Assign to findings that are likely to cause real-world bugs, outages, or significant degradation under normal operating conditions:

- Correctness bugs that produce wrong results or crash the application for common inputs.
- Race conditions on data structures or resources accessed in production paths.
- Error handling gaps that cause cascading failures (e.g., unhandled exceptions in request middleware that crash the entire process).
- Missing cleanup on error paths that leak resources over time.

### Medium

Assign to findings that degrade quality, maintainability, or performance but are unlikely to cause immediate failures:

- Performance issues that cause slowdowns under realistic load (N+1 queries, blocking in async contexts, O(n^2) algorithms on growing datasets).
- Anti-patterns that make the code significantly harder to maintain or extend (god objects, deep nesting, SRP violations).
- Missing input validation on internal APIs where the blast radius is limited and no cross-trust-boundary impact exists.
- Test quality gaps on important code paths (vacuous assertions, missing failure-path coverage).

### Low

Assign to findings that represent minor quality issues with limited practical impact:

- Dead code (unused imports, unreachable branches, orphaned tests) that adds noise but does not affect runtime behavior.
- Minor code smells (magic numbers in non-critical paths, slightly duplicated code blocks).
- Commented-out code that should be cleaned up.
- Missing pagination on endpoints with currently small datasets but potential future growth.
