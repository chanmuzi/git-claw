---
name: handoff
description: >-
  Generate a copy-ready handoff prompt for transferring work context to a new session.
  TRIGGER when: user asks to hand off work, create a handoff prompt, transfer context, wrap up session, prepare for next session (e.g., "handoff 해줘", "다음 세션으로 넘겨줘", "작업 이관해줘", "handoff prompt 만들어줘").
  DO NOT TRIGGER when: user is committing, creating PRs, reviewing code, or performing other git operations without handoff intent.
version: "1.1.0"
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

## The Handoff Is an Order, Not a Briefing

The output is a **prompt the user pastes to put the next agent to work immediately**. It is not a status report the user then has to act on by writing a second message. If the user has to follow the handoff with "so go do it," the handoff failed.

Two consequences for how you write it:

**Write it as the user speaking to the agent.** Use the user's own conversational register, the way they actually talk in this session — plain spoken Korean (`~해줘`, `~부터 시작하면 돼`, `~는 이미 해놨어`), not spec prose (`~할 것`, `~를 수행한다`). You are drafting the message the user would have typed themselves, so it should sound like them, not like a requirements document.

**Every handoff ends in an action.** Even when the direction is genuinely undecided, the directive still commands: *"먼저 X 훑어보고 어떻게 갈지 정리해서 알려줘"* — investigate and report is an action. What is never acceptable is a directive-free summary that leaves the agent waiting for instructions.

## Build the Handoff

Three parts, in order. Nothing else.

**Directive** — 1~3 lines opening the prompt, in the user's spoken voice. Name the branch, say what to do first, and make it concrete enough to start on without asking a follow-up question. Mention uncommitted changes here if there are any.

**맥락** — 0~4 bullets. Only decisions, reasons, dead ends, and constraints — still in the user's voice ("미들웨어로 하려다 접었어. SSR에서 쿠키를 못 읽더라"). If the session produced nothing that survives the One Principle, omit this section entirely. An empty 맥락 is a correct outcome, not a failure.

**참고** — paths, PR numbers, issue numbers. Only ones the next session will actually open to do its first action. Omit if there are none.

Hard limits: 15 lines total, 4 bullets in 맥락. If it does not fit, the extra content was recap.

### Skills and slash commands

Do NOT mention skills, slash commands, or plugins in the handoff by default — not `/commit`, not `/pr`, not third-party plugin commands. The next session picks its own tools, and it may not be running in the same agent or with the same plugins installed.

The single exception: the user explicitly asks for a skill in the handoff. In that case, pick only from the skills actually available in the current session, and weave the name into the directive so a copy-paste triggers it.

### Example

```markdown
feat/auth-refactor 브랜치에서 세션 만료 처리 이어서 해줘.
커밋 안 한 변경이 남아 있으니 git status부터 확인하고 시작하면 돼.

**맥락**
- 토큰 갱신은 미들웨어 말고 클라이언트 인터셉터에서 처리하기로 했어. 미들웨어로 하려다 접었는데, SSR 경로에서 쿠키를 못 읽더라
- 리프레시 토큰 회전은 이번 스코프에서 빼기로 했으니 건드리지 마

**참고**
- docs/auth-design.md
- #128
```

Read that as the user's own message: it tells the agent to start, in the voice the user actually uses. The 맥락 bullets carry things no file states — a rejected approach with the reason it failed, and a scope boundary. The 참고 lines are paths, not summaries.

Notice what is absent: no file list, no commit log, no recap of what this session accomplished, no skill names, and no "상황을 파악하라"-style preamble that would leave the agent idling instead of working.

## Output

Show the handoff wrapped in a fenced ` ```markdown ` block so `/copy` can pick it up as a single selectable item. Put nothing above it except, at most, one line noting a referenced artifact if one was found.

Do not append a confirmation question — the block speaks for itself, and the user will ask if they want changes. When `-y` is passed, output with no preamble at all.

**Constraints:**

- Create no files. The terminal output IS the deliverable.
- Do not execute the next action. Only describe it.
- Do not touch git state. This skill is strictly read-only.
