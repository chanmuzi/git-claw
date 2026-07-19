---
name: code-review
description: >-
  Review code changes using context-aware multi-agent pipeline with severity-based findings.
  TRIGGER when: user asks to review code, analyze PR quality, check for issues, run code review, or audit changes (e.g., "코드 리뷰해줘", "review this PR", "코드 분석해줘", "리뷰 돌려줘").
  DO NOT TRIGGER when: user is replying to review comments (use review-reply), creating PRs, committing, or performing git operations without review intent.
argument-hint: "[PR번호|경로] [-d|--domain security,perf] [-y|--yes] [-g|--graph] [-s|--sub] [-q|--quick] [-a|--all] [--wd] [--full-scan] [--no-codex|--codex|--codex-general|--codex-both]"
version: "1.9.0"
allowed-tools: Bash(git *), Bash(gh *), Bash(node *), Bash(find *), Read, Grep, Glob, Agent, TeamCreate, TaskCreate, TaskList, TaskUpdate, TaskGet, SendMessage
---

## Parse Arguments

Parse `$ARGUMENTS` to determine the review mode and flags.

### Mode Detection

| Priority | Mode | Condition | Description |
|----------|------|-----------|-------------|
| 1 | **PR Review (explicit)** | `$ARGUMENTS` contains a standalone numeric token (e.g., `42`) or `--pr <number>` | Review a specific PR's diff + commit context |
| 2 | **Working Dir (forced)** | `--wd` flag is set | Force working directory review, even on a PR branch |
| 3 | **Path Review** | `$ARGUMENTS` contains a file/directory path | Review code at the specified path |
| 4 | **PR Review (auto)** | `$ARGUMENTS` is empty → auto-detect (see below) | Auto-detected PR on current branch |
| 5 | **Working Dir** | `$ARGUMENTS` is empty and no PR detected | Review current working directory changes (staged + unstaged). If no changes, falls through to Path Review (cwd) |

### PR Auto-Detection

When `$ARGUMENTS` is empty and `--wd` is not set, attempt to detect an open PR on the current branch:

```
gh pr list --head "$(git branch --show-current)" --author "@me" --state open --json number,title --jq '.[0]'
```

Key behaviors:
- `--author "@me"` ensures only the current user's PRs are matched, skipping bot PRs (dependabot, renovate, etc.)
- If a PR is found → enter **PR Review (auto)** mode with the detected PR number
- If no PR is found (empty result, detached HEAD, or `main`/`master` branch) → fall through to **Working Dir** mode

### Context-Aware Scope Adjustment

After resolving the initial mode, review the conversation history to assess the user's actual intent. This applies ONLY when the resolved mode is **Working Dir** (Priority 5) — i.e., no positional arguments and no PR detected. Explicit flags like `--wd` (Priority 2) and PR modes (Priority 1, 4) are not subject to this adjustment.

This adjustment operates before Step 1 (Context Builder) and complements the no-changes fallback in Working Dir Mode: Context-Aware decides based on conversation signals pre-diff; the no-changes fallback decides based on actual diff results in Step 1.

Evaluate whether the user wants a **diff-focused review** (changes only) or a **broader directory review**. Consider:

- **Broaden to Path Review (cwd)**: User's conversation implies analysis, diagnosis, exploration, or state assessment of existing code — not just reviewing recent changes. Examples: "분석해줘", "현재 상태 봐줘", "진단해줘", "코드 살펴봐", "조사해줘", codebase onboarding context
- **Keep Working Dir**: User is actively developing and wants feedback on their changes. Examples: "수정한 거 봐줘", "변경사항 리뷰", iterating on feature branch with substantial diff

If broadening, display:

> **💡 Scope adjusted** — 대화 맥락에 따라 현재 디렉토리 코드 리뷰로 전환합니다

When both changes and broader analysis are relevant, use Path Review mode with the diff included as supplementary context for the domain agents.

### Flag Detection

| Flag | Default | Description |
|------|---------|-------------|
| `-y\|--yes\|--force\|-f` | off | Publish without approval |
| `-g\|--graph` | off | Generate a visual change-flow graph for PR review (Mermaid on GitHub, summary in terminal preview). Ignored in Working Dir / Path modes. |
| `-d\|--domain` | auto | Override domain selection (e.g., `-d security,perf`) |
| `-s\|--sub` | off | Use sub-agents instead of team agents for domain analysis |
| `--pr` | — | Explicit PR mode (e.g., `--pr 42`). Use when path arguments contain digits |
| `--wd` | off | Force Working Dir mode, skipping PR auto-detection |
| `-q\|--quick` | off | Quick mode: single-pass analysis without agent spawn. Auto-detected domains capped at 2, Codex disabled, Info findings omitted, graph skipped. Designed for fast iteration during development/testing. |
| `--no-codex` | off | Disable Codex integration entirely (skip Codex detection and execution) |
| `--codex` | off | Force enable Codex with default behavior (adversarial only). Use when auto-detection is unreliable |
| `--codex-general` | off | Use Codex general review (`codex:review`) only, without adversarial review |
| `--codex-both` | off | Run both Codex general review and adversarial review in parallel. Shorthand for `--codex --codex-general` |
| `--full-scan` | off | Include pre-existing issues (unrelated to PR changes) in General Findings. By default, out-of-diff findings without causal relationship to the PR are dismissed. PR mode only; ignored in Working Dir / Path mode. |
| `-a\|--all` | off | Bypass the Reporting Bar (Step 4.5). Report every Confirmed finding, including `out-of-scope` and `drop` dispositions. Use when auditing rather than reviewing a change. |

Codex flag precedence: `--no-codex` > `--codex-both` > `--codex` + `--codex-general` (combo = **both**) > `--codex-general` alone > `--codex` alone > default.
When `--codex` and `--codex-general` are both present, the combined effect is **both** mode (equivalent to `--codex-both`).

If `$ARGUMENTS` contains explicit publish intent ("comment 달아", "바로 올려", "게시해", "post it"), treat as `-y`.

### Quick Mode Implicit Effects

When `--quick` is set, the following flags are implicitly forced regardless of other arguments:

| Implicit Override | Effect |
|-------------------|--------|
| `--no-codex` | Codex always disabled |
| `-g` / `--graph` ignored | Graph generation skipped |

Display hint immediately after flag parsing:

> **⚡ Quick mode** — 단일 패스 분석, Critical/Warning 우선 (없으면 Info fallback)

---

## Step 1: Context Builder

Gather all available context based on the review mode.

### PR Review Mode

```
gh pr view {number} --json title,body,baseRefName,headRefName,commits,files,labels --jq '.'
```

```
gh pr diff {number}
```

```
git log --oneline {baseRefName}..{headRefName} 2>/dev/null | head -30
```

Read the PR description to understand the author's intent. This is critical for cross-validation.

Extract a 1-2 sentence **PR Purpose** summary from the title, description, and commit messages. If the PR description is empty or minimal, derive the purpose from the title and commit messages alone. This summary is injected into the domain agent prompts via `## PR Context` in Step 3's Common Prompt Suffix.

### Working Dir Mode

```
git diff HEAD --stat
```

```
git diff HEAD
```

```
git log --oneline -5
```

If there are no changes (no staged or unstaged diffs), transition to **Path Review mode** targeting the current working directory. Display hint:

> **💡 변경사항 없음** — 현재 디렉토리 코드 리뷰로 전환합니다

Then proceed with Path Review Mode logic below.

Note: In **diff review mode**, `git diff HEAD` only includes tracked files. Untracked (new) files are not included — stage them first (`git add`) to include. In the **Path Review fallback** (no changes detected → cwd review), files are read directly from the filesystem, so untracked files are typically included.

### Path Review Mode

Use Glob to list files under the specified path. Read each file's content.

```
git log --oneline -10 -- {path}
```

### Common (all modes)

For each changed file, read sufficient surrounding context (at least 30 lines around each change hunk) to understand the code's purpose. Use Read to examine files directly.

If `-g` (graph) flag is set **and the mode is PR Review**, build a **change-flow graph** for the GitHub review output. In Working Dir / Path mode, `-g` is ignored — the terminal cannot render Mermaid and there is no GitHub publishing target.

1. **Trace relationships**: For each changed file, identify imports, exports, function calls, and data flow to/from other changed files using Grep.
2. **Map edges**: Record directional relationships — code-level (`imports`, `calls`, `extends`, `emits/consumes`) and conceptual (`references`, `shared logic`, `configures`).
3. **Group by module**: If changed files exceed 7, group by parent directory into logical modules (subgraph nodes). Individual files become child nodes.
4. **Construct Mermaid source**: Build a `flowchart LR` diagram. Nodes = changed files (or module groups). Edges = relationships found in step 2, labeled with relationship type.

**Skip condition**: If no edges (relationships) are found between changed files after step 2, skip graph generation entirely. In the terminal preview, show: `📊 Change Graph: 파일 간 관계가 감지되지 않아 그래프를 생략합니다`. In the GitHub format, omit the graph section silently.

The graph is rendered differently per output format (see Step 5):
- **GitHub**: Full Mermaid code block (native rendering).
- **Terminal (PR preview only)**: One-line summary with module/file counts and primary flow path.

---

## Step 2: Domain Router

Determine which domain agents to activate based on changed files.

### Auto-Activation Rules

