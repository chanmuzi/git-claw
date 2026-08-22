---
name: eli5
description: >-
  Explain a subject to someone who knows nothing about it, as a self-contained HTML page where big diagrams carry the meaning and the prose is reduced to one line per picture.
  TRIGGER when: user types /eli5 <topic>, or asks for a dead-simple picture explainer aimed at someone outside the domain (e.g., "eli5 해줘", "아무것도 모르는 사람한테 설명하듯 만들어줘", "그림으로 쉽게 풀어줘", "비전공자한테 보여줄 자료로 만들어줘").
  DO NOT TRIGGER when: the reader is the developer who has to merge the change (use explain-diff), the subject is a behavior worth inhabiting interactively (use micro-world), the reader is domain-literate and wants the findings structured (use visual-doc), or a code review verdict is wanted (use code-review).
version: "1.0.0"
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

Plan the page as **a sequence of pictures**, each with a one-line caption, before writing any HTML. Five to eight beats is the usual range; fewer is fine, more usually means two pages.

The default arc, which fits most technical subjects:

1. **The problem** — draw the thing failing or blocked. Start here, never with the solution.
2. **The obvious fix** — the naive approach a reasonable person would try.
3. **The trap** — why the obvious fix breaks. **This beat is the point of the whole page.** It is also the beat most likely to be buried in one line of a table in the source material, so dig for it.
4. **The real fix** — the mechanism actually chosen, drawn as a mechanism.
5. **The cost** — what it takes in time, money, or complexity. Never omit.
6. **The rule** — the one thing a reader should remember about when this applies.

Adapt the arc when the subject is not a change (a "how does X work" page may go: what goes in, what happens to it, what comes out, what breaks). Keep the failure beat wherever it exists.

**One picture, one claim.** If a drawing needs two captions, it is two drawings.

## Step 3: Draw the Mechanism

The drawings are hand-authored inline SVG, one per beat.

- **Depict the mechanism, not its name.** A box labeled "cache" says less than the prose. Draw the path a request takes, the two stores it sits between, the arrow that disappears when the cache is gone.
- **Comparing options? Draw the difference.** Two states side by side with the one edge that changes between them. Separate labeled boxes with nothing connecting them is a restated list, not a comparison.
- **Label the arrows.** An unlabeled arrow means "related somehow". `writes`, `20장씩`, `19배 압축` is information.
- **No jargon inside the picture.** Identifiers, flag names, and function names belong in the caption or the footer, never as a shape's label. A drawing that needs `use_doc_chunking` to be understood is drawn for an insider.
- **Draw the reader's world, not the code's.** Pages, forms, queues, doors, votes. Concrete nouns the outsider already owns.
- **Size by `viewBox`** (`viewBox="0 0 W H"`, CSS `width:100%; height:auto`), pick W and H for the content, and align shapes to a shared grid. Eyeballed offsets read as noise.
- **Theme with `currentColor`.** Strokes and text inherit the page's foreground so both themes work. Reserve `var(--blue)` for the one element carrying the point of that picture and `var(--red)` for the thing that breaks. Two accent colors per drawing at most.
- **Text stays around 11-13px at drawn scale**, a word or three per label. Explanatory sentences go in the caption below the figure, never inside the drawing.
- **Every figure gets** `role="img"` and an `aria-label` stating the same claim as its caption.
- No `<script>`, `<style>`, or `<foreignObject>` inside the SVG. No emoji, no icon fonts, no external resources. Long decorative path data means the drawing is too elaborate: simplify it.

### The word budget

- One `.say` line per beat: **one sentence**, occasionally two. It names what the picture showed; it does not re-explain it.
- No paragraphs of body prose anywhere on the page. If a beat genuinely needs three paragraphs, the subject wants `visual-doc`.
- Captions (`figcaption`) may carry one extra factual detail the drawing could not hold.
- Numbers appear as before/after pairs in the number strip, four at most, and at least one of them is a cost.

## Step 4: Build the Page

Read `frame.html` from this skill's base directory. It carries the locked design tokens (shared with `explain-diff`, `micro-world`, `visual-doc`), the page frame, the beat primitives (`.hero`, `.beat`, `.fig`, `.say`, `.numbers`, `.rule`, `.foot`), and the SVG label classes with a worked example drawing. **Copy the tokens and primitives verbatim, author the drawings bespoke.** Every page has different pictures; that is the whole point.

Output is a single self-contained HTML file. Fill `<title>`: a short noun phrase naming the subject, not a summary.

### Design rules (shared across eli5, explain-diff, micro-world, visual-doc)

- **Title never wraps mid-word.** `word-break: keep-all` stays on titles and narrative text so a Korean particle (`로`, `를`, `이`) can never fall to the start of a line. Titles are short noun phrases.
- **No decorative gradients.** Backgrounds are solid tokens. A gradient is allowed only when functional (a fade scrim), never as panel decoration.
- **No em-dash or en-dash (`—`, `–`) anywhere in the output.** Not in titles, prose, captions, SVG labels, anywhere. They are a machine-writing tell. Use a colon, parentheses, a comma, or split into two sentences.
- **Color must survive its background.** A mark carrying meaning (a legend swatch, a status dot, an emphasized stroke) must contrast with the surface under it in both themes.
- **Both light and dark.** The tokens carry a `prefers-color-scheme: dark` override. Never hardcode a color that breaks one mode.
- **Natural Korean, not machine translation.** Do not coin stiff 한자어 Koreans do not say; keep a code identifier verbatim rather than force-translate it. Read every caption aloud once: if it sounds like a translated subtitle, rewrite it plainer.

## Step 5: Verify the Render

The rules are the first defense; looking at the page is the second. If a headless browser is available:

```bash
npx --no-install playwright screenshot --full-page "file://<abs-path>" /tmp/eli5-check.png
```

Then check, in this order:

1. **The pictures-only pass.** Look at the drawings and ignore every sentence. Does the idea still come through? This is the skill's actual success criterion, and it is the one check that cannot be skipped.
2. No drawing is cut off, overlapping, or scaled into illegibility.
3. Titles do not wrap mid-word; no em-dash survived (`grep '—\|–'` the file).
4. Dark mode holds (`--color-scheme=dark`).

Fix and re-render until clean. If no browser is available, still do check 1 by reading your own SVG, and say the render was not visually verified.

## Step 6: Output

1. Write the file to the **repository root**: `eli5-<slug>.html` (slug from the subject, kebab-case). If not in a repo, write to the current directory.
2. Report the **absolute path**. Do NOT auto-open, do NOT commit, do NOT add to .gitignore. Delete only when the user asks.
3. State the one idea in a single line in the report, so the user can check it matches what they wanted explained.
4. Content language follows the project's AGENTS.md (or CLAUDE.md as fallback) setting; if none, the user's conversational language. Korean output uses 해요체.

**Constraints:**

- Read-only with respect to git state: this skill creates exactly one HTML file and nothing else.
- Tokens are fixed by `frame.html` and the design-system record (`docs/decisions/2026-07-explain-diff-design-system.md`, `docs/decisions/2026-08-eli5.md`). Do not improvise styles.
- Never present the page as complete documentation or proof of correctness. It is an on-ramp for an outsider, and it says so by being short.
- If the request is really a diff explainer (`explain-diff`), a behavior to inhabit (`micro-world`), or a structured report for a colleague who already knows the domain (`visual-doc`), say so and hand off rather than forcing an eli5.
