# Injection

## Scope

All weaknesses where untrusted data is interpreted as control-plane content by a downstream interpreter — SQL, shell, code, templates, search DSLs, LDAP, XPath, headers, logs. The unifying root cause is mixing data and code on a string boundary.

The defense pattern is consistent across all forms: separate data from code at the API boundary using parameter binding, structured APIs, or context-appropriate encoding — never string concatenation, format strings, or template interpolation that treats the data as part of the syntax.

## Checks

### SQL Injection

- **Raw query construction with user input** — `db.query("SELECT * FROM users WHERE id=" + id)`, `cursor.execute(f"SELECT ... {x}")`, `db.exec("UPDATE ... SET name='" + name + "'")`. Trace every raw query API for user-derived input.
- **ORM raw escapes** — `Sequelize.literal`, `SQLAlchemy text()` without bound parameters, Django `RawSQL`, Hibernate `createNativeQuery` with concatenation, ActiveRecord `find_by_sql` with interpolation, Knex `raw` with concatenation. ORMs do not protect raw queries.
- **Dynamic ORDER BY / LIMIT / column names** — Bound parameters protect values, not identifiers. `ORDER BY ${userInput}` is injection even with the rest parameterized. Allowlist column names against a fixed set.
- **LIKE patterns with user input** — Unescaped `%` and `_` allow pattern abuse; not strictly injection but a bypass class. Escape pattern metacharacters.
- **Stored procedures with dynamic SQL** — `EXEC sp_executesql @stmt` where `@stmt` is built from user input; same risk as inline.
- **Second-order SQL injection** — Data written from input X (sanitized at write) used in a query at write site Y (not sanitized at read). Trace database-as-source flows in Phase 5.
- **Database-specific extensions** — PostgreSQL `format()` with `%s` (no quoting), `EXECUTE` in PL/pgSQL with user input, Oracle `DBMS_SQL` with concatenation.

### NoSQL Injection

- **MongoDB query operators in user input** — When `req.body.username` arrives as `{"$ne": null}` and the code does `User.findOne({username: req.body.username})`, the user has injected a Mongo operator. Coerce types or use schemas that reject non-scalar inputs.
- **MongoDB `$where` with user input** — `$where` evaluates JavaScript; concatenated user input is RCE-class.
- **MongoDB dynamic field names** — `User.find({[req.body.field]: req.body.value})` allows querying any field, including `password_hash`.
- **CouchDB / DynamoDB / Firestore** — Each has its own query DSL; trace user input into any operator-bearing structure.

### LDAP Injection

- **Filter string concatenation** — `(&(uid=${user})(objectClass=person))` with unescaped `${user}` allows breaking out with `*)(uid=*)(`. Use parameterized LDAP libraries or escape with the LDAP-filter character class.
- **DN injection** — User input in DN construction (commas, `=`, special chars) corrupts hierarchy.

### XPath Injection

- **String-built XPath queries** — `//user[name='${name}' and password='${pw}']`. Same root cause as SQLi; same fix (parameterize via XQuery variables or `setXPathVariableResolver`).

### XML / XXE

- **XML parsers with external entity processing enabled** — Java `DocumentBuilderFactory` defaults vary; verify `setFeature("disallow-doctype-decl", true)` and `setExpandEntityReferences(false)`. Python `lxml` `resolve_entities=False`. Node `libxmljs` `noent=false`. .NET `XmlReaderSettings.DtdProcessing = Prohibit`.
- **Billion-laughs / quadratic-blowup** — DTD entity expansion DoS even when XXE is mitigated. Disable DTDs entirely on untrusted XML.
- **XInclude expansion** — Less common but same class.

### Command Injection (OS)

- **Shell-true subprocess** — Python `subprocess.run(cmd, shell=True)` with concatenated input, `os.system(...)`, `os.popen(...)`. Use `shell=False` with argument list.
- **Backticks / `$()`** — In any embedded shell context (Bash heredoc in Python/Node, shell helpers).
- **Node child_process** — `exec`, `execSync` with concatenation. Use `execFile`, `spawn` with arg array.
- **Java `Runtime.exec(String)`** — Single-string form is a shell tokenizer; user input can break tokenization. Use `String[]` form or `ProcessBuilder` with separate arguments.
- **PHP** — `system`, `exec`, `passthru`, `shell_exec`, backticks, `popen`. `escapeshellarg` and `escapeshellcmd` have known edge cases (multi-byte chars, certain shells); prefer not invoking shell.
- **Argument injection** — Even without `shell=True`, passing user input as the first or unexpected arg position can pass attacker-controlled flags. `git --upload-pack=<malicious>` style. Validate that user inputs occupy expected positions and don't begin with `-`.
- **Hidden shell invocations** — Functions that accept "options" and end up forking a shell internally (some `subprocess` wrappers, `make`, certain `git` operations). Read library source if uncertain.

### Code Injection

- **`eval` / `Function` / `setTimeout(string)`** — JavaScript variants. Reject any pattern that compiles user input as code.
- **Python `eval`, `exec`, `compile`** — even on "trusted" templated code; if any user value reaches the compiled string, it's injection.
- **Reflection-based instantiation** — `Class.forName(userInput)`, Python `getattr(module, userName)`, Ruby `Object.const_get(userInput)`. Allowlist when reflection is unavoidable.
- **`vm` modules** — Node `vm.runInThisContext`, `vm.runInNewContext`. Sandbox claims are weak; treat as RCE if user input reaches them.

