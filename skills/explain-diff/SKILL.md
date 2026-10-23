---
name: explain-diff
description: >-
  Generate a self-contained interactive HTML explainer for a code diff, commit, branch, or PR so the developer genuinely understands the change before sharing or merging it.
  TRIGGER when: user asks to explain a diff/PR/commit/changes or wants an understanding document (e.g., "diff 설명해줘", "이 변경 이해하게 해줘", "explain this PR", "변경사항 설명 문서 만들어줘").
  DO NOT TRIGGER when: user wants defect findings or a review verdict (use code-review), the reader is someone outside the domain who needs a picture explainer (use eli5), user is committing or creating PRs, or asks a quick question about a specific line that a direct answer serves better.
version: "1.5.1"
allowed-tools: Bash(git *), Bash(gh *), Bash(google-chrome *), Bash(chromium *), Bash(cat *), Bash(python3 *), Bash(which *), Bash(grep *), Read, Grep, Glob, Write
---

## The One Principle

**Build genuine understanding first, then explain it.** The document exists so the reader can pass a quiz about the change without opening the files: and honestly could not before reading it. Inspired by Geoffrey Litt's explain-diff: understanding, not correctness, is the bottleneck of agent-written code.

The default reader is the **author-practitioner**: someone who works on this codebase and is about to review, merge, or defend this change. Skip beginner background; include behavior deltas, failure modes, and design rationale.

Honesty rules apply throughout:

- Never present a suspicion as a confirmed bug: frame it as an attention pointer.
- Distinguish stated intent (commit messages, PR body) from your inferred interpretation.
- Quote code **verbatim only**: never paraphrase or reconstruct from memory. The one allowed change is removing the indentation common to every quoted line (relative indentation stays). Every quoted block must survive comparison against the actual file.
- Do not invent content for a section with nothing to say: omit the section.

## Parse Arguments

| Argument | Type | Description |
|----------|------|-------------|
| (none) | (none) | Explain the working diff: `git diff HEAD` plus branch commits vs the default branch |
| PR number | e.g. `42` | Explain that PR via `gh pr view` / `gh pr diff` |
| commit / range | e.g. `a1b2c3d`, `main..feat/x` | Explain that commit or range |
| path | e.g. `src/auth/` | Restrict the diff to this path |

## Step 1: Acquire the Diff

Resolve the target and gather, in one pass:

```bash
git diff HEAD                                   # or: gh pr diff {n} / git diff {range}
git log --oneline {range}                       # commit messages = stated intent
gh pr view {n} --json title,body 2>/dev/null    # PR mode only
git diff {range} --stat                         # file count, +/- totals for the metabar
```

## Step 2: Investigate Before Writing

Do NOT start writing the document from the diff alone.

1. Read every hunk **plus its enclosing function/class** (Read the file, not just the hunk).
2. Extract **stated intent** from commit messages and the PR body.
3. Explore the surrounding system: callers of changed functions (Grep), related tests, config/schema the change touches.
4. Apply the analysis lenses: change taxonomy (feature/fix/refactor/removal), removed-behavior audit (what no longer happens), cross-file impact, invariant/contract delta, design pressure (what alternative was rejected and why), test delta.
5. Group hunks into **narrative themes ordered by importance**: one theme per Part 2 card. A theme is a behavioral unit ("token refresh moved to interceptor"), never a file list.

## Step 3: Build the Document

Read `template.html` from this skill's base directory. It carries the full design system (tokens, components, generic quiz/gate JS): **fill it, never restyle it**. No emoji (source quoted verbatim in a code block keeps its emoji, like any other character), no hand-drawn SVG icons (a flow figure is a diagram, not an icon; see "Flow figure" below), no external resources (CDN, webfonts, remote images). The output must stay a single self-contained HTML file.

