---
name: eli5
description: >-
  Explain a subject to someone who knows nothing about it, as a self-contained HTML page where big diagrams carry the meaning and the prose is a short factual paragraph per picture.
  TRIGGER when: user types /eli5 <topic>, or asks for a dead-simple picture explainer aimed at someone outside the domain (e.g., "eli5 해줘", "아무것도 모르는 사람한테 설명하듯 만들어줘", "그림으로 쉽게 풀어줘", "비전공자한테 보여줄 자료로 만들어줘").
  DO NOT TRIGGER when: the reader is the developer who has to merge the change (use explain-diff), the subject is a behavior worth inhabiting interactively (use micro-world), the reader is domain-literate and wants the findings structured (use visual-doc), or a code review verdict is wanted (use code-review).
version: "1.5.1"
allowed-tools: Bash(git *), Bash(gh *), Bash(npx *), Bash(google-chrome *), Bash(chromium *), Bash(cat *), Bash(python3 *), Bash(which *), Bash(grep *), Read, Grep, Glob, Write
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

## Step 2: 그림의 순서를 먼저 정한다 (storyboard)

(이 절과 다음 절은 산출물의 문장을 정하는 자리라서 산출물 언어로 쓴다. 지시문을 영어로 읽고 한글로 옮기는 단계에서 번역투 제목이 생기기 때문이다.)

HTML을 쓰기 전에 페이지를 **그림의 순서**로 계획한다. 그림마다 제목 한 줄. 보통 5~8개 beat이고, 더 적어도 되며, 더 많으면 대개 두 페이지다.

**구성안은 처음부터 산출물 언어로, 최종 제목 그대로 쓴다.** 구성안의 제목이 그대로 페이지의 h2가 된다. 역할을 영어로 세운 뒤 한글로 옮기지 않는다: `how scores are read`를 옮기면 `점수 읽는 법`이 되고, `why the obvious fix breaks`를 옮기면 `왜 안 되는가`가 된다. 채팅에 구성안을 보여줄 때도 그 제목들이 아래 "제목은 그림이 무엇인지 말한다"의 기준을 통과해야 한다. 구성안에서 걸리면 그 자리에서 고친다.

기본 흐름. 대부분의 기술 주제에 맞는다. 굵은 글씨는 beat의 **역할**이고, `예:` 뒤는 그 역할에서 나온 제목의 **모양**이다. 역할 이름은 제목이 아니다.

1. **막히는 장면**: 실패하거나 막히는 모습부터 그린다. 해법으로 시작하지 않는다. 예: `쿼리 1개에 LLM 호출 4회, 응답 6초`
2. **처음 떠오르는 해법**: 합리적인 사람이 먼저 해 볼 방법. 예: `캐시 추가: 같은 쿼리는 호출 0회`
3. **그 해법이 깨지는 조건**: **이 beat가 페이지의 요점이다.** 원문에서는 표의 한 줄에 묻혀 있기 쉬우니 찾아낸다. 예: `쿼리 글자 하나만 달라도 캐시 미스, 적중률 3%`
4. **실제로 택한 방식**: 기계적인 동작으로 그린다. 예: `정규화 후 캐시: 공백과 대소문자 무시, 적중률 41%`
5. **대가**: 시간, 돈, 복잡도. 절대 빼지 않는다. 예: `정규화 코드 120줄, 오탐 0.8%`
6. **기억할 한 문장**: 이 방식이 언제 적용되는지. 페이지에서는 맨 끝 `.close` 카드이고 위에 제목을 두지 않는다. 문장 자체가 제목이고, 제목과 카드 제목이 있으면 같은 말을 두 번 하게 된다.

주제가 변경이 아니면 흐름을 바꾼다. "X는 어떻게 동작하나" 페이지는 들어오는 것, 거기에 일어나는 일, 나오는 것, 깨지는 곳 순서가 된다. 실패 beat는 있는 한 남긴다.

**그림 하나, 주장 하나.** 그림 하나에 제목이 둘 필요하면 그림이 둘이다.

