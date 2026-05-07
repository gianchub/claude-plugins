# XSS, CSRF, and Frontend Security

## Scope

Browser-side weaknesses: cross-site scripting (stored, reflected, DOM-based, mutation), cross-site request forgery, clickjacking, security headers, cookie flags, postMessage handling, subresource integrity, and prototype pollution. Applies to any application that serves HTML or interacts with browsers; out of scope for pure CLI tools and machine-to-machine APIs.

## Cross-Site Scripting (XSS)

### Stored XSS

- **Persisted user input rendered without encoding** — Comments, profile fields, document titles, file names, log viewer pages. Look at every place server-side templates render database content; verify encoding context.
- **Rich-text editors** — TinyMCE, Quill, ProseMirror, Slate often output HTML stored verbatim. Sanitize with DOMPurify (browser) or bleach/sanitize-html (server) using strict allowlists. Verify the sanitizer is invoked and the configuration is restrictive.
- **Markdown rendering** — Markdown processors that allow inline HTML (`marked` with `gfm: true` plus no sanitization, `markdown-it` without `html: false`) preserve `<script>` tags. Configure to disable HTML, or post-sanitize.
- **HTML inside JSON** — JSON response includes HTML strings that the client renders via `innerHTML` or `dangerouslySetInnerHTML`. Sink-side problem; address at the consumer.
- **SVG uploads / inline SVG rendering** — SVG is XML and runs script. Treat user-uploaded SVG as HTML for XSS purposes. Strip `<script>`, event handlers, and `xlink:href="javascript:..."` server-side.
- **Filename rendering** — Uploaded filenames containing `<script>` rendered in admin UI / file lists.

### Reflected XSS

- **Query/path/header values reflected in error pages** — `<h1>404: ${path}</h1>` with no encoding. Audit error pages, search-result pages, login pages echoing username.
- **Reflected via headers** — `Referer`, `Host` reflected in response (custom analytics or debug pages). Particularly bad: `X-Forwarded-Host` reflected, allowing host header injection → XSS.
- **JSONP-style endpoints** — `?callback=alert(1)` reflected as `alert(1)({...})`; even if response is `application/javascript`, browsers execute when loaded as script. Strict callback name validation (allowlist `[A-Za-z_][A-Za-z0-9_]*`) is required.
- **Dynamic JavaScript built from input** — `<script>var x = "${userInput}";</script>` — even with HTML-escape, JavaScript context requires JS-escape. Audit any script block built from user data.

### DOM XSS

- **Sinks** — `element.innerHTML`, `outerHTML`, `insertAdjacentHTML`, `document.write`, `document.writeln`, `eval`, `Function`, `setTimeout(string)`, `setInterval(string)`, `location = ...`, `location.href = ...`, jQuery `$(html)`, `$(...).html(html)`, React `dangerouslySetInnerHTML`, Vue `v-html`, Angular `[innerHTML]` (without sanitizer trust override), Svelte `{@html ...}`.
- **Sources** — `document.location`, `document.URL`, `document.referrer`, `window.name`, `localStorage`, `sessionStorage`, `postMessage` data, `document.cookie`, fragment (`location.hash`), URL parameters parsed client-side.
- **Trace each DOM source to each DOM sink.** Common manifestation: `location.hash` or query parameter parsed and inserted into the DOM.
- **Frameworks default-encode** — React, Vue, Angular auto-encode by default; the bypasses are explicit (`dangerouslySetInnerHTML`, `v-html`, `bypassSecurityTrustHtml`). Each use is a finding worth reviewing.
- **`href` and `src` attributes** — `<a href={userUrl}>` allows `javascript:alert(1)` URL. Validate scheme allowlist (http/https/mailto only).

### Mutation XSS (mXSS)

The browser parses what looks like safe HTML in a way that mutates into executable HTML. Sanitizers operating on input HTML may be bypassed when the HTML is reparsed in a different context (e.g., `<noscript>` content, `<svg>` content, namespace transitions).

- **Sanitize once, in the right context** — DOMPurify is the well-maintained browser sanitizer; configure correctly and use the specific output context (HTML body vs. SVG vs. MathML).
- **Server-side sanitizer that doesn't account for browser quirks** — Sanitize on the rendering side or use a sanitizer that understands target browser behavior.

### Trusted Types (Mitigation)

- Modern Chromium browsers support the Trusted Types API: `trustedTypes.createPolicy(...)` plus `Content-Security-Policy: require-trusted-types-for 'script'` blocks injection sinks.
- Adoption is a positive signal in the Intent Brief; absence in modern apps is a recommendation, not a finding.

