# Threat Modeling

## Purpose

Produce the Threat Model Brief that drives Phases 3–6 of the audit. Without this brief, the audit becomes a generic checklist scan; with it, the audit prioritizes the threats that actually apply to this application and scores findings against realistic exposure.

The brief is short (typically 200–400 words). It is regenerated on every audit run and never persisted across runs.

## Inference Heuristics

Detect the following from code, configuration, and documentation. Prefer evidence from code over assumptions from naming. Where multiple signals conflict, document the ambiguity in the brief.

### Application Kind

| Kind | Signals |
|------|---------|
| Web API (REST/GraphQL) | Express/Flask/FastAPI/Spring/Gin/Rails routes returning JSON; OpenAPI spec; GraphQL schema; absence of view templates |
| Server-rendered web | Templates (Jinja2, ERB, Twig, Thymeleaf, Handlebars), session middleware, form handlers, CSRF middleware |
| Single-page app + API | `index.html` shell, frontend framework (React/Vue/Angular/Svelte) plus a separate API surface |
| CLI | `argparse`/`click`/`cobra`/`commander`; entry point with `main`; installed via console_scripts/bin |
| Library | Public API surface, no entry-point binary, documented for embedding |
| Mobile backend | API specifically used by an iOS/Android client (look for client repos, mobile-specific endpoints, push notification code) |
| Desktop app | Electron, Tauri, Qt, GTK, native shell entry points |
| Embedded/IoT | C/C++ on cross-compiled toolchain, RTOS, hardware-specific code |
| Browser extension | `manifest.json` with `manifest_version`, content/background scripts |

If the codebase contains multiple kinds (a monorepo with both an SSR frontend and an API), produce one brief per service or one brief covering both with each kind's threats noted.

### Exposure

| Exposure | Signals |
|----------|---------|
| Internet-facing, anonymous | Public routes without auth middleware, public-facing infra (CloudFront/Cloudflare/Nginx as edge), marketing pages, login/signup endpoints |
| Internet-facing, authenticated | All routes behind auth middleware, public API requiring API keys |
| Internal-only | Bound to private network in deployment manifests, behind VPN/Bastion; routes referenced in internal docs only |
| Localhost / single-user | CLI tools, desktop apps, dev-only servers |
| Single-tenant | One organization per deployment; no tenant-scoping logic |
| Multi-tenant | `tenant_id`/`org_id`/`workspace_id` columns on most tables, tenancy middleware |

For multi-tenant, exposure also includes whether tenants are mutually trusted (e.g., enterprise SSO with one paying customer per tenant) or mutually hostile (e.g., consumer SaaS with anonymous signups). Hostile multi-tenancy is the highest-risk exposure for authorization issues.

### Sensitive Data Classes

Look at database schemas, ORM models, log lines, and request/response shapes to classify what the application handles. Common classes:

- Authentication credentials (passwords, password hashes, MFA secrets, OAuth tokens, API keys, session tokens, JWTs).
- Personally identifiable information (names, emails, addresses, government IDs, phone numbers).
- Payment data (PAN, CVV, bank account numbers — typically only present if PCI-scope).
- Health records (diagnoses, prescriptions, insurance — typically HIPAA-scope).
- Financial records (transactions, balances, statements).
- Source code or trade secrets (the application stores customer code, designs, or other IP).
- User-uploaded files of arbitrary content type.
- Internal infrastructure secrets (cloud credentials, service-to-service tokens).

The classes present determine which threats matter most. Health/payment data raise the floor on every authorization finding; user-uploaded files raise the floor on file-handling findings.

### Trust Boundaries

Identify where untrusted data enters and where privilege transitions occur:

- **External-to-internal**: HTTP endpoints, message queue consumers, file ingestion, third-party API responses (treat as untrusted unless the third party is contractually trusted), webhook receivers.
- **Anonymous-to-authenticated**: Login, signup, password reset, magic link, social auth callback.
- **User-to-admin**: Role transitions, impersonation features, admin-mode flags.
- **Tenant-to-tenant**: Anywhere a request can read or write data owned by a different tenant.
- **Process-to-process**: IPC, named pipes, sockets, shared memory.
- **Service-to-service**: Internal RPC, message buses; treat as untrusted unless mTLS and peer identity are verified.

### Authentication Model

Identify the primary mechanism. Some apps have multiple (browser session for the SSR app, JWT for the API). Document each.

- Form login with server session (cookie session ID, server-side session store).
- JWT bearer (stateless, signed; check alg, exp, kid handling).
- OAuth/OIDC client (the application acts as a client to an IdP; check state, PKCE, redirect URI).
- OAuth/OIDC provider (the application issues tokens; full provider security applies).
- API keys (header or query; check rotation and scoping).
- mTLS (peer cert verification).
- Signed URLs / pre-signed URLs (HMAC; check expiration and scope).
- None (unauthenticated public service).

### Authorization Model

- RBAC (roles like admin/member/viewer assigned to users).
- ABAC (attribute-based — policy engine, OPA, custom rules).
- Ownership-based (user owns row → user can access row).
- Tenancy-scoped (every query is filtered by tenant).
- None (no authorization layer; rare and dangerous).

Note where authorization is enforced: middleware, decorator, framework filter, ad-hoc per route, database policy (RLS), or external service.

### External Dependencies and Integrations

List databases, caches, queues, third-party APIs, payment processors, identity providers. For each, note the trust assumption — is the response treated as trusted? Is the connection authenticated? Is data signed or otherwise tamper-resistant?

### Deployment Context

