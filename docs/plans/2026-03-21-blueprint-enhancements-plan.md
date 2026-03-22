# Blueprint Skill Enhancements Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Improve the blueprint skill by incorporating design exploration patterns (approach proposals, hard gates, visual diagrams), trimming redundant prose, and adding visual companion support — all inspired by but not copied from the superpowers:brainstorming skill.

**Architecture:** Edit blueprint/SKILL.md to restructure the workflow (add approach exploration + hard gate), replace verbose prose sections with references, add a dot digraph, and introduce a new `references/visual-companion.md` for optional browser-based architecture visualization during planning.

**Tech Stack:** Markdown, Graphviz dot notation (inline)

---

## Tool Chain

| Category | Tool | Command |
|---|---|---|
| Validation | word count | `wc -w <file>` |
| Validation | content search | `grep -r "<pattern>" plugins/blueprint/` |
| Validation | manual review | Read changed files, verify no verbatim copying |

---

## Steps

### ✅ Step 1: Trim Redundant Content from SKILL.md

**Objective**: Reduce blueprint/SKILL.md word count by ~400-500 words by removing content duplicated in reference files. Currently 2,983 words — target ~2,500 before new content is added.

**Acceptance criteria**:
- "Prose Over Code" in Design Principles reduced to 2-3 sentences referencing `references/step-template.md` for the full policy and exceptions list.
- "What Belongs in a Plan Step" section removed. Before removing, migrate its "Do not include" items (implementation details that go stale, generic advice, premature optimization notes) into `references/step-template.md` Phase 1 guidance as a new "What not to include" sub-section. The "Include" items already exist in step-template.md.
- "Handling Plan Execution" fallback section condensed to 3 sentences: prefer `blueprint:execute`, fallback is one-step-at-a-time with all three phases, mark progress with checkmarks.
- "Handling Ambiguity and Scope Changes" condensed from 7 bullets to 4, removing obvious guidance ("acknowledge the change", "note why it was skipped").
- Adversarial Plan Review section: remove the review categories summary list — it duplicates `references/plan-review-subagent.md`. Keep the existing framing sentence ("The subagent prompt contains the full review methodology..."), the fast-path exception, placeholder substitution, and after-review handling.
- No information is lost — every removed sentence has a corresponding source in a reference file.
- SKILL.md word count is under 2,600 after trimming.

#### Phase 1 — Build

**First**, migrate the "Do not include" items from SKILL.md's "What Belongs in a Plan Step" into `references/step-template.md`. Add a "What not to include in Phase 1" sub-section after the existing "What to specify" list in the Phase 1 guidance, containing: (1) Implementation details that go stale — algorithm pseudocode, variable names, internal data structure choices; (2) Generic advice — "write clean code", "follow best practices"; (3) Premature optimization notes — defer unless performance is an acceptance criterion. Then remove the "What Belongs in a Plan Step" section from SKILL.md.

**Then**, remove the other redundant sections. For "Prose Over Code", replace with:

> Plan steps describe intent in prose. Do not include code blocks except for interface signatures, config keys, and schema shapes (full policy in `references/step-template.md`). Tool commands in Phase 3 checklists are operational instructions, not code — they are always permitted.

For "Handling Plan Execution", replace with:

> When the user asks to execute, invoke `blueprint:execute` — it provides full subagent orchestration with batching, git handling, and progress tracking. If unavailable, work through steps one at a time completing all three phases before advancing. Mark completed steps with ✅ in the plan heading and tick Phase 3 checkboxes for cross-session resumability.

For "Handling Ambiguity and Scope Changes", keep only:
- If the request is too vague, ask for specifics. A clear problem statement is a prerequisite.
- If scope changes invalidate more than half the plan, recommend starting fresh.
- If new steps are needed during execution, propose them with the same 3-phase structure.
- If conflicting requirements surface, flag immediately with a recommended resolution.

For the adversarial review section, remove the categories bullet list and replace with a single reference: "The subagent prompt in `references/plan-review-subagent.md` contains the full review methodology covering completeness, dependencies, sizing, criteria quality, phase quality, prose compliance, architecture, and risk."

