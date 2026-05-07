# Security Intent Discovery

## Purpose

Detailed subagent instructions for security-intent discovery (SKILL.md Phase 3). Three subagents run in parallel; their combined output is merged into a Security Intent Brief. The brief is consumed in Phase 5 to skip findings that the team has explicitly addressed and to downgrade findings that are partially mitigated by documented decisions.

The brief differs from the generic-audit Intent Brief in three ways:

1. It weights security-specific documentation (`SECURITY.md`, threat models, pentest reports) more heavily.
2. It scans for security-specific suppression markers and rationale comments.
3. It surfaces history of security fixes — recurring patterns of fixes signal areas where new audits should look harder, not weaker.

---

## Documentation Scanner

### Objective

Discover and read all documentation files across the project. Scan project-wide regardless of audit scope. Extract security intent: threat models, acknowledged risks, security conventions, suppressed findings with rationale, and historical security decisions.

### File Discovery

Scan for documentation files matching:

**By well-known security-specific name**: `SECURITY`, `SECURITY.md`, `SECURITY.txt`, `THREAT_MODEL`, `THREAT-MODEL`, `THREATMODEL`, `THREAT_MODELS`, `RISK_REGISTER`, `SECURITY_POLICY`, `SECURITY-POLICY`, `.well-known/security.txt`, `docs/security/*`, `docs/threat-model*`, `compliance/*`, `audits/*`, `pentest*`.

**By extension**: `.md`, `.txt`, `.rst`, `.adoc`, `.org` — full project scan.

**By keyword in name** (case-insensitive): `security`, `threat`, `risk`, `compliance`, `audit`, `pentest`, `pen-test`, `vulnerability`, `disclosure`, `responsible-disclosure`.

**Architecture / decision records**: `ADR`, `ADRs/`, `architecture-decisions/`, `docs/decisions/`. Filter for ones mentioning security keywords.

**Agent instruction files**: `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.cursorrules`, `.windsurfrules`, `.github/copilot-instructions.md`. Often contain security expectations or "don't change without security review" notes.

**Compliance documents**: `SOC2*`, `HIPAA*`, `PCI*`, `GDPR*`, `compliance/*`. Often contain security control statements.

### Extraction

Read each discovered file and extract:

- **Documented threat models** — adversaries, in-scope assets, attack surface, accepted risks. The most valuable signal.
- **Documented security architecture** — auth model, trust boundaries, key/secret management approach, deployment posture, network segmentation, defense-in-depth layers.
- **Acknowledged risks and trade-offs** — features the team knows are risky but accepts the risk; mitigations they rely on; rationale for accepting.
- **Security conventions and standards** — required patterns (always parameterize queries, never log request bodies, all admin actions audited, all secrets in <secret store>).
- **Historical security incidents and fixes** — incidents the team has had, what was fixed, and what processes were added.
- **Pentest summaries** — most recent test, scope, findings, status. Pentest scope tells you what has and hasn't been examined.
- **Rate limiting / WAF / external mitigations** — controls the team is relying on outside the application code.

Ignore purely procedural content (installation guides, API reference) unless it contains security-relevant rationale.

### Report Format

```
- **Source:** <file path>
  **Theme:** <Threat Model | Architecture | Acknowledged Risk | Convention | History | Pentest | External Control>
  **Summary:** <one-line summary of the security intent signal>
  **Quote:** "<direct quote when useful>" (optional)
```

---

## Code Intent Scanner

### Objective

Search in-scope source files for security-specific suppressions, rationale comments, and intent markers. Capture both blanket suppressions ("we know this is flagged but we accept it") and per-instance acknowledgments.

### Security Suppression Markers

Scan for inline directives that suppress security tools:

**SAST and linter security rules**:

- Bandit (Python): `# nosec`, `# nosec B<id>`
- Semgrep: `# nosemgrep`, `// nosemgrep`, `# nosemgrep: rule-id`
- ESLint security plugins: `// eslint-disable-next-line security/<rule>`, `// eslint-disable security/<rule>`
- CodeQL: `// codeql[<query>]`, `// lgtm`, `// lgtm[<query>]`
- gosec: `//nolint:gosec`, `// #nosec`
- Brakeman: `# brakeman:disable`, `# brakeman_safe`
- Snyk: `// snyk:ignore`
- SonarQube: `// NOSONAR`
- Checkmarx: `// CxIgnore`

