# 스킬 간 공유 로직

> 이 문서는 `AGENTS.md`의 작업별 문서 표에서 연결된다. 스킬의 SKILL.md나 템플릿을 고치기 전에 읽는다.

일부 로직이 여러 SKILL.md에 동일하게 기술되어 있다. 각 스킬은 self-contained여야 하므로 중복은 의도된 설계이나, 한쪽 수정 시 반드시 cross-check해야 한다.

고치려는 내용이 아래 표에 있으면 관련 스킬을 같이 고친다. 찾을 표식은 SKILL.md나 템플릿에서 그 규칙을 검색할 때 쓰는 문자열이고, 무엇이 같고 무엇이 다른지는 아래 "상세"의 같은 번호에 있다. 표의 행에는 값, 날짜, 개정 내용을 적지 않고 300자를 넘기지 않는다. 그런 글은 상세 절이나 결정 문서에 쓴다.

| # | 공유 로직 | 관련 스킬 | 찾을 표식 | 정본 |
|---|---|---|---|---|
| 1 | `-g`/`--graph` Mermaid 분석 | `code-review`, `pr` | `-g\|--graph` | 없음 |
| 2 | Label System | `pr`, `issue` | `## Label System` | 없음 |
| 3 | Evidence 블록 | `code-review`, `review-reply` | `Evidence rules` | 없음 |
| 4 | 프로젝트 설정 조회 | 전 스킬 | `AGENTS.md (or CLAUDE.md as fallback)` | 없음 |
| 5 | 공유 디자인 규칙 | `visual-doc`, `explain-diff`, `micro-world`, `eli5` | `Design rules (shared across` | docs/decisions/2026-07-html-shared-design-rules.md |
| 6 | 문장 규칙 | `visual-doc`, `explain-diff`, `micro-world`, `eli5` | `One name per thing` | docs/decisions/2026-07-html-shared-design-rules.md |
| 7 | 손으로 쓴 SVG 그림 규칙 | `eli5`, `visual-doc`, `explain-diff` | `Fit the text before you draw` | 없음 |
| 8 | 렌더 측정 | `eli5`, `visual-doc`, `explain-diff`, `micro-world` | `measure.html` | docs/decisions/2026-07-html-shared-design-rules.md |
| 9 | 렌더 검증 절차 | `eli5`, `visual-doc`, `explain-diff`, `micro-world` | `Render check (shared` | docs/decisions/2026-07-html-shared-design-rules.md |
| 10 | 코드 칩 줄바꿈 | `eli5`, `visual-doc`, `explain-diff`, `micro-world` | `inline code chip wrapping (SHARED` | docs/decisions/2026-07-html-shared-design-rules.md |
| 11 | 부가 텍스트 금지와 말투 | `eli5`, `visual-doc`, `explain-diff`, `micro-world` | `No redundant or meta text` | docs/decisions/2026-08-eli5.md |
| 12 | 제목 아래 줄 없음과 숫자 한 곳 | `eli5` (현재 단독) | `One number, one place` | docs/decisions/2026-08-eli5.md |
| 13 | 산문 원문 카드 | `eli5` (현재 단독) | `.prose-card` | docs/decisions/2026-08-eli5.md |
| 14 | 텍스트 스킬 글쓰기 규칙 | `commit`, `pr`, `issue`, `review-reply`, `code-review`, `handoff` | `Writing rules (shared across commit` | docs/decisions/2026-10-machine-writing-symbols.md, docs/decisions/2026-10-heading-hierarchy.md |
| 15 | 제목 규칙과 산출물 언어 구성안 | `eli5`, `visual-doc`, `explain-diff`, `micro-world` | `Headings are answers, not frames` | docs/decisions/2026-10-heading-hierarchy.md |
| 16 | 위치 표시 | `eli5`, `visual-doc`, `explain-diff`, `micro-world` | `section hierarchy (SHARED` | docs/decisions/2026-10-heading-hierarchy.md, docs/decisions/2026-08-eli5.md |
| 17 | 그림의 선 한 벌, 마무리 줄 배지, 그림 안의 글, 그림 안 제목, 해설 머리 표시 | `eli5` (현재 단독) | `One stroke set` | docs/decisions/2026-08-eli5.md |
| 18 | 조작할 수 있는 그림의 조건 | `eli5`, `visual-doc`, `explain-diff` | `Static first` | docs/decisions/2026-08-eli5.md |
| 19 | 근거 카드, 번호 배지, 캡션 칩, 해설 목록 | `eli5` (현재 단독) | `Evidence card` | docs/decisions/2026-08-eli5.md |
| 20 | 산출물 독자 구분 | `explain-diff`, `micro-world`, `visual-doc`, `eli5` | `DO NOT TRIGGER` | docs/decisions/2026-08-eli5.md |

