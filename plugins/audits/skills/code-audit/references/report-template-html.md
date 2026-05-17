# Audit Report Template — HTML

Use this template when the user has chosen the **HTML** report format. For the Markdown format, use `report-template.md` instead.

## Filename Convention

Save the report at the project root as:

```
AUDIT-REPORT-YYYY-MM-DD.html
```

If a file with that name already exists, append an incrementing numeric suffix:

```
AUDIT-REPORT-YYYY-MM-DD-2.html
AUDIT-REPORT-YYYY-MM-DD-3.html
```

Use the current date at the time of report generation. When searching for prior reports during a re-audit, scan for both `AUDIT-REPORT-*.html` and `AUDIT-REPORT-*.md` and pick the most recent by file modification time.

---

## Authoring Rules

The HTML format is not a place to wrap Markdown content in tags. Each HTML-specific affordance below has a defined purpose; use it when it adds information the Markdown version cannot convey, and omit it otherwise.

1. **Self-contained**: All styling lives in the document — inline `<style>` blocks (anywhere in the document, including inside `<svg>` for SVG-scoped styles) and inline `style=` attributes. All diagrams are inline SVG. No external CSS, no JS libraries, no fonts loaded over the network. The file must render correctly when opened directly from the filesystem with no network access.
2. **Stable scaffold**: The IDs, classes, and `data-*` attributes listed in the "Structural Contract" section below are required and must be spelled exactly as shown. Re-audit comparison and any future tooling depend on them.
3. **Content freedom inside sections**: Within a section, the LLM decides layout, nesting, and which optional affordances (collapsibles, inline SVG, cross-reference anchors) to use.
4. **No embellishment for its own sake**: Do not add inline SVG, collapsibles, or color flourishes that do not carry information. A finding with three sentences of description does not need to be wrapped in `<details>`. A scope of one file does not need a dependency diagram.

---

## Document Structure

