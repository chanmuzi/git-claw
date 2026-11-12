# AGENTS.md

이 저장소에서 작업하는 코딩 에이전트(Codex CLI, Claude Code 등)를 위한 프로젝트 지침이다.

## 프로젝트 문서 체계

- 이 `AGENTS.md`와 여기서 연결한 `docs/` 문서가 프로젝트 문서 정본(SSoT)이다. `AGENTS.md`에는 어느 작업에나 적용되는 규칙과 작업별 문서 표만 두고, 특정 작업에서만 필요한 절차와 상세는 `docs/` 문서에 둔다.
- 이 파일은 32,768바이트 이하로 유지한다 (Codex의 프로젝트 지침 기본 상한. 넘는 부분은 Codex에 전달되지 않는다). 확인: `[ "$(wc -c < AGENTS.md)" -le 32768 ] || echo "AGENTS.md가 32KiB를 넘음"`
- `docs/` 문서는 `@` import로 연결하지 않는다 (import한 파일은 세션 시작 때 전부 로드된다). 경로만 적는다.
- 루트 `CLAUDE.md`는 `@AGENTS.md` 한 줄 adapter다. 공유 내용을 복제하지 않는다.
- 에이전트 세션은 저장소 루트에서 시작한다.
- 배경과 근거: docs/decisions/2026-07-project-doc-adapter.md, docs/decisions/2026-11-doc-routing.md

## 프로젝트 개요

Git commit, PR, review 관련 convention을 강제하는 Agent Skill.
`/commit`, `/pr`, `/pr release`, `/issue`, `/review-reply`, `/code-review`, `/handoff`, `/explain-diff`, `/micro-world`, `/visual-doc`, `/eli5` 열한 가지 slash command를 제공한다.
Agent Skills 오픈 표준(agentskills.io)을 따르며, Claude Code 외에도 Codex CLI, Gemini CLI, Cursor 등 40+ agent에서 사용 가능.

## 아키텍처

Claude Code skill 표준 구조(`anthropics/skills` 기준)를 따른다.

```
git-claw/
├── .claude-plugin/marketplace.json   # marketplace 등록 메타데이터
├── skills/
│   ├── commit/
│   │   ├── SKILL.md                  # /commit
│   │   └── LICENSE.txt
│   ├── pr/
│   │   ├── SKILL.md                  # /pr, /pr release
│   │   └── LICENSE.txt
│   ├── issue/
│   │   ├── SKILL.md                  # /issue [bug|feature|...]
│   │   └── LICENSE.txt
│   ├── review-reply/
│   │   ├── SKILL.md                  # /review-reply
│   │   └── LICENSE.txt
│   ├── code-review/
│   │   ├── SKILL.md                  # /code-review
│   │   └── LICENSE.txt
│   ├── handoff/
│   │   ├── SKILL.md                  # /handoff
│   │   └── LICENSE.txt
│   ├── explain-diff/
│   │   ├── SKILL.md                  # /explain-diff
│   │   ├── template.html             # 산출물 디자인 시스템 템플릿 (스킬과 함께 배포)
│   │   ├── measure.html              # 렌더 측정 스크립트 (네 HTML 스킬 공통)
│   │   └── LICENSE.txt
│   ├── micro-world/
│   │   ├── SKILL.md                  # /micro-world
│   │   ├── template.html             # 셸 템플릿 (토큰과 프레임 고정, 본문은 bespoke)
│   │   ├── measure.html              # 렌더 측정 스크립트 (네 HTML 스킬 공통)
│   │   └── LICENSE.txt
│   ├── visual-doc/
│   │   ├── SKILL.md                  # /visual-doc
│   │   ├── components.html           # 컴포넌트 레퍼런스 팔레트 (토큰 고정, 골격은 조립)
│   │   ├── measure.html              # 렌더 측정 스크립트 (네 HTML 스킬 공통)
│   │   └── LICENSE.txt
│   └── eli5/
│       ├── SKILL.md                  # /eli5
│       ├── frame.html                # 프레임 + beat primitive (토큰 고정, 그림은 bespoke)
│       ├── list.html                 # 대량 목록과 검수 열 (목록이 있는 페이지에서만 읽음)
│       ├── measure.html              # 렌더 측정 스크립트 (네 HTML 스킬 공통)
│       └── LICENSE.txt
├── docs/
│   ├── testing.md, skill-rules.md, shared-logic.md, terminal-rendering.md, release.md, doc-review.md   # 작업별 문서
│   ├── ops/                          # main push 후 동기화, Skills CLI 운영
│   ├── decisions/                    # 결정 기록 (README.md가 목록)
│   └── changelog/                    # v1.8.0 이전의 날짜별 변경 기록
├── .gitignore
├── README.md
└── LICENSE
```

