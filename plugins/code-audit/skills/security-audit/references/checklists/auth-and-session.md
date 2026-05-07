# Authentication and Session Management

## Scope

Authentication (authn) is "who is this subject?" Sessions and tokens persist that identification across requests. This checklist covers password authentication, MFA, session management, password reset, JWT, OAuth/OIDC, API keys, and signed URLs. Authorization (what they're permitted to do) is in `authorization.md`; together they cover the access-control surface.

## Checks

### Password Storage

- **Hashing algorithm** — Must be a memory-hard or iterated KDF: argon2id, scrypt, bcrypt, PBKDF2 (with high iteration count). Never plain SHA-*, MD5, or single-round hashing.
- **Salt** — Per-user random salt. Built-in to bcrypt/argon2/scrypt outputs. Verify implementations don't strip the salt.
- **Cost parameters** — argon2id minimum: m=19MiB, t=2, p=1 (current OWASP guidance). bcrypt cost factor ≥ 12 (CPU-bound, plan upgrades). Verify cost is updatable.
- **Pepper** (optional defense-in-depth) — Server-side secret added to all password hashes; raises bar for offline cracking. Document rotation strategy.
- **Length / charset enforcement** — Maximum length should accommodate at least 64 bytes; very low caps (e.g., 16 chars) signal architectural problems. NIST SP 800-63B disallows arbitrary composition rules; allow long passphrases.
- **Common-password rejection** — Reject top-N most common passwords. NIST recommends.
- **Verification using constant-time comparison** — Bcrypt/argon2 verifiers do this; custom implementations often do not.
- **Re-hash on login** — When cost parameters change or hash algorithm migrates, rehash on successful login with new params.

### Login Endpoint

- **Rate limiting** — Per-account and per-IP. Exponential backoff on repeated failures, account-level (with reset on legitimate login) and IP-level.
- **Account lockout** — Optional. If implemented, must include unlock mechanism (time-based or admin) and not enable DoS by attacker locking arbitrary accounts.
- **Generic error message** — "Invalid username or password" — never specify which was wrong. (Some frameworks return per-field validation errors that leak this; verify.)
- **Timing equality** — Login that returns immediately for unknown user but takes 100ms for known user enables enumeration. Always perform a dummy hash comparison even on unknown users to equalize timing.
- **MFA prompt timing** — MFA-only-after-correct-password also enumerates. Prompt for MFA code consistently or randomize timing.
- **Login enumeration via signup** — `/signup` returning "email already exists" enumerates users. Use email-verification flow instead.
- **Login enumeration via password reset** — Password reset returning "no account with that email" enumerates. Always return the same generic response.
- **Credential stuffing exposure** — At minimum, integrate haveibeenpwned API for password compromise check at password creation. Detect and slow credential-stuffing patterns (many failed logins across many accounts from same IP).

### MFA

- **TOTP** — Standard TOTP (RFC 6238). Verify clock-drift window is reasonable (≤ 1 step ahead/back is typical). Verify replay protection: each code valid only once, or once per window.
- **Recovery codes** — Treated as second factor with the same protection as TOTP secrets: hashed at rest, single-use, regeneratable. Do not log or email in plaintext beyond initial generation.
- **WebAuthn / FIDO2** — Verify origin binding, challenge freshness, attestation handling matches policy.
- **SMS MFA** — Acceptable as fallback but not primary. Rate-limit code sending; cap codes per phone per day; expire codes quickly (3-5 minutes).
- **MFA enrollment** — Require existing factor before adding/removing; require existing factor before disabling MFA entirely.
- **Step-up authentication** — Sensitive operations (change email, change password, change MFA, withdraw funds) should require recent authentication or fresh MFA, not just session validity.

### Session Management

- **Session ID generation** — At least 128 bits of entropy from CSPRNG. Verify with the framework's docs; reject custom RNG-based generators.
- **Session storage** — Server-side store (Redis, DB, signed-cookie with care). For signed-cookie sessions, verify integrity (HMAC) and that secret is strong and rotatable.
- **Cookie flags** — `Secure`, `HttpOnly`, `SameSite` (Lax or Strict for cookies that authenticate). `Domain` not set wider than necessary. `Path=/` typical, but consider narrower if applicable.
- **Session lifetime** — Idle timeout (15-60 minutes typical for sensitive apps), absolute timeout (8-24 hours typical). Server-side enforcement; expired sessions invalidated server-side, not just client-side.
- **Session rotation** — On login, session ID must change to prevent fixation. On privilege change (role elevation, account merge), regenerate.
- **Concurrent sessions** — Policy choice; if limited to N, enforce server-side. List of active sessions accessible to the user.
- **Logout** — Server-side invalidation (delete session record). Just clearing cookie client-side is not logout.
- **"Remember me" tokens** — Long-lived tokens stored separately from session; verify rotation on use, single-use semantics, secure storage. Avoid trivially long-lived tokens.
- **Cross-domain considerations** — Subdomains receiving cookie; subdomain takeover risk amplifies session exposure.

### Password Reset

- **Token generation** — At least 128 bits CSPRNG; not derived from email or username; not predictable (no timestamp+username hash).
- **Token expiration** — Short window (15-60 minutes typical). Expired tokens rejected server-side.
- **Single-use** — Once consumed, invalidated. Failed reset attempts also expire (don't allow brute force).
- **Token leak via Referer** — Reset link click → page with reset form → form posts to attacker-linked external resources, leaking token. Mitigate with Referrer-Policy.
- **Account lockout on reset** — Reset itself shouldn't unlock locked account (unless that's policy); verify intended interaction.
- **Reset email enumeration** — Same as login: same response whether email exists or not.
- **Reset email content** — Don't include current password, MFA secret, or other sensitive info.
- **Old session invalidation on password change** — All other sessions invalidated when password changes; especially critical post-reset.
- **MFA bypass via reset** — Some implementations clear MFA on reset, granting attacker single-factor access. Reset must not bypass MFA.

### Account Recovery / "Forgot Username"

- Same enumeration concerns as login.
- Recovery via security questions: NIST deprecates; if used, treat as low-strength factor.
- Recovery via support agent: out-of-scope for code audit but document the human channel as a residual risk if encountered.

### JWT

- **Algorithm allowlist** — `verify(token, secret, { algorithms: ['HS256'] })`. Without an allowlist, library may accept attacker-controlled `alg` (`none`, RSA-vs-HS256 confusion).
- **`alg=none` rejection** — Verify even with allowlist; some libraries handle differently.
- **Algorithm confusion** — Token signed with HS256 verified with public key of an RSA key pair: attacker uses the public key (often public) as HMAC secret. Allowlist must lock to expected algorithm given the verification key.
- **Key ID (`kid`) handling** — `kid` from header must not be used to look up arbitrary files / DB rows / URLs; allowlist permitted kid values or allowlist storage namespace.
- **Expiration (`exp`)** — Must be present and verified. Acceptable clock skew small (≤ 60s). Tokens without `exp` are forever-valid.
- **Issued at (`iat`)** / **not before (`nbf`)** — Verify when present; reject far-future `iat` (replay attack), `nbf` not yet reached.
- **Issuer (`iss`) / Audience (`aud`)** — Verify. JWT for service A should not be accepted by service B if the issuer/audience differ.
- **Subject (`sub`) usage** — Must reference an actual subject; verify subject is active and not deleted/suspended on every authenticated request, or accept short-lived tokens with no DB check.
- **JWT in URLs** — Logged in proxy logs, browser history, Referer headers; avoid for sensitive tokens.
- **Replay protection** — JWT alone is not replay-safe; if replay matters, use jti (JWT ID) with server-side replay cache, or pair with short expiration.
- **Refresh tokens** — Must be one-time-use (rotation on each refresh) and bound to a session; reuse detection cancels the lineage.

### OAuth / OIDC (as Client)

- **Authorization code flow with PKCE** — Use PKCE for all clients, public or confidential. Without PKCE, public clients are vulnerable to code-interception attacks.
- **`state` parameter** — Random per-flow value, validated on callback; protects against CSRF on the OAuth flow.
- **`nonce` (OIDC)** — Random per-flow, included in id_token, validated.
- **Redirect URI validation** — Exact-match comparison against registered URIs; not prefix or pattern match. Localhost ports should be tightly scoped.
- **Token exchange over HTTPS** — TLS verification not disabled.
- **Token storage** — Access tokens in memory; refresh tokens encrypted at rest if persisted.
- **Provider trust** — Verify identity provider's issuer; pin to expected JWKS endpoint.
- **`id_token` claim validation** — Audience, issuer, expiration, signature all verified.

### OAuth / OIDC (as Provider)

If the application *issues* OAuth tokens, depth dramatically increases:

- Authorization endpoint must require user authentication and consent.
- `response_type` allowlist; deprecated implicit flow disabled.
- Code single-use, short-lived, bound to PKCE, redirect URI, client.
- Token endpoint client authentication enforced for confidential clients.
- Scope enforcement on every authorized resource server.
- Refresh token rotation, theft detection.
- Revocation endpoint (RFC 7009).
- See OAuth 2.1 / OIDC specs in detail; this is its own audit domain.

### API Keys

- **Generation** — CSPRNG, ≥ 128 bits.
- **Storage** — Hashed at rest (like passwords) so DB compromise doesn't yield usable keys. Plain-text-stored keys are findings.
- **Display once** — Show full key only at creation; subsequent views show prefix only.
- **Scoping** — Each key scoped to specific permissions, IPs, or operations.
- **Rotation** — Mechanism for users to rotate; invalidation immediate; no long grace periods.
- **Header vs query** — Query strings appear in logs; prefer Authorization header. If query is supported, document log handling.
- **Key in source code** — Hardcoded API keys in source/config are findings. (See `secrets-and-keys.md`.)

### Signed URLs / Pre-signed URLs

- **HMAC algorithm** — SHA-256 minimum.
- **Expiration** — Required parameter; validated.
- **Scope** — Bound to specific resource and operation; cannot be repurposed.
- **Replay** — Within expiration window, replay is allowed by design; ensure operation is idempotent or protect downstream.
- **Time-skew tolerance** — Bounded; not "accepts ±24 hours."
- **Algorithm confusion** — Same risk as JWT; lock to expected algorithm.

### Service-to-Service Authentication

- **mTLS** — Mutual TLS with peer certificate verification. Verify the certificate chain, hostname, and revocation handling.
- **Service tokens** — JWTs or opaque tokens issued by an internal IdP; verify scope and lifetime.
- **Network-only trust** — "Internal network is trusted" is dangerous; verify lateral-movement assumptions match the threat model.
- **Default service account** — Cloud workloads often have a default account with broad access; verify least privilege.

## Bypass Patterns

- **Header trust on `X-Forwarded-User`** — Trusting any user-controlled or proxy-stripped-but-not-sanitized header for identity.
- **Cookie value coercion** — Cookies parsed in non-string ways enabling NoSQL operator injection.
- **Session puzzles** — Combining state across requests where the server "remembers" partial progress and trusts the next request's claim of where it was.
- **Race on session validity** — Session expiring mid-request; behavior on the boundary may grant access on a stale token.
- **JWT not actually verified** — Code calls `decode(token)` (no signature check) instead of `verify(token, secret)`. Easy mistake; common.

## Recommendation Patterns

- Use a battle-tested auth library; do not implement password hashing, JWT verification, or OAuth flows from scratch.
- Centralize auth in middleware; per-route auth is easy to forget.
- Tests must cover negative cases: forged tokens, expired tokens, wrong-issuer tokens, mass-assignment of role.
- Periodic credential and key rotation; document the rotation runbook in `SECURITY.md`.
- Step up authentication for sensitive operations; do not rely on session validity alone.
