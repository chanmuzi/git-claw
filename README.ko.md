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
<p align="center">일관된 Git 워크플로우를 위한 Agent Skill</p>

<p align="center">
  <img src="https://img.shields.io/badge/skills-9-8B5CF6?logo=git&logoColor=white" alt="Skills" />
  <a href="https://agentskills.io"><img src="https://img.shields.io/badge/Agent_Skills-compatible-0EA5E9?logo=robotframework&logoColor=white" alt="Agent Skills" /></a>
  <img src="https://img.shields.io/github/license/chanmuzi/git-claw?color=blue" alt="License" />
  <img src="https://img.shields.io/github/last-commit/chanmuzi/git-claw?color=orange" alt="Last Commit" />
</p>

---

사람에게도, 에이전트에게도 읽기 좋은 Git 히스토리.

git-claw는 커밋, PR, 이슈, 코드 리뷰의 포맷을 하나로 통일하는 [Agent Skill](https://agentskills.io)입니다. 한 번 설치하면 모든 협업 산출물이 같은 컨벤션을 따릅니다 — 혼자 작업할 때도, 팀으로 협업할 때도.

- **읽기 좋은 히스토리** — 구조화된 커밋, PR, 이슈로 누구든 (어떤 에이전트든) 한눈에 맥락을 파악
- **멀티 모델 코드 리뷰** — 도메인 전문 에이전트 + Codex adversarial 분석을 교차 검증하여 severity 기반으로 정렬
- **세션 연속성** — `/handoff`로 작업 컨텍스트를 캡처해 다음 세션에서 바로 이어 작업

---

### 스킬

| | Skill | 설명 |
|:---:|---|---|
| 📝 | `/commit` | diff 기반 conventional commit 생성 |
| 🔀 | `/pr` | 구조화된 PR 생성 및 자동 labeling |
| 📋 | `/issue` | 템플릿 기반 이슈 생성 및 우선순위 labeling |
| 💬 | `/review-reply` | 리뷰 코멘트 분석 및 답글 |
| 🔍 | `/code-review` | 멀티 에이전트 severity 기반 코드 리뷰 |
| 🤝 | `/handoff` | 세션 이관 프롬프트 생성 |
| 📖 | `/explain-diff` | diff 이해용 인터랙티브 HTML 설명 문서 생성 |
| 🧪 | `/micro-world` | 동작을 만져볼 수 있는 일회성 인터랙티브 시뮬 생성 |
| 📄 | `/visual-doc` | 조사·분석 내용을 온-브랜드 인터랙티브 문서로 생성 |

## 설치

### Claude Code Plugin (권장)

```bash
# marketplace 추가
/plugin marketplace add chanmuzi/git-claw

# plugin 설치
/plugin install git-claw@git-claw
```

### Skills CLI

```bash
npx skills add chanmuzi/git-claw
```

대화형 installer에서 스킬, 대상 agent, 범위(project/global), 설치 방식을 선택할 수 있습니다.

<details>
<summary>프롬프트 없이 한번에 설치</summary>

```bash
npx skills add chanmuzi/git-claw --skill commit --skill pr --skill issue --skill review-reply --skill code-review --skill handoff --skill explain-diff --skill micro-world --skill visual-doc -g
```

</details>

### 업데이트

**Claude Code Plugin:**

```bash
/plugin update git-claw@git-claw
```

또는 auto-update 활성화: `/plugin` → **Marketplaces** 탭 → marketplace 선택 → **Enable auto-update**

**Skills CLI:**

```bash
npx skills check    # 업데이트 확인
npx skills update   # 전체 업데이트
```

> Symlink(권장)으로 설치한 경우 한 번의 업데이트로 모든 agent에 즉시 반영됩니다. Copy로 설치한 경우 각 복사본을 개별 업데이트해야 합니다.

> **Skill rename은 자동으로 추적되지 않습니다.** upstream에서 skill이 rename되면 Skills CLI는 옛 디렉터리를 그대로 남겨둡니다. 동일한 description을 가진 skill이 두 개로 보이거나, 옛 이름으로 노출될 수 있습니다. 옛 폴더를 수동으로 제거하세요 — **설치한 scope과 동일한 scope으로 제거해야 합니다**:
> ```bash
> # Global 설치 (npx skills add ... -g)  →  ~/.codex/skills/{old-name}
> npx skills remove {old-name} -g     # 또는
> rm -rf ~/.codex/skills/{old-name}
>
> # Project 설치 (-g 없이 설치)  →  ./.codex/skills/{old-name}
> npx skills remove {old-name}        # 또는
> rm -rf ./.codex/skills/{old-name}
> ```
> 이후 Codex CLI 세션을 재시작해야 skill 목록이 갱신됩니다.

## 스킬

> **Agent별 호출 prefix:** Claude Code는 `/` (예: `/commit`), Codex CLI는 `$` (예: `$commit`)를 사용합니다.

### `/commit` — Git 커밋 생성

staged/unstaged 변경 사항을 분석하고 conventional commit 메시지를 제안합니다. staging 전에 변경 파일을 의도(intent)별로 그룹핑합니다 — 인프라, agent 설정, 앱 코드, 빌드 도구, 문서, 테스트. 여러 의도가 섞인 변경은 자동으로 별도 commit으로 분리되며, 강하게 의존하는 변경(스키마+코드, 시그니처+호출처, 코드+검증 테스트)은 same-intent exception으로 같은 commit에 유지됩니다. 메시지에는 em-dash·en-dash를 쓰지 않아 AI 생성 흔적을 남기지 않습니다.

```
/commit              # 분석 후 커밋
/commit --amend      # 직전 커밋에 변경사항 합치기
```

타입: `feat` · `fix` · `refactor` · `style` · `docs` · `test` · `perf` · `chore` · `hotfix`

### `/pr` — Pull Request 생성

구조화된 템플릿으로 PR을 생성합니다. Individual PR과 Release PR 모드를 자동으로 구분합니다. 제목·본문에는 em-dash·en-dash를 쓰지 않아 AI 생성 흔적을 남기지 않습니다.

```
/pr              # Individual PR (Feat, Fix, Refactor 등)
/pr -g           # Mermaid 변경 흐름 그래프 포함
/pr release      # Release PR (dev → main 통합)
```

Individual PR을 만든 뒤, 이 변경의 이해용 설명 문서(`/explain-diff`)를 만들지 한 줄로 물어봅니다 (기본 No, 재질문 없음).

### `/issue` — GitHub 이슈 생성

구조화된 템플릿으로 이슈를 생성하고, type/priority label을 자동으로 부여합니다.

```
/issue             # 일반 이슈 (컨텍스트에서 유형 추론)
/issue bug         # Bug Report
/issue feature     # Feature Request
```

### `/review-reply` — PR 리뷰 분석 및 답글

PR의 리뷰 코멘트(CodeRabbit, Copilot, 팀원 등)를 수집하고, 실제 코드 대비 유효성을 분석한 뒤 결과를 논의합니다. 각 항목은 근거가 되는 원문을 그대로 인용하고 수정안을 diff로 보여주므로, 파일을 직접 열지 않고도 수용/기각을 판단할 수 있습니다. 기각 항목도 리뷰어의 주장을 반박하는 코드를 인용합니다.

```
/review-reply          # 현재 브랜치의 PR 리뷰
/review-reply 42       # PR #42 리뷰
```

### `/code-review` — Context-Aware 코드 리뷰

PR 또는 로컬 코드 변경사항을 도메인별 전문 agent(Security, Performance, Architecture, Domain Logic)로 병렬 분석합니다. PR 모드에서는 PR의 목적을 리뷰의 핵심 렌즈로 활용하여 불완전한 구현, 일관성 누락, 목적 불일치를 감지합니다. diff 외부 finding에 대해 인과관계 필터를 적용하여 PR과 무관한 기존 이슈를 자동 제외합니다. 이어서 **Reporting Bar**가 "참인 finding"과 "보고할 finding"을 분리합니다. 살아남은 각 finding에 severity와 독립된 판정을 부여하여, `ship-blocker`와 `in-scope-gap`만 전문 보고하고, 선행 이슈는 `밖으로 미룸`에 한 줄로만 남기며 인라인 코멘트로 달지 않고, 취향 수준의 관측은 건수로만 집계합니다. 잘라낸 것은 `기각 {n}건` 한 줄로 드러나므로, 짧은 리뷰가 누락이 아니라 판단으로 읽힙니다. severity 기반 구조화된 리뷰를 생성합니다 (🔴 Critical · 🟡 Warning · 🟢 Info). 모든 finding은 문제가 되는 코드를 그대로 인용하고 수정안을 diff로 제시하므로, 파일을 열어 무엇이 문제였는지 되짚지 않아도 리포트만으로 판단할 수 있습니다. 변경사항이 없으면 자동으로 현재 작업 디렉토리 리뷰로 전환하며, 대화 맥락을 반영하여 최적의 리뷰 범위를 결정합니다.

```
/code-review           # PR 자동 감지 → 변경사항 리뷰 → 현재 디렉토리 리뷰 (대화 맥락 반영)
/code-review 42        # PR #42 코드 리뷰
/code-review src/auth/ # 특정 경로 코드 리뷰
```

PR 모드에서 findings는 GitHub Review API를 통해 diff의 특정 라인에 inline comment로 게시됩니다.

<details>
<summary>플래그</summary>

- `--wd` — PR 자동 감지를 건너뛰고 working directory 모드 강제
- `--domain security,perf` — 자동 감지 대신 도메인 수동 지정
- `-y` / `-f` — 승인 없이 즉시 게시
- `-g` — Mermaid 변경 흐름 그래프 생성 (PR 모드 전용)
- `-q` / `--quick` — Quick 모드: 단일 패스, 도메인 최대 2개, Critical/Warning 우선
- `--full-scan` — diff 외부 pre-existing issue도 General Findings에 포함 (PR 모드 전용)
- `-a` / `--all` — Reporting Bar를 건너뛰고 confirmed finding 전체 보고 (기각분 포함, 감사 목적)
- `--no-codex` — Codex 통합 비활성화
- `--codex-both` — Codex 일반 리뷰 + adversarial 동시 실행

</details>

<details>
<summary>Codex 통합 (선택 사항)</summary>

[Codex 플러그인](https://github.com/openai/codex) 설치 및 인증(`claude plugin add codex` + `!codex setup`) 환경에서 `/code-review` 실행 시 Codex adversarial review가 자동 병렬 실행됩니다. findings는 교차 검증 후 출처 태그와 함께 통합됩니다. `--no-codex`로 비활성화 가능합니다.

</details>

### `/handoff` — 세션 Handoff 프롬프트

현재 세션의 작업 컨텍스트를 다음 세션으로 이관하기 위한 copy-ready handoff 프롬프트를 생성합니다. 단 하나의 원칙 위에 서 있습니다: **다음 세션이 저장소를 읽어서 알아낼 수 있는 것은 쓰지 않는다.** 다음 세션은 코드를 스스로 탐색하는 에이전트이므로, handoff에는 저장소에 남지 않는 것만 담습니다 — 내린 결정과 그 이유, 시도했다 버린 접근, 사용자가 정한 제약, 그리고 첫 행동. 나머지는 경로로만 참조합니다.

```
/handoff                   # 자동 감지 후 handoff 프롬프트 생성
/handoff -y                # 확인 없이 즉시 출력
/handoff auth 리팩토링      # 특정 주제에 집중한 handoff 생성
```

<details>
<summary>상세</summary>

**브리핑이 아니라 지시:** 붙여넣는 순간 다음 에이전트가 바로 작업을 시작하는 프롬프트이며, 사용자가 평소 말하는 구어체로 작성됩니다. 읽고 나서 다시 "그럼 이거 해줘"라고 시켜야 하는 상태 요약이 아닙니다. 모든 handoff는 행동으로 끝납니다 — 방향이 미정이어도 "먼저 X 훑어보고 어떻게 갈지 알려줘"라는 지시가 됩니다.

**구조:** directive(어느 브랜치에서 무엇부터 할지) + 맥락 불릿 최대 4개 + 참고 경로. 전체 15줄 상한. 쓸 내용이 없는 섹션은 통째로 생략합니다 — 맥락이 비는 것도 정상적인 결과입니다.

**Git:** 브랜치 이름과 커밋되지 않은 변경의 유무만. commit log, diff stat, 파일 목록은 넣지 않습니다. 다음 세션이 `git status` 한 번이면 확인할 수 있기 때문입니다.

**스킬 이름 미언급:** 명시적으로 요청하지 않는 한 slash command나 플러그인 스킬을 추천하지 않습니다. 다음 세션은 다른 에이전트에서, 다른 플러그인 구성으로 실행될 수 있으므로 도구는 스스로 고릅니다.

**출력:** `/copy`에 최적화된 터미널 텍스트. 파일 내용을 반복하지 않고 경로만 참조합니다.

</details>

### `/explain-diff` — Diff 이해하기

diff, 커밋, 브랜치, PR을 위한 self-contained 인터랙티브 HTML 설명 문서를 생성합니다 — 공유하거나 머지하기 전에 변경을 진짜로 이해할 수 있도록. Geoffrey Litt의 explain-diff에서 영감을 받았습니다: 에이전트가 짠 코드의 병목은 정확성이 아니라 사람의 이해입니다. 문서는 변경을 literate diff(산문 + 원문 그대로의 코드 발췌)로 풀어내고, 건드린 모든 경로를 주석 달린 디렉토리 트리로 보여주며, 5문항 퀴즈로 끝납니다. 오답이면 정답 대신 관련 섹션을 가리키는 힌트가 나오며, 이 힌트는 해당 섹션으로 바로 가는 링크입니다 (돌아올 때는 "퀴즈로 돌아가기" 칩). 완료 게이트는 5/5에서만 열립니다.

```
/explain-diff          # working diff 설명 문서 생성
/explain-diff 42       # PR #42 설명 문서 생성
/explain-diff src/     # 특정 경로로 diff 범위 제한
```

<details>
<summary>상세</summary>

**조사 우선:** 문서를 쓰기 전에 모든 hunk와 그걸 감싸는 함수를 읽고, 커밋/PR 본문에서 명시된 의도를 추출하고, 호출자와 테스트를 탐색한 뒤 hunk를 서사적 테마로 묶습니다 — 문서는 라인이 아니라 동작을 설명합니다.

**3파트 구조:** 개요(배경 + 디렉토리 트리 + 선택적 인터랙티브 피규어), 변경 사항(테마별 카드, 각 카드는 핵심 정리로 마무리), 이해 점검(리스크 포인터 + 퀴즈).

**정직한 설계:** 의심은 확정된 버그가 아닌 주의 포인터로만 제시하고, 코드 인용은 원문 그대로만 하며, 완료 버튼은 이해 상태만 표현합니다 — 실제로 수행하지 않는 액션을 흉내 내지 않습니다.

**자연스러운 한국어:** 제목은 문장형이 아니라 명사형/개조식으로 짧게 쓰고(한글 `word-break: keep-all`로 어절 단위 개행), 대시(—)와 억지 한자어·번역투를 피합니다. AI가 쓴 티를 남기는 표현을 걷어냅니다.

**출력:** 외부 의존성 없는(CDN·웹폰트 금지) 단일 HTML 파일을 저장소 루트에 `explain-diff-<slug>.html`로 저장하고 절대경로를 안내합니다. 자동으로 열거나 커밋하지 않습니다.

</details>

### `/micro-world` — 동작 안에 들어가 보기

특정 동작(상태 머신, 알고리즘, 데이터 변환, 프로토콜 흐름)을 직접 조작하며 체감할 수 있는 일회성 인터랙티브 HTML 시뮬레이션을 만듭니다 (Seymour Papert의 "Mathland" 개념). `/explain-diff`의 형제 스킬입니다: 그쪽이 일관된 구조의 문서를 만든다면, 이쪽은 시각 언어만 공유하는 주문제작 시뮬을 만듭니다. 적용성 게이트가 있어 시뮬로 얻는 게 없는 변경(설정, rename, 의존성 범프)에는 정직하게 거절하고 `/explain-diff`를 권합니다. 시뮬은 실제 코드 로직을 모델링하며, 단순화한 지점은 각주로 고지됩니다. 각 월드는 가이드 시나리오 하나로 돌아갑니다: 미션 바의 목표 칩, 매 순간 권장 액션 하나만 활성화, 그리고 스냅샷 기반 재생 컨트롤(→ 키만 연타해도 전체 스토리를 완주, ←로 되감기, R로 재시작).

```
/micro-world                          # 현재 변경에서 동작을 골라 시뮬 생성
/micro-world "api.ts의 재시도 로직"     # 설명한 동작을 시뮬로 생성
```

### `/visual-doc` — 내용을 온-브랜드 문서로

조사·분석한 내용을 self-contained 인터랙티브 HTML 문서로 만듭니다: 리서치 보고서, 레포·코드 분석, 개념 설명, 비교/평가, 디자인 정리 등. `/explain-diff`·`/micro-world`의 형제 스킬입니다 (디자인 토큰은 공유, 구조는 공유하지 않음). `/explain-diff`와 달리 **퀴즈도 이해 게이트도 없습니다**: 전달이 목적이지 검증이 아닙니다. 두 층 구조로 만듭니다: 잠금층(토큰·primitive·말투)이 모든 문서를 한 가족으로 묶고, 생성층은 콘텐츠 유형·규모에 맞춰 컴포넌트 팔레트(`components.html`)에서 골라 골격을 조립합니다. 팔레트에 없는 블록이 필요하면 **같은 토큰으로 새로 만듭니다** (팔레트는 완결된 목록이 아니라 예시집). 긴 문서는 분량에 맞게 네비게이션을 고릅니다: 상단 진행바 + 플로팅 TOC, 슬림 챕터 페이저, 또는 접이식 섹션. 라이트·다크 모두 지원합니다.

```
/visual-doc              # 현재 대화의 내용을 문서로
/visual-doc src/         # 특정 경로를 분석해 문서로
/visual-doc 42           # PR #42를 분석해 문서로
```

**출력:** 외부 의존성 없는(CDN·웹폰트 금지) 단일 HTML 파일을 저장소 루트에 `visual-doc-<slug>.html`로 저장하고 절대경로를 안내합니다. 자동으로 열거나 커밋하지 않습니다.

## 언어 동작

모든 커맨드의 출력(커밋 메시지, PR 제목/본문, 이슈 제목/본문)은 프로젝트의 `AGENTS.md`(없으면 `CLAUDE.md`)에 설정된 언어로 작성됩니다. 설정이 없으면 사용자의 대화 언어를 따릅니다. 기술 용어는 원어 그대로 유지합니다.

## Convention 요약

| 항목 | 형식 | 예시 |
|------|------|------|
| Commit | `{type}: {설명}` (소문자) | `feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Branch | `{type}/{kebab-case}` (영어) | `feat/multiturn-context-persistence` |
| PR 제목 | `{Type}: {설명}` (대문자) | `Feat: 멀티턴 컨텍스트 유지 기능 추가` |
| Issue 제목 | `{설명}` (type prefix 없음) | `E2E 실험에서 non-default retrieval index snapshot 지원` |
| Release PR | `Release: dev → main 통합 (vX.Y.Z)` | `Release: dev → main 통합 (v0.4.1)` |

## CLAUDE.md 연동

글로벌 `~/.claude/CLAUDE.md`에 아래 내용을 추가하면 convention을 참조할 수 있습니다:

```markdown
## Git Conventions
- Commit: `{type}: {description}` (소문자 prefix: feat, fix, refactor, style, docs, test, perf, chore, hotfix)
- Branch: `{type}/{english-kebab-case}` (feat/, fix/, refactor/, docs/, hotfix/)
- PR title: `{Type}: {description}` (대문자 prefix: Feat, Fix, Refactor, Perf 등)
- Release PR: `Release: dev → main 통합 (vX.Y.Z)`
- `/commit`, `/pr`, `/pr release`, `/issue`, `/review-reply`, `/code-review`, `/handoff`, `/explain-diff`, `/micro-world`, `/visual-doc` 커맨드로 전체 워크플로우 실행
```

## Label 시스템

`/pr`과 `/issue`는 통일된 label 체계를 공유합니다. Label은 최초 사용 시 자동 생성되며, 이미 존재하면 그대로 유지됩니다.

<details>
<summary>Label 목록</summary>

| Label | Color | 출처 |
|-------|-------|------|
| `bug` | 🔴 `d73a4a` | GitHub 기본 |
| `feature` | 🔵 `0075ca` | GitHub 기본 |
| `enhancement` | 🩵 `a2eeef` | GitHub 기본 |
| `docs` | 🟣 `5319e7` | GitHub 기본 |
| `chore` | 🟡 `e4e669` | 표준 |
| `refactor` | 🟪 `d4c5f9` | 표준 |
| `style` | 🩵 `c5def5` | 표준 |
| `test` | 🟢 `bfd4f2` | 표준 |
| `perf` | 🟠 `f9d0c4` | 표준 |
| `hotfix` | 🔴 `b60205` | 표준 |
| `release` | 🔵 `1d76db` | 표준 |
| `critical` | 🔴 `b60205` | 우선순위 |
| `high` | 🟠 `d93f0b` | 우선순위 |
| `medium` | 🟡 `fbca04` | 우선순위 |
| `low` | 🟢 `0e8a16` | 우선순위 |

- **Namespace 접두사 없음** — 깔끔하고 간결한 이름 (`type: bug` 대신 `bug`)
- **GitHub 표준 색상** — GitHub 기본 palette과 동일한 색상 사용
- **충돌 안전** — `gh label create ... 2>/dev/null || true` (없으면 생성, 있으면 스킵)

</details>

## 라이선스

MIT
