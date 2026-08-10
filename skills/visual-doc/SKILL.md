---
name: visual-doc
description: >-
  Generate a self-contained, on-brand interactive HTML document from researched or analyzed content (a research brief, repo/code analysis, concept explainer, comparison, design writeup) by composing a bespoke structure from a fixed design-system component palette.
  TRIGGER when: user asks to turn content into a document/report/writeup in this visual style, visualize findings, or make a shareable page from analysis (e.g., "이 스타일로 문서 만들어줘", "리서치 보고서로 정리해줘", "레포 분석 문서 만들어줘", "이거 문서화해줘", "make a document out of this").
  DO NOT TRIGGER when: user wants a diff/PR/commit explainer with a comprehension quiz (use explain-diff), an interactive simulation to inhabit a behavior (use micro-world), a code review verdict (use code-review), or a short text answer serves better.
version: "1.2.2"
allowed-tools: Bash(git *), Bash(gh *), Bash(npx *), Read, Grep, Glob, Write
---

## The One Principle

**Style is locked; structure is decided by the content.** The document must look like it belongs to one design system no matter what it contains, yet its skeleton must expand or contract to fit what was actually researched, its scale, and its kind. A fixed template produces the same shape every time; that is the failure mode this skill exists to avoid.

So the deliverable is built in two layers:

- **Locked layer** — design tokens, base primitives, tone. These never change. They are what makes every visual-doc recognizably one family (a sibling of `explain-diff` and `micro-world`; they share tokens, never structure).
- **Composed layer** — the section skeleton, chosen per document. Assemble it from the component palette in `components.html`, and **when the content needs a block the palette does not have, author a new one from the same tokens.** The palette is a cookbook of exemplars, not an exhaustive list.

This is not a diff explainer and carries **no quiz, no comprehension gate, no diff-specific machinery**. It communicates; it does not test.

The default reader is whoever the content is for: a teammate reading a research brief, an engineer reading a repo analysis, a stakeholder reading a design writeup. Write for that reader, not for a beginner.

Honesty rules apply throughout:

- State sourced facts as facts; state inferences as inferences. Do not present a guess as established.
- Quote source material **verbatim** when you quote it (code, docs, data). Never reconstruct from memory.
- Do not invent content to fill a section. If a block has nothing real to hold, drop the block.
- Never present the document as proof of correctness. It is a communication artifact.

## Parse Arguments

| Argument | Meaning |
|----------|---------|
| (none) | Build the document from the current conversation's content (the research, analysis, or material already discussed) |
| a topic / question | Focus the document on that subject |
| a path (e.g. `src/`) | The document analyzes that code/directory as its subject |
| a PR number | The document analyzes that PR as its subject |

If the content does not yet exist (nothing has been researched or analyzed), gather it first — read the code, run the search, do the analysis — then document it. Do not fabricate a subject.

## Step 1: Understand the Request and Secure the Content

1. Name the **document kind**: research brief, repo/code analysis, concept explainer, comparison/evaluation, design writeup, status report, or something else. The kind drives which components fit.
2. Name the **one thing the reader must leave with**. Everything in the document serves that.
3. Secure the **actual content**. Read the files, gather the facts, extract verbatim quotes and real numbers. A document is only as good as what it is built from.
4. Estimate **scale**: a few sections, or many. This decides navigation (Step 2).

## Step 2: Design the Skeleton (structure follows content)

Do not open `components.html` and fill it top to bottom. Design the skeleton first, from the content.

**Choose sections by the content's own logic.** A repo analysis might go overview → structure → design principles → workflow. A research brief might go question → findings → evidence → implications. A comparison might go criteria → contenders → verdict. There is no fixed part count.

**Choose a component for each section by what it must show** (the palette in `components.html`):

- Summary / thesis → `.tldr` highlight, `.takeaway`
- Scale / metrics → `.stats` row (4 items max; use `.statchips` for 5+), `.chart` (CSS bars), `.linechart` / `.donut` (SVG)
- Structure / layout → `.tree`, `.pillars`
- Comparison → `.ctable`, `.proscons`
- Sequence / process → `.timeline`, `.flow`
- Evidence → `.quote2` (card quote, never a left-band), annotated `.codeblock`, `.spec` key-value
- Q&A / reference → `.qa` (collapsible), callout variants (`info` / `ok` / `warn` / `err`; merge 3+ consecutive same-type callouts into one `.group`)
- Diagram (mermaid) → `.bleed > .card.zoomable` wrapping a `pre.mermaid`, plus the `.lightbox` overlay (see the Diagrams component rule below)
- Illustrative interaction → `.knob` and similar, **only when the reader's input changes the outcome** (a value they drag, a scenario they toggle). This is a reading aid, not a playable simulation of behavior — that boundary belongs to `micro-world`. Never build a step-through that reveals already-visible content in order.

