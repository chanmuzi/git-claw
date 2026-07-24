---
name: micro-world
description: >-
  Build a throwaway interactive HTML simulation (a "micro-world") of a specific behavior — a state machine, algorithm, data transform, or protocol flow — so the developer can inhabit and feel how the code works instead of reading about it.
  TRIGGER when: user asks to simulate, play with, or interactively explore a behavior (e.g., "micro-world 만들어줘", "이거 시뮬로 보여줘", "만져볼 수 있게 만들어줘", "동작을 인터랙티브로 이해하고 싶어").
  DO NOT TRIGGER when: user wants a document explaining a diff (use explain-diff), a code review (use code-review), or the change is config/rename/dependency-bump material with no behavior to inhabit.
version: "1.0.1"
allowed-tools: Bash(git *), Bash(gh *), Read, Grep, Glob, Write
---

## The One Principle

**Let the developer live inside the behavior.** A micro-world (after Seymour Papert's Mathland) is an ephemeral, purpose-built interactive environment where understanding comes from manipulating the system, not reading about it. It is bespoke by nature: the embodiment is designed per behavior, only the visual language is fixed.

This skill is a sibling of `explain-diff`: that one produces a consistent document; this one produces a one-off simulation. They share design tokens, never structure.

## Step 0: Applicability Gate — decline honestly when a sim adds nothing

Micro-worlds shine when the subject has **state that evolves under input**:

- state machines and transitions (auth flows, retries, lifecycle, cache/deploy procedures)
- algorithms (scheduling, matching, caching, parsing — anything steppable)
- data transforms (pipeline stages, before/after shapes)
- visual/spatial logic (layout, geometry, rendering)
- protocol/interaction flows (request/response ordering, races)

They add nothing for config changes, renames, dependency bumps, doc edits, or straight-line glue code. In those cases, say so plainly — "이 변경은 시뮬레이션으로 얻는 게 거의 없어요" — and suggest `/explain-diff` instead. Do not manufacture a hollow sim to satisfy the request.

Set expectations once at the start: micro-worlds are **best-effort** — quality tracks how simulatable the behavior is.

## Step 1: Identify the World

Read the actual code (diff, PR, file, or the behavior the user described). Extract:

1. **State variables** — what the world tracks (queue contents, token expiry, cache directories).
2. **User inputs** — the actions the developer performs (fire an event, run a command, advance time).
3. **Observable outcomes** — what visibly changes in response (state chips, file trees, logs, shapes).
4. **The one confusion** this sim exists to dissolve — name it; it drives every design choice. **One confusion = one world.** If the subject holds several distinct confusions, build several small worlds rather than one bloated page.

## Step 2: Design the Scenario and Embodiment

### Guided single cycle — no free-form exploration

The sim runs **one authored scenario**, framed as a mission:

- A **mission bar** states 2-3 concrete goals as check chips that tick as the state satisfies them.
- At every moment exactly **one recommended next action** is enabled and visually highlighted; all other actions are disabled. The user cannot wander off the narrative — free-form mode is intentionally absent, because unguided branching dilutes understanding.
- The scenario should route **through the failure the world exists to teach** (the trap fires mid-story, then the story recovers), not around it.
- An **inline restart** lives in the action palette itself, styled like the other actions (`↻ 시나리오 처음부터`); it gets the recommendation highlight when the mission completes so the natural next move is another cycle. No detached reset link far from the controls.

### Playback: every action is a step

- Each completed action pushes a **full snapshot** (environment output, narration, world state). `‹` `›` controls in the environment header move through them; going back and acting anew truncates forward history.
- **Forward is a play button**: when there is no forward history, `›` executes the recommended next action — so from a fresh page, pressing `→` repeatedly walks the entire scenario.
- Keyboard: `←` back, `→` forward/execute, `R` restart (also match the Korean IME key, e.g. `ㄱ` for R). Ignore modifier-key combos; disable during output animation.

### Fact / interpretation separation

- The environment (terminal, canvas, timeline) shows **only faithful system output** — what the real tool would actually print or display. No judgment coloring, no commentary inside it.
- All meaning lives in a separate **narration strip** directly below the environment: label ("지금 무슨 일이") + one bold core sentence + at most 2 bullets, with the whole strip tinted semantically (ok/warn/err). This is the same visual language as explain-diff's takeaway cards.

### World state panel

- Group all observable state into **one card** ("세계 상태") with internally divided sub-panels (e.g., filesystem tree | session memory), never scattered sibling cards.
- Changed values give brief visual feedback (a pulse, an enter animation) so the action → state connection is legible.

### Embodiment is chosen by the subject

The interaction rule is absolute — **the user's input must change the outcome**; a step-through that reveals already-visible content in order is banned. Beyond that, pick the embodiment that matches the behavior:

- **Terminal command center** — CLI procedures, deploy/ops flows: dark terminal pane with a command palette (real commands as the actions, `❯` shell / `✳` agent prompts), full width and dominant.
- **Timeline scrub** — execution traces: drag through steps, each position shows real internal state (stack, bindings, queue).
- **Drag-and-drop + side-by-side structure views** — structural reorganizations: manipulate one side, watch the other (e.g., directory trees, schemas) evolve in parallel.
- **Parameter knobs** — formulas and thresholds: sliders/toggles feeding the real computation, outputs re-render live.

Fidelity rule: the sim must model the **actual logic** — port the real algorithm/state machine into inline JS where feasible. Distinctions that exist in the real system (e.g., "copy changes disk, reload changes the session") must exist in the model. Any simplification must be disclosed in a **small footnote below the sim** (plain faint text — never an alarm-colored callout).

## Step 3: Build the Page

Read `template.html` from this skill's base directory — it carries the design tokens, the command-center frame (mission bar, terminal + palette + step nav, narration strip, world-state card), and the **generic runtime JS** (output helpers, snapshot/restore history, keyboard playback). **Fill the bespoke parts; do not restyle the frame.** For a non-terminal embodiment, replace the environment pane's interior while keeping the frame (mission bar, narration strip, world card, playback) intact. No emoji, no hand-drawn SVG icons, no external resources. Single self-contained HTML file.

- Top: `micro-world` eyebrow + title naming the behavior + lede stating the mission and how to drive it (one line).
- Close with a `핵심 정리` takeaway card: what the user should have felt after playing.
- Korean output uses 해요체; content language follows the project's AGENTS.md (or CLAUDE.md as fallback) setting, else the user's conversational language.

### Design rules (shared across micro-world, explain-diff, visual-doc)

- **Title never wraps mid-word.** `word-break: keep-all` stays on the title and narrative text so a Korean particle (`로`, `를`, `이`) can never fall to the start of a line. Keep the title a short noun phrase.
- **No decorative gradients.** Backgrounds are solid tokens. A gradient is allowed only when functional (a fade scrim), never as panel decoration.
- **No em-dash or en-dash (`—`, `–`) anywhere in the output.** They are a machine-writing tell. Use a colon, parentheses, a comma, or split into two sentences.
- **Natural Korean, not machine translation.** Do not coin stiff 한자어 Koreans do not say; keep a code/identifier term verbatim rather than force-translate it.

## Step 4: Output

1. Write the file to the **repository root**: `micro-world-<slug>.html` (slug from the behavior, kebab-case).
2. Report the **absolute path**. Do NOT auto-open, do NOT commit, do NOT gitignore. Delete only when the user asks — these are throwaway artifacts by design.
3. In the report, state in one line how to drive it ("→ 키만 눌러도 전체 사이클을 완주할 수 있어요").

**Constraints:**

- Read-only with respect to git state — exactly one HTML file is created.
- Visual language is fixed by the template and the design-system record (`docs/decisions/2026-07-explain-diff-design-system.md`). Do not improvise styles.
- Never present the sim as proof of correctness — it is an understanding aid, not a test. If the sim surfaces a suspected bug, report it as an attention pointer and suggest a code review.