**Light and dark, switchable.** The page follows the system setting by default, and the fixed `.theme-btn` in the top-right corner lets the reader override it. Keep three things from the template verbatim and together: the button markup right after `<body>`, the `.theme-btn` CSS, and the small theme `<script>` in `<head>` (the shipped toggle icon is the one sanctioned SVG icon). The dark tokens are defined twice on purpose (`@media (prefers-color-scheme: dark)` on `:root:not([data-theme="light"])`, and `:root[data-theme="dark"]`); keep the two blocks identical and never drop one, or the toggle stops winning in one direction. Any CSS you author yourself takes every color from a token: `var(--surface)` for a raised white surface, `var(--on-accent)` for text on a `--blue`/`--green` fill, `var(--track)` and `var(--shadow-sm)` for tracks and small shadows. A literal `#fff`, a black-alpha `rgba(...)`, or an inline `style="color:..."` with a hex value reads in one theme only. This applies above all to the optional interactive figure, which is the only CSS you write (a flow figure needs none: its classes ship in the template).

**Rows by default; side by side only when the render proves it fits.** (shared across eli5, explain-diff, micro-world, visual-doc) When two things are shown for comparison (two states, two options, two code blocks, a request and its response, two evidence cards), stack them as rows: each keeps the full column width and the eye reads down instead of jumping across. Side by side is a judgment call, not a ban, and it is judged on the rendered page, not by counting characters: it stays only when nothing inside either half scrolls sideways and the reader gains from seeing the two at the same height (two short lists, two table schemas, a control and the view it changes). To measure, append `measure.html` from this skill's base directory to a copy of the page and read the result with the one-line command in its header comment: `data-sbs` names every multi-column grid with something scrolling sideways inside it, and every block it names is stacked. Without a browser there is no measurement, so stack.

**Code blocks: one shared component.** Show code only inside the template's `.codeblock` (`.cb-head` with the location on the left (`파일, 함수`) and the language on the right, then `.cb-body > pre`, one `.ln` span per line). Copy its CSS block verbatim; it is identical in all four templates. Strip the indentation common to every quoted line and keep the relative indentation; nothing else in the text changes. Replace skipped lines with one `<span class="ln gap">… N줄 생략</span>`, indented to the depth of the code it replaces. Color syntax by wrapping tokens yourself, never with a highlighter script or CDN: `sx-k` keywords (`if`, `and`, `return`, `function`), `sx-f` function names where called or defined, `sx-s` string literals, `sx-n` numbers and language constants (`True`, `None`, `null`), `sx-c` comments. Leave variables, operators, and punctuation unwrapped, and HTML-escape `<`, `>`, `&`. Never hard-wrap a long line or set `pre-wrap`: the block scrolls sideways with an always-visible scrollbar. In a diff excerpt every line starts with its sign: `<span class="ln add"><span class="sign">+</span> …</span>`, `<span class="ln del"><span class="sign">-</span> …</span>`, or two spaces for a context line. Wrap tokens on added and context lines; removed lines need no wrapping (the CSS mutes them to gray so the new code draws the eye).

**Before writing anything else, clear the template's own scaffolding:**

- Delete the authoring comment at the top of the file (`explain-diff output template. Fill every {{PLACEHOLDER}}…`). It is instructions for you, not content for the reader.
- Fill `<title>`: it names the browser tab. `explain-diff: {한 줄 요약} ({대상})`.
- Delete every `REPEAT` / `FIGURE SLOT` comment (both the interactive slot in Part 1 and the `FLOW FIGURE SLOT` in the theme card) once its block is filled or dropped.
- Before writing the file, grep your output for `{{`: a surviving placeholder means an unfilled slot.
- Confirm all three parts of the toggle survived: `grep -c '<button type="button" class="theme-btn"'`, `grep -c '\.theme-btn{'`, and `grep -c 'git-claw-doc-theme'` must each print at least 1 (a bare `theme-btn` match can be prose that merely mentions it).
- If you wrote any CSS or inline `style` of your own, re-read it for a literal `#fff`, a hex color, or an `rgba(`: every color comes from a token. (The `:root` token blocks contain literals by design, so a whole-file grep for these is not a usable check.)

### Writing style

The reader skims first and reads second. Prose that runs on defeats both.

