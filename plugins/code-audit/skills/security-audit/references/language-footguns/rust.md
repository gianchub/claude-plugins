# Rust Security Footguns

## Scope

Rust on the server side. Memory-safe by design; the unsafe operations are concentrated in `unsafe` blocks, FFI, and specific dependency choices. Cross-reference deserialization.md (serde-related), injection.md (process spawning), crypto.md.

Rust eliminates entire vulnerability classes (buffer overflows, use-after-free, double-free) at the language level for safe code. The remaining surface is application-level (logic, auth, injection at sinks) plus `unsafe` blocks and dependencies.

## `unsafe` Blocks

`unsafe` blocks bypass Rust's safety guarantees. Inside, the developer takes responsibility for invariants normally checked by the compiler.

### What to Audit

- **Pointer dereferencing** — `*raw_ptr` requires the pointer to be valid, aligned, and pointing to a valid initialized value.
- **`transmute`** — Reinterpret bits as another type; many invariant requirements.
- **Calling `unsafe fn`** — FFI, intrinsics; verify documented preconditions are met.
- **`unsafe impl`** — Manual `Send`/`Sync` implementations; if incorrect, data race surface.
- **`union` field access** — Reads of union fields are unsafe.

### Heuristics

- Search for `unsafe` blocks; for each, verify the safety comment (`// SAFETY: ...`) is plausible and the invariants hold.
- `unsafe` block without a safety comment is a code-quality finding worth noting; in security context, an unverified `unsafe` invariant is potential memory unsafety.
- Lints: `clippy::missing_safety_doc`, `clippy::undocumented_unsafe_blocks`. Verify enabled.

### Common Pitfalls

- `Vec::set_len(n)` extending without initializing — Reading uninitialized memory.
- `slice::from_raw_parts(ptr, len)` with mismatched len.
- Dropping a value while a `&` reference still exists (only possible with `unsafe`).
- FFI returning a pointer that the caller frees with the wrong allocator.

## FFI

When calling C/C++ libraries via `extern` blocks:

- Memory ownership boundaries — Who frees what?
- Buffer length parameters — Off-by-one off-by-many.
- Strings: C strings vs. Rust `&str` (UTF-8 vs. arbitrary bytes; null terminator).
- Errno and error handling — Many C functions return error codes; verify checks.
- Shared library ABI — Version mismatches.

## Code Execution Sinks

### `std::process::Command`

- `Command::new("sh").arg("-c").arg(userInput)` — Shell-interpreted; injection. Avoid.
- `Command::new("program").args(&[arg1, arg2])` — Argv form; safer.
- Per-arg user input is generally safe with argv form, but verify positional placement.

### `eval`-like

- Rust has no built-in `eval`. Embedded scripting (`mlua`, `rlua`, `rhai`, `boa-engine`) brings a scripting language; verify sandbox claims and user-input handling.

### Dynamic library loading

- `libloading::Library::new(userInput)` — Load arbitrary shared library; treat as RCE.

## SQL / DB

### `sqlx`

- Compile-time-checked queries (`sqlx::query!(...)`) — Macro-checked; safe.
- `sqlx::query("..." + x)` — Concat; injection.
- `sqlx::query("... ?", args)` — Bind args; safe.

### `diesel`

- Type-safe DSL; query building doesn't permit injection in normal flows.
- `diesel::sql_query("..." + x)` — Concat; injection.
- `bind` for user input.

### `sea-orm`, `tokio-postgres`, `rusqlite`

- All support parameterized queries; verify user input is bound.

## Web Frameworks

### `actix-web`, `axum`, `warp`, `rocket`

- Body extraction (`web::Json<T>`, `Json<T>`, `Form<T>`) — Strongly typed; mass assignment surface limited by type definition. Audit struct definitions for sensitive fields exposed.
- Path / query / header extractors — Sources.
- Middleware for auth — Verify applied to all routes.
- CORS — Crate-specific; verify allowlist.
- Templates (e.g., `tera`, `askama`) — Auto-escape behavior; verify.

### `tonic` (gRPC)

- Reflection enabled in production exposes service definitions.
- TLS via `tonic::transport::ServerTlsConfig`; verify configured.

## Crypto

### `ring`, `RustCrypto` Crates

- `ring::aead` — AEAD primitives; preferred.
- `ring::digest` — SHA-256, SHA-512; not for passwords.
- `ring::rand::SystemRandom` — CSPRNG.
- `RustCrypto/aes-gcm`, `chacha20poly1305` — AEAD.
- `RustCrypto/sha2`, `sha3` — Hashing primitives.
- Avoid `RustCrypto/cbc` without HMAC for integrity.
- `subtle::ConstantTimeEq` — Constant-time comparison.

### Password Hashing

- `argon2` crate — Argon2id, modern parameters.
- `bcrypt` crate — Bcrypt; cost ≥ 12.
- `scrypt` crate — scrypt.
- Never bare hash (`sha256`) for passwords.

### Random

- `rand` crate `thread_rng()` — CSPRNG (since rand 0.8 default is `OsRng`-seeded ChaCha; verify version).
- `rand::random()` — Convenience using thread_rng.
- For security context, prefer explicit `OsRng` from `rand_core` or `getrandom` for direct access to the OS CSPRNG.
- `rand::rngs::SmallRng` — NOT cryptographic; never for security.
- `rand::distributions` — Uniform, etc.; combined with secure RNG is fine.

### TLS

- `rustls` — Modern, memory-safe TLS.
- `native-tls` — Wraps OS TLS (OpenSSL on Linux, Schannel on Windows, SecureTransport on macOS); platform-dependent CVE coverage.
- Disabling certificate verification — `rustls::ClientConfig` with custom verifier returning Ok always; flag.