## 상세

### 1. `-g`/`--graph` Mermaid 분석

- 범위: `-g`/`--graph` Mermaid 분석 (4단계 프로세스 + graph rules)
- 관련 스킬: `code-review`, `pr`
- 분석 절차, skip condition, Mermaid 포맷 동일

### 2. Label System

- 범위: Label System (type labels, color hex)
- 관련 스킬: `pr`, `issue`
- 각 SKILL.md에 inline 정의. Agent Skills 배포 독립성(`npx skills add chanmuzi/git-claw --skill issue` 등 개별 설치) 제약으로 중앙화하지 않음

### 3. Evidence 블록

- 범위: Evidence 블록 (verbatim 인용, diff 포맷, 12줄 상한, 양쪽 인용, 변경 불필요 시 인용)
- 관련 스킬: `code-review`, `review-reply`
- finding 제시 스키마 동일. 한쪽 문구 수정 시 다른 쪽 evidence rules도 함께 갱신

### 4. 프로젝트 설정 조회

- 범위: 프로젝트 설정 조회 (AGENTS.md 우선, 없으면 CLAUDE.md fallback)
- 관련 스킬: 전 스킬
- Branch Strategy, 언어, 릴리스, label 규칙 등 프로젝트 설정을 읽는 모든 지점에 동일 적용. adapter 체계(`@AGENTS.md` 한 줄 CLAUDE.md) 프로젝트에서 @ import를 해석하지 않는 호스트도 규칙을 읽도록 보장

### 5. 공유 디자인 규칙

- 범위: 공유 디자인 규칙 (제목 `word-break: keep-all`, 장식 그라데이션 금지, em-dash, en-dash, 가운뎃점 금지, 배경 대비 색상, 비교는 행으로 쌓고 좌우 배치는 렌더 측정을 통과할 때만, 자연스러운 한국어, 비유와 번역투 금지, 라이트/다크 전환 버튼, 코드 블록)
- 관련 스킬: `visual-doc`, `explain-diff`, `micro-world`, `eli5`
- HTML 산출물을 내는 네 스킬 공통. 한쪽 규칙 수정 시 나머지 세 스킬의 SKILL.md와 템플릿도 함께 갱신. 전환 버튼(`.theme-btn` CSS와 마크업, `<head>` 테마 스크립트)과 코드 블록 CSS(`code block (SHARED` 주석으로 시작하는 블록: `.codeblock`, `sx-*` 구문 강조, 가로 스크롤)는 네 템플릿에 같은 코드로 들어 있으므로 한 곳을 고치면 나머지도 맞춘다. explain-diff의 diff 줄 규칙(`.ln.add`/`.ln.del`/`.sign`)은 그 템플릿 전용이고, eli5 근거 카드의 diff 형태(`.ev .ln.add`/`.ln.del`/`.sign`)는 같은 색과 같은 뜻을 쓰되 선택자가 다르다. 다크 토큰은 "두 번 정의하고 두 블록을 같게 유지"하는 구조만 공통이고 값은 템플릿마다 의도적으로 다르다 (explain-diff, micro-world의 다크 `--gray25`와 `--surface`, `--on-accent`, `--track`, explain-diff의 버튼 고정 기준 폭 1000px). 다른 템플릿의 다크 블록을 통째로 복사해 맞추지 않는다. 정본: docs/decisions/2026-07-html-shared-design-rules.md