#### Phase 2 — Adversarial Review

1. Does every removed section have its content preserved in a reference file?
2. Are there any forward references in SKILL.md that now point to removed content?
3. Does the condensed "Handling Plan Execution" still convey the checkmark progress-tracking mechanism?
4. Is the "Prose Over Code" condensed version still clear about what's permitted vs forbidden?
5. Do any remaining sections reference line numbers or headings that no longer exist?

#### Phase 3 — Verification

- [x] "Do not include" items migrated to `references/step-template.md` — verify with `grep "go stale" plugins/blueprint/skills/blueprint/references/step-template.md`.
- [x] All trimmed content has a corresponding source in reference files.
- [x] SKILL.md word count: `wc -w plugins/blueprint/skills/blueprint/SKILL.md` — target under 2,600.
- [x] No broken internal references — `grep -n "What Belongs" plugins/blueprint/skills/blueprint/SKILL.md` returns zero hits.
- [x] No verbatim copying from superpowers:brainstorming.
- [x] Read the full SKILL.md end-to-end — it should flow naturally with no gaps.

---

### ✅ Step 2: Add Hard Gate and Anti-Pattern Section

**Objective**: Add an explicit gate preventing plan generation before clarification is complete and tool chain is confirmed. Add an anti-pattern section addressing the "too simple to plan" failure mode. The gate goes at the top of the Workflow section (before step 1) where the process it controls is defined. The anti-pattern goes after Effort Level and before Design Principles, as a top-level concern.

**Acceptance criteria**:
- A clearly marked gate section exists that prevents proceeding to plan generation until: (a) tool chain is confirmed by user, and (b) critical ambiguities are resolved.
- An anti-pattern section exists addressing "this is too simple to need a plan" with brief reasoning (2-3 sentences).
- Both sections use blueprint's existing voice and terminology — not borrowed phrasing.
- Planning gate is under 50 words. Anti-pattern section is under 80 words. Combined under 130 words.

#### Phase 1 — Build

Add two sections in different locations:

**Anti-pattern: "Too Simple to Plan"** — place after "Effort Level" and before "Design Principles". 2-3 sentences explaining that even small changes benefit from the build-review-verify structure, and that "simple" tasks are where unexamined assumptions cause wasted rework. A plan can be as short as a single step, but it must exist. Acknowledge that the fast-path for small plans (≤2 steps) still applies — the anti-pattern prevents skipping planning entirely, not forcing heavyweight plans for trivial tasks.

**Planning Gate** — place at the top of `## Workflow`, immediately before `### 1. Understand the Task`. Use a `<PLANNING-GATE>` tag with blueprint-specific language: plan generation must not begin until the tool chain is confirmed by the user and critical ambiguities are resolved. This positions the gate right where the workflow it controls is defined.

#### Phase 2 — Adversarial Review

1. Does the gate language actually prevent premature plan generation, or is it advisory?
2. Does the anti-pattern section conflict with the existing "fast-path for small plans" in the adversarial review section?
3. Is the tone consistent with the rest of SKILL.md (instructional, not preachy)?
4. Could an agent misinterpret the gate as requiring excessive clarification for obvious tasks?

#### Phase 3 — Verification

- [x] Gate section exists with `<PLANNING-GATE>` tags.
- [x] Anti-pattern section exists, under 80 words.
- [x] No verbatim copying from superpowers:brainstorming — compare against brainstorming's `<HARD-GATE>` and "Anti-Pattern" sections.
- [x] Fast-path for small plans is still honored (the anti-pattern says plans can be short, not that every plan needs 10 steps).
- [x] SKILL.md reads naturally with the new sections in place.

---

### ✅ Step 3: Add "Propose Approaches" Step to Workflow

**Objective**: Insert a new workflow step between "Understand the Task" and "Assess Complexity" where 2-3 implementation approaches are proposed with trade-offs before committing to a plan structure. Currently the workflow jumps from clarification to complexity assessment with an implicit single approach.