### 제목은 그림이 무엇인지 말한다

위 흐름의 역할 이름(막히는 장면, 처음 떠오르는 해법, 깨지는 조건, 실제 방식, 대가, 한 문장)은 beat의 **역할**이지 제목이 아니다. 제목, hero 제목, 행 캡션, 부제는 **주제에 대한 사실을 그대로** 쓴다. 개조식 환영.

- **hero 제목**: 주제의 실제 이름이 든 명사구. `vector_store 테이블 10개의 구조`, `파일 업로드가 청크가 되기까지`. `세 줄로 읽는 벡터 저장소`도, `보이지 않는 선반`도 아니다.
- **beat 제목**: 그림이 보여주는 사실. `장부 테이블 3개, 내용물 테이블 1개`, `standard 모드: LLM 호출 1회`, `description이 비어 있는 경로 3곳`. 예고(`함정`), 질문, 비유(`마법은 없다`)는 쓰지 않는다.
- **행 캡션과 부제**: 글자 그대로의 말이 먼저다 (`원본 테이블`, `장부`, `본문과 벡터`). 비유는 페이지에 한 번, `.say` 줄 안에서, 글자 그대로의 말이 담지 못하는 뜻을 실을 때만 쓴다. 제목이 되지는 않는다.
- **번역투 금지.** `~의 여정`, `숨은 영웅`, `마법처럼`, `~를 만나다`, `한눈에 보는 ~`는 기계 번역의 표지다. 동료에게 소리 내어 말하면 어색한 구절은 지운다.
- **이식 검사**: 다른 주제의 페이지에 그대로 옮겨도 말이 되는 제목은 너무 추상적이다. `문제`, `진짜 해법`, `드는 것과 안 드는 것`은 모두 탈락이고, `마이그레이션 0건, 새 엔드포인트 1개`는 통과다.
- **제목은 답이지 틀이 아니다.** 아래 패턴은 사실을 가리는 틀이다. 각각 가리고 있던 사실로 바꾼다.
  - 질문형 (`~하는가`, `~인가`, `~일까`, `왜 ~인가`): 답을 쓴다. `시험을 어떻게 내는가`는 `벤치마크 구성: 과제 묶음, 채점 규칙, 점수판`.
  - 분류 틀 명사 (`N축`, `~한 축`, `차원`, `관점`, `레이어`, `측면`): 축의 이름을 직접 쓴다. `평가의 4축`은 `채점 방식 3가지: 테스트, 루브릭, 사람 비교`.
  - 방법 명사화 (`~하는 법`, `~읽는 법`, `~하려면`): 대상과 값을 쓴다. `점수 읽는 법`은 `점수 세 종류: pass@1, 승률, 시간 지평`.
  - 단독 `A vs B`: 갈리는 값을 쓴다. `pass@1 vs pass^k`는 `pass@1은 1회 성공, pass^k는 k회 연속 성공`.
  - 강조 틀 (`핵심은`, `결국`, `본질적으로`, `진짜`, `단순히 ~가 아니다`, `~에 불과`): 지우고 사실만 남긴다.
  - 개수를 먼저 정하기 (`세 가지`를 먼저 세고 셋째를 채워 넣기): 항목이 먼저고 개수는 세어서 적는다.
  - 주제 전체를 감싸는 비유 (`모의고사`, `시험지`, `장부`): 제목과 구성안에 쓰지 않는다. `.say` 한 줄의 예외는 그대로다.
  - 본문에서는 `A vs B`나 `~하려면`이 평범한 문장일 수 있다. 이 규칙은 제목, 캡션, 구성안에 대한 것이다. Render check의 제목 grep은 그래서 제목 태그 안만 보고, 걸린 줄은 소리 내어 읽고 판단할 목록이지 실패가 아니다.

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
- **Comparing options? Draw the difference.** Two states as two rows of the same path, with the one edge that changes between them. Separate labeled boxes with nothing connecting them is a restated list, not a comparison.
- **Label the arrows.** An unlabeled arrow means "related somehow". `writes`, `20장씩`, `19배 압축` is information.
- **Concrete by default.** Draw the specific things, not categories of things: eight named tables in their three rows beats three ovals labeled 입력 / 처리 / 저장. If you catch yourself drawing "data flows to storage", name the data and name the storage. Abstract shapes are for subjects that have no artifacts (a concept, a policy), not a shortcut for subjects that do.
- **Real names, plain sub-line.** When the subject is a system, a box is the real thing under its real name: the table `vector_store_file`, the endpoint `POST /files`, the flag `use_doc_chunking`. Under the name, one sub-line says what it is for an outsider (`장부 1줄`, `실제 본문과 벡터`). The sub-line is literal too; the one place a metaphor may appear is a `.say` line (see "제목은 그림이 무엇인지 말한다"). What stays out of the drawing is **explanation**: the why, the mechanism in prose, file paths, line numbers. Those go in `.say` or your report.
- **Size by `viewBox`** (`viewBox="0 0 W H"`, CSS `width:100%; height:auto`), pick W and H for the content, and align shapes to a shared grid. Eyeballed offsets read as noise.
- **Theme with `currentColor`.** Strokes and text inherit the page's foreground so both themes work. Reserve `var(--blue)` for the one element carrying the point of that picture and `var(--red)` for the thing that breaks. Two accent colors per drawing at most.
- **Text stays at the two label sizes** (`.dg-l` 13px, `.dg-s` 11px mono) plus at most one `.dg-b` big number per drawing (and the optional `.dg-q` focal string below). A box holds its real name and **at most one sub-line**, never a sentence. The sub-line is either the one fact the reader needs (`statement_timeout 2000ms`) or one example (`"보험약관 암보험 보장 내역"`), never both, and never a list of fields from the source (`filters eq, tool_choice`, `failed_stores [vs_…]`): detail of that kind goes to the notes under the figure or to an evidence card. (A schema box in an ERD-style drawing is the exception: its lines are the columns.) Fit every line to its box with the arithmetic in "Fit the text before you draw" below.
- **State captions and the focal string, when rows compare states.** (optional; most figures use neither) A figure whose rows show the same thing in different states may give each row a header line above it: a pill `<rect class="dg-pill" height="22" rx="11">` at the row's left edge with a `.dg-cap` label centered on it, both carrying the same modifier, then a `.dg-s` sub-line after the pill saying what that state concretely does. The header line takes its own height in the layout plan, not a caption column. Pick the caption by what the rows compare; these pairs are recommendations, not a fixed list. One change that has already happened reads well as `BEFORE` / `AFTER`; one change that is proposed or planned, as `AS-IS` / `TO-BE`. Rows that are not one change over time (two cases, two modes, two inputs) read well when each is named by what is true in that row, a short noun phrase in the subject's terms. Use `.bad` for the state that breaks, `.em` for the state carrying the point, and no modifier for a state that is neither: a red caption claims the state is wrong. When the point of the figure is the text itself (a search query, a command, an error message), that one kind of string may add `.dg-q` to its `.dg-l` (15px bold; `.dg-em`/`.dg-bad` still color it); every other label stays at the regular sizes, and the pills count toward the two accent colors.
- **Every figure gets** `role="img"` and an `aria-label` stating the same claim as its heading. (An interactive figure points `aria-labelledby` at its `그래서` line instead, because the claim changes with the input.)
- No `<script>`, `<style>`, or `<foreignObject>` inside the SVG (the script of an interactive figure sits at the end of `<body>`; see "An interactive figure" below). No emoji, no icon fonts, no external resources. Long decorative path data means the drawing is too elaborate: simplify it.

