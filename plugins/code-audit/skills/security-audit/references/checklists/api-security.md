# API Security

## Scope

API-specific security concerns mapped to the OWASP API Security Top 10 (2023). Many items overlap with other checklists (`authorization.md` covers BOLA/BFLA in depth; `injection.md` covers injection); this checklist focuses on what's API-specific or what gets missed when only thinking in web-app terms. Apply to REST, GraphQL, gRPC, WebSocket, and other API surfaces.

## API1:2023 — Broken Object Level Authorization (BOLA)

Covered in depth in `authorization.md`. API-specific notes:

- **REST**: every endpoint with a path parameter is a candidate; trace each.
- **GraphQL**: every field resolver returning per-object data is a candidate; framework-level guards rarely cover all fields.
- **gRPC**: every method receiving an object identifier; check authorization on each.
- **Batch endpoints** (`POST /api/users:batchGet`, GraphQL field-level fragmentation): batch authz often weaker than singular endpoint authz.
- **Cursor-based pagination tokens** opaquely encoding internal IDs: attacker decodes tokens to enumerate; verify authz on cursor-based reads.

## API2:2023 — Broken Authentication

Covered in depth in `auth-and-session.md`. API-specific notes:

- **API key in URL** — Logged in proxy logs, browser history, Referer. Use `Authorization` header.
- **Long-lived bearer tokens** — Without expiration; rotation on demand only. Pair with refresh tokens or short expiration.
- **OAuth 2.0 misuse**:
  - Client credentials grant for inter-service calls; verify scope.
  - Resource owner password credentials grant — deprecated, indicates trust assumptions that often don't match reality.
  - Implicit flow (deprecated; replaced by Authorization Code with PKCE).
- **Refresh token reuse without rotation** — Attacker who steals a refresh token retains access indefinitely. Rotate on each use; detect reuse and invalidate the lineage.
- **Server-to-server JWT without `aud`** — Token issued for service A reused at service B if audience not enforced.
- **Bearer tokens in WebSocket connect messages** — Verify on connect; revalidate on protocol-level events.

## API3:2023 — Broken Object Property Level Authorization

Covered in `authorization.md` (mass assignment + excessive data exposure). API-specific:

- **GraphQL field-level authz** — Schema may include sensitive fields per-type; resolver-level checks needed if fields aren't always permitted.
- **REST sparse fieldsets** (`?fields=...`) — Client requests specific fields; ensure server-side authorization decides what's returnable, not the request.
- **Returning entire ORM models** vs. **DTOs / serializers**:
  - Always return DTOs/serializers with explicit field allowlists.
  - Hidden danger: a new column added to a model auto-exposes if the serializer is `__all__` / `*`.
- **JSON Patch (RFC 6902)** — Each operation must be authorized; not just the top-level request.
- **Diff-based update endpoints** — Same as JSON Patch; allowlist paths.

## API4:2023 — Unrestricted Resource Consumption

The umbrella for rate-limiting, request-size, query-complexity, and concurrency limits.

### Rate Limiting

