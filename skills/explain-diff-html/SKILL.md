---
name: explain-diff-html
description: Use when the user asks for a rich explanation of a code change, diff, branch, or PR. Publishes the result as a Claude Artifact.
---

# Explain Diff

Produce a rich, interactive explanation of the specified code change (diff, branch, or PR),
published as a Claude Artifact.

## Output artifact

- Load the `artifact-design` skill before writing the file — it calibrates the design pass for
  this page.
- One self-contained HTML file: inline CSS and inline JavaScript, no external assets.
- Write it to a scratch path first (the scratchpad directory if one is set for this session,
  otherwise `/tmp/YYYY-MM-DD-explanation-<slug>.html`), then publish it with the `Artifact` tool.
  The local file is a build step, not the deliverable.
- Set a `<title>` naming the change (short noun phrase, e.g. the feature or component touched),
  and pass a one-sentence `description` summarizing what changed.
- Pick one or two emoji as `favicon`; keep it stable across redeploys of the same explanation.
- If the user asks to update a previously published explanation, pass its artifact `url` to
  redeploy in place instead of creating a new one.
- One long page, section headers, and a table of contents at the top. No tabs for top-level
  structure.
- Basic responsive styling: readable on a phone. Theme-aware: support both light and dark, per
  the Artifact tool's theming rules.

## Sections

1. **Background** — explain the existing system relevant to this change. Explore the
   surrounding code broadly before writing this section. Split it into a deep background for
   readers new to the area (mark it skippable for readers who already know it), then a narrow
   background scoped directly to what the change touches.
2. **Intuition** — explain the core idea of the change. Favor the essence over exhaustive
   detail. Use concrete examples with toy data. Use figures and diagrams liberally.
3. **Code** — a high-level walkthrough of the changes, grouped and ordered so the change reads
   as a story, not a file-by-file dump.
4. **Quiz** — five medium-difficulty multiple-choice questions. Hard enough that answering them
   requires understanding the substance of the change, not just trivia or gotchas. Make them
   interactive: clicking an answer reveals whether it's correct, with feedback explaining why.

## Diagrams

- No ASCII diagrams. Build diagrams as plain HTML/CSS (boxes, arrows via CSS, flex/grid
  layouts), not embedded images.