### 6. 문장 규칙

- 범위: 문장 규칙 (한 대상에 한 이름, 행위자를 주어로, 횟수는 숫자로, 붙인 이름은 풀이 뒤에만, 조사는 앞말에 붙여 쓰기, 수량 표현 묶기)
- 관련 스킬: `visual-doc`, `explain-diff`, `micro-world`, `eli5`
- 네 SKILL.md에 `One name per thing`으로 시작하는 항목과 `A particle is part of the word before it`으로 시작하는 항목이 들어 있다. 앞 항목은 micro-world만 적용 범위 문구(미션 칩, 액션 라벨, 해설, 세계 상태)가 다르고, 뒤 항목(영문, 숫자, 코드 뒤 조사 붙여 쓰기, 숫자+단위 붙여 쓰기, 수 관형사+단위 `&nbsp;`, 조사 앞 공백 grep 검사)은 네 파일이 같은 글이다. 한쪽 수정 시 나머지 셋도 갱신. 문단 `text-wrap:pretty`와 제목 `text-wrap:balance`도 네 템플릿에 공통이다 (선택자는 템플릿마다 다르다). 정본: docs/decisions/2026-07-html-shared-design-rules.md (개정 2026-10-03 2차, 3차)
- 붙인 이름 문장(`A coined name`)은 `One name per thing` 항목 끝에 네 파일 같은 글로 들어 있다. 검사(Render check 6 `Coined names`)는 eli5에만 있다. 배경: docs/decisions/2026-08-eli5-history.md (개정 2026-11-11 3차)

### 7. 손으로 쓴 SVG 그림 규칙

- 범위: 손으로 쓴 SVG 그림 규칙 (그리기 규칙 6항목, `Fit the text before you draw` 산술, `figure labels (SHARED` CSS 블록)
- 관련 스킬: `eli5`, `visual-doc`, `explain-diff`
- 산술 문단은 세 SKILL.md에 같은 글로 들어 있고 viewBox 상한과 그 상한에 맞춘 배치 계획 예시만 다르다 (eli5 840, visual-doc 750, explain-diff 720). 상한은 카드 안 그림의 실제 폭(782, 700, 668px) × 13/12 이하로 잡는다. `.dg-*` 라벨 CSS는 세 템플릿에 같은 이름으로 들어 있다 (두 상태 비교 그림용 선택 클래스 `.dg-pill`, `.dg-cap` 상태 캡션과 `.dg-q` 주인공 문자열 포함). 값은 2026-11-09 Lift 개정 뒤 eli5만 다르다(`.dg-s` 본문 글꼴 11.5px와 `.dg-s.code`, `.dg-q` 14px 600, `.dg-cap` 11.5px 700과 `.dg-pill`의 가는 테두리): visual-doc, explain-diff 블록을 eli5에 복사하지 않는다. 글자 폭 표의 `.dg-cap` 행도 eli5만 11.5px 실측값(한글 11.5, 숫자 7, 공백 3.5)이고 나머지 둘은 12px 행(한글 13, 영숫자 10, 공백 5)이다: CSS 크기를 바꾸면 표 행을 같이 바꾼다 (2026-11-09 캡션 칩 폭 사고). 글자 폭 표의 `.dg-t` 행과 4번 규칙의 `.dg-t` 세로 범위도 eli5에만 있다 (2026-11-11 2차): visual-doc, explain-diff의 산술 문단을 eli5에 덮어쓰지 않는다. `.fig` 컨테이너 CSS는 템플릿마다 다르다 (eli5는 카드 자체, 나머지는 카드 안). visual-doc의 mermaid는 손으로 배치하기 어려운 큰 다이어그램용 대체 수단이다

### 8. 렌더 측정

