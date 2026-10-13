---
name: eli5
description: >-
  Explain a subject to someone who knows nothing about it, as a self-contained HTML page where big diagrams carry the meaning and the prose is a short factual paragraph per picture.
  TRIGGER when: user types /eli5 <topic>, or asks for a dead-simple picture explainer aimed at someone outside the domain (e.g., "eli5 해줘", "아무것도 모르는 사람한테 설명하듯 만들어줘", "그림으로 쉽게 풀어줘", "비전공자한테 보여줄 자료로 만들어줘").
  DO NOT TRIGGER when: the reader is the developer who has to merge the change (use explain-diff), the subject is a behavior worth inhabiting interactively (use micro-world), the reader is domain-literate and wants the findings structured (use visual-doc), or a code review verdict is wanted (use code-review).
version: "1.4.0"
allowed-tools: Bash(git *), Bash(gh *), Bash(npx *), Read, Grep, Glob, Write
---

## The One Principle

**The pictures carry the explanation; the words only name what was just seen.** A reader who scrolls through this page looking at nothing but the drawings must still come away with the idea. If a sentence is doing the explaining and the drawing is illustrating the sentence, the drawing has failed and must be redrawn.

This is a sibling of `explain-diff`, `micro-world`, and `visual-doc`: they share design tokens, never structure. What separates this one is **the reader**, and the reader decides everything:

| Skill | Reader | Succeeds when |
|---|---|---|
| `explain-diff` | the developer who must merge it | they pass the comprehension quiz |
| `micro-world` | someone who learns by manipulating | they drive the scenario and feel the behavior |
| `visual-doc` | a domain-literate colleague | the findings are structured and navigable |
| **`eli5`** | **someone outside the domain entirely** | **they get it from the pictures alone** |

Because the reader is an outsider, the hard work is **subtraction**: choosing the few things that matter and drawing them, not compressing everything that exists. A complete-but-dense page is a failure here even though it would pass as a `visual-doc`.

Honesty rules apply throughout:

- Simplify by **omitting**, never by stating something false. "It works roughly like this" is fine; a wrong mechanism drawn confidently is not.
- Real numbers only, quoted from the source. Never invent a figure to make a bar chart look better.
- Do not flatten a genuine tradeoff into a happy ending. If the change costs latency or money, that beat stays in.

## Parse Arguments

| Argument | Meaning |
|----------|---------|
| (none) | Explain the subject already discussed in this conversation |
| a topic / question | Explain that subject ("how does DNS work") |
| a PR number / commit / branch | Explain what that change does and why, for an outsider |
| an issue number | Explain the problem the issue describes |
| a path (e.g. `src/auth/`) | Explain what that code does |

For a PR or issue, read it properly first: `gh pr view <n>`, `gh issue view <n>`, the diff, the files it touches. The page is only as good as what it is built from, and a PR body's own summary is usually written for insiders.

## Step 0: Applicability Gate

This skill needs **a mechanism worth drawing**: something moves, splits, fails, gets chosen between, or changes shape. That covers most architecture, algorithms, protocols, incidents, and design tradeoffs.

It has nothing to draw for a dependency bump, a rename, a formatting pass, or a config flip. Say so plainly ("이건 그림으로 얻는 게 거의 없어요") and offer `/explain-diff` instead. Do not manufacture filler pictures.

Also stop and hand off when **the reader is not actually an outsider**. If the person asking is the one shipping the change, they want `explain-diff`.

## Step 1: Find the One Idea and the Outsider

Before drawing anything, write down three things for yourself:

1. **The one idea.** A single sentence a non-expert could repeat afterward. Everything on the page serves it; anything that does not is cut.
2. **Who the outsider is.** A designer, a PM, an engineer from another team, a family member. This sets how much can be assumed and which analogies land.
3. **The confusion to dissolve.** The specific thing that makes this subject hard for that person.

If you cannot state the one idea in a sentence, you do not understand the subject well enough to simplify it yet. Go read more.

## Step 2: Storyboard Before You Draw

Plan the page as **a sequence of pictures**, each with a one-line heading, before writing any HTML. Five to eight beats is the usual range; fewer is fine, more usually means two pages.