| File Pattern | Activated Domains |
|-------------|-------------------|
| `.tsx`, `.jsx`, `.css`, `.html`, `.vue`, `.svelte` | Architecture, Domain Logic |
| `.sql`, `.prisma`, `*migration*` | Performance, Security |
| `auth/`, `middleware/`, `security/`, `*token*` | Security |
| `*.test.*`, `*.spec.*` | Domain Logic |
| `Dockerfile`, `k8s/`, `terraform/`, `*.yml` (CI) | Architecture, Security |
| `package.json`, `requirements.txt`, `go.mod` | Security |
| General source code (`.ts`, `.py`, `.go`, `.java`, `.rs`, etc.) | Architecture, Domain Logic |

Default (no pattern match): Architecture + Domain Logic.

### Override

If `--domain` flag is provided, use ONLY the specified domains regardless of file patterns.

Collect the union of all activated domains from all changed files. Deduplicate.

### Quick Mode Domain Cap

When `--quick` is set (and `--domain` is NOT provided), cap the auto-detected domains to **at most 2** using this priority:

```
Security > Domain Logic > Architecture > Performance
```

Process: run auto-detection as normal, then sort by priority and take the top 2. If auto-detection returns ≤ 2, use all of them as-is.

When `--domain` is explicitly provided alongside `--quick`, respect the override — do not cap.

---

## Step 2.5: Codex Detection

Determine whether the Codex plugin is available and resolve the Codex execution mode.

**Quick mode**: Skip this entire step. Codex mode is already forced to **disabled** (see Quick Mode Implicit Effects). Proceed directly to Step 3.

### Companion Detection

**REQUIRED: Run this command before proceeding to Mode Resolution.**

```bash
COMPANION=$(find ~/.claude/plugins/cache/openai-codex -name "codex-companion.mjs" -print 2>/dev/null | sort | tail -1)
```

- `COMPANION` is non-empty → Codex **available**
- `COMPANION` is empty → Codex **unavailable**

**Do not proceed to Mode Resolution until this command has been executed and the result evaluated.**

### Mode Resolution

| Priority | Condition | Codex Mode | Hint |
|----------|-----------|-----------|------|
| 1 | `--no-codex` flag is set | **disabled** | None |
| 2 | Codex unavailable + any Codex flag set (`--codex`, `--codex-general`, `--codex-both`) | **disabled** | `⚠️ Codex unavailable` — {flag} 요청했으나 companion을 찾을 수 없습니다 |
| 3 | Codex unavailable + no Codex flag | **disabled** | None |
| 4 | `--codex-both` flag is set, OR `--codex` + `--codex-general` both present | **both** | `💡 Codex detected` — review + adversarial 병렬 실행 |
| 5 | `--codex-general` flag is set (without `--codex`) | **review** | `💡 Codex detected` — general review 실행 |
| 6 | `--codex` flag is set (without `--codex-general`) | **adversarial** | `💡 Codex detected` — adversarial review (강제 활성화) |
| 7 | Default (no Codex flag) | **adversarial** | `💡 Codex detected` — adversarial review 실행 |

Companion subcommands per mode:
- **adversarial**: `adversarial-review --wait`
- **review**: `review --wait`
- **both**: `review --wait` + `adversarial-review --wait`

In PR mode, append `--base {baseRefName}` to each subcommand to scope the review to the PR diff.

Display the resolved Hint (if any) immediately after detection, before launching domain agents. Hints follow the project's Hint 패턴 (`> **{icon} {action}** — {reason}`).

Store the resolved mode for use in Steps 3, 4, and 5. Each spawned Codex agent re-resolves the companion path independently via `find` since agents run in separate contexts. If mode is **disabled**, skip all Codex-related logic in subsequent steps and proceed exactly as before (full backward compatibility).

---

## Step 3: Domain Agents (Parallel Execution)

Launch activated domain agents in parallel. Each agent receives the full diff and context from Step 1.

### Quick Mode: Single-Pass Analysis

When `--quick` is set, skip all agent spawning (team and sub-agent). Instead, perform the analysis directly in the main context:

1. For each activated domain (max 2 from Step 2), analyze the diff using that domain's Investigation Protocol (see Domain Definitions below).
2. Produce findings in the same format as agent results — the **Output Format in the Common Prompt Suffix below applies in full**, even though no agent is spawned: title, file path, primary line number, occurrence count, `current_code` (verbatim), `proposed_code` (when a concrete fix exists), description, and action line. Evidence is mandatory in Quick mode too; read the file to quote the offending lines.
3. Produce findings at whatever severity the evidence supports, then apply the Reporting Bar (Step 4.5) exactly as in Normal mode. Do not lower the bar to fill a severity level — a Quick review that reports one `ship-blocker` and nothing else is a complete result, not a thin one.
4. Skip Codex execution entirely (mode is **disabled**).

This eliminates the overhead of agent creation, context window allocation, and inter-agent communication. Proceed directly to Step 4 after analysis.

### Execution Mode (Normal)

| Mode | Condition | Description |
|------|-----------|-------------|
| **Team agents** (default) | `-s`/`--sub` flag NOT set | Each domain runs as an independent team agent with its own context window. Better result quality for large diffs. |
| **Sub-agents** | `-s`/`--sub` flag set | Each domain runs as a sub-agent via the Agent tool. Results return to the main context. Lighter for small diffs. |

### Team Agent Mode (default)

Used when `-s`/`--sub` flag is NOT set. Each domain agent gets its own context window.

1. `TeamCreate` — name: `code-review`.
2. For each activated domain:
   - `TaskCreate` — subject: `"{Domain} domain analysis"`. Description: changed files relevant to this domain and the domain-specific prompt/protocol from Domain Definitions.
   - Spawn teammate via `Agent` with `team_name: "code-review"`, `name` set to domain name (lowercase, e.g., `"security"`, `"architecture"`), `model: "opus"`. Prompt: domain-specific prompt from Domain Definitions (including Common Prompt Suffix), full diff from Step 1, changed file context, and the finding format defined by the Common Prompt Suffix Output Format (which is the single source of truth for finding fields — do not restate a shorter field list here, it drifts).
   - `TaskUpdate` — set `owner` to the agent name.
3. Monitor `TaskList` until all domain tasks complete. Collect findings from agent messages.
4. Shut down agents via `SendMessage` with `shutdown_request`.
5. Pass aggregated findings to Step 4.

Fallback: if `TeamCreate` is unavailable, switch to Sub-agent mode.

### Sub-agent Mode

Used when `-s`/`--sub` flag IS set, or as fallback. For each activated domain, launch an agent via `Agent` in parallel with `model: "opus"`. Prompt: domain-specific prompt from Domain Definitions (including Common Prompt Suffix), full diff from Step 1, changed file context, and the finding format defined by the Common Prompt Suffix Output Format. Results return to the main context.

### Domain Definitions

Each domain agent — whether team or sub — receives its domain-specific prompt via the Agent tool's `prompt` parameter. All agents are spawned with `model: "opus"` and **without `subagent_type`** (general-purpose). Domain specialization is handled entirely through the prompt — do not delegate to external agent types. The prompt for each domain consists of: the domain-specific prompt below, followed by the Common Prompt Suffix.

#### Common Prompt Suffix

Append the following sections to every domain agent prompt.

**When `-a` / `--all` is set, omit the two bullets marked `[bar-only]` from the Constraints section below.** They shape *reporting*, not *collection* — the 3-finding cap and the nameable-cost requirement both discard candidates before Step 4.5 ever runs, so leaving them in would make `--all` unable to restore what it promises to restore. Everything else in the suffix, evidence requirements included, stays exactly as written: `--all` widens what survives, it never lowers the evidence bar.

