---
name: visual-doc
description: >-
  Generate a self-contained, on-brand interactive HTML document from researched or analyzed content (a research brief, repo/code analysis, concept explainer, comparison, design writeup) by composing a bespoke structure from a fixed design-system component palette.
  TRIGGER when: user asks to turn content into a document/report/writeup in this visual style, visualize findings, or make a shareable page from analysis (e.g., "이 스타일로 문서 만들어줘", "리서치 보고서로 정리해줘", "레포 분석 문서 만들어줘", "이거 문서화해줘", "make a document out of this").
  DO NOT TRIGGER when: user wants a diff/PR/commit explainer with a comprehension quiz (use explain-diff), an interactive simulation to inhabit a behavior (use micro-world), a picture explainer for a reader outside the domain (use eli5), a code review verdict (use code-review), or a short text answer serves better.
version: "1.5.0"
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
- Diagram (a flow, a sequence, a structure with relationships) → a hand-authored inline SVG in `.card > figure.fig` (see "Figures" below). Mermaid (`.bleed > .card.zoomable` wrapping a `pre.mermaid`, plus the `.lightbox` overlay) is the fallback for a diagram too large to lay out by hand
- Illustrative interaction → `.knob` and similar, or a figure redrawn by one or two inputs, **only when it passes the test in "Interactive figures" below**. This is a reading aid, not a playable simulation of behavior — that boundary belongs to `micro-world`.

**When no palette block fits, build one from the tokens.** Same variables, same radii, same shadow, same spacing rhythm. A net-new chart, diagram, or layout is expected; an off-token block is not.

**Choose a navigation strategy by scale:**

- Medium (roughly 5–12 sections): single scroll with the top progress bar and the floating TOC rail.
- Large (the document splits into distinct chapters): chapter pager (slim, text-forward) between chapters.
- Reference-shaped (FAQ, specs, lookup): collapsible sections so the reader scans instead of reading straight through.

Combine them when it helps. If you cap or truncate anything (top-N items, a sampled subset), say so in the document; silent truncation reads as completeness.

## Step 3: Build from the Palette

Read `components.html` from this skill's base directory. Copy the `:root` tokens, the base primitives, and only the component CSS you actually use into a single self-contained HTML file. **Fill and compose; never restyle the tokens.** No emoji, no ad-hoc SVG icons (the callout icons and the theme-toggle icon shipped in the palette are the sanctioned set; copy them verbatim), no external resources (CDN, webfonts, remote images, remote scripts). The output must stay one self-contained HTML file.

### Figures (hand-authored SVG)

A diagram is drawn by hand as inline SVG so it shares the document's tokens, follows the light/dark toggle, and renders wherever the file is opened. These rules are shared with `eli5` and `explain-diff`:

- Hand-authored inline SVG inside `<figure class="fig">`, sized by `viewBox` (CSS `width:100%; height:auto`), with `role="img"` and an `aria-label` that states the figure's claim.
- Draw the mechanism, not a row of labeled boxes: the path a request takes, the edge that disappears, a count drawn as that many lines. Label every arrow (`재발급 1회`, `20장씩`); an unlabeled arrow only says "related somehow".
- Real names in the boxes (the function, table, or endpoint as the code calls it), with at most one plain sub-line under the name. A line in a box is a name or a short field list, never a sentence.
- Strokes and text use `currentColor` so both themes work. `var(--blue)` marks the one element that carries the point, `var(--red)` the thing that fails. Two accent colors per figure at most.
- Text uses the label classes shipped in the template, copied verbatim (the block that starts with the `figure labels (SHARED` comment): `.dg-l` 13px, `.dg-s` 11px mono, at most one `.dg-b` big number, `.dg-halo` for a label that sits on a line, `.dg-em` and `.dg-bad` for the two accents.
- No `<script>`, `<style>`, or `<foreignObject>` inside the SVG, no emoji, no external resources.

**Mermaid is the fallback, not the default.** Use it only when the diagram has more elements than a hand-laid figure holds (roughly 15 or more nodes, or a dense graph whose layout you cannot plan row by row). It costs three things: the host has to draw it (Artifacts does; the same file opened in a local browser shows the source text), its colors ignore the theme toggle, and no single edge can be emphasized. When you use it, say so in your report.

#### Fit the text before you draw (no renderer needed)