The default arc, which fits most technical subjects:

1. **The problem** — draw the thing failing or blocked. Start here, never with the solution.
2. **The obvious fix** — the naive approach a reasonable person would try.
3. **The trap** — why the obvious fix breaks. **This beat is the point of the whole page.** It is also the beat most likely to be buried in one line of a table in the source material, so dig for it.
4. **The real fix** — the mechanism actually chosen, drawn as a mechanism.
5. **The cost** — what it takes in time, money, or complexity. Never omit.
6. **The rule** — the one thing a reader should remember about when this applies. On the page this is the `.close` card at the very end, with no heading above it: the statement is the heading, and a heading plus a card title would say the same thing twice.

Adapt the arc when the subject is not a change (a "how does X work" page may go: what goes in, what happens to it, what comes out, what breaks). Keep the failure beat wherever it exists.

**One picture, one claim.** If a drawing needs two headings, it is two drawings.

### Titles say what the picture is of

The arc above (problem, obvious fix, trap, real fix, cost, rule) names the **role** of each beat for you. It is not the heading. Headings, the hero title, row captions, and sub-lines are **facts about the subject, stated plainly**. 개조식 is welcome.

- **Hero title**: the subject by its real name, as a noun phrase. `vector_store 테이블 10개의 구조`, `파일 업로드가 청크가 되기까지`. Not `세 줄로 읽는 벡터 저장소`, not `보이지 않는 선반`.
- **Beat heading**: the fact the picture shows. `장부 테이블 3개, 내용물 테이블 1개`, `standard 모드: LLM 호출 1회`, `description이 비어 있는 경로 3곳`. Never a tease (`함정`), a rhetorical question, or a metaphor (`마법은 없다`).
- **Row captions and sub-lines**: the literal word first (`원본 테이블`, `장부`, `본문과 벡터`). A metaphor may appear once per page, in a `.say` line, when it carries a meaning the literal word cannot; it never becomes a heading.
- **No translated idiom.** `~의 여정`, `숨은 영웅`, `마법처럼`, `~를 만나다`, `한눈에 보는 ~` are machine-translation tells. If a phrase would sound odd said aloud to a colleague, cut it.
- **The portability test**: if a heading could sit unchanged on a page about a different subject, it is too abstract. `문제`, `진짜 해법`, `드는 것과 안 드는 것` all fail; `마이그레이션 0건, 새 엔드포인트 1개` passes.

## Step 3: Draw the Mechanism

The drawings are hand-authored inline SVG, one per beat.

- **Depict the mechanism, not its name.** A box labeled "cache" says less than the prose. Draw the path a request takes, the two stores it sits between, the arrow that disappears when the cache is gone.
- **The cover test, before anything else.** Cover every piece of text in the drawing. If what is left says nothing (same-looking boxes joined by same-looking lines), you drew a labeled list, not a mechanism; redraw before you lay anything out. Boxes-and-lines alone is right only when the relationships *are* the content (a table structure, a dependency graph).
- **Draw the behavior with these shapes.** Most mechanisms are one of a few things, and each has a drawing:
  - *one becomes many*: one shape on the left, a stack of same-shaped cards on the right, a line from each slice of the source to its card, the cut marks drawn on the source
  - *repeat up to N*: a loop drawn back over the box it re-enters, with round markers `1 2 3 … N` on the loop
  - *the same thing comes back*: the same token shape (`A`) drawn where it is kept and where it arrives, so the eye matches them
  - *hit or miss, before or after*: two rows of the same path, one difference between them; what is skipped is drawn dimmed, not removed
  - *a choice*: the path forks; the branch not taken is drawn dashed and dim
  - *a failure*: the line stops short with a red mark where it stops; the unreachable part is dashed
  - *cost or size*: lengths and counts drawn to scale (one wide bar vs three narrow ones), never a number alone
  The arithmetic rules below position these shapes; they do not replace them. A row of labeled boxes is the fallback, not the default.
