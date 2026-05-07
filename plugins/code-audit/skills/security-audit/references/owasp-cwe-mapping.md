# OWASP and CWE Mapping

## Purpose

Quick-reference lookup for tagging findings with OWASP Top 10, OWASP API Top 10, and CWE identifiers. Each finding in the report should carry at least one CWE ID and at least one OWASP category tag where applicable. Mappings make findings consumable for compliance discussions, link to widely-used taxonomies, and let downstream tooling (ticketing, dashboards) categorize findings consistently.

## OWASP Top 10 (2021)

| ID | Title | Typical findings |
|----|-------|------------------|
| A01:2021 | Broken Access Control | IDOR/BOLA, BFLA, missing authz checks, mass assignment, privilege escalation, path traversal due to lack of permission checks, CORS misconfig with credentials |
| A02:2021 | Cryptographic Failures | Weak algorithms (MD5/SHA1 for security), insecure modes (ECB, CBC w/o integrity), poor randomness, key/cert mishandling, plaintext credential transmission, missing TLS, weak password hashing |
| A03:2021 | Injection | SQL/NoSQL/LDAP/XPath/command/code/template/header/log injection, ORM injection, unsafe XSLT, OS-command injection |
| A04:2021 | Insecure Design | Architectural flaws: missing rate limiting on sensitive flows, missing trust boundaries, no anti-automation on signup/login, business-logic flaws, lack of segregation between trust zones |
| A05:2021 | Security Misconfiguration | Default credentials, verbose errors, unhardened framework defaults, unnecessary features enabled, missing security headers, insecure cookie flags, debug enabled in prod, permissive CORS |
| A06:2021 | Vulnerable and Outdated Components | Known-CVE dependencies, unsupported runtime versions, unpinned versions allowing supply-chain swap, transitive vulns |
| A07:2021 | Identification and Authentication Failures | Weak password policies, broken session management, credential stuffing exposure, missing MFA, weak password reset, JWT alg confusion, login enumeration via timing or messaging |
| A08:2021 | Software and Data Integrity Failures | Insecure deserialization, untrusted updates, CI/CD pipeline tampering risk, missing signing/verification on artifacts, supply-chain ingestion without integrity check |
| A09:2021 | Security Logging and Monitoring Failures | Missing audit logs on critical actions, sensitive data in logs (passwords/tokens), log injection, no detection on auth failures or admin actions |
| A10:2021 | Server-Side Request Forgery (SSRF) | Unvalidated URL fetches, cloud metadata access, internal service access via URL parsing inconsistencies, redirect-following SSRF, blind SSRF |

## OWASP API Top 10 (2023)

Use these tags in addition to the main Top 10 when the application exposes an API. They can co-occur (an API authz finding is both A01 and API1).

| ID | Title | Typical findings |
|----|-------|------------------|
| API1:2023 | Broken Object Level Authorization (BOLA) | Object identifier from request used to fetch/modify without checking ownership or tenancy; the canonical IDOR finding |
| API2:2023 | Broken Authentication | Auth flaws specific to API surface: weak token handling, JWT issues, refresh-token reuse, misissued tokens |
| API3:2023 | Broken Object Property Level Authorization | Mass assignment (overposting) and excessive data exposure: client can write or read fields they should not access |
| API4:2023 | Unrestricted Resource Consumption | Missing rate limiting, missing request size limits, GraphQL query complexity, expensive operations callable in loop, no concurrency limits, no per-tenant quotas |
| API5:2023 | Broken Function Level Authorization (BFLA) | Function-level checks missing: admin endpoints reachable by non-admin users; sibling endpoints share authz code path that one endpoint omits |
| API6:2023 | Unrestricted Access to Sensitive Business Flows | Sensitive flows (account creation, password reset, refunds, item purchase) without anti-automation protection |
| API7:2023 | Server-Side Request Forgery | SSRF specifically in API contexts (preview generators, webhook tests, image proxying, federation features) |
| API8:2023 | Security Misconfiguration | API-specific: misconfigured CORS, permissive default policies, undocumented endpoints exposed, debug routes in prod |
| API9:2023 | Improper Inventory Management | Old API versions still reachable, undocumented endpoints, env-mismatch (staging exposed in prod) |
| API10:2023 | Unsafe Consumption of APIs | Treating third-party API responses as trusted: no validation of response content, deserialization of attacker-influenced response data, server-side rendering of third-party data |