## Cross-Site Request Forgery (CSRF)

### When CSRF Applies

- Any state-changing request authenticated by ambient credentials (cookies, HTTP Basic, client certs).
- Pure-bearer-token APIs (JWT in Authorization header) typically do not need CSRF protection because the token isn't sent automatically.
- Cookie-bound JWTs *do* need CSRF protection.

### CSRF Defenses

- **`SameSite=Lax` (default in modern browsers)** — Mitigates most CSRF for top-level navigations. Verify cookies have `SameSite=Lax` or `Strict`. `SameSite=None` is required for cross-origin cookies but combined with CSRF tokens is necessary.
- **CSRF token (anti-forgery token)** — Server-issued token bound to session; required on every state-changing form. Verify token is unpredictable, bound to the session/user, and validated on every state-changing endpoint.
- **Double-submit cookie pattern** — Token in cookie + matching header/form value. Acceptable for sessionless apps; verify cookie has `SameSite` and the comparison is constant-time.
- **Origin / Referer validation** — Verify `Origin` header matches an allowlist of expected origins. Less robust alone (header may be missing); use as part of layered defense.
- **Custom header** — Requiring a custom header (e.g., `X-Requested-With: XMLHttpRequest`) plus strict CORS prevents simple form-based CSRF. Effective only with strict CORS.

### CSRF Bypass Patterns

- **Token issued but not validated** — Token exists in form but server doesn't check; common copy-paste bug.
- **GET requests changing state** — `GET /api/account/delete` violates HTTP semantics and is CSRF-vulnerable even with token (image src, link). Audit any GET that mutates.
- **JSON with `Content-Type: text/plain`** — Browser treats as simple request, no preflight; if server accepts, CSRF possible. Reject non-JSON Content-Type for JSON endpoints.
- **Per-form token reuse** — Same token across many actions: lower attacker bar but still better than nothing; per-action tokens are stronger but high-cost.
- **Token in URL** — Logged in proxy logs and Referer; mitigate.

### CORS and CSRF

CORS does not protect against CSRF for "simple" requests (most form posts). Wide-open CORS (`Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: false`) does not enable CSRF beyond what cookies do. CORS with `Access-Control-Allow-Credentials: true` and reflected `Origin` is dangerous: any origin can read responses including authenticated data. Review CORS config separately:

- **`Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true`** — Browsers reject this combination, but framework misconfig may produce it; verify resulting headers.
- **Reflected `Origin` header** — `Access-Control-Allow-Origin: <whatever the request sent>` — effectively no origin restriction. Allowlist explicit origins.
- **`null` origin** — Some setups allow `null`, which a sandboxed iframe / data URL can produce; never allow.
- **Wildcard subdomain** — `*.example.com` can allow `evil.example.com` if a takeover-vulnerable subdomain exists. Audit subdomain ownership.

## Clickjacking / UI Redressing

- **`X-Frame-Options: DENY` or `SAMEORIGIN`** — Header on all pages that perform sensitive operations.
- **`Content-Security-Policy: frame-ancestors`** — More flexible; preferred in CSP-aware setups. Required for modern browsers.
- **JavaScript framebuster** — Easily bypassed; do not rely.

## Security Headers

Audit response headers globally and per route. Each header has a recommended value:

| Header | Recommended | Notes |
|--------|-------------|-------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | Once preloaded, hard to revert; verify coverage. |
| `Content-Security-Policy` | Strict; nonce or hash for scripts | Auditing CSP is its own subskill (see below). |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME sniffing of responses to scripts/styles. |
| `Referrer-Policy` | `strict-origin-when-cross-origin` or `same-origin` | Limits leakage of URL paths and tokens via Referer. |
| `Permissions-Policy` | Deny by default; allow only what's used | Limits browser features (camera, geolocation, payment). |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | Subsumed by CSP `frame-ancestors`. |
| `Cross-Origin-Opener-Policy` | `same-origin` | Isolates browsing context group; required for SharedArrayBuffer. |
| `Cross-Origin-Resource-Policy` | `same-origin` or `same-site` | Blocks cross-origin reading. |
| `Cross-Origin-Embedder-Policy` | `require-corp` (when needed) | Together with COOP enables cross-origin isolation. |
| `Cache-Control` | `no-store` for sensitive responses | Prevents proxy / browser caching of authenticated content. |

Missing header: typically Medium severity at most, defense-in-depth. Combination of misconfigs may amplify other vulnerabilities — note in description.

### CSP Audit

A Content Security Policy is strong only when strict. Common weaknesses:

