# Step Template: 3-Phase Build-Review-Verify Cycle

Every step in a blueprint plan follows this exact structure. Each step is self-contained: it builds one increment, reviews it adversarially, and verifies it before moving on.

## Template

```markdown
### Step N: [Step Title]

**Objective**: What this step achieves and why it matters in the broader plan.

**Acceptance criteria**:
- [Criterion 1 — concrete, testable condition]
- [Criterion 2 — concrete, testable condition]
- [Criterion N — as many as needed, no more]

#### Phase 1 — Build
[Prose instructions for what to construct...]

#### Phase 2 — Adversarial Review
[Step-specific review checklist...]

#### Phase 3 — Verification
[Verification checklist with tool commands...]
```

---

## Phase 1 — Build: Guidance

Describe **intent**, not implementation. The builder (human or agent) should understand *what* to create and *why*, then choose the best implementation approach in context.

**Prose-first approach**:
- Describe the component's responsibility and behavior in plain language.
- Specify which files or modules to create or modify.
- Define data flow in prose: "The request handler receives X, validates it against Y, transforms it into Z, and persists via W."
- State what tests to write: "Unit tests covering valid input, missing required fields, and duplicate entries."
- Do NOT include code blocks, function bodies, or algorithmic pseudocode.

**Acceptable code exceptions** (use sparingly):
- **Interface signatures**: When an exact function or method signature is critical for cross-step compatibility, include it. Example: `def authenticate(token: str) -> User | None`.
- **Exact config keys**: When a configuration shape must match a specific external contract. Example: `[tool.ruff.lint] select = ["E", "F", "W"]`.
- **Schema shapes**: When a data schema is the primary deliverable of the step. Example: a database migration column definition or an API response shape.

Everything else stays in prose. If the instruction feels like it needs code to be clear, that is a signal the step is too large — split it.

**What to specify**:
- Components to create or modify (file paths, module names).
- Relationships between components ("the handler calls the service, which calls the repository").
- Constraints and invariants ("all timestamps must be UTC", "maximum 100 items per page").
- Error scenarios to handle ("return 409 on duplicate email", "retry transient failures up to 3 times").
- What tests to write and what they cover.

---

## Phase 2 — Adversarial Review: Guidance

This is NOT a generic code review. Build a **step-specific checklist** that targets the most likely failure modes for what was just built.

**Constructing the checklist**:
- Start from the acceptance criteria — check each one is actually met, not just partially addressed.
- Identify edge cases specific to this step's domain (not generic "check for null" advice).
- Evaluate security impact if the step touches authentication, authorization, input parsing, or data exposure.
- Evaluate performance impact if the step introduces queries, loops over collections, or external calls.
- Verify error handling: does every failure mode produce a clear, actionable error? Are errors logged appropriately?
- Check test quality: do tests cover the acceptance criteria? Do they test failure paths, not just happy paths? Are assertions specific (not just "no error")?

**Anti-patterns to flag**:
- Tests that only verify the happy path.
- Error messages that leak internal details.
- Missing validation on external input.
- Hard-coded values that should be configurable.
- Tight coupling that will block future steps.

**Format**: Write the review as a numbered checklist of specific questions. Each question should be answerable with "yes" or "no/needs fix."

---

## Phase 3 — Verification: Guidance

Run the full verification checklist. Every item must pass before the step is considered complete.

**Standard checklist**:
1. All new and modified code has corresponding tests.
2. All tests pass: `[test command from tool discovery]`.
3. Test coverage is adequate for the step's scope (no untested branches in new code).
4. Linter passes: `[lint command from tool discovery]`.
5. Type checker passes (if applicable): `[type-check command from tool discovery]`.
6. All acceptance criteria from this step are met (re-check each one explicitly).
7. Step-specific verification items (filled per step).

**Tool chain commands**: Populate the bracketed commands from the project's discovered tool chain. If no tool exists for a check, note it as "manual review required" rather than skipping it.

**Failure protocol**: If any check fails, fix the issue and re-run the full checklist — do not selectively re-run only the failing check, as fixes can introduce regressions.

