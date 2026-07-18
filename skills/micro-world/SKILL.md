---
name: micro-world
description: >-
  Build a throwaway interactive HTML simulation (a "micro-world") of a specific behavior — a state machine, algorithm, data transform, or protocol flow — so the developer can inhabit and feel how the code works instead of reading about it.
  TRIGGER when: user asks to simulate, play with, or interactively explore a behavior (e.g., "micro-world 만들어줘", "이거 시뮬로 보여줘", "만져볼 수 있게 만들어줘", "동작을 인터랙티브로 이해하고 싶어").
  DO NOT TRIGGER when: user wants a document explaining a diff (use explain-diff), a code review (use code-review), or the change is config/rename/dependency-bump material with no behavior to inhabit.
version: "1.0.0"
allowed-tools: Bash(git *), Bash(gh *), Read, Grep, Glob, Write
---

## The One Principle

**Let the developer live inside the behavior.** A micro-world (after Seymour Papert's Mathland) is an ephemeral, purpose-built interactive environment where understanding comes from manipulating the system, not reading about it. It is bespoke by nature: structure and interaction are designed per behavior, only the visual language is fixed.

This skill is a sibling of `explain-diff`: that one produces a consistent document; this one produces a one-off simulation. They share design tokens, never structure.

## Step 0: Applicability Gate — decline honestly when a sim adds nothing

Micro-worlds shine when the subject has **state that evolves under input**:

- state machines and transitions (auth flows, retries, lifecycle)
- algorithms (scheduling, matching, caching, parsing — anything steppable)
- data transforms (pipeline stages, before/after shapes)
- visual/spatial logic (layout, geometry, rendering)
- protocol/interaction flows (request/response ordering, races)

They add nothing for config changes, renames, dependency bumps, doc edits, or straight-line glue code. In those cases, say so plainly — "이 변경은 시뮬레이션으로 얻는 게 거의 없어요" — and suggest `/explain-diff` instead. Do not manufacture a hollow sim to satisfy the request.

Set expectations once at the start: micro-worlds are **best-effort** — quality tracks how simulatable the behavior is.

## Step 1: Identify the World

Read the actual code (diff, PR, file, or the behavior the user described). Extract:

1. **State variables** — what the world tracks (queue contents, token expiry, pointer positions).
2. **User inputs** — what the developer should be able to manipulate (fire an event, tweak a parameter, advance time, reorder requests).
3. **Observable outcomes** — what visibly changes in response (state chips, timelines, logs, shapes).
4. **The one confusion** this sim exists to dissolve — name it; it drives every design choice.

## Step 2: Design the Interaction

The interaction rule is absolute: **the user's input must change the outcome.** A step-through that reveals already-visible content in order is banned. Pick whatever shape fits — these are proven patterns, not a menu limit:

- **Timeline scrub** — drag through execution; each position shows real internal state (stack, bindings, queue).
- **Step executor** — buttons fire actual transitions; state chips and a log update per step; illegal inputs visibly rejected.
- **Before/after side-by-side** — two panes running old vs new logic on the same user-chosen input.
- **Parameter knobs** — sliders/toggles feeding the real formula; outputs re-render live.

Fidelity rule: the sim must model the **actual logic** — port the real algorithm/state machine into inline JS where feasible. Any simplification must be visibly labeled in the page (e.g., "실제 구현은 재시도 3회지만 여기선 1회로 단순화했어요"). A sim that behaves differently from the code teaches the wrong thing — that is worse than no sim.

## Step 3: Build the Page

Read `template.html` from this skill's base directory — a shell carrying the design tokens, base components (cards, buttons, segmented control, state chips, code block), and the page frame. **The body is yours to design; the look is not.** No emoji, no hand-drawn SVG icons, no external resources (CDN, webfonts, remote images). Single self-contained HTML file.

Fixed frame, free body:

- Top: `micro-world` eyebrow + title naming the behavior + 1-2 sentence lede stating what to try first.
- One or more cards containing the bespoke sim UI and inline JS.
- Close with a short `핵심 정리` takeaway card: what the user should have felt after playing (same component as explain-diff — blue tinted card = "the thing to remember").
- Korean output uses 해요체; content language follows the project's CLAUDE.md/AGENTS.md setting, else the user's conversational language.

## Step 4: Output

1. Write to the **repository root**: `micro-world-<slug>.html` (slug from the behavior, kebab-case).
2. Report the **absolute path**. Do NOT auto-open, do NOT commit, do NOT gitignore. Delete only when the user asks — these are throwaway artifacts by design.
3. In the report, state in one line what interaction to try first ("타임라인을 끝까지 끌어보세요").

**Constraints:**

- Read-only with respect to git state — exactly one HTML file is created.
- Visual language is fixed by the template and the design-system record (`docs/decisions/2026-07-explain-diff-design-system.md`). Do not improvise styles.
- Never present the sim as proof of correctness — it is an understanding aid, not a test. If the sim surfaces a suspected bug, report it as an attention pointer and suggest a code review.
