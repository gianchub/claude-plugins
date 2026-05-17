# Step Template — HTML

Use this template when the user has chosen the **HTML** blueprint format (the default). For the Markdown format, use `step-template.md` instead.

Every step in an HTML blueprint follows the same 3-phase structure as the Markdown version (Build, Adversarial Review, Verification). This document defines the HTML scaffold, the required IDs and `data-*` attributes that `execute-blueprint` depends on, and when to use HTML-specific affordances that Markdown cannot express.

---

## Filename Convention

**Single document**: `docs/plans/YYYY-MM-DD-<topic>-plan.html`

Example: `docs/plans/2026-03-15-user-auth-plan.html`

**Milestone folder**: `docs/plans/YYYY-MM-DD-<topic>/`

Contents:

```
docs/plans/2026-03-15-user-auth/
  README.html
  01_data-layer.html
  02_api-endpoints.html
  03_frontend-integration.html
```

The folder's `README.html` lists the milestones with one-sentence descriptions, restates the tool chain, and notes any cross-cutting concerns. It uses the same plan-header scaffold below but with `<section id="milestones">` in place of `<section id="steps">`.

---

## Authoring Rules

1. **Self-contained**: All styling lives in the document — inline `<style>` blocks (anywhere in the document, including inside `<svg>` for SVG-scoped styles) and inline `style=` attributes. All diagrams are inline SVG. No external CSS, no JS libraries, no fonts loaded over the network. The file must render correctly when opened directly from the filesystem with no network access.
2. **Stable scaffold**: The IDs, classes, and `data-*` attributes listed in the "Structural Contract" section below are required and must be spelled exactly as shown. `execute-blueprint` walks these to locate steps, mutate progress state, and tick verification checkboxes.
3. **Content freedom inside sections**: Within a phase section, the LLM decides layout. Build phase content must remain prose-first per the design principle in `SKILL.md` — inline SVG and other HTML affordances are for *structural* expression (dependencies, state, relationships), not for replacing prose with diagrams.
4. **No embellishment for its own sake**: Do not add diagrams, collapsibles, or styling that does not carry information. A two-step linear plan does not need a dependency graph. A one-paragraph build phase does not need to be collapsed.

---