### Template Injection (SSTI)

- **User input as template *source*** — `Jinja2.Template(req.body.template).render(...)`, `Twig.from_string(input).render(...)`, `Handlebars.compile(input)({...})`. RCE-class in many engines (Jinja2, Twig, ERB, Velocity).
- **User input via `safe` / `raw` filters** — Jinja2 `{{ x | safe }}`, Django `{% autoescape off %}`, Liquid `{{ x | raw }}`, EJS `<%- %>` (unescaped), ERB without `h()` helper.
- **Triple-stash / unescaped emit** — Handlebars `{{{x}}}`, Mustache `{{{x}}}`, Blade `{!! $x !!}`. By design unescaped; only correct with already-trusted HTML.

### Header Injection / CRLF

- **CR/LF in response headers** — `res.setHeader('X-Foo', userInput)` where `userInput` contains `\r\n`. Some frameworks reject CRLF; verify the specific framework. Direct output to socket without framework protection is unsafe.
- **`Set-Cookie`, `Location`, custom headers** — Same risk; same fix (validate or use framework helpers that reject invalid characters).
- **Email header injection** — `To: ${userEmail}` where `userEmail` contains `\r\nBcc: attacker@...`. Use library APIs that take recipient lists, not concatenation.

### Log Injection

- **CRLF in log messages** — User input with `\r\n` injects forged log lines. Strip or replace control characters before logging.
- **Sensitive log fields** — Logging request body with passwords, tokens, full session cookies. Audit every log call near auth, payment, and identity flows. (See `error-and-logging.md` for depth.)

### Search-DSL Injection

- **Elasticsearch / OpenSearch** — Building query DSL bodies from user strings rather than structured objects. Especially dangerous with `script` fields (scripted RCE).
- **GraphQL** — Field selection, fragment names, alias names should not be built from user input on the server side.

### Format-String / Logger Injection

- **C / C++ `printf`-family with user input as format string** — Direct memory disclosure / overwrite. Always pass user input as an argument, never as the format.
- **Java `String.format` with user format** — Less impact (no memory write) but information disclosure and DoS via format misuse.
- **`logger.info(userInput)` in libraries that interpret format placeholders** — log4j-style `${jndi:...}` (CVE-2021-44228). Check log4j version; verify format-message-converter behavior. Even after log4j patches, third-party formatters can interpret payloads.

## Framework Notes

- **Express + Sequelize/Mongoose**: search for `.literal`, `.query` (raw), `$where`, dynamic field paths.
- **Django**: search for `RawSQL`, `extra(where=...)`, `cursor.execute`. Default ORM is parameterized.
- **Spring Data + JPA**: search for `@Query(nativeQuery=true)`, `EntityManager.createNativeQuery`. JPQL is parameterized but native is up to the developer.
- **ActiveRecord**: search for `find_by_sql`, `where("...#{x}...")`, `connection.execute`. The `where(hash)` form is safe.
- **Go database/sql**: search for `db.Exec("..." + x)`, `fmt.Sprintf` building queries. Parameter placeholders are `$1` (Postgres) / `?` (MySQL).
- **PHP PDO / mysqli**: prepared statements are safe; `mysql_query` (deprecated) and concatenated PDO are not.

## Bypass Patterns

- **Encoding tricks** — URL-encoding (`%27` instead of `'`), Unicode normalization (`U+FF07` fullwidth apostrophe collapsing to `'`), HTML entity decoding before SQL execution. Validation must happen on the canonical form used at the sink.
- **Type coercion** — Submitting an array where a string is expected (`?id[]=1&id[]=' OR ...`) bypasses naïve string-only sanitizers. Validate type, not just content.
- **Multi-byte / encoding inconsistency** — Sanitizers operating on UTF-8 strings used as Latin-1 at the database layer. Make encoding explicit and consistent throughout.
- **Sanitizer-then-modify** — Sanitize → trim → use. Trim or normalization after sanitization can re-introduce dangerous characters. Sanitize last, at the sink.
- **Allowlist that evaluates user input** — `if user.role in [adminRoleString, ...]` where user controls `role`; the comparison string itself comes from input. Compare against fixed constants only.

## Recommendation Patterns

- **For SQL**: parameterized queries / prepared statements. For ORDER BY / column names: allowlist against a fixed set of column names compiled into the application.
- **For shell**: avoid shells (use process-spawn APIs with argument lists). When unavoidable, use language-specific quoting libraries and validate that the position/role of user input is what the program expects (no leading `-`, no path separators, etc.).
- **For code execution sinks**: redesign to remove the sink. There is no safe way to allow user input to reach `eval`-class APIs.
- **For templates**: render data, never compile user-controlled templates. If templating is required, use a sandboxed engine (and review the sandbox's CVE history).
- **For headers**: use framework helpers (`res.cookie(...)`, `res.redirect(...)`); reject CR/LF on any user input bound for header context.
- **For logs**: structured logging with separate field for user data; sanitize control chars; never log secrets.
