# claude-plugins

A collection of Claude Code plugins for planning, auditing, and more.

## Plugins

### Blueprints

Collaborative implementation planning and execution with build-review-verify cycles.

- **Dialogue-first planning** — asks clarifying questions and explores the codebase exhaustively before writing a single plan step. Never plans in a vacuum.
- **Automatic tool discovery** — detects your project's test runner, linter, formatter, and type checker from config files and CI pipelines, then embeds the exact verification commands into every step.
- **Build-review-verify cycle** — every step goes through three phases: build the thing, adversarial review targeting likely failure modes, then run the full tool chain to verify.
- **Adaptive format** — produces a single plan document for small tasks or a milestone folder structure for larger efforts, based on complexity.
- **Subagent execution** — the `execute-blueprint` skill drives plans step-by-step using dedicated subagents for each phase, with git handling (manual or automatic with adaptive pause cadence) and progress tracking.

**Skills:**

| Skill | Description | Trigger with |
| --- | --- | --- |
| `write-blueprint` | Produces structured implementation plans through codebase exploration, tool discovery, iterative clarification, and approach comparison. Fast-path for ≤2-step plans | "create a blueprint", "make a plan", "design the architecture" |
| `execute-blueprint` | Drives blueprint plans through their build-review-verify cycles using subagents, with adaptive pause cadence and git handling | "execute this blueprint", "run the plan", "start building from the plan" |

**Commands:**

| Command | Description |
| --- | --- |
| `/blueprints:setup` | Set the blueprints plugin as the default for planning and execution in the current project |

The `setup` command saves a preference to Claude's project memory so that generic planning requests ("make a plan", "execute the plan") route to the blueprints plugin instead of other plugins with overlapping triggers. Run it once per project. You can still invoke other planning skills explicitly by name when needed.

### Audits

Language-agnostic code-quality and security auditing with structured severity-ranked reports.

- **Exhaustive analysis** — reads every line of in-scope code. No skimming, no sampling.
- **Cross-file tracing** — traces data flows from entry points through processing layers to outputs, catching issues that only manifest through component interaction.
- **Two complementary skills** — `code-audit` covers code quality (concurrency, dead code, anti-patterns, performance, correctness, error handling, test quality); `security-audit` covers security exclusively (OWASP Top 10, OWASP API Top 10, CWE-mapped findings, threat-model-first workflow, source-to-sink dataflow, exploit scenarios for High/Critical findings).
- **Severity-ranked output** — produces a deduplicated Markdown report at the project root. Code-quality findings use an impact-based scale; security findings use Impact × Exploitability × Exposure with threat-model modifiers.
- **Parallel subagents** — for large codebases, splits the audit across parallel subagents by module, then merges and deduplicates findings.

**Skills:**

| Skill | Description | Trigger with |
| --- | --- | --- |
| `code-audit` | Comprehensive code-quality audit covering concurrency, dead code, anti-patterns, performance, correctness, error handling, and test quality | "audit this codebase", "code audit", "find dead code", "performance audit" |
| `security-audit` | Threat-model-first security audit covering 15 domains (auth, authz, injection, XSS/CSRF, SSRF, crypto, deserialization, file handling, secrets, API security, dependencies, IaC, CI/CD, etc.) plus per-language footguns | "security audit", "find vulnerabilities", "OWASP audit", "pentest this code" |


## Installation

All commands shown below are slash commands; run them inside an interactive Claude Code session. (CLI equivalents — `claude plugin marketplace add`, `claude plugin install`, etc. — also exist for most of these and behave identically; pick whichever fits your workflow.)

### 1. Add the marketplace

```
/plugin marketplace add gianchub/claude-plugins
```

### 2. Install a plugin

```
/plugin install blueprints@gianchub-plugins
/plugin install audits@gianchub-plugins
```

### 3. Enable auto-update

Auto-update is off by default. To enable it, run `/plugin` to open the interactive plugin manager, navigate to the **Marketplaces** tab, and enable auto-update from its settings.

### 4. Verify

Restart Claude Code, then try:

- "create a blueprint for ..." to trigger the `write-blueprint` skill
- "execute the plan" to trigger the `execute-blueprint` skill
- "audit this codebase" to trigger the `code-audit` skill
- "security audit" to trigger the `security-audit` skill

## Uninstallation

Remove a plugin:

```
/plugin uninstall blueprints@gianchub-plugins
/plugin uninstall audits@gianchub-plugins
```

Remove the marketplace entirely:

```
/plugin marketplace remove gianchub-plugins
```

## Updating

With auto-update enabled, plugins update automatically on session start. To trigger an update manually:

- **From the interactive UI** — run `/plugin`, switch to the **Installed** tab, and update from there.
- **From the shell** — there is no `/plugin update` slash form for individual plugins. Use the CLI:

  ```sh
  claude plugin update blueprints@gianchub-plugins
  claude plugin update audits@gianchub-plugins
  ```

To refresh the marketplace's local cache (so new plugins published to the marketplace become visible):

```
/plugin marketplace update gianchub-plugins
```

## Migrating from earlier versions

The 2.0.0 release renamed the plugins (`blueprint` → `blueprints`, `code-audit` → `audits`) and renamed several skills (`blueprint` → `write-blueprint`, `execute` → `execute-blueprint`, `audit` → `code-audit`; `security-audit` is unchanged). Existing installations of `blueprint` or `code-audit` keep working but stop receiving updates. To move forward, run these commands inside an interactive Claude Code session:

1. Uninstall the old plugins:

   ```
   /plugin uninstall blueprint@gianchub-plugins
   /plugin uninstall code-audit@gianchub-plugins
   ```

2. Refresh the marketplace cache so Claude Code sees the renamed plugins (without this step, `/plugin install blueprints@…` may fail because the local catalog still lists the old names; a session restart alone does not refresh):

   ```
   /plugin marketplace update gianchub-plugins
   ```

3. Install the renamed plugins:

   ```
   /plugin install blueprints@gianchub-plugins
   /plugin install audits@gianchub-plugins
   ```

If you previously ran `/blueprint:setup`, run `/blueprints:setup` again so the project preference points at the renamed plugin.

## License

See [LICENSE](LICENSE).
