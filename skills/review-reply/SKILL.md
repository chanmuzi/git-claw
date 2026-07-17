---
name: review-reply
description: >-
  Review PR comments, discuss improvements, and reply with resolution status.
  TRIGGER when: user asks to review PR feedback, check review comments, address reviewer suggestions, or handle code review (e.g., "리뷰 확인해줘", "review comments", "피드백 반영해줘").
  DO NOT TRIGGER when: user is creating PRs, committing, or performing git operations without review intent.
argument-hint: "[PR번호]"
version: "1.4.0"
allowed-tools: Bash(gh *), Bash(git *), Read, Grep, Glob
---

## Identify Target PR

Parse `$ARGUMENTS` to extract a PR number or URL.
If not provided, detect from the current branch:

```
gh pr view --json number --jq '.number' 2>/dev/null
```

If no PR is found, ask the user to specify one.

### Verify the working tree matches the PR head

An explicitly named PR (`/review-reply 42`) may have nothing to do with the currently checked-out branch. Every evidence step below reads the **local working tree**, so reading the wrong revision would quote unrelated code as "verbatim evidence", misclassify live comments as outdated, and land approved fixes on the wrong branch. Silently reading the wrong revision is the one failure mode that makes every downstream quote in this skill a lie.

Check **both** conditions before reading anything:

```
gh pr view {number} --json headRefOid --jq '.headRefOid'   # PR head
git rev-parse HEAD                                          # local HEAD
git status --porcelain                                      # uncommitted changes
```

**1. Head mismatch** (`headRefOid` ≠ `HEAD`). Tell the user which branch the PR points at, and pick one:
- **Check out the PR branch** (`gh pr checkout {number}`), then proceed normally. Preferred.
- **Read at the PR head without checking out.** The head commit of a fork or an unfetched branch is NOT in the local object store, so fetch it first — otherwise `git show` fails with `fatal: bad object`:
  ```
  git fetch origin pull/{number}/head
  ```
  Then quote all evidence from `git show {head_sha}:{path}` rather than the working tree. **Step 4 (applying fixes) is blocked on this path** — a fix cannot be applied to a tree that is not on the PR's branch.

**2. Dirty working tree** (`git status --porcelain` is non-empty), even when the head matches. Uncommitted edits mean the file on disk is not what the PR contains: evidence quoted from it may show code that exists nowhere in the PR, and an applied fix would be built on top of unrelated work. Tell the user what is uncommitted and pick one: commit/stash first, or read evidence from `git show {head_sha}:{path}` and block Step 4.

**Record the baseline.** Save the `headRefOid` you just read — call it `{baseline_sha}`. It is the revision your analysis is about, and the only correct thing to compare against later.

**Re-verify before acting.** Someone may push to the PR while you are analyzing. Immediately before Step 4 (applying fixes) and again before Step 5 (posting replies, resolving threads), re-read `headRefOid` and compare it to **`{baseline_sha}`** — NOT to local `HEAD`:
- `headRefOid` ≠ `{baseline_sha}` → the PR moved under you. Your judgments were made against a revision that is no longer current. Stop and re-collect. Resolving a thread on a stale judgment is not reversible.
- Do NOT compare against local `HEAD` here. Applying a fix in Step 4 creates a commit, which advances local `HEAD` while the PR head legitimately stays behind until push. Comparing to `HEAD` would read that normal flow as "the head moved" and abort it. And on the read-at-head path, local `HEAD` never matches the PR head by design.

## Step 1: Collect Review Comments

Fetch all review comments on the PR (both human and bot/AI):

```
gh api repos/{owner}/{repo}/pulls/{number}/comments --jq '.[] | {id: .id, path: .path, line: .line, body: .body, user: .user.login, created_at: .created_at, diff_hunk: .diff_hunk, side: .side, subject_type: .subject_type, original_line: .original_line, original_commit_id: .original_commit_id}'
```

`diff_hunk` / `side` / `original_line` / `original_commit_id` are the fallback source of evidence when the code cannot be read at `line` in the current file.

