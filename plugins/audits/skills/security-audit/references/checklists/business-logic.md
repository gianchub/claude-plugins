# Business Logic

## Scope

Vulnerabilities that exploit *intended* application behavior in unintended ways: workflow bypass, race conditions on financial/state operations, abuse of legitimate features at scale, state-machine flaws, and asymmetric outcomes. These are the hardest findings to detect because they don't manifest as broken code patterns; they manifest as policy that doesn't match reality.

The defining technique: for every multi-step or state-bearing operation, ask "what assumption is the developer making about how this is reached / repeated / interleaved?" and challenge each assumption. Skim every checkout, every quota, every "one-time" flow, every coupon/promo code, every account-creation rate, every workflow with an assumed order.

## Workflow Bypass

The application enforces a sequence of steps; the attacker skips, repeats, or reorders steps.

### Patterns

- **Multi-step forms / wizards** — Step 1 asks for email, step 2 asks for password, step 3 issues account. If step 3 is an endpoint reachable directly with a forged session-state cookie or by setting the right URL parameter, the attacker creates accounts with arbitrary email-skipping verification.
- **Verification-then-action** — A link emailed for "click to verify, then the account is active." If the action endpoint doesn't check `verified=true`, the verification step is decorative.
- **Payment-then-fulfillment** — "Pay, then we ship." If the fulfillment endpoint doesn't check payment status, free shipping. Webhook-driven fulfillment that *trusts* the payment provider's call but doesn't verify the signature is also exploitable.
- **Approval-then-execution** — Sensitive admin action requires another admin's approval; if the execute endpoint doesn't check `approved_by`, attacker bypasses the second-eyes control.
- **Phased rollout flags** — Feature gated behind a flag; flag check happens at one entry point but an alternative entry point omits the check.

### Defense

- Encode step state server-side (not in client cookies / localStorage / hidden form fields).
- Each action endpoint independently verifies the prerequisite state in the database; do not trust client-supplied "I'm at step 3" claims.
- State-machine model: a transition table that explicitly enumerates allowed transitions; rejection on any other.

## Race Conditions on Business State

Concurrent execution of operations that should be serialized produces double-spend / over-provision / quota-bypass outcomes.

### Patterns

- **Coupon / promo code redemption** — `SELECT used_at; if NULL, UPDATE used_at = NOW(); apply discount`. Without locking, N concurrent requests pass the check and apply N discounts.
- **One-time discount, gift card, voucher** — Same as above.
- **"Free trial" abuse** — Account creation with bypass of "previous trial" check.
- **Withdrawal / transfer** — Balance check then debit; concurrent requests withdraw more than the balance.
- **Inventory / "reserve seat"** — Concurrent purchases of the last item.
- **Rate-limit checks** — Concurrent requests pass the check before any increment commits.
- **Account creation with rate limit per email** — Race lets attacker create N accounts.
- **Two-factor enrollment race** — Race between MFA being enabled in the database and the next login bypassing MFA.

### Defenses

- **Database-level atomicity** — `UPDATE coupons SET used_at = NOW() WHERE id = ? AND used_at IS NULL` returning rowcount; only the operation that affects 1 row succeeds. Generic pattern: condition in the UPDATE WHERE clause.
- **Pessimistic locking** — `SELECT ... FOR UPDATE` within a transaction; releases on commit/rollback.
- **Optimistic locking** — Version column / row-version; UPDATE checks version equality; conflict on retry.
- **Distributed locks** — Redis-based locks (Redlock has known issues; use with care). Application-level logical locks.
- **Idempotency keys** — Client-supplied unique key per logical operation; server stores key + result; duplicates return same result. Critical for financial APIs (Stripe, Square pattern).
- **Avoid TOCTOU** — Move the check into the UPDATE; eliminate the gap.

### Where to Look

- Any operation where a single record's flag/counter is read-then-conditionally-written.
- Any operation that decrements a balance, count, or quota.
- Any operation that "reserves" or "claims" something.
- Authentication primitives: MFA enrollment, session creation, password change.

## Abuse of Legitimate Features

Features designed for legitimate use, used at scale or in unexpected combinations.

### Examples

- **Mass account creation** — Signup endpoint without anti-automation; attacker creates millions of accounts (for spam, scraping, free-tier abuse, fraud setup).
- **Mass invitation / email send** — User invites colleagues; without rate limit, the application becomes an abuse channel for spam originating from the application's domain (deliverability damage compounds the abuse).
- **Password reset email flood** — Reset endpoint sending email per request; without rate limit, mass spam to victim email or victim's customers.
- **Free-tier / trial abuse** — Sign up, consume free credits, abandon. Detection by linking accounts (browser fingerprint, payment method, IP); mitigation by raising friction.
- **Refund abuse** — Repeated refund requests after delivery; detect via per-user history.
- **API quota bypass** — Single user creates many accounts to multiply free-tier API quota.
- **Vote / review / star manipulation** — One-vote-per-user check easily bypassed by sock-puppet accounts.

