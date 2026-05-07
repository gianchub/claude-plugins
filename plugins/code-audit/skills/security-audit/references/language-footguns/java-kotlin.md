# Java / Kotlin Security Footguns

## Scope

Java and Kotlin on JVM; Spring Boot, Jakarta EE / Java EE, Micronaut, Quarkus. Cross-reference deserialization.md (ObjectInputStream, Jackson, SnakeYAML, XStream), injection.md (Runtime.exec), crypto.md (JCE).

## Code Execution Sinks

### `Runtime.getRuntime().exec(String)`

- Single-string form is shell-tokenized; user input within is injection.
- `Runtime.exec(String[])` — Argv form; safer but still verify positional usage.
- `ProcessBuilder(List<String>)` — Argv form; same considerations.
- Never construct command via concatenation.

### `Class.forName(name)` and reflective instantiation

- `Class.forName(userControlled)` — Loads arbitrary class; combined with `newInstance()` and `getMethod(...).invoke()` allows arbitrary call.
- Allowlist class names; never derive from input.

### `ScriptEngine` (JSR-223)

- Nashorn (deprecated/removed), Rhino, GraalVM JS — Execute JS from string; with user input, RCE.
- JEXL, MVEL, OGNL, SpEL (Spring Expression Language) — All can RCE on user-controlled expressions.
- **OGNL injection** in Apache Struts (CVE-2017-5638 history) — Critical-class.
- **SpEL injection** in Spring `@Value`, expression parsing on user input — RCE.

### Native Code Loading

- `System.loadLibrary(name)` with user-controlled name — Load arbitrary library.

### Java Deserialization

- `ObjectInputStream.readObject()` — Covered in deserialization.md. RCE via gadget chains.
- `XMLDecoder.readObject()` — Reads XML serialized form; same RCE class.
- **Configure `ObjectInputFilter`** (Java 9+) per call site with strict allowlist.

## SQL / DB

### JDBC

- `Statement.executeQuery("..." + x)` — Concat injection. Use `PreparedStatement` with `?` placeholders.
- `PreparedStatement.setString(1, x)` — Safe.
- Dynamic ORDER BY: still concat-vulnerable; allowlist column names.

### JPA / Hibernate

- `EntityManager.createQuery(jpql)` — JPQL is parameterized when used correctly; concat into `jpql` is injection.
- `EntityManager.createNativeQuery(sql)` — Raw SQL; same parameterization rules.
- `@Query(nativeQuery=true)` (Spring Data) — Native SQL; verify parameter binding.
- Criteria API — Type-safe; safer.

### MyBatis

- `${param}` in mapper XML — Direct substitution; injection.
- `#{param}` — Parameter binding; safe.

### jOOQ

- `DSL.field("name = " + x)` — Concat; injection.
- `DSL.field("name = ?", x)` — Bound; safe.

## XML / XXE

- `DocumentBuilderFactory` defaults vary by JVM and version. Set:
  - `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`
  - `setFeature("http://xml.org/sax/features/external-general-entities", false)`
  - `setFeature("http://xml.org/sax/features/external-parameter-entities", false)`
  - `setExpandEntityReferences(false)`
  - `setXIncludeAware(false)`
- `SAXParserFactory`, `XMLInputFactory`, `Transformer`, `Validator`, `SchemaFactory` — All require similar hardening per call site.
- `XMLReader` with default features — Often DTD-enabled.
- Apache Commons XML libraries — Verify version and configuration.

## Web Frameworks

### Spring Framework / Spring Boot

- **Mass assignment via `@RequestBody`** — DTO with all fields exposed; use `@Valid` with strict DTOs that exclude sensitive fields.
- **`@PathVariable` / `@RequestParam`** — Sources; trace.
- **`@PreAuthorize`, `@PostAuthorize`, `@Secured`** — Method-level authz; verify expressions evaluate correctly.
- **CSRF** — Spring Security defaults to CSRF on for browser-facing apps; verify not disabled.
- **Actuator endpoints** — `/actuator/*` exposes health, metrics, env. Default in Spring Boot 2+ exposes `health` and `info`; older versions exposed everything. Verify production restricted.
- **`@CrossOrigin(origins="*")`** — Permissive CORS.
- **`server.error.include-stacktrace=always`** — Debug; never in prod.
- **SpEL injection** — `@Value("#{...}")` with user input, or expression evaluation in business code on user input.
- **Spring4Shell-class** — Older Spring Framework versions on JDK 9+; verify version.
- **Spring Security debug filter** — Logs full requests; never in prod.
- **Filter chain ordering** — Custom filters relative to `SecurityFilterChain`; verify auth happens before sensitive logic.

