# Cryptography

## Scope

Algorithm choice, mode, IV/nonce handling, key derivation, key storage, randomness, signature verification, transport security configuration, and side-channel concerns. Cryptography failures rarely produce immediate compromise on their own; they amplify other failures and reduce the cost of attacks (offline cracking of leaked hashes, replaying captured data, breaking signed tokens).

The defining principle: **don't roll your own crypto, and don't tune the dials.** Use library defaults from well-maintained libraries and verify the call sites match the documented secure usage. Custom protocols, custom modes, or custom KDFs are findings even when they "look right."

## Algorithm Choice

### Symmetric Ciphers

- **Acceptable**: AES-128 / AES-192 / AES-256 with GCM, ChaCha20-Poly1305. Both provide authenticated encryption.
- **Unacceptable for new use**: DES, 3DES, RC4, Blowfish (block size too small), AES-CBC without HMAC, AES-ECB ever, AES-CTR without HMAC.
- **AES-GCM caveats** — Nonce must be unique per key; reuse breaks confidentiality and authenticity catastrophically. Maximum messages per key with random 96-bit nonces is ~2^32 (birthday bound). For high-volume, use deterministic nonces (counter or HKDF-derived) or rotate keys.
- **AES-CBC mode** — Only acceptable with HMAC (encrypt-then-MAC); never bare. Padding-oracle attacks (CVE-class) when MAC is missing or done wrong.
- **AES-ECB** — Always wrong. Repeated plaintext blocks produce identical ciphertext blocks; structurally insecure.

### Asymmetric / Public-Key

- **Acceptable**: RSA ≥ 2048-bit (2048 minimum, 3072 preferred for new), ECDSA on P-256 / P-384 / P-521, Ed25519, X25519 for key exchange.
- **Unacceptable**: RSA < 2048, DSA, ECDSA on small curves (P-192), DH on small groups (< 2048 bits).
- **RSA-PKCS#1 v1.5 padding** — Vulnerable to Bleichenbacher attacks; use OAEP for encryption, PSS for signatures. Many libraries default to v1.5 for compatibility; verify call sites use v1.5 only when interop demands and OAEP/PSS where possible.
- **Custom curves** — Reject. Only standardized curves.
- **EdDSA / Ed25519** — Strongly preferred for signatures when interop allows; deterministic and side-channel resistant.

### Hashing

- **General-purpose hashing** (integrity, fingerprints): SHA-256 / SHA-384 / SHA-512, BLAKE2, BLAKE3.
- **Password hashing**: argon2id (preferred), scrypt, bcrypt, PBKDF2-SHA256 with high iteration count. SHA-2 alone is *not* acceptable for passwords.
- **HMAC**: HMAC-SHA-256 / HMAC-SHA-512.
- **Unacceptable for security**: MD5, SHA-1 (still acceptable for non-security-sensitive integrity, like Git, but not for password hashing, signatures, certificate checks, message authentication).
- **MD5 / SHA-1 as integrity check** — Acceptable only when an authenticated channel guarantees the hash itself wasn't tampered with; rarely justifiable in modern code.

### Key Derivation

- **For passwords**: argon2id, scrypt, bcrypt, PBKDF2.
- **For deriving multiple keys from a master**: HKDF.
- **From shared secrets** (DH, ECDH): HKDF.
- **Bare hashing of password** (`SHA-256(password)`) — Wrong; use a proper password KDF.
- **PBKDF2 iteration count** — OWASP 2024: PBKDF2-SHA256 ≥ 600,000; PBKDF2-SHA512 ≥ 210,000. Older code with 1000-10000 iterations is a finding.

## Modes, IVs, and Nonces

- **Random IV per encryption** — IV must be unique per (key, plaintext) pair. For CBC: random; size = block size. For GCM/CTR: typically a counter or random; never reused with the same key.
- **Static / hardcoded IV** — Always wrong. Find code where IV is a constant, derived from input deterministically without per-message uniqueness, or zeroed.
- **IV from CSPRNG** — Verify the random source is cryptographically strong. `Math.random()` (JS), `random.random()` (Python), `Random` (Java) are not. Use `crypto.randomBytes`, `secrets.token_bytes`, `SecureRandom`.
- **Nonce reuse in GCM** — Catastrophic; an attacker who sees two ciphertexts with the same key and nonce can recover the GHASH key, forge messages, and partially recover plaintext.

## Key Management

- **Key storage at rest** — Should not be in source code. Acceptable storage: KMS service (AWS KMS, GCP KMS, Azure Key Vault), HSM, secret management service (Vault, Doppler), environment variables (acceptable for runtime, not for source).
- **Key in source code** — Find committed `.pem`, `.key`, `.p12`, hardcoded byte arrays that look like keys (32-byte / 64-byte / 256-byte literals near crypto operations).
- **Key derivation from passwords for encryption-at-rest** — Acceptable with proper KDF. Document key rotation strategy.
- **Key rotation** — Mechanism exists; old data re-encrypted or accepts both keys during rotation window. No rotation strategy is a finding.
- **Key reuse across purposes** — One master key signing tokens AND encrypting data is risky; derive purpose-specific keys via HKDF.
- **Public key infrastructure** — Pin certificates or issuer; verify revocation handling (OCSP, CRL); verify chain validation (intermediate CA chain).

## Randomness