## Plan Document Structure

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Plan: [Feature/Change Title]</title>
<style>
:root {
  --c-pending:  #607d8b;
  --c-active:   #1565c0;
  --c-complete: #2e7d32;
  --c-failed:   #c62828;
  --c-skipped:  #757575;
  --c-muted: #555;
  --bg: #fafafa;
  --fg: #1a1a1a;
  --border: #ddd;
  --code-bg: #f4f4f4;
  --card-bg: #ffffff;
}
* { box-sizing: border-box; }
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", system-ui, sans-serif;
  line-height: 1.55;
  color: var(--fg);
  background: var(--bg);
  max-width: 1000px;
  margin: 0 auto;
  padding: 2rem 1.25rem;
}
header.plan-header h1 { margin: 0 0 0.4rem; }
header.plan-header .plan-meta { color: var(--c-muted); margin: 0.15rem 0; }
h2 { border-bottom: 2px solid var(--border); padding-bottom: 0.3rem; margin-top: 2.25rem; }
h3 { margin: 0; font-size: 1.1rem; }
h4 {
  margin: 0.85rem 0 0.3rem;
  font-size: 0.78rem;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--c-muted);
}
code, pre { font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace; }
pre {
  background: var(--code-bg);
  padding: 0.75rem;
  border-radius: 4px;
  overflow-x: auto;
  font-size: 0.85rem;
}
code:not(pre code) {
  background: var(--code-bg);
  padding: 0.1rem 0.35rem;
  border-radius: 3px;
  font-size: 0.9em;
}
table { border-collapse: collapse; width: 100%; margin: 1rem 0; }
th, td {
  text-align: left;
  padding: 0.5rem 0.75rem;
  border-bottom: 1px solid var(--border);
  vertical-align: top;
}
th { background: rgba(0,0,0,0.04); font-weight: 600; }
.status-badge {
  display: inline-block;
  padding: 0.1rem 0.55rem;
  border-radius: 12px;
  font-size: 0.7rem;
  font-weight: 700;
  color: #fff;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  vertical-align: middle;
}
.step[data-status="pending"]  .status-badge { background: var(--c-pending); }
.step[data-status="active"]   .status-badge { background: var(--c-active); }
.step[data-status="complete"] .status-badge { background: var(--c-complete); }
.step[data-status="failed"]   .status-badge { background: var(--c-failed); }
.step[data-status="skipped"]  .status-badge { background: var(--c-skipped); }
.step {
  border: 1px solid var(--border);
  border-left-width: 4px;
  border-radius: 4px;
  margin: 1rem 0;
  padding: 1rem 1.1rem;
  background: var(--card-bg);
}
.step[data-status="pending"]  { border-left-color: var(--c-pending); }
.step[data-status="active"]   { border-left-color: var(--c-active); }
.step[data-status="complete"] { border-left-color: var(--c-complete); opacity: 0.92; }
.step[data-status="failed"]   { border-left-color: var(--c-failed); }
.step[data-status="skipped"]  { border-left-color: var(--c-skipped); opacity: 0.7; }
.step-header {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  flex-wrap: wrap;
  margin-bottom: 0.25rem;
}
.step-objective { margin: 0.25rem 0 0.75rem; }
.acceptance-criteria ul { margin: 0.25rem 0 0.75rem 1.25rem; padding: 0; }
.phase { margin-top: 1rem; }
.checklist { list-style: none; padding-left: 0; }
.checklist li { margin: 0.25rem 0; }
.checklist input[type="checkbox"] { margin-right: 0.5rem; vertical-align: -0.15em; }
.diagram svg { max-width: 100%; height: auto; display: block; margin: 1rem auto; }
.diagram figcaption { color: var(--c-muted); font-size: 0.85em; text-align: center; }
details { margin: 0.5rem 0; }
details > summary { cursor: pointer; font-weight: 600; }
a.xref { text-decoration: none; border-bottom: 1px dotted currentColor; }
@media print {
  body { background: #fff; max-width: none; padding: 0.5in; }
  details { display: block; }
  details > summary { list-style: none; }
  details > summary::-webkit-details-marker { display: none; }
  .step { break-inside: avoid; box-shadow: none; }
}
</style>
</head>
<body>

<header class="plan-header">
  <h1>Plan: [Feature/Change Title]</h1>
  <p class="plan-meta"><strong>Date:</strong> YYYY-MM-DD</p>
  <p class="plan-meta"><strong>Goal:</strong> 1–2 sentence summary of what this plan achieves.</p>
</header>

<section id="tool-chain">
  <h2>Tool Chain</h2>
  <table>
    <thead>
      <tr><th>Category</th><th>Tool</th><th>Command</th></tr>
    </thead>
    <tbody>
      <tr><td>Test runner</td><td><!-- discovered --></td><td><code><!-- test command --></code></td></tr>
      <tr><td>Linter</td><td><!-- discovered --></td><td><code><!-- lint command --></code></td></tr>
      <tr><td>Type checker</td><td><!-- discovered --></td><td><code><!-- type-check command --></code></td></tr>
      <tr><td>Formatter</td><td><!-- discovered --></td><td><code><!-- format command --></code></td></tr>
    </tbody>
  </table>
</section>

<!-- Optional <section id="dependencies"> with inline SVG — see HTML-Specific Affordances. -->

<section id="steps">
  <h2>Steps</h2>

  <article class="step" data-step="1" data-status="pending" id="step-1">
    <header class="step-header">
      <h3>Step 1: [Step Title]</h3>
      <span class="status-badge">Pending</span>
    </header>

    <p class="step-objective"><strong>Objective:</strong>
      <!-- What this step achieves and why it matters in the broader plan. -->
    </p>

    <section class="acceptance-criteria">
      <h4>Acceptance Criteria</h4>
      <ul>
        <li><!-- Criterion 1 — concrete, testable condition --></li>
        <li><!-- Criterion 2 --></li>
      </ul>
    </section>

    <section class="phase phase-build">
      <h4>Phase 1 — Build</h4>
      <!-- Prose instructions for what to construct. Prose-first per SKILL.md design principle:
           interface signatures, config keys, schema shapes are the only acceptable code blocks. -->
    </section>

    <section class="phase phase-review">
      <h4>Phase 2 — Adversarial Review</h4>
      <ol>
        <li><!-- Step-specific review question targeting a likely failure mode. --></li>
        <li><!-- Question about codebase integration / convention fit. --></li>
        <li><!-- Question about ripple effects outside the step's immediate scope. --></li>
      </ol>
    </section>

    <section class="phase phase-verify">
      <h4>Phase 3 — Verification</h4>
      <ul class="checklist">
        <li><label><input type="checkbox" disabled> All new and modified code has corresponding tests.</label></li>
        <li><label><input type="checkbox" disabled> All tests pass: <code><!-- test command --></code></label></li>
        <li><label><input type="checkbox" disabled> Test coverage adequate for the step's scope.</label></li>
        <li><label><input type="checkbox" disabled> Linter passes: <code><!-- lint command --></code></label></li>
        <li><label><input type="checkbox" disabled> Type checker passes: <code><!-- type-check command --></code></label></li>
        <li><label><input type="checkbox" disabled> Acceptance criteria check:
          <ul class="checklist">
            <li><label><input type="checkbox" disabled> Criterion 1 → verified by &lt;test name&gt;</label></li>
            <li><label><input type="checkbox" disabled> Criterion 2 → verified by &lt;test name&gt;</label></li>
          </ul>
        </label></li>
      </ul>
    </section>
  </article>

  <!-- Additional <article class="step" data-step="N" data-status="pending" id="step-N">…</article> elements. -->
</section>

</body>
</html>
```

---

## Milestone Folder README

The README inside a milestone folder uses the same `<head>` and plan header, then lists milestones:

```html
<header class="plan-header">
  <h1>Plan: [Feature/Change Title]</h1>
  <p class="plan-meta"><strong>Date:</strong> YYYY-MM-DD</p>
  <p class="plan-meta"><strong>Goal:</strong> 1–2 sentence summary.</p>
</header>

<section id="tool-chain">
  <h2>Tool Chain</h2>
  <table>…</table>
</section>

<section id="milestones">
  <h2>Milestones</h2>
  <ol class="milestone-list">
    <li><a href="01_data-layer.html">01 — Data layer</a>: one-sentence description.</li>
    <li><a href="02_api-endpoints.html">02 — API endpoints</a>: one-sentence description.</li>
    <li><a href="03_frontend-integration.html">03 — Frontend integration</a>: one-sentence description.</li>
  </ol>
</section>

<section id="cross-cutting">
  <h2>Cross-cutting concerns</h2>
  <!-- Anything that applies to every milestone (logging conventions, error-handling patterns,
       feature-flag gating, etc.). Omit if there are none. -->
</section>
```

Each milestone HTML file uses the full step-document scaffold but its `<header class="plan-header">` may reference the parent plan in addition to the milestone's own title.

---

## Structural Contract

These elements, IDs, classes, and `data-*` attributes are required. `execute-blueprint` depends on them being present and spelled exactly as shown.

| Element | Required attribute(s) | Purpose |
|---|---|---|
| `<header class="plan-header">` | — | Plan title, date, goal. |
| `<section id="tool-chain">` | — | Tool chain table. |
| `<section id="steps">` (or `id="milestones"` in folder README) | — | Container for step articles. |
| `<article class="step">` | `data-step="N"`, `data-status="pending"|"active"|"complete"|"failed"|"skipped"`, `id="step-N"` | One per step. `data-step` is the integer step number; `id` is `step-{N}`. |
| `<header class="step-header">` | — | Holds step title `<h3>` and a `<span class="status-badge">`. |
| `<p class="step-objective">` | — | Objective prose. |
| `<section class="acceptance-criteria">` | — | Acceptance criteria list. |
| `<section class="phase phase-build">` | — | Build phase content. |
| `<section class="phase phase-review">` | — | Adversarial review content. |
| `<section class="phase phase-verify">` | — | Verification checklist. |
| `<ul class="checklist">` inside `phase-verify` | — | Holds `<li>` elements with `<input type="checkbox" disabled>`. |

### Initial Step State

When the plan is first written, every step is in the initial state:

- `data-status="pending"` on the `<article>`.
- `<span class="status-badge">Pending</span>` in the step header.
- All `<input type="checkbox" disabled>` elements **unchecked**.
- No `✅ ` prefix on the step `<h3>`.

### Completion State (mutated by `execute-blueprint`)

When a step passes all three phases, `execute-blueprint` performs these mutations:

1. Change the `<article>` attribute to `data-status="complete"`.
2. Replace the status badge text from `Pending` to `Complete` (the badge color updates automatically via CSS).
3. Prepend `✅ ` to the inner text of the step's `<h3>` (e.g., `<h3>Step 2: Auth middleware</h3>` → `<h3>✅ Step 2: Auth middleware</h3>`).
4. Add the `checked` attribute to every `<input type="checkbox">` inside that step's `<section class="phase phase-verify">`.

### Status Mapping

The full mapping between `data-status`, badge text, and the `✅ ` prefix:

| `data-status` | Badge text  | `✅ ` prefix on `<h3>`? | When used |
|---|---|---|---|
| `pending`  | `Pending`  | no  | Initial state, before execution begins. |
| `active`   | `Active`   | no  | Step currently being executed (optional intermediate state; many runs go straight from `pending` to a terminal state). |
| `complete` | `Complete` | yes | Step passed all three phases (build, review, verify). |
| `failed`   | `Failed`   | no  | Step blocked on a review finding or verification failure that was not resolved before the run ended. |
| `skipped`  | `Skipped`  | no  | Step deliberately skipped (e.g., user requested "execute steps 3-5", or step abandoned after a blocking finding). |

`data-status` and the badge text are the canonical sources of progress state; the `✅ ` prefix is the human-visible signal for completion specifically. Do not prepend `✅ ` for any terminal state other than `complete`.

---

## HTML-Specific Affordances — When to Use Them

These features exist because Markdown cannot express them. Apply when they materially improve the plan; skip when they do not.

### Inline SVG Dependency Graph

Use when the plan has non-linear step dependencies that prose cannot capture cleanly:

- A milestone has parallel work streams that converge later.
- Multiple steps depend on the same earlier artifact and a reader needs to see the fan-out.
- A refactor plan has interleaved "extract" / "swap caller" pairs whose ordering matters.

Skip for purely linear plans (Step 1 → 2 → 3 → …). The standard ordering already conveys the dependency.

Place inside `<section id="dependencies">` immediately after `<section id="tool-chain">`:

```html
<section id="dependencies">
  <h2>Step Dependencies</h2>
  <figure class="diagram" id="diagram-step-deps">
    <svg viewBox="0 0 700 220" xmlns="http://www.w3.org/2000/svg" role="img"
         aria-label="Step dependency graph">
      <!-- nodes labeled with step numbers; edges show "depends on" -->
    </svg>
    <figcaption>Steps 4 and 5 both depend on Step 2 (repository interface).</figcaption>
  </figure>
</section>
```

### Inline SVG for Architectural Decisions

Use a small inline SVG inside a single step's Build phase when the architecture being built is a key deliverable and prose would be ambiguous:

- A state machine the step introduces.
- A new layer's relationship to existing layers.
- A queue/topic topology the step wires up.

Keep the diagram inside the relevant phase section. Do not duplicate prose with a diagram; use the diagram to express what prose handles poorly (multi-edge relationships, parallel paths, state transitions).

### Collapsible Sections

Use `<details>` for content that is contextually important but not central:

- Long verification sub-checklists per acceptance criterion when the parent step has many criteria.
- Optional follow-up notes ("Future work" suggestions) that should not distract from the step's required work.
- Verbatim snippets of large existing code that the step extends, when the snippet is reference material rather than instruction.

Do not collapse the Build, Review, or Verify phase bodies themselves. Those are the step's primary content.

### Cross-Reference Anchors

Each `<article class="step">` has `id="step-N"`. Use anchor links when a later step's Objective or Build phase refers to artifacts from an earlier step:

```html
<p class="step-objective"><strong>Objective:</strong> Add the export subcommand.
Depends on <a class="xref" href="#step-1">Step 1 (query engine)</a>; uses the
<code>QueryEngine</code> interface defined there.</p>
```

### Tool Chain Badges

The tool chain table is already structured. Use the same `<code>` styling for any inline reference to a tool command elsewhere in the document. Do not introduce new colored badges for tool categories — the table is the single source of truth.

---

## Build Phase: Prose-First Still Applies

The HTML format does not relax the prose-first rule for Phase 1. The acceptable code-block exceptions from `step-template.md` (interface signatures, exact config keys, schema shapes) still apply, and they are written using `<pre><code>…</code></pre>` in HTML. Everything else stays in `<p>`, `<ul>`, `<ol>`.

If a Build phase feels like it needs a code block to be clear, the step is still too large — split it.

---

## Example: Complete Step (HTML)

```html
<article class="step" data-step="2" data-status="pending" id="step-2">
  <header class="step-header">
    <h3>Step 2: Add CSV Export Command</h3>
    <span class="status-badge">Pending</span>
  </header>

  <p class="step-objective"><strong>Objective:</strong> Add an <code>export</code>
    subcommand that writes query results to a CSV file. Depends on
    <a class="xref" href="#step-1">Step 1 (query engine)</a>; uses the
    <code>QueryEngine</code> interface defined there.
  </p>

  <section class="acceptance-criteria">
    <h4>Acceptance Criteria</h4>
    <ul>
      <li><code>app export --query "..." --output path.csv</code> writes results as CSV
        with a header row.</li>
      <li>Column order matches the query's field order.</li>
      <li>If the output file already exists, abort with an error unless
        <code>--overwrite</code> is passed.</li>
      <li>Empty result sets produce a file with only the header row.</li>
      <li>Non-zero exit code and a descriptive message on any failure
        (bad query, I/O error, permission denied).</li>
    </ul>
  </section>

  <section class="phase phase-build">
    <h4>Phase 1 — Build</h4>
    <p><strong>CLI wiring:</strong> Register a new <code>export</code> subcommand with
      three arguments: <code>--query</code> (required), <code>--output</code> (required),
      and <code>--overwrite</code> (optional flag, defaults to false). Follow the existing
      subcommand registration pattern established in Step 1.</p>
    <p><strong>Export logic:</strong> Create an export module responsible for executing
      the query via the QueryEngine, transforming results into CSV rows, and writing to
      the target path. Separate CSV formatting from file I/O so each can be tested
      independently. Check for file existence before writing and raise a clear error if
      the file exists and <code>--overwrite</code> is not set.</p>
    <p><strong>Error handling:</strong> Catch query errors, I/O errors, and permission
      errors. Map each to a descriptive user-facing message and a non-zero exit code.
      Do not expose stack traces or internal paths in error output.</p>
    <p><strong>Tests to write:</strong> valid query → correct CSV; column order matches
      field order; empty result → header-only file; existing file without
      <code>--overwrite</code> → error; <code>--overwrite</code> replaces file; invalid
      query → non-zero exit + message; unwritable path → non-zero exit + message.</p>
  </section>

  <section class="phase phase-review">
    <h4>Phase 2 — Adversarial Review</h4>
    <ol>
      <li>Does the export module use the <code>QueryEngine</code> interface from Step 1,
        or does it bypass it?</li>
      <li>Is file existence checked before any write attempt, or could a partial write
        corrupt an existing file?</li>
      <li>Are CSV fields properly escaped (values containing commas, quotes, newlines)?</li>
      <li>Do error messages avoid exposing internal paths or implementation details?</li>
      <li>Do tests verify actual file contents (read back and compare), not just exit codes?</li>
      <li>Does the new command follow the same patterns as existing subcommands
        (argument parsing style, error reporting, exit codes)?</li>
    </ol>
  </section>

  <section class="phase phase-verify">
    <h4>Phase 3 — Verification</h4>
    <ul class="checklist">
      <li><label><input type="checkbox" disabled> All new and modified code has corresponding tests.</label></li>
      <li><label><input type="checkbox" disabled> All tests pass: <code>pytest</code></label></li>
      <li><label><input type="checkbox" disabled> Test coverage adequate — no untested branches in new code.</label></li>
      <li><label><input type="checkbox" disabled> Linter passes: <code>ruff check .</code></label></li>
      <li><label><input type="checkbox" disabled> Type checker passes: <code>mypy .</code></label></li>
      <li><label><input type="checkbox" disabled> Acceptance criteria:
        <ul class="checklist">
          <li><label><input type="checkbox" disabled> <code>export --query --output</code> produces valid CSV → <code>test_valid_export</code></label></li>
          <li><label><input type="checkbox" disabled> Column order matches field order → <code>test_column_order</code></label></li>
          <li><label><input type="checkbox" disabled> Empty result → header-only file → <code>test_empty_results</code></label></li>
          <li><label><input type="checkbox" disabled> Existing file without <code>--overwrite</code> → error → <code>test_no_overwrite</code></label></li>
          <li><label><input type="checkbox" disabled> <code>--overwrite</code> replaces file → <code>test_overwrite</code></label></li>
          <li><label><input type="checkbox" disabled> Bad query → non-zero exit + message → <code>test_invalid_query</code></label></li>
          <li><label><input type="checkbox" disabled> Permission denied → non-zero exit + message → <code>test_unwritable_path</code></label></li>
        </ul>
      </label></li>
    </ul>
  </section>
</article>
```
