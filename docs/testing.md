# 로컬 테스트

> 이 문서는 `AGENTS.md`의 작업별 문서 표에서 연결된다. 스킬을 로컬이나 브랜치에서 테스트할 때 읽는다.

```bash
# 방법 1: Skills CLI
npx skills add ./  # 로컬 경로에서 설치

# 방법 2: Claude Code Plugin (초기 설치 전용, 브랜치 테스트는 아래 '브랜치 테스트' 섹션 참조)
/plugin marketplace add /path/to/git-claw
/plugin install git-claw@git-claw

# SKILL.md 수정 후 반영 (캐시 갱신: 새 세션 불필요)
/reload-plugins

# 각 command 테스트
/commit             → diff 분석 후 conventional commit 메시지 제안 확인
/commit             → 직전 커밋 파일 재수정 후 실행 시, 새 커밋 생성 확인 (auto-amend 없음)
/commit --amend     → 직전 커밋에 현재 변경사항 합치기 (amend)
/commit --amend     → (이미 push된 커밋에 대해) force push 경고 및 확인 절차
/pr            → Individual PR 템플릿 생성 확인
/pr release    → Release PR 템플릿 생성 확인
/issue         → 이슈 템플릿 생성 및 label 자동 부여 확인
/issue bug     → Bug Report 템플릿 확인
/review-reply        → AI 리뷰 코멘트 수집/분석 확인
/review-reply 42     → 특정 PR 번호로 리뷰 확인
/code-review              → PR 자동 감지 → 변경사항 리뷰 → 현재 디렉토리 리뷰 (대화 맥락 반영)
/code-review 42           → 특정 PR 코드 리뷰
/code-review src/         → 특정 경로 코드 리뷰
/code-review --wd         → PR 브랜치에서도 working dir 리뷰 강제
/code-review --quick         → Quick 모드: 단일 패스, 도메인 2개 cap, Critical/Warning 우선 (없으면 Info fallback)
/code-review --full-scan     → PR 리뷰 시 diff 외부 pre-existing issue도 포함
/code-review -a|--all        → Reporting Bar 우회, confirmed finding 전체 보고 (기각분 포함)
/code-review                 → Reporting Bar 동작 확인: 취향 수준 관측이 개별 항목이 아닌 `기각 n건` 집계로만 나오는지,
                               선행 이슈가 `밖으로 미룸`에 한 줄로만 나오고 인라인 코멘트로 달리지 않는지
/code-review                 → Codex 플래그 없음: Codex 미실행 확인 (find, companion 명령이 실행되지 않아야 함, 힌트 없음)
/code-review --codex         → Codex opt-in: `setup --json`으로 로그인 확인 후 adversarial review 병렬 실행 (💡 Codex enabled 힌트)
/code-review --codex         → (codex logout 상태) Stop Notice B 출력 후 리뷰 중단 확인: 도메인 에이전트 미실행, 발행 없음
/code-review --codex         → (codex CLI 미설치, plugin만 있음) Stop Notice C 출력 확인 (`codex login`이 아니라 `npm install -g @openai/codex` 안내)
/code-review -q --codex      → ⚠️ Codex 생략 고지 후 Quick 모드 진행 (무음 폐기 아님)
/code-review --codex-general → Codex 일반 리뷰만 사용 (adversarial 없이)
/code-review --codex-both    → Codex review + adversarial 동시 실행
/code-review --no-codex      → Codex 강제 비활성화 (자연어 "codex도 같이" 요청보다 우선)
/handoff              → 세션 컨텍스트 분석 후 handoff 프롬프트 생성
/handoff -y           → 확인 없이 즉시 출력
/handoff auth 리팩토링 → 특정 주제 필터링된 handoff 생성
/explain-diff         → working diff 이해용 인터랙티브 HTML 문서 생성 (저장소 루트에 저장, 절대경로 안내)
/explain-diff 42      → PR #42 설명 문서 생성
/explain-diff src/    → 특정 경로로 diff 범위 제한
/micro-world          → 특정 동작을 만져볼 수 있는 일회성 시뮬 HTML 생성 (적용성 게이트: 시뮬 가치 없으면 정직하게 거절)
/visual-doc           → 조사와 분석 내용을 온-브랜드 인터랙티브 문서로 생성 (두 층: 토큰 고정 + 골격은 콘텐츠가 결정, 퀴즈 없음)
/visual-doc src/      → 특정 경로 분석 후 문서 생성
/visual-doc 42        → PR #42 분석 후 문서 생성
/eli5 <주제>          → 도메인 밖 사람용 그림 위주 설명 HTML 생성 (그림이 설명하고 글은 그림당 짧은 문단)
/eli5 42              → PR #42를 비전공자 기준으로 설명
/eli5 <설정과 테이블 주제>  → 그림이 말할 수 없는 사실(파일에 없는 줄, 테이블 컬럼)에만 근거 카드가 붙고, 배지 번호가 그림, 근거, 해설에서 일치하는지 확인
/visual-doc <표와 원문 인용이 많은 주제>  → Render check의 `data-ovf`가 none인지, `.wide` 표가 카드 안에서 줄바꿈되는지, `--flag` 칩이 줄 끝에서 갈라지지 않는지, em-dash가 든 원문을 고쳐 쓰지 않고 짧게 인용했는지 확인
/commit, /pr, /code-review  → 출력과 게시물에 `—`, `–`, `·`가 없는지, `> **Recommendation**: Accept, {이유}`처럼 콜론이 연달아 찍히지 않는지, Codex 출처 태그가 `(Codex)`인지 확인
/eli5 <두 상태 비교 주제>  → 비교가 좌우가 아니라 행으로 쌓이는지, 근거 카드가 한 열인지, `measure.html` 측정이 `data-sbs`와 `data-fit` 모두 `none`인지, 상태 캡션이 행 위 머리줄에 있고 비교 종류에 맞게 읽히는지(추천: 끝난 변경 BEFORE/AFTER, 제안 AS-IS/TO-BE, 그 밖은 행마다 참인 것)이며 색이 상태와 맞는지(깨지는 쪽만 빨강) 확인
/eli5 <코드 변경이 든 PR 2건 같은 주제>  → beat 5개 이상이면 차례가 두 단(상위 `1`, 하위 `1-1`, 0 채움과 점 없음)이고 차례 아래 첫 가로선까지 여백이 28px인지, 배지 번호가 그림에만 있고 근거 카드는 머리에 그림 배지를 다시 다는지, 근거 카드가 딸린 배지만 터미널 배지(`.m.code`, `dg-m code`: 어두운 창, 제목줄과 점 3개, 흰 숫자)이고 설명만 있는 배지는 회색 사각이며 빨간 배지가 없는지(`grep -c 'dg-m bad'`가 0), 같은 자리의 변경이 카드 하나의 diff(`.ln.del` 빨강 `-`, `.ln.add` 초록 `+`, `.sign`)로 나오는지, 요청이 시스템을 넘나드는 beat가 시퀀스 그림(`.dg-life`)으로 나오고 한 시스템 안의 단계가 `.dg-sys` 상자 안에 있는지 확인
/eli5 <전후 비교와 근거 카드가 있는 주제>  → 바뀐 상태의 흐린 행(`.dg-dim`)이 본문 보조색 수준으로 읽히고(점선 아님, 불투명도 .7), 근거 카드 머리가 파일 이름, 종류, 배지(오른쪽 끝) 순서이며 `grep -c '<div class="ev-head"><span class="m'`가 0인지, 상자 부제가 숫자, 식별자, 값뿐이고(절이나 조건 없음, 칩 뒤 설명 없음, 같은 부제 3회 이상 반복 없음) 흐름 그림에 큰 숫자(`dg-b`)나 막대가 없으며 그림당 `dg-q`가 1개 이하인지 확인
/eli5 <세팅 여러 개를 비교한 실험 주제>  → 대가 beat 앞에 결과 표 beat가 있고(`grep -c '<table class="res'`가 대개 1 또는 2, 단위가 다른 세트가 있으면 하나 더), 글 열은 왼쪽, 숫자 열(`class="n"`)은 머리글까지 오른쪽이며, 행마다 최고만 굵고 셋 이상 비교면 둘째가 밑줄이고, 칸에 숫자 외의 말(승패, 형용사)이 없고, 값이 하나도 없는 열이 없으며, 빈칸이 있으면 접힌 `res-more`(펼치기 전에는 요약 한 줄)만 표 아래 있는지 확인
/eli5 <묶음이 있고 그림 종류가 섞인 주제>  → 마무리 줄이 해설 목록 뒤에서 배지로 시작하고 형광 배지(`class="lab"`)가 정리 머리 문장의 절 수 이하이며 나머지는 테두리 배지(`lab q`)인지, 같은 배지 문구가 3번 이상 없고 `실측`, `지금은`, `선택`만 돌려 쓰지 않는지, 도형에 `stroke="currentColor"`가 없고(`grep -c`가 0) 흐름 그림과 막대 그림의 선 톤과 글자 크기가 같은지(행 이름 `dg-row`), 그림 맨 아래 조건 줄과 표 캡션 꼬리표가 없고 접힌 주석(`res-more`)이 그림 수보다 훨씬 적은지, 묶음 안 beat 사이에 120px 선이 있는지, 굵은 글자로 시작하는 해설의 그 이름이 그림에 적혀 있는지, 해설, 접힌 요약, 마무리 줄이 그 줄보다 위의 보이는 자리에 나오지 않은 것을 가리키지 않는지 확인
/eli5 <분기와 묶음 주제>    → 해설 하나가 상자 여럿을 말하면 묶음 테두리(`.dg-grp`)에 배지 하나만 붙는지, 세로 목록 그림에는 배지가 없는지(`.notes.plain`), 근거 카드 배지가 카드 머리에 있고(줄 안에는 없음) 꼬리표가 글자 바로 뒤에 붙는지 확인
/eli5 <영어 자료가 원문인 주제>  → 채팅에 보여주는 구성안의 제목이 처음부터 한글 명사구인지(`~하는가`, `~읽는 법`, `N축`, 단독 `A vs B`가 없는지), Render check의 제목 grep 결과가 "읽을 목록"으로 보고되고 남긴 제목에 이유가 붙는지, 머리 끝 차례의 줄 수가 beat 수와 같고 beat마다 `1 / N` 번호 줄이 있으며 h2 글자에 숫자가 없는지, 상단 진행바가 스크롤에 따라 채워지고 섹션 이름이 붙은 고정 머리와 사이드 목차가 없는지 확인
/visual-doc <챕터가 있는 긴 주제>  → 챕터가 `1 / N` 번호 줄과 가로선으로, 그 아래 카드가 왼쪽 막대와 `1-1`로 열리는지, 차례에 하위 줄이 들여 쓰여 있는지, 우측 TOC rail이 없는지 확인
```