- 각 SKILL.md는 YAML frontmatter(`name`, `description`)로 시작
- SKILL.md body에서 Claude가 직접 git/gh 명령을 실행하도록 지시
- `$ARGUMENTS`로 사용자 인자를 참조

## 배포

두 가지 설치 방식을 지원한다:

- **Skills CLI (크로스 플랫폼)**: `npx skills add chanmuzi/git-claw`
- **Claude Code Plugin**: `/plugin marketplace add chanmuzi/git-claw`

## 작업별 문서

아래 작업을 시작하기 전에 해당 문서를 먼저 읽는다. 여기 없는 작업은 이 파일만으로 충분하다.

| 이 작업을 할 때 | 먼저 읽을 문서 |
|---|---|
| 스킬의 SKILL.md나 템플릿 수정, 새 스킬 추가 | docs/skill-rules.md, docs/shared-logic.md |
| 터미널 출력 템플릿 작성 | docs/terminal-rendering.md |
| 스킬 테스트 (로컬, 브랜치) | docs/testing.md |
| main push 뒤 플러그인과 Codex 동기화, 스킬 rename과 제거 | docs/ops/plugin-sync.md |
| `npx skills` 명령 실행 | docs/ops/skills-cli.md |
| 릴리스 | docs/release.md |
| 규칙의 이유, 과거 개정 확인, 결정 문서 작성 | docs/decisions/README.md |
| v1.8.0 이전의 변경 기록 확인 | docs/changelog/2026-02-to-04.md |
| 기록 문서가 길어졌거나 찾기 어려울 때, 문서 구조 점검 요청 | docs/doc-review.md |

## Git Convention (이 프로젝트 자체에 적용)

- Commit: `{type}: {한글 설명}` (소문자 prefix)
- Type: feat, fix, refactor, style, docs, test, perf, chore, hotfix
- 파일 staging 시 `git add -A` 금지, 개별 파일 지정
- 커밋 승인은 세션의 tool permission 설정에 따름
- PR merge 시 squash 금지: 커밋 히스토리를 보존하여 agent/reviewer 추적성 유지
- main push 후에는 플러그인과 Codex를 동기화한다. 절차: docs/ops/plugin-sync.md

## 언어

문서, 주석, commit 메시지 등은 한글 중심으로 작성. 고유 기술 용어(plugin, frontmatter, slash command 등)는 영어 유지.
SKILL.md 내용은 Claude가 읽는 지시문이므로 영어로 작성. 예외는 산출물의 문장을 정하는 절(eli5의 Step 2 storyboard와 "제목은 그림이 무엇인지 말한다", visual-doc의 "구성안은 산출물 언어로", explain-diff "Section titles"의 첫 문단, micro-world의 제목 줄)이다. 이 절들은 산출물 언어(한글)로 쓴다. 영어 역할 이름을 모델이 한글로 옮기는 단계에서 `왜 ~인가`, `~하는 법` 같은 번역투 제목이 생기기 때문이다. 배경과 근거: docs/decisions/2026-10-heading-hierarchy.md

## Change Log

[CHANGELOG.md](./CHANGELOG.md) 참조.
