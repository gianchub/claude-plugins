# Authorization

## Scope

Authorization (authz) is "is this authenticated subject permitted to perform this action on this resource?" Authentication identifies the subject; authorization gates the action. This checklist covers every form of authorization weakness: object-level, function-level, field-level, tenancy, ownership, and chained role manipulation. Authorization issues are the highest-impact, highest-frequency weakness class in modern web and API applications, and the most under-tested by automated tools.

The defining technique for finding authz weaknesses is **enumeration of every protected operation and verification that every required check is present at the right boundary** — typically before any state change, on every endpoint and code path that reaches the resource.

## Checks

### Object-Level Authorization (IDOR / BOLA)

The single most common high-impact API weakness. The attacker substitutes an object identifier in the request to reference an object they should not access.

- **Identifiers from request used directly in fetch** — `Order.findById(req.params.id)` without checking ownership or tenant. Look for any pattern where a request-supplied ID flows into a query that does not filter by the requester's identity.
- **Multiple endpoints sharing a buggy pattern** — When the team has 50 endpoints reading by ID, the same omission pattern often recurs across many of them. Audit a sample of routes; if any are missing the check, audit *all* similar routes.
- **GraphQL field resolvers** — Each field resolver may need its own authz check. Authz on the parent query does not protect child fields if a child resolver makes its own unscoped query.
- **Batch endpoints** — A `GET /orders?ids=1,2,3` endpoint may apply `req.user.tenantId` filter on the list query but not when individual IDs are explicitly named. Verify that batch fetches enforce the same scoping.
- **Numeric vs UUID identifiers** — UUIDs reduce enumeration but do *not* fix authorization. A leaked UUID is still cross-tenant readable if the check is missing. The fix is the check, not the ID format.
- **Predictable identifiers** — Sequential IDs, `email`-based IDs, or hash-of-user-data IDs allow trivial enumeration when authz is also missing. Both are findings: missing authz (the real one) and predictable IDs (defense-in-depth concern).
- **Soft delete and "deleted" objects** — Authz checks that look at active rows but allow access to soft-deleted ones via a different code path.
- **Indirect access** — Object A references Object B; user can access A; therefore code lets the user access B without rechecking. Each access point must verify.

### Function-Level Authorization (BFLA)

The attacker invokes a function (endpoint, RPC method, GraphQL mutation) they should not be permitted to call at all.

- **Admin endpoints reachable via guessing** — `/api/users/:id` allows any user to read; `/api/admin/users/:id/delete` should reject non-admin users. Verify check on each.
- **Sibling endpoints with inconsistent checks** — `POST /api/users/:id/ban` checks admin role; `POST /api/users/:id/disable` does not. Pattern often reflects copy-paste with one-off omission.
- **Hidden HTTP methods** — Endpoint accepts `PUT` / `DELETE` / `PATCH` not documented; framework default routing exposes them. Verify check on every method.
- **Internal endpoints exposed externally** — `/internal/...` reachable without VPN restriction; or "private" RPC methods exposed via public gRPC reflection.
- **GraphQL mutation root** — Any mutation reachable from any authenticated user unless the resolver checks role/permission.
- **Job/task triggers** — Endpoints that enqueue background work; common to omit authz because "the worker is internal" — but the trigger is not.
- **Webhook handlers acting as authz bypass** — Webhook handlers that bypass auth middleware and trigger sensitive operations.

### Property-Level Authorization (Mass Assignment / Excessive Data Exposure)

Two sides of the same issue: the client should not be able to write certain fields, and should not be able to read certain fields.

**Mass Assignment / Overposting:**

- **Whole-object update** — `User.update(req.user.id, req.body)` where `req.body` includes `role`, `is_admin`, `tenant_id`, `verified_email`, `created_at`. Allowlist writable fields explicitly.
- **Framework convenience binders** — `User.create(req.body)` (Sequelize), `User(**request.json)` (Pydantic without `Field(exclude=...)`), Spring `@RequestBody` to entity, Rails `permit(...)`-less `params`, Django `ModelForm` with no `fields=` constraint.
- **Nested objects** — Updating `req.body.user.address` may bypass top-level allowlists if the validator only checks top-level keys.
- **JSON Patch / JSON Merge Patch** — Clients sending RFC 6902 / 7396 patches; verify each patch op against an allowlist of paths.
- **GraphQL input types** — Schema may include a field the user should not set; the schema is the contract. Audit input types for "internal-only" fields.

**Excessive Data Exposure:**

- **Returning full DB row** — `res.json(user)` sends `password_hash`, `mfa_secret`, `internal_notes`. Use serializers with explicit field allowlists.
- **GraphQL `__typename` / introspection** — Production introspection enabled exposes schema; not directly an authz failure but enables enumeration.
- **Verbose error fields** — Error responses including the row that triggered them.
- **JSON dumps of internal models** — Pydantic / Marshmallow / DRF serializers with `__all__` fields.
- **API responses not segregated from internal models** — Domain model and API DTO sharing the same shape.

### Tenancy / Multi-Tenant Boundaries

For any multi-tenant application, every query that reads or writes tenant-scoped data must include `tenant_id = req.user.tenant_id` as a filter — *as a query predicate, not as a post-fetch check.*

