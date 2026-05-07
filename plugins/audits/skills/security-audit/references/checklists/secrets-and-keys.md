# Secrets and Keys

## Scope

Hardcoded credentials, API keys, and certificates in source and configuration; secrets in git history (committed and later removed); secrets logged or returned in error messages; secrets in container images and CI/CD outputs; secret rotation; key storage and access patterns.

A secret committed to a repository is leaked from the moment of commit — even rotation only stops *future* misuse; it does not undo prior exposure. Treat any committed secret as compromised; rotate and audit for misuse, not just remove.

## Secrets in Current Source

### What to Look For

- **High-entropy strings near security keywords** — Variables named `API_KEY`, `SECRET`, `TOKEN`, `PASSWORD`, `PRIVATE_KEY`, `CLIENT_SECRET`, `WEBHOOK_SECRET`, `JWT_SECRET`, with literal values (not placeholders).
- **Recognizable formats**:
  - AWS access key: `AKIA[0-9A-Z]{16}` (and `ASIA` for STS), 40-char secret.
  - GitHub PAT: `ghp_*` (40 chars), `gho_*`, `ghu_*`, `ghs_*`, `ghr_*`.
  - GCP service account JSON: contains `"type": "service_account"`, `"private_key": "-----BEGIN PRIVATE KEY-----..."`.
  - Azure Storage: `AccountKey=` connection strings.
  - Slack: `xoxb-`, `xoxa-`, `xoxp-`, `xoxr-`.
  - Stripe: `sk_live_*` (live secret key), `sk_test_*`, `rk_live_*` (restricted key), `whsec_*` (webhook secret).
  - Twilio: `SK[a-zA-Z0-9]{32}`.
  - SendGrid: `SG.*`, `MG.*`.
  - Google API: `AIza[0-9A-Za-z\-_]{35}`.
  - PEM keys: `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`.
  - JWT in code (typically test fixtures, but real ones leak).
  - Database connection strings with embedded passwords.
- **Hardcoded passwords in test fixtures, examples, README** — Real-world secrets accidentally placed in fixtures get committed. Verify obvious test passwords (`test123`, `admin`) are clearly fake.
- **Default values in `.env.example`, `config.yaml.example`** — Should contain placeholder values, not real ones. Verify no `your-real-key-here` left from copy.
- **Test data files** — `fixtures/`, `test-data/`, `mocks/` may contain real-looking credentials.
- **Build / deploy scripts with inlined credentials** — `Dockerfile` ENV/ARG for secrets, `Makefile`, shell scripts that source from a file later replaced or with fallbacks.

### Files Commonly Targeted