### Defenses

- **Rate limiting** — Per IP, per user, per email, per phone. Combined; layered.
- **Anti-automation** — CAPTCHA on signup, password reset, review submission. Modern alternatives: reCAPTCHA v3 score, hCaptcha, Turnstile.
- **Email verification before privileged action** — Raises bar on disposable-email signup.
- **Phone verification for high-trust actions** — Costs money for attacker; raises bar.
- **Payment-method requirement** — Free tier still requires a card on file; payment fraud detection becomes a layer.
- **Risk scoring** — Behavioral signals (rapid signup, suspicious IP, browser fingerprint linking) feed into challenges.
- **Per-tenant quotas** — Hard limits on per-tenant API usage, email send rate, etc.

## State-Machine Flaws

The application has implicit or explicit state for entities (orders, subscriptions, tickets, etc.). Transitions should be limited to legitimate paths.

### Patterns

- **Resurrecting cancelled / deleted entities** — Endpoint accepts updates that re-activate a cancelled subscription.
- **Skipping intermediate states** — Order goes from "placed" directly to "shipped" without "paid."
- **Concurrent transitions** — Two transitions race; final state inconsistent.
- **Status field updatable by client** — Mass-assignment-style; client provides `status: "approved"` and bypasses workflow.

### Defenses

- Explicit state machine in code; transitions through one function that validates the from-state.
- Status fields not accepted from request bodies; computed server-side.
- Idempotent transitions where appropriate (calling "approve" on an already-approved entity is no-op).

## Asymmetric Outcomes

Operations whose cost differs between attacker and victim, allowing low-cost attacks to cause high-cost effects.

### Examples

- **Email verification flood** — Attacker sends 1 request → server sends 1 email (with delivery cost, deliverability impact, recipient annoyance). Multiplied: spam attack, possibly with attacker burning the victim's domain reputation.
- **Account lockout via reset attempts** — Attacker triggers lockout on victim's account by repeated wrong attempts, denying victim access.
- **Server-side rendering / preview generation** — Attacker submits content, server renders heavy preview (PDF generation, video transcoding, image processing). 1 request → seconds of CPU.
- **Compute-heavy queries / GraphQL deep nesting** — Attacker constructs query that causes joins/recursion / N+1 fanout. 1 request → exponential work.
- **Stamp-collecting / amplification** — Attacker requests one resource that triggers many internal operations.

### Defenses

- Quota / rate limit on amplification endpoints.
- Background processing with bounded queues; reject when queue is full.
- Compute-cost estimation pre-execution (GraphQL query complexity analysis).
- Per-recipient throttling (max 1 verification email per 5 min per recipient).

## Authentication and Authorization Flow Logic

Some of the most impactful business-logic flaws are in auth flows specifically:

- **Registration that lets an attacker pre-claim someone else's email** (claim email A; legitimate user with email A signs up; attacker has the account before legitimate user). Resolution: tie account-finalization to email verification.
- **Account merging** — User merges account A and B; attacker plants their key/password under account B before merge; result has attacker access.
- **OAuth flow with email-based account linking** — Third-party OAuth provider whose email isn't verified by the IdP allows account takeover when the local app trusts the email.
- **"Login with X" overlay** — Multiple identity providers; one provider provides email; attacker registers same email at a different provider with weaker verification; logs in as victim.
- **Password change without re-authentication** — If session can be hijacked by any means (XSS), password change locks legitimate user out.

## Pricing / Payment Logic

- **Negative / zero quantity in checkout** — Refund-by-purchase if quantity not validated.
- **Currency precision** — Truncating fractions of cents to attacker advantage.
- **Discount stacking** — Coupons designed to be exclusive but stackable due to missing checks.
- **Final-price tampering** — Frontend-supplied price accepted server-side without recomputing from authoritative price source.
- **Currency conversion timing** — Locking in a quote, then exploiting price changes.
- **Refund-then-recharge** — Pattern abuse for chargeback gaming.

## Per-User and Per-Tenant Quotas

- **Per-tenant compute** — One tenant's bursty workload starves others; enforce per-tenant resource limits.
- **Per-tenant storage** — One tenant fills storage; verify per-tenant quota.
- **Per-user API quotas** — Free tier overrun; verify hard limits and that throttling actually applies (rather than just billing).

## Recommendation Patterns

- Encode workflow state server-side; verify prerequisites at every action endpoint, not just at the entry point.
- Use database-level atomicity (UPDATE with WHERE conditions; SELECT FOR UPDATE; idempotency keys) for any single-use / single-claim / balance-affecting operation.
- Layer defenses against abuse: rate limit + CAPTCHA + verification + risk scoring; no single layer protects fully.
- Make explicit state machines; don't rely on implicit state.
- Treat all client-supplied state, status, and price fields as untrusted; recompute server-side.
- Test negative cases: workflow shortcuts, concurrent requests, replay, role manipulation. Tests should fail when these aren't blocked.