- **`unsafe-inline`** in `script-src` — defeats most XSS protection. Only acceptable as transitional; recommend nonce or hash.
- **`unsafe-eval`** in `script-src` — required by some frameworks (older Angular) but enables `eval`-based XSS. Recommend removing if possible.
- **Wildcard hosts** — `https://*.example.com` allows takeover-prone subdomains. Audit each allowed origin.
- **Allowlisting JSONP-friendly CDNs** — `googleapis.com`, `azureedge.net`, etc., often serve attacker-controlled JS via JSONP endpoints. Use CSP nonce/hash instead.
- **`object-src` not `none`** — `<object>`, `<embed>` are XSS vectors via Flash/legacy plugins. Default to `'none'`.
- **Missing `base-uri`** — `<base>` can change relative URL resolution; restrict to `'self'` or `'none'`.
- **`script-src 'self'`** with file upload — User-uploaded `.js` served from same origin executes. Don't serve user uploads from app origin, or set strict Content-Type.
- **CSP only via `<meta>` tag** — Some directives (frame-ancestors, sandbox) are ignored from meta; use header.
- **CSP not in report-only mode for new rollouts** — Useful for tuning without breaking the app.

## Cookie Security

- **`Secure`** — Required on auth cookies; ensures HTTPS-only.
- **`HttpOnly`** — Required on auth cookies; prevents JS access (mitigates token theft via XSS).
- **`SameSite`** — `Lax` or `Strict` for auth cookies. `None` requires `Secure` and is mostly for cross-origin embeds.
- **`Domain`** — Set as narrowly as possible. Setting on a parent domain shares cookie with all subdomains; subdomain takeover risk amplified.
- **`Path`** — Default `/` is fine; narrower paths can reduce exposure if the app architecture supports.
- **`__Host-` prefix** — Indicates strict cookie scoping (no Domain, Path=/, Secure, HTTPS); strong signal. `__Secure-` weaker.
- **Long-lived cookies for non-session purposes** — Tracking cookies separate from session cookies; verify scoping.

## postMessage Security

- **Origin check on receiving** — `event.origin === expected` constant; never trust `event.origin` as the value to allowlist (literally allowing anything).
- **Source check** — `event.source === expectedWindow` for stricter validation.
- **Avoid wildcard target origin in `postMessage(message, '*')`** — Any frame embedding the page receives the message, including any sensitive content.
- **Treat received message data as untrusted** — Validate structure, types, and value ranges. Same dataflow rules as HTTP input.
- **Combined with framing** — When the page can be framed, postMessage from the framing context is attacker-controllable.

## Prototype Pollution (JavaScript-Specific)

- **Sinks** — `Object.assign(target, userInput)`, `_.merge(target, userInput)`, `_.extend`, `$.extend` (deep), JSON-merge libraries. User input with key `__proto__`, `constructor.prototype`, or `prototype` writes to the prototype chain.
- **Impact** — Pollutes object prototypes; downstream code reading `obj.someProp` may see attacker-controlled value when `obj` doesn't have its own; can lead to RCE in specific code patterns (e.g., template engines that look up `prototype` properties).
- **Mitigations** — Use libraries with prototype-pollution guards (`lodash@4.17.21+` has fixes for some operations), use `Object.create(null)` for maps that hold user data, validate keys against allowlist before merge, freeze prototypes (`Object.freeze(Object.prototype)`).

## Subresource Integrity (SRI)

- Third-party scripts loaded from CDN should include `integrity="sha384-..."`.
- Missing SRI for npm/CDN-hosted scripts is a defense-in-depth finding (CDN compromise impacts the application).
- For self-hosted bundles SRI is less critical but still good practice with versioned URLs.

## Mixed Content / Insecure Loads

- All resources (scripts, images, iframes, fetch) loaded over HTTPS when the page is HTTPS.
- `Content-Security-Policy: upgrade-insecure-requests` automates upgrades.

## Recommendation Patterns

- Use frameworks' default-encoding paths; treat any escape from defaults (`dangerouslySetInnerHTML`, `v-html`, `safe`) as findings unless trust is documented.
- Set strict CSP; add nonces or hashes; block `unsafe-inline`/`unsafe-eval` if at all possible.
- Cookies: `Secure` + `HttpOnly` + `SameSite=Lax` (or `Strict`) for auth.
- Validate Origin/Referer on state-changing endpoints; combine with CSRF tokens.
- For multi-origin embeds, document the origin allowlist explicitly.
- DOMPurify for any rich-text rendering; bleach / sanitize-html on the server side as defense-in-depth.