### Fit the text before you draw (no renderer needed)

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
3. **Row budget.** For every row: 20 + (caption column + 40, when rows carry a left-hand caption; size that column from its widest caption line like any other label) + Σ box widths + Σ gaps + 20 ≤ viewBox width. Gaps are at least 40; a gap that carries an arrow label is at least the label estimate + 24. When a row does not fit, shorten labels first, then break it into two rows, then (last) widen the viewBox up to 840. Never above 840: the figure is about 780px wide on screen, so a wider viewBox shrinks 13px text below 12px. Height is unlimited: more boxes mean more rows, not smaller text.
4. **Nothing crosses a border.** Every box label is `text-anchor="middle"` at the box center, and rule 1 guarantees it stays inside. Free-standing text (edge labels, row captions, the big number) gets a clear zone: no other text, no line other than the one it labels, and no box edge within 6px of its estimated extent, and that extent stays inside `[20, W-20]`. Vertically, a line of text occupies baseline −12 to +3 for `.dg-l`, −10 to +3 for `.dg-s`, −14 to +4 for `.dg-q`, −18 to +4 for `.dg-b`; use those for the clear zone, not just the horizontal estimate.
5. **Edges never pass through a box.** A line that crosses a box it does not connect is a defect, whichever direction it runs. When an edge has to skip a row, route it down a lane between boxes and turn below that row. Two edges that end on the same box land at least 20px apart along that side (or merge into one elbow before arriving). Give each `<svg>` its own marker id (`ar1`, `ar2`, …): a page has several drawings and duplicate ids are invalid.
   Preferred, not required: horizontal and vertical segments with an elbow (`M x1 y1 V ymid H x2 V y2`) rather than a diagonal, and the label on a straight segment: baseline 10px above a horizontal segment with `text-anchor="middle"`, or beside a vertical one with `text-anchor="end"` at `x = line x − 8`, in class `.dg-s`. If the two endpoints differ by less than 12px, draw the edge straight. A diagonal is fine when it is the clearer line; then its label carries `dg-halo` so it stays readable where it crosses. Rows stacked above each other are at least 40px apart (box bottom to next box top) when an edge label sits between them, 30px otherwise.
