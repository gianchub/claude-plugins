# Blueprint

Collaborative implementation planning and execution with build-review-verify cycles per step.

Blueprint turns vague feature requests, architectural changes, and refactoring goals into structured, sequenced plans, reviews them adversarially for gaps and weaknesses, then executes them with dedicated subagents for each phase. Every step passes through a build, adversarial review, and tool-based verification cycle before it is marked complete, ensuring production-ready output with no silent failures.

## Installation

Add the marketplace, then install:

```
/plugins marketplace add gianchub/claude-plugins
/plugins install blueprint
```

## Skills

### blueprint

Transforms a task into a structured implementation plan through a collaborative dialogue. The skill explores the codebase to understand project structure and discover tooling (test runners, linters, type checkers, formatters), then asks clarifying questions to nail down scope and constraints before generating the plan. Plans adapt to complexity -- small changes get a single document, larger efforts get milestone folders with multiple files. Every step is written in prose (not code) and includes three phases: build instructions, adversarial review questions, and a verification checklist with concrete tool commands. After generation, an adversarial plan review subagent reads the plan cold and evaluates it for gaps, dependency issues, and architectural flaws before the user approves it for execution.

Trigger with: "create a blueprint", "plan this implementation", "make a plan", "break this down into steps"

### execute

Drives a blueprint plan through its build-review-verify cycles using dedicated subagents. Steps are executed strictly serially in batches of up to three. A build subagent implements the step, a review subagent performs an adversarial code review reading files independently from disk, and a verification subagent runs actual tools and reports pass/fail. All failures surface to the user with full error output -- nothing is auto-fixed or silently skipped. The skill supports two git modes (user-managed or skill-managed commits) and tracks progress with checkmarks in the plan file for cross-session resumability.

Trigger with: "execute this blueprint", "run the plan", "start building from the plan"

## Commands

### /blueprint:setup

One-time setup command that saves a preference to use blueprint skills as the default for planning and execution tasks. After running this command, requests like "make a plan" or "execute the plan" will automatically route to the blueprint plugin skills.

## How It Works

### Planning (blueprint skill)

1. Explores the codebase to understand project structure, entry points, data layer, configuration, tests, and documentation
2. Discovers project tooling by scanning config files, CI pipelines, lock files, and script conventions -- then presents the tool chain to the user for confirmation
3. Asks clarifying questions across multiple rounds to understand scope, constraints, and definition of done
4. Assesses complexity and chooses the output format (single document for up to 8 steps, milestone folder for larger efforts)
5. Generates a structured plan where every step has three phases:
   - **Phase 1 -- Build**: Prose instructions describing what to implement, with acceptance criteria and test expectations
   - **Phase 2 -- Adversarial Review**: Step-specific review questions targeting failure modes, codebase integration, and consistency with established patterns
   - **Phase 3 -- Verification**: Checklist with tool commands from the discovered tool chain, plus step-specific verification items
6. Dispatches an adversarial plan review subagent that reads the plan fresh from disk with no planning context, evaluating completeness, step ordering, acceptance criteria quality, architectural coherence, and risk -- then presents findings to the user for approval before execution can begin

### Execution (execute skill)

1. Reads the plan file and identifies remaining steps by scanning for unmarked step headings
2. Asks which git mode to use (user-managed pauses after each step, skill-managed commits automatically)
3. Groups steps into batches of up to three and executes them strictly serially, dispatching subagents for each phase:
   - **Build subagent**: Implements the step with full filesystem access, returns a structured summary of changes
   - **Review subagent**: Reads all changed files fresh from disk and performs adversarial code review, categorizing findings as blocking or advisory
   - **Verification subagent**: Runs actual tools (test suite, linter, type checker, formatter) and reports pass/fail per checklist item
4. Surfaces all failures to the user with full error output and file references -- no auto-fix, no auto-retry, no silent skipping
5. Marks completed steps with checkmarks in the plan file and ticks verification checkboxes for cross-session resumability

## Requirements

- Claude Code

## License

MIT