```
## PR Context (PR Review Mode only — omit this section in Working Dir / Path Review mode)

**PR Purpose**: {1-2 sentence summary of the PR's goal, extracted from the PR title, description, and commit messages gathered in Step 1}

When reviewing code, use this PR purpose as your primary lens:
- Check whether all changes are consistent with the stated purpose
- Look for incomplete implementations: if the PR aims to do X, are there
  places where X is only partially done? (e.g., i18n PR that leaves hardcoded
  strings in some files)
- Identify files in the diff that were changed but not fully aligned with the
  PR's goal
- For data files (YAML, JSON, config), verify cross-language/cross-environment
  consistency when the PR's purpose involves such concerns

## Constraints
- You are read-only. Do not attempt to modify any files.
- Write findings in the language configured in the project's AGENTS.md (or CLAUDE.md as fallback). If no language is configured, follow the user's conversational language.
- If no issues are found, return an empty findings list (no items) and state "No issues found." Do not manufacture findings.
- [bar-only] **Report at most 3 findings.** If you have more than 3, you have not prioritized — rank them by the concrete cost of shipping the code as-is, and return only the top 3. An empty list is a legitimate and common result for a well-written change; returning fewer findings is never penalized, and padding the list with style observations to look thorough actively degrades the review.
- [bar-only] For each finding, state the cost of merging it as-is in one sentence. If you cannot, do not report it — an observation with no nameable consequence is not a finding.

## Severity Criteria
- Critical: Security vulnerability, data loss risk, or crash-inducing bug
- Warning: Potential bug, performance issue, or maintainability concern
- Info: Suggestion, minor improvement, or style note

## Output Format
Return findings as a structured list. Each finding must contain:
- title: Short descriptive name (do not repeat the file name)
- file_path: Exact file path
- primary_line: Line number in the NEW version of the file where the issue is most visible; this is the canonical line-reference field for evidence requirements. **Omit it entirely** when the new version has no such line — the file is deleted by this change, is a binary/generated artifact, or does not exist yet. Never emit `null` and never invent a line number.
- location_marker: Required **only** when `primary_line` is omitted. One of:
  - `deleted` — the file is deleted by this change. Evidence quotes the **removed lines** (`-`-only, from the diff).
  - `binary` — a binary/generated artifact with no readable hunk. Evidence quotes the diff's `Binary files ... differ` line.
  - `missing` — the file **genuinely does not exist**. Evidence quotes the anchor per the absence rule.
  - `file-level` — **the file exists, but the finding has no single line** (it concerns the file as a whole, or the source gave no line). Evidence quotes the part of the file the finding is actually about, as a plain block. Do NOT use `missing` for this: claiming a file is absent when it is present makes the evidence itself false, and the removed-lines rule would demand lines that do not exist.

  Such findings are NOT dropped for lack of a line — a validation or auth check deleted by this change is exactly the kind of finding that must survive.
- occurrence_count: Number of instances of this pattern in the diff
- current_code: The offending source lines, **copied verbatim** from the file — never paraphrased, summarized, or reconstructed from memory. Read the file to obtain them. Include only enough surrounding context to make the lines intelligible (12 lines max; elide the middle with `…` if longer).
- proposed_code: The replacement lines, whenever a concrete localized fix exists — **at any severity, Info included**. Whether you produce this field is decided by whether a replacement exists, NOT by the severity. Omit only when there is nothing concrete to replace (the fix is directional/structural, or the finding needs no change at all).
- description: Why the current code is a problem. If it conflicts with a rule, convention, or another file, quote that source verbatim too (file path + the conflicting lines) — a claim of inconsistency is not verifiable without both sides shown. Reference locations by function name or code pattern; do not repeat line numbers here, since `primary_line` provides the line-based evidence.
- action: For Critical/Warning — suggested fix. For Info — one of: Accept (intentional, no action needed) / Monitor (could become an issue at scale), with reason. If neither fits because the finding is not worth acting on at all, do not report the finding.

**Evidence is mandatory.** A finding without `current_code` is not reportable. If you cannot quote the exact lines you are objecting to, you have not verified the issue exists — drop it rather than reporting it from inference.

**When the problem is absent code** — a missing check, a missing entry, a file never updated to match a new pattern — there are no offending lines to quote. Do NOT drop these findings; quote the **anchor** instead:
- the existing lines the missing code should sit next to (the insertion point), or
- the sibling that already does it right, which this location fails to match (quote the sibling with its own path).
Set `current_code` to that anchor and `proposed_code` to the anchor with the addition included. In the output, a `+`-only diff block is the correct rendering. "Nothing to quote" is never grounds to drop an absence finding — it is grounds to quote the anchor.
```

#### 🛡️ Security

```
You are a security engineer conducting a focused security audit of code changes.
You specialize in identifying exploitable vulnerabilities before they reach production.
Prioritize findings by: severity × exploitability × blast radius.

## Investigation Protocol (follow this order)
1. Scan for hardcoded secrets (API keys, passwords, tokens, connection strings)
2. Check input validation: all user inputs sanitized? Parameterized queries?
3. Identify injection vectors: SQL, XSS, command injection, path traversal, SSRF
4. Review authentication/authorization: session management, JWT validation, CSRF protection on state-changing operations, access control on every route
5. Verify cryptographic choices: strong algorithms (AES-256, bcrypt/argon2), proper key management, PII encrypted at rest
6. Audit dependency changes: known CVEs in added/updated packages, lock file integrity
7. Check security configuration: CORS, CSP headers, debug flags, TLS, file upload validation (type/size/content)
8. Assess security logging: auth failures and access denials logged? No sensitive data in logs?

## Evidence Gate
Every finding MUST cite the exact file path, and a line number whenever one exists in the new version. If you cannot point to a specific line in the diff or surrounding context, do not report it — with two exceptions: **absent-code** findings (a missing auth check, a missing validation) satisfy this gate via the anchor line; **no-line** findings (`deleted` / `binary` / `missing` / `file-level`) satisfy it via the file path plus `location_marker` and whatever evidence that marker calls for. See the absence and no-line rules in the Output Format below. "No line to cite" is grounds to use those branches, never grounds to drop the finding — a security control deleted or never added is exactly what this gate must not silently swallow.
```

#### ⚡ Performance

```
You are a performance engineer analyzing code changes for runtime efficiency issues.
Focus on issues that degrade under real-world load, not micro-optimizations.

## Investigation Protocol (follow this order)
1. Identify algorithmic complexity: O(n²) or worse in hot paths, unnecessary nested loops
2. Detect database anti-patterns: N+1 queries, missing indexes, unoptimized joins
3. Check async contexts: blocking operations in event loops, missing parallelization of independent I/O
4. Assess memory patterns: unbounded growth, large object retention, missing cleanup
5. Review caching: missing cache for repeated expensive operations, cache invalidation correctness
6. Check resource management: connection pool sizing, file handle leaks, stream backpressure

## Evidence Gate
Every finding MUST cite the exact file path, and a line number whenever one exists in the new version (otherwise use the absence / no-line branches defined in the Output Format — never fabricate a line, never drop the finding for lack of one). Only report issues with measurable impact under realistic load. Do not flag theoretical micro-optimizations.
```

#### 🏗️ Architecture

```
You are a staff engineer reviewing code changes for long-term maintainability and design coherence.
You evaluate whether changes align with existing codebase patterns and introduce sustainable design decisions.

## Investigation Protocol (follow this order)
1. Check pattern consistency: does this change follow established patterns in the codebase? Use Grep to find similar implementations and compare approaches.
2. Evaluate SOLID principles: SRP (single reason to change?), DIP (depends on abstractions?), OCP (extensible without modification?)
3. Assess coupling: are new dependencies appropriate? Is the dependency direction correct?
4. Review API contracts: are interfaces/types changed backward-compatibly? Are contracts clear?
5. Check module boundaries: does the change respect existing boundaries? Is responsibility placed correctly?
6. Assess technical debt: does this change introduce shortcuts that will compound?

## Evidence Gate
Every finding MUST cite the exact file path, and a line number whenever one exists in the new version (otherwise use the absence / no-line branches defined in the Output Format — never fabricate a line, never drop the finding for lack of one). Reference the existing pattern or file that the change should align with. When recommending a change, state the trade-off (what is gained vs. what is sacrificed). Do not report subjective style preferences.
```

#### 🔍 Domain Logic

```
You are a senior engineer who owns this codebase's business domain, reviewing changes for correctness.
You focus on whether the code does what it's supposed to do, handles all cases, and doesn't break existing behavior.

## Investigation Protocol (follow this order)
0. Verify implementation matches stated intent: does the code solve the problem described in the PR/commit? Anything missing? Anything extra that wasn't requested?
1. Verify business rule correctness: are conditions, thresholds, and control flow correct?
2. Check error handling completeness: all error paths handled? Errors propagate correctly? Resource cleanup?
3. Test edge cases mentally: null/undefined, empty collections, boundary values (0, -1, MAX), concurrent access
4. Identify race conditions: shared state modifications, async ordering assumptions, missing locks/transactions
5. Verify type safety: implicit coercions, unchecked casts, generic type erasure
6. Check state management: initialization order, invalid state transitions, stale data

## Scope
Your scope is correctness and behavioral soundness. Do not flag style, pattern consistency, or maintainability concerns — the Architecture domain covers those.

## Evidence Gate
Every finding MUST cite the exact file path, and a line number whenever one exists in the new version (otherwise use the absence / no-line branches defined in the Output Format — never fabricate a line, never drop the finding for lack of one). Describe the specific input or scenario that triggers the bug. Do not report hypothetical issues without a concrete trigger.
```

Each finding must include: title, file path, **primary line number**, occurrence count, **verbatim current code**, description, and action line — `suggested fix` for Critical/Warning severity, `recommendation label` (Accept / Monitor) for Info severity. Any finding with a concrete localized fix must also include the proposed replacement lines, **regardless of severity** — severity decides the action line, not whether a replacement is produced.

**Primary line number**: The line number in the new version of the file where the issue is most clearly visible. Used for inline comment targeting in PR mode and for the location header above the evidence block. In the finding's description text, reference locations by section heading, function name, or code pattern — not by raw line number. When the new version has no such line (deleted file, binary artifact, file not yet created, or a finding that is about the file as a whole), omit the field and set `location_marker` instead — never fabricate a line, and never drop the finding for lack of one.

**Verbatim current code**: The reader must be able to see what the code actually says without opening the file. Quote it exactly as written; do not reconstruct it. This is also the reviewer's own guard against false positives — a finding whose quoted lines do not actually say what the description claims is a false positive, and quoting exposes it before it reaches the user.

### Codex Parallel Execution

If Codex mode (from Step 2.5) is NOT **disabled**, launch Codex alongside the domain agents in parallel. Codex runs as an independent second reviewer — it is NOT a domain agent, but its execution is concurrent with domain agents.