- **Comparing options? Draw the difference.** Two states side by side with the one edge that changes between them. Separate labeled boxes with nothing connecting them is a restated list, not a comparison.
- **Label the arrows.** An unlabeled arrow means "related somehow". `writes`, `20장씩`, `19배 압축` is information.
- **Concrete by default.** Draw the specific things, not categories of things: eight named tables in their three rows beats three ovals labeled 입력 / 처리 / 저장. If you catch yourself drawing "data flows to storage", name the data and name the storage. Abstract shapes are for subjects that have no artifacts (a concept, a policy), not a shortcut for subjects that do.
- **Real names, plain sub-line.** When the subject is a system, a box is the real thing under its real name: the table `vector_store_file`, the endpoint `POST /files`, the flag `use_doc_chunking`. Under the name, one sub-line says what it is for an outsider (`장부 1줄`, `실제 본문과 벡터`). The sub-line is literal too; the one place a metaphor may appear is a `.say` line (see "Titles say what the picture is of"). What stays out of the drawing is **explanation**: the why, the mechanism in prose, file paths, line numbers. Those go in `.say` or your report.
- **Size by `viewBox`** (`viewBox="0 0 W H"`, CSS `width:100%; height:auto`), pick W and H for the content, and align shapes to a shared grid. Eyeballed offsets read as noise.
- **Theme with `currentColor`.** Strokes and text inherit the page's foreground so both themes work. Reserve `var(--blue)` for the one element carrying the point of that picture and `var(--red)` for the thing that breaks. Two accent colors per drawing at most.
- **Text stays at the two label sizes** (`.dg-l` 13px, `.dg-s` 11px mono) plus at most one `.dg-b` big number per drawing. A box holds its real name and **at most one sub-line**, never a sentence. The sub-line is either the one fact the reader needs (`statement_timeout 2000ms`) or one example (`"보험약관 암보험 보장 내역"`), never both, and never a list of fields from the source (`filters eq · tool_choice`, `failed_stores [vs_…]`): detail of that kind goes to the notes under the figure or to an evidence card. (A schema box in an ERD-style drawing is the exception: its lines are the columns.) Fit every line to its box with the arithmetic in "Fit the text before you draw" below.
- **Every figure gets** `role="img"` and an `aria-label` stating the same claim as its heading. (An interactive figure points `aria-labelledby` at its `그래서` line instead, because the claim changes with the input.)
- No `<script>`, `<style>`, or `<foreignObject>` inside the SVG (the script of an interactive figure sits at the end of `<body>`; see "An interactive figure" below). No emoji, no icon fonts, no external resources. Long decorative path data means the drawing is too elaborate: simplify it.

### Fit the text before you draw (no renderer needed)

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
3. **Row budget.** For every row: 20 + (caption column + 40, when rows carry a left-hand caption; size that column from its widest caption line like any other label) + Σ box widths + Σ gaps + 20 ≤ viewBox width. Gaps are at least 40; a gap that carries an arrow label is at least the label estimate + 24. When a row does not fit, shorten labels first, then break it into two rows, then (last) widen the viewBox up to 840. Never above 840: the figure is about 780px wide on screen, so a wider viewBox shrinks 13px text below 12px. Height is unlimited: more boxes mean more rows, not smaller text.
4. **Nothing crosses a border.** Every box label is `text-anchor="middle"` at the box center, and rule 1 guarantees it stays inside. Free-standing text (edge labels, row captions, the big number) gets a clear zone: no other text, no line other than the one it labels, and no box edge within 6px of its estimated extent, and that extent stays inside `[20, W-20]`. Vertically, a line of text occupies baseline −12 to +3 for `.dg-l`, −10 to +3 for `.dg-s`, −18 to +4 for `.dg-b`; use those for the clear zone, not just the horizontal estimate.
5. **Edges never pass through a box.** A line that crosses a box it does not connect is a defect, whichever direction it runs. When an edge has to skip a row, route it down a lane between boxes and turn below that row. Two edges that end on the same box land at least 20px apart along that side (or merge into one elbow before arriving). Give each `<svg>` its own marker id (`ar1`, `ar2`, …): a page has several drawings and duplicate ids are invalid.
   Preferred, not required: horizontal and vertical segments with an elbow (`M x1 y1 V ymid H x2 V y2`) rather than a diagonal, and the label on a straight segment: baseline 10px above a horizontal segment with `text-anchor="middle"`, or beside a vertical one with `text-anchor="end"` at `x = line x − 8`, in class `.dg-s`. If the two endpoints differ by less than 12px, draw the edge straight. A diagonal is fine when it is the clearer line; then its label carries `dg-halo` so it stays readable where it crosses. Rows stacked above each other are at least 40px apart (box bottom to next box top) when an edge label sits between them, 30px otherwise.