- **One idea per sentence.** Split any sentence carrying two clauses joined by "그런데", "때문에", "면서" into two.
- **Three to four sentences per paragraph, max.** Break on the turn in the argument, not at an arbitrary length. A `<p class="prose">` that fills more than ~5 lines on screen needs splitting.
- **Lede is 2-3 sentences, not one long one.** State the problem, then the fix. Do not chain the whole causal story into a single sentence.
- Bold the load-bearing phrase in a paragraph (`<b>`), not whole clauses. If three things are bold, nothing is.
- Prefer a concrete subject over a nominalization: "호스트가 규칙을 못 읽어요" beats "규칙 조회가 실패해요".
- **One name per thing, the actor as subject, the count as a number.** (shared across eli5, explain-diff, micro-world, visual-doc) Pick one name for each thing and each action and keep it everywhere on the page, labels and drawings included: once it is `액세스 토큰` it is never `인증 토큰`, `세션 키`, or `자격 증명`; once it is `재발급` it is never `갱신`. A second name reads as a second thing. When two different things share a word, qualify both (`API 서버`, `인증 서버`, never a bare `서버`). Make the subject the thing that acts: `인증 서버가 두 번째 요청을 거부했어요`, not `두 번째 요청이 거부됐어요`; a sentence that hides the actor hides where the cause is. Write the count the source gives (`요청 3개`, `3번`), not `여러 개` or `요청 수만큼`.
- **A particle is part of the word before it; a count stays on one line.** (shared across eli5, explain-diff, micro-world, visual-doc) Attach every particle and ending to the word before it, also after an English word, a number, or a code chip: `Job은`, `store에도`, `API와`, `400이`, `<code>job_indexing.mode</code>가`, `automatic인`. Never `Job 은` or `</code> 을`: where `word-break: keep-all` applies a line breaks only at a space, so that space lets the particle start a line. Choose the particle by how the word is read aloud (`Job은`, `API는`, `store를`). Write a digit and its unit together (`3개`, `0행`, `1회`), and put `&nbsp;` between a native numeral and its unit (`한&nbsp;번`, `두&nbsp;줄`, `세&nbsp;가지`) so the line never breaks inside the count. Bind nothing else with `&nbsp;`: each one makes a longer unbreakable run and a more ragged right edge. Before finishing, run `grep -nP '[A-Za-z0-9_)]+(</code>)? (은|는|이|가|을|를|의|에|와|과|로|으로|도|만|에서|에는|에도|인|인데)(?=[ ,.<)]|$)'` on the file and fix every hit that is a particle (`이` and `가` can also be a word of their own), except inside source quoted verbatim in a code block or an evidence card, which keeps its spacing.

### Natural Korean (do not read like machine translation)

- **No metaphor, no implied meaning, no translationese in titles and labels.** A heading, a chip, a row caption, a sub-line, a tooltip states the fact in the subject's own words (`vector_store_file 1행`, `호출 최대 N회`, `원본 테이블`), terse 개조식 welcome. Not `장부`, `선반`, `고리를 돈다`, `~의 여정`, `한눈에 보는 ~`, `숨은 비용`, `마법은 없다`. A phrase that would sound odd said aloud to a colleague is cut; a phrase that could head a page about a different subject is too abstract.

The document must read as if a Korean engineer wrote it, not as if English or a raw code term were transliterated. Translation-ese and coined 한자어 are the loudest "AI wrote this" tell: remove them.

