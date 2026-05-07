# JavaScript / TypeScript Security Footguns

## Scope

JavaScript and TypeScript on Node.js (server) and browser (client). TypeScript adds compile-time type safety but does not change runtime behavior; all runtime issues apply equally. Cross-reference `xss-csrf-frontend.md` for browser issues, `deserialization.md` for server-side parsing, `ssrf-redirect-url.md` for fetch/axios concerns.

## Code Execution Sinks

### `eval`, `Function`, `setTimeout(string)`, `setInterval(string)`

- `eval(s)` — Full code execution.
- `new Function(...args, body)` — Equivalent to eval; common in expression engines.
- `setTimeout("code", 100)` — Legacy; `setTimeout` accepts string and evaluates.
- `setInterval("code", ...)` — Same.
- **No safe usage on untrusted input.** No "limited" mode.

### `vm` Module (Node)

- `vm.runInThisContext(code)` — Runs in current global; full access.
- `vm.runInNewContext(code, sandbox)` — Sandbox is *not* a security boundary; the contextified object proxies; gadget chains can escape. Don't rely on it for untrusted code.
- `vm.runInContext(code, ctx)` — Same.
- VM2 / isolated-vm libraries — Better attempts but VM2 has had multiple sandbox-escape CVEs; treat as potentially escapable; isolated-vm uses V8 isolate which is more robust.

### `child_process.exec`, `execSync`

- `exec("cmd " + userInput)` — Shell-interpreted; command injection.
- Use `execFile(file, args[])` or `spawn(file, args[])` for safer arg passing (no shell).

### `serialize-javascript` with `unsafe: true`, `node-serialize`

- Function deserialization → RCE; covered in `deserialization.md`.

### Dynamic `require(...)`, ESM `import(name)`

- User-controlled module names — Module loading can have side effects (install scripts, top-level code).
- Allowlist module names; never derive from input.

### Function constructor variants

- `Function.apply`, `Function.call` with attacker-controlled function — verify origin.

## Prototype Pollution

User input merged into objects with `__proto__`, `constructor.prototype`, or `prototype` keys writes to the prototype chain.

### Sinks

- `Object.assign(target, userInput)` — Generally safe (own properties only) but verify.
- `_.merge`, `_.mergeWith`, `_.defaultsDeep` (lodash) — Vulnerable in older versions; 4.17.21+ has fixes for documented cases.
- `_.set(obj, path, val)` with user-controlled `path` — Pollutes via path traversal.
- `$.extend(true, target, userInput)` (jQuery deep extend).
- Custom merge functions; audit each.

### Mitigations

- `Object.create(null)` for maps holding user data — no prototype to pollute.
- Reject `__proto__`, `constructor`, `prototype` keys in input validation.
- `Object.freeze(Object.prototype)` — Defense-in-depth; may break libraries.

### Downstream Impact

- After pollution, code reading `obj.someProp` (without `hasOwn`) sees the polluted value; can lead to RCE in template engines, command-builders, or auth checks.

## DOM XSS Sinks (Browser)

- `element.innerHTML = userInput`
- `element.outerHTML = userInput`
- `element.insertAdjacentHTML(pos, userInput)`
- `document.write(userInput)`, `document.writeln(userInput)`
- `eval(userInput)` (also code execution)
- `Function(userInput)`
- `setTimeout(userInput, ...)`, `setInterval(userInput, ...)` — string form
- `location = userInput`, `location.href = ...`
- `window.location.assign / replace`
- `window.open(userInput)`
- jQuery `$(html)` — Constructs DOM from string
- jQuery `$(...).html(html)`, `.append(html)`, `.before(html)`, `.after(html)`, `.replaceWith(html)`

### Framework-specific bypasses

- React `dangerouslySetInnerHTML={{__html: userInput}}` — XSS.
- Vue `v-html="userInput"` — XSS.
- Angular `[innerHTML]="userInput"` — Sanitized by default; bypass via `bypassSecurityTrustHtml`.
- Svelte `{@html userInput}` — XSS.

### URL sinks

- `<a href={userInput}>` allows `javascript:alert(1)`. Validate scheme allowlist.
- `<iframe src={userInput}>` similar.
- React 16.9+ logs a console warning for `javascript:` URLs in `href` / `src` attributes but does not block them. Validate the scheme against an allowlist (`http:`, `https:`, `mailto:`, etc.) before rendering user-supplied URLs.