6. **Text comes last.** Inside each `<svg>`, write every `<rect>`, `<line>`, and `<path>` first and every `<text>` after them. SVG paints in document order, so a line written after a label is drawn over it, and `dg-halo` only works when the text is painted on top.
7. **Plan first, as a comment.** The first child of every `<svg>` is a one-line layout plan, and writing it is the check:
   `<!-- W=760 · row1 y=56 h=88: [40..190] gap45 [235..385] gap45 [430..580] gap40 [620..720] -->`
   Every range is closed, adjacent ranges differ by exactly the gap, the last end is ≤ W-20. If the arithmetic does not close, the drawing does not ship.
8. **No ceiling on boxes.** A subject with twelve tables gets twelve boxes in one drawing if that is what the reader needs to see at once; the row budget and the 840px width are the only limits, and height grows with rows. Split into two drawings only when the subject itself has two stories (a current path and an old path), each with its own `.say`.

### An interactive figure, only when a value has to move

Every figure is static unless the subject passes this test (the same test as in `visual-doc` and `explain-diff`):

- **Static first.** Two fixed states (before/after, on/off, hit/miss) are drawn as two rows of one static figure: the eye compares both at once, and a toggle would hide one while showing the other.
- **Interactive only when the reader has to move a value to see the point.** That means the outcome changes across three or more values of one input (a count, a size, a rate, a threshold), or the point is the value at which the behavior flips (`요청 1개면 거부 없음, 2개부터 거부`). Then one or two inputs redraw the figure.
- **Three questions before building.** What is the input? What moves in the drawing when it changes? What does the reader learn at a value other than the default? If the third has no answer, draw it static.
- **The default state already makes the point.** A skimmer, a screenshot, and a printed page see only the state at load.
- **Never a step-through** that reveals already-visible content in order, and never an input that only shows or hides text.

On an eli5 page, additionally:

- At most one interactive figure per page. It replaces the static figure of its beat; it is not an extra picture.
- Controls are the `.ctl` row from `frame.html` (a range with its `<output>`, or a `.seg` switch), placed above the `.fig`. One or two inputs. No free-text input, no play button.
- The script is one `<script>` at the end of `<body>` that redraws a `<g>` inside the figure; nothing goes inside the `<svg>` element itself. It computes the outcome with the subject's real rule (the honesty rule covers it: no invented curve) and builds shapes first, labels last.
- The drawing rules do not relax. Do the fit arithmetic for the smallest and the largest input; the layout-plan comment describes the widest state.
- The `그래서` line under the figure is the readout: the script rewrites that one `.say` (keep its `lab`, `aria-live="polite"`, two sentences at most) to say what happens at the current value, and `aria-labelledby` on the figure points at it. The notes and badges stay static; they describe the parts, not the current value. The default value is the one that shows the page's point.
- A subject that needs several steps in order, a mission, or state that carries over between actions is a `micro-world`, not a figure. Say so and hand off.

### The word budget