- **CSPRNG only for security purposes** — `crypto.randomBytes` (Node), `secrets` (Python), `SecureRandom` (Java), `crypto/rand` (Go), `OpenSSL.RAND` (PHP), `RNGCryptoServiceProvider` / `RandomNumberGenerator` (.NET).
- **Non-CSPRNG used for security** — `Math.random()`, `random.random()`, `Random` class, `rand()`, `mt_rand()` for tokens, IVs, salts, nonces, password generation, anti-CSRF tokens, session IDs. Findings.
- **Seeded with low-entropy** — Time-based seeds, predictable PIDs, etc.
- **Random sampling (e.g., shuffling) using non-CSPRNG when bias matters** — Acceptable for non-security shuffling; finding when used for security like deck of authentication cards or password generation.

## Signature Verification

- **Verification actually performed** — Code calls `verify(...)`, not `decode(...)`. Common JWT mistake.
- **Verification of all relevant fields** — Subject, audience, issuer, expiration; not just signature.
- **Constant-time comparison of MAC outputs** — `hmac.compare_digest`, `crypto.timingSafeEqual`, etc. `==` on the bytes is timing-side-channel.
- **Allowlist of acceptable algorithms** — JWT alg confusion class; covered in `auth-and-session.md` JWT section.
- **Webhook signature verification** — Stripe, GitHub, etc., sign webhooks. Verify signature with the documented algorithm before processing payload. Trust no field of an unverified webhook.
- **Algorithm downgrade** — Code accepting "any algorithm" or selecting based on user-supplied header.

## Transport Security

- **TLS configuration** — TLS 1.2 minimum, 1.3 preferred. Old protocols (SSLv2/3, TLS 1.0/1.1) disabled at the application or framework level when configurable.
- **Cipher suites** — Restrict to AEAD ciphers (GCM, ChaCha20-Poly1305) where configurable. RC4, NULL, EXPORT, anonymous cipher suites disabled.
- **HSTS** — `Strict-Transport-Security` header (covered in xss-csrf-frontend.md). Force HTTPS at the network level too.
- **Certificate verification** — Library default verification used; never `verify=False`, `rejectUnauthorized: false`, `InsecureSkipVerify: true` in production code.
- **Certificate pinning** — For mobile clients or sensitive integrations; document if used.
- **Mutual TLS** — Where used, verify the peer cert chain, hostname (optional, depending on identity model), and revocation handling.

## Padding and Padding Oracles

- **Block cipher padding** — PKCS#7 typical; oracle attacks possible if MAC is verified after decryption (decrypt-then-MAC). Always encrypt-then-MAC, or use AEAD modes (GCM, ChaCha20-Poly1305).
- **RSA PKCS#1 v1.5 padding** — Bleichenbacher attacks; switch to OAEP.
- **Decryption error vs. integrity error vs. format error** — Should be indistinguishable to attacker; same response, same timing. Common bug: returning different error codes leaks oracle.

## Side Channels

- **Constant-time comparison** — Required for: password verification, MAC verification, token comparison, signature verification.
- **Timing variability in cryptographic operations** — Library implementations should be constant-time; verify if rolling crypto or using non-standard library.
- **Cache-timing in lookup tables** — Unlikely in application code; relevant when implementing primitives.
- **Power / EM side channels** — Out of scope for source code audit; document if hardware threat model applies.

## Specific Pitfalls by Use Case

### Tokens (Session, API, Reset, etc.)

- 128 bits CSPRNG output, base64url-encoded.
- Never include user-controlled or guessable data in token entropy.
- Verify on use with constant-time comparison.
- Hash before storing if exposure-on-DB-read is a concern.

### Anti-CSRF Tokens

- 128 bits CSPRNG.
- Bound to session.
- Verified server-side per request, with constant-time comparison.

### Cookie Encryption / Signing

- For signed cookies: HMAC-SHA-256 with secret stored server-side, verified before trust.
- For encrypted cookies: AEAD only; never bare CBC.
- Secret rotation supported.

### Encryption-at-Rest of Sensitive Fields

- Per-field encryption with envelope encryption (data key encrypted by KMS-managed master key).
- Bind encrypted ciphertext to context (associated data) so it cannot be moved between rows or fields.

### TOTP / HOTP

- Library implementation (RFC 6238 / RFC 4226) preferred; verify time-step, drift window, replay protection.

### Hashing user-supplied content for deduplication / lookup

- SHA-256 acceptable. Salting unnecessary if the salt would prevent the lookup. Document the threat model: an attacker who can submit content can confirm whether a given content already exists.

## Library Choice

- **Avoid**: cryptography by hand, untested or unmaintained crypto libs, "rolled your own" KDF / signature / cipher.
- **Prefer**: language-blessed libraries (Python `cryptography`, JS `crypto` (Node), `@noble/*` libraries, Java JCE / BouncyCastle, .NET `System.Security.Cryptography`, Go `crypto/*` and `golang.org/x/crypto`, libsodium / NaCl bindings).
- **libsodium / NaCl** — Strong defaults; preferred when available.

## Recommendation Patterns

- Use library defaults; do not tune algorithm/mode/iteration parameters unless you understand the implications and document the rationale.
- Centralize crypto operations behind small, well-named wrapper functions (`hashPassword`, `verifyPassword`, `signPayload`). Audit the wrappers; downstream callers don't have to understand crypto.
- Move secrets to a dedicated store (KMS, secret manager) with a clear rotation policy.
- Replace any custom crypto with a battle-tested library; treat custom crypto as a finding even when it appears correct, because correctness review is impractical without specialized expertise.
- Provide migration paths for legacy data: deprecate weak algorithms with compatibility for existing data, plus a re-hash-on-login path for passwords or a re-encrypt batch job for data.
