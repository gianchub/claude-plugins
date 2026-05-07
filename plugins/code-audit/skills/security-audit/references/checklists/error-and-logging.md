# Error Handling and Logging (Security Aspects)

## Scope

Information disclosure via errors, log injection, sensitive data in logs, audit-log gaps, and error handling that breaks security invariants. Logging is dual-purpose: missing logs mean attacks go undetected; over-logging leaks sensitive data; broken logs (injection) corrupt audit trails. The recommendation pattern is "log enough, log right, never log too much."

## Information Disclosure via Errors

### Stack Traces to Users

- **Framework debug mode in production** — Django `DEBUG=True`, Flask `app.debug = True`, Rails `config.consider_all_requests_local = true`, ASP.NET `<customErrors mode="Off">`, Spring Boot `server.error.include-stacktrace=always`. Renders full traces with code snippets to anonymous users.
- **Generic 500 handler returning the exception** — `res.status(500).json({error: err.message})`, particularly with `err.stack`.
- **Database errors propagated** — "Column 'password_hash' does not exist" → schema disclosure.
- **GraphQL** — Apollo Server / GraphQL.js return formatted errors; production should mask non-user-facing errors.

### Defenses

- **Generic error response in production** — `{ "error": "An error occurred. Reference: <correlation_id>" }`. Real details go to logs.
- **Correlation IDs** — Returned to client; logged server-side with full detail. Allows support without exposure.
- **Differentiate user errors from system errors** — Validation errors should explain what's wrong with input; system errors should be opaque.
- **Custom error classes** — Distinguish "safe to expose" from "internal" at the type level; serializer maps to public response.

### Verbose Validation Errors

- **Per-field error fields** — May leak existence of fields, valid value ranges, type constraints used in business logic. Generally acceptable for documented public APIs; document the leak when business logic is sensitive.
- **"User not found" vs. "Invalid password"** — Login enumeration; covered in auth-and-session.md.
- **"Invalid email" vs. "Email already registered"** — Signup enumeration.
- **Rate limit messages** — "Too many requests; current count: 47" leaks rate limit threshold.

## Sensitive Data in Logs

### Categories

- **Authentication**: passwords, password hashes, tokens (session, JWT, refresh, MFA codes, recovery codes), API keys, signed-URL signatures.
- **PII**: full names, emails, government IDs, addresses, phone numbers, dates of birth.
- **Payment**: PAN, CVV, bank account numbers, full transaction history.
- **Health**: diagnoses, prescriptions, biometrics.
- **Internal infrastructure**: service-to-service tokens, KMS responses, IAM credentials, instance metadata responses, internal hostnames and IPs (lower sensitivity but operationally important).
- **Application secrets**: webhook signing secrets, encryption keys, JWT secrets.

### Common Leak Patterns

- **Logging full request / response bodies** — Middleware that logs `req.body`, `req.headers`, `res.body`. Authentication endpoints leak passwords; session-creation endpoints leak tokens.
- **Logging exception with `e.args` or `e.message`** — When the exception was raised with sensitive data as an argument.
- **Structured logging dumping objects** — `logger.info("user", user)` where `user` includes `password_hash` and `mfa_secret`.
- **Debug logs left enabled** — `logger.debug("got token: " + token)` left from development.
- **Performance metrics including values** — Tracing systems that capture argument values to functions.
- **HTTP client logging at DEBUG level** — `requests`, `axios`, etc., when set to verbose log all headers and bodies.

### Defenses

- **Field allowlist on the logger** — Logger configured to drop or hash fields by name (`password`, `token`, `authorization`, `cookie`, etc.). Centralized; one place to update.
- **Structured logging with explicit fields** — `logger.info("user_login", user_id=u.id)` — only named fields; never log the whole object.
- **Sentinel for partial values** — Log `password=***` or `token=eyJ***...***xyz` (prefix + suffix only) — proves correct flow without leaking value.
- **Tested redaction** — Tests that assert the logger doesn't write a known-secret value. Without tests, redaction silently breaks.
- **Sample rate** — High-volume sensitive endpoints may sample logs at lower rate to reduce exposure.

## Log Injection

User input with control characters injected into log lines corrupts the log.

- **CRLF injection** — `\r\n` in user input creates fake log lines:
  ```
  username=alice\r\n2024-01-01 10:00:00 INFO admin_login user=root
  ```
  An incident review reading the log sees "admin_login user=root" as a real event.
