# claude-plugins

Personal Claude Code plugins marketplace.

## Plugins

| Plugin | Skills | Description |
|--------|--------|-------------|
| **code-audit** | `audit` | Language-agnostic codebase auditing with structured severity-ranked reports |
| **blueprint** | `blueprint`, `execute` | Collaborative implementation planning with build-review-verify cycles, plus plan execution via subagents |

## Installation

### 1. Register the marketplace

Add an entry to `~/.claude/plugins/known_marketplaces.json`:

```json
{
  "claude-plugins-official": { ... },
  "gianchub-plugins": {
    "source": {
      "source": "github",
      "repo": "gianchub/claude-plugins"
    },
    "installLocation": "~/.claude/plugins/marketplaces/gianchub-plugins"
  }
}
```

### 2. Install plugins

Restart Claude Code. The marketplace will be discovered automatically. Install plugins through the plugin install flow (`/install-plugin` or the plugins menu).

### 3. Verify

After installation, the skills should appear in your session. Try:
- `"audit this codebase"` to trigger the audit skill
- `"create a blueprint for ..."` to trigger the blueprint skill
- `"execute the plan"` to trigger the execute skill

## Auto-Update

Pushing to `main` triggers automatic updates. Claude Code tracks the git commit SHA of installed plugins and pulls the latest version on session start.

## License

See [LICENSE](LICENSE).