- `.env`, `.env.production`, `.env.local`, `*.env`
- `config/secrets.yml`, `config/database.yml`, `config/credentials.yml.enc` (Rails encrypted; verify key isn't committed alongside)
- `application.properties`, `application.yml` (Spring)
- `appsettings.json`, `appsettings.Production.json` (.NET)
- `wrangler.toml`, `serverless.yml` with hardcoded values
- Terraform `.tfvars`, especially `terraform.tfvars`
- Kubernetes manifests with literal `Secret` data (often base64-encoded plaintext)
- CI configs: `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`, `bitbucket-pipelines.yml` with `${{ secrets.X }}` references vs. inline values

### Detection Heuristics

- Entropy: high Shannon entropy of long strings is a signal but produces false positives on hashes, base64-encoded large data.
- Context match: combine entropy with surrounding identifier names.
- Structural patterns: known prefixes/formats above.
- Cross-reference with what *should* be in the file (placeholder vs. real value).
- Run as a manual semi-grep across the repo: `grep -r -E '(api[_-]?key|secret|password|token).{0,5}[=:].{0,5}["\047][^"\047]{16,}'`.

## Secrets in Git History

A secret removed from the current code may still be in history. Anyone with read access to the repository (including all forks, even after the original is private) has read access to history.

### Detection

- **Tools** (when tool augmentation is allowed): gitleaks, trufflehog, git-secrets — designed for this. The skill is configured for pure-Claude analysis, but if the user allows: a quick `gitleaks detect --source .` on the repo is high-signal.
- **Manual inspection**: `git log -p --all -- <suspect-files>` to see prior versions; `git log --all -S 'pattern'` to find commits adding/removing a string.
- **`.git/packed-refs` and stale branches** — Removed branches may still have refs; force-push doesn't delete the objects until garbage collection (and not always then).

### What to Report

- Each historical secret found: file, commit hash, date, what kind of secret, and current status (rotated? still active? unknown?).
- Recommendation: rotate immediately, audit for misuse during exposure window, then optionally rewrite history (BFG, git filter-repo). Note that history rewrite breaks forks and requires coordination; document accordingly.

## Secrets in Logs

User-supplied data containing secrets may be logged inadvertently; structured logging may serialize an entire object including a `password` field.

- **Audit log calls near auth flows** — Login, signup, password reset, token issuance. Are request bodies logged? Are response bodies logged?
- **Audit logging of HTTP requests** — Middleware that logs full requests captures `Authorization` headers, cookies, body params.
- **Audit error handling** — Exception messages including the row that failed, with all fields.
- **Stack traces in logs** — May include argument values.
- **Log shipping pipelines** — Logs may flow to less-trusted destinations; sensitive data in logs is sensitive in those destinations.

### Common Field Names to Redact

- `password`, `passwd`, `secret`, `token`, `api_key`, `authorization`, `cookie`, `session`, `private_key`, `client_secret`, `refresh_token`, `access_token`, `id_token`, `mfa_secret`, `otp_seed`.
- PII fields: `ssn`, `tax_id`, `dob`, `passport`, `id_number`, `national_id`, `cc_number`, `card`, `cvv`, `card_pin`.

### Patterns

- **Allowlist logging** — Only log explicitly-named fields; never log a full request/response object.
- **Redact at the logger level** — Logger configured with field-name allowlist or denylist; redaction is centralized.
- **Test the logger** — A logger redaction missing a field is invisible until production. Verify with a test that asserts a log line doesn't contain the secret.

## Secrets in Errors / Responses

- **Stack traces returned to client** — `DEBUG=True` left on in prod (Django, Flask, Rails). Verify `DEBUG=False` enforced in production config and that error handlers don't include traceback in response.
- **Error fields including the input** — `{"error": "Cannot validate user", "input": {...full body...}}`.
- **Verbose database errors** — Returning the database error message exposes table/column names; can leak cell values too.
- **Diff / merge tools** — Returning the conflicting field's value in error.

## Secrets in Container Images

- **`ENV` / `ARG` with secret values** — Layered into image; persists in image history even if removed in later layer.
- **`COPY .env .` / `COPY config/`** — Copying env files into image.
- **Build-time secrets** — Use Docker BuildKit `--secret` mounts (don't end up in layers) or out-of-image build.
- **Image labels / metadata** — Some teams use labels for build info; verify no secrets.
- **Detect**: `docker history <image>`, `docker inspect <image>`, `dive` (TUI). Audit Dockerfile.

## Secrets in CI/CD Pipelines

- **Inline secret values in workflow files** — Always wrong; use the platform's secret store.
- **`echo $SECRET` / printing for debugging** — Captured in workflow logs; log scrubbing may not catch.
- **Permissive `GITHUB_TOKEN`** — Default may grant write to PRs and packages; restrict per-job.
- **Self-hosted runners** — Persistent state; one PR's malicious code can read secrets from prior jobs.
- **`pull_request_target` event with PR code checkout** — "Pwn-request"; covered in `cicd.md`.

## Secret Rotation

- **Rotation mechanism exists** — Documented or codified.
- **Rotation does not require redeploy** — For cleanly rotated secrets, the application picks up new values from the secret store without code change.
- **Old secrets accept window** — During rotation, both old and new accepted briefly; verify cleanup.
- **Rotation log / audit** — Each rotation timestamped; access logs reviewable.
- **Forced rotation on suspicion** — Mechanism exists; not a code finding but worth noting in recommendation.

## Key Storage Patterns

### Acceptable

- **Cloud KMS** (AWS KMS, GCP KMS, Azure Key Vault) — keys never leave the service; envelope encryption for application-level needs.
- **HashiCorp Vault** — secret store with rotation, audit, RBAC.
- **Cloud secret managers** (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, Doppler) — for non-key secrets.
- **Environment variables (runtime)** — acceptable for small footprint; verify the env is populated from a secret store at deploy time, not committed.
- **HSM-backed keys** for high-value scenarios.

### Unacceptable

- Secrets in source.
- Secrets in container image layers.
- Secrets in unencrypted Kubernetes manifests committed to the repo (sealed-secrets / external-secrets-operator are acceptable).
- Secrets in client-accessible config (frontend env, mobile app bundle).
- Secrets in env vars exposed via debug endpoints (`/debug/env`, framework default debug pages).

## Public-Repository Risk Multiplier

For any finding involving a public repository:

- Even if the secret has been removed from current code, it's been read by anyone watching the repo and indexed by search engines and clone services.
- Treat any discovery as Critical (assuming the secret has any access value) and recommend immediate rotation and audit for unauthorized use.
- Note any repository-search aggregators (`grep.app`, GitHub code search, GitHub-history scrapers) as amplifying the exposure.

## Recommendation Patterns

- Move every committed secret to a secret store; replace with environment-variable lookup.
- Add `gitleaks` / `trufflehog` / `detect-secrets` pre-commit hook to prevent recurrence.
- For existing committed secrets: rotate first, then optionally rewrite history. Order matters; rotating before rewrite ensures rotation completes regardless of rewrite outcome.
- For logs: configure structured logger with field allowlist/denylist; test redaction.
- For container images: build-time secrets via BuildKit secret mounts; verify layer hygiene with `docker history`.
- For CI/CD: use platform secret store; restrict secret access to specific jobs; audit workflow files for inline values; restrict `GITHUB_TOKEN` permissions to minimum.
- Document rotation runbook in `SECURITY.md`; rotation cadence per secret class.
