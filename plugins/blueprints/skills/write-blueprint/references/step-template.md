# Step Template — Markdown: 3-Phase Build-Review-Verify Cycle

Use this template when the chosen plan format is **Markdown**. For HTML plans (the default), use `step-template-html.md` instead.

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

**What not to include in Phase 1**:
- **Implementation details that go stale**: Algorithm pseudocode, variable names, internal data structure choices. These belong in the code, not in the plan.
- **Generic advice**: "Write clean code," "follow best practices," "handle errors properly." Every instruction must be specific to the step.
- **Premature optimization notes**: Unless performance is an acceptance criterion for the step, defer optimization concerns.

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

**Beyond the step — codebase integration:**
- Does the new code follow the conventions, patterns, and idioms already established in the codebase? Check naming conventions, error handling style, module structure, and import patterns against existing code.
- Does the implementation sit at the right abstraction layer? Does it respect existing boundaries (service layers, repository patterns, API contracts)? Would the change surprise a developer familiar with the codebase?
- Could this change break or degrade anything outside its immediate scope? Check imports, shared utilities, configuration, and any module that depends on modified interfaces.
- The review should aim to eliminate all issues introduced by the build phase before proceeding. Anything that passes review should be genuinely production-ready — not just "meets acceptance criteria."

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

**Human checkpoints**: If the step cannot be signed off without a person looking — a rendered UI, a migration against real data, an external integration — write that into the checklist in plain words ("have the user confirm the export opens correctly in a spreadsheet app"). Execution remediates ordinary findings on its own and pauses only where the plan says a human is needed.

---

## Example: Complete Step

```markdown
### Step 2: Add CSV Export Command

**Objective**: Add an `export` subcommand that writes query results to a CSV file. Depends on Step 1 (query engine). Uses the `QueryEngine` interface defined there.

**Acceptance criteria**:
- `app export --query "..." --output path.csv` writes results as CSV with a header row.
- Column order matches the query's field order.
- If the output file already exists, abort with an error unless `--overwrite` is passed.
- Empty result sets produce a file with only the header row.
- Non-zero exit code and a descriptive message on any failure (bad query, I/O error, permission denied).

#### Phase 1 — Build

**CLI wiring**: Register a new `export` subcommand with three arguments: `--query` (required, the query string), `--output` (required, file path), and `--overwrite` (optional flag, defaults to false). Follow the existing subcommand registration pattern established in Step 1.

**Export logic**: Create an export module responsible for executing the query via the QueryEngine, transforming results into CSV rows, and writing to the target path. Separate the CSV formatting from file I/O so each can be tested independently. Check for file existence before writing and raise a clear error if the file exists and `--overwrite` is not set.

**Error handling**: Catch query errors, I/O errors, and permission errors. Map each to a descriptive user-facing message and a non-zero exit code. Do not expose stack traces or internal paths in error output.

**Tests to write**:
- Valid query produces correct CSV with header and data rows.
- Column order matches query field order.
- Empty result set produces header-only file.
- Existing file without `--overwrite` exits with error, does not modify the file.
- Existing file with `--overwrite` replaces the file.
- Invalid query exits with non-zero code and descriptive message.
- Unwritable path (permission denied) exits with non-zero code and descriptive message.

#### Phase 2 — Adversarial Review

1. Does the export module use the QueryEngine interface from Step 1, or does it bypass it?
2. Is file existence checked before any write attempt, or could a partial write corrupt an existing file?
3. Are CSV fields properly escaped (values containing commas, quotes, newlines)?
4. Do error messages avoid exposing internal paths or implementation details?
5. Do tests verify actual file contents (read back and compare), not just exit codes?
6. Does the new command follow the same patterns as existing subcommands (argument parsing style, error reporting, exit codes)?

#### Phase 3 — Verification

- [ ] All new and modified code has corresponding tests.
- [ ] All tests pass: `[test command]`
- [ ] Test coverage is adequate — no untested branches in new code.
- [ ] Linter passes: `[lint command]`
- [ ] Type checker passes: `[type-check command]`
- [ ] Acceptance criteria check:
  - [ ] `export --query --output` produces valid CSV → verified by test_valid_export
  - [ ] Column order matches field order → verified by test_column_order
  - [ ] Empty result → header-only file → verified by test_empty_results
  - [ ] Existing file without --overwrite → error → verified by test_no_overwrite
  - [ ] --overwrite replaces file → verified by test_overwrite
  - [ ] Bad query → non-zero exit + message → verified by test_invalid_query
  - [ ] Permission denied → non-zero exit + message → verified by test_unwritable_path
```
