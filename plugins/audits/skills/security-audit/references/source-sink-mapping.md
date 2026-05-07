# Source/Sink Mapping

## Purpose

Enumerate every untrusted-input source and every dangerous sink in scope, so Phase 5 can trace each source to each reachable sink and evaluate validation, encoding, and authorization at each step. This is the technique that distinguishes a real security audit from a pattern-matching scan.

The map does not need to be exhaustive prose; a list of `source@file:line` and `sink@file:line` entries with the input shape and the operation suffices. Build the map per service in monorepos.

## Sources to Enumerate

A "source" is anywhere data enters the application from a less-trusted boundary. Treat every source as hostile unless proven otherwise.

### HTTP Sources

For every request handler, every input field is a separate source. Enumerate:

- **Body** — JSON, form-encoded, multipart fields. Multipart files are a separate source class (file-handling checklist).
- **Query string** — every parameter; URL-decoded values may differ from raw values.
- **Path parameters** — route variables. Often used directly in DB queries (IDOR risk).
- **Headers** — User-Agent, Referer, Accept-Language, Authorization (the credential is a source for the auth subsystem; the rest of the value is hostile after decoding), X-Forwarded-For, X-Forwarded-Host, X-Real-IP, custom `X-*` headers.
- **Cookies** — every cookie value, including session IDs (which are an internal trust artifact but become hostile if integrity is not enforced).
- **Form fields** — for SSR apps; same as body but with framework-specific binding (mass-assignment risk).
- **Multipart files** — filename, declared content-type, and content. All three are attacker-controlled.
- **WebSocket frames** — text and binary frames; identical to body for analysis purposes.
- **GraphQL** — operation name, query string, variables, fragments. Each variable is a separate source.

Framework signals to enumerate sources:

| Framework | Source patterns |
|-----------|----------------|
| Express | `req.body`, `req.query`, `req.params`, `req.headers`, `req.cookies`, `req.files` |
| Koa | `ctx.request.body`, `ctx.query`, `ctx.params`, `ctx.headers`, `ctx.cookies` |
| Flask/Quart | `request.json`, `request.form`, `request.args`, `request.values`, `request.headers`, `request.cookies`, `request.files` |
| FastAPI | Function parameters with `Body()`, `Query()`, `Path()`, `Header()`, `Cookie()`, `Form()`, `File()`, `UploadFile` |
| Django | `request.POST`, `request.GET`, `request.body`, `request.META`, `request.COOKIES`, `request.FILES`, `request.headers` |
| Rails | `params[:*]`, `request.headers`, `cookies`, `session` |
| Spring | `@RequestBody`, `@RequestParam`, `@PathVariable`, `@RequestHeader`, `@CookieValue` |
| Gin | `c.Param`, `c.Query`, `c.PostForm`, `c.GetHeader`, `c.BindJSON` |
| ASP.NET | `[FromBody]`, `[FromQuery]`, `[FromRoute]`, `[FromHeader]`, `Request.Form`, `Request.Cookies` |

### Authentication Artifacts as Input

Once decoded, the *content* of authentication artifacts is attacker-controllable in many threat models:

- JWT claims after decode (especially custom claims).
- OAuth/OIDC `id_token` claims.
- SAML assertion attributes.
- Session data not server-side-bound (e.g., signed cookies storing user state).

Treat the *signature* as trusted only if verification is correct (see crypto checklist). Treat the *content* as a source for downstream code that uses it.

### CLI and Local Sources

- `argv` / command-line arguments.
- `stdin`.
- Environment variables (sometimes attacker-controlled in CI/container contexts).
- Config files read at runtime (sometimes attacker-controlled if the config path is itself a source).

### Message and Event Sources

- Message queue consumers (Kafka, SQS, RabbitMQ, NATS, Pub/Sub) — message body is a source. Producer authentication does not make the content trusted unless cryptographically bound.
- Webhook receivers — body and headers; verify signatures before treating as trusted.
- Scheduled job payloads — if payload is generated from prior user input, it is a source.

### External Calls Treated as Trusted

The application calls a third-party API and trusts the response. The response is a source if any of the following hold:

- The third party is not contractually trusted (any public API).
- The response includes user-controlled fields (e.g., a payment provider returning the user's billing address).
- The response is deserialized into a typed structure without validation.
- The response is rendered into HTML, executed as code, or used in a security decision.

### Storage Sources (Second-Order Taint)

Data read from a database, cache, file, or object store is a source if it was *originally written from a different source*. This is the root of:

- Stored XSS (HTML written by user A, rendered to user B).
- Second-order SQL injection (string written from input X, later concatenated into a query Y).
- Log injection from stored content (user-controlled fields rendered in operator dashboards).
- Email injection (CRLF in a field written from user input, later used in an email header).

Trace originating writes to identify second-order sources.

## Sinks to Enumerate

A "sink" is anywhere data flows into a dangerous operation. The danger depends on the operation; the categorization below maps sinks to weakness classes.

### Database Query Construction (Injection Risk)

- Raw query APIs: `db.query(string)`, `cursor.execute(string)`, `connection.exec(string)`, `client.queryRaw(string)`.
- ORM raw-mode escapes: `Model.find_by_sql`, `repository.query`, `Sequelize.literal`, `SQLAlchemy text()`, `Django RawSQL`.
- NoSQL operators: MongoDB `$where`, `$expr`, `$regex` with user input; Mongo dynamic field names.
- Search query DSLs: ElasticSearch query bodies built from strings, OpenSearch.
- LDAP filter strings.
- Anywhere a query is built via string concatenation, template interpolation (`f""`, backticks, `.format()`, sprintf), or string concatenation of user input.

### Shell and Process Execution (Command Injection Risk)

- `subprocess.run/Popen` with `shell=True`.
- `os.system`, `os.popen`, `commands.getoutput` (Python).
- `child_process.exec`, `child_process.execSync` (Node).
- `Runtime.exec(String)` with a single string (Java); the array form is safer but still risky if the array contains user input.
- Backticks, `$()` in any embedded shell.
- `system`, `exec`, `passthru`, `shell_exec`, ``\` (PHP).
- `Open3.popen` with single string (Ruby); `Kernel#system` with single string.
- `cmd.exe /c` constructions.

### Code Execution (Code Injection Risk)

- `eval(string)`.
- `Function(string)`, `setTimeout(string)`, `setInterval(string)` (JavaScript).
- `vm.runInThisContext`, `vm.runInNewContext`, `vm.runInContext` (Node).
- `exec()` (Python). Ruby `eval`, `instance_eval`, `class_eval`.
- `Function.new` (Ruby).
- Reflective invocation that selects the method or class from user input (Java `Class.forName`, Python `getattr` with user-controlled name).

### Deserialization Sinks

- Python: `pickle.loads`, `pickle.load`, `marshal.loads`, `yaml.load` (without `SafeLoader`), `shelve.open`.
- Java: `ObjectInputStream.readObject`, `XMLDecoder`, `XStream` without typed allowlist, `Jackson` with default typing enabled, `SnakeYAML` non-safe constructor.
- PHP: `unserialize`.
- .NET: `BinaryFormatter`, `SoapFormatter`, `NetDataContractSerializer`, `LosFormatter`, `ObjectStateFormatter`, `JavaScriptSerializer` with `SimpleTypeResolver`, `Json.NET` with `TypeNameHandling != None`.
- Ruby: `Marshal.load`, `Marshal.restore`, `YAML.load` (older Psych without safe mode).
- Node: `JSON.parse` with a reviver that instantiates objects, `node-serialize`.

### Filesystem Sinks (Path Traversal, Arbitrary Read/Write)

- `open(path)`, `fs.readFile`, `fs.writeFile`, `pathlib.Path(path)`, `File.open`, `FileInputStream`, `FileOutputStream`.
- Path joining: `os.path.join`, `path.join`, `Paths.get` — joining user input still allows traversal if not canonicalized.
- Archive extraction APIs: `zipfile.extract*`, `tarfile.extractall`, `archive/zip` Reader, `unzipper`. Risk: zip slip.
- File deletion / rename: `os.remove`, `os.rename`, `unlink`, `fs.unlink`.

### HTTP Egress (SSRF Risk)

- `requests.get/post`, `urllib.request`, `http.client`, `httpx`.
- `fetch`, `axios`, `node-fetch`, `got`, `request`, `superagent`.
- `net/http` Get/Post (Go), `http.Client.Do`.
- `HttpClient`, `WebClient` (.NET), `Net::HTTP` (Ruby).
- DNS resolution as a side effect (`socket.gethostbyname` with user input).
- WebSocket clients, SMTP clients, FTP clients, any networking with user-controlled destination.

### Template Rendering (SSTI / XSS Risk)

- Server-side: `Jinja2.Template(...).render`, `Twig.render`, `Handlebars.compile`, `Mustache.render`, `EJS.render`, `ERB.new(...).result`, `Razor`, `Thymeleaf`, `Liquid`, `Pug`. SSTI risk when user input becomes template *source*; XSS risk when user input becomes template *data* without auto-escape.
- Triple-stash and unescaped output markers: `{{{ }}}` (Handlebars), `{!! !!}` (Blade), `{% raw %}` (Jinja2 with `safe`), `<%= %>` (ERB without `h()`), `dangerouslySetInnerHTML` (React), `v-html` (Vue), `[innerHTML]` (Angular without sanitizer).
- HTML construction in code: `innerHTML =`, `document.write`, `outerHTML`, `insertAdjacentHTML`.

### HTTP Response Writing (XSS, Header Injection, Open Redirect)

- Response body writes that include user input without encoding.
- `res.redirect(userControlledUrl)`, `header("Location: " + ...)` — open redirect.
- `Set-Cookie` with user input — cookie injection / fixation.
- Custom header writes containing CR/LF — header injection.
- `Content-Type` set from user input — MIME confusion.

### Logging Sinks

- Any log call with user input is a sink for two reasons: log injection (CRLF / control chars), and sensitive-data leak (logging tokens, passwords, request bodies).
- Common: `logger.info("...{user_input}...")`, `console.log`, `log.Println`, `slog.Info`.
- Structured loggers (Zap, Serilog, structlog, pino) reduce log injection risk for the message field but still leak sensitive fields.

### Cryptographic Operations as Sinks

- Key derivation, signing, verification, encryption, decryption with user-influenced parameters (algorithm, mode, IV, KDF iterations, salt).
- JWT verification with user-influenced `alg` selection (alg confusion).
- HMAC verification not using constant-time comparison.

### Authentication / Authorization Decision Points

- Any code that grants or denies access based on data: `if user.role == "admin"`, `if user_id == row.owner_id`, `if user.has_perm(...)`. The data being checked is a sink for the decision; if data flow allows the data to be tampered with before the check, the decision is broken.
- Password verification, session validation, MFA verification.
- Signature verification on webhooks, JWTs, signed URLs.

### Identity-Sensitive Sinks

- Email composition (header fields, recipient lists) — header injection / mass-mail abuse.
- SMS composition — toll fraud / smishing routes.
- Push notifications — abuse channel for delivering attacker content to victim devices.

## Building the Map

For each in-scope service:

1. **List all entry points.** Routes, message consumers, scheduled jobs, CLI commands, lambdas. For each, list the source classes the entry point exposes.
2. **List all sinks reachable from the service.** Walk imports and call graphs from entry points; record every sink with file:line and the sink's category.
3. **For each source, identify reachable sinks.** Trace the call graph. When the path is long, record the path as `entry → middleware → handler → service → repository → query`.
4. **Annotate validation, sanitization, and authorization at each step.** Note the function names of validators and encoders used, and whether they apply to the specific sink (e.g., HTML escape does not protect a SQL sink).

The map does not need to be a finished artifact; a working list updated during analysis is sufficient. The point is to make the auditor *think in terms of source-to-sink reachability*, not pattern matching.

## Heuristics

- **A sink without a source is benign.** SQL string concatenation that only contains compile-time constants is not a vulnerability.
- **A source without a reachable sink is harmless.** A request body field that is logged once and never used elsewhere is at most a logging concern.
- **The risk lives at the sink, but the proof lives in the path.** A proven path from a source to a sink with no adequate validation is a vulnerability. A theoretical path through code that is unreachable in practice is not.
- **Validation is sink-specific.** HTML-encoding does not protect SQL; SQL-parameterization does not protect shell; shell-quoting does not protect path. Match the validation/encoding to the sink.
- **Allowlists beat denylists.** Validators that allow known-good are harder to bypass than validators that block known-bad.
- **Type validation is necessary but not sufficient.** Confirming that a value is a string does not protect against injection if the string flows into a sink. Confirming that a value is a positive integer often does, because integers don't carry payloads.

## Worked Example

A small Express + Mongoose application with an endpoint:

```js
// src/api/orders.ts:42
router.get('/orders/:id', requireAuth, async (req, res) => {
  const order = await Order.findById(req.params.id);
  if (!order) return res.status(404).end();
  res.json(order);
});
```

**Sources** at this entry point:

- `req.params.id` (path param; expected ObjectId).
- `req.user` (set by `requireAuth`; trusted artifact, but its `tenantId` is the basis for authorization).
- `req.headers.*` (any header; not used here).

**Sinks** reachable:

- `Order.findById(req.params.id)` (database query sink at `src/api/orders.ts:43`).
- `res.json(order)` (HTTP response sink at `src/api/orders.ts:45`).

**Path analysis**:

- `req.params.id` → `Order.findById` → MongoDB query.
  - Validation: Mongoose coerces to ObjectId; throws on invalid format. Type-validation is sufficient for injection at this sink.
  - Authorization: **MISSING.** No check that `order.tenantId === req.user.tenantId`. This is BOLA / IDOR. Cross-tenant read is possible by guessing or enumerating ObjectIds. **Finding.**
- `order` → `res.json(order)` → response.
  - Sink-specific validation: JSON serialization does not need encoding. But: does `order` contain fields not safe to expose? If the schema includes `internalNotes` or `paymentToken` fields, this is excessive data exposure (OWASP API3). **Possible finding** depending on schema review.

A pattern-matching scan would not find the missing authorization check because there is no "bad pattern" — the code is clean syntactically. A source-to-sink walk finds it because the auditor explicitly asks "is the right authorization check between this source and this sink?" and finds the answer is "no."

## Pitfalls

- **Stopping at the function boundary.** A sink in `userService.getById` is a sink at the route handler that calls it. Trace through.
- **Trusting middleware names.** A middleware called `validate` may validate format but not authorize; a middleware called `authenticate` may authenticate but not authorize. Read the middleware.
- **Trusting framework abstractions.** ORMs claim to "prevent SQL injection." Most do, for the parameter-binding API. They do *not* protect against string-concatenated raw queries. Verify which API the call uses.
- **Ignoring storage paths.** A response that reads from the database does not have a "source" in the request. The source is the prior request that wrote the data. Map second-order paths.
- **Forgetting non-route sources.** Webhooks, cron jobs, message consumers, and admin tools are all entry points and must be in the map.