- **Terminal injection** — ANSI escapes (`\x1b[`) in logs viewed in terminals can hide content or change perceived output.
- **JSON corruption in NDJSON logs** — User-supplied `"` breaking out of a string field if message construction is via concatenation rather than JSON serialization.

### Defenses

- **Strip or escape control characters** — Before logging user input, replace `\r`, `\n`, `\t`, ESC, NUL with safe representations (`\\r`, `\\n`).
- **Structured logging serializes correctly** — A correct JSON-emitting logger handles escaping; manual concatenation does not.
- **Logger versions** — log4j-style format-string vulnerabilities (`${jndi:...}` CVE-2021-44228) — verify version of logging library; upstream fixes generally cover, but configuration may re-enable.

## Audit Logging

Detection requires evidence; missing audit logs means attacks succeed silently.

### What to Log

- **Authentication events** — Login (success and failure), MFA challenge / success / failure, logout, session creation, session destruction.
- **Authorization events** — Access denied (especially repeated denials), privilege elevation, role change.
- **Account lifecycle** — Signup, email verification, password reset request and completion, password change, MFA setup / removal, deletion.
- **Sensitive data access** — Read of high-sensitivity fields (PII / PHI / payment), bulk export.
- **Administrative actions** — User create / suspend / delete, role assignment, configuration change, feature flag toggle, billing change.
- **Security configuration changes** — IAM policy edits, API key generation / revocation, webhook configuration.
- **Anomalies** — Unusual request rate, IP geography mismatch, multiple-account-from-IP patterns. (Often handled by a separate detection layer; verify it exists.)

### What to Capture per Event

- Timestamp (ISO 8601, UTC, monotonic where possible).
- Actor identity (user ID, session ID, source — "actor" not "user" because automated processes act).
- Action and target (verb-noun: "user.delete", "billing.update").
- Source IP and user-agent (with header injection mitigations).
- Outcome (success / failure; reason for failure when relevant).
- Correlation/request ID (for cross-log joining).
- *Not* the actual values — never log the new password value, the old API key, etc. Reference by ID.

### Audit Log Integrity

- **Append-only storage** — Audit logs to a write-once medium, separate from operational logs, with restricted write permissions.
- **Tamper-evident** — Hash chain or external log service that detects tampering.
- **Retention** — Per regulatory requirement; commonly 1+ year.
- **Access controls on logs** — Read access restricted; admin actions on logs themselves audited.

## Error-Handling Patterns That Break Security

### Fail Open

Default behavior on error is "permit," not "deny." Common in:

- Authorization checks wrapped in try/except that returns "allow" on exception.
- Token validation that returns "valid" if the validation library throws.
- Rate-limit checks that "skip rate limit" if Redis is down.

Each is a finding. Default deny on error.

### Race-Condition-Inducing Error Handling

- Releasing a lock in the catch block but not re-acquiring before retry.
- Cleanup that completes despite incomplete mutation, leaving inconsistent state visible.
- Error path skipping audit log emission.

### Catch-All That Hides Real Issues

- `except Exception` / `catch (Throwable)` that logs only at debug level. Real attacks may produce exceptions; if those exceptions are swallowed silently, detection fails.

### Resource Leak on Error

- Authentication tokens / database transactions / files not closed in the error path. Generally a reliability concern; security concern when leaked tokens persist or transaction state is partial.

### Different Error Codes for Different Internal Reasons

- Returning HTTP 401 for "wrong password" and HTTP 403 for "account locked" — exposes account state.
- Returning HTTP 200 with `{"success": false}` for one path and HTTP 500 for another, when the difference reveals internal state.

## Logging Pipeline Considerations

- **Log forwarding to third parties** — Splunk, Datadog, Logtail, etc. Verify no sensitive data exported. Document the data classification flowing to each destination.
- **Log retention and deletion** — Aligns with data-classification policy.
- **Log sampling at the source** — High-volume endpoints sampled; verify security-relevant events not sampled out.
- **Log encryption at rest and in transit** — Especially when crossing trust boundaries.

## Recommendation Patterns

- Generic error responses to clients with correlation IDs; full detail in logs.
- Structured logging with field allowlist; redaction tested.
- Audit log to dedicated store, separate from operational logs.
- Default-deny error handling; never fail open.
- Audit log every authentication, authorization, and configuration change.
- Document log retention and access policy in `SECURITY.md`.