- 범위: 렌더 측정 (`measure.html`: `data-sbs` 2열 이상 grid 안 가로 넘침, `data-fit` 그림 라벨의 viewBox 이탈, 상자 경계 걸침, 라벨 겹침, 캡션 칩(`rect.dg-pill`)이 `.dg-cap` 글자 + 16보다 16px 넘게 넓음, `data-ovf` 테두리나 배경이 있는 상자 밖으로 넘치거나 잘리는 내용과 페이지 가로 넘침)
- 관련 스킬: `eli5`, `visual-doc`, `explain-diff`, `micro-world`
- 네 스킬 폴더에 같은 파일이 있다. 페이지 사본 끝에 붙여 `--dump-dom`으로 읽고, 읽는 폭과 500px(headless Chrome의 최소 폭)에서 각각 돌리고, 진행에 따라 내용이 바뀌는 페이지는 `#steps=N`으로 단계를 넘겨 잰다. 페이지 자체에는 넣지 않는다. 좌우 배치 판단(`Rows by default` 항목)과 그림 라벨 맞춤(`Fit the text before you draw`)은 브라우저가 있으면 이 측정이 글자 폭 표보다 우선한다. 한 곳을 고치면 네 파일을 같게 맞춘다. 정본: docs/decisions/2026-07-html-shared-design-rules.md (개정 2026-10-23, 2026-10-23 2차)

### 9. 렌더 검증 절차

- 범위: 렌더 검증 절차 (`Render check (shared` 소절: Chrome 라이트/다크 스크린샷, 1,800px 조각 확인, `measure.html` 두 폭, 금지 문자와 조사 grep)
- 관련 스킬: `eli5`, `visual-doc`, `explain-diff`, `micro-world`
- 네 SKILL.md에 같은 글로 들어 있고, 그 뒤의 스킬별 확인 항목만 다르다. `allowed-tools`의 `Bash(google-chrome *)`, `Bash(chromium *)`, `Bash(cat *)`, `Bash(python3 *)`, `Bash(which *)`, `Bash(grep *)`가 이 절차에 쓰인다. 정본: docs/decisions/2026-07-html-shared-design-rules.md (개정 2026-10-23 2차)

### 10. 코드 칩 줄바꿈

- 범위: 코드 칩 줄바꿈 (`inline code chip wrapping (SHARED` CSS 블록: 칩 안에서 줄을 바꾸지 않고, 600px 이하에서는 한 덩어리로 다음 줄로 넘기며 넘치면 칩 안에서 스크롤, 칩 좌우 여백 3px)
- 관련 스킬: `eli5`, `visual-doc`, `explain-diff`, `micro-world`
- 네 템플릿에 같은 코드로 들어 있다. 칩 색과 글자 크기는 템플릿마다 다르다. 정본: docs/decisions/2026-07-html-shared-design-rules.md (개정 2026-10-23 2차)

### 11. 부가 텍스트 금지와 말투

- 범위: 부가 텍스트 금지와 말투 (`No redundant or meta text; this rule comes first` 항목: 제목과 칩이 말한 것을 부제나 꼬리표로 되풀이하지 않고 페이지가 자기 형태(글꼴, 단위, 강조, 접기)를 설명하지 않음; `Voice: prose in 해요체` 항목: 산문은 동료에게 설명하는 해요체, 유치한 어미와 한다체와 명사로 끝나는 전보체 금지, 기대는 명사는 구체적으로, 개조식은 제목, deck, 칩, 라벨, 표 칸에만; Render check `Voice` 항목의 어미 집계 스크립트)
- 관련 스킬: `eli5`, `visual-doc`, `explain-diff`, `micro-world`
- 네 SKILL.md의 공유 Design rules 맨 앞 두 항목과 Render check 마지막 항목에 같은 글로 들어 있다 (eli5는 9번, 나머지는 6번). Step 6 Output의 언어 문장은 `Voice` 규칙을 가리키기만 한다. 한쪽을 고치면 넷을 같게 맞춘다. 배경: 2026-11-09 산출물(멀티턴 복원 검증 세트 200건 계획)의 해설 81문장이 모두 한다체와 전보체로 나온 사고와 시안 리뷰에서 지적된 부가 설명 꼬리표. 정본: docs/decisions/2026-08-eli5-history.md (개정 2026-11-09 5차)

