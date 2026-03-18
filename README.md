# claude-plugins

A collection of Claude Code plugins for planning, auditing, and more.

## Plugins

### Blueprint

Collaborative implementation planning and execution with build-review-verify cycles per step.

**Skills:**


| Skill       | Trigger phrases                                                                                                                                    |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `blueprint` | "create a blueprint", "blueprint this feature", "plan this implementation", "make a plan", "design the architecture", "break this down into steps" |
| `execute`   | "execute this blueprint", "run the plan", "execute the plan", "start building from the plan", "implement the blueprint"                            |


### Code Audit

Language-agnostic codebase auditing with structured severity-ranked reports.

**Skills:**


| Skill   | Trigger phrases                                                                                                                          |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `audit` | "audit this codebase", "security audit", "code audit", "find vulnerabilities", "check for bugs", "review code quality", "find dead code" |


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