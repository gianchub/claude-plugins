# Plugins Marketplace Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugins marketplace containing three plugins: code-audit, blueprint, and execute.

**Architecture:** Each plugin is a self-contained directory under `plugins/` with a `.claude-plugin/plugin.json` manifest and a `skills/` directory. Skills follow progressive disclosure: lean SKILL.md (~1,500-3,000 words) with detailed content in `references/` files.

**Spec:** `docs/specs/2026-03-15-plugins-marketplace-design.md`

**Parallelization:** Chunks 1 (code-audit) and 2 (blueprint + execute) are independent of each other after Task 1 (scaffold). If subagents are available, Tasks 2-5 and 6-12 can be executed in parallel.

**Pre-existing files:** `LICENSE` already exists in the repo root and does not need to be created.

---

## Chunk 1: Marketplace Scaffold and code-audit Plugin

### Task 1: Marketplace Scaffold

**Files:**
- Modify: `README.md`
- Create: `plugins/` (directory structure only)

- [ ] **Step 1: Create the directory scaffold for both plugins**

Create the full directory tree as defined in the spec:
```
plugins/
├── code-audit/
│   ├── .claude-plugin/
│   └── skills/
│       └── audit/
│           └── references/
└── blueprint/
    ├── .claude-plugin/
    └── skills/
        ├── blueprint/
        │   └── references/
        └── execute/
            └── references/
```

- [ ] **Step 2: Update README.md**

Replace the placeholder README with a marketplace README that describes:
- What this repository is (personal Claude Code plugins marketplace)
- How to install: register as a marketplace in Claude Code
- List of available plugins with one-line descriptions
- Auto-update behavior (push to `main` = new release)

Keep it concise — under 50 lines.

- [ ] **Step 3: Commit**

```bash
git add plugins/ README.md
git commit -m "feat: scaffold marketplace directory structure"
```

---

### Task 2: code-audit — plugin.json

**Files:**
- Create: `plugins/code-audit/.claude-plugin/plugin.json`

- [ ] **Step 1: Write plugin.json**

```json
{
  "name": "code-audit",
  "description": "Language-agnostic codebase auditing with structured severity-ranked reports",
  "author": {
    "name": "fab"
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add plugins/code-audit/.claude-plugin/plugin.json
git commit -m "feat(code-audit): add plugin manifest"
```

---

### Task 3: code-audit — references/report-template.md

**Files:**
- Create: `plugins/code-audit/skills/audit/references/report-template.md`

- [ ] **Step 1: Write the report template**

This file contains the exact report structure from the spec (lines 99-124): title, summary block (scope, categories, finding counts), severity sections (Critical, High, Medium, Low), and the finding template (AUDIT-NNN, category, location, description, impact, recommendation).

Include:
- The full markdown template with placeholder values
- Instructions for filename handling: `AUDIT-REPORT-YYYY-MM-DD.md`, incrementing suffix on collision
- Zero-findings behavior: produce report with all counts at 0 and a "No issues found" note

- [ ] **Step 2: Commit**

```bash
git add plugins/code-audit/skills/audit/references/report-template.md
git commit -m "feat(code-audit): add report template reference"
```

---

### Task 4: code-audit — references/categories.md

**Files:**
- Create: `plugins/code-audit/skills/audit/references/categories.md`

- [ ] **Step 1: Write the detailed audit categories checklist**

For each of the 7 audit categories from the spec, provide a detailed checklist of specific things to look for. This is the reference the skill loads when performing an audit. Each category should include:

**Security vulnerabilities:**
- SQL/NoSQL injection, command injection, path traversal
- Authentication bypasses, broken access control, privilege escalation
- Hardcoded secrets, API keys, tokens, passwords in source
- Insecure deserialization, SSRF, XXE
- Missing input validation/sanitization at system boundaries
- Insecure cryptographic choices (weak hashing, broken algorithms)
- CORS misconfigurations, missing security headers

**Race conditions and concurrency:**
- Shared mutable state without synchronization
- TOCTOU (time-of-check-time-of-use) vulnerabilities
- Missing locks/mutexes on critical sections
- Async hazards: unhandled promises, dangling coroutines
- Database transaction isolation issues
- File system race conditions
- Signal handler safety

**Dead code:**
- Unreachable branches (always-true/false conditions)
- Unused imports, variables, functions, classes
- Commented-out code blocks
- Unreferenced configuration values
- Orphaned test files or fixtures
- Feature flags that are always on/off

