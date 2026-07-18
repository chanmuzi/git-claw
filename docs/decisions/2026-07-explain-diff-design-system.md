# 2026-07 — explain-diff / micro-world 공유 디자인 시스템

## 상태

확정 (2026-07-18). 예정된 두 스킬(`explain-diff`, `micro-world`)이 생성하는 모든 HTML 산출물은
아래 단일 디자인 토큰 세트를 따른다. 기준 시안: **B v2.3** (Toss Card 계열, 사용자 확정).

## 배경

Geoffrey Litt(Notion)의 `/explain-diff` 개념을 도입하기로 했다: 코드 변경을 raw diff가 아닌
literate diff(산문 + 임베드 코드 + 인터랙티브 요소 + 이해도 퀴즈)로 설명하는 HTML 문서를
생성하는 스킬. micro-world(변경 전용 일회성 인터랙티브 시뮬)는 **별도 스킬**로 분리한다 —
원작자도 두 기법을 분리했고, explain-diff는 구조까지 정형, micro-world는 룩만 정형이기 때문.

두 스킬의 산출물이 일관된 룩앤필을 가지려면 공유 토큰이 필요했다. 시안 3종
(A: Editorial Ink / B: Toss Card / C: Graphite)을 제작·비교했고 사용자가 B를 선택,
이후 TDS(Toss Design System) 공개 문서의 패턴을 참고해 3차례 개정(v2.1→v2.3)했다.

## 결정: 디자인 토큰 (v2.3)

### 컬러

| 토큰 | 값 | 용도 |
|------|-----|------|
| gray900 | `#191F28` | 본문 잉크 |
| gray700 | `#333D4B` | 보조 잉크 |
| gray600 | `#4E5968` | 서브 텍스트 |
| gray500 | `#6B7684` | 옅은 서브 |
| gray400 | `#8B95A1` | faint (목차 번호, 메타) |
| gray300 | `#B0B8C1` | chevron 등 |
| gray200 | `#D1D6DB` | 비활성 CTA 배경 |
| gray100 | `#E5E8EB` | 구분선 |
| gray50 | `#F2F4F6` | 페이지 배경 |
| gray25 | `#F9FAFB` | 코드블록 배경 |
| blue | `#3182F6` | 악센트 (버튼, 강조) |
| blue-dark | `#1B64DA` | hover |
| blue-soft | `#E8F3FF` | 틴트 배경 (eyebrow, figure) |
| green / green-soft | `#0CA678` / `#E6F6F0` | diff 추가, 정답 |
| red / red-soft | `#F04452` / `#FDEDEE` | diff 삭제, 오답, Critical |
| amber / amber-soft | `#B76E00` / `#FFF4DE` | Watch |

### 형태·타이포

- 라운드 3단: 카드 20px / 요소 14px / 버튼 10px
- 섀도: `0 1px 2px rgba(25,31,40,.03), 0 8px 24px rgba(25,31,40,.06)` (소프트 단일)
- 폰트: 시스템 스택 `-apple-system, "Apple SD Gothic Neo", "Pretendard", …` + 모노 `ui-monospace, "SF Mono", …`
  (Toss Product Sans는 전용 폰트라 사용 불가)
- 타이틀 700 / 섹션 700 / 버튼·본문 강조 600. 800은 쓰지 않는다 (과볼드 금지)
- letter-spacing `-0.01em ~ -0.03em`

### 컴포넌트 패턴 (TDS 패턴 재해석)

- **Top**: 필 eyebrow + 볼드 타이틀 + 리드문 + 메타 칩(pill)
- **ListHeader**: 넘버 스퀘어(22px, blue-soft) + 볼드 제목 + 트레일링 메타
- **List row**: 제목/설명 2줄 + 트레일링(chevron 또는 배지). 리딩 아이콘 없음
- **버튼**: fill(blue)/weak(white+blue) 2단 위계, 13px/600/패딩 8×14, `:active` scale(.97)
- **diff 블록**: gray25 배경, 추가=green-soft wash, 삭제=red-soft wash (저채도)
- **인터랙티브 피규어**: blue-soft 그라데이션 컨테이너 + 스텝 노드(활성 시 blue fill + lift)
- **퀴즈**: 선택지 리스트 행(gray25, hover 시 blue-soft), 정답 green/오답 red 판정 + 해설 공개
- **BottomCTA × 퀴즈 게이트**: 하단 고정 풀폭 CTA. 퀴즈 통과 전 gray200 비활성,
  전원 정답 시 blue 활성("이해 완료"). Litt의 speed regulator 규칙의 UI 구현

### 라이팅·금지 규칙

- **해요체 전면 적용**, 능동태, 긍정 프레이밍 (TDS UX 라이팅 원칙 차용)
- **이모지 금지** (파비콘 제외)
- **손그림 SVG 아이콘 금지** — 아웃라인·필드 두 차례 시도 후 "임의로 그린 티"로 기각.
  아이콘 없이 타이포·여백·배지로 위계를 만든다
- **외부 의존성 0** — self-contained HTML (인라인 CSS/JS, CDN·웹폰트·원격 이미지 금지)
- 산출물은 작업 저장소 루트에 저장하고 절대경로만 안내한다 (자동 열기 없음, 삭제는 사용자 지시 시)

## 검토한 대안

- **A: Editorial Ink** (애플 문서 계열, 카드 없는 타이포 중심) — 사용자 선호로 기각
- **C: Graphite** (모던 다크) — 동일
- **아웃라인 SVG 아이콘 → 필드 SVG 아이콘 → 아이콘 제거**: 실제 TDS 아이콘은 라이선스상
  사용 불가, 손그림 근사치는 품질 미달. "못생긴 아이콘보다 아이콘 없음"으로 확정

## 법적 주의

TDS 자산(UI Kit, 아이콘, Toss Product Sans)은 토스 지식재산으로 앱인토스 파트너 라이선스
대상이다. 이 디자인 시스템은 **공개 문서의 원칙·패턴을 참고한 독자 구현**이며 TDS 자산을
포함하지 않는다. 토큰 값은 자체 정의다.

## 참고

- 확정 시안(작업 아티팩트): https://claude.ai/code/artifact/c8b0a983-8225-4a01-b862-0a096802da34 (버전 라벨 B-v2.3-no-icons-slim)
- Geoffrey Litt 원작 스레드: https://threadreaderapp.com/thread/2072522251300409556.html
