# 결정 문서 목록

규칙을 그렇게 정한 이유를 주제별로 적은 기록이다. 지금 적용되는 규칙의 글은 `AGENTS.md`, `docs/`의 작업별 문서,
각 스킬의 `SKILL.md`에 있고, 여기에는 배경과 검토한 대안이 있다. 아래 표에서 주제를 찾아 그 파일만 읽는다.

| 문서 | 다루는 것 | 적용 범위 | 상태 |
|---|---|---|---|
| [2026-07-project-doc-adapter.md](./2026-07-project-doc-adapter.md) | `AGENTS.md` 정본과 `CLAUDE.md` adapter | 저장소 전체 | 확정 |
| [2026-07-explain-diff-micro-world-design.md](./2026-07-explain-diff-micro-world-design.md) | 두 스킬의 디자인 토큰, 산출물 구조, 퀴즈 규격 | explain-diff, micro-world | 확정 |
| [2026-07-html-shared-design-rules.md](./2026-07-html-shared-design-rules.md) | visual-doc의 두 층 구조와 HTML 스킬 공유 디자인 규칙 (전환 버튼, 코드 블록, 문장 규칙, 렌더 측정) | HTML 스킬 4개 | 확정, 개정 7건 |
| [2026-08-eli5.md](./2026-08-eli5.md) | eli5 결정의 현재 상태와 개정 목록 | eli5 | 확정 |
| [2026-08-eli5-history.md](./2026-08-eli5-history.md) | eli5 원본 결정과 개정 본문 19건 | eli5 | 기록 |
| [2026-10-heading-hierarchy.md](./2026-10-heading-hierarchy.md) | 제목 규칙, 산출물 언어 구성안, 위치 표시 | 스킬 10개 | 확정 |
| [2026-10-machine-writing-symbols.md](./2026-10-machine-writing-symbols.md) | 기계 문장 기호 금지 | 스킬 10개 | 확정 |
| [2026-11-doc-routing.md](./2026-11-doc-routing.md) | `AGENTS.md`의 작업별 문서 표와 문서 분리 (계획 문서: [html](./2026-11-doc-routing.html)) | 저장소 전체 | 확정 |

옛 이름: `2026-07-html-shared-design-rules.md`는 `2026-07-visual-doc.md`, `2026-07-explain-diff-micro-world-design.md`는
`2026-07-explain-diff-design-system.md`였다 (2026-11-11 개명).

## 결정 문서를 쓰고 고칠 때

- 파일 이름은 `YYYY-MM-<다루는 것>.md`이고, 이름은 스킬 하나가 아니라 문서가 실제로 다루는 범위를 말한다.
- 새 문서를 만들면 위 표에 한 줄을 더한다. 표에 없는 문서는 찾을 수 없다.
- 기존 결정을 뒤집으면 덧붙이지 않고 새 문서를 만들고, 옛 문서의 상태를 `YYYY-MM-<새 문서>로 대체`로 바꾼다.
- 같은 결정 안의 개정은 그 문서에 `## 개정 YYYY-MM-DD: 제목` 절로 붙인다. 개정이 5건을 넘으면 eli5처럼
  현재 상태와 개정 목록을 담은 문서, 원본과 개정 본문을 담은 `-history` 문서로 나눈다.
- 개정 내용을 `docs/shared-logic.md`나 CHANGELOG에 다시 길게 쓰지 않는다. 300자 안에서 쓰고 이 폴더의 문서를 가리킨다.
- 위 표의 행에는 값, 날짜별 변경, 개정 내용을 적지 않고 300자를 넘기지 않는다. "다루는 것" 칸은 그 문서를 언제 여는지 알 수 있는 말로 쓴다.