A missing `line` does NOT by itself mean the comment is stale. Check `subject_type` and `side` before concluding anything — three different situations look identical if you only look at `line`:

| Condition | Meaning | How to quote |
|-----------|---------|--------------|
| `subject_type: "file"` | **File-level comment** — attached to the whole file, so it never had a line. Perfectly live feedback | Quote the part of the file the comment is actually about (or the file's structure), and use the file-level header. Do NOT treat it as outdated. |
| `side: RIGHT`, code found at `line` | Normal case | Quote the current file at `line`. |
| `side: RIGHT`, `line` is null or the code is gone | **Outdated** — later pushes moved or deleted those lines | Quote `diff_hunk` and label it: `현재 파일에서 해당 코드를 찾을 수 없습니다 (후속 push로 outdated). 리뷰 시점 기준 인용:` |
| `side: LEFT` | **Not outdated** — the comment targets a line *this PR deletes*. Its absence from the new file is the point, not staleness | Quote `diff_hunk` and label it as the removed code: `이 PR이 삭제한 코드에 대한 코멘트입니다. 삭제된 원본:` |

Never quote the current file at a stale line number — that quotes the wrong code, which is worse than quoting nothing. Never label a `LEFT` comment as outdated: doing so dismisses valid feedback about deleted logic (a removed validation, a dropped auth check) as a stale artifact. And never label a file-level comment as outdated either — it has no line because it never had one.

Also fetch PR-level review summaries:

```
gh api repos/{owner}/{repo}/pulls/{number}/reviews --jq '.[] | {id: .id, user: .user.login, state: .state, body: .body}'
```

Also fetch issue-level comments (used by AI reviewers like CodeRabbit):

```
gh api repos/{owner}/{repo}/issues/{number}/comments --jq '.[] | {id: .id, body: .body, user: .user.login, html_url: .html_url, created_at: .created_at}'
```

Fetch review thread IDs for resolving conversations later (GraphQL):

```
cat <<'GRAPHQL' | gh api graphql --input -
{"query": "query($owner: String!, $repo: String!, $number: Int!) { repository(owner: $owner, name: $repo) { pullRequest(number: $number) { reviewThreads(first: 100) { nodes { id isResolved comments(first: 1) { nodes { databaseId } } } } } } }", "variables": {"owner": "{owner}", "repo": "{repo}", "number": {number}}}
GRAPHQL
```

Map each `databaseId` (= REST comment ID) to its GraphQL `threadId` for the resolve step.

Include all reviewers — human teammates and AI bots alike. Do NOT filter by user type.
Track each comment's source type (review comment vs. issue comment) for the reply and resolve steps.

## Step 2: Analyze Each Suggestion

For each review comment:

1. Read the referenced file and surrounding code context (at least 20 lines around the mentioned line).
2. Understand the reviewer's suggestion in the context of the actual code.
3. **Capture the evidence you just read.** Copy the relevant lines verbatim — you will show them to the user in Step 3, not just reason over them privately. If the reviewer claims the code conflicts with a rule, a convention, or another file, open that source too and copy its lines as well.
   - **No line, or code not in the current file**: check `subject_type` and `side` before concluding anything (see the table in Step 1). `subject_type: "file"` → a live file-level comment that never had a line. `RIGHT` + missing → outdated, quote `diff_hunk` as the review-time state. `LEFT` → the comment targets a line this PR deletes; it is valid, not stale. Never guess at a line number in any of these cases, and never call a comment outdated merely because `line` is null.
   - **Comment not anchored to code**: PR-level review bodies and issue comments (CodeRabbit summaries, "이 PR은 범위가 너무 큽니다" 등) have no `path`/`line`. There is no source line to quote — quote **the reviewer's own words** verbatim instead, and state which files you checked to reach your assessment. Do not fabricate a `{file}:{line}` for these.
4. Evaluate the suggestion's validity:

| Category | Criteria |
|----------|----------|
| **Valid** | Real bug, security issue, or meaningful improvement |
| **Debatable** | Stylistic preference or trade-off that could go either way |
| **Can Safely Ignore** | Misunderstands the code, already handled elsewhere, or not applicable |

Verifying against the real code is what separates an assessment from an echo of the reviewer. If the file does not actually say what the reviewer claims, the comment belongs in **Can Safely Ignore** — and the quoted lines are how you show that.

## Step 3: Present Findings

Group findings by category and present to the user.

The user is deciding whether to accept each suggestion, and they should be able to decide **without opening the files themselves**. Every item therefore shows the code or document as it is actually written, and what it would become. A summary of the reviewer's opinion is not evidence; the source lines are.

The evidence block is a fenced ` ```diff ` block. The `-` / `+` characters carry the meaning as plain text, so the block survives agents whose renderers lack syntax highlighting (Codex CLI, Gemini CLI, plain pipes). Never rely on color alone to convey before/after.

````
## Review Analysis for PR #{number}

Reviewers: {reviewer names joined by " · "}
Comments: ✅ {valid_count} valid · 🤔 {debatable_count} debatable · ❌ {ignore_count} ignored

────────────────────

### < ✅ Valid Suggestions ({n}) >

---

**V1. {summary}**
`{file}:{line}` · {reviewer_name}

**리뷰어**: {verbatim quote of the review comment — the reviewer's own words, not your paraphrase}

```diff
- {current code, verbatim}
+ {proposed replacement}
```

**왜**: {why the reviewer is right — what breaks, or what rule this violates}
> {if the finding is about a conflict: verbatim quote of the conflicting rule / code}
> — `{source_file}:{line}`

---

**V2. {summary}**
`{file}:{line}` · {reviewer_name}

**리뷰어**: {verbatim quote of the review comment}

{When the fix is directional — no single localized replacement — quote the current code alone:}

```{lang}
{current code, verbatim}
```

**왜**: {why}

> **Fix**: {directional suggestion}

────────────────────

### < 🤔 Debatable ({n}) >

---

**D1. {summary}**
`{file}:{line}` · {reviewer_name}

**리뷰어**: {verbatim quote of the review comment}

```diff
- {current code, verbatim}
+ {what the reviewer proposes}
```

**Pros**: {what applying it buys}

**Cons**: {what it costs — why this is a genuine trade-off, not an obvious win}

> **추천**: {Apply | Won't Fix | Follow-up} — {1-line reasoning}

────────────────────

### < ❌ Can Safely Ignore ({n}) >

---

**X1. {summary}**
`{file}:{line}` · {reviewer_name}

**리뷰어 주장**: {verbatim quote of the review comment — what the reviewer claims the code does / should do}

```{lang}
{current code, verbatim — the lines that disprove or already satisfy the claim}
```

**왜 불필요**: {how the quoted code refutes the claim — already handled here, convention differs, reviewer read a different path, etc.}

────────────────────

{Comments whose code is not in the current file — outdated (`RIGHT`, lines moved/deleted by later pushes) or `LEFT` (lines this PR deletes). `line` may be null; do NOT fabricate one. Use the header below, in whichever category the item belongs to:}

**{V|D|X}{n}. {summary}**
`{file}` ({outdated · 리뷰 시점 L{original_line} | 삭제된 코드}) · {reviewer_name}

**리뷰어**: {verbatim quote of the comment}

{현재 파일에서 해당 코드를 찾을 수 없습니다 (후속 push로 outdated). 리뷰 시점 기준 인용: | 이 PR이 삭제한 코드에 대한 코멘트입니다. 삭제된 원본:}

```diff
{diff_hunk, verbatim}
```

**왜**: {assessment}

---

{File-level comments (`subject_type: "file"`) — attached to a file but never to a line. Live feedback, not outdated. Use this header, and keep the labels of whichever category the item lands in — V/D use `**리뷰어**` + `**왜**`; X uses `**리뷰어 주장**` + `**왜 불필요**`:}

**{V|D|X}{n}. {summary}**
`{file}` (파일 전체) · {reviewer_name}

**{리뷰어 | 리뷰어 주장}**: {verbatim quote of the comment}

```{lang}
{V/D: the part of the file the comment is actually about, verbatim.
 X: the lines that refute the claim — which are often NOT where the reviewer was looking.
    Quote what disproves them or already satisfies them, elsewhere in the file or in
    another file (with its own path). Quoting only what the reviewer described proves nothing.}
```

**{왜 | 왜 불필요}**: {assessment}

---

{Comments with no file anchor — PR-level review bodies, issue comments (CodeRabbit summaries, etc.). They have no `path`/`line`, so there is no source line to quote. Use this header instead, in whichever category the item belongs to:}

**{V|D|X}{n}. {summary}**
PR 전체 코멘트 · {reviewer_name}

**리뷰어**: {verbatim quote of the comment}

{Then quote whatever code your assessment actually rests on — the files you checked — or, if the comment is about process rather than code, state explicitly which files you examined and what you found. Never invent a `{file}:{line}` for these.}

**왜**: {assessment}
````

Rejecting a suggestion demands **more** evidence than accepting one, not less. `❌ Can Safely Ignore` therefore quotes the code that actually refutes the reviewer — the user cannot check "this is already handled elsewhere" against a claim alone. An X item with no evidence at all is not dismissible; re-examine it. (For an unanchored comment there is no code to quote — then the evidence is the files you checked and what you found there, stated explicitly.)

**Evidence rules** (kept in sync with `code-review` — see CLAUDE.md 스킬 간 공유 로직):
- **Two sources per item**: the reviewer's words AND the code. Quote both. The comment is the claim under evaluation; the code is what decides it. Dropping either leaves the user unable to check your assessment.
- **Verbatim only**: lines inside an evidence block are copied from the file as-is. Never paraphrase, re-indent, or reconstruct them from memory. If the file does not say what you expected, that is a finding in itself — the reviewer (or you) misread it.
- **Block format is decided by the fix, NOT by the category**: a concrete replacement exists → ` ```diff `; nothing to replace (directional fix, or the item needs no change) → a plain ` ```{lang} ` block quoting the current code; the problem is *absent* code → a `+`-only ` ```diff ` anchored on the lines the addition should sit next to, or on the sibling that already does it right. Never drop an item for lack of quotable lines — quote the anchor.
- **No line, or code not in the current file**: branch on `subject_type` and `side` (Step 1 table). `subject_type: "file"` → a live file-level comment; quote the part of the file it is about. `RIGHT` + gone → outdated; quote `diff_hunk` as the review-time state. `LEFT` → the comment targets a line this PR deletes; it is valid feedback, not stale — quote `diff_hunk` as the removed code. A null `line` is never by itself proof of staleness. Never quote the current file at a stale line number, and never dismiss a `LEFT` or file-level comment as outdated.
- **Unanchored comments**: PR-level and issue comments have no code location. Quote the reviewer's words and name the files you checked. Do not fabricate a `{file}:{line}`.
- **Length**: 12 lines max per block. Keep the essential lines and elide the middle with `…` if they are far apart.
- **Conflict claims**: quote **both** sides verbatim with their paths. One-sided evidence cannot establish an inconsistency.
- **Redundancy**: when a `diff` block already shows the fix, omit the `> **Fix**:` line. Use `> **Fix**:` only for directional suggestions no single replacement expresses.

**Layout rules**:
- Use bold paragraphs with category prefix (`**V1. ...**`, `**D1. ...**`, `**X1. ...**`) instead of numbered lists so that blank lines between items render correctly in terminal. Prefixes: V = Valid, D = Debatable, X = Ignore. Each category numbers independently (V1, V2, …; D1, D2, …; X1, X2, …).
- Item order: summary (bold) → location header → `**리뷰어**` quote → evidence block → assessment.
- Location header has four forms, by what the comment is attached to: `` `{file}:{line}` · {reviewer_name} `` (normal) · `` `{file}` (outdated · 리뷰 시점 L{original_line} | 삭제된 코드) · {reviewer_name} `` (code not in the current file) · `` `{file}` (파일 전체) · {reviewer_name} `` (`subject_type: "file"`) · `PR 전체 코멘트 · {reviewer_name}` (unanchored). Never emit `:null` and never invent a line number.
- `**리뷰어**`, `**Pros**`, `**Cons**`, `**왜**` are bold paragraphs, each separated by a blank line. Without the blank line, consecutive lines collapse into one wrapped paragraph in the terminal.
- Three-level visual hierarchy: `────────────────────` (Unicode thin × 20) between categories, `---` between items within the same category, blank lines within items.
- Category headers use `### < {icon} Category ({n}) >` format with count.
- No HTML tags and no tables — neither renders reliably across host agents.
- Omit categories that have 0 items.

## Step 4: Discuss and Apply

**Precondition**: **both** guards in "Verify the working tree matches the PR head" must have passed — the PR head matches local `HEAD`, **and** the working tree is clean. If either check routed you to the read-only `git show {head_sha}:{path}` path, Step 4 is blocked. Never apply a fix to a tree that is not the PR's branch content: the change would land on the wrong branch, or on top of unrelated uncommitted work. Also re-check `headRefOid` against `{baseline_sha}` before applying.

For each Valid or Debatable suggestion:
1. Ask the user whether to apply the change.
2. If approved, make the code change using Edit.
3. For items NOT applied, determine the disposition:
   - **Won't Fix**: Not applicable, intentionally not addressing, or disagree with the suggestion. Record reason in reply only.
   - **Follow-up (out of scope)**: Valid but out of scope for this PR. Will be tracked as an issue.
   - **Follow-up (blocked)**: The user approved it, but Step 4 was blocked (head mismatch or dirty tree), so it could not be applied here. Never record it as Applied (there is no commit) and never as Won't Fix (nobody decided against it). Tracked as an issue like any other Follow-up.
4. After all approved changes are applied, suggest a commit message following the project's commit convention (e.g., `fix: 리뷰 피드백 반영`).
5. **Record what you produced.** After the fix commit is created and pushed, save its SHA as `{expected_head_sha}`. The PR head is now *supposed* to differ from `{baseline_sha}` — you moved it. Step 5 uses `{expected_head_sha}` to tell your own push apart from someone else's, so without this the normal review-reply flow would abort on its own commit.

## Step 5: Reply to Review Comments

**Precondition**: re-read `headRefOid` and compare it against what you expect it to be. **Which SHA you expect depends on whether Step 4 produced a commit** — conflating the two either aborts the normal flow or records work that is not on the remote:

| Step 4 outcome | Expected `headRefOid` | If it differs |
|----------------|----------------------|---------------|
| Fixes applied, committed **and pushed** | `{expected_head_sha}` (the fix commit you pushed) | Someone else pushed on top of yours. Stop and re-collect. |
| No fixes applied (all Won't Fix, or Step 4 blocked) | `{baseline_sha}` (unchanged since collection) | The PR moved under you. Stop and re-collect. |

Never compare against local `HEAD`: it advances when Step 4 commits, and on the read-at-head path it never matches by design.

Two guards, both about not recording work that did not happen:
- **No commit exists** (Step 4 was blocked, or the user approved a change that could not be applied): do NOT post `✅ 반영 완료` — there is no commit to link — and do NOT resolve the thread. An approved-but-unapplied item is **Follow-up (blocked)**, never Applied and never Won't Fix. Alternatively defer Step 5 entirely until the branch is checked out.
- **The fix commit exists locally but is NOT pushed**: `✅ 반영 완료` links a commit SHA, and a link to a commit that is not on the remote resolves to nothing for the reviewer. **Push first, then reply.** Do not post the reply and do not resolve the thread until the remote PR head is `{expected_head_sha}`. Posting and resolving are not reversible; pushing afterwards does not repair a dead link that reviewers already saw.

After the commit is created, reply to each review comment on GitHub to record the resolution.

For each comment that was discussed (Valid, Debatable, or Ignored), reply using the appropriate API based on the comment's source type:

**For review comments (inline file comments):**
```
gh api repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies -f body="{reply_body}"
```

**For issue comments (CodeRabbit, etc.):**
```
gh api repos/{owner}/{repo}/issues/{number}/comments -f body="{reply_body}"
```
For issue comments only, prepend a quote line before the structured format: `> Re: @{user} [{comment_url}]`
This quote line is part of the format for issue comments — it does not apply to review comments.

### Reply format

Replies MUST follow the exact structured format below. Do NOT use free-form prose or unstructured sentences.

Write the reply body in the language configured in the project's CLAUDE.md. If no language is configured, follow the user's conversational language. Examples below are in Korean:

**If applied:**
```
✅ **반영 완료**

- **커밋**: [{short_sha}](https://github.com/{owner}/{repo}/commit/{short_sha})
- **변경**: {1-line summary of what was changed}
```

**If Won't Fix:**
```
⏭️ **미반영 (Won't Fix)**

- **사유**: {concise reason — e.g., 프로젝트 컨벤션과 상충, 이미 다른 방식으로 처리됨}
```

**If Follow-up:**
```
🔜 **후속 작업 예정**

- **사유**: {concise reason — e.g., 현재 PR 범위 밖, 별도 작업으로 진행 예정}
- **이슈**: #{issue_number}
```

If no blanket approval was given, present all planned replies for approval before posting.
A blanket instruction (e.g., "모두 반영", "다 적용해", "apply all") covers all remaining steps — code changes, replies, and thread resolution — so do not re-ask per step.

### Resolve concluded threads

After posting replies, resolve the conversation thread using the GraphQL API:

```
cat <<'GRAPHQL' | gh api graphql --input -
{"query": "mutation($id: ID!) { resolveReviewThread(input: {threadId: $id}) { thread { isResolved } } }", "variables": {"id": "{thread_id}"}}
GRAPHQL
```

Use the `databaseId → threadId` mapping collected in Step 1.

#### Resolve rules

| Disposition | Resolve? | Reason |
|-------------|:--------:|--------|
| ✅ Applied | Yes | Work is complete |
| ⏭️ Won't Fix | Yes | Decision is final, no further action |
| 🔜 Follow-up | No | Open work remains — issue tracks it |

Resolve MUST happen immediately after posting the reply for each concluded thread. Do NOT skip this step.
Issue comments (CodeRabbit, etc.) do not have resolvable threads — skip the resolve step for those.

### Create follow-up issue for deferred items

If there are **Follow-up** items — whether classified in Step 4 (out of scope) or forced by the Step 5 precondition (approved but blocked) — group related ones and create a consolidated issue. Every Follow-up reply carries an issue number, so every Follow-up item needs an issue.
Do NOT create issues for Won't Fix items — their reasons are already recorded in the PR reply.

```
gh issue create --title "[Review] PR #{number} 후속 작업" --body "$(cat <<'EOF'
PR #{number} 리뷰에서 확인된 후속 작업 항목.

### 1. {summary}
- 원본: {comment_url}
- 사유: {why deferred}

### 2. {summary}
- 원본: {comment_url}
- 사유: {why deferred}
EOF
)"
```

- Group by relevance — do NOT create one issue per comment.
- If all follow-up items are closely related, create a single issue.
- If there are clearly distinct groups, create one issue per group (max 2-3).
- Skip issue creation if there are no follow-up items.

**Important:**
- Do NOT apply any code changes without explicit user approval for each item.
- Do NOT post reply comments without explicit user approval.
- A blanket instruction (e.g., "모두 반영해줘", "apply all") counts as explicit approval for all steps. Do not ask again per-step.
- Read the actual code context before judging — do not rely solely on the review comment.
- **Show the evidence, do not just consult it.** Every item in Step 3 quotes the source verbatim. An item you cannot quote is an item you have not verified, in either direction: neither a suggestion nor a dismissal of it may rest on recollection.
- Consider the project's existing patterns, conventions, and CLAUDE.md instructions.
- Be honest when a review catches a genuine issue — do not dismiss valid feedback.
- If no review comments are found, inform the user.
