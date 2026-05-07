# SSRF, Redirects, and URL Handling

## Scope

Server-Side Request Forgery (SSRF), open redirects, URL parser confusion, DNS rebinding, and host header injection. The unifying theme: a URL or hostname derived from user input flows into a server-side network operation or response context where the destination matters for security.

## Server-Side Request Forgery (SSRF)

The attacker causes the application's server to make an HTTP/network request to a destination they control or to an internal destination they should not reach.

### Sources to Trace

- **HTTP egress with user-controlled URL** — `requests.get(req.body.url)`, `axios.get(req.body.url)`, `fetch(userUrl)`, `URL.openConnection(userUrl)`, etc.
- **Indirect URL controls** — User supplies a hostname (`?host=...`), port, path, or query separately, and the server constructs a URL. Validate the assembled URL, not just the parts.
- **Webhook delivery** — User configures a webhook URL; server delivers events. Must validate URL on configuration AND at delivery time (TOCTOU on DNS).
- **PDF / image / preview generators** — User supplies content with embedded URLs; server fetches them during rendering. WeasyPrint, wkhtmltopdf, headless browsers, image processors all fetch resources.
- **OEmbed / link previews / unfurl** — Server fetches URL the user posted to generate preview metadata.
- **OAuth flows / redirect-following** — Token endpoint URL or callback URL controlled by client.
- **Federated features** — ActivityPub, Matrix, RSS aggregation: server fetches arbitrary URLs by design. Validate but expect tighter constraints.
- **XML / SVG / image library URL fetches** — XXE; SVG `<image href="http://...">`; PDF embedded resources. Often hidden in third-party libraries.
- **DNS resolution as side effect** — `socket.gethostbyname(userHost)` triggers DNS resolution; even without an HTTP fetch, DNS leaks internal infrastructure to attacker.

### Attack Targets

- **Cloud metadata endpoints** — `169.254.169.254` (AWS, GCP, Azure, Alibaba, OpenStack), `metadata.google.internal`, `100.100.100.200` (Aliyun). On EC2, returns IAM role credentials. Mitigate with IMDSv2 (require token); but application code must still block.
- **Localhost services** — `127.0.0.1`, `localhost`, `0.0.0.0`, `[::1]` reach in-process or sidecar services (admin endpoints, monitoring, cache).
- **Private network ranges** — `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10` (CGNAT). Internal databases, internal APIs.
- **Link-local** — `169.254.0.0/16`, `fe80::/10` — metadata, link-local services.
- **Loopback IPv6** — `::1`, IPv4-mapped IPv6 (`::ffff:127.0.0.1`).
- **Bypassing IPv4 checks via IPv6** — `[::ffff:127.0.0.1]` or `0.0.0.0` may skip naive IPv4 string-prefix checks.
- **DNS-resolver-controlled targets** — `attacker.example.com` resolves to `127.0.0.1`; allowlists on hostname strings without resolution don't catch.
- **Other protocols** — `file://` (read local files), `gopher://` (raw TCP, can hit Redis/SMTP), `dict://`, `ldap://`, `ftp://`, `tftp://`. Many HTTP libraries support these by default.
- **Localhost via 0.0.0.0 or unusual decimal forms** — `0`, `0.0`, `127.1`, `2130706433` (decimal), `0x7f000001` (hex), `017700000001` (octal).

### Defenses

- **Allowlist hosts** — When the destination set is known (specific third-party APIs), allowlist hostnames; reject everything else. The strongest defense.
- **Block private/reserved/link-local ranges** — Resolve the hostname server-side; reject if any resolved address is in a reserved range. Re-resolve right before connecting (DNS rebinding mitigation).
- **Disable redirects or limit redirect depth** — A response redirecting to `127.0.0.1` bypasses initial host validation; either disable HTTP redirects or revalidate the new target on each redirect.
- **Restrict protocol** — Allow only `http://` and `https://` if those are the only required schemes. Reject `file://`, `gopher://`, etc., explicitly; do not rely on default behavior.
- **Egress firewall / VPC** — Network-level controls preventing the application's network from reaching internal endpoints. Strong defense; verify configured. Document in Intent Brief.
- **IMDSv2 (AWS) / GCE-MDS-Tokens (GCP)** — Require a token to access metadata; reduces SSRF impact on cloud metadata. Strong defense; verify enforced.
- **Use libraries with built-in SSRF protection** — `safeurl-py`, `ssrf-protection`, `defenders-bag` style; avoid hand-rolled validation.

