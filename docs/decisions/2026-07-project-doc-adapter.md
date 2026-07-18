# 2026-07 — 프로젝트 문서 체계: AGENTS.md 정본 + CLAUDE.md adapter

## 상태

확정 (2026-07-18). 이 저장소의 프로젝트 문서 정본은 **루트 `AGENTS.md`(실파일) 하나**이며,
루트 `CLAUDE.md`는 `@AGENTS.md` 한 줄 adapter다.

## 배경

기존 체계는 정반대 방향의 **symlink**였다: `AGENTS.md → CLAUDE.md` (git mode 120000),
즉 `CLAUDE.md`가 정본이고 Codex는 symlink를 통해 같은 내용을 읽었다.

내용 동기화 자체는 symlink가 보장했지만, 세 가지 문제가 있었다.

- **정본 방향이 표준과 반대** — 에이전트 중립 파일(`AGENTS.md`)이 정본이어야
  Codex를 포함한 비-Claude 에이전트가 1급 시민이 된다. 기존 구조에선 Codex가
  `# CLAUDE.md`라는 제목의 문서를 읽었다
- **symlink 이식성** — Windows는 관리자 권한/Developer Mode가 필요하고,
  `core.symlinks=false` 환경에서는 대상 경로 문자열만 담긴 일반 파일로 checkout 된다
- **Claude 전용 규칙을 분리해 둘 자리가 없음** — adapter 파일이면 import 아래에 추가 가능

EWS 에이전트 지침 로딩 가이드가 같은 이유로 symlink를 기각하고 일반 파일 adapter를
표준 패턴으로 선택했으며, chanmuzi-agent-harness 저장소도 동일 체계로 먼저 전환했다
(harness의 `docs/decisions/2026-07-agent-instruction-loading.md` 참조 — 로딩 모델 상세 포함).

## 결정

1. `AGENTS.md`를 **실파일 정본(SSoT)** 으로 전환 — 기존 CLAUDE.md 내용을 이관하고
   인트로를 에이전트 중립으로 수정
2. `CLAUDE.md`는 첫 줄이 정확히 `@AGENTS.md`인 **adapter** — 공유 내용을 복제하지 않는다
3. 에이전트 세션은 저장소 루트에서 시작한다
4. `docs/decisions/`를 신설하고 harness와 같은 규칙(`YYYY-MM-<topic>.md`)으로 결정을 기록한다

## 검토한 대안

- **기존 symlink 유지** — 동기화는 되지만 위 세 문제가 상수로 남는다. 기각
- **양쪽 실파일 복제** — drift 감시 비용 발생. harness가 렌더러 체계에서 같은 이유로 이탈했다. 기각

## 영향

- 프로젝트 규칙 변경 시 `AGENTS.md`만 편집한다
- 스킬 체크리스트의 문서 갱신 대상이 `CLAUDE.md`에서 `AGENTS.md`로 바뀐다
- 배포되는 스킬(`skills/*/SKILL.md`)은 이 결정의 대상이 아니다 — 각 스킬은 self-contained