### 12. 제목 아래 줄 없음과 숫자 한 곳

- 범위: 제목 아래 줄 없음과 숫자 한 곳 (제목 다음이 바로 그림, `.deck` 폐지 2026-11-10, 긴 설명은 접힌 `.more` 블록 한 곳(요약 줄에 이름, 3문장, beat 절반까지), 그림이 보여주는 숫자는 해설과 마무리 줄에 다시 쓰지 않음, 해설 80자, 마무리 줄 60자 한 문장, 본문 14.5px)
- 관련 스킬: `eli5` (현재 단독)
- SKILL.md Step 2 "제목은 그림이 무엇인지 말한다", The word budget, Render check 7번. `.deck` CSS는 옛 페이지 재스킨용으로 `frame.html`에 남아 있고 마크업 예시에서는 뺐다. visual-doc, explain-diff의 deck 규칙은 별개다. 정본: docs/decisions/2026-08-eli5-history.md (개정 2026-11-10)

### 13. 산문 원문 카드

- 범위: 산문 원문 카드 (`.prose-card`: 프롬프트, 에러 메시지, 안내문처럼 문장으로 쓰인 원문. `.pc-h` 종류, 이름, 파일, `.pc-b` 원문 문단마다 `<p>`, 모노 12.5px, 줄 간격 1.75, 76ch, 공백만 바꿈, 강조와 꼬리표와 접기와 요약 없음)
- 관련 스킬: `eli5` (현재 단독)
- 근거 카드의 네 번째 형태. 코드 인용 형태(`.ln`)로 산문을 넣거나 스킬 밖의 상자(`.prompt`)를 만들던 산출물을 대체한다. `.close` 머리 문장 규칙(45자, 쉼표 1개, 조건은 문단으로, 진행 메모 금지)도 같은 개정. visual-doc, explain-diff 이식은 eli5에서 검증한 뒤. 정본: docs/decisions/2026-08-eli5-history.md (개정 2026-11-09 5차)

### 14. 텍스트 스킬 글쓰기 규칙

- 범위: 텍스트 스킬 글쓰기 규칙 (`Writing rules (shared across commit` 소절: 기계 문장 기호 금지, 원문 인용 예외, 자연스러운 한국어, 제목은 답이지 틀이 아님, 보내기 전 확인)
- 관련 스킬: `commit`, `pr`, `issue`, `review-reply`, `code-review`, `handoff`
- 여섯 SKILL.md에 같은 글로 들어 있다. HTML 스킬 네 개의 `No machine-writing symbols` 항목과 금지 문자, 예외 규칙이 같고, `Titles are answers` 항목은 HTML 스킬의 `Headings are answers` 항목의 축약이므로 한쪽을 고치면 다른 쪽도 맞춘다. 정본: docs/decisions/2026-10-machine-writing-symbols.md, docs/decisions/2026-10-heading-hierarchy.md

### 15. 제목 규칙과 산출물 언어 구성안

- 범위: 제목 규칙과 산출물 언어 구성안 (`Headings are answers, not frames` 항목: 질문형, 분류 틀 명사, 방법 명사화, 단독 `A vs B`, 강조 틀, 개수 먼저 정하기, 주제를 감싸는 비유를 제목과 구성안에서 금지, 구성안은 산출물 언어로; Render check 5번 제목 grep은 읽을 목록)
- 관련 스킬: `eli5`, `visual-doc`, `explain-diff`, `micro-world`
- 네 SKILL.md에 같은 글로 들어 있다. 각 스킬의 구성안 절(eli5 Step 2와 "제목은 그림이 무엇인지 말한다", visual-doc "구성안은 산출물 언어로", explain-diff "Section titles" 첫 문단, micro-world 제목 줄)은 한글이고 스킬마다 예시가 다르다. 텍스트 스킬의 `Titles are answers` 항목과 같이 맞춘다. 정본: docs/decisions/2026-10-heading-hierarchy.md