SVG text does not wrap and does not push boxes apart. Every clipped label and every pair of overlapping labels in a shipped page came from a label that was wider than the room drawn for it. So **size the room from the label, never the label from the room**, with these per-character estimates. They are deliberately generous for the system fonts the page will be read in (Apple SD Gothic Neo, Pretendard, Segoe UI); a line that passes them fits.

| Text | Width per character |
|---|---|
| Hangul, in any class | 1.0 × font-size (`.dg-l` 13px, `.dg-s` 11px) |
| `m` `w` `M` `W` in `.dg-l` | 1.0 × font-size (13px) |
| other Latin lowercase, digits, `_` `-` `/` `.` in `.dg-l` | 0.6 × font-size (8px) |
| Latin uppercase in `.dg-l` | 0.75 × font-size (10px) |
| space, `·`, `(`, `)`, `:` in `.dg-l` | 4px |
| every non-Hangul character in `.dg-s` (mono), spaces included | 7px |
| anything not listed (`×`, `→`, `=`, `+`, `…`) | 1.0 × font-size |

Line height: 18px per `.dg-l` line, 15px per `.dg-s` line, 26px for `.dg-b`. A `<tspan>`'s `dy` is the height of the line **above** it (a `.dg-s` sub-line right under the `.dg-l` name has `dy="18"`; the next `.dg-s` line has `dy="15"`), so the box-height formula below and the `dy` values agree.