Codex is invoked via the companion runtime directly through Bash, NOT via `Skill("codex:...")`. The `codex:adversarial-review` and `codex:review` skills have `disable-model-invocation: true`, which blocks `Skill()` invocation. The companion runtime (`codex-companion.mjs`) has no such restriction and is the official programmatic interface for invoking Codex from Claude Code.

#### Team Agent Mode (default, `-s`/`--sub` NOT set)

For each Codex subcommand to invoke (per the mode table in Step 2.5):

1. `TaskCreate` — subject: `"Codex {mode} review"` (e.g., `"Codex adversarial review"`).
2. Spawn teammate via `Agent` with:
   - `team_name: "code-review"`
   - `name: "codex"` (or `"codex-general"` / `"codex-adversarial"` when mode is **both**, to distinguish the two)
   - Prompt: Instruct the agent to run the companion via Bash and return the findings. The Bash command:
     ```
     COMPANION=$(find ~/.claude/plugins/cache/openai-codex -name "codex-companion.mjs" -print 2>/dev/null | sort | tail -1) && node "$COMPANION" {subcommand} --wait
     ```
     Where `{subcommand}` is `adversarial-review` or `review` per the mode table. Use `--base {baseRefName}` for PR mode.
3. `TaskUpdate` — set `owner` to the agent name.

The Codex agent(s) run in parallel with domain agents (Security, Architecture, etc.) within the same `code-review` team. They appear in `TaskList` output alongside domain agents, satisfying traceability (Acceptance Criterion F).

#### Sub-agent Mode (`-s`/`--sub` set)

For each Codex subcommand to invoke:

- Launch via `Agent` with `name: "codex"` (or `"codex-general"` / `"codex-adversarial"` for **both** mode). No `team_name`. The agent runs the companion via Bash (same command as Team Agent Mode) and returns findings.

Launch Codex agents at the same time as domain agents — do NOT wait for domain agents to finish first.

### Codex Failure Handling

If a Codex agent reports a non-zero exit code or returns an error (e.g., quota exhausted, authentication failure, network error):

1. **Do NOT retry** — treat the Codex contribution as unavailable for this run.
2. **Findings = empty** — proceed with domain agent findings only. Do not attempt to parse error output as findings.
3. **Terminal notice** — classify the error and display the appropriate message:

| Error Signal | Terminal Notice |
|-------------|----------------|
| stderr contains `auth`, `login`, `API key`, `unauthorized`, `401` | `⚠️ Codex auth required` — `!codex setup` 실행 권장 |
| Any other non-zero exit | `ℹ️ Codex: unavailable (skipped)` |

4. **GitHub format** — do NOT include any Codex failure notice. Codex availability is an internal infrastructure detail, not relevant to PR reviewers.

### Fallback (non-Claude Code runners)

If neither team agents nor sub-agents are available (e.g., Codex CLI, Gemini CLI as the runner platform), perform all domain analyses sequentially in a single pass. Analyze each domain's Investigation Protocol one by one and collect findings.

The **Output Format in the Common Prompt Suffix applies in full on this path too**, even though no agent is spawned and the Suffix is never injected as a prompt: every finding carries `current_code` (verbatim, read from the file), plus `proposed_code` when a concrete fix exists. Evidence is mandatory here as well — this is the path non-Claude-Code hosts actually take, so dropping the requirement here would exempt exactly the agents this skill is meant to support.

---

## Step 4: Cross-Validation

This is the quality gate. Review ALL findings from domain agents against the full context to filter false positives.

### Quick Mode: Lightweight Validation

When `--quick` is set, perform a reduced validation pass:

1. **Expanded context**: Read at least 15 lines around the flagged location using Read (half of normal). For `location_marker` findings, apply the No-Line Findings rules below instead — a failed Read is never grounds for Dismissed there either.
2. **Sanity check**: Verify the finding references real code (not a false match from diff noise).

Skip git history, comments/docs search, and PR description cross-reference. Apply Confirmed / Dismissed verdicts only (no Demoted). This trades thoroughness for speed — obvious false positives are still caught, but edge cases may pass through.

Out-of-Diff Finding Filter (see below) applies identically in Quick mode.

### Out-of-Diff Finding Filter (PR Mode Only)

For findings that reference files NOT in the PR diff, apply a causality test
before proceeding to the normal verdict process:

**Causality Test**: Does this finding have a direct causal relationship with
the PR's changes?

| Causality Type | Description | Verdict |
|---------------|-------------|---------|
| **Consistency gap** | The PR introduces or modifies a pattern, but the same pattern remains not yet updated in other files. The PR is incomplete without this change. | → Proceed to normal verdict (Confirmed/Demoted/Dismissed) |
| **Side effect** | A function/API/contract changed in the PR is called or depended on by code outside the diff, and that code will break or behave incorrectly. | → Proceed to normal verdict |
| **Pre-existing issue** | A code defect that existed before this PR and is unrelated to the PR's changes. Discovered incidentally during review. | → If `--full-scan`: proceed to normal verdict. Otherwise: **Dismissed** (not in scope for this review). |

To determine causality:
1. Identify what the PR changed (from Step 1 context).
2. For the out-of-diff finding, ask: "Would this finding exist even if this PR
   had never been created?" If yes → pre-existing issue.
3. If no → check whether it's a consistency gap or side effect, and proceed
   to normal verdict.

Findings that pass the causality test (or are included via `--full-scan`)
retain their original severity and are included via contextual mapping
(Step 6 tier 2) or General Findings (tier 3) as appropriate.

### No-Line Findings (`location_marker` set)

A finding carrying `location_marker` instead of `primary_line` has no line to Read around. For three of the four markers the file itself cannot be Read either — and that absence *is* the finding's premise, not a verification failure. A failed read must never be mistaken for "unverifiable → Dismissed". Substitute the context source per marker:

- `deleted`: verify against the diff's LEFT side (`gh pr diff {number}`). The removed lines must actually say what the finding claims. Then check whether the removed logic reappears elsewhere in the diff — an auth check *moved* to middleware is **Dismissed**; one simply *removed* is **Confirmed**.
- `missing`: verify the anchor / sibling quoted as evidence, not the absent file.
- `binary`: verify the diff's `Binary files ... differ` line plus whatever was claimed out-of-band.
- `file-level`: **the file exists** — Read it normally and verify the finding against the part of the file it concerns. This marker means "no single line", not "no file". Do not look for removed lines here; there are none.

"I could not Read the file" is never grounds for Dismissed on this path. A deleted validation or auth check is precisely the finding this branch exists to carry to the user.

### Normal Mode

For each finding **that has a `primary_line`**:

1. **Expanded context**: Read at least 30 lines around the flagged location using Read.
2. **Git history**: Check if the code was intentionally written this way:
   ```
   git log --oneline -5 -- {file}
   ```
3. **Comments/docs**: Search for TODO, FIXME, or design notes near the flagged code. Check if the project's AGENTS.md (or CLAUDE.md as fallback) or other documentation addresses the pattern.
4. **PR description/commit messages**: Cross-reference against the author's stated intent (from Step 1).

### Verdict per finding

| Verdict | Action |
|---------|--------|
| **Confirmed** | Real issue — include in final output |
| **Demoted** | Real but already acknowledged — downgrade to Info with context note |
| **Dismissed** | False positive — remove from output |

Log dismissed findings internally (do not output them) to avoid noise.

### Codex Findings Integration

If Codex mode is NOT **disabled**, collect findings from the Codex agent(s) after they complete. Codex findings join the cross-validation process identically to domain agent findings:

0. **Hydrate the Codex finding into the standard schema first.** What you receive depends on the subcommand — they are NOT the same artifact, and the agent sees **rendered text, not JSON** (the Bash command has no `--json`):

   - **`adversarial-review`**: the model's output is validated against the companion's finding schema, then rendered as `- [{severity}] {title} ({file}:{line_start}-{line_end})` followed by the body and `Recommendation: ...`. Parse those rendered lines.
   - **`review`**: the built-in codex reviewer's free-form stdout — **this schema does not apply**. Extract file / line / severity / rationale from the prose as best you can. A field the prose does not state is simply absent; do not invent it.

   Map to the standard schema:

   | Standard field | From Codex |
   |----------------|-----------|
   | `file_path` | the cited file |
   | `primary_line` | the line from `{file}:{n}` or `{file}:{n}-{m}`. **The line suffix may be absent** — the renderer omits it when the finding carries no line, and the companion does not enforce per-finding required fields at runtime. The free-form `review` output routinely has no line at all. When there is no line, omit `primary_line` and pick the marker **by the file's actual state, not by default**: `deleted` if the diff deletes the file; `missing` only if the file genuinely does not exist; otherwise **`file-level`** — the file is right there, the finding simply has no single line. Check before you label. Marking a present file `missing` makes the evidence a lie (it would demand removed lines that do not exist) and tells the reader the file is gone when it is not. Never fabricate a line to force an inline anchor. |
   | `title` | carry over as-is. Do not re-invent it from the body. |
   | `description` | the body |
   | `severity` | Map `critical` → Critical; `high` → Critical if it meets the local Critical criteria (security vulnerability, data loss, crash), otherwise Warning; `medium` → Warning; `low` → Info. **Do not free-reclassify from the body** — re-deriving severity silently promotes or demotes findings. Override only when cross-validation gives positive grounds, and say so. |
   | `occurrence_count` | Not provided — count the instances you can actually verify in the diff (default 1). |
   | `current_code` / `proposed_code` | **Not provided.** Read the cited file and quote the cited lines verbatim; add the replacement when the recommendation implies a concrete one. **If the file is deleted in this diff, do not try to read it** — quote the removed lines from the diff's LEFT side and set `location_marker: deleted` (see No-Line Findings in Step 4). A failed read is not a false positive. |

   The source read is itself a verification step: if the cited lines do not say what the body claims, the finding is a false positive and is dismissed in cross-validation below.