**Secret-scanner suppressions**:

- gitleaks: `# gitleaks:allow`, `// gitleaks:allow`
- detect-secrets: `# pragma: allowlist secret`
- TruffleHog: scope-based (no comment marker but check `.trufflehogignore`)

**Generic linter suppressions that often hide security issues**:

- Python: `# noqa: S<id>` (where Sxxx are security rules in flake8-bandit/ruff), `# noqa: TRY<id>`
- TypeScript: `@ts-ignore`, `@ts-expect-error` near security-sensitive code

For each suppression, capture: file path, line, the rule being suppressed, and any comment explaining why. A suppression *with* a rationale comment is intent. A suppression *without* rationale is a code-quality concern that may itself be a finding.

### Security Rationale Markers

Scan for comments containing security-keyword rationale (case-insensitive):

`security`, `auth`, `csrf`, `xss`, `xsrf`, `injection`, `sanitize`, `escape`, `untrusted`, `trusted`, `unsafe`, `safe`, `sandbox`, `harden`, `mitig`, `threat`, `attacker`, `exploit`, `vuln`, `cve`, `pwn`, `pentest`, `risk`, `confidential`, `pii`, `secret`, `credential`, `password`, `token`, `key`.

Combine with rationale signals like `because`, `intentional`, `deliberate`, `by design`, `we chose`, `to prevent`, `to avoid`, `mitigates`. Only record comments that *explain why* — comments that merely restate what the code does are noise.

### Explanatory Block Comments

Multi-sentence comments that describe security trade-offs (e.g., "We allow `eval` here because the input is constrained to <X> and validated upstream by <Y>; if either changes, this becomes unsafe."). These are high-value intent signals.

### Config File Security Comments

Check security-relevant config files for inline rationale:

- `.env.example`, `Dockerfile`, `docker-compose.yml`
- CI configuration: `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`, `.circleci/config.yml`
- Web server configs: `nginx.conf`, `Apache vhosts`, `Caddyfile`
- Framework security configs: `helmet.config.js`, CSP definitions, `corsOptions`, `securityContext` in K8s manifests, IAM policy files

### Report Format

```
- **Source:** <file path>:<line number>
  **Type:** <Suppression | Rationale | Block | Config>
  **Comment:** "<comment text>"
  **Interpretation:** <what intent signal this provides; e.g., "team accepts shell=true here because input is hardcoded">
```

Distinguish suppressions *with* explanation (intent signal) from suppressions *without* explanation (potential finding: undocumented suppressions are themselves a security smell).

---

## History Scanner

### Objective

Extract intent signals from git history. Look for commits whose messages explain security decisions, commits that fixed security issues (and what they fixed), and patterns that signal areas needing extra audit attention.

### Execution

Run `git log` on in-scope files. Limit to the most recent 200 commits per file (or use `--since="2 years ago"` for long-lived repos). Use `--follow` for renamed files.

For repository-wide signals, also run `git log --all --grep` for security-related keywords across the whole history (without file restriction), to catch security-related fixes in files no longer in scope.

### Filtering

**Include** commits with security-relevant messages:

- Keywords: `security`, `cve`, `vulnerability`, `vuln`, `auth`, `permission`, `authz`, `authn`, `escalation`, `disclosure`, `harden`, `mitigat`, `patch`, `bypass`, `exploit`, `pentest`, `csrf`, `xss`, `xxe`, `ssti`, `ssrf`, `idor`, `bola`, `injection`, `sanitize`, `escape`, `secret`, `credential`, `token`, `password`.
- Commits referencing CVE IDs (`CVE-YYYY-NNNNN`).
- Commits referencing security advisory IDs (`GHSA-*`, `RHSA-*`).
- Pull-request merge commits whose body mentions security review or pentest follow-up.
- Commits that touch security-sensitive paths: anything under `auth/`, `crypto/`, `secrets/`, `permissions/`, `security/`, `sanitiz*`.

**Exclude** noise:

- `fix typo`, `bump version`, `update dependencies` (unless dependency bump is security-motivated; check the message).
- Single-word messages, auto-generated messages (Renovate, Dependabot — but capture security-specific Dependabot PRs).
- Formatting / lint-only commits.

### Patterns to Surface

Beyond individual commits, look for *patterns*:

- **Recurring fixes in the same area** — multiple commits over time fixing the same vulnerability class in the same module. The module is a hot spot; audit it harder, not weaker.
- **Fixes that introduce new code without removing the broken pattern** — partial fixes that leave the unsafe pattern usable in other paths.
- **Reverts of security fixes** — sometimes done for legitimate reasons (over-correction, test breakage); always worth noting.
- **Commits that disable security mechanisms** — adding suppressions, removing checks, loosening configs. Particularly worth surfacing when the rationale is thin.

### Graceful Handling

If no git history is available (fresh `git init`, exported tarball, shallow clone), report "No meaningful git history available" and complete without error. Do not fail the overall intent discovery.

### Report Format

```
- **Commit:** <short hash>
  **Date:** <YYYY-MM-DD>
  **Theme:** <Security Fix | Hardening | Suppression Added | Mechanism Disabled | Pattern>
  **Summary:** <one-line summary>
  **Files:** <comma-separated list of affected in-scope files>
```

For pattern signals (multiple commits), aggregate:

```
- **Pattern:** <description of pattern>
  **Commits:** <list of short hashes>
  **Files:** <affected paths>
  **Audit guidance:** <what this pattern suggests for the current audit>
```

---

## Security Intent Brief Template

Merge and deduplicate entries from the three subagents into the following structure. Omit sections that have no entries.

```markdown
## Security Intent Brief

### Documented Threat Model
- <key element of the documented threat model> (source: <file>)

### Security Architecture & Conventions
- <documented architectural decision or convention> (source: <file>)

### Acknowledged Risks & Trade-offs
- <known risk the team has accepted, with rationale> (source: <file>)

### Suppressed Findings & Rationale
- <what is suppressed, where, why> (source: <file>:<line>)

### Historical Security Fixes & Patterns
- <pattern or notable past fix that should inform audit attention> (source: commits/files)

### External Controls & Defense-in-Depth
- <controls outside the application code that the team relies on (WAF, gateway, IAM, etc.)> (source: <file or doc>)
```

---

## Cross-Referencing in Phase 5

In Phase 5, every candidate finding is cross-referenced against the Brief:

| Brief content | Effect on finding |
|---------------|-------------------|
| Brief explicitly addresses the exact pattern at the exact location with rationale | **Skip** the finding; do not report. |
| Brief addresses the pattern in general (e.g., "we use shell=true on internal-only scripts") but not at this specific location | **Downgrade** by one severity level; cite the brief entry in the finding. |
| Brief acknowledges a category of risk but does not justify the specific instance | **Report** at full severity; cite the acknowledgment but note that this instance was not explicitly acknowledged. |
| Brief documents an external control mitigating this finding (e.g., "WAF blocks all SQLi payloads") | **Downgrade** if the external control is verifiable from the codebase or documented configuration; **report** at full severity if the external control is asserted but unverifiable. Note the assertion. |
| Brief shows recurring fixes in this area | **No downgrade**; if anything, treat findings here with extra scrutiny — recurring fixes signal a fragile area. |

Never apply intent-based downgrades below High for findings whose default severity is Critical. Critical severity indicates risk significant enough that even documented intent warrants attention; the appropriate response is a recommendation that may include "verify the documented mitigation actually applies."

---

## Size and Prioritization

Target no more than 100 entries in the Brief. When raw entries exceed this:

- Prioritize by relevance to the confirmed audit categories from the Threat Model Brief.
- Prefer per-file/per-line entries over general statements (per-instance acknowledgments are more actionable than blanket policies).
- Discard entries unrelated to security (the generic-audit Intent Brief covers code-quality intent).
- For repeated patterns, keep the first instance and note the count.

---

## Re-audit Behavior

Regenerate the Brief from scratch on every audit run. Never persist or cache it. The codebase, threat model, and intent all evolve; a stale brief gives bad advice.