- **Post-fetch tenant check** — `obj = Foo.find(id); if obj.tenant_id != user.tenant_id: 404` — TOCTOU-vulnerable in some patterns and trivial to forget; misses count queries entirely. Better: filter at query time so the row is never returned.
- **Tenant ID in client body / query** — `?tenant_id=X` on the request; trusting it without verifying it matches `req.user.tenant_id` is direct cross-tenant read.
- **Aggregations and counts** — `COUNT(*) FROM logs` for a global metric on a per-tenant page leaks cross-tenant info. Audit aggregation endpoints.
- **Background jobs and async work** — Job handler receives `tenant_id` as part of payload; if the payload originates from user-supplied data, treat as untrusted.
- **Cache keys without tenant** — Cache key built from user-controlled ID without tenant prefix → cross-tenant cache hit serves wrong data.
- **Database-level RLS** — When Row-Level Security policies exist (Postgres RLS, etc.), verify that application code sets the session variable correctly and consistently. Bypassed if `SET LOCAL app.user_id = ...` is omitted on a connection.

### Vertical Privilege Escalation

User gains capabilities of a higher role.

- **Role assignable via mass assignment** — Already covered above; common path to admin.
- **Role check on string equality with user-controlled comparison value** — `if request.role == "admin"` where `request.role` is from the client.
- **Default role on signup** — New users receive elevated role due to misconfiguration (env var defaulted wrong, factory default).
- **Role inheritance** — Hierarchical roles where a partial implementation grants more than intended ("manager" role granted "admin" implicitly via missing role check).
- **Token scoping** — OAuth tokens with broader scopes than the calling client should have; refresh-token reuse expanding scope.
- **Impersonation features** — "Login as user" features that don't restrict to admin caller, or that issue full session tokens for the impersonated user.

### Horizontal Privilege Escalation

User accesses peer-user data they should not (subset of IDOR; called out separately because it's the most common manifestation).

- **Endpoints expecting `:userId`** — `/api/users/:userId/profile` where check is "is the requester authenticated" but not "is the requester `:userId`".
- **Side-channel ID derivation** — Client sends `email`; server derives user; server should still verify the derived user is the requester.

### Authorization Logic Bugs

- **Negative / inverted checks** — `if not user.is_admin: return ...` followed by code that should *only* run for admins.
- **Permissive default** — `allowed = True; if X: allowed = False; if Y: allowed = False; return allowed` — easy to miss a case.
- **OR where AND is needed** — `if user.is_owner OR user.is_admin` may be intended; `if user.is_owner OR user.is_member` may grant too much.
- **String role comparison case sensitivity** — Database stores `Admin`, code checks `admin` — depending on direction of bug, fails open or fails closed.
- **Missing `else` clauses** — `if forbidden: ...` without `return`/`raise` — execution continues after the "deny" branch.

### Authorization at the Wrong Boundary

- **Check after state change** — `transaction(); modifyData(); if not allowed: rollback()` — race-vulnerable; any side effects (logs, downstream calls) are not rolled back.
- **Check on display, not on action** — Frontend hides buttons but backend doesn't enforce. Always assume the client is malicious.
- **Check in middleware that doesn't apply to the route** — Middleware mounted on a path prefix that doesn't include the protected route. Verify route-to-middleware mapping per endpoint.
- **Check on parent resource only** — `parent.can_user_access()` true → child operations succeed without rechecking child-specific permissions.

### Authorization in Distributed Systems

- **Service-to-service trust** — Internal RPC trusts caller without re-verifying user identity. If service A is compromised, anything it can call falls.
- **Token forwarding without re-verification** — Service A receives JWT, forwards to service B; B trusts A's processing without verifying signature itself.
- **Eventual consistency boundaries** — User loses permission; cached permissions still allow operations until cache expiry.

### Database-Level Authorization

- **RLS bypassed by privileged connections** — Application connection runs as superuser bypassing RLS. RLS only protects when the connection is restricted.
- **Direct DB writes from migrations / admin tools** — Migrations run as DB superuser; if migrations include data manipulation derived from user input (rare but happens), authz is non-existent.

## Framework Notes

- **Express**: authz is typically per-route middleware or in the handler. Search for routes lacking obvious authz; verify shared middleware list.
- **Spring Security**: `@PreAuthorize`, `@Secured` annotations. Search for endpoints without annotations; verify expression-based annotations evaluate correctly.
- **Django REST Framework**: `permission_classes` per ViewSet. Default `AllowAny` is dangerous; verify each view sets explicit permissions.
- **Rails**: Pundit / CanCanCan policies. Search for actions without `authorize` calls.
- **GraphQL**: each resolver may need its own check; framework-level guards rarely cover all fields.
- **Hasura / PostgREST**: row-level rules in the database; misconfiguration means full DB read.

## Bypass Patterns

- **Path normalization mismatch** — Authz applied to `/admin/...` but `/admin/../admin/...` or `/admin//foo` slips past depending on framework.
- **HTTP method tampering** — `GET` is checked; `HEAD`/`OPTIONS`/`POST` to the same path may not be.
- **Trailing slash inconsistency** — Some frameworks treat `/foo` and `/foo/` as different routes; authz may apply to one and not the other.
- **Case-sensitive routing on case-insensitive filesystems** — Rare but happens; route `/Admin` may bypass `/admin` middleware.
- **Race-on-permission-revocation** — Window between admin removing a user's permission and the user's existing session/cache reflecting the change.

## Recommendation Patterns

- Centralize authz logic. Per-route ad-hoc checks are easy to forget.
- Default deny: every new endpoint requires explicit authz declaration; audits flag the absence.
- Filter at query time for tenancy and ownership; never rely solely on post-fetch checks.
- Allowlist writable and readable fields explicitly; do not mix domain models and API contracts.
- Use serializers / DTOs as the boundary; do not return ORM model instances directly.
- Test authz negatively: tests must verify that the wrong user gets denied, not just that the right user gets allowed.
