# Changelog

All notable changes to the claude-plugins project are documented in this file.

Version numbers refer to the **blueprints** plugin version (renamed from `blueprint` in 2.0.0), which has been the primary driver of releases. The `audits` plugin version (renamed from `code-audit` in 2.0.0) is noted where it differs.

## [2.3.0] - 2026-08-25

Rewrite `execute-blueprint`'s failure policy. Findings that the run itself created are now fixed by a remediation subagent and re-verified automatically; the user is stopped only for decisions that are genuinely theirs. The `audits` plugin is unchanged and remains at 2.1.0.

### Blueprints (2.3.0)

- **Automatic remediation replaces blanket escalation.** Previously *every* blocking review finding and *every* failed verification check stopped the run and waited for the user — a policy that interrupted for a missing null check as readily as for a schema redesign. Now in-scope correctness, test, and integration findings are dispatched to a new **remediation subagent** that fixes the listed defects and nothing else, after which the review and the verification both run again in full
- **Escalation classes.** Every blocking finding carries a remediation class. `in-scope` findings are fixed automatically; four classes stop the run and hand the decision to the user:
  - `design-decision` — the plan does not settle what the correct behaviour is, and choosing is a product or design call
  - `scope-expansion` — the fix needs a new dependency, a change to a public interface, schema, or configuration contract, or edits outside the step's scope
  - `destructive-or-external` — the fix would alter existing data, drop or rewrite columns, delete files the step did not create, touch git history, or reach remote, production, or credentialed systems
  - `environment` — a check failed because a tool, service, or credential is missing or misconfigured, not because the code is wrong

  The review subagent assigns the class (its prompt template and return format now require one per blocking finding); the remediation subagent, which has read the code, has the final say and can escalate anything it was handed. If any finding in a set escalates, the run stops before remediating *anything* — a design decision can change what the right fix is, and a half-remediated tree is harder to reason about than an untouched one
- **Two budgets bound the loop.** A step gets at most **two** remediation cycles before it escalates. A subsystem — keyed on the nearest common parent directory of the files cited in the findings — gets at most **two** cycles per session once two different steps have contributed to them; the third stops the run and asks for an **architectural decision**, presenting the pattern across steps rather than asking about one more finding. Both counters are session-scoped and reset once the user resolves an escalation and asks to continue
- **Manual checkpoints are honoured.** A step whose plan text explicitly asks for human confirmation, sign-off, approval, or manual verification pauses in both git modes. `write-blueprint` now tells plan authors to state such a checkpoint explicitly, since execution no longer pauses by default
- **`<PASS-BEFORE-COMMIT>` added to the Iron Laws.** Remediation makes the preceding review and verification stale, so a step is complete only when a full review pass *and* a full verification pass both succeed against its final state. The commit captures that state — never an intermediate attempt, never an open finding, never a partially remediated tree. This is the pre-existing no-partial-commit rule, promoted to a law and extended to cover remediation
- **Guardrails against green-washing.** The remediation prompt forbids weakening a test, deleting a test, loosening an assertion, adding a skip/xfail marker, or relaxing a lint or type-checker rule to make a check pass — if a check can only pass that way, it is escalated. It also forbids touching advisory findings, refactoring, or tidying. A finding the remediation subagent believes is wrong is **disputed** rather than silently accepted, with evidence; the next review is the arbiter
- **Never re-run only the failed check.** After remediation, the full review and the full verification checklist both re-run, because a fix can regress a check that was green
- **Remediation stays visible.** Step reports carry the remediation cycles run and what each one fixed, plus any disputed findings; condensed step lines carry a cycle count; milestone and end-of-run summaries carry a remediation roll-up, so auto-fixed work can be reviewed in one place
- **Subagent-Only law extended** to cover remediation — a one-character fix for a review finding is still dispatched as a subagent — and `references/subagent-prompts.md` gains the fourth (remediation) template
- Ten new entries in the Red Flags table covering the new failure policy, and READMEs (root and plugin) updated to describe remediate-or-escalate

### Audits

- No changes; remains at 2.1.0

## [2.2.0] - 2026-06-04

Simplify `execute-blueprint`'s pause model by removing the fixed-size step batching, and reinforce the subagent-driven, lean-context design of both blueprint skills. The `audits` plugin is unchanged and remains at 2.1.0.