## CWE Quick Reference

The following CWEs cover the great majority of security findings. Pick the most specific CWE that applies; chain to a parent if the leaf is not a clear fit.

### Injection-class

- **CWE-89** — SQL Injection
- **CWE-943** — NoSQL / Object-Relational Mapping Injection
- **CWE-77** — Command Injection
- **CWE-78** — OS Command Injection
- **CWE-94** — Code Injection
- **CWE-95** — Eval Injection (specific to `eval`-style)
- **CWE-1336** — Improper Neutralization of Special Elements Used in a Template Engine (SSTI)
- **CWE-90** — LDAP Injection
- **CWE-91** — XML Injection
- **CWE-643** — XPath Injection
- **CWE-93** — Improper Neutralization of CRLF Sequences (HTTP header / log injection)
- **CWE-117** — Improper Output Neutralization for Logs
- **CWE-1284** — Improper Validation of Specified Quantity in Input

### XSS / Frontend

- **CWE-79** — Cross-Site Scripting (XSS) — covers stored, reflected, DOM
- **CWE-1021** — Improper Restriction of Rendered UI Layers / Frames (clickjacking)
- **CWE-352** — Cross-Site Request Forgery (CSRF)
- **CWE-693** — Protection Mechanism Failure (broad; use only when a more specific CWE doesn't fit)

### Authorization / Access Control

- **CWE-285** — Improper Authorization
- **CWE-639** — Authorization Bypass Through User-Controlled Key (IDOR / BOLA)
- **CWE-862** — Missing Authorization
- **CWE-863** — Incorrect Authorization
- **CWE-915** — Improperly Controlled Modification of Dynamically-Determined Object Attributes (mass assignment)
- **CWE-269** — Improper Privilege Management
- **CWE-732** — Incorrect Permission Assignment for Critical Resource

### Authentication / Session

- **CWE-287** — Improper Authentication
- **CWE-306** — Missing Authentication for Critical Function
- **CWE-307** — Improper Restriction of Excessive Authentication Attempts (no lockout / brute force)
- **CWE-384** — Session Fixation
- **CWE-613** — Insufficient Session Expiration
- **CWE-352** — CSRF (also listed above)
- **CWE-345** — Insufficient Verification of Data Authenticity (signature checks)
- **CWE-347** — Improper Verification of Cryptographic Signature (e.g., JWT alg=none, alg confusion)

### Cryptography

- **CWE-327** — Use of a Broken or Risky Cryptographic Algorithm
- **CWE-326** — Inadequate Encryption Strength
- **CWE-330** — Use of Insufficiently Random Values
- **CWE-338** — Use of Cryptographically Weak Pseudo-Random Number Generator
- **CWE-916** — Use of Password Hash with Insufficient Computational Effort
- **CWE-759** — Use of a One-Way Hash without a Salt
- **CWE-760** — Use of a One-Way Hash with a Predictable Salt
- **CWE-200** — Exposure of Sensitive Information (broad; pair with a more specific CWE when possible)
- **CWE-208** — Observable Timing Discrepancy (timing side channel)
- **CWE-321** — Use of Hard-coded Cryptographic Key
- **CWE-798** — Use of Hard-coded Credentials
- **CWE-916** — Use of Password Hash with Insufficient Computational Effort

### Deserialization / Data Integrity

- **CWE-502** — Deserialization of Untrusted Data
- **CWE-915** — Improperly Controlled Modification of Dynamically-Determined Object Attributes (mass assignment)
- **CWE-913** — Improper Control of Dynamically-Managed Code Resources

### File Handling / Path

- **CWE-22** — Path Traversal (`../`)
- **CWE-23** — Relative Path Traversal
- **CWE-36** — Absolute Path Traversal
- **CWE-434** — Unrestricted Upload of File with Dangerous Type
- **CWE-552** — Files or Directories Accessible to External Parties
- **CWE-409** — Improper Handling of Highly Compressed Data (zip bomb)
- **CWE-918** — SSRF (canonical CWE for SSRF)

### Information Disclosure

- **CWE-200** — Exposure of Sensitive Information to Unauthorized Actor
- **CWE-209** — Generation of Error Message Containing Sensitive Information
- **CWE-532** — Insertion of Sensitive Information into Log File
- **CWE-525** — Use of Web Browser Cache Containing Sensitive Information
- **CWE-538** — Insertion of Sensitive Information into Externally-Accessible File or Directory

### Configuration / Misconfiguration

- **CWE-16** — Configuration (parent; use only when a leaf doesn't fit)
- **CWE-732** — Incorrect Permission Assignment
- **CWE-942** — Permissive Cross-domain Policy with Untrusted Domains (CORS)
- **CWE-1004** — Sensitive Cookie Without HttpOnly Flag
- **CWE-1275** — Sensitive Cookie with Improper SameSite Attribute
- **CWE-614** — Sensitive Cookie in HTTPS Session Without Secure Attribute
- **CWE-319** — Cleartext Transmission of Sensitive Information
- **CWE-693** — Protection Mechanism Failure
- **CWE-829** — Inclusion of Functionality from Untrusted Control Sphere (third-party scripts without SRI)

### Race Conditions / Concurrency (Security-relevant)

- **CWE-362** — Concurrent Execution using Shared Resource with Improper Synchronization (race condition)
- **CWE-367** — TOCTOU
- **CWE-364** — Signal Handler Race Condition

### Resource Management / DoS

- **CWE-770** — Allocation of Resources Without Limits (rate limit, quota)
- **CWE-400** — Uncontrolled Resource Consumption
- **CWE-776** — Improper Restriction of Recursive Entity References (XML XEE / billion-laughs)
- **CWE-611** — Improper Restriction of XML External Entity Reference (XXE)

### Supply Chain / Dependencies

- **CWE-1104** — Use of Unmaintained Third Party Components
- **CWE-1357** — Reliance on Insufficiently Trustworthy Component
- **CWE-829** — Inclusion of Functionality from Untrusted Control Sphere

### Business Logic

Business-logic flaws often map to multiple CWEs depending on the manifestation. Common picks:

- **CWE-840** — Business Logic Errors (parent)
- **CWE-841** — Improper Enforcement of Behavioral Workflow
- **CWE-708** — Incorrect Ownership Assignment

## How to Tag a Finding

1. Pick the **most specific CWE** that fits. Use the parent CWE only if no leaf is appropriate.
2. Pick the **OWASP Top 10** category that fits (one is usually enough; two when the finding genuinely spans).
3. If the application is API-based and the finding has an API-specific manifestation, also pick the **OWASP API Top 10** category.
4. Tag format in the report:
   ```
   CWE: CWE-639
   OWASP: A01:2021 — Broken Access Control
   OWASP API: API1:2023 — Broken Object Level Authorization
   ```

## When Mappings Don't Fit

Some real findings don't fit cleanly into Top 10 categories — for example, a defense-in-depth gap that reduces blast radius but isn't itself a vulnerability. In those cases:

- Use the closest CWE; CWE-693 (Protection Mechanism Failure) is acceptable when nothing else fits.
- Omit the OWASP tag; do not invent a fit. Note in the description: "Does not map cleanly to OWASP Top 10; reported as defense-in-depth."

The goal is accurate communication, not full taxonomy coverage. Better to leave a tag empty than to mislead the reader with a wrong tag.

## CWE Top 25 (2024) Quick Reference

The CWE Top 25 (most dangerous software weaknesses) is useful when prioritizing for compliance contexts that reference it. The 2024 list:

```
CWE-79   Cross-site Scripting
CWE-787  Out-of-bounds Write
CWE-89   SQL Injection
CWE-352  Cross-Site Request Forgery
CWE-22   Path Traversal
CWE-125  Out-of-bounds Read
CWE-78   OS Command Injection
CWE-416  Use After Free
CWE-862  Missing Authorization
CWE-434  Unrestricted Upload of File with Dangerous Type
CWE-94   Code Injection
CWE-20   Improper Input Validation
CWE-77   Command Injection
CWE-287  Improper Authentication
CWE-269  Improper Privilege Management
CWE-502  Deserialization of Untrusted Data
CWE-200  Exposure of Sensitive Information
CWE-863  Incorrect Authorization
CWE-918  SSRF
CWE-119  Improper Restriction of Operations within the Bounds of a Memory Buffer
CWE-476  NULL Pointer Dereference
CWE-798  Use of Hard-coded Credentials
CWE-190  Integer Overflow or Wraparound
CWE-400  Uncontrolled Resource Consumption
CWE-306  Missing Authentication for Critical Function
```

When a finding falls into the Top 25, optionally note this in the description (`(CWE Top 25)`). It does not change severity but helps consumers prioritize.
