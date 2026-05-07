# C# / .NET Security Footguns

## Scope

C# on .NET (Core / 5/6/7/8/9), legacy .NET Framework. ASP.NET Core, ASP.NET MVC, Blazor, WPF/WinForms. Cross-reference `deserialization.md` (BinaryFormatter, Newtonsoft.Json), `injection.md` (Process.Start), `crypto.md` (System.Security.Cryptography).

## Code Execution Sinks

### `System.Reflection`

- `Type.GetType(userInput)`, `Assembly.Load(userBytes)` — Load arbitrary types/assemblies; combined with `Activator.CreateInstance` and method invocation, full RCE.
- Allowlist type names; never derive from input.

### `CSharpCodeProvider`, `Roslyn` Scripting

- `CSharpCodeProvider.CompileAssemblyFromSource(...)` — Compile and run user-controlled C# code; full RCE.
- Roslyn `CSharpScript.EvaluateAsync(...)` — Same.

### `Process.Start`

- `Process.Start(string fileName)` — Single-string form is shell-like (`cmd.exe /c` semantics on Windows); dangerous with user input.
- `Process.Start(ProcessStartInfo)` with `Arguments` — User input concatenated into Arguments still passes through shell-style argument parsing; verify.
- `ProcessStartInfo.UseShellExecute = true` — Uses OS shell (Windows: shell-execute; URL handlers, file association). Disable for executing programs.
- Best: `Process.Start(new ProcessStartInfo { FileName = "cmd", ArgumentList = { ... } })` — `ArgumentList` (.NET 6+) provides per-arg quoting on Windows.

### `Eval`-Style

- VBScript / JScript embedding (legacy COM) — Execute scripts; RCE.

### `Razor` Template Compilation

- Compiling Razor templates from user input is RCE.

## Deserialization

Covered in `deserialization.md`. .NET-specific:

- **`BinaryFormatter.Deserialize`** — Officially deprecated by Microsoft. Critical RCE class.
- **`SoapFormatter`, `NetDataContractSerializer`, `LosFormatter`, `ObjectStateFormatter`** — Same risk class.
- **`JavaScriptSerializer`** with `SimpleTypeResolver` — Type-name handling; RCE.
- **`Newtonsoft.Json`** with `TypeNameHandling != None` — RCE via `$type` field.
- **`System.Text.Json`** — Safer; no type-name handling. Default for new code.
- **XML**: `XmlSerializer` is generally safe; `DataContractSerializer` with `KnownTypes`; `XmlReader` settings should set `DtdProcessing = Prohibit`.

## SQL / DB

### ADO.NET

- `SqlCommand("..." + x)` — Concat; injection.
- `SqlCommand("...", connection)` + `Parameters.AddWithValue("@p", x)` — Parameterized; safe.
- `Parameters.Add` with explicit type — Slightly safer than `AddWithValue` (better type handling).
- Dynamic ORDER BY — Allowlist column names.

### Entity Framework / EF Core

- `ctx.Users.Where(u => u.Name == x)` — LINQ; safe (parameterized).
- `ctx.Users.FromSqlRaw("..." + x)` — Concat injection.
- `ctx.Users.FromSqlInterpolated($"... {x}")` (EF Core 3.0+) — Tagged interpolation; safe.
- `ctx.Database.ExecuteSqlRaw("..." + x)` — Concat; injection.
- `ctx.Database.ExecuteSqlInterpolated(...)` — Safe.

### Dapper

- `connection.Query<T>("... @p", new { p = x })` — Safe.
- `connection.Query<T>("..." + x)` — Concat; injection.

## XML / XXE

- `XmlReader` / `XmlReaderSettings`:
  - `DtdProcessing = DtdProcessing.Prohibit` — Disable DTD; prevents XXE.
  - `XmlResolver = null` — Prevents external resource resolution.
- `XmlDocument`:
  - `.XmlResolver = null` — Same.
  - In .NET Framework 4.5.1+ default is null; older default was permissive.
- `XmlSerializer` — Generally safe (no arbitrary type construction without explicit `KnownTypes`).
- `XmlTextReader` — Older API; verify settings.
- `XPathDocument` — Verify `XmlReaderSettings` passed.
- `XslCompiledTransform` — Compiled XSLT; if XSLT source is user-controlled, code execution surface.

## Web Frameworks

### ASP.NET Core

