# Python Security Footguns

## Scope

Python-specific patterns that produce security findings. Apply alongside the domain checklists when Python is detected in scope. Cross-reference deserialization.md (pickle/yaml), injection.md (subprocess), and crypto.md (hashlib defaults).

## Code Execution Sinks

### `eval`, `exec`, `compile`

- `eval(s)` — Evaluates expression; full RCE if `s` is influenced by user input.
- `exec(s)` — Executes statements; same risk; some teams reach for this when implementing plugins/scripting.
- `compile(s, ...)` — Compiles to code object; combined with `exec` is the same risk.
- **No safe usage on untrusted input.** AST-based evaluation (`ast.literal_eval`) is safe for limited literal types only — verify the use case fits and that `literal_eval` is what's actually called.

### `pickle.loads`, `pickle.load`, `marshal.loads`

Covered in `deserialization.md`. RCE via crafted byte stream. Migrate to JSON/msgpack.

### `yaml.load(...)`

Without `Loader=yaml.SafeLoader`, RCE via `!!python/object/apply`. Use `yaml.safe_load`.

### `subprocess` with `shell=True`

- `subprocess.run(cmd, shell=True)` with concatenated input — command injection.
- `subprocess.Popen(cmd, shell=True)` — same.
- **Defense**: `shell=False` with arg list (`subprocess.run(["cmd", "arg"], ...)`).

### `os.system`, `os.popen`, `commands.getoutput` (Py2 legacy)

- All use shell; user input concatenation is injection.
- Use `subprocess` with arg lists.

### `__import__(name)` and reflective imports

- User-controlled `name` allows arbitrary module import; combined with `getattr(module, fn)` allows arbitrary call.
- Allowlist module names; never derive from input.

### `getattr(obj, name)(args)` with user-controlled `name`

- Reaches private attributes (`__class__`, `__globals__`, `__init_subclass__`) leading to escapes.
- Common in template engines (Jinja2 sandbox bypass historic class), plugin systems, RPC dispatchers.
- Allowlist permitted attribute names.

### `getattr(__builtins__, name)`, `globals()`, `locals()`

- All entry points to broader exec via reflection; verify any reflective lookup uses allowlists.

### `pty`, `os.exec*`, `os.spawn*`

- Same shell/exec risks; verify input handling.

## SQL / DB

### `cursor.execute("... " + x)`, `f"SELECT ... {x}"`

- Direct string concatenation/interpolation into SQL. Use parameterized form: `cursor.execute("SELECT ... WHERE id = %s", (x,))`.

### Django ORM

- `Model.objects.raw("...")` — Raw SQL; verify parameters.
- `Model.objects.extra(where=...)` — Deprecated; pre-parameterized but historic injection.
- `Model.objects.filter(**user_dict)` — When `user_dict` keys come from user input, `__` queries can pivot to unintended fields. Allowlist keys.

### SQLAlchemy

- `session.execute(text("..."), params)` — Safe with bind params; unsafe with concatenation.
- `text("WHERE id = " + str(x))` — Concat injection.
- `session.query(Model).filter_by(**user_dict)` — Same key-injection risk as Django.

### Pony ORM, Peewee, etc.

- Verify query construction; concat into raw SQL is the injection vector.

## Web Frameworks

### Flask

- `render_template_string(input)` — SSTI when input is user-derived.
- `request.args.get(...)`, `request.form.get(...)`, `request.json` — Sources; trace.
- `flash(message)` with HTML — Escaping context; verify default.
- `send_file` / `send_from_directory` with user-controlled path — Path traversal; verify `safe_join`.
- `redirect(url)` with user input — Open redirect; validate.
- Default debug mode (`app.debug = True`) — RCE via Werkzeug debug console PIN; never enable in prod.

### Django

- `mark_safe(s)` — Disables auto-escape; verify s is trusted.
- `format_html(...)` — Like sprintf for HTML; escape arguments correctly.
- `request.GET / POST / META / FILES` — Sources.
- `HttpResponseRedirect(url)` — Open redirect risk.
- `RawSQL` — Direct SQL; parameter binding required.
- CSRF middleware enabled for non-API views.
- `DEBUG=True` in production — Catastrophic info disclosure.
- `SECRET_KEY` rotation policy.

### FastAPI

- Pydantic models prevent mass assignment when types are strict; `Optional` and `Any` weaken.
- `Body(...)` accepts arbitrary JSON if not bound to a model.
- Path operation dependencies (auth) — verify applied to all endpoints.

### Quart, Sanic, Starlette

- Similar concerns to Flask / FastAPI.

## Templating

### Jinja2

- `Environment(autoescape=False)` or `Environment()` (default off if not enabling) — XSS by default.
- `{{ x | safe }}` — Disables auto-escape for `x`.
- `{% autoescape false %}` block.
- User-controlled template source via `Template(input).render(...)` — SSTI; engine-specific gadgets reach `__class__`, `__mro__`, `subclasses()` to RCE.
- Sandboxed environment (`SandboxedEnvironment`) — Limits but historic CVEs; not a complete defense.

### Other

- `Mako`: `<%! ... %>` python code; verify template source not user-controlled.
- `Cheetah`: similar dynamic features.
- `Django templates`: more restricted by design; SSTI usually requires custom tag/filter.

## File Handling

