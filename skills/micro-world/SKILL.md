---
name: micro-world
description: >-
  Build a throwaway interactive HTML simulation (a "micro-world") of a specific behavior (a state machine, algorithm, data transform, or protocol flow), so the developer can inhabit and feel how the code works instead of reading about it.
  TRIGGER when: user asks to simulate, play with, or interactively explore a behavior (e.g., "micro-world 만들어줘", "이거 시뮬로 보여줘", "만져볼 수 있게 만들어줘", "동작을 인터랙티브로 이해하고 싶어").
  DO NOT TRIGGER when: user wants a document explaining a diff (use explain-diff), a picture explainer for someone outside the domain (use eli5), a code review (use code-review), or the change is config/rename/dependency-bump material with no behavior to inhabit.
version: "1.6.1"
allowed-tools: Bash(git *), Bash(gh *), Bash(google-chrome *), Bash(chromium *), Bash(cat *), Bash(python3 *), Bash(which *), Bash(grep *), Read, Grep, Glob, Write
---

## The One Principle

**Let the developer live inside the behavior.** A micro-world (after Seymour Papert's Mathland) is an ephemeral, purpose-built interactive environment where understanding comes from manipulating the system, not reading about it. It is bespoke by nature: the embodiment is designed per behavior, only the visual language is fixed.

This skill is a sibling of `explain-diff`: that one produces a consistent document; this one produces a one-off simulation. They share design tokens, never structure.

## Step 0: Applicability Gate (decline honestly when a sim adds nothing)

Micro-worlds shine when the subject has **state that evolves under input**:

- state machines and transitions (auth flows, retries, lifecycle, cache/deploy procedures)
- algorithms (scheduling, matching, caching, parsing: anything steppable)
- data transforms (pipeline stages, before/after shapes)
- visual/spatial logic (layout, geometry, rendering)
- protocol/interaction flows (request/response ordering, races)

They add nothing for config changes, renames, dependency bumps, doc edits, or straight-line glue code. In those cases, say so plainly ("이 변경은 시뮬레이션으로 얻는 게 거의 없어요") and suggest `/explain-diff` instead. Do not manufacture a hollow sim to satisfy the request.

Set expectations once at the start: micro-worlds are **best-effort**; quality tracks how simulatable the behavior is.

## Step 1: Identify the World

Read the actual code (diff, PR, file, or the behavior the user described). Extract:

1. **State variables**: what the world tracks (queue contents, token expiry, cache directories).
2. **User inputs**: the actions the developer performs (fire an event, run a command, advance time).
3. **Observable outcomes**: what visibly changes in response (state chips, file trees, logs, shapes).
4. **The one confusion** this sim exists to dissolve: name it; it drives every design choice. **One confusion = one world.** If the subject holds several distinct confusions, build several small worlds rather than one bloated page.

## Step 2: Design the Scenario and Embodiment

### Guided single cycle: no free-form exploration

The sim runs **one authored scenario**, framed as a mission:

- A **mission bar** states 2-3 concrete goals as check chips that tick as the state satisfies them.
- At every moment exactly **one recommended next action** is enabled and visually highlighted; all other actions are disabled. The user cannot wander off the narrative: free-form mode is intentionally absent, because unguided branching dilutes understanding.
- The scenario should route **through the failure the world exists to teach** (the trap fires mid-story, then the story recovers), not around it.
- An **inline restart** lives in the action palette itself, styled like the other actions (`↻ 시나리오 처음부터`); it gets the recommendation highlight when the mission completes so the natural next move is another cycle. No detached reset link far from the controls.

### Playback: every action is a step

- Each completed action pushes a **full snapshot** (environment output, narration, world state). `‹` `›` controls in the environment header move through them; going back and acting anew truncates forward history.
- **Forward is a play button**: when there is no forward history, `›` executes the recommended next action: so from a fresh page, pressing `→` repeatedly walks the entire scenario.
- Keyboard: `←` back, `→` forward/execute, `R` restart (also match the Korean IME key, e.g. `ㄱ` for R). Ignore modifier-key combos; disable during output animation.

### Fact / interpretation separation

- The environment (terminal, canvas, timeline) shows **only faithful system output**: what the real tool would actually print or display. No judgment coloring, no commentary inside it. When you cannot find the real output of a command (no source, no way to run it), do not invent wording: show a neutral line such as `(출력 없음)` or the command's effect as state, and say in the footnote which outputs are not the tool's real text.
- All meaning lives in a separate **narration strip** directly below the environment: label ("지금 무슨 일이") + one bold core sentence + at most 2 bullets, with the whole strip tinted semantically (ok/warn/err). This is the same visual language as explain-diff's takeaway cards.

### World state panel

- Group all observable state into **one card** ("세계 상태") with internally divided sub-panels (e.g., filesystem tree | session memory), never scattered sibling cards.
- Changed values give brief visual feedback (a pulse, an enter animation) so the action → state connection is legible.

### Embodiment is chosen by the subject

The interaction rule is absolute: **the user's input must change the outcome**; a step-through that reveals already-visible content in order is banned. Beyond that, pick the embodiment that matches the behavior:

- **Terminal command center** for CLI procedures and deploy/ops flows: dark terminal pane with a command palette (real commands as the actions, `❯` shell / `✳` agent prompts), full width and dominant.
- **Timeline scrub** for execution traces: drag through steps, each position shows real internal state (stack, bindings, queue).
- **Drag-and-drop + side-by-side structure views** for structural reorganizations: manipulate one side, watch the other (e.g., directory trees, schemas) evolve in parallel.
- **Parameter knobs** for formulas and thresholds: sliders/toggles feeding the real computation, outputs re-render live.

Fidelity rule: the sim must model the **actual logic**; port the real algorithm/state machine into inline JS where feasible. Distinctions that exist in the real system (e.g., "copy changes disk, reload changes the session") must exist in the model. Any simplification must be disclosed in a **small footnote below the sim** (plain faint text: never an alarm-colored callout).

## Step 3: Build the Page

Read `template.html` from this skill's base directory: it carries the design tokens, the command-center frame (mission bar, terminal + palette + step nav, narration strip, world-state card), and the **generic runtime JS** (output helpers, snapshot/restore history, keyboard playback). **Fill the bespoke parts; do not restyle the frame.** For a non-terminal embodiment, replace the environment pane's interior while keeping the frame (mission bar, narration strip, world card, playback) intact. No emoji (source quoted verbatim in a code block keeps its emoji, like any other character), no hand-drawn SVG icons, no external resources. Single self-contained HTML file.

- Top: `micro-world` eyebrow + title naming the behavior + lede stating the mission and how to drive it (one line). 제목, 미션 칩, 액션 라벨은 산출물 언어로 처음부터 쓴다. 영어로 세운 뒤 옮기지 않는다 (`how retries back off`를 옮기면 `재시도가 물러나는 법`이 된다): `재시도 간격 1초, 2초, 4초`처럼 사실을 쓴다.
- Position marks (shared with eli5, explain-diff, visual-doc; the `section hierarchy (SHARED` CSS block and the script at the end of `<body>`): the template carries the 3px `.progress` bar after the theme button, with no text on it. A micro-world is one screen, so it has no numbered sections and no outline; use `sec-top` with a `.sec-no` line (`1 / 4`) and the `.outline` card only when a page grows to four or more top-level sections, which is rare here.
- Close with a `핵심 정리` takeaway card: what the user should have felt after playing.
- Content language follows the project's AGENTS.md (or CLAUDE.md as fallback) setting, else the user's conversational language. Korean prose follows the `Voice` rule in the design rules.

### Design rules (shared across micro-world, explain-diff, visual-doc, eli5)

- **No redundant or meta text; this rule comes first.** (shared across eli5, explain-diff, micro-world, visual-doc) Every line on the page says something the reader does not already have. A heading, a chip, a row caption or a card head is said once: no sub-line, tag or caption that restates it in other words (`현재: 모노 글꼴 영어 벽` beside a block that visibly is one, `결과 문서의 블록을 그대로 옮김`). The page never describes its own form: no words about fonts, sizes, units, emphasis, folding, columns, or what a block "shows" (`본문 글꼴`, `문단 단위`, `해설이 가리키는 문단만 강조`, `표로 정리`), and it never names the skill, the template or the process that made it. A deck states a fact about the subject, never what the section is about (`이 절에서는`). When a sentence would be true of the page rather than of the subject, delete it. Render check: read every sub-line, tag, deck and caption aloud and strike the ones that describe the page.
- **Voice: prose in 해요체, as to a colleague; 개조식 only where the heading rules apply.** (shared across eli5, explain-diff, micro-world, visual-doc) Korean running text (lede, notes, closing lines, takeaways, prose paragraphs, sentence-shaped tooltips) is in 해요체, the register of explaining something to a colleague who asked: `검색에는 쓰이지 않아요`, `원문 그대로 둬요`. Not the childish end of 해요체 (`만들어요`, `안 가요`, `하나도 안 나빠졌어요` in a technical sentence) and not 한다체 or 합니다체. A sentence ends in a predicate: a noun or a stand-in noun at the end (`~하는 꼴. 20건.`, `~한 것.`) is telegraphese and is rewritten as a sentence (`~하는 질문이에요. 20건.`). Every noun the sentence leans on is concrete: `빠진 이름` becomes `이전 질문에만 있던 문서 이름이나 판`. 개조식 (nominal fragments) belongs to headings, decks, chips, SVG labels and sub-lines, table cells and the outline. The length caps on notes do not change the register: a shorter note has fewer facts, not clipped endings. Render check: the Voice count below.
- **Title never wraps mid-word.** `word-break: keep-all` stays on the title and narrative text so a Korean particle (`로`, `를`, `이`) can never fall to the start of a line. Keep the title a short noun phrase.
- **No decorative gradients.** Backgrounds are solid tokens. A gradient is allowed only when functional (a fade scrim), never as panel decoration.
- **No machine-writing symbols anywhere in the output.** (shared across eli5, explain-diff, micro-world, visual-doc) No em-dash (`—`), en-dash (`–`), or middle dot (`·`) in titles, prose, captions, labels, SVG text, TOC descriptions, hints, quiz text, eyebrows, or code-block headers. They are machine-writing tells. A dash becomes a colon, parentheses, a comma, or two sentences: write `옵션 1만 vs 옵션 1+2: 권장안과 근거`, never the same with a dash. A middle dot between items becomes a comma or `와`/`과` (`설치와 로그인`, `제목, 소제목, 라벨`); a dot used as a separator in a label becomes a comma or a line break. A range is `~` (`1~3`). The one exception is source quoted verbatim in a code block or an evidence card, which keeps its characters exactly; in a prose quote (`.quote2`, an inline quote) quote a shorter span that does not contain one, and never rewrite a quote to remove it. Before finishing, `grep -nP '[—–·]' <file>` must print nothing except lines of such verbatim source.
- **Code blocks: one shared component.** Show code only inside the template's `.codeblock` (`.cb-head` with the location on the left (`파일, 함수`) and the language on the right, then `.cb-body > pre`, one `.ln` span per line). Copy its CSS block verbatim; it is identical in all four templates. Strip the indentation common to every quoted line and keep the relative indentation; nothing else in the text changes. Replace skipped lines with one `<span class="ln gap">… N줄 생략</span>`, indented to the depth of the code it replaces. Color syntax by wrapping tokens yourself, never with a highlighter script or CDN: `sx-k` keywords (`if`, `and`, `return`, `function`), `sx-f` function names where called or defined, `sx-s` string literals, `sx-n` numbers and language constants (`True`, `None`, `null`), `sx-c` comments. Leave variables, operators, and punctuation unwrapped, and HTML-escape `<`, `>`, `&`. Never hard-wrap a long line or set `pre-wrap`: the block scrolls sideways with an always-visible scrollbar.
- **Natural Korean, not machine translation.** Do not coin stiff 한자어 Koreans do not say; keep a code/identifier term verbatim rather than force-translate it.
- **One name per thing, the actor as subject, the count as a number.** (shared across eli5, explain-diff, micro-world, visual-doc) Pick one name for each thing and each action and keep it everywhere on the page, mission chips, action labels, narration, and the world-state panel included: once it is `액세스 토큰` it is never `인증 토큰`, `세션 키`, or `자격 증명`; once it is `재발급` it is never `갱신`. A second name reads as a second thing. When two different things share a word, qualify both (`API 서버`, `인증 서버`, never a bare `서버`). Make the subject the thing that acts: `인증 서버가 두 번째 요청을 거부했어요`, not `두 번째 요청이 거부됐어요`; a sentence that hides the actor hides where the cause is. Write the count the source gives (`요청 3개`, `3번`), not `여러 개` or `요청 수만큼`.
- **A particle is part of the word before it; a count stays on one line.** (shared across eli5, explain-diff, micro-world, visual-doc) Attach every particle and ending to the word before it, also after an English word, a number, or a code chip: `Job은`, `store에도`, `API와`, `400이`, `<code>job_indexing.mode</code>가`, `automatic인`. Never `Job 은` or `</code> 을`: where `word-break: keep-all` applies a line breaks only at a space, so that space lets the particle start a line. Choose the particle by how the word is read aloud (`Job은`, `API는`, `store를`). Write a digit and its unit together (`3개`, `0행`, `1회`), and put `&nbsp;` between a native numeral and its unit (`한&nbsp;번`, `두&nbsp;줄`, `세&nbsp;가지`) so the line never breaks inside the count. Bind nothing else with `&nbsp;`: each one makes a longer unbreakable run and a more ragged right edge. Before finishing, run `grep -nP '[A-Za-z0-9_)]+(</code>)? (은|는|이|가|을|를|의|에|와|과|로|으로|도|만|에서|에는|에도|인|인데)(?=[ ,.<)]|$)'` on the file and fix every hit that is a particle (`이` and `가` can also be a word of their own), except inside source quoted verbatim in a code block or an evidence card, which keeps its spacing.
- **No metaphor, no implied meaning, no translationese in titles and labels.** A heading, a chip, a row caption, a sub-line, a tooltip states the fact in the subject's own words (`vector_store_file 1행`, `호출 최대 N회`, `원본 테이블`), terse 개조식 welcome. Not `장부`, `선반`, `고리를 돈다`, `~의 여정`, `한눈에 보는 ~`, `숨은 비용`, `마법은 없다`. A phrase that would sound odd said aloud to a colleague is cut; a phrase that could head a page about a different subject is too abstract.
- **Headings are answers, not frames; plan them in the output language.** (shared across eli5, explain-diff, micro-world, visual-doc) Write the section plan in the language the page will be in, with the final heading text, before any HTML: a heading drafted in English and translated comes out as `시험을 어떻게 내는가` or `점수 읽는 법`. The patterns below are the tells, and each is replaced by the fact it was hiding. A question (`~하는가`, `~인가`, `~일까`, `왜 ~인가`) becomes its answer: `점수를 어떻게 읽는가` is `점수 세 종류: pass@1, 승률, 시간 지평`. A classification noun (`N축`, `~한 축`, `차원`, `관점`, `레이어`, `측면`) becomes the names of the things: `평가의 4축` is `채점 방식 3가지: 테스트, 루브릭, 사람 비교`. A method noun (`~하는 법`, `~읽는 법`, `~하려면`) becomes the subject and its values. A bare `A vs B` becomes the value that differs: `pass@1 vs pass^k` is `pass@1은 1회 성공, pass^k는 k회 연속 성공`. An emphasis frame (`핵심은`, `결국`, `본질적으로`, `진짜`, `단순히 ~가 아니다`, `~에 불과`) is deleted. A count is counted after the items exist, never chosen first and padded (`세 가지`). A metaphor that frames the whole subject (`모의고사`, `시험지`, `장부`) stays out of every heading and out of the plan. None of these words is banned in body prose, where `A vs B` or `~하려면` can be the plain sentence: the rule is about headings, captions, outline lines, and the plan, which is why the Render check's heading grep reads only heading tags and reports a reading list, not failures.
- **Rows by default; side by side only when the render proves it fits.** (shared across eli5, explain-diff, micro-world, visual-doc) When two things are shown for comparison (two states, two options, two code blocks, a request and its response, two evidence cards), stack them as rows: each keeps the full column width and the eye reads down instead of jumping across. Side by side is a judgment call, not a ban, and it is judged on the rendered page, not by counting characters: it stays only when nothing inside either half scrolls sideways and the reader gains from seeing the two at the same height (two short lists, two table schemas, a control and the view it changes). To measure, append `measure.html` from this skill's base directory to a copy of the page and read the result with the one-line command in its header comment: `data-sbs` names every multi-column grid with something scrolling sideways inside it, and every block it names is stacked. Without a browser there is no measurement, so stack.
- **Both light and dark, switchable.** The page follows the system setting by default, and the fixed `.theme-btn` in the top-right corner lets the reader override it. Keep three things from the template verbatim and together: the button markup right after `<body>`, the `.theme-btn` CSS, and the small theme `<script>` in `<head>` (the shipped toggle icon is the one sanctioned SVG icon). The dark tokens are defined twice on purpose (`@media (prefers-color-scheme: dark)` on `:root:not([data-theme="light"])`, and `:root[data-theme="dark"]`); keep the two blocks identical and never drop one, or the toggle stops winning in one direction. Any CSS you author yourself takes every color from a token: `var(--surface)` for a raised white surface, `var(--on-accent)` for text on a `--blue`/`--green` fill, `var(--track)` and `var(--shadow-sm)` for tracks and small shadows. A literal `#fff`, a black-alpha `rgba(...)`, or an inline `style="color:..."` with a hex value reads in one theme only. The terminal is the one deliberate exception: `.term`, its header (`.term-head`), and the action palette are a dark surface in both themes and keep their own `--term-*` tokens and white-alpha borders; leave them as shipped. When the embodiment is not a terminal, only the interior you put inside `.term-body` follows the page theme, through the tokens above (override `.term-body`'s terminal defaults on your own modifier class). Content that must look the same in both real themes, such as a simulated screen with its own light/dark state, may use the `--term-*` tokens, which do not flip. Before finishing, confirm all three parts of the toggle survived: `grep -c '<button type="button" class="theme-btn"'`, `grep -c '\.theme-btn{'`, and `grep -c 'git-claw-doc-theme'` must each print at least 1 (a bare `theme-btn` match can be prose that merely mentions it).

## Step 4: Verify the Render

### Render check (shared across eli5, explain-diff, micro-world, visual-doc)

The rules are the first defense; looking at the rendered page is the second, and a defect counts as fixed only when the render shows it fixed. Work in a scratch directory of your own (`D`, e.g. `/tmp/micro-world-<slug>`); never a shared fixed path, because other runs may be writing there at the same time. Find a browser first (`which google-chrome google-chrome-stable chromium chromium-browser`). Do not rely on `npx --no-install playwright screenshot`: with no browser binary installed it prints an install banner, exits 0, and writes no file.

1. **Screenshot both themes.** The window must be taller than the page; a page cut at the bottom looks finished, so raise the height if the last block is missing.
   ```bash
   google-chrome --headless=new --no-sandbox --disable-gpu --hide-scrollbars --window-size=1300,12000 --screenshot=D/light.png "file://<abs-path>"
   google-chrome --headless=new --no-sandbox --disable-gpu --hide-scrollbars --force-dark-mode --window-size=1300,12000 --screenshot=D/dark.png "file://<abs-path>"
   ```
2. **Look at every piece.** Crop each screenshot to the content (the window is taller than the page and the rest is blank), cut it into pieces about 1,800px tall and read them all; a page-sized thumbnail hides a clipped label, a chip split across two lines, and a lone word on a last line.
   ```bash
   python3 -c "from PIL import Image,ImageChops; im=Image.open('D/light.png').convert('RGB'); b=ImageChops.difference(im,Image.new('RGB',im.size,im.getpixel((im.width-1,im.height-1)))).getbbox(); h=min(im.height,(b[3] if b else im.height)+40); [im.crop((0,y,im.width,min(y+1800,h))).save(f'D/light-{i}.png') for i,y in enumerate(range(0,h,1800))]"
   ```
   If Pillow is not installed, look at the full screenshots and say in the report that the check was at thumbnail size.
3. **Measure.** Run `measure.html` from this skill's base directory (the command is in its header comment) at the reading width and again at `--window-size=500,2000` (headless Chrome does not go narrower than 500px, so a 420px request silently measures 500). The `<body>` tag must carry `data-sbs`, `data-fit`, and `data-ovf`, all `none`; a body tag without them means the measurement never ran, which is a failure, not a pass.
4. **Text.** Run every grep in this skill with `LC_ALL=C.UTF-8` (under the `C` locale `[—–·]` matches every Hangul line). `grep -nP '[—–·]'` and the particle grep from the design rules print nothing outside verbatim source.
5. **Headings.** `grep -nP '<(h[1-3]|span class="dg-cap")[^>]*>[^<]*(는가|인가|일까|나요|\?|[0-9]축|한 축|하는 법|읽는 법|하려면|핵심|본질|결국| vs )' <file>` lists the headings and figure captions that match a translation tell. This is a reading list, not a failure: read each hit aloud, rewrite it only when it reads as a frame rather than a fact, and name in the report each hit you kept and why. Body prose is not searched.

6. **Voice.** `LC_ALL=C.UTF-8 python3 -c "import re,html;s=open('<file>',encoding='utf-8').read();t=' '.join(html.unescape(re.sub('<[^>]+>',' ',re.sub(r'<code>.*?</code>','C',x))) for m in re.findall(r'<div class=\"note\">(.*?)</div></div>|<p class=\"(?:say|lede|prose|takeaway)[^\"]*\">(.*?)</p>',s,re.S) for x in m if x);S=[x.strip().rstrip('.!?)') for x in re.split(r'(?<=[.!?])\s+',t) if x.strip()];print(len(S),'sentences;',sum(1 for x in S if re.search(r'(요|죠)$',x)),'in 해요체; not:',[x[-28:] for x in S if re.search(r'(다|음|함|됨|임|것|꼴)$',x)])"` counts the sentences of the page's prose (add this skill's own prose classes to the alternation if they differ) and lists the ones that end in 한다체 or in a bare noun. The list is a defect list: every entry outside a verbatim quote is rewritten in 해요체 as a full sentence, and the count of 해요체 sentences must be the sentence count minus the quotes.
Fix and repeat until 1 to 4 and 6 pass and the fifth has been read. Only when no browser exists at all, say plainly that the page was not visually verified. Screenshots, crops, and the measurement copy live in `D`, are not part of the output, and are deleted when you finish.

Then check: pressing `→` from a fresh page walks the whole scenario; something the reader can see changes at every step (the world-state panel, the environment, or both; a read-only command may change only the environment); dark mode holds (the terminal panel stays dark in both themes). The page changes as the reader steps through it, so run the measurement at the start, the middle, and the last step by adding `#steps=N` to the measured URL (the header of `measure.html` shows how).

## Step 5: Output

1. Write the file to the **repository root**: `micro-world-<slug>.html` (slug from the behavior, kebab-case).
2. Report the **absolute path**. Do NOT auto-open, do NOT commit, do NOT gitignore. Delete only when the user asks: these are throwaway artifacts by design.
3. In the report, state in one line how to drive it ("→ 키만 눌러도 전체 사이클을 완주할 수 있어요").

**Constraints:**

- Read-only with respect to git state: exactly one HTML file is created (plus the scratch files of the Render check, deleted when it finishes).
- Visual language is fixed by the template and the design-system record (`docs/decisions/2026-07-explain-diff-micro-world-design.md`). Do not improvise styles.
- Never present the sim as proof of correctness: it is an understanding aid, not a test. If the sim surfaces a suspected bug, report it as an attention pointer and suggest a code review.
