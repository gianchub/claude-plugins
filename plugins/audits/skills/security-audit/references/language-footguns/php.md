# PHP Security Footguns

## Scope

PHP server-side. Laravel, Symfony, WordPress, Drupal, raw PHP. Cross-reference `deserialization.md` (unserialize), `injection.md` (system/exec, SQL), `crypto.md` (openssl, hash).

## Code Execution Sinks

### `eval`, `assert` (with string), `create_function`

- `eval($code)` — Direct RCE.
- `assert($code)` — Pre-PHP 7.2 evaluated string as code; deprecated, but legacy code may use.
- `create_function` — Removed in PHP 7.2+; same risk in older code.
- `preg_replace` with `/e` modifier — Removed in PHP 7+; legacy code may use.

### `system`, `exec`, `shell_exec`, `passthru`, `popen`, `` ` `` (backticks)

- All shell-interpreted with single string; user input → command injection.
- `escapeshellarg` and `escapeshellcmd` have known edge cases (multi-byte handling, Windows shells); prefer `proc_open` with separated arguments where possible.
- `proc_open` with `cmd` array (PHP 7.4+) — Argv form; safer.

### `unserialize`

- See `deserialization.md`. Object injection via magic methods (`__wakeup`, `__destruct`, `__toString`, etc.); RCE via gadget chains.
- `unserialize($input, ['allowed_classes' => false])` (PHP 7+) — Disables class instantiation; safer.

### Dynamic `include`, `require`, `include_once`, `require_once`

- `include $userInput . '.php'` — Local file inclusion; combined with file upload becomes RCE; combined with PHP wrappers (`php://input`, `php://filter`, `data://`) becomes remote/data inclusion.
- `allow_url_include` setting — Should be `Off`; with `On`, `include "http://attacker/..."` is RFI → RCE.
- Allowlist included filenames; never derive from input.

### `call_user_func`, `call_user_func_array` with user-controlled callable

- `call_user_func($userFunc, $args)` — Calls arbitrary function/method.
- Same with `array_map`, `usort`, `preg_replace_callback`, `array_walk` if first arg is user-controlled.

### `extract($_POST)` and similar

- Imports variables from array into local scope; with user input, overrides local variables (security/auth state) — variable injection.
- `parse_str($queryString)` (without second arg) — Same; imports into globals (PHP 7+ removes default global behavior).

### Reflection

- `ReflectionClass`, `ReflectionMethod` with user input — Instantiate / invoke arbitrary; allowlist.

## SQL / DB

### `mysqli`

- `mysqli_query($conn, "..." . $x)` — Concat; injection.
- `mysqli_prepare` + `mysqli_stmt_bind_param` — Parameterized; safe.

### PDO

- `$pdo->query("..." . $x)` — Concat; injection.
- `$pdo->prepare(...)` + `bindValue` / `bindParam` — Parameterized; safe.
- `PDO::ATTR_EMULATE_PREPARES = true` (default for some drivers) — String-substitution emulation; can leak through edge cases (multi-byte). Set to false where possible.

### Doctrine ORM

- `entityManager->createQuery($dql)` — DQL is parameterized when used correctly; concat into `$dql` is injection.
- `entityManager->createNativeQuery($sql)` — Raw SQL; same parameterization rules.
- `findBy([...])` — Parameterized.

### Eloquent (Laravel)

- `User::whereRaw("..." . $x)` — Concat injection.
- `User::where('name', '=', $x)` — Safe.

### Other

- `mysql_query` (deprecated, removed PHP 7+) — String concat; injection.

## XML / XXE

- `simplexml_load_string($input)` — Modern PHP disables external entities by default; verify version.
- `DOMDocument::loadXML($input)` — Same.
- `LIBXML_NOENT` flag — DO NOT pass; enables entity expansion → XXE.
- `libxml_disable_entity_loader(true)` (deprecated PHP 8) — Modern PHP libxml2 has safer defaults.

## YAML

- `yaml_parse` (PECL extension) — Generally safe.
- Symfony `Yaml::parse` — Safe by default; `Yaml::PARSE_OBJECT_FOR_MAP` enables object instantiation, more risk.
- Spyc (third-party) — Verify version.

## Web Frameworks

### Laravel

