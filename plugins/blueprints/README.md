# Blueprints

Collaborative implementation planning and execution with build-review-verify cycles per step.

The plugin turns vague feature requests, architectural changes, and refactoring goals into structured, sequenced plans, reviews them adversarially for gaps and weaknesses, then executes them with dedicated subagents for each phase. Every step passes through a build, adversarial review, and tool-based verification cycle before it is marked complete, ensuring production-ready output with no silent failures.

## Installation

Add the marketplace, then install:

```
/plugins marketplace add gianchub/claude-plugins
/plugins install blueprints
```

## Skills

### write-blueprint

Transforms a task into a structured implementation plan through a collaborative dialogue. The skill explores the codebase to understand project structure and discover tooling (test runners, linters, type checkers, formatters), then presents the discovered tool chain for user confirmation before proceeding. Once tooling is confirmed, the skill asks clarifying questions to nail down scope and constraints, then proposes 2-3 implementation approaches with trade-offs before generating the plan. Plans adapt to complexity -- small changes get a single document, larger efforts get milestone folders with multiple files. A fast-path for narrowly scoped changes (≤2 steps) streamlines tool confirmation, approach selection, and plan review. Every step is written in prose (not code) and includes three phases: build instructions, adversarial review questions, and a verification checklist with concrete tool commands. After generation, an adversarial plan review subagent reads the plan cold and evaluates it for gaps, dependency issues, and architectural flaws before the user approves it for execution.

Trigger with: "create a blueprint", "write a blueprint", "plan this implementation", "make a plan", "design this feature", "break this down into steps"

### execute-blueprint

Drives a blueprint plan through its build-review-verify cycles using dedicated subagents. Steps are executed strictly serially in batches of up to three. A build subagent implements the step, a review subagent performs an adversarial code review reading files independently from disk, and a verification subagent runs actual tools and reports pass/fail. All failures surface to the user with full error output -- nothing is auto-fixed or silently skipped. In skill-managed mode, commits are never created for partial steps -- if a step requires human intervention (blocking review findings or failed verification), no commit is made until the intervention is resolved and the step passes all remaining phases. The skill supports two git modes: user-managed (pause after each step for manual commits) or skill-managed (automatic commits with adaptive pause cadence -- simple plans run straight through, complex multi-milestone plans pause at milestone boundaries by default with a full-auto option). Progress is tracked with checkmarks in the plan file for cross-session resumability.

Trigger with: "execute this blueprint", "execute the blueprint", "run the plan", "start building from the plan", "implement the plan", "continue the plan", "resume execution"

## Commands

### /blueprints:setup

One-time setup command that configures the `blueprints` plugin as the default for planning and execution tasks in the current project. Adds a `## Skill Overrides` section to the project's CLAUDE.md (highest priority) and saves a reinforcing feedback memory. After running this command, requests like "make a plan" or "execute the plan" will automatically route to the `write-blueprint` and `execute-blueprint` skills. The superpowers equivalents (`superpowers:brainstorming`, `superpowers:executing-plans`) can still be used if explicitly mentioned by name.

## How It Works

### Planning (write-blueprint skill)

1. Explores the codebase to understand project structure, entry points, data layer, configuration, tests, and documentation while discovering project tooling by scanning config files, CI pipelines, lock files, and script conventions
2. Presents the discovered tool chain to the user for confirmation -- this is a hard gate; no clarifying questions or other work happens until the user confirms the tools
3. Asks clarifying questions across multiple rounds to understand scope, constraints, and definition of done -- a second gate prevents plan generation until all critical ambiguities are resolved
4. Proposes 2-3 implementation approaches with trade-offs and a recommendation, then waits for the user to choose before proceeding (fast-path: single obvious approach stated briefly)
5. Assesses complexity and chooses the output format (single document for up to 8 steps, milestone folder for larger efforts)
6. Generates a structured plan where every step has three phases:
   - **Phase 1 -- Build**: Prose instructions describing what to implement, with acceptance criteria and test expectations
   - **Phase 2 -- Adversarial Review**: Step-specific review questions targeting failure modes, codebase integration, and consistency with established patterns
   - **Phase 3 -- Verification**: Checklist with tool commands from the discovered tool chain, plus step-specific verification items
7. Dispatches an adversarial plan review subagent that reads the plan fresh from disk with only a brief scope summary (not the full planning conversation), evaluating completeness, step ordering, acceptance criteria quality, architectural coherence, and risk -- then presents findings to the user for approval before execution can begin
8. Handles scope changes during execution -- recommends starting fresh if changes invalidate most of the plan, or inserts new steps with the same 3-phase structure

### Execution (execute-blueprint skill)

1. Reads the plan file and identifies remaining steps by scanning for unmarked step headings
2. Asks which git mode to use: user-managed (pauses after each step for manual commits) or skill-managed (commits automatically with adaptive pause cadence -- simple plans run without stopping, complex plans pause at milestone boundaries by default, full-auto option skips all pauses)
3. Groups steps into batches of up to three and executes them strictly serially, dispatching subagents for each phase:
   - **Build subagent**: Implements the step with full filesystem access, returns a structured summary of changes
   - **Review subagent**: Reads all changed files fresh from disk and performs adversarial code review, categorizing findings as blocking or advisory
   - **Verification subagent**: Runs actual tools (test suite, linter, type checker, formatter) and reports pass/fail per checklist item
4. Supports partial execution (specific step ranges) and warns about skipped dependencies
5. Surfaces all failures to the user with full error output and file references -- no auto-fix, no auto-retry, no silent skipping. In skill-managed mode, no commit is created for a step that requires human intervention until the intervention is resolved and remaining phases pass cleanly
6. Adapts step reporting to the execution mode -- full reports in user-managed mode, condensed single-line reports in continuous modes with comprehensive summaries at pause points or run completion
7. Marks completed steps with checkmarks in the plan file and ticks verification checkboxes for cross-session resumability

## Requirements

- Claude Code

## License

MIT