1. **Merge into unified pool**: Combine hydrated Codex findings with domain agent findings before cross-validation.
2. **Apply the same verdict process**: Each Codex finding undergoes the same Confirmed / Demoted / Dismissed evaluation (expanded context, git history, comments/docs, PR description).
3. **Tag preservation**: Preserve the Codex origin tag on each finding (see Step 5 for tag rules). The tag indicates which Codex subcommand produced the finding.

Codex findings are NOT given special treatment — they must pass the same quality gate as domain agent findings.

### Deduplication

When multiple agents (domain or Codex) flag the same code location:
- **Same root cause**: Merge into a single finding. Keep the higher severity and credit all relevant sources (e.g., `Architecture • Domain Logic`, `Architecture • Codex`). Codex tags follow Step 5 Source Tags rules (`Codex` or `Codex Adv`).
- **Different concerns**: Keep as separate findings, each under its own domain.

---

## Step 4.5: Reporting Bar

Step 4 decided **which findings are true**. This step decides **which true findings belong in this review**. They are not the same question, and collapsing them is what produces reviews where six real observations bury the two that matter.

Severity answers "how bad is this". It does not answer "is this mine to raise here". A `Warning` that predates the branch under review and a `Warning` the branch just introduced are the same severity and completely different asks of the reader. **Disposition is a second, independent axis** — assign it to every Confirmed and Demoted finding.

### Disposition

| Disposition | Test | Output |
|-------------|------|--------|
| **`ship-blocker`** | Merging this causes a concrete cost — exploit, data loss, crash, wrong result, broken caller. Name the cost. | Reported in full |
| **`in-scope-gap`** | The change under review set out to do X and did X only partly. The pattern applied to 3 of 4 call sites; the test added does not assert what its name promises; the rename skipped one file. | Reported in full |
| **`out-of-scope`** | Real, but not this change's doing — it predates the branch, or fixing it means restructuring something the change does not touch. | One line in the summary, under `밖으로 미룸`. **Never an inline comment** (unless `--full-scan` is set, which promotes these to the ordinary publishing path). |
| **`drop`** | Style preference, a defensible alternative the author chose, a risk with no trigger in today's code, or a restatement of something already reported. | Not reported. Counted only. |

Resolve the disposition by asking, in order:

1. **Would this finding exist if the change under review had never happened?** Yes → `out-of-scope` (or `drop` if also trivial). This is the same causality test the Out-of-Diff Filter applies in Step 4, now extended to findings that sit *inside* the diff — code the change touched but did not break.
2. **Does the change's own stated purpose demand this?** Yes → `in-scope-gap`. Take the purpose from the PR description and commit messages (PR mode), or from the user's request and the working diff (Working Dir / Path mode). When no purpose can be established, this disposition is unavailable — do not infer one.
3. **Name the cost of merging as-is.** If you cannot state a concrete consequence in one sentence, it is `drop`, not a lowered severity. "Could be cleaner" is not a cost.

Assign exactly one disposition per finding. When two apply, the higher row wins.

### Cutting is a deliverable

The count of what you cut is part of the review, not an omission from it. Report it as one line (see Step 5 templates):

```
기각 {n}건 — 취향 {a} · 선행 이슈 {b} · 오늘 발현 안 함 {c}
```

Never expand this into per-finding write-ups, and never re-litigate a `drop` elsewhere in the output. The line exists so the reader knows the cut was a judgment rather than a gap; that is its entire job.

### Interaction with other flags

- **`-a` / `--all`**: skip this step. Every Confirmed finding is reported at full length, `drop` included. The `기각` line is omitted (nothing was cut). This flag also relaxes the domain agents' collection limits back in Step 3 (the `[bar-only]` constraints) — a finding an agent never returned cannot be restored here, so the two must move together.
- **`--full-scan`**: promotes `out-of-scope` findings to full reporting. They take the ordinary publishing path — inline comment when the line maps, General Findings when it does not — and the `밖으로 미룸` section is omitted entirely, since nothing was deferred. `drop` still applies: `--full-scan` widens the *scope*, it does not lower the *bar*.
- **`--quick`**: the bar applies unchanged. Quick mode already reduces how many findings are produced; this step governs which of them are worth the reader's attention, which is orthogonal.

### The failure mode this step exists to prevent

Reporting everything true is not thoroughness — it transfers the triage work to the reader, who has less context than you do. A review of nine findings where two are actionable is worse than a review of two, because the reader must now re-derive the same disposition you already had the evidence to assign. **A finding you cannot defend as `ship-blocker` or `in-scope-gap` is one you should not be spending the reader's attention on.**

The inverse failure is real too: do not reach for `drop` to keep the output short. `drop` requires the same evidentiary standard as any other verdict — you are asserting the finding does not matter, and Step 4 already established that it is true.

---

## Step 5: Output Generator

Produce severity-first structured output.

### Quick Mode Output

When `--quick` is set, the output severity is determined by results:

- **Critical/Warning 1건 이상**: Critical/Warning만 출력, Info 생략. Summary: `Findings: 🔴 {n} critical · 🟡 {n} warnings`.
- **Critical/Warning 0건, Info 1건 이상**: Info findings를 fallback으로 출력 (Recommendation labels 포함). Summary: `Findings: 🟢 {n} info`.
- **전체 0건**: Terminal/GitHub 공통 zero-findings 규칙 적용 (`✅ No issues found.`).

Common rules:
- **Evidence** — same as Normal mode. Quick mode reduces the number of findings, never the evidence behind one. A finding without verbatim current code is not reportable in any mode.
- **Source Tags** — same as Normal mode, using domain names.
- **Graph** — always omitted (see Quick Mode Implicit Effects).
- **Codex tags** — not applicable (Codex disabled).
- **GitHub format**: If publishing in PR mode, use the same GitHub format with the same severity filtering rules above.

### Severity Levels

| Severity | Icon | Criteria |
|----------|------|----------|
| Critical | 🔴 | Security vulnerability, data loss risk, crash-inducing bug |
| Warning | 🟡 | Potential bug, performance issue, maintainability concern |
| Info | 🟢 | Suggestion, minor improvement, style note |

### Recommendation Labels

Info findings use a recommendation label as their **action line**, in place of `> **Fix**:`. This concerns the action line only — an Info finding still carries a `diff` / `suggestion` block when a concrete fix exists (block format is decided by the fix, not the severity; see Formatting Rules). The label conveys the reviewer's assessment of whether action is needed.

| Label | Meaning | When to use |
|-------|---------|-------------|
| **Accept** | Acknowledged, no action needed | Intentional design choice, acceptable trade-off, or cosmetic preference |
| **Monitor** | No immediate action, track over time | Could become an issue at scale, under load, or after future changes |

There is deliberately no "Won't Fix" label. A finding not worth acting on is a `drop` disposition at Step 4.5 and does not reach the output at all — labelling it in the review would spend the reader's attention on a conclusion you already reached for them.

### Source Tags

Each finding displays a source tag after the title (e.g., `**Finding title** — Security`). Domain agent findings use their domain name. Codex findings use tags based on the active Codex mode:

| Codex Mode | Source(s) | Tag(s) |
|-----------|-----------|--------|
| **disabled** | Domain agents only | Domain name (e.g., `Security`, `Architecture`) |
| **adversarial** (default / `--codex`) | Domain agents + Codex adversarial | Domain name / `Codex` |
| **review** (`--codex-general`) | Domain agents + Codex review | Domain name / `Codex` |
| **both** (`--codex-both` / `--codex --codex-general`) | Domain agents + Codex review + Codex adversarial | Domain name / `Codex` (review findings) / `Codex Adv` (adversarial findings) |

When Codex mode is **adversarial** or **review** (single source), all Codex findings are tagged `— Codex`.
When Codex mode is **both** (dual source), review findings are tagged `— Codex` and adversarial findings are tagged `— Codex Adv` to distinguish the two sources.

All findings (domain + Codex) are sorted together by severity (Critical → Warning → Info), not grouped by source.

### Output Templates

Write the review in the language configured in the project's AGENTS.md (or CLAUDE.md as fallback).
If no language is configured, follow the user's conversational language.
Examples below are in Korean.

There are two output formats depending on the rendering medium:
- **Terminal format**: Used for Working Dir / Path mode, and for PR mode preview (before approval).
- **GitHub format**: Used only for the final PR comment published to GitHub (after approval).

#### Terminal Format

Optimized for CLI readability. No HTML tags, no tables, flat structure.
Three-level visual hierarchy: `────────────────────` (Unicode thin × 20) between severity sections, `---` between findings and after severity headers, blank lines within findings.
Each finding reads top-down as: what it is (title) → where (location) → **what the code says now and what it should say (evidence block)** → why that is a problem (근거).

