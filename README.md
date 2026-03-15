# claude-plugins

Personal Claude Code plugins marketplace.

## Plugins

| Plugin | Skills | Description |
|--------|--------|-------------|
| **code-audit** | `audit` | Language-agnostic codebase auditing with structured severity-ranked reports |
| **blueprint** | `blueprint`, `execute` | Collaborative implementation planning with build-review-verify cycles, plus plan execution via subagents |

## Installation

Register this repository as a marketplace in Claude Code:

1. Push this repo to GitHub
2. In Claude Code, add the marketplace via `~/.claude/plugins/known_marketplaces.json`
3. Install individual plugins through the plugin install flow

## Auto-Update

Pushing to `main` triggers automatic updates. Claude Code tracks the git commit SHA of installed plugins and pulls the latest version on session start.

## License

See [LICENSE](LICENSE).