**Acceptance criteria**:
- New section titled "2. Propose Approaches" exists between current sections 1 and 2.
- Current "2. Assess Complexity" renumbered to "3. Assess Complexity".
- Current "3. Generate the Plan" renumbered to "4. Generate the Plan".
- Current "4. Adversarial Plan Review" renumbered to "5. Adversarial Plan Review".
- The new section requires presenting 2-3 approaches with trade-offs and a recommended option.
- The section specifies leading with the recommendation and reasoning.
- For trivially scoped tasks (single obvious approach), a fast-path allows skipping with a brief note.
- Section is under 150 words.

#### Phase 1 — Build

Add a new "### 2. Propose Approaches" section. Content should cover:

- After understanding the task and confirming the tool chain, propose 2-3 different implementation approaches with trade-offs before committing to a plan structure.
- Lead with the recommended approach and explain why. Present alternatives with their strengths and weaknesses.
- The user confirms which approach to pursue before proceeding to complexity assessment and plan generation.
- Fast-path: if the task is narrowly scoped with only one reasonable approach, state the approach briefly and proceed — no need to fabricate alternatives.

Renumber subsequent sections (2→3, 3→4, 4→5).

#### Phase 2 — Adversarial Review

1. Does the new section fit naturally in the workflow flow (after tool chain confirmation, before complexity assessment)?
2. Are the renumbered sections updated consistently throughout SKILL.md (including any cross-references)?
3. Does the fast-path for single-approach tasks prevent unnecessary overhead?
4. Could this step be confused with the clarification rounds in step 1?
5. Is the section voice consistent with the existing workflow sections?

#### Phase 3 — Verification

- [x] New section exists at position 2 in the Workflow.
- [x] All subsequent sections renumbered correctly.
- [x] Any references to "step 2", "step 3", "step 4" elsewhere in SKILL.md updated — check Additional Resources section and `references/plan-review-subagent.md` for stale step numbers.
- [x] Fast-path for trivial tasks is included.
- [x] Section word count under 150.
- [x] No verbatim copying from brainstorming's "Exploring approaches" section.

---

### ✅ Step 4: Add Process Flow Digraph

**Objective**: Add a dot-notation process flow diagram to SKILL.md that makes the full workflow visually scannable. Place it after the Workflow heading and before the first workflow step.

**Acceptance criteria**:
- A `dot` code block exists showing the complete workflow as a digraph.
- All workflow steps are represented as nodes.
- Decision points (fast-path conditions, user confirmations) are diamond-shaped.
- The flow matches the actual workflow after Steps 1-3 modifications.
- The diagram includes the new "Propose Approaches" step and the planning gate.

#### Phase 1 — Build

Create a dot digraph covering the full workflow:

Nodes (boxes): Explore codebase & discover tools → Confirm tool chain → Clarify requirements → Propose 2-3 approaches → User picks approach → Assess complexity → Generate plan → Adversarial review

Decision diamonds: Planning gate ready? (tool chain confirmed + requirements clear), Fast-path approach? (skip alternatives), Fast-path review? (≤2 steps), Review passed?

Terminal state: double-circle for "Plan ready for execution"

Feedback loops: Review issues → fix and re-dispatch, User requests changes → modify and re-review.

Place the diagram right after `## Workflow` heading, before the planning gate (added in Step 2). The reading order should be: `## Workflow` → digraph (visual overview) → `<PLANNING-GATE>` → `### 1. Understand the Task`.

#### Phase 2 — Adversarial Review

1. Does the diagram match the actual workflow after all previous steps' modifications?
2. Are all decision points represented (fast-paths, user confirmations)?
3. Is the diagram readable — not too many nodes crammed together?
4. Does the terminal state correctly lead to execution?
5. Are feedback loops (review → fix → re-review) shown?

#### Phase 3 — Verification