### Jakarta EE / Servlets

- `request.getParameter`, `request.getHeader`, `request.getCookies` — Sources.
- `response.setHeader(name, value)` — Header injection if value contains CRLF.
- `request.getRequestDispatcher(userPath).forward(...)` — Path traversal / forced forwarding.
- `ServletContext.getResource`, `getResourceAsStream` — Path-based access.

### Micronaut, Quarkus

- Same patterns as Spring; verify framework-specific authz declaration.

## Templating

### Thymeleaf

- Default escaping is XML/HTML safe.
- `[(unescaped)]` syntax — Disables escaping.
- Inline `[[${expr}]]` — XSS if `expr` is user-controlled HTML.
- Server-side template injection if template source is user-controlled.

### JSP

- `<%= expr %>` — Unescaped output by default; XSS.
- `<c:out value="${x}">` — Escapes (better).
- Scriptlets `<% ... %>` — Java code in template; SSTI if template source from user.

### FreeMarker, Velocity

- SSTI if template source user-controlled; sandbox restrictions exist but historic CVEs.

### Mustache, Handlebars (Java ports)

- Generally safer; logic-less templates.

## Crypto

### JCE

- `MessageDigest.getInstance("MD5")`, `"SHA-1"` — Not for security purposes.
- `Cipher.getInstance("AES")` — Default mode is ECB on Oracle JDK; explicit mode required: `Cipher.getInstance("AES/GCM/NoPadding")` or `"AES/CBC/PKCS5Padding"` (CBC requires HMAC for integrity).
- `KeyGenerator` / `SecureRandom` — `SecureRandom.getInstance("SHA1PRNG")` is older and on some JDKs deterministic; prefer `SecureRandom.getInstanceStrong()` or default `new SecureRandom()`.
- `Random` (java.util.Random) — NOT cryptographic; never for tokens.
- `MessageDigest.isEqual` (recent JDK) — Constant-time comparison; older JDK versions did not implement constant-time. Verify version or use a known-constant-time helper.
- `MessageDigest.update + digest` — Non-constant-time `==` comparison; use `MessageDigest.isEqual`.

### Password Hashing

- bcrypt: `BCrypt.hashpw(password, BCrypt.gensalt(12))` (Spring Security or jBCrypt).
- Argon2: `argon2-jvm` library; preferred.
- PBKDF2: `SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")` with high iteration count.

### TLS

- `SSLContext` — Disabling certificate verification (custom `TrustManager` accepting all certs) — disclaimer findings.
- `HttpsURLConnection.setDefaultHostnameVerifier((host, session) -> true)` — Disables hostname verification.
- Older TLS versions enabled — verify only TLS 1.2 / 1.3.
- Cipher suite selection — typically library default; verify if customized.

## Network

### `URL.openStream`, `HttpURLConnection`, `HttpClient` (Java 11+)

- SSRF risks; covered in ssrf-redirect-url.md.
- `setSSLSocketFactory` with permissive context.
- Redirect-following defaults may need restriction.

### Apache HttpClient

- Custom `HttpClient` with permissive trust strategy.

### OkHttp

- Custom `TrustManager`, `HostnameVerifier`; verify defaults.

## File Handling

- `Paths.get(BASE).resolve(user).normalize()` — Normalizes; verify `startsWith(BASE)` after.
- `new File(BASE, user)` — No traversal protection alone; canonicalize via `getCanonicalPath`.
- `ZipInputStream` / `ZipFile` — Manual canonicalization for entry paths; Zip Slip.
- `Files.copy`, `Files.move` — User input in path components; same risks.

## Spring-Specific Detail

### Spring Security

- `WebSecurityConfigurerAdapter` (older) / `SecurityFilterChain` (5.7+) — Configuration code; verify rule order.
- `permitAll()` / `anonymous()` — Explicit permission; verify what's exempted.
- `csrf().disable()` — Often done for stateless APIs; verify the API is genuinely stateless and bearer-token-authenticated.
- `httpBasic()` — Acceptable for service APIs; verify TLS.
- `formLogin()` — Verify session config and CSRF interaction.
- `oauth2Login()` — Configure correctly; verify redirect URI allowlist.
- Authorization expressions — `hasRole("ADMIN")` vs `hasAuthority("ROLE_ADMIN")` — easy to miss the prefix; tests should verify.
- Method security — `@EnableGlobalMethodSecurity(prePostEnabled = true)` — required to use `@PreAuthorize`.