**When no palette block fits, build one from the tokens.** Same variables, same radii, same shadow, same spacing rhythm. A net-new chart, diagram, or layout is expected; an off-token block is not.

**Choose a navigation strategy by scale:**

- Medium (roughly 5–12 sections): single scroll with the top progress bar and the floating TOC rail.
- Large (the document splits into distinct chapters): chapter pager (slim, text-forward) between chapters.
- Reference-shaped (FAQ, specs, lookup): collapsible sections so the reader scans instead of reading straight through.

Combine them when it helps. If you cap or truncate anything (top-N items, a sampled subset), say so in the document; silent truncation reads as completeness.

## Step 3: Build from the Palette

Read `components.html` from this skill's base directory. Copy the `:root` tokens, the base primitives, and only the component CSS you actually use into a single self-contained HTML file. **Fill and compose; never restyle the tokens.** No emoji, no ad-hoc SVG icons (the callout icons shipped in the palette are the one sanctioned set; copy them verbatim), no external resources (CDN, webfonts, remote images, remote scripts). The output must stay one self-contained HTML file.

### Component rules (visual-doc)

- **Callout anatomy.** Every callout opens with a label row: the palette SVG icon plus the category word (참고 / 확인됨 / 주의 / 함정). Never use a bare text glyph (`i`, `!`, `×`) as an icon; fonts render them as thin bars or small dots. Inside a `.group`, each `.co-item` is a bold title line (`.ci-t`) plus a body; a standalone callout's label row serves as its title, so it carries only a body. Either way, do not fuse a bold lead into the body's first sentence.
- **Sentence-per-line in callouts.** Applies only when the body is long enough to need it: if the whole body fits in roughly one line, leave it as one flowing line; do not force two short stubs. When the body does exceed a line, wrap each sentence in `.sent` so lines break at sentence boundaries instead of mid-sentence, and keep each sentence short enough to hold one line (split or fall back to flowing text when it cannot).
- **Stacking rule.** 3 or more consecutive callouts of the same category are a list, not separate alarms: merge them into one `.group` callout (label row once, items divided by faint rules). One or two stay individual blocks.
- **Fill the column.** The layout is one centered content column (`.shell`, 800px) with the TOC rail fixed to the viewport's right edge (hidden under 1280px). Body prose fills that column: no `ch`-based `max-width` on prose (with Korean text `ch` computes to roughly half the intended width), and no `word-break: keep-all` on body-level text — in visual-doc, keep-all is reserved for titles and labels (a deliberate divergence from the explain-diff/micro-world narrative-text rule, recorded in docs/decisions/2026-07-visual-doc.md); on body paragraphs it drops whole words to the next line and leaves the right edge half-empty. Inline code chips are a brand-blue tint: `color-mix(in srgb, var(--blue) 9%, transparent)` background, `--chip-ink` text, `font-weight:500`, no border. `--chip-ink` is its own token (light `#1659C9`, dark `#7FB4FF`) because `--blue-dark` fails AA contrast on dark surfaces and on `--blue-soft` callouts — do not substitute `--blue-dark` or `--blue` for it. The translucent fill stacks naturally on white cards, gray panels, and soft callouts alike (never a fixed near-background fill like `--chipbg` or `--card`, which disappears on white).
- **Diagrams break out and zoom.** Mermaid renders its SVG scaled down to fit the container, so a diagram with many elements becomes unreadably small inside the 800px column. Wrap every diagram card in `.bleed > .card.zoomable` and give that card `role="button" tabindex="0" aria-label="다이어그램 확대"` (the script binds Enter/Space to `.zoomable`; without these attributes the diagram is mouse-only). The bleed is full-bleed to the viewport, capped so it never overlaps the fixed TOC rail at ≥1280px. Whenever the document contains at least one `pre.mermaid`, copy the `.lightbox` overlay markup and the lightbox `<script>` from the palette verbatim — click opens a full-screen view with wheel zoom (scroll-proportional exponential factor, trackpad-friendly), drag pan, double-click reset, and Esc/backdrop close. Do not re-derive the script: the drag-vs-click guard (`moved` threshold) and the cursor-anchored zoom math are easy to get subtly wrong. Omit both when the document has no diagrams.
- **Wide blocks extend right only.** When a table's cells wrap awkwardly at column width (a label or link column forced onto two lines) or a grid is inherently horizontal (4-column stats, a 3+ column comparison), wrap that block in `.wide`. The left edge stays on the column axis and the block grows rightward toward the TOC rail (capped just before it at ≥1280px, converging to column width on narrow viewports) — never use the symmetric `.bleed` for in-flow tables: jutting out on both sides breaks the shared left axis with chapter titles and prose and reads as misalignment (diagrams are the exception because they read as figures). If the block has its own subheading, put the subheading inside the `.wide` wrapper so heading and table share the same left edge. `.wide` also pins the table's first and last columns with `white-space:nowrap` at ≥800px viewports (labels and links stay on one line — verify neither column holds long free text; below 800px the pin is released so narrow screens wrap instead of overflowing). Prose, callouts, and TL;DR always stay at column width; widen only blocks that demonstrably need it.
- **Links carry the base underline.** Body links inherit the palette's `a` base (`--chip-ink` text, translucent `border-bottom` that solidifies on hover) — never re-define it per document. Any block-shaped element authored as an `<a>` (a card link, a nav item, a pager) must reset it with `border-bottom:none`, as `.toc-rail a` and `.pg-btn` do; otherwise the underline runs along the block's full width.
- **Stat copy.** Label first (what is being counted), number below, supplements on the `.d` line. Use only natural counters (개, 곳, 종); never coin a forced unit noun ("갈래") or compress the label into a translated noun pile. Keep the `.d` supplement to 3-4 comma-separated tokens at most (CSS clamps it at 2 lines with ellipsis; write short enough that the clamp never fires, and if a supplement would trip it, restate the full content in nearby body prose so the clamp never silently hides information). Cap `.stats` at 4 items; 5 or more become `.statchips`.