The evidence block is a fenced ` ```diff ` block. This format is deliberate: the `-` / `+` characters carry the meaning as plain text, so the block stays readable in agents whose renderers lack syntax highlighting (Codex CLI, Gemini CLI, plain pipes). Never rely on color alone to convey before/after.

The location header below is written as `` `{file}:{primary_line}` `` throughout. When the finding carries a `location_marker` instead (no line exists for it), the header becomes `` `{file}` ({marker}) `` and the evidence block follows that marker's rule: `deleted` quotes the removed lines (`-`-only), `binary` quotes the diff's `Binary files ... differ` line, `missing` quotes the anchor, and **`file-level` quotes the relevant part of the file as a plain block** (the file exists — there are no removed lines to show). This substitution applies to every template that follows; it is not repeated in each one.

````markdown
## Code Review: {target}

Domains: {activated domains joined by " • "}{if Codex enabled: " · Codex 🤖"}
Findings: 🔴 {critical_count} critical · 🟡 {warning_count} warnings · 🟢 {info_count} info
{if dropped_count > 0:}
기각 {dropped_count}건 — {취향 {a} · 선행 이슈 {b} · 오늘 발현 안 함 {c}, omitting any zero bucket}
{end if}
{if -g flag set AND PR mode AND relationships found:}
📊 Change Graph: GitHub 게시 시 Mermaid 플로우차트 포함
   {module_count} modules · {file_count} files · 주요 흐름: {primary_flow_path}
{else if -g flag set AND PR mode AND no relationships found:}
📊 Change Graph: 파일 간 관계가 감지되지 않아 그래프를 생략합니다
{end if}

────────────────────

### < 🔴 Critical ({n}) >

---

**C1. {finding title}** — {domain}
`{file}:{primary_line}` ({N}곳)

```diff
- {current code, verbatim}
+ {proposed replacement}
```

**왜**: {why the current code is wrong — what breaks, under what input or condition}

---

**C2. {finding title}** — {domain}
`{file}:{primary_line}` ({N}곳)

```diff
  {unchanged context line, if needed for intelligibility}
- {current code, verbatim}
+ {proposed replacement}
```

**왜**: {why}

────────────────────

### < 🟡 Warning ({n}) >

---

**W1. {finding title}** — {domain}
`{file}:{primary_line}` ({N}곳)

```diff
- {current code, verbatim}
+ {proposed replacement}
```

**왜**: {why — if this conflicts with a rule or another file, name it and quote it:}
> {verbatim quote of the conflicting rule / code, with its source path}

{one or two sentences tying the quote back to the finding}

---

**W2. {finding title}** — {domain}
`{file}:{primary_line}` ({N}곳)

{When the fix is directional or structural — no single localized replacement — quote the current code alone and put the direction in the Fix line:}

```{lang}
{current code, verbatim}
```

**왜**: {why}

> **Fix**: {directional suggestion, referencing sections/functions by name}

────────────────────

### < 🟢 Info ({n}) >

---

{Info with a concrete fix — the block is a diff, exactly as at any other severity. Severity changes the action line, not the evidence format:}

**I1. {finding title}** — {domain}
`{file}:{primary_line}` ({N}곳)

```diff
- {current code, verbatim}
+ {proposed replacement}
```

**왜**: {what was observed and why it is worth changing}

> **Recommendation**: {Accept | Monitor} — {why this judgment — NOT how to fix it; the diff above already shows that}

---

{Info with nothing to replace — an intentional trade-off, or an observation to watch. Quote the code alone:}

**I2. {finding title}** — {domain}
`{file}:{primary_line}` ({N}곳)

```{lang}
{current code, verbatim}
```

**왜**: {what was observed, and why it does not warrant a change}

> **Recommendation**: {Accept | Monitor} — {reason}

{if any finding has disposition out-of-scope AND --full-scan is NOT set — omit this whole section otherwise. Under --full-scan these findings are reported in full above, not deferred here:}

────────────────────

### < 밖으로 미룸 ({n}) >

이 변경이 만든 문제가 아니라 리뷰 범위 밖으로 분류했습니다. 각 한 줄로만 적습니다.

{location follows the same rule as every other template: `{file}:{primary_line}` normally, `{file}` ({marker}) when the finding has no line. Never emit `:null`.}

- **{finding title}** — `{file}:{primary_line}` · {왜 이 변경 소관이 아닌지, 한 문장}
````

#### GitHub Format: Review Summary

Posted as the pull request review `body`. Contains the severity counts, key changes, and any findings that could not be mapped to diff lines (unmapped findings).

````markdown
## Code Review: {target}

**Domains**: {activated domains joined by " • "}{if Codex enabled: " · **Codex 🤖**"}
**Findings**: {severity counts joined by " · ", e.g., "🔴 1 Critical · 🟡 2 Warning · 🟢 1 Info"}{if unmapped > 0: " ({mapped count} inline, {unmapped count} general)"}
{if dropped_count > 0:}
**기각**: {dropped_count}건 — {취향 {a} · 선행 이슈 {b} · 오늘 발현 안 함 {c}, omitting any zero bucket}
{end if}

### Summary

{1-2 sentence overview of the PR's purpose and the review's key judgment — e.g., "Codex 통합을 위한 Step 2.5 추가 및 companion runtime 연동. 전반적으로 하위호환성이 잘 유지되나, 스킬 가용성 검증에 보완이 필요하다."}

### Key Changes
- {개념적 변경 1: 무엇을 왜}
- {개념적 변경 2}
- {개념적 변경 3}

{if -g flag set AND relationships found:}

### Change Graph

```mermaid
flowchart LR
  subgraph {module_name}[{Module Display Name}]
    {file_node_id}[{filename}]
  end
  {source_node} -->|{relationship_label}| {target_node}
```

{end if}

{if unmapped findings exist:}

---

### 📋 General Findings

{for each unmapped finding, sorted by severity:}

{severity_icon} **{finding title}** — {domain}
{if primary_line exists:}
`{file}:{primary_line}` ({N}곳)
{else — location_marker is set:}
`{file}` ({deleted | binary | missing | file-level}) ({N}곳)
{end if}

{if location_marker is `file-level`:}
{The file EXISTS; the finding just has no single line. Quote the relevant part of it as a plain block — there are no removed lines here.}
```{lang}
{the part of the file the finding concerns, verbatim}
```
{else if location_marker is set (deleted / binary / missing):}
{The code is gone from the new version — quote the REMOVED lines from the diff, `-`-only. For a binary artifact, quote the diff's `Binary files ... differ` line and state what was verified out-of-band. For `missing`, quote the anchor.}
```diff
- {removed lines, verbatim from the diff}
```
{else if a concrete replacement exists:}
```diff
- {current code, verbatim}
+ {proposed replacement}
```
{else:}
```{lang}
{current code, verbatim}
```
{end if}

**왜**: {why — quote the conflicting rule/code verbatim if the finding is about an inconsistency}

{if severity is Critical or Warning AND no concrete replacement was shown above:}
> **Fix**: {directional suggestion}
{end if}
{if severity is Info:}
> **Recommendation**: {Accept | Monitor} — {reason}
{end if}

---

{end for}

{if any finding has disposition out-of-scope AND --full-scan is NOT set — omit this whole section otherwise. Under --full-scan these findings publish through the ordinary path (inline or General Findings), so nothing is deferred here:}

### 밖으로 미룸

이 PR이 만든 문제가 아니라 리뷰 범위 밖으로 분류했습니다. 인라인 코멘트로 달지 않았습니다.

{location follows the same rule as every other template: `{file}:{primary_line}` normally, `{file}` ({marker}) when the finding has no line. Never emit `:null`.}

- **{finding title}** — `{file}:{primary_line}` · {왜 이 PR 소관이 아닌지, 한 문장}

{end if}

Generated with [Claude Code](https://claude.com/claude-code)
````

- **Summary**: 1-2문장으로 PR 목적과 리뷰 핵심 판단을 기술. PR description의 요약이 아닌 리뷰어 관점의 평가.
- **기각 line**: Step 4.5에서 `drop`으로 분류된 건수와 사유 분포만 표시. 개별 finding은 절대 전개하지 않는다.
- **밖으로 미룸**: `out-of-scope` disposition을 가진 finding만. 각 한 줄. 이 섹션의 finding은 `comments` 배열에 넣지 않는다 (Step 6 참조). `--full-scan`에서는 이 섹션 자체가 없다 (해당 finding이 일반 경로로 전문 게시되므로).
- **Key Changes**: 개념적 변경사항 bullet. 각 bullet은 하나의 논리적 변경 단위를 기술. 개수 제한 없이, PR이 수행한 변경만큼 기재.
- **Findings count**: unmapped가 0이면 total count만 표시. unmapped가 1 이상이면 `(N inline, M general)` breakdown 추가.

If there are zero findings overall, post: `## Code Review: {target}\n\n✅ No issues found.\n\nGenerated with [Claude Code](https://claude.com/claude-code)`

#### GitHub Format: Inline Comment

Posted as individual review comments, each attached to a specific file and line in the diff. One finding per comment.

The comment is already anchored to the offending lines in GitHub's diff view, so the reader can see the current code without an evidence block. Do NOT re-quote the commented-on lines here. The `왜` line still applies, and if the finding asserts a conflict with a rule or another file, that other side is NOT visible in the diff — quote it verbatim.

````markdown
{severity_icon} **{prefix}{n}. {finding title}** — {domain}

