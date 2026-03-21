# Visual Companion for Planning

During blueprint planning, some decisions are easier to evaluate visually than as text. Architecture diagrams, data flow visualizations, and side-by-side approach comparisons all benefit from browser-based presentation.

## When Browser vs Terminal

Not every planning question needs the browser. Apply this filter per question: **does seeing this convey more than reading about it?**

**Browser** — content that has spatial structure:

- Architecture diagrams showing component boundaries and connections
- Data flow visualizations across system layers
- Side-by-side comparison of competing approaches with structural differences
- Component relationship maps and dependency graphs

**Terminal** — content that is textual or decisional:

- Scope and requirements questions
- Trade-off lists and pros/cons
- Tool chain confirmation
- Priority and sequencing decisions
- Clarifying questions about constraints

A question about architecture is not automatically visual. "Should we use a monolith or microservices?" is a trade-off discussion for the terminal. "Here are the three service boundaries we could draw" is a diagram for the browser.

## Offering the Companion

When a planning session will involve architectural decisions that benefit from diagrams, offer the companion once at the start. Frame it as an optional tool, not a requirement — the user opts in and it becomes available for questions where visuals help. Not every subsequent question routes through the browser; each question is evaluated individually.

If the user declines, continue entirely in the terminal. Planning quality does not depend on visual presentation.

## Serving Visual Content

**With the superpowers plugin**: Use the brainstorming server (`scripts/start-server.sh`) which provides themed presentation, interactive option selection, and event capture. Refer to superpowers brainstorming's visual-companion documentation for server setup and content authoring details.

**Without superpowers**: Write standalone HTML files to a temporary directory and serve them with `python3 -m http.server` or a similar lightweight server. This approach supports static diagrams and comparison layouts but lacks interactive selection and theming. For planning purposes — where the goal is communicating structure, not collecting UI preferences — static rendering is often sufficient.

## Per-Question Routing

After the user accepts the companion, continue deciding per question:

- Proposing approaches with structural differences → browser diagram
- Asking which approach to pick based on trade-offs → terminal
- Showing how data flows through a proposed architecture → browser
- Confirming scope boundaries or constraints → terminal

Default to the terminal. Use the browser only when spatial layout genuinely aids comprehension.