- **Model Binding / Mass Assignment** — Default model binding fills all fields of a class. Use specific DTOs, [Bind(Include = ...)], or [JsonIgnore] on sensitive fields.
- **Anti-Forgery** — `[ValidateAntiForgeryToken]` on POST/PUT/DELETE actions; or use `[AutoValidateAntiforgeryToken]` globally.
- **Authentication** — `[Authorize]` attribute; verify routes covered.
- **Authorization** — Policy-based; verify policies cover all sensitive actions.
- **Cookie Auth** — Cookie flags via `CookieOptions`: `Secure`, `HttpOnly`, `SameSite`.
- **JWT Bearer Auth** — `JwtBearerOptions.TokenValidationParameters` — Verify `ValidateIssuer`, `ValidateAudience`, `ValidateLifetime`, `ValidateIssuerSigningKey`, `ValidIssuer`, `ValidAudience`, `IssuerSigningKey`.
- **Data Protection** — `IDataProtectionProvider` for encrypted cookies / tokens; verify configuration.
- **CORS** — `services.AddCors` with explicit policy; never `AllowAnyOrigin` with `AllowCredentials`.
- **Developer Exception Page** — `app.UseDeveloperExceptionPage()` — Stack traces with code; only in development.
- **HSTS** — `app.UseHsts()` for production.
- **HTTPS Redirection** — `app.UseHttpsRedirection()`.
- **Custom Headers** — `Content-Security-Policy`, etc., via `app.Use(ctx => ...)`.

### ASP.NET MVC (Framework)

- Same concepts; AntiForgery, Authorize, model binding.
- `<customErrors mode="Off">` in `web.config` — Stack traces to users; never in prod.
- `web.config` with secrets — Verify encryption (`aspnet_regiis -pe`).

### Blazor

- Server-side: SignalR-bound, server-rendered; standard ASP.NET Core auth applies.
- WebAssembly: client-side only — code shipped to browser; never put secrets here.
- `@((MarkupString)userInput)` — Renders raw HTML; XSS if user-controlled.

### WCF, Web API (Framework)

- WCF: Custom serialization formats; verify same rules as `deserialization.md`.
- Web API: Model binding similar to MVC.

## Templating

### Razor

- `@variable` — Auto-escapes HTML.
- `@Html.Raw(variable)` — Disables escape; XSS if user input.
- `@(new HtmlString(variable))` — Same.
- Razor template *compilation* from user input is RCE.

### T4 Templates

- Compiled at build time; not runtime SSTI risk for application; verify build pipeline doesn't take user input.

## Crypto

### `System.Security.Cryptography`

- `MD5.Create()`, `SHA1.Create()` — Not for security purposes.
- `SHA256.Create()`, `SHA512.Create()` — General hashing.
- `HMACSHA256` — HMAC; use `CryptographicOperations.FixedTimeEquals` (.NET Core 2.1+) for constant-time MAC compare.

### Symmetric Encryption

- `AesGcm` (.NET Core 3.0+) — AEAD; preferred.
- `AesCcm` — AEAD.
- `Aes` with default mode is CBC; requires HMAC for integrity (encrypt-then-MAC pattern).
- `RijndaelManaged` — Older; same as Aes.
- `DESCryptoServiceProvider`, `TripleDESCryptoServiceProvider` — Weak; legacy.

### Asymmetric

- `RSACryptoServiceProvider` — Older; PKCS#1 v1.5 default.
- `RSA` (modern) — Use `RSAEncryptionPadding.OaepSHA256` for encryption; `RSASignaturePadding.Pss` for signatures.
- `ECDsa`, `ECDiffieHellman` — Modern curve-based.

### Random

- `RandomNumberGenerator.Create()` / `RandomNumberGenerator.GetBytes(...)` — CSPRNG.
- `RandomNumberGenerator.GetInt32(...)` (.NET 6+) — Convenient.
- `Random` class — NOT cryptographic; never for tokens.

### Password Hashing

- `Rfc2898DeriveBytes` (PBKDF2):
  - `new Rfc2898DeriveBytes(pw, salt, 600000, HashAlgorithmName.SHA256)` — Modern iteration count.
  - Older code with low iterations (1000) — Finding.
- ASP.NET Core Identity uses PBKDF2 by default (10000 iterations historically; verify cadence).
- `BCrypt.Net-Next` (third-party) — Bcrypt; cost ≥ 12.
- `Konscious.Security.Cryptography.Argon2` (third-party) — Argon2.
- Never `MD5(password)`, `SHA256(password)` alone for passwords.

### Constant-Time Comparison

- `CryptographicOperations.FixedTimeEquals(a, b)` (.NET Core 2.1+) — Use for tokens / MACs.
- Older: manual implementation needed; `==` and SequenceEqual leak timing.

### TLS

- `ServicePointManager.ServerCertificateValidationCallback = (sender, cert, chain, errors) => true` — Disabled validation; finding.
- `HttpClientHandler.ServerCertificateCustomValidationCallback = ...` (per-handler) — Same risk.
- `SecurityProtocol` — Modern .NET defaults to OS choice (good); explicit `SslProtocols.Tls12 | Tls13`.

## Network

### `HttpClient`, `WebClient`, `HttpWebRequest`

- SSRF risks; covered in `ssrf-redirect-url.md`.
- `HttpClient.MaxResponseContentBufferSize` — Limit response size; without it, memory DoS possible.
- Redirect handling — set `HttpClientHandler.AllowAutoRedirect = false` when per-redirect validation is required.

