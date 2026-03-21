# claude-plugins

A collection of Claude Code plugins for planning, auditing, and more.

## Plugins

### Blueprint

Collaborative implementation planning and execution with build-review-verify cycles.

- **Dialogue-first planning** — asks clarifying questions and explores the codebase exhaustively before writing a single plan step. Never plans in a vacuum.
- **Automatic tool discovery** — detects your project's test runner, linter, formatter, and type checker from config files and CI pipelines, then embeds the exact verification commands into every step.
- **Build-review-verify cycle** — every step goes through three phases: build the thing, adversarial review targeting likely failure modes, then run the full tool chain to verify.
- **Adaptive format** — produces a single plan document for small tasks or a milestone folder structure for larger efforts, based on complexity.
- **Subagent execution** — the `execute` skill drives plans step-by-step using dedicated subagents for each phase, with git handling (manual or automatic) and batch progress tracking.

**Skills:**

| Skill | Description | Trigger with |
| --- | --- | --- |
| `blueprint` | Produces structured implementation plans through codebase exploration, tool discovery, iterative clarification, and approach comparison | "create a blueprint", "make a plan", "design the architecture" |
| `execute` | Drives blueprint plans through their build-review-verify cycles using subagents, with configurable git handling | "execute this blueprint", "run the plan", "start building from the plan" |

**Commands:**

| Command | Description |
| --- | --- |
| `/blueprint:setup` | Set blueprint as the default planning skill for the current project |

The `setup` command saves a preference to Claude's project memory so that generic planning requests ("make a plan", "execute the plan") route to the blueprint plugin instead of other plugins with overlapping triggers. Run it once per project. You can still invoke other planning skills explicitly by name when needed.

### Code Audit

Language-agnostic codebase auditing with structured severity-ranked reports.

- **Exhaustive analysis** — reads every line of in-scope code. No skimming, no sampling.
- **Cross-file tracing** — traces data flows from entry points through processing layers to outputs, catching issues that only manifest through component interaction.
- **Seven audit categories** — security vulnerabilities, race conditions, dead code, anti-patterns, performance, correctness, and error handling gaps. Categories are configurable per audit.
- **Severity-ranked output** — produces a deduplicated Markdown report at the project root with findings ranked by severity, each with file path, line number, impact, and a concrete recommendation.
- **Parallel subagents** — for large codebases, splits the audit across parallel subagents by module, then merges and deduplicates findings.

**Skills:**

| Skill | Description | Trigger with |
| --- | --- | --- |
| `audit` | Performs a thorough code audit producing a structured, severity-ranked report covering security, correctness, performance, and more | "audit this codebase", "security audit", "find vulnerabilities" |


## Installation

### 1. Add the marketplace

```sh
claude plugin marketplace add gianchub/claude-plugins
```

### 2. Install a plugin

```sh
claude plugin install blueprint@gianchub-plugins
claude plugin install code-audit@gianchub-plugins
```

### 3. Enable auto-update

Auto-update is off by default. To enable it, run `/plugin` in Claude Code, navigate to the marketplace, and enable auto-update from its settings.

### 4. Verify

Restart Claude Code, then try:

- "create a blueprint for ..." to trigger the blueprint skill
- "execute the plan" to trigger the execute skill
- "audit this codebase" to trigger the audit skill

## Uninstallation

Remove a plugin:

```sh
claude plugin uninstall blueprint@gianchub-plugins
claude plugin uninstall code-audit@gianchub-plugins
```

Remove the marketplace entirely:

```sh
claude plugin marketplace remove gianchub-plugins
```

## Updating

With auto-update enabled, plugins update automatically on session start. To manually trigger an update:

```sh
claude plugin update blueprint@gianchub-plugins
claude plugin update code-audit@gianchub-plugins
```

## License

See [LICENSE](LICENSE).