## Authentication / Sessions

### JWT

- `jsonwebtoken` library:
  - `jwt.verify(token, secret, options)` — Always pass `algorithms: ['HS256']` (or expected algorithm) in options. Without it, accepts attacker-chosen algorithm.
  - `jwt.decode(token)` — Does NOT verify signature. Common bug: using decode where verify is intended.
- `jose` library — Modern; safer defaults but verify configuration.
- Storing JWT in `localStorage` — Vulnerable to XSS exfiltration; cookies with `HttpOnly` are better for browser apps. (Trade-off with CSRF.)

### Sessions

- `express-session`:
  - Default session store is `MemoryStore`; warns "for development only." Production needs Redis / DB store.
  - Cookie flags: `secure: true`, `httpOnly: true` (default), `sameSite: 'lax'` or `'strict'`.
  - Session ID rotation (`req.session.regenerate()`) on login.
- Passport.js — Strategy-specific behavior; verify each strategy's config.

## SQL / DB

### `mysql` / `mysql2`

- `connection.query("SELECT ... " + x)` — Concat injection.
- `connection.query("SELECT ... ?", [x])` — Parameterized; safe.
- `mysql2/promise` — same.

### `pg` (Postgres)

- `pool.query("..." + x)` — Concat injection.
- `pool.query("... $1", [x])` — Parameterized.

### Sequelize

- `User.findAll({where: {name: x}})` — Safe (parameterized).
- `User.findAll({where: literal("...")})` — Raw; concat is injection.
- `sequelize.query("..." + x)` — Concat injection.
- `User.findAll({where: req.query})` — Operator injection (Sequelize accepts `[Op.gt]` etc.); use schema-based filtering.

### Knex

- `knex.raw("..." + x)` — Concat injection.
- `knex('users').where({id: x})` — Safe.

### Mongoose

- `User.findOne({username: req.body.username})` — When `username` is `{$ne: null}`, returns first user (NoSQL injection). Coerce types: `String(req.body.username)`.
- `Model.find({}, req.query.field)` — Field selection from user input; can leak sensitive fields.
- `$where` operator with user-controlled JS — RCE (covered in `injection.md`).

### Prisma

- Generally parameterized via type-safe API.
- `prisma.$queryRaw\`SELECT ... ${x}\`` — Tagged template; safe.
- `prisma.$queryRawUnsafe("..." + x)` — Concat injection.

## File Handling

- `fs.readFile(userPath)` — Path traversal.
- `path.join(BASE, user)` — Doesn't prevent `..`; canonicalize with `path.resolve` and verify prefix.
- Static file serving with raw `req.url` — Verify framework's serve-static / express.static behavior; default is reasonable, but custom serving needs validation.
- Archive extraction:
  - `unzipper`, `adm-zip`, `tar` — Historic Zip Slip; verify entry path resolution.
  - `extract-zip` — Modern; check version for fixes.
- File upload: `multer` configuration — `dest` directory permissions; filename sanitization (`multer.diskStorage` with custom `filename` function).

## Network

### `fetch`, `axios`, `node-fetch`, `got`, `superagent`

- SSRF risks; covered in `ssrf-redirect-url.md`.
- TLS verification — Disabled via `axios.create({httpsAgent: new https.Agent({rejectUnauthorized: false})})` or `--tls-reject-unauthorized=0`.
- Redirect handling defaults vary; verify per library.

### `request` (deprecated)

- Deprecated and unmaintained since 2020; carries historic vulnerabilities. Migrate to built-in `fetch` (Node 18+), `undici`, or `axios`. `node-fetch` is in maintenance mode and is no longer the preferred new dependency.

### `socket.io`

- WebSocket auth on connect; per-event authorization separate.
- CORS configuration (`io.attach(server, {cors: {...}})`).

### `http`, `https` (built-in)

- TLS via `rejectUnauthorized` option; disabled values are findings.

## Crypto

### `crypto` (built-in)