- Under every figure come the **notes**: a `.notes` list with one `.note` per numbered badge in the drawing, each `badge + the thing's name in bold + one or two plain sentences` saying what that part is and the one condition the drawing could not show. Then **one** `.say` line that starts with the label `그래서` (`<span class="lab">그래서</span>`): the conclusion of the beat in one or two sentences. When something stayed unverified, one more `.say` line labeled `확인 못 한 것` (`lab warn`) goes before it, and `[추정]` never hides inside a sentence. Facts only, in the subject's own words, body size. Do not repeat the heading, do not repeat what a label in the picture already says, do not add background. A figure with no badges (a number strip, a single-row picture) keeps a plain `.say` of two or three sentences instead of notes; nothing goes inside the card (no caption).
- **Numbered badges tie the three places together.** A badge is the same shape everywhere: `<g class="dg-m">` (circle + number) on the top-left corner of a box in the drawing, `<span class="m">` on the left border of an evidence card beside the line it marks, and at the head of the matching `.note`. Blue is the default, `bad` (red) for the thing that breaks, `dim` for a part not taken. Number in reading order of the drawing, restart at 1 in every beat, and never badge what has no note.
- **A badge earns its place; it is not an index.** Badge a box only when the note says something the box label cannot (a limit, an error code, a condition) or when the name alone could not point the reader at it. When every box in a drawing carries a badge, the numbers stop highlighting and start counting; merge the notes that describe one action (an LLM call and its output are one step, not two) until some boxes stay unbadged. A drawing that is itself a list (three decision rows one under the other, each with its own title) gets no badges at all: the order is the number, and its notes open with the row's title in bold (`.notes.plain`, no `.m`), in the same order as the rows.
- **One note for several boxes means one group badge.** When a note is about several same-shaped boxes at once (every store in a fan-out, the three results that are noise), never badge the first box and leave the rest bare; the reader takes the badge to mean that one box. Draw a dashed group outline (`<rect class="dg-grp">`, 8px outside the boxes on every side, `rx` 10, `bad` for a red group) before the boxes, give the boxes `fill="var(--card)"` so they sit on top of it, and put the one badge on the group's top-left corner. The note then names the group with its count (`store 5개`, `무관 결과 3묶음`). Arrows may cross the group outline; the badge never does.
- Prose chips: a `<code>` chip in a note names a thing the reader can see in the drawing or in an evidence card of that beat; a chip that appears nowhere else on the page is noise and becomes plain words.
- **One name per thing, the actor as subject, the count as a number.** (shared across eli5, explain-diff, micro-world, visual-doc) Pick one name for each thing and each action and keep it everywhere on the page, labels and drawings included: once it is `액세스 토큰` it is never `인증 토큰`, `세션 키`, or `자격 증명`; once it is `재발급` it is never `갱신`. A second name reads as a second thing. When two different things share a word, qualify both (`API 서버`, `인증 서버`, never a bare `서버`). Make the subject the thing that acts: `인증 서버가 두 번째 요청을 거부했어요`, not `두 번째 요청이 거부됐어요`; a sentence that hides the actor hides where the cause is. Write the count the source gives (`요청 3개`, `3번`), not `여러 개` or `요청 수만큼`.
- **A particle is part of the word before it; a count stays on one line.** (shared across eli5, explain-diff, micro-world, visual-doc) Attach every particle and ending to the word before it, also after an English word, a number, or a code chip: `Job은`, `store에도`, `API와`, `400이`, `<code>job_indexing.mode</code>가`, `automatic인`. Never `Job 은` or `</code> 을`: where `word-break: keep-all` applies a line breaks only at a space, so that space lets the particle start a line. Choose the particle by how the word is read aloud (`Job은`, `API는`, `store를`). Write a digit and its unit together (`3개`, `0행`, `1회`), and put `&nbsp;` between a native numeral and its unit (`한&nbsp;번`, `두&nbsp;줄`, `세&nbsp;가지`) so the line never breaks inside the count. Bind nothing else with `&nbsp;`: each one makes a longer unbreakable run and a more ragged right edge. Before finishing, run `grep -nP '[A-Za-z0-9_)]+(</code>)? (은|는|이|가|을|를|의|에|와|과|로|으로|도|만|에서|에는|에도|인|인데)(?=[ ,.<)]|$)'` on the file and fix every hit that is a particle (`이` and `가` can also be a word of their own).
- No paragraphs of body prose anywhere on the page. If a beat genuinely needs three paragraphs, the subject wants `visual-doc`.
- There is no `figcaption`. A fact that used to go in a caption goes in the `.say` if the reader needs it to read the picture, and is dropped otherwise.
- **File paths and line numbers stay off the page.** An outsider will never open `app/models/store.py:103`. The names of the things drawn (a table, an endpoint, a flag) are not evidence of that kind; they are the nouns of the subject and belong in the drawing (see "Real names, plain sub-line" above). A name the drawing does not show may still appear once, in that beat's notes, when the reader will actually meet it. The one place a file name is allowed is the head of an evidence card (`config/config.onprem.yaml`), because there the file itself is the thing shown; line numbers stay off even there.
- **No footer, no source list.** File paths, line numbers, and the list of what you read are evidence for the person who asked, not content for the reader. They go in your report (Step 6), never in the HTML.
- **Provenance is one line in the hero eyebrow**: the source and a date, e.g. `studio-pipeline · 2026-09-28`. Nothing else goes there: no commit hash, no "기준", no status word. If there is no meaningful source (a general "how does DNS work"), the eyebrow is just the date.
- Numbers appear as before/after pairs in the number strip, four at most, and at least one of them is a cost.