- **Per-IP** — Easy bypass via proxies / botnets but blocks unsophisticated abuse.
- **Per-user (post-auth)** — Stronger; required for authenticated abuse.
- **Per-API-key** — Required for public APIs.
- **Layered** — Edge/CDN limits + app-level limits + per-endpoint limits.
- **Backoff strategy** — Exponential backoff on repeated failures; not just constant rejection.
- **Headers** — `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (or `RateLimit-*` per RFC 9333). Useful for legitimate clients.
- **Sensitive flows special case** — Login, password reset, MFA enrollment, signup: stricter limits than general API.

### Request Size

- **Per-request body limit** — At the framework level: Express `express.json({ limit: '...' })`, Nginx `client_max_body_size`, etc. Default values may be too permissive.
- **Per-field length limits** — A "name" field accepting 10MB is wrong. Schema-level constraints.
- **File upload limits** — Covered in `file-handling.md`.
- **Streaming vs. buffering** — Large bodies streamed end-to-end vs. buffered into memory.

### Query Complexity (GraphQL)

- **Depth limits** — Reject queries deeper than N (typically 10).
- **Cost analysis** — Per-field cost; reject queries above total cost threshold.
- **Persisted queries** — Pre-registered queries; clients can only execute hash-identified queries; eliminates query crafting.
- **Introspection** — Disable in production for non-public APIs.
- **Aliases as query multipliers** — `{ user1: user(id:1), user2: user(id:2), ... user1000: user(id:1000) }` — apply alias count as part of complexity.

### Concurrency

- **Per-user / per-tenant concurrent request limits** — Prevent one user from saturating service capacity.
- **Long-running operations** — Async with status endpoint; not synchronous request blocking.
- **Backpressure** — Queue limits; reject when queue full rather than memory-exhaust.

### Per-Tenant Resource Quotas

- Compute (CPU minutes / quota units), storage, queue depth, requests/sec.
- Verified at code level, not just billing level.
- Document limits; communicate to customers.

## API5:2023 — Broken Function Level Authorization (BFLA)

Covered in `authorization.md`. API-specific:

- **HTTP method authorization** — `GET /api/users/:id` allowed for any user; `DELETE` and `PUT` should require admin or self. Verify each method.
- **Hidden endpoints** — Internal admin endpoints reachable from external network. Audit routing tables.
- **GraphQL mutation root** — Every mutation reachable from any authenticated user unless field-level authz enforces.
- **Custom verbs / actions** — `/api/users/:id/promote` style; same authz rules as anything else.

## API6:2023 — Unrestricted Access to Sensitive Business Flows

Sensitive business flows abused at scale, even when each individual call is authorized.

### Examples

- Account creation without anti-automation → mass fake accounts.
- Reservation / booking → competitive scraping or denial-of-availability for legitimate users.
- Purchase flow → carding (testing stolen cards), inventory exhaustion.
- Refund flow → repeated refund abuse.
- Password reset → DoS-by-spam to victim.

### Defenses

- Layered: rate limit (per IP, per identity, per recipient), CAPTCHA / behavioral signals, payment-method requirement, email/phone verification.
- Per-business-flow: design with abuse modes in mind, not just functional success.

## API7:2023 — SSRF

Covered in `ssrf-redirect-url.md`. API-specific:

- **Webhook test/dispatch endpoints** — User configures a webhook URL; server fetches. Validate at configuration AND at every dispatch.
- **Federation / fediverse APIs** — Fetch arbitrary remote resources by design; isolate the fetcher.
- **OEmbed / preview generators** — Often reachable as APIs.

## API8:2023 — Security Misconfiguration

API-specific configurations:

- **CORS** — `Access-Control-Allow-Origin: *` with credentials; reflected origin; null origin allowed; wildcard subdomain matching takeover-vulnerable subdomains.
- **HTTP method restrictions** — TRACE, TRACK, OPTIONS, PUT, DELETE on routes that don't expect them.
- **Verbose error responses** — Stack traces, debug info, schema disclosure.
- **Default credentials in admin APIs** — Especially in self-hosted products and infrastructure (databases, message brokers, monitoring tools).
- **Unauthenticated debug endpoints** — `/debug/pprof`, `/_debug`, `/api/v1/debug`, framework default error pages.
- **Open metrics / health endpoints** — Prometheus `/metrics` endpoint exposing internal state; verify authentication or network restriction.
- **Insecure defaults** — Frameworks defaulting to permissive (e.g., older Spring Boot exposing actuator endpoints; older Drupal/WordPress with default admin paths).
- **TLS configuration** — Old protocol versions enabled; weak cipher suites; covered in `crypto.md`.

## API9:2023 — Improper Inventory Management

What endpoints exist, where, and at what version — without an authoritative inventory, things slip through.

### Issues

- **Old API versions reachable** — `v1`, `v2`, `v3` all live; `v1` is deprecated and unmaintained but reachable; `v1` lacks `v3`'s authz fixes. Audit deprecation status; deprecate doesn't mean unreachable.
- **Staging endpoint exposed in prod** — `staging.example.com` accessible from internet, with weaker security controls.
- **Mobile-only API endpoints** — Designed for mobile client; reachable by anyone with the URL; same auth/authz checks must apply.
- **Internal-only endpoints** reachable externally due to misconfiguration.
- **GraphQL** — Schema introspection disabled in prod, but field discovery still possible via error messages or direct probing.
- **Documentation drift** — Real endpoints not documented; documented endpoints don't exist or have moved.
- **Test endpoints** — `/test`, `/healthz` (with payload echo), `/echo` left in prod.

### Defenses

- **API gateway as authoritative inventory** — All endpoints registered; deprecated endpoints disabled at the gateway.
- **Documented inventory** — `openapi.yaml` / `schema.graphql` with current state; CI verifies code matches schema.
- **Decommission policy** — Old versions removed (not just deprecated) on a timeline.

## API10:2023 — Unsafe Consumption of APIs

The application calls third-party APIs and trusts the responses unsafely.

### Patterns

- **TLS verification disabled** — `verify=False`, `rejectUnauthorized: false`, `InsecureSkipVerify: true` — accepts MITM-modified responses.
- **Response treated as code** — Third-party returns HTML / JS that the app renders without sanitization.
- **Response treated as a security decision** — IdP returns user info; app trusts the email field and creates / merges accounts based on it; attacker controls the IdP or compromises the connection.
- **Deserialized into typed objects** — Third-party JSON deserialized into a class hierarchy; treat as untrusted (covered in `deserialization.md`).
- **Unvalidated redirects from third party** — App receives a third-party-supplied URL and redirects to it; attacker controls third party → open redirect.
- **Webhook receivers with no signature verification** — Trusting that the webhook came from the expected sender without verifying signature.
- **Following redirects in third-party calls** — Library default may follow to internal addresses; redirect validation required.

### Defenses

- TLS verification always on.
- Treat responses as untrusted: validate types, ranges, content.
- For identity / IdP responses: only trust signed claims; verify signature.
- For webhooks: verify signatures with documented algorithm; constant-time comparison.
- Set strict timeouts; circuit breaker on flaky third parties.
- Log third-party request/response (with redaction) for incident response.

## GraphQL-Specific Concerns

### Introspection

- Disable in production for non-public APIs.
- Public APIs: introspection acceptable; combined with persisted queries reduces query-crafting surface.

### Batch Queries

- `POST /graphql` with array of queries; verify authz on each.

### Field-Level Authorization

- Per-resolver checks; not just at the top of the query.
- `@auth` directives; verify they're applied to all sensitive fields.

### Subscription Authorization

- WebSocket / SSE subscription connection checks identity; per-subscription authorization separate from connection.

### Aliasing for Enumeration

- `{ a: user(id:1) { email } b: user(id:2) { email } ... }` — single query enumerates many users. Combine alias count with rate limit.

## REST-Specific Concerns

### `PATCH` semantics

- JSON Patch (RFC 6902) vs. JSON Merge Patch (RFC 7396) — different attack surfaces for mass assignment.

### Method Override

- `X-HTTP-Method-Override` header changing the effective method; attackers using `POST` with override `DELETE` to bypass middleware that only filters on method. Disable header processing or verify all middleware applies post-override.

### Content Negotiation

- `Accept: application/xml` causing JSON-only logic to take a different path — XML parsing surface introduced. Verify content-type allowlist.

## Recommendation Patterns

- API gateway with authoritative endpoint inventory; strict deprecation/decommission lifecycle.
- Rate limiting at edge AND application; per-IP, per-user, per-key.
- Request-size and per-field limits; query complexity analysis for GraphQL.
- DTOs with explicit field allowlists; never expose ORM models.
- OpenAPI / GraphQL schema as contract; CI ensures code matches.
- Treat third-party API responses as untrusted; validate, verify signatures, never disable TLS.
- Layered abuse defenses on sensitive flows: signup, login, password reset, payment, refund.