1. **Box from label.** Take the widest line of the box, estimate it, add 28 (14px padding each side), round up to a multiple of 10: that is the box width. Height is 14 + (sum of line heights) + 14; that is a minimum, and boxes in one row share the tallest height. First-line baseline: `y = box top + 25` (14 padding + the 13px name's ascent); each later line moves by its `dy`. In a box taller than the minimum, center the block instead: `y = box center − (Σ dy)/2 + 4`. A box is never narrower than its text; if the result is wider than you want, shorten the text.
2. **Lines grow the box, never the other way.** A box holds a name line and as many sub-lines as the content needs, each its own `<tspan x="{box center}" dy="…">`; the height formula absorbs them. Do not cut a field list to make a box small: a schema box that hides half its columns is a wrong drawing, not a tidy one.
3. **Row budget.** For every row: 20 + (caption column + 40, when rows carry a left-hand caption; size that column from its widest caption line like any other label) + Σ box widths + Σ gaps + 20 ≤ viewBox width. Gaps are at least 40; a gap that carries an arrow label is at least the label estimate + 24. When a row does not fit, shorten labels first, then break it into two rows, then (last) widen the viewBox up to 800. Never above 800: the figure is about 750px wide on screen, so a wider viewBox shrinks 13px text below 12px. Height is unlimited: more boxes mean more rows, not smaller text.
4. **Nothing crosses a border.** Every box label is `text-anchor="middle"` at the box center, and rule 1 guarantees it stays inside. Free-standing text (edge labels, row captions, the big number) gets a clear zone: no other text, no line other than the one it labels, and no box edge within 6px of its estimated extent, and that extent stays inside `[20, W-20]`. Vertically, a line of text occupies baseline −12 to +3 for `.dg-l`, −10 to +3 for `.dg-s`, −18 to +4 for `.dg-b`; use those for the clear zone, not just the horizontal estimate.
5. **Edges never pass through a box.** A line that crosses a box it does not connect is a defect, whichever direction it runs. When an edge has to skip a row, route it down a lane between boxes and turn below that row. Two edges that end on the same box land at least 20px apart along that side (or merge into one elbow before arriving). Give each `<svg>` its own marker id (`ar1`, `ar2`, …): a page has several drawings and duplicate ids are invalid.
   Preferred, not required: horizontal and vertical segments with an elbow (`M x1 y1 V ymid H x2 V y2`) rather than a diagonal, and the label on a straight segment: baseline 10px above a horizontal segment with `text-anchor="middle"`, or beside a vertical one with `text-anchor="end"` at `x = line x − 8`, in class `.dg-s`. If the two endpoints differ by less than 12px, draw the edge straight. A diagonal is fine when it is the clearer line; then its label carries `dg-halo` so it stays readable where it crosses. Rows stacked above each other are at least 40px apart (box bottom to next box top) when an edge label sits between them, 30px otherwise.
6. **Text comes last.** Inside each `<svg>`, write every `<rect>`, `<line>`, and `<path>` first and every `<text>` after them. SVG paints in document order, so a line written after a label is drawn over it, and `dg-halo` only works when the text is painted on top.
7. **Plan first, as a comment.** The first child of every `<svg>` is a one-line layout plan, and writing it is the check:
   `<!-- W=760 · row1 y=56 h=88: [40..190] gap45 [235..385] gap45 [430..580] gap40 [620..720] -->`
   Every range is closed, adjacent ranges differ by exactly the gap, the last end is ≤ W-20. If the arithmetic does not close, the drawing does not ship.
8. **No ceiling on boxes.** A subject with twelve tables gets twelve boxes in one drawing if that is what the reader needs to see at once; the row budget and the 800px width are the only limits, and height grows with rows. Split into two drawings only when the subject itself has two stories (a current path and an old path).

### Interactive figures

- **Static first.** Two fixed states (before/after, on/off, hit/miss) are drawn as two rows of one static figure: the eye compares both at once, and a toggle would hide one while showing the other.
- **Interactive only when the reader has to move a value to see the point.** That means the outcome changes across three or more values of one input (a count, a size, a rate, a threshold), or the point is the value at which the behavior flips (`요청 1개면 거부 없음, 2개부터 거부`). Then one or two inputs redraw the figure.
- **Three questions before building.** What is the input? What moves in the drawing when it changes? What does the reader learn at a value other than the default? If the third has no answer, draw it static.
- **The default state already makes the point.** A skimmer, a screenshot, and a printed page see only the state at load.
- **Never a step-through** that reveals already-visible content in order, and never an input that only shows or hides text.

The `.knob` in the palette is one shape of this (a value that feeds a formula). A figure whose SVG is redrawn by a range or a `.seg` switch is the other; its script goes at the end of `<body>`, computes the outcome with the subject's real rule, and builds shapes first, labels last. Do the fit arithmetic for the smallest and the largest input.

### Component rules (visual-doc)

- **Callout anatomy.** Every callout opens with a label row: the palette SVG icon plus the category word (참고 / 확인됨 / 주의 / 함정). Never use a bare text glyph (`i`, `!`, `×`) as an icon; fonts render them as thin bars or small dots. Inside a `.group`, each `.co-item` is a bold title line (`.ci-t`) plus a body; a standalone callout's label row serves as its title, so it carries only a body. Either way, do not fuse a bold lead into the body's first sentence.
- **Sentence-per-line in callouts.** Applies only when the body is long enough to need it: if the whole body fits in roughly one line, leave it as one flowing line; do not force two short stubs. When the body does exceed a line, wrap each sentence in `.sent` so lines break at sentence boundaries instead of mid-sentence, and keep each sentence short enough to hold one line (split or fall back to flowing text when it cannot).
- **Stacking rule.** 3 or more consecutive callouts of the same category are a list, not separate alarms: merge them into one `.group` callout (label row once, items divided by faint rules). One or two stay individual blocks.
- **Fill the column.** The layout is one centered content column (`.shell`, 800px) with the TOC rail fixed to the viewport's right edge (hidden under 1280px). Body prose fills that column: no `ch`-based `max-width` on prose (with Korean text `ch` computes to roughly half the intended width), and no `word-break: keep-all` on body-level text — in visual-doc, keep-all is reserved for titles and labels (a deliberate divergence from the explain-diff/micro-world narrative-text rule, recorded in docs/decisions/2026-07-visual-doc.md); on body paragraphs it drops whole words to the next line and leaves the right edge half-empty. Inline code chips are a brand-blue tint: `color-mix(in srgb, var(--blue) 9%, transparent)` background, `--chip-ink` text, `font-weight:500`, no border. `--chip-ink` is its own token (light `#1659C9`, dark `#7FB4FF`) because `--blue-dark` fails AA contrast on dark surfaces and on `--blue-soft` callouts — do not substitute `--blue-dark` or `--blue` for it. The translucent fill stacks naturally on white cards, gray panels, and soft callouts alike (never a fixed near-background fill like `--chipbg` or `--card`, which disappears on white).
- **Mermaid diagrams break out and zoom.** (Applies to the mermaid fallback only; a hand-authored figure stays in the column and needs no lightbox.) Mermaid renders its SVG scaled down to fit the container, so a diagram with many elements becomes unreadably small inside the 800px column. Wrap every diagram card in `.bleed > .card.zoomable` and give that card `role="button" tabindex="0" aria-label="다이어그램 확대"` (the script binds Enter/Space to `.zoomable`; without these attributes the diagram is mouse-only). The bleed is full-bleed to the viewport, capped so it never overlaps the fixed TOC rail at ≥1280px. Whenever the document contains at least one `pre.mermaid`, copy the `.lightbox` overlay markup and the lightbox `<script>` from the palette verbatim — click opens a full-screen view with wheel zoom (scroll-proportional exponential factor, trackpad-friendly), drag pan, double-click reset, and Esc/backdrop close. Do not re-derive the script: the drag-vs-click guard (`moved` threshold) and the cursor-anchored zoom math are easy to get subtly wrong. Omit both when the document has no diagrams.
- **Wide blocks extend right only.** When a table's cells wrap awkwardly at column width (a label or link column forced onto two lines) or a grid is inherently horizontal (4-column stats, a 3+ column comparison), wrap that block in `.wide`. The left edge stays on the column axis and the block grows rightward toward the TOC rail (capped just before it at ≥1280px, converging to column width on narrow viewports) — never use the symmetric `.bleed` for in-flow tables: jutting out on both sides breaks the shared left axis with chapter titles and prose and reads as misalignment (diagrams are the exception because they read as figures). If the block has its own subheading, put the subheading inside the `.wide` wrapper so heading and table share the same left edge. `.wide` also pins the table's first and last columns with `white-space:nowrap` at ≥800px viewports (labels and links stay on one line — verify neither column holds long free text; below 800px the pin is released so narrow screens wrap instead of overflowing). Prose, callouts, and TL;DR always stay at column width; widen only blocks that demonstrably need it.
- **Links carry the base underline.** Body links inherit the palette's `a` base (`--chip-ink` text, translucent `border-bottom` that solidifies on hover) — never re-define it per document. Any block-shaped element authored as an `<a>` (a card link, a nav item, a pager) must reset it with `border-bottom:none`, as `.toc-rail a` and `.pg-btn` do; otherwise the underline runs along the block's full width.
- **Stat copy.** Label first (what is being counted), number below, supplements on the `.d` line. Use only natural counters (개, 곳, 종); never coin a forced unit noun ("갈래") or compress the label into a translated noun pile. Keep the `.d` supplement to 3-4 comma-separated tokens at most (CSS clamps it at 2 lines with ellipsis; write short enough that the clamp never fires, and if a supplement would trip it, restate the full content in nearby body prose so the clamp never silently hides information). Cap `.stats` at 4 items; 5 or more become `.statchips`.

Delete the authoring comment at the top of `components.html` and the git-claw sample copy. Fill `<title>` — it names the browser tab: `{문서 한 줄 요약}`. Before writing, grep your output for `{문서` (the unfilled title token) and any leftover git-claw sample strings.

### Design rules (shared across visual-doc, explain-diff, micro-world, eli5)

- **Title never wraps mid-word.** Keep `word-break: keep-all` on the hero title and every narrative title so a Korean particle (`로`, `를`, `이`) can never fall to the start of a line. Keep titles short: a hero title is a noun phrase of ~20 Korean characters that holds one line; section titles are noun phrases of ~12 or fewer. Let the lede carry the what/why.
- **No decorative gradients.** Backgrounds are solid tokens (`var(--blue-soft)`, `var(--card)`, ...). A gradient is allowed only when it is functional, e.g. a fade scrim under a fixed bar, never as panel decoration.
- **No em-dash or en-dash (`—`, `–`) anywhere in the output.** Not in titles, prose, captions, quiz-free callouts, anywhere. They are a machine-writing tell. Use a colon, parentheses, a comma, or split into two sentences.
- **Color must survive its background.** A mark that carries meaning (a legend swatch, a chart track, a status dot) must contrast with the surface it sits on. A near-background gray (`--chipbg` on a card) reads as invisible; use `--gray200` or a real token so the mark is legible in both light and dark.
- **Both light and dark, switchable.** Do not hardcode colors that break one mode. The page follows the system setting by default, and the fixed `.theme-btn` in the top-right corner lets the reader override it. Copy three things verbatim and together: the button markup right after `<body>`, the `.theme-btn` CSS, and the small theme `<script>` in `<head>`. The dark tokens are defined twice on purpose (`@media (prefers-color-scheme: dark)` on `:root:not([data-theme="light"])`, and `:root[data-theme="dark"]`); keep the two blocks identical and never drop one, or the toggle stops winning in one direction.

### Writing style

The reader skims first, reads second.

- One idea per sentence. Split a sentence that joins two clauses with "그런데", "때문에", "면서" into two.
- Three to four sentences per paragraph, maximum. Break on the turn in the argument.
- A lede is 2-3 sentences: state the subject, then the point. Do not chain the whole story into one sentence.
- Bold the load-bearing phrase, not whole clauses. If three things are bold, nothing is.
- **One name per thing, the actor as subject, the count as a number.** (shared across eli5, explain-diff, micro-world, visual-doc) Pick one name for each thing and each action and keep it everywhere on the page, labels and drawings included: once it is `액세스 토큰` it is never `인증 토큰`, `세션 키`, or `자격 증명`; once it is `재발급` it is never `갱신`. A second name reads as a second thing. When two different things share a word, qualify both (`API 서버`, `인증 서버`, never a bare `서버`). Make the subject the thing that acts: `인증 서버가 두 번째 요청을 거부했어요`, not `두 번째 요청이 거부됐어요`; a sentence that hides the actor hides where the cause is. Write the count the source gives (`요청 3개`, `3번`), not `여러 개` or `요청 수만큼`.
- **Code blocks: one shared component.** Show code only inside the template's `.codeblock` (`.cb-head` with the location on the left (`파일 · 함수`) and the language on the right, then `.cb-body > pre`, one `.ln` span per line). Copy its CSS block verbatim; it is identical in all four templates. Strip the indentation common to every quoted line and keep the relative indentation; nothing else in the text changes. Replace skipped lines with one `<span class="ln gap">… N줄 생략</span>`, indented to the depth of the code it replaces. Color syntax by wrapping tokens yourself, never with a highlighter script or CDN: `sx-k` keywords (`if`, `and`, `return`, `function`), `sx-f` function names where called or defined, `sx-s` string literals, `sx-n` numbers and language constants (`True`, `None`, `null`), `sx-c` comments. Leave variables, operators, and punctuation unwrapped, and HTML-escape `<`, `>`, `&`. Never hard-wrap a long line or set `pre-wrap`: the block scrolls sideways with an always-visible scrollbar.
- **Natural Korean, not machine translation.** Do not coin stiff 한자어 Koreans do not say; keep a code/identifier term verbatim rather than force-translate it. Read each title and takeaway aloud once; if it sounds like a translated caption, rewrite it plainer.
- **No metaphor, no implied meaning, no translationese in titles and labels.** A heading, a chip, a row caption, a sub-line, a tooltip states the fact in the subject's own words (`vector_store_file 1행`, `호출 최대 N회`, `원본 테이블`), terse 개조식 welcome. Not `장부`, `선반`, `고리를 돈다`, `~의 여정`, `한눈에 보는 ~`, `숨은 비용`, `마법은 없다`. A phrase that would sound odd said aloud to a colleague is cut; a phrase that could head a page about a different subject is too abstract.

## Step 4: Verify the Render

The rules above are the first defense; a screenshot is the second. If a headless browser is available, render the output and look at it before reporting:

```bash
npx --no-install playwright screenshot --full-page "file://<abs-path>" /tmp/visual-doc-check.png
```

Check: the title does not wrap mid-word; charts (line/donut) render with legible, background-contrasting marks and sane proportions; no figure label is clipped or overlapping (if you cannot render, redo the fit arithmetic), and an interactive figure makes its point in the state at load; no block is broken; dark mode holds (`--color-scheme=dark`) and all three parts of the toggle survived: `grep -c '<button type="button" class="theme-btn"'`, `grep -c '\.theme-btn{'`, and `grep -c 'git-claw-doc-theme'` must each print at least 1 (a bare `theme-btn` match can be prose that merely mentions it). Fix and re-render until it is clean. If no browser is available, rely on the rules and say the render was not visually verified.

## Step 5: Output

1. Write the file to the **repository root**: `visual-doc-<slug>.html` (slug from the topic, kebab-case). If not in a repo, write to the current directory.
2. Report the **absolute path**. Do NOT auto-open, do NOT commit, do NOT add to .gitignore. Delete only when the user asks.
3. Content language follows the project's AGENTS.md (or CLAUDE.md as fallback) setting; if none, the user's conversational language. Korean output uses 해요체 throughout (headings may stay nominal).

**Constraints:**

- Read-only with respect to git state: this skill creates exactly one HTML file and nothing else.
- Tokens and the component design are fixed by `components.html` and the design-system record (`docs/decisions/2026-07-explain-diff-design-system.md`, `docs/decisions/2026-07-visual-doc.md` in the git-claw repo). Do not improvise styles.
- If the request is really a diff explainer (use `explain-diff`) or a behavior to inhabit (use `micro-world`), say so and hand off rather than forcing a document.