- **No machine-writing symbols anywhere in the output.** (shared across eli5, explain-diff, micro-world, visual-doc) No em-dash (`—`), en-dash (`–`), or middle dot (`·`) in titles, prose, captions, labels, SVG text, TOC descriptions, hints, quiz text, eyebrows, or code-block headers. They are machine-writing tells. A dash becomes a colon, parentheses, a comma, or two sentences: write `옵션 1만 vs 옵션 1+2: 권장안과 근거`, never the same with a dash. A middle dot between items becomes a comma or `와`/`과` (`설치와 로그인`, `제목, 소제목, 라벨`); a dot used as a separator in a label becomes a comma or a line break. A range is `~` (`1~3`). The one exception is source quoted verbatim in a code block or an evidence card, which keeps its characters exactly; in a prose quote (`.quote2`, an inline quote) quote a shorter span that does not contain one, and never rewrite a quote to remove it. Before finishing, `grep -nP '[—–·]' <file>` must print nothing except lines of such verbatim source.
- **Do not coin stiff 한자어 that Koreans do not actually say.** If a natural word or the plain English term reads better, use it. `접두` → `앞에 붙는 upstage/` 또는 그냥 `prefix`; `회귀 커밋` → `되돌린 커밋`; `귀속` → `~때문에`. When unsure, say it the way you would out loud.
- **Keep a code/identifier term verbatim rather than force-translating it.** `providerId`, `llmProxyUse`, `prefix` stay as-is; wrapping them in an awkward Korean coinage only adds noise.
- Read each title and takeaway aloud once. If it sounds like a translated caption, rewrite it plainer.

### Section titles

All titles (the hero title included) are **short noun phrases (개조식/명사형), not sentences.** Match the built-in ones (`배경`, `구조 한눈에 보기`, `주의해서 볼 지점`). Aim for 12 Korean characters or fewer per theme title.

A title is not just a heading: it is reused verbatim inside the TOC and inside the quiz hint button (`{title} 섹션 →`). A sentence-shaped title like `한 번의 sweep으로 끝나지 않았어요` makes that button eat an entire line, and a long title wraps mid-word in the hero. Write `sweep이 놓친 3곳` instead.

Structure is fixed at three parts:

**Top + TOC.** The hero title is a **noun phrase naming the change topic**, not an action sentence: write `모델명 prefix 버그와 수정 방향`, not `모델명에 upstage/ 접두가 붙어 채팅이 막혔어요`. Keep it to ~20 Korean characters so it holds one line; the lede (2-3 sentences) carries the what/why, so the title does not need to. Metabar: target, commit range, file count, +/- totals, date. Each TOC row's description is that section's **one-line takeaway** so reading only the TOC summarizes the whole change; keep it a short phrase, no dash. Keep the Part group headers.

**Part 1: 개요.**
- `배경`: the before-state and its cost, then what the change achieves. Behavior level, not file level.
- `구조 한눈에 보기`: annotated directory tree of every touched path. Chips state *what happened* (`정본으로 승격`, `67줄 → 1줄`, `삭제 (렌더러)`), deleted files get strikethrough. No section-jump links on tree rows.
  - **It is a real tree, not a flat list of full paths.** A row shows only its own name, indented under its parent directory. Never emit `├─ graph/workflows/file_indexing.py` next to `├─ graph/models/llm/utils.py`: the shared `graph/` becomes one directory row, and the children nest under it. If every row starts at the same depth, the tree is wrong.
  - Build the guide prefix per row (see the template comment): `│  ` for each ancestor that still has siblings below, `   ` for closed ancestors, then `├─ ` or `└─ ` (last sibling) for the node itself. The last row of the whole tree therefore begins with `└─ `, and every deeper row above it carries `│` in the ancestor columns.
  - Collapse a chain of single-child directories into one row (`openspec/changes/split-embedding/`), and collapse siblings that changed the same way into one leaf with a brace glob (`config.{prod,beta,kr}.yaml`, `tests/unit/{graph,app}/…`) with a chip like `8파일 × 2블록`. Directory rows carry a chip only when the whole directory was added or deleted.
  - Order children directories-first, then files, alphabetically, so the shape stays scannable. Depth beyond 4 levels is a signal to collapse, not to indent further.
  - The tree is always visible at page load and is never wrapped in a `<details>` disclosure or replaced by summary cards. Keep the tree header text `디렉토리 변화` as-is (the repo root is the first tree row, not the header). If you need a disclosure elsewhere, use `<details class="fold">` so it inherits the template's typography instead of the browser default marker and font.
  - Trees longer than 20 rows fold automatically: the template JS caps the body at 20 rows with a fade and a `나머지 n개 행 펼치기` button that animates open and closed. Do not hand-build this, and do not trim or reorder real rows to dodge it; the collapse rules above are the way to keep a tree short.