**왜**: {why the flagged lines are a problem}
{if the finding asserts a conflict with a rule or another file:}
> {verbatim quote of the conflicting rule / code}
> — `{source_file}:{line}`
{end if}

{Block: decided by whether a concrete replacement exists — at ANY severity.}
{if concrete code change:}
```suggestion
{suggested code — only the replacement lines, matching GitHub suggestion block format}
```
{end if}

{Action line: decided by severity.}
{if severity is Critical or Warning:}
{if no suggestion block was emitted above:}
> **Fix**: {directional suggestion}
{end if}
{else (Info):}
> **Recommendation**: {Accept | Monitor} — {why this judgment, not how to fix}
{end if}
````

Numbering uses the same scheme as Terminal format: `C{n}` / `W{n}` / `I{n}`, starting from 1 per severity. Terminal preview and GitHub inline comments share the same numbers for cross-reference.

For contextual match findings (mapped via contextual fallback), the comment is anchored to a diff line, but the issue lives in a **different** file that the reader cannot see. Quote that file's current code verbatim — without it the comment is unreadable.

````markdown
{severity_icon} **{prefix}{n}. {finding title}** — {domain}

📍 이 변경의 영향: `{affected_file}:{primary_line}` ({N}곳)

{if a concrete replacement exists:}
```diff
- {current code of affected_file, verbatim}
+ {proposed replacement}
```
{else:}
```{lang}
{current code of affected_file, verbatim}
```
{end if}

**왜**: {why}

{if severity is Critical or Warning AND no concrete replacement was shown above:}
> **Fix**: {directional suggestion}
{end if}
{if severity is Info:}
> **Recommendation**: {Accept | Monitor} — {reason}
{end if}
````

Contextual-match comments do not use ```suggestion``` blocks — GitHub applies suggestions to the commented line, which is in the wrong file here. Use a ```diff``` block instead.

Use GitHub `suggestion` blocks when the fix is a concrete, localized code change (variable rename, parameter addition, line replacement). The suggestion block enables one-click "Apply suggestion" in GitHub's UI. Use `> **Fix**: ...` for directional or structural suggestions that span multiple locations.

Severity does not decide this. An **Info** finding with a concrete one-line fix gets a `suggestion` block too — it simply carries `> **Recommendation**: ...` as its action line instead of `> **Fix**: ...`. Only findings with nothing concrete to replace (an accepted trade-off, an observation to monitor) go without a block.

### Formatting Rules

**Evidence rules** — a finding must let the reader judge it without opening the file. These apply to Terminal format and to the GitHub Review Summary (General Findings). **GitHub inline comments are the one exception**: the flagged lines are already visible in the diff there, so they are never re-quoted (see GitHub-specific rules). Everything the diff does NOT show — a conflicting rule, another file, a contextual-match target — is still quoted verbatim even in inline comments.
- **Verbatim only**: lines inside an evidence block are copied from the file as-is. Never paraphrase, re-indent, translate, or reconstruct them from memory. If the quoted lines do not actually say what the description claims, the finding is a false positive — drop it.
- **Block format is decided by the fix, NOT by the severity.** These are independent axes; do not couple them:
  - A concrete replacement exists → ` ```diff ` (`-` current, `+` proposed, unprefixed context lines as needed). This holds **at every severity** — an Info finding with a one-line fix gets a `diff` block just as a Critical one does.
  - Nothing to replace (the fix is directional/structural, or the finding needs no change at all) → a plain ` ```{lang} ` block quoting the current code.
  - Nothing exists yet (absent code) → a `+`-only ` ```diff ` block anchored per the absence rule below.
  - `-`/`+` must carry the meaning as text — never depend on the renderer's colors, which are absent in several host agents.
- **Absent code**: when the problem is that code is missing (a missing check, a missing entry, a file not updated to match a new pattern), quote the **anchor** — the existing lines the addition should sit next to, or the sibling that already does it right (with its own path) — and show the addition as `+` lines. Never drop an absence finding for lack of quotable lines.
- **Length**: 12 lines max per block. If the relevant lines are farther apart, keep the essential ones and elide the middle with `…`.
- **No change needed** (Info marked Accept): still quote the current code, then state why it is acceptable. "No action" without evidence is the least verifiable claim in a review and needs the most support, not the least.
- **Conflict claims**: if a finding asserts that code contradicts a rule, a convention, or another file, quote **both** sides verbatim with their paths. One-sided evidence cannot establish an inconsistency.
- **Action line is decided by the severity** (the other axis): Critical/Warning → `> **Fix**:`, Info → `> **Recommendation**:`. When a `diff` block already shows the replacement, omit `> **Fix**:` — it would just restate the block. Use `> **Fix**:` only for directional or structural suggestions no single replacement expresses.
- The `{reason}` in `> **Recommendation**: {label} — {reason}` explains **why that judgment**, not how to fix it. If a fix needs describing, it belongs in the evidence block (as a `diff`), not squeezed into the reason.

**Common rules (both formats)**:
- Location: `` `{file}:{primary_line}` `` on its own line, followed by the occurrence count (e.g., "(3곳)", "(1곳)"). The line number anchors the evidence block that follows; because the quoted lines are shown, the reader can still find the code if the line has since shifted.
- **No line exists for the finding** (a deleted file, a binary/generated artifact, a file that does not exist yet, or a finding about the file as a whole): omit `:{primary_line}` and write `` `{file}` `` alone with a marker — `` `{file}` (deleted) ``, `` `{file}` (binary) ``, `` `{file}` (missing) ``, `` `{file}` (file-level) ``. Never emit `:null` and never invent a line number. The evidence block follows the marker: removed lines (`-`-only, from the diff) for `deleted`, the diff header for `binary`, the anchor for `missing`, and the relevant part of the file as a plain block for `file-level` (the file exists, so there is nothing removed to quote). These findings are not dropped for lack of a quotable line.
- Line numbers belong in the location header (and in the source attribution of a quoted conflict) only. Do NOT scatter them through descriptions, `왜` text, or Fix suggestions — those reference locations by section heading, function name, or searchable code pattern.
- Finding title must NOT repeat the file name (location is already on its own line).
- Omit severity sections that have 0 findings.
- **Bullet management**: If a single finding has more than 5 sub-points, consolidate.
- **Commit SHA references**: Never use backticks around SHAs. Use plain text or markdown links.

**Graph rules (when `-g` flag is set)**:
- Mermaid direction: `flowchart LR` (left-to-right) for readability.
- Node labels: filename only (no full path). Use `[filename.ext]` format (basename + extension).
- Edge labels: relationship type — code-level (`imports`, `calls`, `extends`, `emits/consumes`, `reads/writes`) or conceptual (`references`, `shared logic`, `configures`).
- Module grouping: When changed files > 7, group by parent directory using `subgraph`. When ≤ 7, show individual file nodes without subgraph.
- Terminal: One-line summary only — module count, file count, primary flow path (longest chain of connected nodes, max 4 nodes joined by ` → `).
- GitHub: Full Mermaid code block placed after Key Changes, before General Findings.

**Terminal-specific rules**:
- No `<details>` or HTML tags — they don't render in CLI.
- Summary line with severity icons: `Findings: 🔴 1 critical · 🟡 3 warnings · 🟢 1 info`.
- Severity headers use `### < {icon} {Severity} ({n}) >` format with `< >` brackets.
- Severity sections are separated by `────────────────────` (Unicode thin box drawing × 20).
- Severity header is followed by `---` before the first finding. Findings within the same severity are also separated by `---`.
- Each finding is numbered with a severity prefix: `C{n}` (Critical), `W{n}` (Warning), `I{n}` (Info), starting from 1 per severity.
- Finding order: title (bold — renders bright in CLI) → location → evidence block → `**왜**` → action line (if any).
- `**왜**` is a bold paragraph, not a list item, so blank lines around it render. Quoted conflicting rules go in a `>` blockquote directly under it.
- Action line by severity: Critical/Warning use `> **Fix**: ...` only when the evidence block did not already show the replacement. Info uses `> **Recommendation**: {Accept | Monitor} — {reason}` to convey whether action is needed.
- Domains with no findings: omit entirely (no "✅ ... No issues found" line).
- Zero findings overall: display `## Code Review: {target}\n\n✅ No issues found.` — severity sections, summary line 모두 생략.
- Codex failure notice (if applicable): append after the summary line per the error classification in Step 3 Codex Failure Handling (`⚠️` for auth errors, `ℹ️` for other failures). Only shown when Codex mode was NOT disabled but companion returned non-zero exit. Do NOT include in GitHub format.

**GitHub-specific rules (inline review comments)**:
- Review Summary: severity counts and key changes at the top. Unmapped findings (after contextual mapping) included below under "General Findings" as fallback only.
- Inline Comment: one finding per comment. No `<details>` tags — each comment is self-contained.
- Evidence in inline comments: the flagged lines are visible in GitHub's diff view, so do NOT re-quote them. Anything the diff does NOT show — a conflicting rule, another file's code, a contextual-match target — must still be quoted verbatim.
- Block vs. action line are independent axes in inline comments too:
  - **Block** (decided by the fix, at any severity): concrete localized change → GitHub `suggestion` block (enables one-click apply). Nothing concrete to replace → no block. Contextual-match findings target a different file, so `suggestion` would apply to the wrong lines — use a `diff` block there instead.
  - **Action line** (decided by severity): Critical/Warning → `> **Fix**: ...`, emitted only when no `suggestion` block already showed the fix. Info → `> **Recommendation**: {Accept | Monitor} — {reason}`, which may accompany a `suggestion` block when the Info finding has a concrete fix.
