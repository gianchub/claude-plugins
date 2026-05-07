# Security Severity Guide

## Model

Security severity is a function of three dimensions, evaluated against the Threat Model Brief from Phase 2:

```
Severity = Impact x Exploitability x Exposure
```

A weakness with the same root cause can be Critical in one application and Medium in another. The same SQL injection is Critical when on an unauthenticated internet endpoint touching customer data, and Medium when on an internal admin tool used by trusted engineers behind a VPN.

Score each dimension independently, then combine using the matrix below. Always document the three component scores in the finding so the consumer can recalibrate if the threat model changes.

## Dimensions

### Impact (what does an attacker achieve?)

| Score | Meaning |
|-------|---------|
| Severe | Full account takeover, full data exfiltration of sensitive class, remote code execution, full system compromise, payment manipulation, irreversible data destruction. |
| High | Cross-tenant or cross-user data read/write of sensitive class, privilege escalation, persistent unauthorized state changes, mass data extraction within one tenant, secrets disclosure. |
| Moderate | Limited data disclosure (one record, low-sensitivity), denial-of-service requiring sustained attack, information disclosure that materially aids further attack (internal hostnames, schema, software versions). |
| Low | Negligible direct impact in isolation; weakness contributes to chains but is not exploitable on its own (e.g., missing security header where another control mitigates). |

### Exploitability (how easy is it to trigger?)

> Note: Exploitability uses **Multistep** rather than "Moderate" to keep its scale visually distinct from the Impact scale (which uses Moderate). The two dimensions never share a level name.

| Score | Meaning |
|-------|---------|
| Trivial | Reproducible from a single crafted request or input with no precondition. Off-the-shelf tools can find or exploit it. |
| Easy | A small number of straightforward steps; standard attacker tooling; no privileged position required. |
| Multistep | Multiple steps or specific timing; requires some reconnaissance, a valid low-privilege account, or targeted crafting. |
| Hard | Requires unusual conditions: a privileged starting position, narrow timing windows, controlling an external system, or chaining multiple weaknesses. |

### Exposure (who can reach the affected code?)

| Score | Meaning |
|-------|---------|
| Public | Reachable by anonymous internet users without any prior credential or invitation. |
| Authenticated public | Reachable by anyone who can sign up or otherwise obtain a low-privilege account on a public service. For mutually-hostile multi-tenant SaaS, treat as effectively Public for cross-tenant impact. |
| Authenticated restricted | Reachable only by users from a defined set: paying customers, enterprise tenants with vetted users, internal employees behind SSO. |
| Internal | Reachable only from an internal network or via VPN; not exposed to the public internet. |
| Local | Single-user context (CLI tool, desktop app, dev-only server). Attackers must already have local access. |

## Combining the Dimensions

Use the following matrix as the default, then apply Threat-Model adjustments documented in the Threat Model Brief's "Severity-Modifier Notes" section.

### Default Severity Matrix

```
Public exposure:
  Severe Impact + (Trivial | Easy)             -> Critical
  Severe Impact + Multistep                    -> Critical
  Severe Impact + Hard                         -> High
  High Impact + (Trivial | Easy)               -> Critical
  High Impact + Multistep                      -> High
  High Impact + Hard                           -> High
  Moderate Impact + (Trivial | Easy)           -> High
  Moderate Impact + Multistep                  -> Medium
  Moderate Impact + Hard                       -> Medium
  Low Impact + any                             -> Low

Authenticated public exposure:
  Severe Impact + (Trivial | Easy)             -> Critical
  Severe Impact + Multistep                    -> High
  Severe Impact + Hard                         -> High
  High Impact + (Trivial | Easy)               -> High
  High Impact + Multistep                      -> High
  High Impact + Hard                           -> Medium
  Moderate Impact + (Trivial | Easy)           -> Medium
  Moderate Impact + (Multistep | Hard)         -> Medium
  Low Impact + any                             -> Low

Authenticated restricted exposure:
  Severe Impact + (Trivial | Easy)             -> High
  Severe Impact + (Multistep | Hard)           -> Medium
  High Impact + (Trivial | Easy)               -> Medium
  High Impact + (Multistep | Hard)             -> Medium
  Moderate Impact + any                        -> Low
  Low Impact + any                             -> Low

Internal exposure:
  Severe Impact + Trivial                      -> Medium
  Severe Impact + (Easy | Multistep | Hard)    -> Low
  Other combinations                           -> Low

Local exposure:
  Severe Impact + Trivial                      -> Low
  Other combinations                           -> Informational (typically not reported unless aggregating)
```

### Threat-Model Adjustments

The Threat Model Brief may contain "Severity-Modifier Notes" that override the default matrix. Common modifiers:

- **PHI / payment data raise the floor**: any authorization or injection finding affecting PHI/payments rounds up to at least High regardless of matrix output.
- **Hostile multi-tenancy raises authorization findings**: cross-tenant IDOR/BOLA in mutually-hostile multi-tenant context rounds up to at least High.
- **Defense-in-depth credit**: when a second layer demonstrably mitigates the issue (e.g., a WAF rule blocks the exact payload, a database layer enforces RLS), one-step downgrade is allowed if the second layer is documented in the Security Intent Brief.
- **Dependency CVE depth**: known-exploited (KEV catalog or active exploitation) raises by one level; theoretical/limited-PoC keeps default.

When applying a modifier, cite the source: "Modifier: PHI floor (Threat Model Brief, line 12)."