**Anti-patterns and code smells:**
- God objects/classes with too many responsibilities
- Deep nesting (>3 levels)
- Magic numbers and strings
- Excessive coupling between modules
- Copy-paste duplication
- Inappropriate abstraction (premature or missing)
- Violation of single responsibility principle
- Mutable global state

**Performance:**
- N+1 query patterns
- Unnecessary allocations in hot paths
- Missing pagination on unbounded queries
- Blocking operations in async contexts
- Algorithmic complexity issues (quadratic where linear is possible)
- Missing caching for expensive repeated computations
- Unnecessary serialization/deserialization cycles

**Correctness:**
- Off-by-one errors in loops and slices
- Null/undefined/None handling gaps
- Integer overflow/underflow potential
- Floating point comparison pitfalls
- Type coercion surprises
- Boundary condition handling
- Incorrect error propagation
- Missing edge case handling (empty collections, zero values, max values)

**Error handling gaps:**
- Swallowed/silenced exceptions (bare except, empty catch)
- Missing error paths in branching logic
- Incomplete cleanup (resources not freed on error)
- Error messages leaking internal details
- Missing retry/backoff for transient failures
- Unhandled promise rejections
- Missing finally/defer cleanup blocks

- [ ] **Step 2: Commit**

```bash
git add plugins/code-audit/skills/audit/references/categories.md
git commit -m "feat(code-audit): add detailed audit categories checklist"
```

---

### Task 5: code-audit — SKILL.md

**Files:**
- Create: `plugins/code-audit/skills/audit/SKILL.md`

- [ ] **Step 1: Write SKILL.md**

Target: ~1,500-2,000 words. Imperative/infinitive form. Third-person description in frontmatter.

**Frontmatter:**
```yaml
---
name: audit
description: >
  This skill should be used when the user asks to "audit this codebase",
  "audit this code", "security audit", "code audit", "find vulnerabilities",
  "check for bugs", "review code quality", "find dead code",
  "check for anti-patterns", "performance audit", or mentions wanting a
  comprehensive code quality analysis. Produces a structured severity-ranked
  report file.
---
```

**Body structure:**

1. **Purpose** (2-3 sentences): Language-agnostic codebase auditing producing structured severity-ranked reports.

2. **Effort Level**: Maximum effort directive — read every line, trace data flows, follow call chains, consider component interactions. Use parallel subagents for large scopes.

3. **Workflow**:
   - Scope resolution: interpret natural language scope from user prompt; if none given, ask
   - Category confirmation: present the 7 categories, let user add/remove
   - Systematic analysis: describe the scanning approach — file-by-file, tracing data flows across files, deduplication of findings
   - For large codebases: split scope across parallel subagents by directory/module, merge and deduplicate findings
   - Report generation: reference `references/report-template.md` for format

4. **Severity Classification Guide**: Brief guidance on what constitutes critical vs high vs medium vs low:
   - Critical: exploitable security vulnerabilities, data loss risks
   - High: correctness bugs that will manifest in production, race conditions
   - Medium: performance issues, anti-patterns that increase maintenance cost
   - Low: dead code, minor code smells, style issues

5. **Additional Resources**:
   - `references/categories.md` — detailed checklist per audit category
   - `references/report-template.md` — report structure and filename conventions

- [ ] **Step 2: Validate SKILL.md**

Check:
- Frontmatter has `name` and `description` fields
- Description uses third person ("This skill should be used when...")
- Description includes specific trigger phrases
- Body uses imperative/infinitive form (no "you should")
- Word count is in 1,500-2,000 range
- Both reference files are mentioned
- No duplicated content between SKILL.md and references

- [ ] **Step 3: Commit**

```bash
git add plugins/code-audit/skills/audit/SKILL.md
git commit -m "feat(code-audit): add audit skill with workflow and references"
```

---

## Chunk 2: blueprint Plugin

### Task 6: blueprint — plugin.json

**Files:**
- Create: `plugins/blueprint/.claude-plugin/plugin.json`

- [ ] **Step 1: Write plugin.json**

```json
{
  "name": "blueprint",
  "description": "Collaborative implementation planning and execution with build-review-verify cycles per step",
  "author": {
    "name": "fab"
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add plugins/blueprint/.claude-plugin/plugin.json
git commit -m "feat(blueprint): add plugin manifest"
```

---

### Task 7: blueprint — references/step-template.md

**Files:**
- Create: `plugins/blueprint/skills/blueprint/references/step-template.md`

- [ ] **Step 1: Write the 3-phase cycle step template**

This file contains:

1. The full step template from the spec (lines 192-219) with detailed guidance on what each section should contain.