### Evidence card, only when the drawing cannot say it

A drawing shows mechanism; it cannot show that a line is **absent** from a file, which **columns** a table has, or that a field is **missing** from a request. When the beat's claim is one of those, put an **evidence card** (`.ev`) between the figure and the notes: the real thing in its own shape, quoted verbatim, with the badge on the line that matters. The card is optional, and the test is strict: if the notes could state the fact and the reader would believe them, there is no card. A page where every beat earns one may carry one on every beat; a page where none does carries none. Never add a card to look thorough, and never let a card replace the drawing.

Three forms, one component:

- **A file** (YAML, config, a request body, a model class): `.ev-head` with a kind word on the left (`설정`, `코드`, `요청`) and the file name or endpoint on the right, then `.ev-body > pre` with one `.ln` per line, each `.ln` holding only `.t` (the text) and an optional `.tag`. Quote the lines as they are in the source, strip only the indentation common to all of them, and keep to about 12 lines (`… N줄 생략` for the rest). Mark the line the beat is about with `.ln.hi`; its badge does not go inside the line. **Badges sit on the card's left border**, in a `.rail` that is the card's first child: one `<span class="m" style="--i:N">` per badged line, where `N` is that line's 0-based index counting every `.ln` in the `pre` (gap lines included). The rail is outside the scrolling body, so the code keeps its full width and the badge stays put when the reader scrolls sideways; a badge inside the line would push the text or eat the width the annotation needs. For the same reason a `.tag` follows the text directly (`margin-left` only), never pushed to the right edge: a note pushed past the card's edge reads as cut off. A line that **should be there and is not** is a ghost line: `.ln.ghost`, the text the reader would expect, dashed outline, a `.tag` saying `이 줄이 없음`; it gets no syntax color. Color the real lines with the shared syntax classes (`sx-k` keywords and booleans, `sx-f` names and YAML keys, `sx-s` strings, `sx-n` numbers, `sx-c` comments). A YAML file has keys, numbers, booleans and strings: it gets the colors too.
- **A table** (`.erd`): one `.tbl` per table, the real table name with a plain sub-line, one `.col` per column with the badge slot (`<span class="g">`, empty or holding the badge; the rail is for the file form only) then `PK`/`FK` in the gutter and a short type or role on the right (`vector(1024) = 4 KB`, `→ vector_store_chunk.id`), the columns the beat uses in `.hi`, the rest listed and folded (`+ 7 컬럼`) rather than dropped. Two tables sit side by side with the `.rel` arrow between them; more than two is a `visual-doc`.
- **A request or response**: the file form with JSON, the fields of this case only, values the source gives (`…` for the rest), the missing field as a ghost line.

A card never carries a sentence; what the reader should take from it is the note with the same badge.

## Step 4: Build the Page

Read `frame.html` from this skill's base directory. It carries the locked design tokens (shared with `explain-diff`, `micro-world`, `visual-doc`), the page frame, the beat primitives (`.hero`, `.beat`, `.fig`, `.notes`/`.note` and `.notes.plain`, `.say` with `.lab`, `.numbers`, `.close`), the `.m`/`.dg-m` badges and the `.dg-grp` group outline, the optional `.ev` evidence card (badges in its `.rail`) and `.ctl` controls for an interactive figure, and the SVG label classes with a worked example drawing. **Copy the tokens and primitives verbatim, author the drawings bespoke.** Every page has different pictures; that is the whole point.

