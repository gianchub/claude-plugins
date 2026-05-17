# Security Audit Report Template — HTML

Use this template when the user has chosen the **HTML** report format. For the Markdown format, use `report-template.md` instead.

## Filename Convention

Save the report at the project root as:

```
SECURITY-AUDIT-REPORT-YYYY-MM-DD.html
```

If a file with that name already exists, append an incrementing numeric suffix:

```
SECURITY-AUDIT-REPORT-YYYY-MM-DD-2.html
SECURITY-AUDIT-REPORT-YYYY-MM-DD-3.html
```

Use the current date at the time of report generation. When searching for prior reports during a re-audit, scan for both `SECURITY-AUDIT-REPORT-*.html` and `SECURITY-AUDIT-REPORT-*.md` and pick the most recent by file modification time.

---

## Authoring Rules

The HTML format is not a place to wrap Markdown content in tags. Each HTML-specific affordance below has a defined purpose; use it when it adds information the Markdown version cannot convey, and omit it otherwise.

1. **Self-contained**: All styling lives in the document — inline `<style>` blocks (anywhere in the document, including inside `<svg>` for SVG-scoped styles) and inline `style=` attributes. All diagrams are inline SVG. No external CSS, no JS libraries, no fonts loaded over the network. The file must render correctly when opened directly from the filesystem with no network access.
2. **Stable scaffold**: The IDs, classes, and `data-*` attributes listed in the "Structural Contract" section below are required and must be spelled exactly as shown. Re-audit comparison and any future tooling depend on them.
3. **Content freedom inside sections**: Within a section, the LLM decides layout, nesting, and which optional affordances (collapsibles, inline SVG, cross-reference anchors) to use.
4. **No embellishment for its own sake**: Do not add inline SVG, collapsibles, or color flourishes that do not carry information. An exploit scenario with three sentences does not need to be wrapped in `<details>`.

---

## Document Structure