### Actuator Endpoints

- `management.endpoints.web.exposure.include` — Defaults to `health,info` in Boot 2+; older versions exposed all.
- Sensitive endpoints (`env`, `heapdump`, `threaddump`, `metrics`, `mappings`) — Restrict access; auth required.

### Spring Cloud Config

- `/encrypt` and `/decrypt` endpoints when configured — Verify auth.
- Config server source repos — Don't include secrets in plain.

## Common Library Footguns

### Apache Commons

- `commons-collections` 3.x with `InvokerTransformer` — Gadget for deserialization RCE.
- `commons-text` `StringSubstitutor` `${script:...}`, `${url:...}` — RCE / SSRF (CVE-2022-42889).
- `commons-fileupload` — Historic CVEs.

### Log4j

- log4j 2.x < 2.17.0 — Log4Shell (`${jndi:...}`); critical.
- log4j 1.x — EOL, vulnerabilities.
- Logback — Less impacted but verify version.

### Jackson

- `enableDefaultTyping`, `@JsonTypeInfo(use=Id.CLASS)` — Polymorphic with class info; allows attacker to instantiate arbitrary types → RCE via gadgets.
- Use `Id.NAME` with explicit `@JsonSubTypes`.
- `PolymorphicTypeValidator` required for polymorphic on untrusted JSON.

### Hibernate

- HQL injection via concat; same fix (parameter binding).
- `Session.createNativeQuery` with concat.

### Tomcat / Jetty / Undertow

- Default error pages may leak server version.
- HTTP/2 features enabled — verify CVE coverage.
- Connector configurations — verify TLS, header limits, etc.

### Bouncy Castle

- Crypto provider; verify version.
- Older versions had timing-side-channel issues.

## Kotlin-Specific

### Coroutines

- `runBlocking` in request paths — DoS surface.
- `GlobalScope.launch` for fire-and-forget — Same risks as unmanaged threads; auth context may not propagate.

### `let`, `apply`, `also`, `run`

- Idiomatic but doesn't change security; review same operations as Java.

### `data class` and Mass Assignment

- Constructor accepting all fields; if exposed via JSON binding, mass assignment risk.
- Use sealed DTOs and explicit field allowlists.

### Kotlin Serialization (`kotlinx.serialization`)

- Generally safe; verify polymorphic configurations.

### Ktor

- Ktor framework: routes, auth (JWT, session, OAuth), CORS configuration. Same patterns as Spring at a higher level.

## Common Findings Patterns

- `Runtime.exec("cmd " + userInput)` — Command injection.
- `Class.forName(userInput).newInstance()` — RCE via reflection.
- `new ObjectInputStream(stream).readObject()` on untrusted — RCE.
- `XMLDecoder.readObject()` — RCE.
- `MessageDigest.getInstance("MD5")` for password hashing.
- `new Random().nextInt()` for tokens.
- `Cipher.getInstance("AES")` (default ECB).
- `csrf().disable()` on stateful Spring app.
- `@CrossOrigin(origins = "*")` permissive CORS.
- Hardcoded secrets in `application.properties` / `application.yml`.
- `ObjectMapper.enableDefaultTyping()` with untrusted JSON.
- `request.getParameter(...)` flowing into Statement concat without prepared.
- log4j 2.x < 2.17.0 in dependencies.
- `(host, session) -> true` HostnameVerifier.

## Recommendation Patterns

- Use prepared statements universally; never concat into JDBC/JPQL/HQL.
- Configure `ObjectInputFilter` strictly; better, eliminate Java native serialization.
- Use modern crypto (`AES/GCM/NoPadding`, SHA-256+, argon2/bcrypt for passwords).
- Use `SecureRandom` for tokens; never `Random`.
- Configure XXE-mitigated XML parsers per call site.
- Use Spring Security with declarative authz; verify configuration in tests.
- Disable Spring Boot Actuator's sensitive endpoints in production or restrict to internal network.
- Pin dependencies via Maven / Gradle; use OWASP Dependency-Check or Snyk in CI.
- For SpEL / OGNL / JEXL: never evaluate expressions from user input.
- For deserialization: prefer JSON via Jackson with strict types; never accept Java native serialization from untrusted sources.