6. **Text comes last.** Inside each `<svg>`, write every `<rect>`, `<line>`, and `<path>` first and every `<text>` after them. SVG paints in document order, so a line written after a label is drawn over it, and `dg-halo` only works when the text is painted on top.
7. **Plan first, as a comment.** The first child of every `<svg>` is a one-line layout plan, and writing it is the check:
   `<!-- W=760 | row1 y=56 h=88: [40..190] gap45 [235..385] gap45 [430..580] gap40 [620..720] -->`
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
- **A particle is part of the word before it; a count stays on one line.** (shared across eli5, explain-diff, micro-world, visual-doc) Attach every particle and ending to the word before it, also after an English word, a number, or a code chip: `Job은`, `store에도`, `API와`, `400이`, `<code>job_indexing.mode</code>가`, `automatic인`. Never `Job 은` or `</code> 을`: where `word-break: keep-all` applies a line breaks only at a space, so that space lets the particle start a line. Choose the particle by how the word is read aloud (`Job은`, `API는`, `store를`). Write a digit and its unit together (`3개`, `0행`, `1회`), and put `&nbsp;` between a native numeral and its unit (`한&nbsp;번`, `두&nbsp;줄`, `세&nbsp;가지`) so the line never breaks inside the count. Bind nothing else with `&nbsp;`: each one makes a longer unbreakable run and a more ragged right edge. Before finishing, run `grep -nP '[A-Za-z0-9_)]+(</code>)? (은|는|이|가|을|를|의|에|와|과|로|으로|도|만|에서|에는|에도|인|인데)(?=[ ,.<)]|$)'` on the file and fix every hit that is a particle (`이` and `가` can also be a word of their own), except inside source quoted verbatim in a code block or an evidence card, which keeps its spacing.
- No paragraphs of body prose anywhere on the page. If a beat genuinely needs three paragraphs, the subject wants `visual-doc`.
- There is no `figcaption`. A fact that used to go in a caption goes in the `.say` if the reader needs it to read the picture, and is dropped otherwise.
- **File paths and line numbers stay off the page.** An outsider will never open `app/models/store.py:103`. The names of the things drawn (a table, an endpoint, a flag) are not evidence of that kind; they are the nouns of the subject and belong in the drawing (see "Real names, plain sub-line" above). A name the drawing does not show may still appear once, in that beat's notes, when the reader will actually meet it. The one place a file name is allowed is the head of an evidence card (`config/config.onprem.yaml`), because there the file itself is the thing shown; line numbers stay off even there.
- **No footer, no source list.** File paths, line numbers, and the list of what you read are evidence for the person who asked, not content for the reader. They go in your report (Step 6), never in the HTML.
- **Provenance is one line in the hero eyebrow**: the source and a date, e.g. `studio-pipeline, 2026-09-28`. Nothing else goes there: no commit hash, no "기준", no status word. If there is no meaningful source (a general "how does DNS work"), the eyebrow is just the date.
- Numbers appear as before/after pairs in the number strip, four at most, and at least one of them is a cost.