- **Mass Assignment** — `$request->all()` to model `update()` without `$fillable` / `$guarded`. Verify each Eloquent model has appropriate guards.
- **CSRF** — `VerifyCsrfToken` middleware; verify all routes (web group) protected; API routes typically use bearer tokens.
- **Authentication** — `auth` middleware; `Auth::check()`, `Auth::user()`. Verify routes covered.
- **Authorization** — Policies and Gates; `$this->authorize(...)`; verify all sensitive actions.
- **Sessions** — Driver: file (default), database, redis. Verify production uses appropriate.
- **Queue jobs** — Serialized to job store; same considerations as serialization.
- **Validation** — `$request->validate(...)` with rules; verify strict.
- **`debug` mode** — `APP_DEBUG=true` — Reveals stack traces with code context; never in production.
- **Telescope, Horizon, Nova** — Verify auth on dashboards.
- **Encryption** — `Crypt::encrypt`, `Crypt::decrypt` — Use these instead of raw OpenSSL.
- **Routes** — `Route::get('/...', ...)` — Verify middleware applied.

### Symfony

- **Mass assignment** — Form components define allowed fields; verify forms don't allow arbitrary fields.
- **CSRF** — Form CSRF tokens; verify enabled.
- **Authentication** — Security bundle config; firewalls and providers; verify each firewall covers the right URL pattern.
- **Authorization** — Voters and `is_granted`; verify all sensitive actions.
- **Twig templating** — Auto-escape default; `{{ var | raw }}` disables; verify usage.
- **Doctrine** — DQL parameterization rules.
- **`debug` mode** — `APP_ENV=prod`, `APP_DEBUG=0` for production.

### CodeIgniter, Zend, Yii

- Each has its own pattern; same questions: mass assignment, CSRF, authn/authz, debug mode.

### WordPress / Drupal / Joomla

- Plugin / module ecosystems are huge attack surface.
- For WordPress: `wp-config.php` secrets, plugin updates, file editor in admin, REST API surface.
- For Drupal: rendering pipeline, Twig SSTI, configuration management.
- These are large topics; flag as needing specialized review if in scope.

## Templating

### Twig

- Default auto-escape (HTML).
- `{{ var | raw }}` disables; verify usage.
- SSTI when template source is user-controlled.

### Blade (Laravel)

- `{{ $var }}` — Auto-escapes.
- `{!! $var !!}` — Unescaped; XSS if user-controlled.

### Smarty

- `<{$var}>` — Auto-escape config; verify.

## Crypto

### `openssl_*` Functions

- `openssl_encrypt($data, 'aes-256-gcm', $key, OPENSSL_RAW_DATA, $iv, $tag)` — AEAD; preferred.
- `openssl_encrypt(..., 'aes-256-cbc', ...)` — CBC; requires HMAC for integrity.
- `openssl_random_pseudo_bytes($n, $strong)` — Older; check `$strong` is true. Modern PHP: `random_bytes`.
- `random_bytes($n)`, `random_int($min, $max)` (PHP 7+) — CSPRNG; safe.

### `hash`, `hash_hmac`

- `hash('md5'/'sha1', ...)` for security — Wrong.
- `hash('sha256', ...)` — General hashing.
- `hash_hmac('sha256', $data, $key)` — HMAC.
- `hash_equals($known, $user)` — Constant-time comparison; use for tokens / MACs.

### Password Hashing

- `password_hash($pw, PASSWORD_BCRYPT, ['cost' => 12])` — Bcrypt; built-in.
- `password_hash($pw, PASSWORD_ARGON2ID)` — Argon2id; PHP 7.3+.
- `password_verify($pw, $hash)` — Constant-time; use this.
- `password_needs_rehash($hash, ALGO)` — For migration.
- Never `md5($password)`, `sha1($password)`, `crypt($password, ...)` (without proper salt format).

### Random

- `mt_rand`, `rand` — NOT cryptographic; never for tokens.
- `random_bytes`, `random_int` — Use these.

### TLS

- `curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false)` — Disabled verification.
- `stream_context_create(['ssl' => ['verify_peer' => false]])` — Same.

## Network

### cURL, Guzzle, Symfony HttpClient