- Optional interactive figure: include ONLY when it passes the test in "Interactive figure" below. If nothing in this diff passes, omit the figure entirely.

**Part 2: 변경 사항.** One card per theme from Step 2, ordered by importance:
- Prose explaining behavior meaning: what the reader must understand, not a line-by-line narration.
- Optional flow figure (see "Flow figure" below), between the prose and the diff excerpt, when the theme changes who calls what, in what order, or how many times.
- Verbatim diff excerpts (12 lines max per block; elide the middle with a `.ln.gap` line `… N줄 생략`). Only the lines that carry the theme. A flow figure never replaces the excerpt.
- Close every card with a `핵심 정리` takeaway card: one bold sentence to remember + 2-3 supporting bullets. This is the same visual language as quiz explanations: blue tinted card = "the thing to remember".

### Flow figure (optional, Part 2)

A diff shows which lines changed; it does not show what the system did before. When a theme changes **who calls what, in what order, or how many times** (a call moved to another layer, a retry added, N requests collapsed into one, a branch that now fails differently), draw it:

- Two rows of the same path, the state before the change above the state after it, captioned as the State captions item recommends (a change that has happened reads well as `BEFORE` / `AFTER`), on the same columns so the one difference lines up. What the change removed or what used to fail is `--red` in the top row; what the change added is `--blue` in the bottom row. Everything that did not change is drawn identically in both rows.
- It sits between the prose and the diff excerpt. The figure shows what differs; the excerpt shows which lines cause it. Keep both.
- Counts and names come from the code, as in the prose: three requests are three lines, not "several".
- Omit it when the theme changes a value, a name, a type, or a message: there is nothing to draw. Never add a figure so that every card has one.

Drawing rules (shared with `eli5` and `visual-doc`):

- Hand-authored inline SVG inside `<figure class="fig">`, sized by `viewBox` (CSS `width:100%; height:auto`), with `role="img"` and an `aria-label` that states the figure's claim.
- Draw the mechanism, not a row of labeled boxes: the path a request takes, the edge that disappears, a count drawn as that many lines. Label every arrow (`재발급 1회`, `20장씩`); an unlabeled arrow only says "related somehow".
- Real names in the boxes (the function, table, or endpoint as the code calls it), with at most one plain sub-line under the name. A line in a box is a name or a short field list, never a sentence.
- Strokes and text use `currentColor` so both themes work. `var(--blue)` marks the one element that carries the point, `var(--red)` the thing that fails. Two accent colors per figure at most.
- Text uses the label classes shipped in the template, copied verbatim (the block that starts with the `figure labels (SHARED` comment): `.dg-l` 13px, `.dg-s` 11px mono, at most one `.dg-b` big number, `.dg-halo` for a label that sits on a line, `.dg-em` and `.dg-bad` for the two accents.
- **State captions and the focal string, when rows compare states.** (optional; most figures use neither) A figure whose rows show the same thing in different states may give each row a header line above it: a pill `<rect class="dg-pill" height="22" rx="11">` at the row's left edge with a `.dg-cap` label centered on it, both carrying the same modifier, then a `.dg-s` sub-line after the pill saying what that state concretely does. The header line takes its own height in the layout plan, not a caption column. Pick the caption by what the rows compare; these pairs are recommendations, not a fixed list. One change that has already happened reads well as `BEFORE` / `AFTER`; one change that is proposed or planned, as `AS-IS` / `TO-BE`. Rows that are not one change over time (two cases, two modes, two inputs) read well when each is named by what is true in that row, a short noun phrase in the subject's terms. Use `.bad` for the state that breaks, `.em` for the state carrying the point, and no modifier for a state that is neither: a red caption claims the state is wrong. When the point of the figure is the text itself (a search query, a command, an error message), that one kind of string may add `.dg-q` to its `.dg-l` (15px bold; `.dg-em`/`.dg-bad` still color it); every other label stays at the regular sizes, and the pills count toward the two accent colors.
- No `<script>`, `<style>`, or `<foreignObject>` inside the SVG, no emoji, no external resources.

#### Fit the text before you draw (no renderer needed)