### Blueprints (2.2.0)

- Remove the "batches of up to three steps" grouping from `execute-blueprint`. The batch concept is redundant now that plans are organized into milestones: pause cadence is governed entirely by git mode and milestone boundaries. Steps still execute strictly serially — each step completes its full Build → Review → Verify cycle before the next step's build begins — but there is no longer any fixed-size grouping, no per-batch summary, and no batch-boundary logic. The "Batching and Order" section is replaced with "Execution Order"; the batch-summary reporting line and the batch-related red-flag guidance are removed
- Restate the pause cadence cleanly: **user-managed** pauses after every step; **skill-managed** commits each passing step and either runs a simple (single-document) plan straight through to the end, or — for a complex (multi-milestone) plan — pauses at each milestone boundary by default, with a full-auto option that runs across all milestones without pausing
- The Git Mode Gate now presents **skill-managed as the default option**. It remains a hard gate that is surfaced every session; the default is a recommendation, not a license to skip the question
- Reinforce the lean-context rationale throughout `execute-blueprint`: every build, review, and verification runs inside a subagent whose context is reclaimed on return, so the coordinating conversation holds only the plan and each subagent's distilled summary — never file contents or tool output. This is what keeps the context window lean across long, multi-milestone plans
- `write-blueprint` gains a "Context Discipline (Subagent-Driven)" design principle and now recommends dispatching exploration subagents for bulk codebase reading during Step 1 (in addition to the existing adversarial plan-review subagent), so planning also stays a lean coordinator that holds distilled findings rather than the contents of every file surveyed
- READMEs (root and plugin) and the `/blueprints:setup` rationale updated to drop batching language and describe the subagent-driven, lean-context model

### Audits

- No changes; remains at 2.1.0

## [2.1.0] - 2026-05-17

Add HTML as a first-class output format alongside Markdown for both plugins. HTML is the default; Markdown output is fully preserved. HTML files are self-contained — inline CSS, inline SVG, no external resources — and follow a stable structural contract (required IDs, classes, `data-*` attributes) so `execute-blueprint` can mutate progress state surgically.

### Audits (2.1.0)

- Add `report-template-html.md` for the `code-audit` skill and a parallel one for `security-audit`, defining the HTML scaffold, required IDs/classes/`data-*` attributes, and when to use HTML-specific affordances (inline SVG dataflow / trust-boundary diagrams, severity badges, CWE/OWASP tag styling, collapsible Intent Brief themes, anchor cross-references between findings). The existing Markdown templates are unchanged behaviourally; each now declares its format in its title and points at the HTML variant
- Introduce a `<FORMAT-GATE>` hard gate before scope resolution in both audit skills: the launching message is scanned for HTML/MD signals; absent a signal, the user is asked once and the answer fixes the format for the run. Pre-stated formats in the launching message bypass the gate without ceremonial reconfirmation
- Re-audit search now scans both `AUDIT-REPORT-*.html` and `AUDIT-REPORT-*.md` (and likewise for `SECURITY-AUDIT-REPORT-*`), picks the most recent by file modification time, and lets the new report's format float independently of the prior report's

### Blueprints (2.1.0)

