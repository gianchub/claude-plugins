---
description: Set blueprint as the default planning skill (one-time setup)
allowed-tools: Read, Edit, Write
---

Save a feedback memory with the following content:

**File name**: `feedback_blueprint_preference.md`

```markdown
---
name: Blueprint plugin preference
description: User prefers blueprint plugin skills over superpowers for planning and execution tasks
type: feedback
---

When the user asks to "make a plan", "create a plan", "plan this", "design the architecture", or "break this down into steps", use the `blueprint:blueprint` skill — not superpowers:writing-plans or superpowers:brainstorming.

When the user asks to "execute the plan", "run the plan", or "start building from the plan", use the `blueprint:execute` skill — not superpowers:executing-plans.

**Why:** The blueprint plugin provides a more structured planning workflow with tool discovery, adaptive complexity, and build-review-verify cycles. The execute skill provides subagent-based orchestration with git handling and batch progress tracking.

**How to apply:** Whenever the user's request matches planning or plan execution intent, default to the blueprint plugin skill instead of the superpowers equivalent. If the user explicitly requests a superpowers skill by name, use that instead.
```

Write the file to the project's memory directory and update MEMORY.md with a pointer to it. If a memory about blueprint preference already exists, update it rather than creating a duplicate.

After saving, confirm to the user that the preference has been saved to project memory.