### Evidence card, only when the drawing cannot say it

A drawing shows mechanism; it cannot show that a line is **absent** from a file, which **columns** a table has, or that a field is **missing** from a request. When the beat's claim is one of those, put an **evidence card** (`.ev`) between the figure and the notes: the real thing in its own shape, quoted verbatim, with the badge on the line that matters. The card is optional, and the test is strict: if the notes could state the fact and the reader would believe them, there is no card. A page where every beat earns one may carry one on every beat; a page where none does carries none. Never add a card to look thorough, and never let a card replace the drawing.

Three forms, one component:

- **A file** (YAML, config, a request body, a model class): `.ev-head` with a kind word on the left (`설정`, `코드`, `요청`) and the file name or endpoint on the right, then `.ev-body > pre` with one `.ln` per line, each `.ln` holding only `.t` (the text) and an optional `.tag`. Quote the lines as they are in the source, strip only the indentation common to all of them, and keep to about 12 lines (`… N줄 생략` for the rest). Mark the line the beat is about with `.ln.hi`; its badge does not go inside the line. **Badges sit on the card's left border**, in a `.rail` that is the card's first child: one `<span class="m" style="--i:N">` per badged line, where `N` is that line's 0-based index counting every `.ln` in the `pre` (gap lines included). The rail is outside the scrolling body, so the code keeps its full width and the badge stays put when the reader scrolls sideways; a badge inside the line would push the text or eat the width the annotation needs. For the same reason a `.tag` follows the text directly (`margin-left` only), never pushed to the right edge: a note pushed past the card's edge reads as cut off. A line that **should be there and is not** is a ghost line: `.ln.ghost`, the text the reader would expect, dashed outline, a `.tag` saying `이 줄이 없음`; it gets no syntax color. Color the real lines with the shared syntax classes (`sx-k` keywords and booleans, `sx-f` names and YAML keys, `sx-s` strings, `sx-n` numbers, `sx-c` comments). A YAML file has keys, numbers, booleans and strings: it gets the colors too.
- **A table** (`.erd`): one `.tbl` per table, the real table name with a plain sub-line, one `.col` per column with the badge slot (`<span class="g">`, empty or holding the badge; the rail is for the file form only) then `PK`/`FK` in the gutter and a short type or role on the right (`vector(1024) = 4 KB`, `→ vector_store_chunk.id`), the columns the beat uses in `.hi`, the rest listed and folded (`+ 7 컬럼`) rather than dropped. Two tables sit side by side with the `.rel` arrow between them; more than two is a `visual-doc`.
- **A request or response**: the file form with JSON, the fields of this case only, values the source gives (`…` for the rest), the missing field as a ghost line.

A card never carries a sentence; what the reader should take from it is the note with the same badge.

Cards stack one per row (`.evs`). Two cards go side by side (`.evs.pair`) only when they pass "Rows by default" in the design rules: short lines in both, and the reader reads them together.

## Step 4: Build the Page