2. **Phase 1 — Build guidance:**
   - Describe intent, not implementation details
   - Specify components to create/modify and their responsibilities
   - Define data flow and integration points in prose
   - Specify what tests to write (types, coverage expectations, edge cases)
   - Acceptable code exceptions: interface signatures at boundaries, exact config keys, schema shapes
   - Keep it prose-first — someone reading this should understand *what* to build without seeing code

3. **Phase 2 — Adversarial Review guidance:**
   - Review checklist must be specific to the step, not generic
   - Check correctness against acceptance criteria
   - Look for edge cases the build instructions might have missed
   - Evaluate security implications of the changes
   - Assess performance impact
   - Verify error handling coverage
   - Check that tests actually cover meaningful scenarios (not just happy path)

4. **Phase 3 — Verification guidance:**
   - Must include the full verification checklist from the spec
   - Tool chain commands are filled in from the discovered tools (not hardcoded)
   - Acceptance criteria from the step are embedded as checklist items
   - Testing requirements: new/modified code has tests, all tests pass, coverage is adequate

5. **Examples:** One complete example step showing all three phases filled in for a realistic task (e.g., "Add user authentication endpoint").

- [ ] **Step 2: Commit**

```bash
git add plugins/blueprint/skills/blueprint/references/step-template.md
git commit -m "feat(blueprint): add 3-phase step template reference"
```

---

### Task 8: blueprint — references/tool-discovery.md

**Files:**
- Create: `plugins/blueprint/skills/blueprint/references/tool-discovery.md`

- [ ] **Step 1: Write the tool discovery reference**

This file documents how to detect project tooling across languages and frameworks. Structure by detection method:

**Config file detection:**
- `pyproject.toml` → check `[tool.ruff]`, `[tool.mypy]`, `[tool.pytest]`, `[build-system]`
- `package.json` → check `scripts`, `devDependencies` for eslint, prettier, jest, vitest, tsc
- `Cargo.toml` → Rust toolchain (clippy, rustfmt, cargo test)
- `Makefile` / `justfile` → parse targets for lint, test, format, build commands
- `go.mod` → Go toolchain (go vet, golangci-lint, go test)
- `.rubocop.yml`, `Gemfile` → Ruby toolchain
- `pom.xml`, `build.gradle` → Java/Kotlin toolchain

**Lock file signals:**
- `uv.lock` → Python with uv package manager
- `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` → Node.js
- `Cargo.lock` → Rust
- `go.sum` → Go
- `Gemfile.lock` → Ruby

**CI configuration:**
- `.github/workflows/*.yml` → parse for test/lint/build commands
- `.gitlab-ci.yml` → same
- `.circleci/config.yml` → same
- CI configs often reveal the canonical tool invocations

**Script conventions:**
- npm scripts (test, lint, format, build, typecheck)
- Makefile targets
- `scripts/` directory

**Presentation format:** After discovery, present to user as an ordered list:
```
Discovered tools for this project:
1. Linter: ruff (from pyproject.toml)
2. Type checker: mypy (from pyproject.toml)
3. Test runner: pytest (from pyproject.toml)
4. Formatter: ruff format (from pyproject.toml)

Add, remove, or reorder? (or confirm to proceed)
```

- [ ] **Step 2: Commit**

```bash
git add plugins/blueprint/skills/blueprint/references/tool-discovery.md
git commit -m "feat(blueprint): add tool discovery reference"
```

---

### Task 9: blueprint — SKILL.md

**Files:**
- Create: `plugins/blueprint/skills/blueprint/SKILL.md`

- [ ] **Step 1: Write SKILL.md**

Target: ~2,000-3,000 words. Imperative/infinitive form. Third-person description.

**Frontmatter:**
```yaml
---
name: blueprint
description: >
  This skill should be used when the user asks to "create a blueprint",
  "blueprint this feature", "plan this implementation", "make a plan",
  "create an implementation plan", "design the architecture",
  "break this down into steps", or needs a structured plan with
  build-review-verify cycles. Produces plan artifacts with a 3-phase
  cycle per step.
---
```

**Body structure:**

1. **Purpose** (2-3 sentences): Collaborative implementation planning. Produces plan artifacts where every step includes build, adversarial review, and verification phases.

2. **Effort Level**: Maximum effort — exhaustive exploration of requirements, deep codebase understanding before planning.