## Critical Reservation

Reserve Critical exclusively for findings that meet *all* of the following:

1. Severe or High Impact.
2. Public or Authenticated Public exposure.
3. A concrete exploit scenario constructed and verified (or marked "Not Confirmed" with strong reasoning that the construction is blocked by infrastructure outside the source code, not by any weakness in the construction itself).

Critical implies "drop everything and fix this." When a finding's underlying weakness is clear but a concrete exploit scenario cannot be constructed, ship it at its assessed severity with the "Exploit Scenario — Not Confirmed" structure from `references/exploit-scenarios.md`. Never silently downgrade a suspected Critical to High solely because exploit construction is hard; defenders need to know what's questionable.

## Examples

### Example 1: SQL injection on a search endpoint

| Dimension | Score | Reasoning |
|-----------|-------|-----------|
| Impact | Severe | Database read/write across all tenants; payment data accessible. |
| Exploitability | Trivial | Single crafted query parameter, sqlmap-discoverable. |
| Exposure | Public | Search endpoint is unauthenticated. |

Default matrix → Critical. Exploit scenario constructed. Final: **Critical**.

### Example 2: Same SQL injection, but on an internal admin tool

| Dimension | Score | Reasoning |
|-----------|-------|-----------|
| Impact | Severe | Same data access. |
| Exploitability | Trivial | Same payload. |
| Exposure | Internal | Admin tool only reachable on corporate VPN. |

Default matrix → Low. But: the operator is staff, not an external attacker. If an attacker compromises a single staff laptop or otherwise reaches the corporate network, the same weakness becomes reachable. Document: **Low** with a note "Severity rises to High (Authenticated-restricted, per matrix) if internal-network exposure increases; recommend fixing the root cause regardless, since the cost of remediation is small relative to the residual risk."

### Example 3: Missing CSRF token on a state-changing form

| Dimension | Score | Reasoning |
|-----------|-------|-----------|
| Impact | Moderate | Forces an authenticated user to perform a state change (e.g., update profile). Not full account takeover unless chained. |
| Exploitability | Easy | Standard CSRF payload; victim must visit attacker's page while authenticated. |
| Exposure | Authenticated public | Any signed-in user is a target. |

Default matrix → Medium. If `SameSite=Lax` cookies are configured globally (verify), Exploitability drops to Hard → **Medium → Low**. Cite the SameSite control in the recommendation.

### Example 4: Hardcoded API key in a config file

| Dimension | Score | Reasoning |
|-----------|-------|-----------|
| Impact | High to Severe | Depends on the key's scope. A read-only API key for a public service is High; an admin key for AWS is Severe. |
| Exploitability | Trivial | Anyone who can read the file (including via the public git repo) has the key. |
| Exposure | Depends | Public repo or container image → Public. Private repo with limited access → Authenticated restricted. |

For a public repo with an AWS admin key: Severe + Trivial + Public → **Critical**. Mandatory exploit scenario: enumerate access, list S3 buckets, etc. The fact that the key may have been rotated does not downgrade the finding — historical exposure is still a leak; cite "rotate immediately AND audit for prior misuse."

### Example 5: Server-Side Template Injection (SSTI)

| Dimension | Score | Reasoning |
|-----------|-------|-----------|
| Impact | Severe | RCE via template engine. |
| Exploitability | Easy to Multistep | Engine-dependent; Jinja2/Twig with sandbox bypasses are well-documented; Handlebars is harder. |
| Exposure | Public if endpoint is public | Often reachable via report-generation or email-template features. |

Severe + Easy + Public → **Critical**. Construct exploit scenario showing escape from template into RCE.

## Applying Severity to Phase 5 Output

Every finding gets a "Severity" block in the report:

```
Severity: <level> (Impact: <score>, Exploitability: <score>, Exposure: <score>)
```

When a modifier was applied, add a line:

```
Modifier: <name> (cite source)
```

When exploitability could not be confirmed despite the underlying weakness being clear:

```
Severity: <level> (Impact: <score>, Exploitability: Not Confirmed, Exposure: <score>)
```

with the "Exploit Scenario — Not Confirmed" section explaining why.

## Defense-in-Depth Findings

Some findings are *missing controls* whose presence would not, on their own, prevent an attack — but whose absence widens the blast radius if another layer fails. Examples: missing security headers, lack of rate limiting on a non-auth endpoint, missing audit logging on an admin action.

Score these on Impact alone (typically Low to Moderate). Always include a recommendation framed as defense-in-depth, not vulnerability-fix: "Add this layer so that <other failure mode> does not lead to <bad outcome>."

Defense-in-depth findings rarely deserve High and never Critical, but should not be dropped — they are the difference between a contained incident and a catastrophic one when the primary control fails.

## Severity Inflation and Deflation Pitfalls

**Inflation**: Do not score every weakness as High. Most well-built applications have a long tail of Low/Medium findings. A report full of Critical/High findings either reflects a genuinely bad application or an over-eager auditor. The latter loses the consumer's trust.

**Deflation**: Do not downgrade a finding because the fix is "small" or "simple." Severity is about realistic risk if exploited, not about effort to remediate. A one-line fix can mitigate a Critical.

**Theoretical-only deflation**: Do not downgrade purely because exploitation is "theoretical." A weakness in code is exploitable until proven otherwise; the burden is to prove non-exploitability, not to assume it.