Read `frame.html` from this skill's base directory. It carries the locked design tokens (shared with `explain-diff`, `micro-world`, `visual-doc`), the page frame, the beat primitives (`.hero`, `.beat`, `.fig`, `.notes`/`.note` and `.notes.plain`, `.say` with `.lab`, `.numbers`, `.close`), the `.m`/`.dg-m` badges and the `.dg-grp` group outline, the optional `.ev` evidence card (badges in its `.rail`) and `.ctl` controls for an interactive figure, the section hierarchy block, and the SVG label classes with a worked example drawing. **Copy the tokens and primitives verbatim, author the drawings bespoke.** Every page has different pictures; that is the whole point.

**Where the reader is on the page.** (shared across eli5, explain-diff, micro-world, visual-doc; the CSS block starts with the `section hierarchy (SHARED` comment and the script at the end of `<body>` is copied with it) Three marks, all inside the column: the 3px `.progress` bar right after the theme button fills with the scroll ratio and carries no text; the `.outline` card at the end of the hero lists every beat once, each line the same text as its h2; and every `.beat` carries `sec-top` (the rule above it), an `id`, and a `.sec-no` line before the h2 (`01 / 08`: this beat's number and the beat count). The h2 text never carries the number. The `.close` card has no heading and stays out of the outline. Nothing floats beside or above the column: no side rail, no sticky header with the section name. An eli5 page has no sub-sections, so `.sub-h` and `.sub-no` stay unused here.

The type scale is part of what is locked: title 24-30px, beat heading 18px, lede 15.5px, `.say` 15px regular in the secondary text color (the same as visual-doc body prose; bold or the cost color on one phrase at most), number-strip values 16px bold (they are often words, not numbers, so they do not get a display size). Do not enlarge anything to make it "pop", and do not give the lede or `.say` a `max-width`; both run the full column width, the same as the figure card. The page ends at the `.close` card.

Delete the authoring comment at the top of `frame.html` and every placeholder token (every Korean phrase in braces, such as `{출처 한 가지, YYYY-MM-DD}` and `{이 그림이 보여주는 사실 한 가지. 개조식 가능}`, plus the sample beat copy) as you fill each block. A shipped page must contain none of them.

Output is a single self-contained HTML file. Fill `<title>`: a short noun phrase naming the subject, not a summary.

### Design rules (shared across eli5, explain-diff, micro-world, visual-doc)

- **Title never wraps mid-word.** `word-break: keep-all` stays on titles and narrative text so a Korean particle (`로`, `를`, `이`) can never fall to the start of a line. Titles are short noun phrases that state a fact about the subject (see "제목은 그림이 무엇인지 말한다").
- **No decorative gradients.** Backgrounds are solid tokens. A gradient is allowed only when functional (a fade scrim), never as panel decoration.
- **No machine-writing symbols anywhere in the output.** (shared across eli5, explain-diff, micro-world, visual-doc) No em-dash (`—`), en-dash (`–`), or middle dot (`·`) in titles, prose, captions, labels, SVG text, TOC descriptions, hints, quiz text, eyebrows, or code-block headers. They are machine-writing tells. A dash becomes a colon, parentheses, a comma, or two sentences: write `옵션 1만 vs 옵션 1+2: 권장안과 근거`, never the same with a dash. A middle dot between items becomes a comma or `와`/`과` (`설치와 로그인`, `제목, 소제목, 라벨`); a dot used as a separator in a label becomes a comma or a line break. A range is `~` (`1~3`). The one exception is source quoted verbatim in a code block or an evidence card, which keeps its characters exactly; in a prose quote (`.quote2`, an inline quote) quote a shorter span that does not contain one, and never rewrite a quote to remove it. Before finishing, `grep -nP '[—–·]' <file>` must print nothing except lines of such verbatim source.
- **Color must survive its background.** A mark carrying meaning (a legend swatch, a status dot, an emphasized stroke) must contrast with the surface under it in both themes.
- **Rows by default; side by side only when the render proves it fits.** (shared across eli5, explain-diff, micro-world, visual-doc) When two things are shown for comparison (two states, two options, two code blocks, a request and its response, two evidence cards), stack them as rows: each keeps the full column width and the eye reads down instead of jumping across. Side by side is a judgment call, not a ban, and it is judged on the rendered page, not by counting characters: it stays only when nothing inside either half scrolls sideways and the reader gains from seeing the two at the same height (two short lists, two table schemas, a control and the view it changes). To measure, append `measure.html` from this skill's base directory to a copy of the page and read the result with the one-line command in its header comment: `data-sbs` names every multi-column grid with something scrolling sideways inside it, and every block it names is stacked. Without a browser there is no measurement, so stack.
- **Both light and dark, switchable.** Never hardcode a color that breaks one mode. The page follows the system setting by default, and the fixed `.theme-btn` in the top-right corner lets the reader override it. Copy three things verbatim and together: the button markup right after `<body>`, the `.theme-btn` CSS, and the small theme `<script>` in `<head>`. The dark tokens are defined twice on purpose (`@media (prefers-color-scheme: dark)` on `:root:not([data-theme="light"])`, and `:root[data-theme="dark"]`); keep the two blocks identical and never drop one, or the toggle stops winning in one direction. Before finishing, confirm all three parts of the toggle survived: `grep -c '<button type="button" class="theme-btn"'`, `grep -c '\.theme-btn{'`, and `grep -c 'git-claw-doc-theme'` must each print at least 1 (a bare `theme-btn` match can be prose that merely mentions it).
- **Code blocks: one shared component.** Show code only inside the template's `.codeblock` (`.cb-head` with the location on the left (`파일, 함수`) and the language on the right, then `.cb-body > pre`, one `.ln` span per line). Copy its CSS block verbatim; it is identical in all four templates. Strip the indentation common to every quoted line and keep the relative indentation; nothing else in the text changes. Replace skipped lines with one `<span class="ln gap">… N줄 생략</span>`, indented to the depth of the code it replaces (in an evidence card the gap line is `<span class="ln gap"><span class="t">… N줄 생략</span></span>` and still counts toward the rail index). Color syntax by wrapping tokens yourself, never with a highlighter script or CDN: `sx-k` keywords (`if`, `and`, `return`, `function`), `sx-f` function names where called or defined, `sx-s` string literals, `sx-n` numbers and language constants (`True`, `None`, `null`), `sx-c` comments. Leave variables, operators, and punctuation unwrapped, and HTML-escape `<`, `>`, `&`. Never hard-wrap a long line or set `pre-wrap`: the block scrolls sideways with an always-visible scrollbar. On this page there is no standalone `.codeblock`: quoted source lines go in the file form of the evidence card (`.ev`, see "Evidence card" above), which reuses only the `sx-*` classes and the strip-indent and `… N줄 생략` rules from this item, and appears only when the drawing cannot say it.
- **Natural Korean, not machine translation.** Do not coin stiff 한자어 Koreans do not say; keep a code identifier verbatim rather than force-translate it. Read every heading and `.say` aloud once: if it sounds like a translated subtitle, rewrite it plainer.
- **No metaphor, no implied meaning, no translationese in titles and labels.** A heading, a chip, a row caption, a sub-line, a tooltip states the fact in the subject's own words (`vector_store_file 1행`, `호출 최대 N회`, `원본 테이블`), terse 개조식 welcome. Not `장부`, `선반`, `고리를 돈다`, `~의 여정`, `한눈에 보는 ~`, `숨은 비용`, `마법은 없다`. A phrase that would sound odd said aloud to a colleague is cut; a phrase that could head a page about a different subject is too abstract.
- **Headings are answers, not frames; plan them in the output language.** (shared across eli5, explain-diff, micro-world, visual-doc) Write the section plan in the language the page will be in, with the final heading text, before any HTML: a heading drafted in English and translated comes out as `시험을 어떻게 내는가` or `점수 읽는 법`. The patterns below are the tells, and each is replaced by the fact it was hiding. A question (`~하는가`, `~인가`, `~일까`, `왜 ~인가`) becomes its answer: `점수를 어떻게 읽는가` is `점수 세 종류: pass@1, 승률, 시간 지평`. A classification noun (`N축`, `~한 축`, `차원`, `관점`, `레이어`, `측면`) becomes the names of the things: `평가의 4축` is `채점 방식 3가지: 테스트, 루브릭, 사람 비교`. A method noun (`~하는 법`, `~읽는 법`, `~하려면`) becomes the subject and its values. A bare `A vs B` becomes the value that differs: `pass@1 vs pass^k` is `pass@1은 1회 성공, pass^k는 k회 연속 성공`. An emphasis frame (`핵심은`, `결국`, `본질적으로`, `진짜`, `단순히 ~가 아니다`, `~에 불과`) is deleted. A count is counted after the items exist, never chosen first and padded (`세 가지`). A metaphor that frames the whole subject (`모의고사`, `시험지`, `장부`) stays out of every heading and out of the plan. None of these words is banned in body prose, where `A vs B` or `~하려면` can be the plain sentence: the rule is about headings, captions, outline lines, and the plan, which is why the Render check's heading grep reads only heading tags and reports a reading list, not failures.

## Step 5: Verify the Render

### Render check (shared across eli5, explain-diff, micro-world, visual-doc)

The rules are the first defense; looking at the rendered page is the second, and a defect counts as fixed only when the render shows it fixed. Work in a scratch directory of your own (`D`, e.g. `/tmp/eli5-<slug>`); never a shared fixed path, because other runs may be writing there at the same time. Find a browser first (`which google-chrome google-chrome-stable chromium chromium-browser`). Do not rely on `npx --no-install playwright screenshot`: with no browser binary installed it prints an install banner, exits 0, and writes no file.

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
5. **Headings.** `grep -nP '<(h[1-3]|span class="dg-cap")[^>]*>[^<]*(는가|인가|일까|나요|\?|[0-9]축|한 축|하는 법|읽는 법|하려면|핵심|본질|결국| vs )' <file>` lists the headings and figure captions that match a translation tell. This is a reading list, not a failure: read each hit aloud, rewrite it only when it reads as a frame rather than a fact, and name in the report each hit you kept and why. Body prose is not searched.

Fix and repeat until the first four pass and the fifth has been read. Only when no browser exists at all, say plainly that the page was not visually verified. Screenshots, crops, and the measurement copy live in `D`, are not part of the output, and are deleted when you finish.

Then check, in this order:

1. **The pictures-only pass.** Look at the drawings and ignore every sentence. Does the idea still come through? This is the skill's actual success criterion, and it is the one check that cannot be skipped.
2. No drawing is cut off, overlapping, or scaled into illegibility (the Render check's measurement prints `none`). If you cannot render, redo the "Fit the text before you draw" arithmetic for every row and every free-standing label instead; a label estimate that exceeds its box, or a row that exceeds W-40, is a defect even if you cannot see it.
3. Titles do not wrap mid-word; no machine-writing symbol survived (`grep -nP '[—–·]'` the file prints nothing outside verbatim source); no placeholder token or authoring comment survived (`grep -n '{[가-힣]\|eli5 frame'` the file must print nothing: every frame.html placeholder is a Korean phrase in braces); no footer or source list crept back in (`grep 'class="foot"'` the file).
4. The position marks survived: `grep -c 'id="progress"'` prints 1, `grep -c '<li><b>'` (outline lines) equals `grep -c 'class="beat sec-top"'` (beats), every beat has a `.sec-no` line whose second number is that beat count, and no h2 text starts with a digit followed by a space or a period.
5. Dark mode holds (the dark screenshot).
6. If the page has an interactive figure: the state at load passes check 1 on its own, and the drawing holds at the smallest and the largest input (move the control, or redo the fit arithmetic for both ends).

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