3. **Design Principles**: Collaborative (ask, don't assume), prose over code (with defined exceptions), adaptive complexity (heuristics for single-file vs milestones).

4. **Workflow** (the core section):
   - **Understand the task**: Read codebase for context. Ask clarifying questions iteratively (2-3 related questions per turn is fine). Do not proceed until requirements are solid.
   - **Tool discovery**: Detect project tooling per `references/tool-discovery.md`. Present to user for confirmation. User may add/remove/reorder.
   - **Assess complexity**: Apply heuristics — ≤5 steps with no natural grouping → single file; distinct phases or >8 steps → milestone folder. When in doubt, ask.
   - **Generate the plan**: Write plan artifact(s) following `references/step-template.md`. Every step has objective, acceptance criteria, and the 3-phase cycle. Tool chain from discovery is embedded in Phase 3 verification checklists.

5. **Output Formats**: Single doc path (`docs/plans/YYYY-MM-DD-<topic>-plan.md`) vs milestone folder (`docs/plans/YYYY-MM-DD-<topic>/01_name.md`, etc.)

6. **What Belongs in a Plan Step**: Brief guidance — prose intent, acceptance criteria, what to test, what to review. Not code. Not implementation details that will go stale.

7. **Additional Resources**:
   - `references/step-template.md` — 3-phase cycle template with examples
   - `references/tool-discovery.md` — how to detect project tooling

- [ ] **Step 2: Validate SKILL.md**

Same validation as Task 5 Step 2, plus:
- Verify the collaborative questioning approach is clearly described
- Verify tool discovery workflow references the correct file
- Verify complexity heuristics are included
- Word count is in 2,000-3,000 range

- [ ] **Step 3: Commit**

```bash
git add plugins/blueprint/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): add blueprint skill with collaborative planning workflow"
```

---

## Chunk 3: execute Skill (within blueprint plugin)

### Task 10: execute — references/subagent-prompts.md

**Files:**
- Create: `plugins/blueprint/skills/execute/references/subagent-prompts.md`

- [ ] **Step 1: Write the subagent prompt templates**

This file contains detailed prompt templates for each of the three subagent types. These are the actual prompts the orchestrator will use when dispatching subagents via Claude Code's Agent tool.

**Build subagent prompt template:**
- Receives: step's Phase 1 instructions (verbatim from plan)
- System context: "Implement the following step from an implementation plan. Read all relevant files to understand context before making changes. Write thorough tests for all new/modified functionality."
- Must return structured summary: list of files changed with intent per file, key decisions made, any deviations from plan instructions and why
- Constraints: follow existing project conventions, do not modify files outside the step's scope without noting it

**Review subagent prompt template:**
- Receives: build summary (structured) + step's Phase 2 instructions (verbatim from plan)
- System context: "Perform an adversarial code review. Read the actual files — do not trust the build summary as source of truth. Actively try to find flaws."
- Must return: findings list, each categorized as **blocking** or **advisory**, with file:line references and explanation
- Blocking = must fix before proceeding (correctness bugs, security issues, missing tests, acceptance criteria not met)
- Advisory = worth noting but does not block (style suggestions, minor improvements, future considerations)

**Verification subagent prompt template:**
- Receives: step's Phase 3 checklist (verbatim from plan, includes acceptance criteria) + tool chain configuration
- System context: "Run each verification check and report pass/fail. Execute the actual tools — do not estimate or guess results."
- Must return: pass/fail per checklist item with output excerpts on failure
- On any failure: include the full error output and which specific check failed

- [ ] **Step 2: Commit**

```bash
git add plugins/blueprint/skills/execute/references/subagent-prompts.md
git commit -m "feat(blueprint): add execute subagent prompt templates reference"
```

---

### Task 11: execute — SKILL.md

**Files:**
- Create: `plugins/blueprint/skills/execute/SKILL.md`

- [ ] **Step 1: Write SKILL.md**

Target: ~2,000-2,500 words. Imperative/infinitive form. Third-person description.

**Frontmatter:**
```yaml
---
name: execute
description: >
  This skill should be used when the user asks to "execute this blueprint",
  "run the plan", "execute the plan", "start building from the plan",
  "implement the blueprint", "execute 01_milestone_name.md",
  or wants to drive a blueprint plan through its build-review-verify cycles.
  Orchestrates plan execution using subagents for each phase.
---
```

**Body structure:**

1. **Purpose** (2-3 sentences): Execute blueprint plans by driving each step through build→review→verify using dedicated subagents. Main conversation stays lean; subagents do heavy work.

2. **Effort Level**: Maximum effort — every phase is thorough, no rubber stamps.

3. **Workflow** (the core section):
   - **Locate the plan**: Find plan file(s) in `docs/plans/`. If multiple exist, ask which one. For milestone folders, present list and start from first unmarked step.
   - **Git handling**: Ask user preference. Two modes:
     - User-managed: pause after each individual step for review/commit, even within a batch
     - Skill-managed: commit after each step, no Claude Code/AI attribution in messages
   - **Step execution**: Batch up to 3 steps. Strictly serial within batch: Step A (build→review→verify) passes before Step B begins. Each phase gates the next.
   - **Subagent dispatch**: Reference `references/subagent-prompts.md` for prompt templates. Dispatch via Agent tool. Build subagent has full filesystem access. Review subagent gets build summary + Phase 2 instructions. Verification subagent gets Phase 3 checklist + tool chain config.
   - **Failure handling**: Any blocking review finding or verification failure → stop, surface to user, wait for guidance. No auto-retry. Completed steps keep checkmarks. Failed step stays unmarked.
   - **Progress tracking**: Mark steps with ✅ in plan file immediately after passing all three phases. On resume, find first unmarked step.
   - **Plan modifications**: Re-read plan on resume to pick up user edits during pauses.

4. **Subagent Architecture**: Brief table (from spec) showing what each subagent receives and does. Emphasize: review subagent reads files independently; build summary is for focus, not trust.

5. **Batching Rules**: Max 3, strictly serial within batch, pause after batch.

6. **Design Principles**: User stays in control (no silent loops), context preservation (subagents do heavy lifting), resumable (checkmarks in plan files), no Claude Code attribution.

7. **Additional Resources**:
   - `references/subagent-prompts.md` — prompt templates for build, review, and verification subagents

- [ ] **Step 2: Validate SKILL.md**

Same validation as previous skills, plus:
- Verify git handling modes are clearly distinguished
- Verify batching rules match spec exactly (serial, max 3, each phase gates)
- Verify failure handling is explicit (stop, surface, no retry)
- Verify progress tracking mechanism is described
- Word count is in 2,000-2,500 range

- [ ] **Step 3: Commit**

```bash
git add plugins/blueprint/skills/execute/SKILL.md
git commit -m "feat(blueprint): add execute skill with subagent orchestration workflow"
```

---

## Chunk 4: Final Validation and Marketplace Registration

### Task 12: Cross-Plugin Validation

- [ ] **Step 1: Validate all plugin structures**

For each plugin (code-audit, blueprint), verify:
- `.claude-plugin/plugin.json` exists and has valid JSON with `name`, `description`, `author`
- Each `skills/<name>/SKILL.md` exists with valid YAML frontmatter (`name`, `description`)
- All files referenced in SKILL.md actually exist
- No duplicated content between SKILL.md and reference files
- All SKILL.md files use imperative/infinitive form
- All frontmatter descriptions use third person with specific trigger phrases

- [ ] **Step 2: Verify 3-phase cycle consistency**

Check that the 3-phase cycle terminology is consistent across all skills:
- blueprint defines it (Phase 1 — Build, Phase 2 — Adversarial Review, Phase 3 — Verification)
- execute references it with matching names
- code-audit does not use it (different workflow) — verify no accidental cross-contamination

- [ ] **Step 3: Commit any fixes**

If validation found issues, fix them and commit with specific file paths:
```bash
git add plugins/code-audit/skills/audit/SKILL.md plugins/blueprint/skills/blueprint/SKILL.md plugins/blueprint/skills/execute/SKILL.md
git commit -m "fix: address cross-plugin validation issues"
```

Skip this step if no issues were found.

---

### Task 13: Marketplace Registration Test

This is a manual/exploratory task. Claude Code's CLI interface for plugin testing may vary — adapt commands as needed.

- [ ] **Step 1: Test plugin discovery locally**

Test each plugin loads correctly by running Claude Code with `--plugin-dir`:
```bash
claude --plugin-dir plugins/code-audit
```

Once Claude starts, ask "what skills do you have?" or similar. Verify the `audit` skill appears. Repeat for `blueprint` and `execute`. If `--plugin-dir` is not available in your Claude Code version, skip to Step 2.

- [ ] **Step 2: Register as marketplace (manual)**

This step requires pushing the repo to a GitHub remote first. Once pushed:
1. Open `~/.claude/plugins/known_marketplaces.json`
2. Add an entry pointing to the GitHub repo (follow the same format as the existing `claude-plugins-official` entry)
3. Restart Claude Code and verify plugins are discoverable via the plugin install flow

If the repo is not yet on GitHub, defer this step — it can be done after the initial push.