The type scale is part of what is locked: title 24-30px, beat heading 18px, lede 15.5px, `.say` 15px regular in the secondary text color (the same as visual-doc body prose; bold or the cost color on one phrase at most), number-strip values 16px bold (they are often words, not numbers, so they do not get a display size). Do not enlarge anything to make it "pop", and do not give the lede or `.say` a `max-width`; both run the full column width, the same as the figure card. The page ends at the `.close` card.

Delete the authoring comment at the top of `frame.html` and every placeholder token (every Korean phrase in braces, such as `{출처 한 가지 · YYYY-MM-DD}` and `{이 그림이 보여주는 사실 한 가지. 개조식 가능}`, plus the sample beat copy) as you fill each block. A shipped page must contain none of them.

Output is a single self-contained HTML file. Fill `<title>`: a short noun phrase naming the subject, not a summary.

### Design rules (shared across eli5, explain-diff, micro-world, visual-doc)

- **Title never wraps mid-word.** `word-break: keep-all` stays on titles and narrative text so a Korean particle (`로`, `를`, `이`) can never fall to the start of a line. Titles are short noun phrases that state a fact about the subject (see "Titles say what the picture is of").
- **No decorative gradients.** Backgrounds are solid tokens. A gradient is allowed only when functional (a fade scrim), never as panel decoration.
- **No em-dash or en-dash (`—`, `–`) anywhere in the output.** Not in titles, prose, captions, SVG labels, anywhere. They are a machine-writing tell. Use a colon, parentheses, a comma, or split into two sentences.
- **Color must survive its background.** A mark carrying meaning (a legend swatch, a status dot, an emphasized stroke) must contrast with the surface under it in both themes.
- **Both light and dark, switchable.** Never hardcode a color that breaks one mode. The page follows the system setting by default, and the fixed `.theme-btn` in the top-right corner lets the reader override it. Copy three things verbatim and together: the button markup right after `<body>`, the `.theme-btn` CSS, and the small theme `<script>` in `<head>`. The dark tokens are defined twice on purpose (`@media (prefers-color-scheme: dark)` on `:root:not([data-theme="light"])`, and `:root[data-theme="dark"]`); keep the two blocks identical and never drop one, or the toggle stops winning in one direction. Before finishing, confirm all three parts of the toggle survived: `grep -c '<button type="button" class="theme-btn"'`, `grep -c '\.theme-btn{'`, and `grep -c 'git-claw-doc-theme'` must each print at least 1 (a bare `theme-btn` match can be prose that merely mentions it).
- **Code blocks: one shared component.** Show code only inside the template's `.codeblock` (`.cb-head` with the location on the left (`파일 · 함수`) and the language on the right, then `.cb-body > pre`, one `.ln` span per line). Copy its CSS block verbatim; it is identical in all four templates. Strip the indentation common to every quoted line and keep the relative indentation; nothing else in the text changes. Replace skipped lines with one `<span class="ln gap">… N줄 생략</span>`, indented to the depth of the code it replaces (in an evidence card the gap line is `<span class="ln gap"><span class="t">… N줄 생략</span></span>` and still counts toward the rail index). Color syntax by wrapping tokens yourself, never with a highlighter script or CDN: `sx-k` keywords (`if`, `and`, `return`, `function`), `sx-f` function names where called or defined, `sx-s` string literals, `sx-n` numbers and language constants (`True`, `None`, `null`), `sx-c` comments. Leave variables, operators, and punctuation unwrapped, and HTML-escape `<`, `>`, `&`. Never hard-wrap a long line or set `pre-wrap`: the block scrolls sideways with an always-visible scrollbar. On this page there is no standalone `.codeblock`: quoted source lines go in the file form of the evidence card (`.ev`, see "Evidence card" above), which reuses only the `sx-*` classes and the strip-indent and `… N줄 생략` rules from this item, and appears only when the drawing cannot say it.
- **Natural Korean, not machine translation.** Do not coin stiff 한자어 Koreans do not say; keep a code identifier verbatim rather than force-translate it. Read every heading and `.say` aloud once: if it sounds like a translated subtitle, rewrite it plainer.
- **No metaphor, no implied meaning, no translationese in titles and labels.** A heading, a chip, a row caption, a sub-line, a tooltip states the fact in the subject's own words (`vector_store_file 1행`, `호출 최대 N회`, `원본 테이블`), terse 개조식 welcome. Not `장부`, `선반`, `고리를 돈다`, `~의 여정`, `한눈에 보는 ~`, `숨은 비용`, `마법은 없다`. A phrase that would sound odd said aloud to a colleague is cut; a phrase that could head a page about a different subject is too abstract.