- `crypto.createHash('md5')`, `'sha1'` — Not for security purposes.
- `crypto.createHmac('sha256', secret)` — Use HMAC-SHA-256+; verify constant-time comparison via `crypto.timingSafeEqual`.
- `crypto.randomBytes(n)` — CSPRNG; safe.
- `Math.random()` — NOT cryptographically secure; never for tokens, salts, IDs.
- `crypto.createCipher` — Deprecated, derives key from password unsafely; use `createCipheriv`.
- `crypto.createCipheriv('aes-256-gcm', key, iv)` — Modern; provide unique IV per encryption.
- `crypto.scrypt`, `crypto.pbkdf2` — Built-in KDFs; use for passwords with proper iteration counts.
- `crypto.randomUUID()` — Generates v4 UUID; cryptographically secure.

### Bcrypt / Argon2

- `bcrypt` (npm) — Cost factor ≥ 12; verify implementation matches docs.
- `argon2` (npm `argon2`, native) — Argon2id mode; modern parameters.

### Node's TLS

- Default TLS version supported; verify `secureProtocol` not pinned to old.

## XML

- `xml2js` — Parsed safely by default; verify no XXE-like options enabled.
- `libxmljs` — Lower-level; verify entity resolution disabled.
- `fast-xml-parser` — Fast; verify config.

## Frameworks and Routes

### Express

- Route order matters; `app.use('/admin', adminRouter)` should be after auth middleware.
- `express.static` for serving files; verify directory.
- `body-parser` size limits (`express.json({limit: '...' })`).
- `helmet` middleware for security headers.
- `cors` middleware — Verify origin allowlist.
- `csurf` is deprecated and the GitHub repo is archived (CVE / advisory history); presence of `csurf` in `package.json` is itself a finding. Replace with `csrf-csrf`, `lusca`, or framework-native CSRF protection paired with `SameSite=Lax` cookies.

### NestJS

- Decorator-based; `@UseGuards`, `@Roles` declarative; verify applied to all sensitive routes.
- `class-validator` for DTOs; mass assignment guarded by DTO shape.

### Next.js

- API routes (`/pages/api/`) — Auth applied per-route; verify.
- `getServerSideProps` running server-side; verify SSRF / SQL exposure.
- Middleware (Edge); verify TypeScript strict-typed configs.

### Hapi, Koa, Fastify

- Each has its own middleware/auth pattern; same questions: are routes protected, body parsed safely, errors not leaking.

## Browser-Specific

### `postMessage`

- `event.origin` validation — Constant origin check; never `event.origin === '*'`.
- Treat `event.data` as untrusted.

### `localStorage` / `sessionStorage`

- Accessible by JS; XSS exfiltrates everything. Don't store auth tokens here for high-security contexts.

### Web Workers / Service Workers

- Origin scope; ensure ServiceWorker scope is limited; user-controlled fetch in worker.

### `navigator.sendBeacon`, `EventSource` — Verify URL targets.

## Common Findings Patterns

- `eval(JSON.stringify(userInput))` — Even after stringify, still RCE on untrusted.
- `new Function('return ' + userInput)()` — RCE.
- `child_process.exec("cmd " + userInput)` — Command injection.
- `verify=False` equivalents in HTTPS clients.
- `Math.random()` for tokens / IDs.
- `bcrypt.hash(password, 4)` — Cost too low.
- `User.findOne({username: req.body.username})` without type coercion in Mongoose.
- `dangerouslySetInnerHTML={{__html: userMarkdown}}` without sanitization.
- `_.merge({}, JSON.parse(req.body))` — Prototype pollution.
- `jwt.decode(token)` where `jwt.verify` was intended.
- `app.use(cors())` — Default permissive CORS.
- `JSON.parse(input, reviver)` where reviver instantiates classes from `__class__`.

## Recommendation Patterns

- Use `crypto.randomBytes` / `crypto.randomUUID` for any randomness in security context.
- Use `bcrypt` (cost ≥ 12) or `argon2` for password storage.
- Use parameterized queries via library APIs; never concat into raw SQL.
- Use `helmet` for security headers.
- Use schema validators (Zod, Joi, class-validator) for input — typed DTOs prevent mass assignment.
- Use `Object.create(null)` for user-data maps; reject `__proto__`/`constructor`/`prototype` keys.
- Pin dependencies; `npm audit` in CI; treat findings as blocking.
- For JWT, always specify `algorithms` in `verify` options.
- For browsers: prefer `HttpOnly` cookies for auth tokens; combined with strict CSRF defense.
