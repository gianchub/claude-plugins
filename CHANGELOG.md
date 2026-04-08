# Changelog

All notable changes to the claude-plugins project are documented in this file.

Version numbers refer to the **blueprint** plugin version, which has been the primary driver of releases. The code-audit plugin version is noted where it differs.

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