SVG text does not wrap and does not push boxes apart. Every clipped label and every pair of overlapping labels in a shipped page came from a label that was wider than the room drawn for it. So **size the room from the label, never the label from the room**. Plan with the per-character widths below, then measure: when a browser exists, the same `measure.html` run reports `data-fit`, every label that leaves its viewBox, crosses a box edge, or overlaps another label in the rendered font (number badges excluded), and the page ships only when it prints `none`. The measurement wins over the table both ways: a label the table says fits but the render says crosses is a defect, and a gap the table made wider than the measured text may be tightened. The widths are Chrome measurements rounded up, deliberately generous for the system fonts the page will be read in (Apple SD Gothic Neo, Pretendard, Segoe UI); a line that passes them fits, and with no browser they are the only check.

| Text | Width per character |
|---|---|
| Hangul, in any class | 1.0 × font-size (`.dg-l` 13px, `.dg-s` 11px) |
| `m` `w` `M` `W` in `.dg-l` | 1.0 × font-size (13px) |
| other Latin lowercase, digits, `_` `-` `/` `.` in `.dg-l` | 0.6 × font-size (8px) |
| Latin uppercase in `.dg-l` | 0.75 × font-size (10px) |
| space, `(`, `)`, `:` in `.dg-l` | 5px |
| every non-Hangul character in `.dg-s` (mono), spaces included | 7px |
| anything not listed (`×`, `→`, `=`, `+`, `…`) | 1.0 × font-size |
| `.dg-q` (optional focal string, 15px bold) | Hangul 15px, `m` `w` and Latin uppercase 14px, every other non-Hangul character 10px, space 5px |
| `.dg-cap` (optional state caption, 12px bold) | Hangul 13px, Latin 10px, space 5px; its pill is the text + 16 wide |

Line height: 18px per `.dg-l` line, 15px per `.dg-s` line, 20px per `.dg-q` line, 26px for `.dg-b`. A `<tspan>`'s `dy` is the height of the line **above** it (a `.dg-s` sub-line right under the `.dg-l` name has `dy="18"`; the next `.dg-s` line has `dy="15"`), so the box-height formula below and the `dy` values agree.