- [x] Dot code block exists in SKILL.md after `## Workflow`.
- [x] All 5 workflow steps represented.
- [x] Decision diamonds for: planning gate, fast-path approach, fast-path review, review verdict.
- [x] Terminal state is double-circle.
- [x] Diagram is original (not copied from brainstorming's digraph).
- [x] Read the dot source and mentally trace the flow — it should match the prose.

---

### ✅ Step 5: Add Visual Companion Integration

**Objective**: Add a brief section to SKILL.md that delegates visual planning aid to superpowers' visual companion when the superpowers plugin is installed. No standalone reference file, no fallback mechanism — if superpowers is not present, visual aid is simply not available.

**Acceptance criteria**:
- No new `references/visual-companion.md` file is created.
- SKILL.md has a brief addition (under 60 words) in the "1. Understand the Task" section noting that if superpowers is installed, its visual companion can be used for architecture diagrams and approach comparisons during planning.
- The addition makes clear this is optional and depends on superpowers being installed.
- No server scripts, HTML templates, or fallback mechanisms are introduced.

#### Phase 1 — Build

Add to SKILL.md in section "1. Understand the Task", after the clarification paragraph:

> **Visual companion (optional):** When planning involves architectural decisions that benefit from diagrams or visual comparison of approaches, the superpowers plugin's visual companion can present these in a browser. This requires the superpowers plugin to be installed — no fallback is provided.

Do NOT create a `references/visual-companion.md` file. Do NOT add an entry to "Additional Resources" for visual companion.

#### Phase 2 — Adversarial Review

1. Does the SKILL.md addition integrate naturally without disrupting the existing flow?
2. Is it clear that this is optional and depends on superpowers?
3. Is the addition under 60 words?
4. Is there any verbatim text from brainstorming's visual-companion.md?

#### Phase 3 — Verification

- [x] No file exists at `plugins/blueprint/skills/blueprint/references/visual-companion.md`.
- [x] SKILL.md references visual companion in the workflow section.
- [x] Addition is under 60 words.
- [x] No verbatim copying from superpowers:brainstorming.
- [x] Read the SKILL.md addition — it should feel native to blueprint's voice.

---

### Step 6: Final Review and Version Bump

**Objective**: Verify the complete SKILL.md reads coherently after all changes, check word count targets, and bump the plugin version.

**Acceptance criteria**:
- SKILL.md word count is between 2,200 and 2,500. Must stay under 3,000 words per skill-development guidelines.
- The document flows naturally from top to bottom with no jarring transitions.
- All internal references (section numbers, file references) are consistent.
- Plugin version bumped from 1.1.0 to 1.2.0 in plugin.json.
- No verbatim phrases from superpowers:brainstorming anywhere in changed files.

#### Phase 1 — Build

1. Read the complete SKILL.md end-to-end. Fix any flow issues, inconsistent numbering, or awkward transitions introduced by the changes.
2. Verify all file references in "Additional Resources" point to files that exist.
3. Verify `plugins/blueprint/.claude-plugin/plugin.json` version is already `1.2.0` (bumped in prior session).
4. Run a final word count check.

#### Phase 2 — Adversarial Review

1. Does the document flow logically from Purpose → Gate → Principles → Workflow → Formats → Execution → Ambiguity → Resources?
2. Are all 5 workflow steps numbered consistently?
3. Do any sections reference removed content or old section numbers?
4. Is the version bump justified by the scope of changes?
5. Has the SKILL.md stayed under 3,000 words (the skill-development guideline)?

#### Phase 3 — Verification

- [ ] Word count: `wc -w plugins/blueprint/skills/blueprint/SKILL.md` — between 2,200 and 2,500.
- [ ] All references in Additional Resources point to existing files.
- [ ] Plugin version is `1.2.0` in `plugins/blueprint/.claude-plugin/plugin.json`.
- [ ] `grep -r "Do NOT invoke" plugins/blueprint/` returns zero hits (no copied gate language).
- [ ] `grep -r "turn ideas into" plugins/blueprint/` returns zero hits (no copied brainstorming phrasing).
- [ ] Full read-through of SKILL.md confirms coherent flow.
