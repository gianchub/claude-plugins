# Ruby Security Footguns

## Scope

Ruby on Rails, Sinatra, Hanami, raw Ruby. Cross-reference `deserialization.md` (Marshal, YAML), `injection.md` (system/exec), `crypto.md` (OpenSSL).

## Code Execution Sinks

### `eval`, `instance_eval`, `class_eval`, `module_eval`

- All evaluate Ruby code; user input → RCE.
- `instance_eval` particularly common in DSLs and template engines.
- Never on untrusted input.

### `Object.const_get`, `Object.const_set`, `send`, `public_send`

- `Object.const_get(userInput)` — Resolves constant by name; combined with `.new` allows arbitrary instantiation.
- `obj.send(method, args)` with user-controlled method — Calls private methods (use `public_send` for safer subset, but allowlist still required).
- `Class.new(obj.const_get(...))` — Class creation pivots.

### `system`, `exec`, `Kernel#open`, backticks

- `system("cmd #{userInput}")`, `exec(...)` — Single string form is shell-interpreted; injection.
- `system("cmd", arg1, arg2, ...)` — Argv form; safer.
- Backticks `` `cmd #{user}` `` — Same as `system`.
- `Kernel#open` (or `URI.open`) — Historic gotcha: `open("|cmd")` executes a command if argument starts with `|`. Use `URI.parse(...).open` or `Net::HTTP` for HTTP, `File.open` for files.
- `IO.popen("cmd")` — Same risks.

### `Marshal.load`, `YAML.load`

- See `deserialization.md`. RCE via gadgets.

### `Process.spawn`, `Process.exec`

- Same shell vs. argv distinction as `system`.

### `define_method`, `method_missing`

- Dynamic method definition; if user input determines the method name or body, code injection.

## SQL / DB

### ActiveRecord

- `User.where("name = '#{x}'")` — Concat; SQL injection.
- `User.where("name = ?", x)` — Safe.
- `User.where(name: x)` — Safe (hash form).
- `User.find_by_sql("..." + x)` — Concat injection.
- Dynamic ORDER BY via user input — Allowlist column names.
- `User.connection.execute("..." + x)` — Raw concat.

### Sequel

- `dataset.where(Sequel.lit("..." + x))` — Concat.
- `dataset.where(name: x)` — Safe.

## Web Frameworks

### Rails

#### Mass Assignment

- `User.update(params[:user])` without `permit` — Mass assignment; `User.update(params.require(:user).permit(:name, :email))`.
- Strong Parameters (`params.permit(...)`) — Defense; verify every controller uses it.
- `accepts_nested_attributes_for` — Nested mass assignment; verify nested permit.

#### CSRF

- `protect_from_forgery` in `ApplicationController` — Verify present.
- `protect_from_forgery with: :exception` (default in modern Rails) — Recommended.
- `skip_before_action :verify_authenticity_token` — Verify justified per-controller (typically API endpoints with separate auth).

#### Authentication / Sessions

- `session[:user_id] = user.id` — Server-side store recommended for sensitive data; cookie-store works but verify rotation and secret.
- `cookies.signed[:user_id]` — Tamper-evident; verifies HMAC.
- `cookies.encrypted[:user_id]` — Encrypted; preferred for non-public values.
- `Devise` gem — Industry standard; verify configuration.

#### Routes / Controllers

- `permit` excludes role/admin fields if those are server-managed.
- `before_action :authenticate!` applied to controllers; verify per-action.
- `cancancan` / `pundit` — Authorization; verify policies cover all actions.

#### Templates

- ERB `<%= x %>` — Auto-escapes in Rails (since 3.0). `<%= raw x %>` — Disables escape.
- `html_safe` method — Marks string as already-escaped; if user input flows through `html_safe`, XSS.
- `sanitize` — Built-in HTML sanitizer; strict allowlist.

#### File Handling

- `Rails.root.join(BASE, params[:file])` — Path traversal if not validated.
- `send_file path` with user input — Same; use `send_data` with bytes from a known-safe source or validate path.

#### Rails Shell

- Production credentials (`config/credentials.yml.enc` + `config/master.key`) — Master key must not be committed; rest is encrypted.
- Older Rails: `secrets.yml`, `database.yml` with plaintext secrets.

#### XML / YAML

- `params` with `Content-Type: application/xml` — Default XML parsing in older Rails was YAML-based and unsafe (CVE-2013-0156). Modern Rails (5+) safe.
- `YAML.load` on params — Verify usage.
- `to_yaml` / `from_yaml` — Same as YAML.load.

### Sinatra

- Lightweight; explicit configuration required for sessions, CSRF, etc.
- `enable :sessions` — Enable; verify secret.
- `params[...]` — Sources.

### Hanami

- Modern framework; explicit; same security considerations applied per layer.

## Templating

- ERB — Auto-escape in Rails; not in raw Ruby (`require 'erb'`).
- Haml, Slim — Similar to ERB; verify escape behavior.
- Liquid (Shopify) — Sandboxed by design; SSTI-resistant; verify version.

