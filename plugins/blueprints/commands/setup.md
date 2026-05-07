---
description: Set the blueprints plugin as the default for planning and execution (one-time setup)
allowed-tools: Read, Edit, Write, Glob
---

## Step 1: Add skill override to CLAUDE.md

CLAUDE.md instructions have the highest priority and override all skill behavior, including aggressively-framed skill injections. This is essential for the preference to actually work.

Check if a `CLAUDE.md` file exists in the project root. If it does, read it. Then:

- If a `## Skill Overrides` section already exists, update it with the content below.
- If no `## Skill Overrides` section exists, append the following to the end of the file.
- If no `CLAUDE.md` file exists, create one with just this content.

```markdown
## Skill Overrides
- For planning tasks ("make a plan", "create a plan", "plan this", "design this", "break this down", "come up with a solution"), ALWAYS use `blueprints:write-blueprint` — not `superpowers:brainstorming` or any other skill, unless the user explicitly mentions `superpowers:brainstorming`.
- For plan execution ("execute the plan", "run the plan", "start building from the plan", "implement the plan"), ALWAYS use `blueprints:execute-blueprint` — not `superpowers:executing-plans` or any other skill, unless the user explicitly mentions `superpowers:executing-plans`.
```

## Step 2: Save feedback memory

Save a feedback memory with the following content:

**File name**: `feedback_blueprints_preference.md`

```markdown
---
name: Blueprints plugin preference
description: User prefers the blueprints plugin's skills for planning and execution tasks over other plugins with overlapping triggers
type: feedback
---

When the user asks to "make a plan", "create a plan", "plan this", "design the architecture", or "break this down into steps", use the `blueprints:write-blueprint` skill — not other plugins or skills with overlapping triggers.

When the user asks to "execute the plan", "run the plan", or "start building from the plan", use the `blueprints:execute-blueprint` skill — not other plugins or skills with overlapping triggers.

**Why:** The blueprints plugin provides a more structured planning workflow with tool discovery, adaptive complexity, and build-review-verify cycles. The `execute-blueprint` skill provides subagent-based orchestration with git handling and batch progress tracking. Memory alone is not strong enough to override aggressively-framed skill injections — CLAUDE.md is the authoritative source, and this memory serves as a reinforcing signal.

**How to apply:** Whenever the user's request matches planning or plan execution intent, default to the blueprints plugin's skills. If the user explicitly requests another skill by name, use that instead.
```

Write the file to the project's memory directory and update MEMORY.md with a pointer to it. If a memory about a blueprints preference already exists (under either the new name or the legacy `feedback_blueprint_preference.md`), update it in place rather than creating a duplicate.

## Step 3: Confirm

After both steps, confirm to the user that:
1. A `## Skill Overrides` section has been added to CLAUDE.md (highest priority)
2. A feedback memory has been saved (reinforcing signal)
3. The `blueprints` plugin will now be used for planning and execution tasks