The complete scaffold below is the required structure. Replace placeholder text and counts; keep IDs, classes, and `data-*` attributes verbatim.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Security Audit Report — YYYY-MM-DD</title>
<style>
:root {
  --c-critical: #b71c1c;
  --c-high: #e65100;
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
.tag {
  display: inline-block;
  padding: 0.05rem 0.45rem;
  border-radius: 4px;
  background: rgba(0,0,0,0.06);
  font-size: 0.8em;
  font-family: ui-monospace, Menlo, monospace;
}
.tag-cwe { background: rgba(21, 101, 192, 0.12); color: #0d3b66; }
.tag-owasp { background: rgba(230, 81, 0, 0.12); color: #8a3a00; }
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
.finding-tags {
  display: flex;
  gap: 0.35rem;
  flex-wrap: wrap;
  margin: 0.35rem 0 0.5rem;
}
.finding-meta { margin: 0.4rem 0 0.6rem; font-size: 0.9em; }
.finding-meta dt {
  display: inline-block;
  font-weight: 600;
  color: var(--c-muted);
  margin-right: 0.35rem;
}
.finding-meta dd { display: inline; margin: 0 1rem 0 0; }
.exploit-not-confirmed { color: #8a3a00; font-weight: 600; }
.diagram svg { max-width: 100%; height: auto; display: block; margin: 1rem auto; }
.diagram figcaption { color: var(--c-muted); font-size: 0.85em; text-align: center; }
details { margin: 0.5rem 0; }
details > summary { cursor: pointer; font-weight: 600; }
details[open] > summary { margin-bottom: 0.4rem; }
.source-sink-list { font-size: 0.92em; }
.source-sink-list li { margin: 0.2rem 0; }
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
  <h1>Security Audit Report</h1>
  <p class="report-date">YYYY-MM-DD</p>
</header>

<section id="summary">
  <h2>Summary</h2>
  <table>
    <tr><th>Scope</th><td><!-- files, directories, services audited --></td></tr>
    <tr><th>Application kind</th><td><!-- from Threat Model Brief --></td></tr>
    <tr><th>Exposure</th><td><!-- from Threat Model Brief --></td></tr>
    <tr><th>Domains audited</th><td><!-- comma-separated checklist domains --></td></tr>
    <tr><th>Languages analyzed</th><td><!-- comma-separated languages --></td></tr>
    <tr><th>Total findings</th><td>N</td></tr>
    <tr><th>Critical</th><td><span class="severity-badge severity-critical">N</span></td></tr>
    <tr><th>High</th><td><span class="severity-badge severity-high">N</span> (of which M Exploitability Not Confirmed)</td></tr>
    <tr><th>Medium</th><td><span class="severity-badge severity-medium">N</span></td></tr>
    <tr><th>Low</th><td><span class="severity-badge severity-low">N</span></td></tr>
  </table>
</section>

<section id="threat-model">
  <h2>Threat Model</h2>
  <!--
    Insert the Threat Model Brief. Include at minimum: application kind, exposure,
    sensitive data classes, authentication model, authorization model, severity-modifier notes.
    Consider an inline SVG showing trust boundaries when the application has multiple services
    or non-trivial trust transitions.
  -->
</section>

<section id="documented-security-posture">
  <h2>Documented Security Posture</h2>
  <!--
    Insert the Security Intent Brief from Phase 3. Group by theme; use <details> per theme
    when there are many entries. If empty:
    <p>No documented security intent was identified in the codebase.
    All findings are evaluated without intent-based adjustments.</p>
  -->
</section>

<section id="source-sink-map-summary">
  <h2>Source/Sink Map</h2>
  <!--
    One-paragraph summary of the Phase 4 enumeration. State: total sources enumerated,
    total sinks enumerated, services covered, any sinks marked unreachable from any in-scope
    source. The full enumeration goes in Appendix A.
  -->
</section>

<section id="findings-critical" class="findings" data-severity="critical">
  <h2>Critical</h2>

  <article class="finding" data-severity="critical" id="SEC-001">
    <header class="finding-header">
      <span class="finding-id">SEC-001</span>
      <span class="severity-badge severity-critical">Critical</span>
      <h3>Short title</h3>
    </header>
    <p class="finding-tags">
      <span class="tag tag-cwe">CWE-NNN</span>
      <span class="tag tag-owasp">A0N:2021</span>
      <!-- Optional <span class="tag tag-owasp">API0N:2023</span> -->
    </p>
    <dl class="finding-meta">
      <dt>Domain</dt><dd>domain category</dd>
      <dt>Location</dt><dd><code>file:line</code></dd>
      <dt>Severity</dt>
      <dd>Critical (Impact: Severe, Exploitability: Easy, Exposure: Public)</dd>
      <!-- Optional <dt>Modifier</dt><dd>name and source</dd> -->
    </dl>
    <section class="finding-description">
      <h4>Description</h4>
      <p><!-- What the code does, why it is unsafe, and the dataflow path from source to sink. --></p>
    </section>
    <section class="finding-impact">
      <h4>Impact</h4>
      <p><!-- Concrete consequences for this application's data classes. --></p>
    </section>
    <section class="finding-exploit-scenario">
      <h4>Exploit Scenario</h4>
      <p><!-- starting position, action, path, outcome. -->
      <!-- For Not Confirmed cases, use:
        <h4>Exploit Scenario <span class="exploit-not-confirmed">— Not Confirmed</span></h4>
        and explain what is missing to confirm. -->
      </p>
    </section>
    <section class="finding-recommendation">
      <h4>Recommendation</h4>
      <p><!-- Specific, actionable fix. Include defense-in-depth options where relevant. --></p>
    </section>
  </article>

</section>

<section id="findings-high" class="findings" data-severity="high">
  <h2>High</h2>
  <!-- Same structure as Critical; include <h4>Exploit Scenario</h4> (or Not Confirmed variant). -->
</section>

<section id="findings-medium" class="findings" data-severity="medium">
  <h2>Medium</h2>
  <!-- Same structure; Exploit Scenario optional (omit when it does not clarify impact). -->
</section>

<section id="findings-low" class="findings" data-severity="low">
  <h2>Low</h2>
  <!-- Same structure; Exploit Scenario typically omitted. -->
</section>

<section id="appendix-source-sink-map" class="appendix">
  <h2>Appendix A — Source/Sink Map</h2>
  <h3>Sources</h3>
  <ul class="source-sink-list">
    <!-- <li><code>file:line</code> — source class (input shape): reachable sinks</li> -->
  </ul>
  <h3>Sinks</h3>
  <ul class="source-sink-list">
    <!-- <li><code>file:line</code> — sink class: reachable sources</li> -->
  </ul>
  <!-- For monorepos, group by service with <section class="service">. -->
</section>

<section id="appendix-coverage" class="appendix">
  <h2>Appendix B — Coverage and Limitations</h2>
  <!--
    State explicitly:
    - Which files / paths were in scope and read in full.
    - Which files / paths were excluded (generated, vendored, lock files) and why.
    - Any analysis the auditor could not perform and why.
    - Any High/Critical finding with "Exploitability Not Confirmed" and what would confirm it.
    - Any threat-model assumption made due to ambiguity.
  -->
</section>

<section id="appendix-reaudit-delta" class="appendix">
  <!--
    Include only if a prior SECURITY-AUDIT-REPORT-* was found in Phase 1.
    Omit the entire <section> when no prior report exists.
  -->
  <h2>Appendix C — Re-Audit Delta</h2>
  <h3>Resolved since prior report</h3>
  <ul><!-- <li>SEC-NNN-prior (title) — fix verified at file:line</li> --></ul>
  <h3>Persistent from prior report</h3>
  <ul><!-- <li>SEC-NNN-prior (title) — still present; reassigned SEC-NNN-current</li> --></ul>
  <h3>New in this report</h3>
  <p>All findings not in the prior report.</p>
</section>

</body>
</html>
```

---

## Structural Contract

These elements, IDs, classes, and attributes are required.

| Element | Required attribute(s) | Purpose |
|---|---|---|
| `<header class="report-header">` | — | Report title and date. |
| `<section id="summary">` | — | Summary table. |
| `<section id="threat-model">` | — | Threat Model Brief output (Phase 2). |
| `<section id="documented-security-posture">` | — | Security Intent Brief (Phase 3). |
| `<section id="source-sink-map-summary">` | — | One-paragraph summary of Phase 4. |
| `<section id="findings-{severity}">` | `data-severity` ∈ `critical`, `high`, `medium`, `low` | One per severity; always present even when empty. |
| `<article class="finding">` | `data-severity`, `id="SEC-NNN"` | One per finding. `id` matches identifier exactly. |
| `<header class="finding-header">` | — | Id, badge, title. |
| `<p class="finding-tags">` | — | Holds CWE / OWASP tags via `<span class="tag tag-cwe">` and `<span class="tag tag-owasp">`. |
| `<dl class="finding-meta">` | — | Domain, Location, Severity, optional Modifier. |
| `<section class="finding-description">` | — | Description. |
| `<section class="finding-impact">` | — | Impact. |
| `<section class="finding-exploit-scenario">` | — | Required for Critical and High. Use the `<span class="exploit-not-confirmed">— Not Confirmed</span>` variant when applicable. Optional for Medium, typically omitted for Low. |
| `<section class="finding-recommendation">` | — | Recommendation. |
| `<section id="appendix-source-sink-map">` | `class="appendix"` | Full Phase 4 enumeration. |
| `<section id="appendix-coverage">` | `class="appendix"` | Coverage and limitations. |
| `<section id="appendix-reaudit-delta">` | `class="appendix"` | Only present when a prior report existed. |

If a severity has no findings, keep the `<section>` with its `<h2>` and write `<p>No findings at this severity level.</p>` inside.

---

## Severity Field Format

Always include the three component scores in the Severity `<dd>`:

```
<level> (Impact: <Severe|High|Moderate|Low>, Exploitability: <Trivial|Easy|Multistep|Hard|Not Confirmed>, Exposure: <Public|Authenticated public|Authenticated restricted|Internal|Local>)
```

Note that Exploitability uses **Multistep** (not "Moderate") to keep its scale visually distinct from Impact's Moderate level.

When a Threat-Model modifier was applied, add a `<dt>Modifier</dt><dd>name and source</dd>` pair in the meta block.

---

## HTML-Specific Affordances — When to Use Them

These features exist because Markdown cannot express them. Apply when they materially improve the report; skip when they do not.

### Inline SVG Diagrams

The security report has more natural use cases for diagrams than the code audit:

- **Trust-boundary diagram** in the Threat Model section when the application has multiple services, multi-tenant boundaries, or non-trivial trust transitions (anonymous → authenticated → admin).
- **Source-to-sink dataflow** for a specific High/Critical injection-class finding when the path spans many files and the prose alone makes the chain hard to follow.
- **Authorization decision graph** when a finding describes a missing or inconsistent check that is best understood by seeing the protected operations and the checks that gate them.

Skip SVG when the relationship is two-node (just say so in prose), when severity counts are already covered by the summary badges, or when the diagram would simply restate the dataflow already described in the Description text.

Embed inside `<figure class="diagram">` with `<figcaption>` describing what the diagram shows and which finding(s) it relates to. Use `role="img"` and `aria-label` on the `<svg>`.

```html
<figure class="diagram" id="diagram-auth-flow">
  <svg viewBox="0 0 800 220" xmlns="http://www.w3.org/2000/svg" role="img"
       aria-label="Authentication and authorization flow through the request lifecycle">
    <!-- nodes and edges -->
  </svg>
  <figcaption>Auth flow showing the missing tenancy check at the resource handler
  (relates to <a class="xref" href="#SEC-004">SEC-004</a>).</figcaption>
</figure>
```

### Collapsible Sections

Use `<details>` for content that is contextually important but not central:

- Each theme of the Security Intent Brief, when there are many entries.
- The full enumeration of affected locations for a multi-location finding when the list exceeds ~5 entries.
- Long shell commands or curl invocations in an Exploit Scenario, when the scenario's narrative already conveys the attack.
- Sub-services in the Source/Sink Map for monorepos with many services.

Do not collapse Description, Impact, Exploit Scenario, or Recommendation. These are the report's core content.

### Cross-Reference Anchors

Each `<article class="finding">` has an `id` matching its `SEC-NNN` identifier. Use anchor links when:

- Two findings share a root cause.
- A finding's Exploit Scenario relies on another finding as a prerequisite.
- A diagram caption refers to a specific finding.

```html
<p>Combined with <a class="xref" href="#SEC-002">SEC-002</a> (weak token verification),
this enables full account takeover via a single forged request.</p>
```

### Severity, CWE, OWASP Tags

Already provided by the template:

- Severity via `<span class="severity-badge severity-{level}">`.
- CWE via `<span class="tag tag-cwe">CWE-NNN</span>`.
- OWASP via `<span class="tag tag-owasp">A0N:2021</span>` or `<span class="tag tag-owasp">API0N:2023</span>`.

Use them inside finding headers and in any inline reference; do not invent additional colored styles for the same purpose.

---

## Identifier Numbering

Same as the Markdown template: assign `SEC-NNN` identifiers sequentially starting at `SEC-001`, ordered by severity (Critical first, then High, Medium, Low). Within the same severity, order by domain category in the order defined by the Reference Index in `SKILL.md`, then by file path.

The identifier must appear as the `id` attribute on the `<article>` element exactly (no `sec-` prefix, no lowercase variant).

When a re-audit produces persistent findings from a prior report, assign new `SEC-NNN` identifiers in the current report and cross-reference the prior identifier in Appendix C. Do not reuse prior identifiers.

---

## Multi-Location Findings

For up to ~5 locations, list inline in the Location `<dd>`:

```html
<dt>Location</dt>
<dd>
  <code>src/api/users.py:42</code>,
  <code>src/api/orders.py:87</code>,
  <code>src/api/products.py:31</code>
</dd>
```

For more than ~5 locations, show the primary in the meta block and the full set in a collapsible inside the Description.

---

## Zero-Findings Report

When no issues are found, produce the full document with all counts set to 0 and each severity section containing:

```html
<section id="findings-critical" class="findings" data-severity="critical">
  <h2>Critical</h2>
  <p>No findings at this severity level.</p>
</section>
```

Still include `<section id="threat-model">`, `<section id="documented-security-posture">`, `<section id="appendix-source-sink-map">`, and `<section id="appendix-coverage">` — they document what was considered and what was analyzed.