- `open(path)` with user-controlled `path` — Path traversal.
- `os.path.join(BASE, user)` with `..` — Doesn't block; canonicalize and prefix-check.
- `pathlib.Path(...).resolve()` — Resolves; check `.is_relative_to(BASE)`.
- `tarfile.extractall()`, `zipfile.extractall()` — Zip slip; use `filter='data'` (Python 3.12+) or per-member validation.
- `tempfile.mktemp()` — Predictable; use `mkstemp` or `NamedTemporaryFile`.

## Crypto

### `hashlib`

- `hashlib.md5`, `hashlib.sha1` — Not for security purposes (passwords, signatures, tokens). Use SHA-256+ for non-password integrity; argon2/bcrypt/scrypt for passwords.
- `hashlib.sha256(password.encode()).hexdigest()` — Not a password KDF; use `argon2-cffi`, `bcrypt`, `scrypt`, `passlib`.

### `hmac.compare_digest`

- Required for MAC / signature comparison; `==` on bytes is timing-side-channel.

### `random`

- `random.random()`, `random.choice(...)`, `random.randint(...)` — NOT cryptographically secure. Use `secrets.token_bytes`, `secrets.token_urlsafe`, `secrets.choice`.

### `cryptography` library

- Use `cryptography.fernet.Fernet` for symmetric authenticated encryption; high-level safe primitive.
- Low-level `Cipher` API allows CBC without HMAC; verify mode and authentication.

### `pycryptodome`

- `Crypto.Cipher.AES.new(key, AES.MODE_ECB)` — ECB mode; insecure.
- Manually-managed IVs / nonces; verify uniqueness and randomness.

## Network / Web Clients

### `requests`

- `verify=False` — TLS verification disabled; MITM-vulnerable.
- `requests.get(user_url)` — SSRF if URL is user-controlled.
- `allow_redirects=True` (default) — Redirect-following SSRF if not validating per-redirect.
- `requests.utils.urlparse` for SSRF validation — URL parser confusion (covered in ssrf-redirect-url.md).

### `urllib.request.urlopen`

- Same SSRF concerns; default `Context` may not verify SSL on older Python.

### `httpx`

- Async equivalent of requests; same concerns.

### `socket.gethostbyname`

- DNS resolution as side effect; reveals existence to DNS attacker.

## XML

- `xml.etree.ElementTree` — Modern Python disables external entities; verify version.
- `lxml.etree` — Set `resolve_entities=False`, `no_network=True`.
- `xml.dom.minidom` — Same considerations.
- `defusedxml` — Drop-in replacement; use it.

## Command-Line / argv

- `argparse` — Generally safe; verify no `shell=True` consumers downstream.
- `click`, `typer` — Same.
- `getopt`, manual `sys.argv` — Verify input handling.

## Concurrency

- `subprocess.Popen` not waited / not closed — Zombie processes; resource leak.
- `threading.Lock` — TOCTOU on protected state.
- `multiprocessing` with shared state — Verify shared resources synchronized.
- `asyncio` with blocking calls — Not security-critical alone; relevant when blocking call is auth verification (timing).

## Common Library Footguns

### `Werkzeug` debug pin

- Debug console PIN computed from machine info; predictable; RCE if `app.debug = True` and console reachable.

### `Tornado`

- `RequestHandler.write` — XSS if direct HTML and not escaped.
- WebSocket handler authentication — verify per-connection.

### `Pillow / PIL`

- Multiple historic CVEs; track version.
- `Image.open(file)` may invoke external decoders.

### `python-magic`

- File type detection; verify version not affected by past libmagic CVEs.

### `pyyaml`

- See `deserialization.md`.

### `paramiko`

- SSH library; verify host key checking enabled (`AutoAddPolicy` is unsafe in production).

### `requests-toolbelt`

- Multipart streaming; safe if used correctly.

## Python Version Notes

- **Python 2** — EOL since 2020; any Python 2 code is a finding (compatibility / supply chain).
- **Python < 3.8** — Out of upstream support; flag.
- **Older asyncio versions** — Subtle CVEs in `aiohttp` and async ecosystems; track.

## Common Findings Patterns

- `subprocess.run(cmd, shell=True)` with f-string-built `cmd` — Command injection.
- `eval(request.args.get("expr"))` — RCE.
- `pickle.loads(open("...").read())` from network/cache — RCE.
- `yaml.load(open(...))` — RCE; SafeLoader missing.
- `cursor.execute(f"SELECT * FROM users WHERE id = {request.GET['id']}")` — SQL injection.
- `Template(request.form["template"]).render(...)` — SSTI.
- `verify=False` in any `requests.get/post`.
- `random.choice(...)` for token generation.
- `hashlib.md5(password.encode()).hexdigest()` for password storage.
- `app.debug = True` in production (look at config / env handling).
- `mark_safe(user_input)` in Django templates.
- `__import__(user_input)` or `getattr(module, user_input)`.

## Recommendation Patterns

- Use `secrets` for any token or randomness used in security context.
- Use `argon2-cffi` (preferred), `bcrypt`, or `passlib` for password storage.
- Use `cryptography.fernet.Fernet` for symmetric encryption when use case fits.
- Use `subprocess` with arg lists; never `shell=True` with concatenation.
- Use parameterized SQL universally; never concatenate.
- Use `defusedxml` for any XML parsing.
- Pin dependencies; use `pip-audit` or similar in CI.
- Disable debug mode in production; verify via deployment manifest, not just code default.
