---
name: explain-diff
description: >-
  Generate a self-contained interactive HTML explainer for a code diff, commit, branch, or PR so the developer genuinely understands the change before sharing or merging it.
  TRIGGER when: user asks to explain a diff/PR/commit/changes or wants an understanding document (e.g., "diff 설명해줘", "이 변경 이해하게 해줘", "explain this PR", "변경사항 설명 문서 만들어줘").
  DO NOT TRIGGER when: user wants defect findings or a review verdict (use code-review), is committing or creating PRs, or asks a quick question about a specific line that a direct answer serves better.
version: "1.0.0"
allowed-tools: Bash(git *), Bash(gh *), Read, Grep, Glob, Write
---

## The One Principle

**Build genuine understanding first, then explain it.** The document exists so the reader can pass a quiz about the change without opening the files — and honestly could not before reading it. Inspired by Geoffrey Litt's explain-diff: understanding, not correctness, is the bottleneck of agent-written code.

The default reader is the **author-practitioner**: someone who works on this codebase and is about to review, merge, or defend this change. Skip beginner background; include behavior deltas, failure modes, and design rationale.

Honesty rules apply throughout:

- Never present a suspicion as a confirmed bug — frame it as an attention pointer.
- Distinguish stated intent (commit messages, PR body) from your inferred interpretation.
- Quote code **verbatim only** — never paraphrase, re-indent, or reconstruct from memory. Every quoted block must survive comparison against the actual file.
- Do not invent content for a section with nothing to say — omit the section.

## Parse Arguments

| Argument | Type | Description |
|----------|------|-------------|
| (none) | — | Explain the working diff: `git diff HEAD` plus branch commits vs the default branch |
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
5. Group hunks into **narrative themes ordered by importance** — one theme per Part 2 card. A theme is a behavioral unit ("token refresh moved to interceptor"), never a file list.

## Step 3: Build the Document

Read `template.html` from this skill's base directory. It carries the full design system (tokens, components, generic quiz/gate JS) — **fill it, never restyle it**. No emoji, no hand-drawn SVG icons, no external resources (CDN, webfonts, remote images). The output must stay a single self-contained HTML file.

Structure is fixed at three parts:

**Top + TOC.** Title states the change as an action ("~를 ~로 바꿨어요"), lede gives 2-3 sentences of what/why. Metabar: target, commit range, file count, +/- totals, date. Each TOC row's description is that section's **one-line takeaway** — reading only the TOC must summarize the whole change. Keep the Part group headers.

**Part 1 — 개요.**
- `배경`: the before-state and its cost, then what the change achieves. Behavior level, not file level.
- `구조 한눈에 보기`: annotated directory tree of every touched path. Chips state *what happened* (`정본으로 승격`, `67줄 → 1줄`, `삭제 (렌더러)`), deleted files get strikethrough. No section-jump links on tree rows.
- Optional interactive figure: include ONLY when an interaction exists where **the user's input changes the outcome** — a scenario toggle, a state switch whose columns respond differently (e.g. a start-location toggle showing two agents' loading paths side by side). Never build a step-through that reveals already-visible content in order; if no outcome-changing interaction exists for this diff, omit the figure entirely.

**Part 2 — 변경 사항.** One card per theme from Step 2, ordered by importance:
- Prose explaining behavior meaning — what the reader must understand, not a line-by-line narration.
- Verbatim diff excerpts (12 lines max per block; elide the middle with `…`). Only the lines that carry the theme.
- Close every card with a `핵심 정리` takeaway card: one bold sentence to remember + 2-3 supporting bullets. This is the same visual language as quiz explanations — blue tinted card = "the thing to remember".

**Part 3 — 이해 점검.**
- `주의해서 볼 지점`: risk rows with Watch/Critical badges. Attention pointers, never verdicts. If a risk deserves a verdict, tell the user to run a code review — do not deliver one here.
- Quiz (below).

## Quiz Specification

Exactly **5 questions** by default, medium-hard. Each question must test practitioner-critical understanding — something that would change how the reader reviews, merges, or maintains this code:

- behavior prediction ("if X happens, what does the system actually do now?")
- failure modes ("which path is still unprotected?")
- rejected alternatives ("why was Y not chosen?" — only when the record/PR states it)
- invariants ("what does the check actually guarantee, and what not?")
- future-maintainer scenarios ("six months later, someone does Z — what breaks?")

Trivia (line counts, file names, dates) is banned. Distractors must be **plausible misconceptions**: the old system's behavior, a wrong-layer attribution, a swapped condition — never obviously absurd fillers.

Mechanics (already implemented by the template JS — supply only content):

- Exactly one option per question has `data-ok="true"`.
- `data-hint` points to the section to re-read and **never reveals the answer**.
- `data-hint-href` is that section's own id (`#sec2`, `#risks`, …) and `data-hint-label` its title — together they turn the hint into a one-click jump, so the reader never scrolls back by hand. The id must exist in the document; the template drops the link if it does not resolve. Omit both only when no single section covers the question.
- Wrong answer → hint + retry (that question only); correct → explanation + lock.
- Explanation card: one bold core fact + bullets, including why the tempting wrong answer is wrong.
- The bottom gate opens only at 5/5. Its wording states understanding — never a fake action the page cannot perform (no "PR 보내기" buttons).

## Step 4: Output

1. Write the file to the **repository root**: `explain-diff-<slug>.html` (slug from the change topic, kebab-case).
2. Report the **absolute path** to the user. Do NOT auto-open it, do NOT commit it, do NOT add it to .gitignore. Delete it only when the user asks.
3. Content language follows the project's AGENTS.md (or CLAUDE.md as fallback) language setting; if none, the user's conversational language. Korean output uses 해요체 throughout (headings may stay nominal).

**Constraints:**

- This skill is read-only with respect to git state — it creates exactly one HTML file and nothing else.
- Design decisions are fixed by the template and the design-system record (`docs/decisions/2026-07-explain-diff-design-system.md` in the git-claw repo). Do not improvise styles.