### `WebRequest.Create(url)`

- Older API; same SSRF concerns.

## File Handling

- `Path.Combine(BASE, user)` — Doesn't prevent `..` or absolute paths; canonicalize via `Path.GetFullPath` and verify prefix.
- `Path.GetFullPath` — Canonicalizes; verify result starts with intended root.
- `File.Open*` with user-controlled path — Path traversal.
- `ZipArchiveEntry.FullName` — Manual canonicalization; Zip Slip in older APIs.
- `Path.GetTempFileName` — Predictable; better: `Path.GetTempPath` + GUID.

## Common Library Footguns

### `Newtonsoft.Json`

- `TypeNameHandling.All / Auto / Objects / Arrays` — RCE via gadgets.
- Use `System.Text.Json` for new code; `TypeNameHandling.None` if Newtonsoft must remain.
- `JsonSerializerSettings.SerializationBinder` — Custom binder restricting types; required for any TypeNameHandling != None.

### `System.Text.Json`

- Safer by default; no type-name handling.
- `JsonSerializer.Deserialize<dynamic>` — Use concrete DTOs.

### `Dapper`

- See SQL section; tagged-interpolated queries are safe; concat is not.

### Identity

- ASP.NET Core Identity — `PasswordOptions` defaults are reasonable but verify.
- `UserManager.PasswordHasher` — Default is `PasswordHasher<TUser>` using PBKDF2.
- Lockout policy — `LockoutOptions`; verify reasonable.

### SignalR

- Hub authorization via `[Authorize]` attribute on hub or method.
- CORS for browser clients; verify.

### Identity Server / Duende

- OAuth/OIDC provider; large surface; not specifically a footgun but verify configuration matches OAuth checklist.

### `appsettings.json` and User Secrets

- `appsettings.Development.json` not deployed to production; secrets in `appsettings.Production.json` if present must be encrypted or sourced from KeyVault/Secrets Manager.
- User Secrets (development): not committed; verify `.gitignore`.

## .NET Framework Specifics

- **`web.config`** — Connection strings encrypted with `aspnet_regiis -pe`.
- **`<customErrors mode="On">`** — Generic errors; never `Off` in prod.
- **`<httpRuntime requestValidationMode>`** — Default validates request for "dangerous" patterns; not a primary defense, but disabling intentionally requires care.
- **Forms Authentication** — `<authentication mode="Forms">` configuration; verify cookie flags, timeout.
- **ViewState** — Encrypted/MAC'd by default in modern Framework; verify `viewStateEncryptionMode`, `enableViewStateMac`.

## Common Findings Patterns

- `BinaryFormatter.Deserialize(stream)` from untrusted — RCE.
- `JsonConvert.DeserializeObject<object>(json, new JsonSerializerSettings { TypeNameHandling = TypeNameHandling.All })` — RCE.
- `Process.Start("cmd /c " + userInput)` — Command injection.
- `new SqlCommand("SELECT ... " + userInput, conn).ExecuteReader()` — SQL injection.
- `Type.GetType(userClassName)` flowing into `Activator.CreateInstance` — Reflective instantiation.
- `Html.Raw(userContent)` in Razor — XSS.
- Permissive CORS: `AddCors(...AllowAnyOrigin().AllowCredentials()...)` — Browser rejects but signals intent.
- `ServerCertificateValidationCallback = ... => true` — TLS bypass.
- `new Random().Next()` for tokens.
- `MD5(password)` for password storage.
- `app.UseDeveloperExceptionPage()` in production.
- `XmlReaderSettings { DtdProcessing = DtdProcessing.Parse }` on untrusted XML.

## Recommendation Patterns

- Use `RandomNumberGenerator` for any randomness in security context.
- Use `Rfc2898DeriveBytes` (PBKDF2) with 600,000+ iterations, or third-party Argon2 / Bcrypt for passwords.
- Use `CryptographicOperations.FixedTimeEquals` for MAC / token comparison.
- Use `AesGcm` for symmetric encryption (or library wrappers; `AesCcm` alternative).
- Use `System.Text.Json` for new code; never enable `TypeNameHandling` on untrusted input with Newtonsoft.
- Use parameterized queries via ADO.NET, EF Core, Dapper; never concat into SQL.
- For ASP.NET Core: explicit `[Authorize]`, anti-forgery on POST routes, strict CORS, secure cookie options, HSTS.
- Configure `XmlReaderSettings { DtdProcessing = Prohibit, XmlResolver = null }` for any XML on untrusted input.
- Disable `BinaryFormatter` / `SoapFormatter` / similar; migrate to JSON.
- Pin dependencies; use `dotnet list package --vulnerable` and SCA tools in CI.
- Move secrets to Azure Key Vault / AWS Secrets Manager / HashiCorp Vault.