### URL Parser Confusion

URL parsers across languages and libraries handle edge cases differently. The same URL string may yield different `host`, `path`, `query` in different parsers. When validation parses the URL one way and the network library parses it another, attacker can split the difference.

- **Userinfo abuse** — `http://allowed.example.com@attacker.example.com/`. The host is `attacker.example.com`; some parsers report `allowed.example.com` as host.
- **Port confusion** — `http://allowed.example.com:80@attacker.example.com/`.
- **Path confusion** — `http://attacker.example.com/.allowed.example.com`. Substring matches on URL fail.
- **Backslash handling** — Browsers and libraries differ on `http://allowed.example.com\@attacker.example.com`.
- **Encoded characters** — `http://allowed.example.com%2F.attacker.example.com/` — depends on whether `%2F` is decoded before host parsing.
- **Multiple `@` signs** — Various parsers split differently.
- **Unicode normalization** — Punycode/IDN; visually-confusable hostnames.
- **Trailing dots** — `host.example.com.` may equal `host.example.com` to some validators but not others.
- **Bracket confusion (IPv6)** — `[::1]` vs `::1`.

Defense: **always parse the URL with the same library you'll use to make the request, then validate the parsed host/port/scheme directly**. Never validate by string operations on the URL.

### Blind SSRF

Application doesn't return the response body; attacker uses out-of-band channels to confirm. Common signals:

- DNS request to attacker-controlled domain when the application resolves a hostname.
- HTTP request to attacker-controlled server (via callback features, image preview, etc.).
- Time-based (request to fast vs. slow internal endpoint).

Even blind SSRF reaching cloud metadata or internal services is exploitable.

## Open Redirect

Application redirects the browser to a URL derived from user input, allowing attackers to use the application as a redirect to a malicious site. Often combined with phishing or used to chain into XSS via `javascript:` URLs.

- **`res.redirect(req.query.next)`** without validation. The `next`/`continue`/`returnTo` parameter pattern is the canonical case.
- **Fix**: validate redirect targets against an allowlist of paths (relative URLs starting with `/`, no `//` to prevent protocol-relative redirects to other hosts), or against an allowlist of full URLs.
- **`javascript:` URLs in redirects** — Some frameworks/clients allow `javascript:alert(1)` as a Location target → XSS. Reject any non-http(s) scheme.
- **Subdomain confusion** — Allowlisting `example.com` may also allow `evil.example.com` if subdomain takeover risk exists.

## DNS Rebinding

The attacker registers a domain whose DNS responses change between requests (or between resolution and connection): first response returns the application server's allowed external IP (or any allowed IP), second response returns `127.0.0.1` or a private IP. The application validates the first IP, then connects to the second.

- **Mitigation 1**: pin DNS — resolve once, use the resolved IP for the actual request, do not re-resolve.
- **Mitigation 2**: validate the connected IP after connection, not just the hostname / first resolution.
- **Mitigation 3**: short DNS cache TTL is a worse mitigation; attacker controls the TTL.

DNS rebinding is also a browser-side attack against locally-running services (printers, dev servers, apps); document if applicable.

## Host Header Injection

User-controlled `Host` header reflected in:

- Generated URLs (e.g., password-reset emails) — attacker-controlled host means reset link points to attacker-controlled domain, which captures the token.
- Cache keys — cache poisoning if `Host` is part of cache key but not validated.
- Server-side routing — multi-tenant routing by host header bypasses tenant boundary if the application doesn't validate the host against the expected set.

Defenses:

- Validate `Host` against an allowlist; reject unexpected hosts (web server level: nginx `server_name`).
- Don't use `Host` for security-relevant URL generation; use a trusted base URL from configuration.
- For multi-tenant by host, normalize and validate.

## Recommendation Patterns

- Allowlist destinations rather than blocklist; allowlists fail closed, blocklists are bypassable.
- Resolve and re-validate hosts at connection time; don't trust the user-supplied hostname or even the first DNS response.
- Disable or strictly limit HTTP redirects in server-side fetchers.
- For features that require fetching arbitrary URLs (link previews, federation), put fetchers in an isolated network segment with no access to internal resources.
- For cloud-hosted apps, enforce IMDSv2 or the equivalent metadata-token requirement at the infrastructure level.
- Use `Host` and `X-Forwarded-Host` only after validation against an allowlist.
- Construct application URLs from a trusted, configured base, not from request headers.