---

## Example: Complete Step

```markdown
### Step 3: Add User Authentication Endpoint

**Objective**: Provide a POST /auth/login endpoint that accepts email and password, validates credentials against the user store, and returns a signed JWT on success. This unblocks all subsequent steps that require authenticated requests.

**Acceptance criteria**:
- POST /auth/login accepts JSON body with `email` and `password` fields.
- Valid credentials return 200 with a JSON body containing `access_token` (JWT, 15-minute expiry) and `token_type: "bearer"`.
- Invalid credentials return 401 with a generic error message that does not reveal whether the email exists.
- Missing or malformed request body returns 422 with field-level validation errors.
- The endpoint is rate-limited to 10 attempts per IP per minute.
- All passwords are compared using constant-time comparison.

#### Phase 1 — Build

Create the authentication endpoint in the API layer.

**Router and handler**: Add a new route module for authentication. The login handler receives the request body, delegates credential validation to the auth service, and returns the appropriate response. Keep the handler thin — it should only parse input, call the service, and format output.

**Auth service**: Create a service that encapsulates authentication logic. It looks up the user by email via the user repository, compares the submitted password against the stored hash using constant-time comparison, and generates a JWT on success. If the user is not found or the password is wrong, raise the same error — do not distinguish between "user not found" and "wrong password" to avoid user enumeration.

**JWT generation**: Use the project's existing JWT configuration (secret key, algorithm from settings). Set the token expiry to 15 minutes. Include the user ID and email in the token payload.

**Rate limiting**: Apply rate limiting middleware to the login route only. Configure 10 requests per IP per minute. Return 429 with a `Retry-After` header when the limit is exceeded.

**Request validation**: Define the request schema requiring `email` (valid email format) and `password` (non-empty string). Return 422 with per-field errors on validation failure.

**Tests to write**:
- Successful login returns 200 with valid JWT.
- Wrong password returns 401 with generic message.
- Non-existent email returns 401 with the same generic message as wrong password.
- Missing email field returns 422.
- Missing password field returns 422.
- Malformed email returns 422.
- Rate limit triggers after 10 rapid requests, returns 429 with Retry-After header.
- JWT contains expected claims (user ID, email, expiry).

#### Phase 2 — Adversarial Review

1. Does the 401 response for wrong password look identical to the 401 for non-existent user (same status, same body, same timing)?
2. Is password comparison using constant-time comparison (not `==`)?
3. Does the JWT expiry match the 15-minute requirement, not some default?
4. Are the JWT secret and algorithm sourced from configuration, not hard-coded?
5. Does the rate limiter key on IP address, and does it handle `X-Forwarded-For` correctly for proxied deployments?
6. Does the 429 response include a `Retry-After` header with the correct value?
7. Do validation error messages avoid leaking internal field names or schema details?
8. Are tests asserting on specific status codes and response shapes, not just "request succeeded"?
9. Is there a test that verifies the timing side-channel is mitigated (same response time for valid vs invalid email)?
10. Does the login handler avoid logging the submitted password, even at debug level?

#### Phase 3 — Verification

1. [ ] All new and modified code has corresponding tests.
2. [ ] All tests pass: `pytest tests/ -x -q`
3. [ ] Test coverage is adequate: `pytest --cov=src/auth --cov-report=term-missing`
4. [ ] Linter passes: `ruff check src/ tests/`
5. [ ] Type checker passes: `mypy src/`
6. [ ] Acceptance criteria check:
   - [ ] POST /auth/login accepts email + password → verified by test_successful_login
   - [ ] 200 + JWT on valid credentials → verified by test_successful_login, test_jwt_claims
   - [ ] 401 on invalid credentials (generic message) → verified by test_wrong_password, test_nonexistent_email
   - [ ] 422 on bad input → verified by test_missing_email, test_missing_password, test_malformed_email
   - [ ] Rate limited at 10/min/IP → verified by test_rate_limit
   - [ ] Constant-time comparison → verified by code review in Phase 2 item 2
7. [ ] Rate limiter configuration is documented in the project's settings reference.
```