1. **Box from label.** Take the widest line of the box, estimate it, add 28 (14px padding each side), round up to a multiple of 10: that is the box width. Height is 14 + (sum of line heights) + 14; that is a minimum, and boxes in one row share the tallest height. First-line baseline: `y = box top + 25` (14 padding + the 13px name's ascent); each later line moves by its `dy`. In a box taller than the minimum, center the block instead: `y = box center − (Σ dy)/2 + 4`. A box is never narrower than its text; if the result is wider than you want, shorten the text.
2. **Lines grow the box, never the other way.** A box holds a name line and as many sub-lines as the content needs, each its own `<tspan x="{box center}" dy="…">`; the height formula absorbs them. Do not cut a field list to make a box small: a schema box that hides half its columns is a wrong drawing, not a tidy one.
3. **Row budget.** For every row: 20 + (caption column + 40, when rows carry a left-hand caption; size that column from its widest caption line like any other label) + Σ box widths + Σ gaps + 20 ≤ viewBox width. Gaps are at least 40; a gap that carries an arrow label is at least the label estimate + 24. When a row does not fit, shorten labels first, then break it into two rows, then (last) widen the viewBox up to 720. Never above 720: the figure is about 670px wide on screen, so a wider viewBox shrinks 13px text below 12px. Height is unlimited: more boxes mean more rows, not smaller text.
4. **Nothing crosses a border.** Every box label is `text-anchor="middle"` at the box center, and rule 1 guarantees it stays inside. Free-standing text (edge labels, row captions, the big number) gets a clear zone: no other text, no line other than the one it labels, and no box edge within 6px of its estimated extent, and that extent stays inside `[20, W-20]`. Vertically, a line of text occupies baseline −12 to +3 for `.dg-l`, −10 to +3 for `.dg-s`, −14 to +4 for `.dg-q`, −18 to +4 for `.dg-b`; use those for the clear zone, not just the horizontal estimate.
5. **Edges never pass through a box.** A line that crosses a box it does not connect is a defect, whichever direction it runs. When an edge has to skip a row, route it down a lane between boxes and turn below that row. Two edges that end on the same box land at least 20px apart along that side (or merge into one elbow before arriving). Give each `<svg>` its own marker id (`ar1`, `ar2`, …): a page has several drawings and duplicate ids are invalid.
   Preferred, not required: horizontal and vertical segments with an elbow (`M x1 y1 V ymid H x2 V y2`) rather than a diagonal, and the label on a straight segment: baseline 10px above a horizontal segment with `text-anchor="middle"`, or beside a vertical one with `text-anchor="end"` at `x = line x − 8`, in class `.dg-s`. If the two endpoints differ by less than 12px, draw the edge straight. A diagonal is fine when it is the clearer line; then its label carries `dg-halo` so it stays readable where it crosses. Rows stacked above each other are at least 40px apart (box bottom to next box top) when an edge label sits between them, 30px otherwise.
6. **Text comes last.** Inside each `<svg>`, write every `<rect>`, `<line>`, and `<path>` first and every `<text>` after them. SVG paints in document order, so a line written after a label is drawn over it, and `dg-halo` only works when the text is painted on top.
7. **Plan first, as a comment.** The first child of every `<svg>` is a one-line layout plan, and writing it is the check:
   `<!-- W=720 | row1 y=56 h=88: [40..180] gap40 [220..360] gap40 [400..540] gap40 [580..700] -->`
   Every range is closed, adjacent ranges differ by exactly the gap, the last end is ≤ W-20. If the arithmetic does not close, the drawing does not ship.
8. **No ceiling on boxes.** A subject with twelve tables gets twelve boxes in one drawing if that is what the reader needs to see at once; the row budget and the 720px width are the only limits, and height grows with rows. Split into two drawings only when the subject itself has two stories (a current path and an old path).

### Interactive figure (optional, Part 1)

- **Static first.** Two fixed states (before/after, on/off, hit/miss) are drawn as two rows of one static figure: the eye compares both at once, and a toggle would hide one while showing the other.
- **Interactive only when the reader has to move a value to see the point.** That means the outcome changes across three or more values of one input (a count, a size, a rate, a threshold), or the point is the value at which the behavior flips (`요청 1개면 거부 없음, 2개부터 거부`). Then one or two inputs redraw the figure.
- **Three questions before building.** What is the input? What moves in the drawing when it changes? What does the reader learn at a value other than the default? If the third has no answer, draw it static.
- **The default state already makes the point.** A skimmer, a screenshot, and a printed page see only the state at load.
- **Never a step-through** that reveals already-visible content in order, and never an input that only shows or hides text.

Build it inside the template's `.sim` container with a `.seg` switch as the input. What the input redraws is a `<figure class="fig">` SVG (drawing rules and fit arithmetic from "Flow figure" above, checked for the smallest and the largest input); `.sim-note` is the readout. Do not build it from `.sim-grid` / `.sim-step` text lists alone: that is an input that only shows or hides text. Inline the bespoke JS at the bottom of the page; it computes the outcome with the code's real rule and builds shapes first, labels last.

**Part 3: 이해 점검.**
- `주의해서 볼 지점`: risk rows with Watch/Critical badges. Attention pointers, never verdicts. If a risk deserves a verdict, tell the user to run a code review: do not deliver one here.
- Quiz (below).

## Quiz Specification

Exactly **5 questions** by default, medium-hard. Each question must test practitioner-critical understanding: something that would change how the reader reviews, merges, or maintains this code:

- behavior prediction ("if X happens, what does the system actually do now?")
- failure modes ("which path is still unprotected?")
- rejected alternatives ("why was Y not chosen?": only when the record/PR states it)
- invariants ("what does the check actually guarantee, and what not?")
- future-maintainer scenarios ("six months later, someone does Z: what breaks?")

Trivia (line counts, file names, dates) is banned. Distractors must be **plausible misconceptions**: the old system's behavior, a wrong-layer attribution, a swapped condition: never obviously absurd fillers.

Mechanics (already implemented by the template JS: supply only content):

- Exactly one option per question has `data-ok="true"`.
- `data-hint` points to the section to re-read and **never reveals the answer**.
- `data-hint-href` is that section's own id (`#sec2`, `#risks`, …) and `data-hint-label` its title: together they turn the hint into a one-click jump, so the reader never scrolls back by hand. The id must exist in the document; the template drops the link if it does not resolve. Omit both only when no single section covers the question.
- Wrong answer → hint + retry (that question only); correct → explanation + lock.
- Explanation card: one bold core fact + bullets, including why the tempting wrong answer is wrong.
- The bottom gate opens only at 5/5. Its wording states understanding: never a fake action the page cannot perform (no "PR 보내기" buttons).

## Step 4: Verify the Render

### Render check (shared across eli5, explain-diff, micro-world, visual-doc)

The rules are the first defense; looking at the rendered page is the second, and a defect counts as fixed only when the render shows it fixed. Work in a scratch directory of your own (`D`, e.g. `/tmp/explain-diff-<slug>`); never a shared fixed path, because other runs may be writing there at the same time. Find a browser first (`which google-chrome google-chrome-stable chromium chromium-browser`). Do not rely on `npx --no-install playwright screenshot`: with no browser binary installed it prints an install banner, exits 0, and writes no file.

1. **Screenshot both themes.** The window must be taller than the page; a page cut at the bottom looks finished, so raise the height if the last block is missing.
   ```bash
   google-chrome --headless=new --no-sandbox --disable-gpu --hide-scrollbars --window-size=1300,12000 --screenshot=D/light.png "file://<abs-path>"
   google-chrome --headless=new --no-sandbox --disable-gpu --hide-scrollbars --force-dark-mode --window-size=1300,12000 --screenshot=D/dark.png "file://<abs-path>"
   ```
2. **Look at every piece.** Cut each screenshot into pieces about 1,800px tall and read them all; a page-sized thumbnail hides a clipped label, a chip split across two lines, and a lone word on a last line.
   ```bash
   python3 -c "from PIL import Image; im=Image.open('D/light.png'); [im.crop((0,y,im.width,min(y+1800,im.height))).save(f'D/light-{i}.png') for i,y in enumerate(range(0,im.height,1800))]"
   ```
   If Pillow is not installed, look at the full screenshots and say in the report that the check was at thumbnail size.
3. **Measure.** Run `measure.html` from this skill's base directory (the command is in its header comment) at the reading width and again at `--window-size=500,2000` (headless Chrome does not go narrower than 500px, so a 420px request silently measures 500). The `<body>` tag must carry `data-sbs`, `data-fit`, and `data-ovf`, all `none`; a body tag without them means the measurement never ran, which is a failure, not a pass.
4. **Text.** `grep -nP '[—–·]'` and the particle grep from the design rules print nothing outside verbatim source.

Fix and repeat until all four pass. Only when no browser exists at all, say plainly that the page was not visually verified. Screenshots, crops, and the measurement copy live in `D`, are not part of the output, and are deleted when you finish.

Then check: every quiz option reveals its explanation and the gate opens; the diff lines keep their `+`/`-` signs; the tree collapses past 20 rows; dark mode holds.

## Step 5: Output

1. Write the file to the **repository root**: `explain-diff-<slug>.html` (slug from the change topic, kebab-case).
2. Report the **absolute path** to the user. Do NOT auto-open it, do NOT commit it, do NOT add it to .gitignore. Delete it only when the user asks.
3. Content language follows the project's AGENTS.md (or CLAUDE.md as fallback) language setting; if none, the user's conversational language. Korean output uses 해요체 throughout (headings may stay nominal).

**Constraints:**

- This skill is read-only with respect to git state: it creates exactly one HTML file (plus the scratch files of the Render check, deleted when it finishes) and changes nothing else.
- Design decisions are fixed by the template and the design-system record (`docs/decisions/2026-07-explain-diff-design-system.md` in the git-claw repo). Do not improvise styles.