- Reuse a small number of diagram families across the page (e.g. one style for "simplified UI
  view", one style for "system/data-flow view") rather than inventing a new visual language per
  section.
- Include concrete example data in every data-flow diagram.
- Use HTML lists for lists of things, not prose paragraphs.

## Code blocks

Always use `<pre>` tags for code. If a custom-styled `<div>` is used instead, it must set
`white-space: pre-wrap` — otherwise the browser collapses newlines to one line. Before saving
the file, check every code block's CSS includes `white-space: pre` or `pre-wrap`.

## Callouts

Use visually distinct callout boxes for key concepts, definitions, and important edge cases.

## Writing style

Write like Martin Kleppmann: engaging, precise, classic technical prose, with smooth
transitions between sections. Use simple English — the reader may not be a native speaker.

# Workflow

1. Identify the change and its scope. Use the current checkout, diff, branch, PR metadata, or user-supplied files as the source of truth. If the target is ambiguous, infer the most likely change from the available context and state the assumption in the page.
2. Explore relevant surrounding code, tests, configuration, callers, data models, and documentation. Trace the old and new paths far enough to explain behavior, not merely file-by-file edits. Prefer checked-in examples and tests over speculation.
3. Build a narrative before writing HTML:
   - what problem or constraint motivated the change;
   - how the old system behaved;
   - the smallest useful mental model of the new behavior;
   - how the implementation realizes that model;
   - edge cases, trade-offs, and observable consequences.
4. Write the output as one self-contained HTML file with inline CSS and JavaScript. Do not depend on external fonts, CDNs, images, JavaScript packages, or network access. Save it to a scratch path outside the repository (see "Output artifact" above).
5. Validate the file before publishing: confirm it is a complete HTML document, contains no external asset dependencies, has working quiz interactions, and satisfies the code-block, quiz, and color-contrast checks below.
6. Publish it with the `Artifact` tool (`file_path` pointing at the saved file, plus `title`/`description`/`favicon` as described above). Use `url` to redeploy if this explanation already has one.

## Required page structure

Include a clear title, a short summary, and a table of contents linking to these sections in this order:

1. **Background** — Explain only the system needed for the change. Start with an optional beginner-friendly mental model, then narrow to the exact components, contracts, and prior behavior involved.
2. **Intuition** — Explain the core idea before implementation detail. Use small concrete toy inputs and outputs. Show the old and new behavior when comparison makes the change clearer.
3. **Code** — Walk through the changes in conceptual groups, ordered by execution or dependency flow rather than arbitrary file order. Include precise file and line references when available, but do not dump the whole diff.
4. **Quiz** — Include exactly five medium-difficulty, interactive multiple-choice questions. Clicking an option must immediately show whether it is correct and explain why, including the relevant behavior or code path.

Use smooth transitions, plain language, and precise systems-oriented prose. Explain jargon on first use. Use callouts for definitions, invariants, important edge cases, and practical consequences. Keep the page readable on phones with responsive CSS. Do not use top-level tabs; make it one continuous page.

## Diagrams and examples

Use a small, reusable set of HTML/CSS diagram patterns rather than ornamental graphics:

- flow diagrams for requests, data, or control flow;
- before/after panels for changed behavior;
- labeled component cards for system boundaries;
- compact tables for mappings, invariants, and toy data.

Never use ASCII diagrams. Build diagrams with semantic HTML elements and CSS. Label arrows and include example values whenever the diagram describes data movement. Add accessible text or a caption so the explanation does not depend on visual inspection alone.

## Quiz quality rules

Treat quiz design as part of the explanation, not decoration. Before emitting the page, inspect all five questions as a set.

- Randomize the option order independently for each question. Do not always place the correct answer first, second, or in any fixed position. A deterministic shuffle with a per-page seed is acceptable; the visible order must vary across questions.
- Balance correct-answer positions across the five questions as evenly as possible. Never let position, letter, punctuation, or a repeated pattern reveal the answer.
- Keep options comparable in length, grammar, specificity, and confidence. Do not make the correct option conspicuously longer, more qualified, or more technically precise than distractors. Shorten or enrich distractors as needed.
- Make every distractor plausible and tied to a real misunderstanding of the change. Avoid joke answers, obviously impossible claims, “all/none of the above,” and trivia that cannot be inferred from the page.
- Ask about behavior, causality, contracts, edge cases, or trade-offs. Avoid questions whose answer can be guessed from a single copied phrase.
- Keep the correct answer and explanation in the page’s JavaScript data or DOM so the interaction works offline. Reveal feedback only after selection. Mark the selected option and explain both the right reasoning and, when useful, the misconception behind the distractors.
- Ensure the UI does not expose the answer through styling before selection, DOM labels, `title` attributes, source ordering, or accessibility text. Accessibility labels should describe the option, not its correctness.

## HTML and code-block constraints

- Escape user/code-derived text for HTML and JavaScript contexts. Preserve meaningful whitespace in code examples.
- Use `<pre><code>...</code></pre>` for code blocks. The CSS for `pre` must explicitly include `white-space: pre` or `white-space: pre-wrap`; verify every code block in the saved source before delivery.
- Keep JavaScript small, namespaced, and dependency-free. Use event listeners rather than inline handlers when convenient, and handle repeated quiz cards without relying on fragile global selectors.
- Include visible focus states and sufficient color contrast. Do not make correctness depend on color alone.
- Check every text/background color pair against its actual rendered background, not against white. A dark or muted blue (or purple) on a gray or muted background is a common failure — it reads as legible in isolation but fails contrast once placed in a callout, diagram box, or tinted card. Target WCAG AA: at least 4.5:1 for body text, 3:1 for large text (≥18px bold or ≥24px). When in doubt, pick a darker text color or a lighter background rather than trusting the pairing by eye.
- Avoid claiming behavior that the inspected source does not support. Distinguish observed facts from reasonable interpretation.

## Final handoff

Return the published Artifact URL. Briefly state what was inspected and any assumptions or
validation limitations. Do not place the deliverable inside the code repository unless the user
explicitly requests that.


