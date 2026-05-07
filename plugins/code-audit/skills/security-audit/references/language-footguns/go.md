# Go Security Footguns

## Scope

Go (Golang) on the server side. Cross-reference deserialization.md (gob, JSON, XML), injection.md (os/exec), crypto.md (crypto/*).

Go's standard library is generally well-designed for security; many footguns come from idioms that are correct-looking but subtly unsafe, or from third-party packages.

## Code Execution Sinks

### `os/exec`

- `exec.Command(name, args...)` — Argv-based; safe for user input as `args`. The command name itself should not be user-controlled.
- `exec.Command("sh", "-c", userCommand)` — Shell-interpreted; injection. Avoid.
- `bytes.Buffer` for stdin / stdout — verify size handling.

### `os.Exec`, `syscall.Exec`

- Direct exec calls; same considerations.

### `text/template` and `html/template`

- `template.Parse(userInput)` — SSTI when input is user-controlled (template parsing produces Go templates which can call functions).
- `html/template` auto-escapes for HTML context; switching contexts via `template.HTML(userInput)` disables escape — XSS.

## SQL / DB

### `database/sql`

- `db.Exec("..." + x)`, `db.Query("..." + x)`, `fmt.Sprintf` building queries — Concat injection.
- `db.Exec("... ?", x)` (MySQL) / `db.Exec("... $1", x)` (Postgres) — Parameterized; safe.
- Dynamic ORDER BY / column names — Allowlist; still text concatenation.

### `gorm`

- `db.Where("name = ?", x)` — Safe.
- `db.Where("name = '" + x + "'")` — Concat injection.
- `db.Raw("..." + x)` — Concat; injection.
- `db.Exec(query, args...)` — Bind args; safe.
- `gorm.Expr(...)` for SQL fragments — verify usage doesn't include user input.

### `sqlx`, `pgx`, `bun`

- All support parameterized queries; verify user input is bound, not concatenated.

## XML / XXE

- `encoding/xml` — Default does not process external entities; safe by default. Verify no custom XML library used.
- `etree`, `xmlquery` — Third-party; verify entity handling.

## JSON

- `encoding/json`:
  - `json.Unmarshal` into typed struct — Safe (no arbitrary type instantiation).
  - `json.Unmarshal` into `interface{}` — Safe; produces map/slice/primitives only.
  - `json.RawMessage` — Defers parsing; safe by itself.
- Mass assignment via `json:"-"` tag on sensitive fields — Verify sensitive fields are tagged to be excluded from binding (or use separate input DTOs).
- `decoder.UseNumber()` — Numeric precision; not security but may affect.
- `decoder.DisallowUnknownFields()` — Reject unknown JSON fields; defense against unexpected data injection.
- `*json.Encoder` writing to response — Default does HTML-escape; for JSON-in-script-tag contexts, explicit handling required.

## YAML

- `gopkg.in/yaml.v3` — Default load is safe for basic types.
- `yaml.UnmarshalStrict` — Stricter; rejects unknown fields.
- Less risky than Python YAML.

## Web Frameworks

### `net/http` (built-in)

- `http.HandleFunc("/path", handler)` — Verify auth applied via wrapping middleware.
- `r.URL.Query().Get(...)`, `r.PostFormValue(...)`, `r.Header.Get(...)` — Sources.
- `http.Redirect(w, r, url, code)` with user input — Open redirect.
- `http.ServeFile(w, r, path)` with user input — Path traversal.
- `http.FileServer(http.Dir(...))` — Default behavior follows symlinks; restrict if necessary.
- `http.Client` — `Transport.TLSClientConfig.InsecureSkipVerify = true` — Disables TLS verification.

### Gin, Echo, Fiber, Chi

- Body binding (`c.Bind`, `c.BindJSON`, `c.ShouldBindJSON`) — Mass assignment if struct includes sensitive fields.
- Query/param/header sources — same as net/http.
- Middleware chain — Verify auth before handler.
- CORS middleware — Default values vary; verify allowlist.

### gRPC

- `grpc.UnaryInterceptor`, `grpc.StreamInterceptor` for auth.
- TLS configuration via `credentials.NewTLS`.
- Reflection enabled in production exposes service definitions; disable.

## Crypto

### `crypto/rand`

- `rand.Read` / `rand.Int` / `rand.Reader` — CSPRNG; safe.
- `math/rand` — NOT cryptographic; never for tokens / IDs / salts. (Go 1.20 added `math/rand/v2` which still isn't cryptographic; stay with `crypto/rand`.)

### `crypto/sha*`, `crypto/md5`

- MD5, SHA-1 — Not for security purposes.
- SHA-256, SHA-512 — General hashing.
- HMAC: `crypto/hmac.New(sha256.New, key)` — verify constant-time compare via `hmac.Equal`.

### `crypto/cipher`

- `aes.NewCipher(key)` + `cipher.NewGCM(block)` — AEAD; preferred.
- `cipher.NewCBCEncrypter` — CBC mode; requires HMAC for integrity.
- ECB not in std library — good.
- IV / nonce — `crypto/rand` for fresh values; never reuse with same key in GCM.

### `crypto/tls`

- `tls.Config.InsecureSkipVerify = true` — Disables certificate validation; finding.
- `tls.Config.MinVersion = tls.VersionTLS12` (or `TLS13`) — Pin minimum.
- Custom `VerifyPeerCertificate` — Audit for shortcuts.
- `tls.Config.CipherSuites` — Not necessary to restrict in modern Go; defaults are safe.

### Password Hashing

- `golang.org/x/crypto/bcrypt` — `bcrypt.GenerateFromPassword(pw, cost)` with cost ≥ 12.
- `golang.org/x/crypto/argon2` — Direct API; use IDKey with documented parameters.
- `golang.org/x/crypto/scrypt`.
- Never `sha256(password)`.

### Subtle Comparison

- `subtle.ConstantTimeCompare(a, b)` — For MAC / token comparison; `bytes.Equal` is not constant-time.

## Network / Web Client

### `net/http`

- `http.Get(userURL)` — SSRF; covered in ssrf-redirect-url.md.
- Default `http.Client` follows redirects up to 10; can chain to internal addresses.
- `http.Client.CheckRedirect` — Custom function to validate redirects.
- Default timeout: none; sets `http.Client.Timeout` to bound.

### `net.Dialer`

- `dialer.Control` — Hook to validate the connect address (post-DNS-resolution); useful for SSRF defense.

### `net.LookupHost`, `net.ResolveIPAddr`

- DNS-side effect on user input; reveals existence to DNS attacker.

## File Handling

- `filepath.Join(BASE, user)` — Doesn't prevent `..`; canonicalize via `filepath.Clean` and prefix-check.
- `filepath.Clean` collapses `..` but may produce a path outside intended root if base is relative; combine with absolute path resolution.
- `os.OpenFile(path, ...)` — Path-traversal risks.
- `archive/zip`, `archive/tar` — Manual canonicalization for entry paths; Zip Slip.
- `os.CreateTemp` — Safe; replaces older `ioutil.TempFile`.
- File permissions — `os.OpenFile(path, flags, 0644)` — Verify mode appropriate for sensitive content.

## Concurrency

- Race conditions on shared maps — Go's runtime detects with `-race` flag in tests; production race conditions still exist if not tested.
- `sync.Mutex` correctness — Forgot to lock, double-unlock; not security-critical alone unless on auth state.
- `context.Context` propagation — Auth context typically passed via context; verify it propagates through middleware.
- Goroutine leaks — Resource exhaustion DoS; verify goroutine lifecycle.

## Common Library Footguns

### gRPC

- Reflection (`reflection.Register`) in prod — Exposes service definitions.
- TLS not enabled (Insecure transport) — Cleartext.
- Authentication via interceptors — Verify all RPCs covered.

### `gorilla/mux`, `chi`

- Routing patterns — Verify auth middleware applied.
- `mux.Vars(r)` — Path params; sources.

### `jwt-go` (deprecated) / `golang-jwt/jwt`

- `jwt.Parse` with custom keyfunc — Implement allowlist for `alg` to avoid `none` and confusion.
- `jwt.ParseWithClaims` — Same.
- Verify `keyfunc` checks `token.Method`.

### `gopkg.in/yaml.v2` vs v3

- v2 has minor differences; verify behavior; v3 generally preferred.

### `sqlx`

- `db.Get(&dest, query, args...)` — Bind args; safe.
- Verify struct tags for field exposure.

### `viper`, `cobra`, `flag`

- Config / flag parsing; reading from env, files. Treat config sources per Threat Model Brief.

### `go-redis`, `mongo-go-driver`

- Redis: `client.Eval` for Lua; Lua server-side scripting can be a risk if user input becomes script source.
- MongoDB: BSON marshaling; verify type expectations to avoid operator injection.

## Go-Specific Idioms

### Interface Assertions

- `if v, ok := i.(SomeType); ok` — Not security-relevant alone, but if assertion is incorrect, panic in production = DoS.
- `panic` in handlers — Recover middleware should handle; verify.

### Error Handling

- Ignored errors — Common Go bug; missed `if err != nil` allows continuation in error path. Auth checks that don't handle their error result may fail open.

### Nil Pointers / Maps

- Dereferencing nil pointer — Panic; DoS.
- Reading from nil map — Returns zero value (safe); writing to nil map — panic.

### Generics (Go 1.18+)

- Compile-time generics; no runtime impact; same security questions as concrete types.

## Common Findings Patterns

- `exec.Command("sh", "-c", userInput)` — Command injection.
- `db.Exec("SELECT * FROM users WHERE id = " + id)` — SQL injection.
- `template.HTML(userInput)` flowing into html/template render — XSS.
- `tls.Config{InsecureSkipVerify: true}` — TLS bypass.
- `math/rand.Intn(...)` for generating tokens.
- `bcrypt.GenerateFromPassword(pw, 4)` — Cost too low.
- `c.Bind(&User{})` where User has `IsAdmin` field — Mass assignment.
- `r.URL.Query().Get("redirect")` flowing into `http.Redirect` without validation — Open redirect.
- `filepath.Join(BASE, r.URL.Query().Get("file"))` without prefix check — Path traversal.
- `http.Get(r.URL.Query().Get("url"))` — SSRF.
- `jwt.Parse(token, keyfunc)` without alg check in keyfunc — Alg confusion.

## Recommendation Patterns

- Use parameterized queries via `database/sql` or query builders.
- Use `crypto/rand` for tokens; never `math/rand`.
- Use `bcrypt` (cost ≥ 12) or `argon2` for passwords.
- Use `subtle.ConstantTimeCompare` / `hmac.Equal` for MAC comparison.
- Use `html/template` (auto-escape) for HTML; never `text/template` for HTML output.
- Use struct tags `json:"-"` to exclude sensitive fields from JSON binding; or use separate input DTOs.
- Set `http.Client.Timeout` and validate redirect targets.
- Set `tls.Config.MinVersion` to TLS 1.2; never `InsecureSkipVerify`.
- Disable gRPC reflection in production.
- For JWT: explicit `alg` allowlist in keyfunc.
- For DNS-resolution-sensitive ops: pin to resolved IP and validate before connecting.
