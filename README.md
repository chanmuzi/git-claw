<p align="right">
  <a href="README.md">English</a> · <a href="README.ko.md">한국어</a>
</p>

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/banner-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="assets/banner-light.svg">
    <img src="assets/banner-light.svg" alt="git-claw" width="700">
  </picture>
</p>

<h1 align="center">git-claw</h1>
<p align="center">Agent Skills for consistent Git workflows</p>

<p align="center">
  <img src="https://img.shields.io/badge/skills-8-8B5CF6?logo=git&logoColor=white" alt="Skills" />
  <a href="https://agentskills.io"><img src="https://img.shields.io/badge/Agent_Skills-compatible-0EA5E9?logo=robotframework&logoColor=white" alt="Agent Skills" /></a>
  <img src="https://img.shields.io/github/license/chanmuzi/git-claw?color=blue" alt="License" />
  <img src="https://img.shields.io/github/last-commit/chanmuzi/git-claw?color=orange" alt="Last Commit" />
</p>

---

A consistent Git history that's easy to read — for both humans and agents.

git-claw is an [Agent Skill](https://agentskills.io) that keeps your commits, PRs, issues, and code reviews in a unified format. Install once, and every collaboration artifact follows the same conventions — whether you're working solo or in a team.

- **Readable history** — Structured commits, PRs, and issues that anyone (or any agent) can follow at a glance
- **Multi-model code review** — Domain agents + Codex adversarial analysis, cross-validated and severity-ranked
- **Session continuity** — `/handoff` captures work context so the next session picks up where you left off

---

### Skills

| | Skill | What it does |
|:---:|---|---|
| 📝 | `/commit` | Conventional commits from your diff |
| 🔀 | `/pr` | Structured PRs with auto-labeling |
| 📋 | `/issue` | Templated issues with priority labels |
| 💬 | `/review-reply` | Analyze and reply to review comments |
| 🔍 | `/code-review` | Multi-agent severity-based code review |
| 🤝 | `/handoff` | Session transfer prompt generation |
| 📖 | `/explain-diff` | Interactive HTML explainer for understanding a diff |
| 🧪 | `/micro-world` | Throwaway interactive simulation of a behavior |

## Installation

### Claude Code Plugin (Recommended)

```bash
# Add marketplace
/plugin marketplace add chanmuzi/git-claw

# Install plugin
/plugin install git-claw@git-claw
```

### Skills CLI

```bash
npx skills add chanmuzi/git-claw
```

The interactive installer lets you select skills, target agents, scope (project/global), and install method.

<details>
<summary>Install all skills at once without prompts</summary>

```bash
npx skills add chanmuzi/git-claw --skill commit --skill pr --skill issue --skill review-reply --skill code-review --skill handoff --skill explain-diff --skill micro-world -g
```

</details>

### Update

**Claude Code Plugin:**

```bash
/plugin update git-claw@git-claw
```

Or enable auto-update: `/plugin` → **Marketplaces** tab → select marketplace → **Enable auto-update**

**Skills CLI:**

```bash
npx skills check    # Check for updates
npx skills update   # Update all installed skills
```

> Symlink installs (recommended) apply updates to all agents at once. Copy installs require updating each copy individually.

> **Skill renames are not auto-tracked.** When a skill is renamed upstream, the Skills CLI keeps the old folder around, so you may see two skills with the same description, or the skill may surface under its old name. Remove the stale folder manually — **match the scope you installed with**:
> ```bash
> # Global install (npx skills add ... -g)  →  ~/.codex/skills/{old-name}
> npx skills remove {old-name} -g     # or
> rm -rf ~/.codex/skills/{old-name}
>
> # Project install (npx skills add ... without -g)  →  ./.codex/skills/{old-name}
> npx skills remove {old-name}        # or
> rm -rf ./.codex/skills/{old-name}
> ```
> Then restart your Codex CLI session so the skill list is re-cached.

## Skills

> **Skill prefix by agent:** Claude Code uses `/` (e.g., `/commit`), Codex CLI uses `$` (e.g., `$commit`).

### `/commit` — Create a Git Commit

Analyzes your staged/unstaged changes and proposes conventional commit messages. Before staging, files are grouped by intent — infrastructure, agent config, app code, build tooling, docs, tests — and multi-intent changes are automatically split into separate commits with same-intent exceptions for tightly coupled changes (schema+code, signature+call-sites, code+validating tests). Messages avoid em-dash/en-dash to stay clean of AI-generated markers.

```
/commit              # Analyze and commit
/commit --amend      # Amend the last commit
```

Types: `feat` · `fix` · `refactor` · `style` · `docs` · `test` · `perf` · `chore` · `hotfix`

### `/pr` — Create a Pull Request

Creates a PR with a structured template. Automatically detects Individual or Release mode. Titles and bodies avoid em-dash/en-dash to stay clean of AI-generated markers.

```
/pr              # Individual PR (Feat, Fix, Refactor, etc.)
/pr -g           # Include Mermaid change-flow graph
/pr release      # Release PR (dev → main integration)
```

After creating an individual PR, the skill offers a one-line opt-in to generate an `/explain-diff` explainer for the change (default No, never re-asked).

### `/issue` — Create a GitHub Issue

Creates a structured issue with the appropriate template and auto-assigns type/priority labels.

```
/issue             # General issue (type inferred from context)
/issue bug         # Bug report
/issue feature     # Feature request
```

### `/review-reply` — Review & Reply to PR Comments

Collects review comments (CodeRabbit, Copilot, teammates, etc.) from a PR, analyzes their validity against the actual code, and discusses findings with you. Each item quotes the source verbatim and shows the proposed change as a diff, so you can accept or reject a suggestion without opening the files yourself — including dismissals, which quote the code that refutes the reviewer.

```
/review-reply          # Review current branch's PR
/review-reply 42       # Review PR #42
```

### `/code-review` — Context-Aware Code Review

Analyzes PR or local code changes using a multi-agent pipeline with domain-specific agents (Security, Performance, Architecture, Domain Logic). In PR mode, agents receive the PR's purpose as their primary review lens — catching incomplete implementations, consistency gaps, and purpose-misaligned changes. Cross-validates findings to filter false positives, with an out-of-diff causality filter that dismisses pre-existing issues unrelated to the PR. A **Reporting Bar** then separates "true" from "worth reporting": every surviving finding gets a disposition independent of its severity — `ship-blocker` and `in-scope-gap` are reported in full, pre-existing issues get one line under `밖으로 미룸` and never an inline comment, and style preferences are counted rather than written up. The cut appears as a single `기각 {n}건` line, so a short review reads as a judgment rather than a gap. Produces severity-based output (🔴 Critical · 🟡 Warning · 🟢 Info). Every finding quotes the offending code verbatim and shows the fix as a diff, so a report is readable on its own — no jumping to the file to work out what was wrong. When no changes are detected, automatically transitions to reviewing the current working directory. Conversation context is used to determine the optimal review scope.

```
/code-review           # Auto-detect PR, review changes, or review cwd (context-aware)
/code-review 42        # Review PR #42
/code-review src/auth/ # Review specific path
```

In PR mode, findings are published as inline review comments on specific diff lines via the GitHub Review API.

<details>
<summary>Flags</summary>

- `--wd` — Force working directory mode, even on a PR branch
- `--domain security,perf` — Override auto-detected domains
- `-y` / `-f` — Publish without approval
- `-g` — Generate Mermaid change-flow graph (PR mode only)
- `-q` / `--quick` — Quick mode: single-pass, max 2 domains, Critical/Warning first
- `--full-scan` — Include pre-existing out-of-diff issues in General Findings (PR mode only)
- `-a` / `--all` — Bypass the Reporting Bar and report every confirmed finding, dropped ones included (audit mode)
- `--no-codex` — Disable Codex integration
- `--codex-both` — Run both Codex general review and adversarial review

</details>

<details>
<summary>Codex integration (optional)</summary>

When the [Codex plugin](https://github.com/openai/codex) is installed (`claude plugin add codex` + `!codex setup`), `/code-review` automatically runs Codex adversarial review in parallel with domain agents. Findings are cross-validated and merged with source tags. Use `--no-codex` to opt out.

</details>

### `/handoff` — Session Handoff Prompt

Generates a copy-ready handoff prompt that transfers work context to a new session. Built on a single principle: **write only what the next session cannot recover by reading the repo.** The next session is a capable agent that will explore the code itself, so the handoff carries only decisions and their reasons, abandoned approaches, user constraints, and the first action. Everything else is referenced by path.

```
/handoff                   # Auto-detect and generate handoff prompt
/handoff -y                # Skip confirmation, output directly
/handoff auth refactoring  # Focus on a specific topic
```

<details>
<summary>Details</summary>

**An order, not a briefing:** The output is a prompt you paste to put the next agent to work immediately, written in your own conversational voice — not a status report you then have to act on by writing a second message. Every handoff ends in an action; even an undecided direction becomes "look into X first and tell me how you'd go."

**Structure:** A directive (what to do first, on which branch), up to 4 context bullets, and reference paths. 15 lines total, hard cap. Sections with nothing to say are omitted — an empty context section is a correct outcome.

**Git:** Branch name and whether uncommitted changes exist. Nothing more — no commit log, no diff stat, no file list. The next session runs `git status` itself.

**No skill names:** The handoff never suggests slash commands or plugin skills unless you explicitly ask for one. The next session may run in a different agent with different plugins installed, so it picks its own tools.

**Output:** Terminal text optimized for `/copy`. Never repeats file content — references paths instead.

</details>

### `/explain-diff` — Understand a Diff

Generates a self-contained interactive HTML explainer for a diff, commit, branch, or PR — so you genuinely understand the change before sharing or merging it. Inspired by Geoffrey Litt's explain-diff: understanding, not correctness, is the bottleneck of agent-written code. The document walks through the change as a literate diff (prose + verbatim code excerpts), shows an annotated directory tree of everything touched, and ends with a 5-question quiz. Wrong answers get a hint pointing back to the relevant section — never the answer — and the hint is a one-click jump to that section, with a "back to quiz" chip to return. The completion gate opens only at 5/5.

```
/explain-diff          # Explain the working diff
/explain-diff 42       # Explain PR #42
/explain-diff src/     # Restrict the diff to a path
```

<details>
<summary>Details</summary>

**Investigate first:** Before writing, the skill reads every hunk plus its enclosing function, extracts stated intent from commits/PR body, explores callers and tests, and groups hunks into narrative themes — the document explains behavior, not lines.

**Three-part structure:** Overview (background + directory tree + optional interactive figure), Changes (one card per theme, each closing with a key-takeaway card), and Comprehension check (risk pointers + quiz).

**Honest by design:** Suspicions are framed as attention pointers, never verdicts; quoted code is verbatim only; the completion button states understanding — it never pretends to perform an action.

**Output:** A single self-contained HTML file (no CDN, no webfonts) written to the repository root as `explain-diff-<slug>.html`. The absolute path is reported; nothing is auto-opened or committed.

</details>

### `/micro-world` — Inhabit a Behavior

Builds a throwaway interactive HTML simulation of a specific behavior — a state machine, algorithm, data transform, or protocol flow — so you can feel how the code works by manipulating it (after Seymour Papert's "Mathland"). Sibling of `/explain-diff`: that one produces a consistent document; this one produces a bespoke one-off simulation sharing only the visual language. An applicability gate declines honestly when a sim adds nothing (config changes, renames, dependency bumps) and points you to `/explain-diff` instead. Every simulation models the actual code logic, and any simplification is disclosed in a footnote. Each world runs one guided scenario — a mission bar with goal chips, exactly one recommended action enabled at a time — with full step playback: snapshot-based back/forward controls where pressing the forward key from a fresh page walks the entire story (arrow keys and R to restart).

```
/micro-world                          # Simulate a behavior from the current change
/micro-world "retry logic in api.ts"  # Simulate a described behavior
```

## Language Behavior

All commands write output (commit messages, PR titles/body, issue titles/body) in the language configured in your project's `AGENTS.md` (or `CLAUDE.md` as fallback). If no language is set, the user's conversational language is used. Technical terms are kept in their original form.

## Conventions at a Glance

| Item | Format | Example |
|------|--------|---------|
| Commit | `{type}: {desc}` (lowercase) | `feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Branch | `{type}/{kebab-case}` (English) | `feat/multiturn-context-persistence` |
| PR Title | `{Type}: {desc}` (capitalized) | `Feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Issue Title | `{desc}` (no type prefix) | `E2E 실험에서 non-default retrieval index snapshot 지원` |
| Release PR | `Release: dev → main 통합 (vX.Y.Z)` | `Release: dev → main 통합 (v0.4.1)` |

## Integration with CLAUDE.md

Add the following to your global `~/.claude/CLAUDE.md` to reference these conventions:

```markdown
## Git Conventions
- Commit: `{type}: {description}` (lowercase prefix: feat, fix, refactor, style, docs, test, perf, chore, hotfix)
- Branch: `{type}/{english-kebab-case}` (feat/, fix/, refactor/, docs/, hotfix/)
- PR title: `{Type}: {description}` (capitalized prefix: Feat, Fix, Refactor, Perf, etc.)
- Release PR: `Release: dev → main 통합 (vX.Y.Z)`
- Use `/commit`, `/pr`, `/pr release`, `/issue`, `/review-reply`, `/code-review`, `/handoff`, `/explain-diff`, `/micro-world` commands for full workflows
```

## Label System

`/pr` and `/issue` share a unified label scheme. Labels are auto-created on first use — if a label already exists, it is left unchanged.

<details>
<summary>Label reference</summary>

| Label | Color | Source |
|-------|-------|--------|
| `bug` | 🔴 `d73a4a` | GitHub default |
| `feature` | 🔵 `0075ca` | GitHub default |
| `enhancement` | 🩵 `a2eeef` | GitHub default |
| `docs` | 🟣 `5319e7` | GitHub default |
| `chore` | 🟡 `e4e669` | Standard |
| `refactor` | 🟪 `d4c5f9` | Standard |
| `style` | 🩵 `c5def5` | Standard |
| `test` | 🟢 `bfd4f2` | Standard |
| `perf` | 🟠 `f9d0c4` | Standard |
| `hotfix` | 🔴 `b60205` | Standard |
| `release` | 🔵 `1d76db` | Standard |
| `critical` | 🔴 `b60205` | Priority |
| `high` | 🟠 `d93f0b` | Priority |
| `medium` | 🟡 `fbca04` | Priority |
| `low` | 🟢 `0e8a16` | Priority |

- **No namespace prefix** — clean, simple names (`bug` instead of `type: bug`)
- **GitHub-standard colors** — matches GitHub's default palette where applicable
- **Conflict-safe** — `gh label create ... 2>/dev/null || true` (create if missing, skip if exists)

</details>

## License

MIT