Delete the authoring comment at the top of `components.html` and the git-claw sample copy. Fill `<title>` — it names the browser tab: `{문서 한 줄 요약}`. Before writing, grep your output for `{문서` (the unfilled title token) and any leftover git-claw sample strings.

### Design rules (shared across visual-doc, explain-diff, micro-world)

- **Title never wraps mid-word.** Keep `word-break: keep-all` on the hero title and every narrative title so a Korean particle (`로`, `를`, `이`) can never fall to the start of a line. Keep titles short: a hero title is a noun phrase of ~20 Korean characters that holds one line; section titles are noun phrases of ~12 or fewer. Let the lede carry the what/why.
- **No decorative gradients.** Backgrounds are solid tokens (`var(--blue-soft)`, `var(--card)`, ...). A gradient is allowed only when it is functional, e.g. a fade scrim under a fixed bar, never as panel decoration.
- **No em-dash or en-dash (`—`, `–`) anywhere in the output.** Not in titles, prose, captions, quiz-free callouts, anywhere. They are a machine-writing tell. Use a colon, parentheses, a comma, or split into two sentences.
- **Color must survive its background.** A mark that carries meaning (a legend swatch, a chart track, a status dot) must contrast with the surface it sits on. A near-background gray (`--chipbg` on a card) reads as invisible; use `--gray200` or a real token so the mark is legible in both light and dark.
- **Both light and dark.** The tokens already carry a `prefers-color-scheme: dark` override. Do not hardcode colors that break one mode.

### Writing style

The reader skims first, reads second.

- One idea per sentence. Split a sentence that joins two clauses with "그런데", "때문에", "면서" into two.
- Three to four sentences per paragraph, maximum. Break on the turn in the argument.
- A lede is 2-3 sentences: state the subject, then the point. Do not chain the whole story into one sentence.
- Bold the load-bearing phrase, not whole clauses. If three things are bold, nothing is.
- **Natural Korean, not machine translation.** Do not coin stiff 한자어 Koreans do not say; keep a code/identifier term verbatim rather than force-translate it. Read each title and takeaway aloud once; if it sounds like a translated caption, rewrite it plainer.

## Step 4: Verify the Render

The rules above are the first defense; a screenshot is the second. If a headless browser is available, render the output and look at it before reporting:

```bash
npx --no-install playwright screenshot --full-page "file://<abs-path>" /tmp/visual-doc-check.png
```

Check: the title does not wrap mid-word; charts (line/donut) render with legible, background-contrasting marks and sane proportions; no block is broken; dark mode holds (`--color-scheme=dark`). Fix and re-render until it is clean. If no browser is available, rely on the rules and say the render was not visually verified.

## Step 5: Output

1. Write the file to the **repository root**: `visual-doc-<slug>.html` (slug from the topic, kebab-case). If not in a repo, write to the current directory.
2. Report the **absolute path**. Do NOT auto-open, do NOT commit, do NOT add to .gitignore. Delete only when the user asks.
3. Content language follows the project's AGENTS.md (or CLAUDE.md as fallback) setting; if none, the user's conversational language. Korean output uses 해요체 throughout (headings may stay nominal).

**Constraints:**

- Read-only with respect to git state: this skill creates exactly one HTML file and nothing else.
- Tokens and the component design are fixed by `components.html` and the design-system record (`docs/decisions/2026-07-explain-diff-design-system.md`, `docs/decisions/2026-07-visual-doc.md` in the git-claw repo). Do not improvise styles.
- If the request is really a diff explainer (use `explain-diff`) or a behavior to inhabit (use `micro-world`), say so and hand off rather than forcing a document.