## Crypto

### OpenSSL

- `OpenSSL::Digest.new('MD5'/'SHA1')` — Not for security purposes.
- `OpenSSL::HMAC.digest('SHA256', key, data)` — Use HMAC-SHA-256.
- `OpenSSL::Cipher.new('AES-256-GCM')` — Modern AEAD.
- `OpenSSL::Cipher.new('AES-256-CBC')` — Requires HMAC for integrity.
- IV / nonce — `OpenSSL::Random.random_bytes(n)`.

### Password Hashing

- `bcrypt` gem — `BCrypt::Password.create(password, cost: 12)` — Cost ≥ 12.
- `argon2` gem — Argon2id; preferred.
- Never `Digest::MD5.hexdigest(password)`.

### Constant-Time Comparison

- `ActiveSupport::SecurityUtils.secure_compare(a, b)` — Constant-time; use for tokens, MACs.

### Random

- `SecureRandom` module — `SecureRandom.hex`, `SecureRandom.urlsafe_base64`, `SecureRandom.random_bytes` — CSPRNG; safe.
- `Random.rand` (top-level `rand`) — NOT cryptographic; never for tokens.

### TLS

- `Net::HTTP.start(host, port, use_ssl: true, verify_mode: OpenSSL::SSL::VERIFY_NONE)` — Disabled verification; finding.
- `OpenSSL::SSL::VERIFY_PEER` — Default; should be retained.

## Network

### Net::HTTP, HTTParty, RestClient, Faraday

- SSRF risks; covered in `ssrf-redirect-url.md`.
- `OpenSSL::SSL::VERIFY_NONE` — Disabled TLS; finding.
- Default redirect handling — Verify per library.

### `URI.open`, `Kernel#open`

- See above; `open` with `|cmd` is exec.

## File Handling

- `File.open(path)` with user-controlled path — Path traversal.
- `File.expand_path(user, BASE)` — Resolves; verify it's within BASE after.
- `Pathname.new(BASE).join(user).realpath` — Resolves; verify with `start_with?(BASE)`.
- `tarfile` / `archive` gems — Manual canonicalization for Zip Slip.
- `Tempfile` — Safe; uses `mkstemp`.

## Common Library Footguns

### Devise

- Mass assignment via `User.update(params[:user])` without permit; check Devise controller customization.
- Token rotation on critical actions (password change, email change).
- Default password policies — Devise's are minimal; reinforce.

### Pundit / CanCanCan

- Policy classes per resource; verify all actions covered.
- Pundit `authorize @resource` — Forgot to call → no authz; integrate ensure-authorized in tests.

### Sidekiq, Resque

- Background jobs; serialized arguments. If args originate from user input, they're an extension of the source.
- Sidekiq Web UI — Verify auth on `/sidekiq` route.

### `mail` gem

- Email sending; verify recipients aren't user-controlled in ways that allow header injection.
- `Mail::Address` for parsing; safe.

### Carrierwave / Active Storage

- File uploads; verify validation (content type, size, filename).
- Direct upload via signed URLs — Verify expiration and scope.

### Rails Console / Engine Mounts

- Mounting `Rails::Engine` for admin tools — Verify auth.
- Web Console (`web-console` gem) — Allows REPL in browser; never enabled in production.

## Common Findings Patterns

- `system("ls #{params[:dir]}")` — Command injection.
- `User.where("name = '#{params[:name]}'")` — SQL injection.
- `User.update(params[:user])` without permit — Mass assignment.
- `eval(params[:expr])` — RCE.
- `YAML.load(File.read(...))` from untrusted — RCE.
- `Marshal.load(cookie_value)` — RCE.
- `OpenSSL::SSL::VERIFY_NONE` — TLS bypass.
- `BCrypt::Password.create(pw, cost: 4)` — Cost too low.
- `rand(...)` for generating tokens.
- `Digest::SHA256.hexdigest(password)` for password storage.
- `Kernel#open(params[:url])` — RCE if URL starts with `|`.
- `params.permit!` (allow all) — Defeats Strong Parameters.
- `protect_from_forgery` skipped on critical controllers.
- `web-console` gem in production Gemfile.

## Recommendation Patterns

- Use Strong Parameters; explicit `permit` lists; never `permit!`.
- Use parameterized SQL via ActiveRecord hash form or `?` placeholders.
- Use `bcrypt` (cost ≥ 12) or `argon2` for passwords.
- Use `SecureRandom` for any token; never `rand`.
- Use `ActiveSupport::SecurityUtils.secure_compare` for MACs / tokens.
- Use `URI.parse(url).open` or `Net::HTTP` directly, not `Kernel#open` for URLs.
- Apply `protect_from_forgery` globally; opt out per controller with justification.
- Use Pundit / CanCanCan for authz; `authorize` everywhere.
- Pin gems via `Gemfile.lock`; `bundler-audit` in CI.
- Encrypt credentials with Rails encrypted credentials; never commit master key.
- Disable `web-console` and `byebug` in production groups.