The complete scaffold below is the required structure. Replace placeholder text and counts; keep IDs, classes, and `data-*` attributes verbatim.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Audit Report — YYYY-MM-DD</title>
<style>
:root {
  --c-critical: #c62828;
  --c-high: #ef6c00;
  --c-medium: #f9a825;
  --c-low: #1565c0;
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
header.report-header h1 { margin: 0 0 0.25rem; }
header.report-header .report-date { color: var(--c-muted); }
h2 { border-bottom: 2px solid var(--border); padding-bottom: 0.3rem; margin-top: 2.25rem; }
h3 { margin: 0; font-size: 1.05rem; }
h4 {
  margin: 0.85rem 0 0.25rem;
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
  line-height: 1.45;
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
.severity-badge {
  display: inline-block;
  padding: 0.1rem 0.55rem;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 700;
  color: #fff;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  vertical-align: middle;
}
.severity-critical { background: var(--c-critical); }
.severity-high     { background: var(--c-high); }
.severity-medium   { background: var(--c-medium); color: #2a2a2a; }
.severity-low      { background: var(--c-low); }
.finding {
  border: 1px solid var(--border);
  border-left-width: 4px;
  border-radius: 4px;
  margin: 1rem 0;
  padding: 1rem 1.1rem;
  background: var(--card-bg);
}
.finding[data-severity="critical"] { border-left-color: var(--c-critical); }
.finding[data-severity="high"]     { border-left-color: var(--c-high); }
.finding[data-severity="medium"]   { border-left-color: var(--c-medium); }
.finding[data-severity="low"]      { border-left-color: var(--c-low); }
.finding-header {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  flex-wrap: wrap;
  margin-bottom: 0.4rem;
}
.finding-id { font-family: ui-monospace, Menlo, monospace; color: var(--c-muted); font-size: 0.9em; }
.finding-meta { margin: 0.4rem 0 0.6rem; font-size: 0.9em; }
.finding-meta dt {
  display: inline-block;
  font-weight: 600;
  color: var(--c-muted);
  margin-right: 0.35rem;
}
.finding-meta dd { display: inline; margin: 0 1rem 0 0; }
.diagram svg { max-width: 100%; height: auto; display: block; margin: 1rem auto; }
.diagram figcaption { color: var(--c-muted); font-size: 0.85em; text-align: center; }
details { margin: 0.5rem 0; }
details > summary { cursor: pointer; font-weight: 600; }
details[open] > summary { margin-bottom: 0.4rem; }
.intent-theme + .intent-theme { margin-top: 0.4rem; }
a.xref { text-decoration: none; border-bottom: 1px dotted currentColor; }
@media print {
  body { background: #fff; max-width: none; padding: 0.5in; }
  details { display: block; }
  details > summary { list-style: none; }
  details > summary::-webkit-details-marker { display: none; }
  .finding { break-inside: avoid; box-shadow: none; }
  h2 { break-before: page; }
  h2:first-of-type { break-before: auto; }
}
</style>
</head>
<body>

<header class="report-header">
  <h1>Audit Report</h1>
  <p class="report-date">YYYY-MM-DD</p>
</header>

<section id="summary">
  <h2>Summary</h2>
  <table>
    <tr><th>Scope</th><td><!-- files, directories, or modules audited --></td></tr>
    <tr><th>Categories</th><td><!-- comma-separated list of audit categories applied --></td></tr>
    <tr><th>Total findings</th><td>N</td></tr>
    <tr><th>Critical</th><td><span class="severity-badge severity-critical">N</span></td></tr>
    <tr><th>High</th><td><span class="severity-badge severity-high">N</span></td></tr>
    <tr><th>Medium</th><td><span class="severity-badge severity-medium">N</span></td></tr>
    <tr><th>Low</th><td><span class="severity-badge severity-low">N</span></td></tr>
  </table>
</section>

<section id="context-intent">
  <h2>Context &amp; Intent</h2>
  <!--
    Insert the Intent Brief here. Group by theme. When themes contain more than a few
    entries, wrap each theme in <details class="intent-theme"> to keep the report scannable.
    If no intent signals were discovered:
    <p>No documented intent signals were identified in the codebase.</p>
  -->
</section>

<section id="findings-critical" class="findings" data-severity="critical">
  <h2>Critical</h2>

  <article class="finding" data-severity="critical" id="AUDIT-001">
    <header class="finding-header">
      <span class="finding-id">AUDIT-001</span>
      <span class="severity-badge severity-critical">Critical</span>
      <h3>Short title</h3>
    </header>
    <dl class="finding-meta">
      <dt>Category</dt><dd>category name</dd>
      <dt>Location</dt><dd><code>file:line</code></dd>
    </dl>
    <section class="finding-description">
      <h4>Description</h4>
      <p><!-- Clear explanation of the finding. State what the code does and why it is problematic. --></p>
    </section>
    <section class="finding-impact">
      <h4>Impact</h4>
      <p><!-- Concrete consequences. --></p>
    </section>
    <section class="finding-recommendation">
      <h4>Recommendation</h4>
      <p><!-- Specific, actionable fix. Include <pre><code>...</code></pre> when it clarifies. --></p>
    </section>
  </article>

</section>

<section id="findings-high" class="findings" data-severity="high">
  <h2>High</h2>
  <!-- <article class="finding" data-severity="high" id="AUDIT-NNN">…</article> -->
</section>

<section id="findings-medium" class="findings" data-severity="medium">
  <h2>Medium</h2>
  <!-- <article class="finding" data-severity="medium" id="AUDIT-NNN">…</article> -->
</section>

<section id="findings-low" class="findings" data-severity="low">
  <h2>Low</h2>
  <!-- <article class="finding" data-severity="low" id="AUDIT-NNN">…</article> -->
</section>

</body>
</html>
```

---

## Structural Contract

These elements, IDs, classes, and attributes are required. Re-audit comparison and any tooling that walks the report depend on them.

| Element | Required attribute(s) | Purpose |
|---|---|---|
| `<header class="report-header">` | — | Report title and date. |
| `<section id="summary">` | — | Summary table with scope, categories, finding counts. |
| `<section id="context-intent">` | — | Intent Brief. |
| `<section id="findings-{severity}">` | `data-severity` ∈ `critical`, `high`, `medium`, `low` | Container for findings of that severity. Always include all four sections, even when empty. |
| `<article class="finding">` | `data-severity`, `id="AUDIT-NNN"` | One per finding. `id` matches the identifier exactly (no prefix mangling). |
| `<header class="finding-header">` inside finding | — | Holds id, badge, title. |
| `<dl class="finding-meta">` | — | Category, Location, and any additional meta in `<dt>`/`<dd>` pairs. |
| `<section class="finding-description">` | — | Description content. |
| `<section class="finding-impact">` | — | Impact content. |
| `<section class="finding-recommendation">` | — | Recommendation content. |

If a severity has no findings, keep the `<section>` with its `<h2>` and write `<p>No findings at this severity level.</p>` inside.

---

## HTML-Specific Affordances — When to Use Them

These features exist because Markdown cannot express them. Apply when they materially improve the report; skip when they do not.

### Inline SVG Diagrams

Use when summarizing relationships that prose handles poorly:

- A data flow from entry points through transformation layers to terminal operations, when the audit identified missing checks along that path.
- A call-graph fragment when a finding spans many files and the cross-file relationships are central to understanding the issue.
- A module dependency diagram when the report includes findings about circular dependencies or layering violations.

Do not use SVG to render simple lists, severity counts (already covered by the summary table and badges), or single-pair relationships.

When using SVG, embed it inline inside `<figure class="diagram">` with a `<figcaption>`. Keep the SVG small (under ~500 lines) and free of external references.

```html
<figure class="diagram" id="diagram-dataflow">
  <svg viewBox="0 0 800 200" xmlns="http://www.w3.org/2000/svg" role="img"
       aria-label="Data flow from HTTP handlers to persistence layer">
    <!-- nodes and edges -->
  </svg>
  <figcaption>Data flow showing missing validation between layers (see AUDIT-003).</figcaption>
</figure>
```

### Collapsible Sections

Use `<details>` for content that is contextually important but not central:

- Each theme of the Intent Brief, when there are many entries.
- The full enumeration of affected locations for a multi-location finding when the list exceeds ~5 entries.
- Long verbatim code snippets within a Recommendation when the surrounding prose already conveys the fix.

Do not collapse the body of findings themselves. The Description / Impact / Recommendation triplet is the report's core content.

### Cross-Reference Anchors

Each `<article class="finding">` has an `id` matching its `AUDIT-NNN` identifier. Use anchor links when one finding references another:

```html
<p>Same root cause as <a class="xref" href="#AUDIT-007">AUDIT-007</a>; both stem from the
shared validator at <code>src/util/validate.py:42</code>.</p>
```

### Severity Badges

Already in the template scaffold. Use them anywhere severity is referenced inline (summary table, finding headers, cross-references). They are CSS-only; no JS needed.

---

## Identifier Numbering

Same as the Markdown template: assign `AUDIT-NNN` identifiers sequentially starting at `AUDIT-001`, ordered by severity (Critical first, then High, Medium, Low). Within the same severity, order by category, then by file path.

The identifier must appear as the `id` attribute on the `<article>` element exactly (no `audit-` prefix, no lowercase variant).

---

## Multi-Location Findings

When a finding affects multiple locations, list them in the `<dd>` for Location. For up to ~5 locations, list inline:

```html
<dt>Location</dt>
<dd>
  <code>src/api/users.py:42</code>,
  <code>src/api/orders.py:87</code>,
  <code>src/api/products.py:31</code>
</dd>
```

For more than ~5 locations, show the primary in the meta block and the full set in a collapsible inside the Description:

```html
<dl class="finding-meta">
  <dt>Location</dt><dd><code>src/api/users.py:42</code> (primary)</dd>
</dl>
<section class="finding-description">
  <h4>Description</h4>
  <p>…</p>
  <details>
    <summary>All affected locations (12)</summary>
    <ul>
      <li><code>src/api/users.py:42</code></li>
      <li><code>src/api/orders.py:87</code></li>
      <!-- … -->
    </ul>
  </details>
</section>
```

---

## Zero-Findings Report

When no issues are found, produce the full document with all counts set to 0 and each severity section containing a single paragraph:

```html
<section id="findings-critical" class="findings" data-severity="critical">
  <h2>Critical</h2>
  <p>No findings at this severity level.</p>
</section>
```

If no intent signals were discovered, `#context-intent` contains:

```html
<p>No documented intent signals were identified in the codebase.</p>
```