### 16. 위치 표시

- 범위: 위치 표시 (`section hierarchy (SHARED` CSS 블록: `.progress` 3px 진행바, `.outline` 차례, `.sec-top`과 `.sec-no` 상위 번호 줄, `.sub-h`와 `.sub-no` 하위 번호, 그리고 body 끝의 같은 이름 스크립트)
- 관련 스킬: `eli5`, `visual-doc`, `explain-diff`, `micro-world`
- 네 템플릿에 같은 코드로 들어 있다. 제목 글자 크기(`.g-name`, `.sub-h h2`, `.sub-h h3`)와 마크업의 쓰임은 템플릿마다 다르다: eli5는 beat 4개 이하면 beat마다 번호 줄(하위 없음), 5개 이상이고 묶음이 있으면 묶음이 상위(`<section class="sec-top">` + h2)이고 beat가 하위(`.sub-h` + h3)이며 beat 하나뿐인 묶음은 `sec-top single`, visual-doc은 챕터가 상위이고 카드가 하위, explain-diff는 Part가 상위이고 섹션 카드가 하위이며 기존 TOC 카드가 차례 역할(`.outline` 미사용), micro-world는 진행바만. 사이드 TOC rail과 섹션 이름이 붙은 고정 머리는 두지 않는다 (visual-doc의 `.toc-rail`은 2026-10-29 제거). 차례 바로 아래 첫 상위 섹션은 `.hero + .sec-top, .outline + .sec-top`으로 윗여백 28px, 차례의 상위 줄은 굵고 하위 줄(`li.sub`)은 들여 쓴다 (2026-11-08). eli5만 묶음 안의 beat 사이에 왼쪽 120px 선(`.sec-top .beat + .beat::before`)이 더 있다 (2026-11-10 2차): 나머지 셋의 블록을 eli5에 복사하지 않는다. 정본: docs/decisions/2026-10-heading-hierarchy.md, docs/decisions/2026-08-eli5-history.md (개정 2026-11-08, 2026-11-10 2차)

### 17. 그림의 선 한 벌, 마무리 줄 배지, 그림 안의 글, 그림 안 제목, 해설 머리 표시

- 범위: 그림의 선 한 벌(`--dg-line`, `--dg-flow` 토큰, `.dg-box`, `.dg-arrow`, `.dg-head`, `.dg-cell`, 강조 `em`/`bad`, 없는 것 `absent`, 행 이름 `dg-l dg-row`), 마무리 줄 배지 두 종류(형광 `.lab`은 정리 머리 문장이 다시 말하는 줄에만, 테두리 `.lab.q`는 문구 목록 없이 판정 기준), 그림 안의 글(그린 것과 이름만, 읽는 데 필요한 조건은 접힌 `.res-more`, 나머지 수치는 보고서), 그림 안 제목(막대 묶음이 잰 것의 이름과 시스템 상자 이름은 `.dg-t` 13px 700, 막대 묶음 제목 아래 묶음 폭의 `.dg-rule` 선), 해설 머리 표시(번호 배지, 행 칩, 색 조각 `.sw`, 같은 글자), 구성안의 선행 확인
- 관련 스킬: `eli5` (현재 단독)
- SKILL.md "One stroke set", "Two badges", "A figure holds what it draws", "A title inside a figure is bold", "A note opens with a mark", Step 2 "읽기 전에 알아야 하는 것은 앞에 있다", Render check 7번과 9번. `figure labels (SHARED` 블록의 이름은 세 스킬이 같지만 선 클래스와 `.dg-t`, `.dg-rule`은 eli5에만 있다: visual-doc, explain-diff의 그림은 아직 `stroke="currentColor"`로 그린다. 옮길 때 이 행을 공유 항목으로 바꾼다. 정본: docs/decisions/2026-08-eli5-history.md (개정 2026-11-10 2차, 2026-11-11 2차)