- Add `step-template-html.md` for the `write-blueprint` skill, defining the HTML scaffold, the structural contract (`<article class="step" data-step="N" data-status="..." id="step-N">` and friends), the explicit `data-status` → badge-text → `✅ ` prefix mapping for all five states (`pending`/`active`/`complete`/`failed`/`skipped`), HTML-specific affordances to use when they add information (inline SVG dependency graphs for non-linear step dependencies, anchor cross-references between steps), and the milestone-folder README scaffold. The existing Markdown step template is unchanged behaviourally
- `write-blueprint` defaults to HTML silently; HTML/MD signals in the launching message are honoured without a proactive gate. Genuinely ambiguous launching messages get one targeted question, after which the resolution is documented in the plan header
- `execute-blueprint` detects plan format by file extension and applies format-specific progress mutations: for Markdown, the existing `✅ ` heading prefix and `- [ ]` → `- [x]` mutations; for HTML, four surgical mutations (`data-status="pending"` → `"complete"`, badge text `Pending` → `Complete`, `✅ ` prepended to the step `<h3>`, `checked` attribute added to every verification `<input type="checkbox">` in the step's `phase-verify` section). Mutations use `Edit`-style surgical replacement, never full-file rewrites
- Subagent dispatch documented to pass the inner HTML of the matching section verbatim when the plan is HTML-format; semantic markup (lists, code blocks, emphasis, links) is preserved, not flattened. A note in `references/subagent-prompts.md` makes this explicit for the build / review / verification subagent templates
- Mixed-format milestone folders (out-of-spec for `write-blueprint`) are detected by `execute-blueprint` and surfaced to the user with a question about which extension to use, rather than silently picked
- READMEs and skill body wording updated to be format-neutral (no more "Markdown report" or "checkmarks" as the only options)

## [2.0.0] - 2026-05-07

**Breaking change.** The two plugins and three of their skills have been renamed for consistency. Both plugins are now plural to reflect that they each ship multiple skills, and each skill name reads naturally as a noun or verb-noun pair.

| Old name | New name |
| --- | --- |
| `blueprint` (plugin) | `blueprints` |
| `code-audit` (plugin) | `audits` |
| `blueprint:blueprint` (skill) | `blueprints:write-blueprint` |
| `blueprint:execute` (skill) | `blueprints:execute-blueprint` |
| `code-audit:audit` (skill) | `audits:code-audit` |
| `code-audit:security-audit` (skill) | `audits:security-audit` (skill name unchanged; plugin renamed) |
| `/blueprint:setup` (command) | `/blueprints:setup` |

### Blueprints (2.0.0; was `blueprint` 1.3.5)

- Rename plugin from `blueprint` to `blueprints`; rename skills from `blueprint` to `write-blueprint` and from `execute` to `execute-blueprint`; rename command from `/blueprint:setup` to `/blueprints:setup`
- Update `feedback_blueprints_preference.md` (was `feedback_blueprint_preference.md`); the setup command now updates an existing legacy memory in place rather than duplicating it
- Update `write-blueprint` skill body to invoke `blueprints:execute-blueprint` (was `blueprint:execute`) for execution

### Audits (2.0.0; was `code-audit` 1.2.1)

- Rename plugin from `code-audit` to `audits`; rename code-quality skill from `audit` to `code-audit`. The `security-audit` skill name is unchanged (only its containing plugin was renamed)
- Update sibling-skill references in `code-audit/references/categories.md` and `security-audit/SKILL.md` to use the new `code-audit` skill name
- README rewritten to lead with the plural plugin name and group security-audit domains by the same identity / input-handling / cross-cutting / infrastructure groupings as the skill's Reference Index

### Migration

Existing installs of `blueprint` or `code-audit` continue to function but stop receiving updates. To move to 2.0.0, run these commands inside an interactive Claude Code session:

1. Uninstall the old plugins:

   ```
   /plugin uninstall blueprint@gianchub-plugins
   /plugin uninstall code-audit@gianchub-plugins
   ```

2. Refresh the marketplace cache so the renamed plugins become visible (without this step, `/plugin install blueprints@…` may fail because the local catalog still lists the old names; a session restart alone does not refresh):

   ```
   /plugin marketplace update gianchub-plugins
   ```

3. Install the renamed plugins:

   ```
   /plugin install blueprints@gianchub-plugins
   /plugin install audits@gianchub-plugins
   ```

If `/blueprint:setup` was previously run on a project, run `/blueprints:setup` to point the project preference at the renamed plugin. The old `feedback_blueprint_preference.md` memory is updated in place rather than duplicated.

## [1.3.6] - 2026-05-07

### Code Audit (1.2.1)

- Multi-agent review pass on the new `security-audit` skill and the modified `audit` skill; all corrections applied in a single follow-up
- Resolve a contradiction in `references/severity-guide.md`: the "Critical Reservation" rule now matches the policy in `SKILL.md` and `references/exploit-scenarios.md` (High/Critical findings whose exploit scenario cannot be constructed ship at assessed severity marked "Exploit Scenario — Not Confirmed", never silently downgraded)
- Fix factual errors in security guidance: PBKDF2-HMAC-SHA512 minimum iteration count corrected to 220,000 (was 210,000); React `javascript:` URL behaviour corrected (16.9+ warns but does not block); `csurf` flagged as deprecated and archived rather than as a valid choice; `node-fetch` noted as in maintenance mode (lead recommendation is built-in `fetch` / `undici`); `Cipher.getInstance("AES")` default mode qualified as SunJCE-specific; Python `tarfile` `filter='data'` parameter clarified with backport list and 3.14 default; Python EOL guidance updated to 3.10 minimum; Go `math/rand/v2` corrected to 1.22; `WebSecurityConfigurerAdapter` noted as removed in Spring Security 6.0
- Eliminate the Impact / Exploitability "Moderate" naming collision: rename Exploitability "Moderate" to "Multistep" in the severity matrix, definition table, worked examples, and report template field format
- Fix Threat Model Brief scope claim ("drives Phases 4–6", not 3–6 — Phase 3 runs in parallel and does not consume the Brief)
- Remove a stray closing code fence in `references/report-template.md` that broke rendering
- Deduplicate CWE listings in `references/owasp-cwe-mapping.md` (CWE-916 was listed twice; CWE-829 appeared under two categories; CWE-918 split out into its own SSRF section)
- Tighten the `audit` skill after security removal: error-message-leak example reframed for code quality (security-flavored examples deferred to `security-audit`); cross-file analysis bullet updated (no more "injection vulnerabilities" wording); severity-guide rephrased to drop "blast radius" and "cross-trust-boundary" vocabulary; README partition threshold corrected to "50+ files or 10,000+ lines"
- Trim `security-audit/SKILL.md` Phase 4 source/sink enumeration that duplicated `references/source-sink-mapping.md`; add inline pointers to `references/severity-guide.md` (Phase 5) and `references/owasp-cwe-mapping.md` (Phase 6) so progressive disclosure fires; expand trigger phrases (threat model, secure code review, vulnerability assessment, CSRF, supply chain, IDOR alone)
- Soften the "phases run sequentially" claim — Phases 2 and 3 may run in parallel
- Reorganize `security-audit` Reference Index into four semantic groups (identity & access, input handling & injection, cross-cutting concerns, infrastructure & supply chain); these groupings also define the canonical domain-category sort order for the report
- Fix second-person voice across multiple files (SKILL.md, intent-discovery, crypto, business-logic, xss-csrf-frontend, cicd, ssrf-redirect-url, csharp-dotnet); apply consistent backticking on cross-references between markdown files
- Smaller corrections: Example 2 in severity-guide rewritten to match the matrix (was claiming "becomes Critical" where matrix would say High); SMS MFA expiry guidance updated (5–10 min industry norm, drop the unrealistic 3-min floor); argon2id parameter framing updated to list all OWASP profiles; SnakeYAML / Psych / `xml.etree` notes refined; `gopher://` claim qualified to curl/PHP; Rust `thread_rng()` warning strengthened; numerous typos and awkward phrasings polished

## [1.3.5] - 2026-05-07

### Blueprint

- Restructure execute skill around explicit Iron Laws and ordered hard gates so the workflow is followed to the letter
- Add `<SUBAGENT-ONLY>` law: main conversation orchestrates only — every build, review, and verification runs in a subagent regardless of step or plan size
- Add `<HARD-GATES>` law making Phases 1–3 sequential gates that must complete before step execution begins
- Promote git mode question to a `<GIT-MODE-GATE>` block (matches blueprint skill's gate convention) — must be asked every session, never inferred
- Promote cadence question to a conditional `<CADENCE-GATE>` block, fired only for skill-managed + complex plans, removing the previous triple-nested bullet structure
- Add an "Execution Phases" ordered overview at the top so the gate ordering is unmissable
- Add a "Red Flags" rationalization table covering common skip patterns (small plan, simple step, batch builds, reuse last session's mode, etc.)
- Reorder so subagent dispatch contract precedes batching/order rules
- Clarify "Pre-stated answer" handling on both gates: when the user's first message specifies a mode/cadence unambiguously (e.g., "in skill-managed mode", "full auto"), accept it as the gate answer rather than re-asking ceremonially

### Code Audit (1.2.0)

- Split the plugin into two complementary skills: `audit` (code quality) and `security-audit` (security only); the two can run independently or in sequence on the same codebase
- Add `security-audit` skill with a six-phase workflow: scope → threat-model → security-intent discovery → source/sink map → systematic analysis (file-level + cross-trust-boundary dataflow + exploit-scenario construction) → report
- Add `Impact × Exploitability × Exposure` severity model with per-exposure-tier matrix and threat-model modifiers (PHI floor, hostile multi-tenancy, defense-in-depth credit); same weakness scores differently per application context
- Require concrete exploit scenarios for High and Critical findings; when a scenario cannot be constructed despite the underlying weakness being clear, the finding still ships at its assessed severity marked "Exploit Scenario — Not Confirmed" with explicit reasoning
- Tag every finding with CWE ID(s) and OWASP Top 10 / API Top 10 mapping; CWE Top 25 (2024) reference included for prioritization
- Cover 15 security domains in dedicated checklists: auth-and-session, authorization (IDOR/BOLA, BFLA, mass assignment, tenancy), injection, XSS/CSRF/frontend, SSRF/redirect/URL parsing, crypto, deserialization, file handling, secrets-and-keys (current code + git history), error-and-logging, business-logic, api-security (OWASP API Top 10), dependencies, containers-iac, ci-cd
- Add language-specific footgun references for Python, JavaScript/TypeScript, Java/Kotlin, Go, Ruby, PHP, C#/.NET, Rust, and C/C++; loaded only when the threat-model brief flags the language as in scope
- Produce a separate `SECURITY-AUDIT-REPORT-YYYY-MM-DD.md` report at project root with finding fields including CWE/OWASP/severity-decomposition/exploit-scenario plus appendices for the source/sink map, coverage limitations, and re-audit delta
- Remove "Security vulnerabilities" category from the `audit` skill; its triggers ("security audit", "find vulnerabilities") now route to `security-audit`. The `audit` skill retains seven categories: concurrency, dead code, anti-patterns, performance, correctness, error handling, test quality
- Retune the `audit` skill's severity guide to drop security-specific examples and focus on code-quality-relevant Critical/High examples (data corruption, silently wrong results, race conditions on persistent state, resource exhaustion)
- Renumber `audit` checklist categories accordingly (1–7); update plugin README to describe both skills; extend keywords with `owasp`, `pentest`, `cwe`

## [1.3.4] - 2026-04-08

### Blueprint

- Replace DOT diagram with prose workflow summary to save tokens
- Add trigger phrases "plan this refactoring" and "help me plan" to skill description
- Add partial execution trigger phrases ("execute step 3", "run steps 3-5") to execute skill description
- Remove duplicated Subagent Architecture table, Batching Rules section, and Additional Resources section from execute skill (~400 words of redundancy)
- Fold milestone boundary and remaining-steps batching rules into Step Execution section
- Strengthen post-approval git commit instruction with explicit warning

### Code Audit (1.1.1)

- Add guidance for locating prior audit reports during re-audits (search for `AUDIT-REPORT-*.md`, use most recent)
- Clarify partitioning thresholds are disjunctive (either 50 files or 10K lines triggers parallelism)
- Align subagent labels between SKILL.md and intent-discovery.md (remove "Subagent A/B/C" prefixes)
- Fix heading level in severity-guide.md (`##` → `#`)

## [1.3.3] - 2026-04-08

### Blueprint

- Make tool confirmation a hard gate before clarifying questions — the workflow now stops after presenting the discovered tool chain and waits for user confirmation before asking any clarifying questions
- Split step 1 into "Explore Codebase and Discover Tooling" (step 1) and "Clarify Requirements" (step 1b) with a `<TOOL-CONFIRMATION-GATE>` between them
- Reorder workflow diagram from `explore -> clarify -> confirm` to `explore -> confirm -> clarify`
- Update PLANNING-GATE to describe two sequential hard gates

## [1.3.2] - 2026-04-07

### Blueprint

- Prevent skill-managed commits on steps requiring human intervention — no commit is created until the intervention is fully resolved and the step passes all remaining phases

## [1.3.1] - 2026-04-07

### Blueprint

- Consolidate fast-path rules and fix step numbering
- Tighten step reporting format for skill-managed continuous modes

## [1.3.0] - 2026-04-07

### Blueprint

- Add pause cadence options to execution skill: simple plans run without stopping, complex multi-milestone plans pause at milestone boundaries by default, with a full-auto option
- Restructure git handling with user-managed and skill-managed modes
- Update READMEs with detailed execution workflow documentation

## [1.2.1] - 2026-03-23

### Code Audit (1.1.0)

- Add Step 3 — Discover Intent: scans codebase for documented design decisions, trade-offs, conventions, and known limitations before analysis begins, reducing false positives
- Add three parallel discovery subagents: Documentation Scanner, Code Intent Scanner, History Scanner
- Add `references/intent-discovery.md` with detailed subagent prompts and Intent Brief template
- Add "Context & Intent" section to audit report template for transparency into what was considered
- Update workflow from 5 steps to 6 steps with intent cross-referencing in analysis phases

## [1.2.1] - 2026-03-22

### Blueprint

- Reinforce setup command with clearer instructions and improved robustness

## [1.2.0] - 2026-03-22

### Blueprint

- Add "Propose Approaches" step to the planning workflow (step 2), requiring 2-3 implementation strategies with trade-offs before committing to a plan
- Add planning gate to prevent plan generation before tool chain confirmation and ambiguity resolution
- Add process flow digraph visualizing the full planning pipeline
- Delegate visual companion integration to superpowers plugin instead of bundling it
- Trim redundant content from SKILL.md (2,983 → 2,339 words) by consolidating into reference files
- Add partial/selective step execution support to execute skill
- Add PHP and Swift language support to tool-discovery.md
- Standardize skill descriptions to third-person format with consistent trigger phrases
- Re-add scope change handling guidance
- Fix digraph ordering, define "critical ambiguity", merge duplicate bullets, and various polish fixes

### Code Audit

- Extract Severity Classification Guide to references/severity-guide.md
- Add incremental re-audit guidance
- Fix audit category count from seven to eight (test quality)
- Trim verbose content from audit SKILL.md

## [1.1.0] - 2026-03-19

### Blueprint

- Add adversarial plan review process with a dedicated subagent that identifies weaknesses before execution
- Review criteria cover completeness, step ordering, acceptance criteria, and risk assessment
- Add plan-review-subagent.md with prompt template for the review subagent
- Fast-path for small plans (2 or fewer steps) skips the review subagent
- Review subagent operates without full planning context for a more objective review
- Add additional tool-discovery configuration options (ruff formatter, bun.lock support)
- Clarify batch processing and acceptance criteria handling during execution review phase

## [1.0.1] - 2026-03-18

### Blueprint

- Clarify setup command functionality and preference routing to blueprint over other plugins
- Update documentation for setup command

### Code Audit

- Version bump to align with blueprint

## [1.0.0] - 2026-03-18

### Blueprint

- Add LICENSE, README, and structured plugin.json with author, repository, and keywords
- Refactor plugin structure for marketplace discoverability

### Code Audit

- Add LICENSE, README, and structured plugin.json with author, repository, and keywords
- Refactor plugin structure for marketplace discoverability

## [Pre-release] - 2026-03-15 to 2026-03-18

### Project

- Initial commit and marketplace scaffolding
- Add marketplace manifest (marketplace.json) for plugin discovery
- Add design specification and implementation plan for the plugins marketplace
- Add .gitignore and remove tracked settings.local.json

### Blueprint

- Add blueprint skill with collaborative implementation planning workflow
- Add execute skill with subagent orchestration for plan execution
- Add reference files: step-template.md, tool-discovery.md, subagent-prompts.md
- Add `/blueprint:setup` command to set blueprint as default planning skill
- Improve plan update rules
- Expand thin sections in skill documentation

### Code Audit

- Add code-audit skill for language-agnostic codebase auditing
- Add reference files: categories.md, report-template.md
- Expand thin sections in skill documentation

### Improvements (PR #3)

- Refine SKILL.md to integrate tooling discovery into task understanding phase
- Enhance README with detailed plugin descriptions and installation instructions
- Remove workflow summaries from skill descriptions to encourage thorough reading
- Replace vague guidance with contextual instructions for quality
- Add fast-path for tool discovery in narrowly-scoped plans
- Streamline audit category selection and improve user experience