## Step 5: Verify the Render

The rules are the first defense; looking at the page is the second. If a headless browser is available:

```bash
npx --no-install playwright screenshot --full-page "file://<abs-path>" /tmp/eli5-check.png
[ -f /tmp/eli5-check.png ] && echo RENDERED || echo "NO RENDER"
```

**The screenshot command exits 0 even when it renders nothing.** With no browser binary installed, `--no-install` prints an install banner and succeeds, producing no file. So the second line is not optional: without it you will report a render you never looked at. If it prints `NO RENDER`, look for a Chrome binary before giving up (`which google-chrome google-chrome-stable chromium chromium-browser`); any of them renders the page without playwright:

```bash
google-chrome --headless=new --no-sandbox --disable-gpu --hide-scrollbars --window-size=1000,12000 \
  --screenshot=/tmp/eli5-check.png "file://<abs-path>"      # add --force-dark-mode for the dark pass
```

Crop the tall image into pieces of about 1,800px and look at every piece; a page-sized thumbnail hides clipped labels and doubled line spacing. Only when no browser exists at all, say plainly that the page was not visually verified.

Then check, in this order:

1. **The pictures-only pass.** Look at the drawings and ignore every sentence. Does the idea still come through? This is the skill's actual success criterion, and it is the one check that cannot be skipped.
2. No drawing is cut off, overlapping, or scaled into illegibility. If you cannot render, redo the "Fit the text before you draw" arithmetic for every row and every free-standing label instead; a label estimate that exceeds its box, or a row that exceeds W-40, is a defect even if you cannot see it.
3. Titles do not wrap mid-word; no em-dash survived (`grep '—\|–'` the file); no placeholder token or authoring comment survived (`grep -n '{[가-힣]\|eli5 frame'` the file must print nothing: every frame.html placeholder is a Korean phrase in braces); no footer or source list crept back in (`grep 'class="foot"'` the file).
4. Dark mode holds (`--color-scheme=dark`).
5. If the page has an interactive figure: the state at load passes check 1 on its own, and the drawing holds at the smallest and the largest input (move the control, or redo the fit arithmetic for both ends).

Fix and re-render until clean. If no browser is available, still do check 1 by reading your own SVG, and say the render was not visually verified.

## Step 6: Output

1. Write the file to the **repository root**: `eli5-<slug>.html` (slug from the subject, kebab-case). If not in a repo, write to the current directory.
2. Report the **absolute path**. Do NOT auto-open, do NOT commit, do NOT add to .gitignore. Delete only when the user asks.
3. State the one idea in a single line in the report, so the user can check it matches what they wanted explained.
4. List what the page was built from in the report: repo and commit, PR or issue numbers, the key file paths. This is where the evidence lives, since the page itself carries only the eyebrow line.
5. Content language follows the project's AGENTS.md (or CLAUDE.md as fallback) setting; if none, the user's conversational language. Korean output uses 해요체.

**Constraints:**

- Read-only with respect to git state: this skill creates exactly one HTML file and nothing else.
- Tokens are fixed by `frame.html` and the design-system record (`docs/decisions/2026-07-explain-diff-design-system.md`, `docs/decisions/2026-08-eli5.md`). Do not improvise styles.
- Never present the page as complete documentation or proof of correctness. It is an on-ramp for an outsider, and it says so by being short.
- If the request is really a diff explainer (`explain-diff`), a behavior to inhabit (`micro-world`), or a structured report for a colleague who already knows the domain (`visual-doc`), say so and hand off rather than forcing an eli5.