## 브랜치 테스트 (merge 전 스킬 검증)

플러그인이 GitHub remote(`chanmuzi/git-claw`)에서 설치된 경우, `/reload-plugins`는 **main 브랜치**에서 가져온다.
fix/feat 브랜치의 SKILL.md 변경을 merge 전에 테스트하려면 아래 방법을 사용한다.

> **참고**: `/plugin marketplace add /local/path` → remote 복원 방식은 Claude Code 버그([#9537](https://github.com/anthropics/claude-code/issues/9537))로 `settings.json`에 `path` 잔여물이 남아 사용하지 않는다.

플러그인 cache 디렉토리에 로컬 스킬 파일을 복사한 뒤 `/reload-plugins`로 반영한다.
marketplace 디렉토리가 아닌 **cache 디렉토리**에 복사해야 동작한다.

**중요**: 플러그인 루트는 `~/.claude`가 아니라 `${CLAUDE_CONFIG_DIR:-$HOME/.claude}`다. 환경에 따라 `~/.claude-upstage`처럼 다른 디렉토리를 쓰며, 그때 `~/.claude/plugins/...`에 복사하면 조용히 무시된다 (2026-10-10 실제 사고). 활성 cache는 **스킬 실행 시 출력되는 `Base directory`** 가 정본이고, 그 다음이 `installed_plugins.json`의 `installPath`다. `ls -td`(최근 수정 순)는 `/reload-plugins`가 새 hash를 만들거나 다른 cache를 건드린 뒤에는 틀릴 수 있으므로 보조 수단으로만 쓴다.

**복사본이 무시되는 함정**: `/reload-plugins`가 marketplace(main)에서 **새 hash의 cache 디렉토리를 새로 만들어 버리면**, 직전에 복사해둔 cache는 활성 경로에서 밀려나고 구버전이 로드된다. "복사 → reload" 한 번으로 끝났다고 가정하지 말 것. reload 후 **활성 cache에 실제로 신버전이 있는지 grep으로 검증**하고, 구버전이면 새 cache에 재복사한 뒤 reload를 한 번 더 실행한다.

> marketplace commit이 바뀐 뒤 `/reload-plugins`를 실행하면 그 commit 이름의 새 cache 디렉토리가 생겨 누적된다. 오래된 cache는 주기적으로 정리하되, 최근 수정 순이 아니라 **모든 git-claw 항목의 installPath와 다른 세션이 쓰고 있는 디렉토리를 남기고** 지운다 (mtime 최신 디렉토리가 활성 cache가 아닐 수 있다). Claude Code는 세션이 로드한 cache 디렉토리 안에 `.in_use/{pid}` 마커를 남기므로, 살아 있는 pid가 있는 디렉토리는 다른 세션의 것이다 (2026-10-10 확인):
>
> ```bash
> PLUG=${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins
> KEEP=$(grep -o '"installPath": *"[^"]*/cache/git-claw/[^"]*"' "$PLUG/installed_plugins.json" | cut -d'"' -f4 | sed 's:/*$::')   # 항목이 여러 개면(user, project 범위) 전부
> [ -n "$KEEP" ] && for d in "$PLUG"/cache/git-claw/git-claw/*/; do
>   echo "$KEEP" | grep -qxF "${d%/}" && continue
>   live=$(for f in "$d".in_use/*; do p=$(basename "$f"); [ -d "/proc/$p" ] && echo "$p"; done 2>/dev/null)
>   [ -n "$live" ] && { echo "skip $d (세션 사용 중: $(echo $live))"; continue; }
>   rm -rf "$d"
> done
> ```

```bash
# 1. 활성 cache 경로 탐색 + 로컬 스킬 복사
#    installPath를 우선하고, 없으면 최근 수정 순. `skills/.`로 복사해야 한다 (`skills/`는 대상이 있으면 skills/skills/로 중첩됨)
PLUG=${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins
CACHE=$(grep -A6 '"git-claw' "$PLUG/installed_plugins.json" | grep -o '"installPath": *"[^"]*"' | head -1 | cut -d'"' -f4)
[ -d "$CACHE" ] || CACHE=$(ls -td "$PLUG"/cache/git-claw/git-claw/*/ | head -1)
[ -d "$CACHE" ] || { echo "CACHE 탐색 실패"; exit 1; }
cp -r skills/. "$CACHE/skills/"

# 2. (이 세션에서 실행) /reload-plugins   ← 수정된 스킬 메모리 로드

# 3. 검증 (필수): reload가 새 cache를 만들었으면 거기에도 신버전이 있어야 한다
#    /reload-plugins는 marketplace commit 이름(앞 12자리)으로 새 cache를 만들 수 있고, 이때 installPath는 바뀌지 않는다
#    (2026-10-23 확인). installPath만 비교하면 밀려난 옛 cache를 보고 "신버전"으로 잘못 통과하므로, installPath와
#    가장 최근 수정된 cache를 모두 확인한다. Bash 호출 사이에 셸 변수가 남지 않으므로 다시 계산한다
PLUG=${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins
CACHE=$(grep -A6 '"git-claw' "$PLUG/installed_plugins.json" | grep -o '"installPath": *"[^"]*"' | head -1 | cut -d'"' -f4)
NEWEST=$(ls -td "$PLUG"/cache/git-claw/git-claw/*/ | head -1); NEWEST=${NEWEST%/}
stale=0
for c in "$CACHE" "$NEWEST"; do
  [ -d "$c" ] || continue
  diff -q skills/{수정한 스킬}/SKILL.md "$c/skills/{수정한 스킬}/SKILL.md" >/dev/null \
    || { cp -r skills/. "$c/skills/"; echo "재복사: $c"; stale=1; }
done
[ $stale = 0 ] && echo "신버전 확인 (installPath, 최신 cache)" || echo "/reload-plugins 재실행 필요"
#    확정은 4단계의 "Base directory"로 한다: 그 경로의 SKILL.md가 신버전이어야 한다
#    가장 최근 수정된 cache가 다른 세션이 쓰는 디렉토리일 수도 있다 (그 세션이 reload하면 브랜치 버전을 받는다). 5단계에서 두 곳 모두 되돌린다
#    같은 세션에서 이미 호출한 스킬은 본문이 메모리에 남아 "already loaded above; instructions unchanged"로 구버전이 다시 쓰인다.
#    복사 뒤에는 반드시 /reload-plugins를 다시 실행한다

# 4. 수정한 스킬 테스트 수행
/{수정한 스킬}    # 예: /handoff, /commit, /code-review 등
#    스킬 실행 시 출력되는 "Base directory"에서 diff -q skills/{수정한 스킬}/SKILL.md "{Base directory}/SKILL.md"가 같다고 나와야 한다

# 5. 테스트 완료 후 복원 (marketplace는 main 상태이므로 여기서 복사. 3단계처럼 installPath와 최신 cache 모두)
PLUG=${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins
CACHE=$(grep -A6 '"git-claw' "$PLUG/installed_plugins.json" | grep -o '"installPath": *"[^"]*"' | head -1 | cut -d'"' -f4)
NEWEST=$(ls -td "$PLUG"/cache/git-claw/git-claw/*/ | head -1); NEWEST=${NEWEST%/}
for c in "$CACHE" "$NEWEST"; do [ -d "$c" ] && cp -r "$PLUG/marketplaces/git-claw/skills/." "$c/skills/"; done

# 6. (이 세션에서 실행) /reload-plugins   ← 원본 복원
```

> **`/reload-plugins`는 반드시 이 Claude Code 세션에서 실행**해야 한다. 별도 터미널에서 실행하면 현재 세션의 메모리에 반영되지 않는다.

## 테스트 안내 흐름

로컬 테스트 안내 시 아래 구조를 따른다. Claude가 복사, 실행, 리포트, 복원을 전담하고, 사용자는 `/reload-plugins`만 실행한다.

1. **활성 cache 탐색 + 복사**: `installed_plugins.json`의 `installPath`로 활성 cache 경로 탐색(없을 때만 `ls -td`) → 로컬 스킬 복사
2. **reload 요청**: 사용자에게 이 세션에서 `/reload-plugins` 실행 요청 (수정된 스킬 메모리 로드)
3. **검증**: reload 후 활성 cache를 다시 탐색해 `diff -q`로 신버전 여부 확인. 새 cache가 생겨 구버전이 로드됐으면 재복사 후 reload를 한 번 더 요청한다. 스킬 실행 시 표시되는 `Base directory`가 활성 cache와 일치하는지도 함께 확인한다. **이 단계를 건너뛰고 테스트 결과를 보고하지 않는다** (구버전 실행 결과를 신버전 결과로 오인해 보고하게 됨)
4. **테스트 실행 + 결과 리포트**: 스킬 실행 및 결과 보고
5. **복원 + reload 안내**: marketplace에서 최신 cache로 복원 → 사용자에게 이 세션에서 `/reload-plugins` 실행 안내

복원 절차를 테스트 전에 미리 보여주지 않는다. 시나리오 소개와 복원이 한 번에 나오면 정보 과부하.