- SSRF risks; covered in `ssrf-redirect-url.md`.
- `CURLOPT_FOLLOWLOCATION = true` (default off) — May enable; verify redirect target validation.
- `CURLOPT_PROTOCOLS` — Restrict to `CURLPROTO_HTTP | CURLPROTO_HTTPS` to block file://, gopher://, etc.

### `file_get_contents` with URL

- Equivalent to HTTP request; same SSRF concerns.
- `allow_url_fopen` setting controls; verify if disabled (good practice for restricted contexts).

### `gethostbyname`

- DNS resolution as side effect; reveals existence to DNS attacker.

## File Handling

- `file_get_contents($path)` with user input — Path traversal; or RFI if `allow_url_fopen`.
- `realpath($BASE . $user)` — Resolves; verify result starts with BASE.
- `pathinfo`, `basename`, `dirname` — String operations; not security-critical alone but used in path validation.
- `move_uploaded_file($tmpName, $dest)` — Verify `$dest` is within intended dir.
- `ZipArchive::extractTo($dest)` — Zip Slip; iterate manually with canonicalization.
- `tempnam`, `sys_get_temp_dir` — Standard temp; verify cleanup.

## Common Library Footguns

### Composer / Packagist

- See `dependencies.md`.

### `mail` Function

- `mail($to, $subject, $body, $headers)` — Header injection if `$headers` derived from user input with CRLF; recipient injection in `$to`.
- Use PHPMailer / Symfony Mailer with structured APIs.

### `phpmailer`

- Older versions had RCE via injection in sender field (CVE-2016-10033); track version.

### Magic Methods Risk

- `__wakeup`, `__destruct`, `__toString`, `__call`, `__get` — Invoked during deserialization; gadgets exploit these.

### Configuration Files

- `php.ini` settings — `display_errors = Off`, `error_reporting`, `expose_php = Off` for production.
- `.htaccess` rules — Verify no debug routes exposed.

## PHP Version Notes

- PHP 5.x — EOL since 2018; any PHP 5 code is a finding.
- PHP 7.0/7.1/7.2/7.3 — Out of support.
- PHP 7.4 — Security-only support ended; soon EOL.
- PHP 8.0+ — Current; verify 8.x for active support.

## Common Findings Patterns

- `eval($_POST['code'])` — RCE.
- `system("cmd " . $_GET['x'])` — Command injection.
- `unserialize($_COOKIE['data'])` — Object injection.
- `mysql_query("SELECT ... " . $_GET['id'])` — SQL injection (and deprecated).
- `mysqli_query($conn, "SELECT ... " . $_GET['id'])` — SQL injection.
- `include $_GET['page'] . '.php'` — LFI/RFI.
- `User::find($id)->update($_POST)` (Eloquent) without `$fillable` — Mass assignment.
- `password_hash($pw, PASSWORD_BCRYPT, ['cost' => 4])` — Cost too low.
- `mt_rand(...)` for token generation.
- `md5($password)` for password storage.
- `CURLOPT_SSL_VERIFYPEER, false` — TLS bypass.
- `extract($_POST)` — Variable injection.
- `APP_DEBUG=true` in production `.env`.
- `LIBXML_NOENT` enabled — XXE.
- `eval($content)` where `$content` is from a file written by user upload — RCE chain.

## Recommendation Patterns

- Use `random_bytes` / `random_int` for any randomness in security context.
- Use `password_hash` / `password_verify` for passwords.
- Use `hash_equals` for token / MAC comparison.
- Use `PDO` with prepared statements; disable `ATTR_EMULATE_PREPARES`.
- Use `unserialize($data, ['allowed_classes' => false])` if forced to use unserialize; otherwise migrate to JSON.
- Use `proc_open` with arg array instead of `system`/`exec`.
- For Laravel: `$fillable` / `$guarded` on every Eloquent model; CSRF enabled; `APP_DEBUG=false` in production.
- For Symfony: `APP_ENV=prod`, `APP_DEBUG=0` in production; security firewalls explicit.
- Disable `display_errors` in production php.ini.
- Pin dependencies via `composer.lock`; run `composer audit` (PHP 8.2+) or `roave/security-advisories` in CI.
- Disable `allow_url_include`; restrict `allow_url_fopen` if not needed.