## Network

### `reqwest`, `hyper`, `ureq`, `surf`

- SSRF risks; covered in ssrf-redirect-url.md.
- TLS verification — `reqwest::ClientBuilder::danger_accept_invalid_certs(true)` — Disabled; flag.
- Redirect handling — Default policy varies; verify.

## File Handling

- `std::fs::File::open(path)` with user input — Path traversal.
- `std::path::PathBuf::push` — Path joining; doesn't prevent `..`.
- Canonicalize: `path.canonicalize()`; verify result starts with the intended root.
- Note: `canonicalize` requires the path to exist; for paths that don't yet exist, manual normalization needed.
- Archive extraction: `zip` crate, `tar` crate — Manual canonicalization for entry paths; Zip Slip risk.
- `tempfile` crate — Safe temp files.

## Serde

- Generally safe; deserialization types are explicit.
- `serde_json::from_str::<T>(...)` — Typed; safe (no arbitrary instantiation).
- `serde_json::Value` — Untyped JSON tree; safe.
- `bincode::deserialize` — Compact binary format; can deserialize into typed structs; not arbitrary.
- `rmp-serde` (MessagePack) — Same.
- **`#[serde(deny_unknown_fields)]`** — Reject unknown fields; defense against extra-data injection.
- **`#[serde(skip)]`** / **`#[serde(skip_deserializing)]`** — Exclude sensitive fields from deserialization (mass assignment defense).
- `bincode::deserialize` configured with size limits — Without, large input causes memory DoS.

## Concurrency

- Rust's type system prevents data races at compile time for safe code.
- `Arc<Mutex<T>>` for shared mutable state; verify lock discipline.
- Async (`tokio`, `async-std`): blocking calls in async context starve runtime; not security-critical alone but relevant if blocking call is part of auth verification (timing).

## Common Library Footguns

### `tokio`

- `tokio::spawn(async move { ... })` for fire-and-forget — Verify auth context if any.
- Bounded channels for backpressure; unbounded channels for memory DoS surface.

### `actix-web`

- `App::wrap(...)` for middleware; verify ordering (auth before sensitive logic).
- `App::data(...)` for shared state; verify thread-safety.

### `axum`

- Type-safe extractors; mass assignment surface limited by struct definition.
- `axum::Router::route` ordering matters for matching.

### `rocket`

- Macro-based; routes declared with attributes.
- `#[catch(404)]`, etc. — Error handlers.

### `serde`

- `#[serde(default)]` — Provides default for missing fields; verify default is safe (e.g., not "default admin").
- `#[serde(rename_all = "camelCase")]` — Naming; not security but check for unintended exposures.

### `regex`

- ReDoS — Regex engine in `regex` crate is linear time (RE2-style); no backtracking; safer than most languages by default.
- User-supplied regex compiled with `regex::Regex::new(userInput)` — Compilation cost can be DoS surface (large alternations); set `RegexBuilder::size_limit`.

### `sqlx` Macros

- `sqlx::query!` requires database connection at compile time; verify CI has DB access or uses offline mode.

### Logging

- `tracing` and `log` — Structured logging; same concerns as other languages re: sensitive data.
- `dbg!` macro — Debug; never in production code paths.
- `eprintln!` to stderr — Verify not capturing sensitive data in container logs.

## Cargo / Crates

- `cargo audit` — Vulnerability scanning; verify in CI.
- `Cargo.lock` committed.
- `git`-source dependencies — Pin to specific commit (rev), not branch.
- `path` dependencies — Local; OK for monorepos but unusual in published crates.
- Build scripts (`build.rs`) — Execute at build; can fetch and run code; audit.
- `cargo deny` — License / source / advisory checks.

## Common Findings Patterns

- `unsafe { *raw_ptr }` without verified validity — Potential UB.
- `Command::new("sh").arg("-c").arg(format!("cmd {}", user))` — Command injection.
- `sqlx::query(&format!("SELECT ... WHERE id = {}", id))` — SQL injection.
- `reqwest::ClientBuilder::new().danger_accept_invalid_certs(true)` — TLS bypass.
- `rand::thread_rng()` used for cryptographic context — Acceptable since 0.8 default; verify.
- `sha2::Sha256::digest(password)` for password storage — Wrong; use Argon2/Bcrypt.
- Custom TLS verifier returning `Ok(...)` unconditionally — TLS bypass.
- `bincode::deserialize` without size limit on untrusted — DoS.
- Missing `#[serde(deny_unknown_fields)]` on input DTOs — Defense-in-depth.
- Hardcoded secrets in source (search for high-entropy literals).
- `unsafe` blocks without safety comments — Code review red flag, security risk if invariants slip.

## Recommendation Patterns

- Use `argon2` or `bcrypt` crates for password storage; never bare hashes.
- Use `ring::rand` or `getrandom` for randomness in security context.
- Use `subtle::ConstantTimeEq` for MAC / token comparison.
- Use `rustls` (memory-safe) over `native-tls` where possible.
- Use parameterized SQL via `sqlx::query!`, `diesel`, or query builders.
- For cross-trust-boundary deserialization, use typed DTOs with `serde(deny_unknown_fields)`.
- Audit `unsafe` blocks; require safety comments via clippy lint.
- Pin dependencies; `cargo audit` in CI; review build scripts.
- Set `RegexBuilder::size_limit` for user-supplied regex.
- For any FFI: document ownership boundaries clearly; minimize FFI surface.
