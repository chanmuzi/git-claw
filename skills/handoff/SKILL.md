---
name: handoff
description: >-
  Generate a copy-ready handoff prompt for transferring work context to a new session.
  TRIGGER when: user asks to hand off work, create a handoff prompt, transfer context, wrap up session, prepare for next session (e.g., "handoff 해줘", "다음 세션으로 넘겨줘", "작업 이관해줘", "handoff prompt 만들어줘").
  DO NOT TRIGGER when: user is committing, creating PRs, reviewing code, or performing other git operations without handoff intent.
version: "1.0.0"
allowed-tools: Bash(git *), Read, Glob, Grep
---

## The One Principle

**Write only what the next session cannot recover by reading the repo.**

The next session is a capable agent with full access to the code, the docs, the git history, and the GitHub API. It will explore on its own. Anything it can find by reading a file is noise in the handoff — it costs tokens, and worse, a stale restatement of a file competes with the file itself.

What it *cannot* recover by reading:

- **Decisions and their reasons** — the code shows what was chosen, never why, nor what was rejected.
- **Dead ends** — an approach that was tried and abandoned leaves no trace in the repo, so the next session will happily retry it.
- **Constraints stated by the user** — preferences, scope boundaries, "don't touch X".
- **The next action** — what to do first, which the repo cannot tell it.

That is the entire content of a handoff. Everything else is a session recap, and a session recap is not a handoff.

Test every line before writing it: *"Could the next session learn this by opening a file?"* If yes, delete it and reference the path instead.

## Parse Arguments

| Argument | Type | Description |
|----------|------|-------------|
| `topic` | Optional string | Focus filter — scope the handoff to this topic only (e.g., `/handoff auth 리팩토링`) |
| `-y` | Optional flag | Skip the draft step and output the handoff directly |

If `$ARGUMENTS` contains `-y`, treat it as the skip flag; the remaining text is the topic filter. If `$ARGUMENTS` is empty, scope the handoff to whatever the session was actually working on.

## Gather Context

**1. The conversation.** This is the only real source. Identify, from the next session's perspective:

- What it should do first
- What was decided, and why
- What was tried and rejected
- What constraints it must respect

If a topic filter was given, drop everything outside it.

**2. Git — branch name only.**

```bash
git branch --show-current
git status --short
```

From this, take exactly two facts: the branch name, and whether uncommitted changes exist. Do NOT run `git log`, `git diff --stat`, or `git stash list`, and do NOT list changed files — the next session runs `git status` itself in one command. The handoff says *that* there is uncommitted work, never *what* it is.

**3. Referenced paths.** If the session created or worked against a spec, plan, design doc, or issue, note its path or number. Verify the path exists before citing it.

- Reference by path only. Never quote, summarize, or paraphrase the content of a file into the handoff.
- Do NOT go hunting for documents the session never touched. No speculative `Glob` over spec/plan directories, no scanning for artifacts that "might be relevant". If the conversation did not mention it, it does not belong in the handoff.

## Build the Handoff

Three parts, in order. Nothing else.

**Directive** — 1~3 lines opening the prompt. What to do first, concretely enough to act on. If the work is exploratory, say what to investigate and what the output should be. Include the branch here, plus the uncommitted-changes fact if there is any.

**맥락** — 0~4 bullets. Only decisions, reasons, dead ends, and constraints. If the session produced nothing that survives the One Principle, omit this section entirely. An empty 맥락 is a correct outcome, not a failure.

**참고** — paths, PR numbers, issue numbers. Only ones the next session will actually open to do its first action. Omit if there are none.

Hard limits: 15 lines total, 4 bullets in 맥락. If it does not fit, the extra content was recap.

### Skills and slash commands

Do NOT mention skills, slash commands, or plugins in the handoff by default — not `/commit`, not `/pr`, not third-party plugin commands. The next session picks its own tools, and it may not be running in the same agent or with the same plugins installed.

The single exception: the user explicitly asks for a skill in the handoff. In that case, pick only from the skills actually available in the current session, and weave the name into the directive so a copy-paste triggers it.

### Example

```markdown
feat/auth-refactor 브랜치에서 세션 만료 처리 구현을 이어서 진행할 것.
커밋되지 않은 변경이 있으니 git status로 먼저 확인.

**맥락**
- 토큰 갱신은 미들웨어가 아니라 클라이언트 인터셉터에서 처리하기로 결정. 미들웨어 방식은 SSR 경로에서 쿠키를 못 읽어 폐기함
- 리프레시 토큰 회전(rotation)은 이번 스코프에서 제외 (사용자 요청)

**참고**
- docs/auth-design.md
- #128
```

The directive names the branch and the first action. The 맥락 bullets are things no file states: a rejected approach with its reason, and a scope boundary. The 참고 lines are paths, not summaries. Notice what is absent — no file list, no commit log, no recap of what this session accomplished, no skill names.

## Output

Show the handoff wrapped in a fenced ` ```markdown ` block so `/copy` can pick it up as a single selectable item. Put nothing above it except, at most, one line noting a referenced artifact if one was found.

Do not append a confirmation question — the block speaks for itself, and the user will ask if they want changes. When `-y` is passed, output with no preamble at all.

**Constraints:**

- Create no files. The terminal output IS the deliverable.
- Do not execute the next action. Only describe it.
- Do not touch git state. This skill is strictly read-only.