### 18. 조작할 수 있는 그림의 조건

- 범위: 조작할 수 있는 그림의 조건 (정적 우선, 값이 3개 이상이거나 동작이 뒤집히는 경계가 요점일 때만, 세 가지 질문, 기본 상태가 요점, 순서 넘기기 금지)
- 관련 스킬: `eli5`, `visual-doc`, `explain-diff`
- 세 SKILL.md에 `Static first`로 시작하는 같은 5항목이 들어 있다. 두 상태 비교(전/후, 켬/끔)는 조작 요소가 아니라 두 줄짜리 정적 그림으로 그린다. 조작 UI는 템플릿마다 다르다 (eli5 `.ctl`, visual-doc `.knob`, explain-diff `.sim`/`.seg`). 여러 단계와 미션이 필요한 주제는 micro-world. 정본: docs/decisions/2026-08-eli5-history.md (개정 2026-10-03)

### 19. 근거 카드, 번호 배지, 캡션 칩, 해설 목록

- 범위: 근거 카드(`.ev`: 파일 원문, 유령 줄, ERD, 머리에 그림 배지, 같은 자리의 변경은 diff 형태 `.ln.del`/`.ln.add`/`.sign`), 번호 배지(`.dg-m`/`.m`: 색은 싣지 않고 번호는 자리, 모양은 종류. 해설만이면 회색 사각, 근거 카드가 딸리면 터미널 창 모양 `.code`. 한 그림에 번호열은 하나), 캡션 칩(`.dg-pill`/`.dg-cap`, 해설의 `.cap`: 칩이 행 이름이면 그 행은 배지 없이 해설이 같은 칩으로 시작), 순위 띠(칸 안 숫자 없음, 몫 블록 구분선)와 묶음 테두리(`.dg-grp`), 시스템 상자와 시퀀스 레인(`.dg-sys`, `.dg-life`, `.dg-dim`), 해설 목록(`.notes`, 배지 없는 `.notes.plain`)과 `그래서` 결론 줄
- 관련 스킬: `eli5` (현재 단독)
- 2026-10-10 eli5에 먼저 도입, 2026-10-13 묶음 배지, 배지 수 기준, 2026-11-08 배지 번호를 그림에만 두고 카드는 머리에 재사용, 모양으로 종류 구분, diff 카드, 근거 카드 세 트리거(`--code`/`--no-code`), 시퀀스와 시스템 상자 그림. 2026-11-08 2차: 배지 색 제거(회색 사각과 터미널 배지), 캡션 칩 축소와 해설의 `.cap`, 순위 띠 규칙, 인용 카드는 줄 배경 대신 홈 포인터, 제목은 명사구, 번호 형식 `1 / 3`, `1-1`. diff 줄의 색(`--green-soft` 추가, `--red-soft` 삭제, `.sign`)은 explain-diff 템플릿의 diff 줄과 같은 뜻이고 선택자만 다르다(eli5는 `.ev .ln.add`). visual-doc, explain-diff 이식은 eli5에서 한 번 더 검증한 뒤 진행하며, 옮길 때 이 행을 공유 항목으로 바꾼다. 정본: docs/decisions/2026-08-eli5-history.md (개정 2026-11-08, 2차)

### 20. 산출물 독자 구분

- 범위: 산출물 독자 구분 (explain-diff=머지할 개발자, micro-world=조작하며 익히는 사람, visual-doc=도메인 아는 동료, eli5=도메인 밖 사람)
- 관련 스킬: `explain-diff`, `micro-world`, `visual-doc`, `eli5`
- 네 스킬의 DO NOT TRIGGER 절이 서로를 가리키므로 한쪽 경계 수정 시 나머지 세 스킬의 description도 함께 갱신. 정본: docs/decisions/2026-08-eli5.md