Containers (Docker, K8s), serverless (Lambda, Cloud Run, Cloud Functions), VMs, on-prem; cloud provider (AWS/GCP/Azure/none); CI/CD platform (GitHub Actions, GitLab CI, Jenkins, CircleCI). Drives the IaC and CI/CD checklists. Detect from `Dockerfile`, `docker-compose.yml`, `*.tf`, `serverless.yml`, K8s manifests, `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`, `.circleci/config.yml`.

## Threat Model Brief Template

```markdown
## Threat Model Brief

### Application Kind
<one or two sentences; cite key signals>

### Exposure
<internet-facing | authenticated public | internal-only | localhost | single-tenant | multi-tenant; with tenancy trust assumption if applicable>

### Sensitive Data Classes
<bullet list; one line each>

### Trust Boundaries
<bullet list; format: "<source> -> <destination> (<trust transition>)">

### Authentication Model
<primary mechanism(s); cite the file/line where it is enforced>

### Authorization Model
<RBAC / ABAC / ownership / tenancy / none; cite enforcement point>

### External Dependencies & Integrations
<bullet list; one line each, with trust assumption>

### Deployment Context
<containers / serverless / VMs / on-prem; cloud provider; CI/CD platform>

### Applicable Domain Checklists
<list of checklists from references/checklists/ that apply to this application>

### Severity-Modifier Notes
<any application-specific factors that should raise or lower default severities — e.g., "PHI handling raises authorization floor to High", "internal-only deployment lowers exposure on most findings to Medium">
```

## Worked Examples

### Example 1: Public consumer SaaS web API

```
Application Kind: REST API serving a single-page app frontend. Express + Postgres + Redis.
Exposure: Internet-facing; mutually-hostile multi-tenant (anonymous signup).
Sensitive Data Classes: Email, hashed password, session tokens, customer-uploaded files (any content type), Stripe customer IDs (no card data — Stripe Elements).
Trust Boundaries:
- HTTP body/query/headers/cookies -> Express handlers (anonymous untrusted)
- HTTP -> auth middleware -> req.user (anonymous-to-authenticated transition at src/middleware/auth.ts:42)
- req.user.tenantId -> queries (tenant-to-tenant boundary on every read/write)
- Stripe webhook -> /webhooks/stripe (external-to-internal; signature verification required)
Authentication Model: Form login with server session (express-session + Redis). JWT for some service-to-service calls.
Authorization Model: Ownership-based with tenancy scoping; enforced ad-hoc in handlers (no central middleware) — flagged as concern.
External Dependencies: Postgres, Redis, Stripe (webhook signed), SendGrid (outbound only).
Deployment Context: Docker on AWS ECS Fargate; GitHub Actions CI/CD.
Applicable Checklists: auth-and-session, authorization, injection, xss-csrf-frontend, ssrf-redirect-url, crypto, deserialization, file-handling, secrets-and-keys, error-and-logging, business-logic, api-security, dependencies, containers-iac, cicd.
Severity-Modifier Notes:
- Hostile multi-tenancy raises authorization floor to High for every IDOR/BOLA finding.
- User-uploaded arbitrary files raise file-handling findings to High by default.
```

### Example 2: Internal CLI tool

```
Application Kind: Python CLI invoked by engineers locally and by CI workflows.
Exposure: Localhost / single-user (engineer's machine or CI runner). Reads files from current directory and makes outbound HTTPS calls to internal APIs.
Sensitive Data Classes: AWS credentials (read from env), internal API tokens.
Trust Boundaries:
- argv, stdin (single-user untrusted? or trusted? Engineer is operator; threat model is supply-chain compromise of dependencies).
- Internal API responses -> CLI parsing (treated as trusted; flag if responses are deserialized unsafely).
Authentication Model: AWS_PROFILE / AWS credential chain; internal tokens via env.
Authorization Model: None at the CLI layer; relies on AWS IAM and internal API authz.
External Dependencies: boto3 (AWS), requests (internal API).
Deployment Context: pip-installed; runs locally and in GitHub Actions.
Applicable Checklists: secrets-and-keys, deserialization, dependencies, cicd. Skip xss-csrf-frontend, api-security (no API surface), ssrf-redirect-url (only egress is to a fixed internal API).
Severity-Modifier Notes:
- No internet-exposed surface; lower exposure on most findings to Medium.
- Supply-chain risk on dependencies elevated because the CLI runs with engineer credentials.
```

### Example 3: Library

```
Application Kind: TypeScript library published to npm. Provides URL parsing and validation utilities.
Exposure: Embedded in many downstream applications, some internet-facing.
Sensitive Data Classes: None handled directly; the library processes URLs that may contain anything.
Trust Boundaries: Function arguments (assumed untrusted by the library's contract).
Authentication Model: N/A.
Authorization Model: N/A.
External Dependencies: None at runtime; build-time deps in package.json.
Deployment Context: Built and published from GitHub Actions to npm.
Applicable Checklists: injection (URL parsing pitfalls), ssrf-redirect-url (URL parser confusion is the primary risk class for this library), dependencies, cicd.
Severity-Modifier Notes:
- Library context elevates URL-parsing inconsistencies to High because downstream consumers may rely on parsed values for security decisions.
- npm publish supply-chain risk: CI/CD checklist applies with elevated importance.
```

## When the Brief is Ambiguous

Ask a single clarifying question only when the application kind or exposure is genuinely ambiguous from the code (e.g., a service that could be internal-only or public-facing depending on deployment). Otherwise, infer and document the inference in the brief — the report consumer can correct it.

For each ambiguity not asked about, document the assumption clearly: "Assumed internet-facing based on Cloudflare references in `infra/`. If actually internal-only, downgrade exposure-sensitive findings by one level." This makes the brief auditable and gives the report consumer a way to recalibrate.