- Domains with no findings: omit entirely from both summary and inline comments.

---

## Step 6: Publisher

Based on the mode and flags, publish the review output.

**Quick mode**: No changes to publishing behavior. The same Working Dir / PR mode flow applies.

### Working Dir / Path Mode

Display the review output using **Terminal format** directly to the user. No GitHub publishing.

### PR Review Mode

PR mode uses a two-phase flow: preview in terminal, then publish to GitHub.

#### Phase 1: Preview (Terminal format)

Generate the review using **Terminal format** and display it to the user in the conversation.

| Condition | Next step |
|-----------|-----------|
| `-y` or `-f` flag present | Skip preview, go directly to Phase 2 |
| `$ARGUMENTS` contains publish intent ("comment 달아", "바로 올려", "게시해", "post it") | Skip preview, go directly to Phase 2 |
| **Default** | **STOP here. Show the Terminal format output and ask: "PR에 게시할까요?" Do NOT proceed until the user approves.** |

#### Phase 2: Publish (Inline Review)

After approval (or auto-publish flag), publish findings as **inline review comments** via the GitHub Pull Request Review API. Each finding becomes an individual comment attached to the relevant file and line in the diff, enabling per-finding resolution and reply through GitHub's native UI.

**1. Resolve line targets**

For each confirmed finding, verify its primary line number exists in the PR diff:

```
gh pr diff {number}
```

For each confirmed finding, resolve its target line in the PR diff:

0. **No line to target** (`primary_line` was omitted and `location_marker` is set — `deleted` / `binary` / `missing` / `file-level`): there is no line to anchor an inline comment to. A deleted file's code lives on the diff's LEFT side (and this skill does not post LEFT-side comments), a binary file has no hunk at all, and a `file-level` finding is about the file as a whole rather than any one line. Skip tiers 1–2 entirely and route the finding straight to **General Findings** (tier 3) in the review body, keeping its `` `{file}` ({marker}) `` header and evidence block. Never fabricate a line to force an inline anchor — the Review API would 422, and the no-line rule forbids it. Never drop the finding either: a validation or auth check deleted by this PR is exactly what this branch exists to carry.

1. **Exact match**: The finding's file + line appears in a diff hunk (added line `+`,
   or context line on the RIGHT side). Include as an inline comment at that location.

2. **Contextual match**: Exact match failed. Search the diff for the root cause change
   that triggered this finding:
   - Rename not propagated → map to the diff line where the rename occurred
   - Missing entry that should accompany new entries → map to the nearest new entry line
   - Dead code from a removal → map to a nearby RIGHT-side context or addition line adjacent to the removal
   Include as an inline comment at the contextual location (must be a RIGHT-side line). Build the comment body with the **contextual match template** in Step 5 (GitHub Format: Inline Comment) — location header `📍 이 변경의 영향: \`{affected_file}:{primary_line}\` ({N}곳)`, then an evidence block quoting the affected file's current code verbatim, then `**왜**`. The affected file is not visible in this diff, so the quote is what makes the comment readable; do not drop it.

3. **Unmapped**: Neither exact nor contextual match is possible (e.g., the finding references
   a file/pattern with no related changes in the diff), **or the finding has no `primary_line`
   per tier 0**. Include in the review body under "📋 General Findings".

**2. Build review payload**

Construct a JSON payload for the Review API:

```json
{
  "commit_id": "{head_sha}",
  "event": "COMMENT",
  "body": "{Review Summary template}",
  "comments": [
    {
      "path": "{file}",
      "line": {end_line_number},
      "side": "RIGHT",
      "start_line": {start_line_number},
      "start_side": "RIGHT",
      "body": "{Inline Comment template}"
    }
  ]
}
```

- `commit_id`: Obtain from `gh pr view {number} --json headRefOid --jq '.headRefOid'`
- `body`: Review Summary template (severity counts + key changes + unmapped findings if any + footer)
- `comments`: Array of mapped findings, each using the Inline Comment template
- For single-line findings, omit `start_line` and `start_side` — only `line` and `side` are needed.
- Findings with disposition `out-of-scope` (Step 4.5) are body-only **in default mode**, under `밖으로 미룸` — they never enter the `comments` array even when their line maps cleanly. An inline comment is a request for a change in this PR; attaching one to a pre-existing issue asks the author to fix something they did not touch. **Under `--full-scan` this exclusion is lifted**: the user has explicitly asked for pre-existing issues, so out-of-scope findings resolve their line targets and publish like any other finding (tiers 1–3 above), and the `밖으로 미룸` section is omitted.
- Findings carrying a `location_marker` (no `primary_line`) NEVER appear in the `comments` array. They are body-only, under General Findings. The Review API has no valid anchor for them, and coercing one into `"side": "RIGHT"` either 422s or points at unrelated code.
- Serialize the JSON payload using `jq -n` or write to a temp file — do NOT manually escape strings in a heredoc. Comment bodies contain multi-line markdown, code fences, and `suggestion` blocks that break raw string interpolation.

If there are no mapped findings (all unmapped), the `comments` array is empty. The review body contains all findings.

**3. Submit**

```
jq -n \
  --arg commit_id "$HEAD_SHA" \
  --arg body "$REVIEW_BODY" \
  --argjson comments "$COMMENTS_JSON" \
  '{commit_id: $commit_id, event: "COMMENT", body: $body, comments: $comments}' \
| gh api repos/{owner}/{repo}/pulls/{number}/reviews --method POST --input -
```

Where `{owner}/{repo}` is obtained from `gh repo view --json nameWithOwner --jq '.nameWithOwner'`.

**4. Fallback**

If the Review API call fails (e.g., 422 due to invalid line mapping):
1. Move all inline comments to the Review Summary body as General Findings, then resubmit with an empty `comments` array. GitHub's 422 does not specify which comment failed, so individual retry is not possible.
2. If the retry also fails or the API is unavailable, fall back to posting the full review as a single PR comment:
   ```
   gh pr comment {number} --body "$(cat <<'EOF'
   {Review Summary with ALL findings included in body}
   EOF
   )"
   ```

---

## Task

1. Parse `$ARGUMENTS` to determine mode (PR / Working Dir / Path) and flags (including Codex flags and `--quick`).
2. **Context Builder**: Gather diff, commit history, related files, and PR description (if applicable).
3. **Domain Router**: Analyze changed file types and activate relevant domains. Respect `--domain` override. If `--quick`, cap to 2 domains by priority.
4. **Codex Detection**: If `--quick`, skip (Codex disabled). Otherwise, resolve companion path via `find`, and determine Codex mode (adversarial / review / both / disabled).
5. **Domain Agents + Codex**: If `--quick`, perform single-pass analysis in main context (no agent spawn). Otherwise, launch activated domain agents in parallel. If Codex is enabled, launch Codex agent(s) via companion runtime concurrently. Collect all findings. If Codex fails (non-zero exit), proceed with domain findings only.
6. **Cross-Validation**: If `--quick`, lightweight validation (context check + sanity only). Otherwise, verify each finding (domain + Codex) against expanded context, git history, comments, and PR intent. Classify as Confirmed / Demoted / Dismissed.
7. **Reporting Bar**: Assign a disposition (`ship-blocker` / `in-scope-gap` / `out-of-scope` / `drop`) to every Confirmed and Demoted finding. Report the first two in full, the third as one line under `밖으로 미룸`, and the fourth as a count only. Skipped entirely when `-a`/`--all` is set.
8. **Output Generator**: Produce severity-first structured output with source tags, plus the `기각` line when anything was dropped. If `--quick`, show Critical/Warning only; if none, fallback to Info; graph always omitted.
9. **Publisher**:
   - **Working Dir / Path mode**: Display Terminal format output directly. Done.
   - **PR mode without `-y`/`-f`**: Show Terminal format preview → ask "PR에 게시할까요?" → resolve line targets → build inline review payload → publish via Review API ONLY if approved.
   - **PR mode with `-y`/`-f`**: Resolve line targets → build inline review payload → publish via Review API immediately.

**Important:**
- Do NOT publish review comments to GitHub without explicit user approval. The only exceptions are the `-y`/`-f` flag or explicit publish intent in `$ARGUMENTS`. A PR number alone is NOT publish intent.
- Do NOT replace linter checks — focus on semantic, architectural, and logic issues.
- Do NOT report every true finding. Truth is Step 4's bar; **worth the reader's attention is Step 4.5's**, and only the second decides the output. A review whose findings are all actionable is the goal, not a review that is long.
- Do NOT suggest auto-fix or auto-merge — this is review only.
- This skill is independent from `review-reply`. `code-review` generates reviews (proactive); `review-reply` responds to received reviews (reactive).
- **Assignee**: If creating GitHub PRs or issues, always include `--assignee @me`.
- **Commit references**: Never wrap commit SHAs in backticks. Use plain text or explicit markdown links.
- Adapt output language to the project's AGENTS.md (or CLAUDE.md as fallback) language setting.
- When Agent tool is unavailable, perform all analyses sequentially as a single-pass fallback.
