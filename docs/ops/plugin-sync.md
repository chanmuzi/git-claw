# main push 후 동기화와 스킬 cleanup

> 이 문서는 `AGENTS.md`의 작업별 문서 표에서 연결된다. main에 push한 뒤, 또는 스킬을 rename하거나 제거할 때 읽는다.

- main push 후 Claude Code 플러그인 동기화 (anthropics/claude-code#42983 해결 시 제거):
  1. `git -C "${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins/marketplaces/git-claw" pull`: marketplace 최신화
  2. 이 세션에서 `/reload-plugins` 실행: 최신 cache 생성
  3. `installed_plugins.json`(`${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins/installed_plugins.json`)의 git-claw `installPath`를 새 cache 경로로 업데이트
     - 새로 시작하는 다른 프로젝트 세션은 수정된 `installPath`를 자동으로 사용
     - 이미 실행 중인 다른 프로젝트 세션은 자동 반영되지 않으므로 `/reload-plugins` 실행 또는 세션 재시작 필요
  ```bash
  # 새 cache 경로 확인 (reload 직후에는 방금 생긴 디렉토리가 mtime 최신이다)
  PLUG=${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins
  ls -td "$PLUG"/cache/git-claw/git-claw/*/ | head -1
  # $PLUG/installed_plugins.json 에서 git-claw의 installPath를 위 경로로 수정
  ```
  4. 오래된 cache 정리: **반드시 installPath 갱신(3번) 후에** 실행한다. `/reload-plugins`마다 새 hash 디렉토리가 생겨 구버전 cache가 누적되며, 다른 도구가 stale 경로를 참조하면 outdated 경고의 원인이 된다. installPath 디렉토리와 `.in_use/{pid}`에 살아 있는 pid가 있는 디렉토리(다른 세션이 로드 중)는 남긴다:
  ```bash
  PLUG=${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins
  KEEP=$(grep -A6 '"git-claw' "$PLUG/installed_plugins.json" | grep -o '"installPath": *"[^"]*"' | head -1 | cut -d'"' -f4)
  [ -d "$KEEP" ] && for d in "$PLUG"/cache/git-claw/git-claw/*/; do
    [ "${d%/}" = "${KEEP%/}" ] && continue
    live=$(for f in "$d".in_use/*; do p=$(basename "$f"); [ -d "/proc/$p" ] && echo "$p"; done 2>/dev/null)
    [ -n "$live" ] && { echo "skip $d (세션 사용 중: $(echo $live))"; continue; }
    rm -rf "$d"
  done
  ```
- main push 후 Codex 동기화 (**`npx skills update` 사용 금지: 직접 복사**):
  - `npx skills`는 자동 갱신되지 않고, update 계열 명령이 install 형태를 universal(`~/.agents/skills`)로 바꿔놓는 경우가 있다. 그렇게 되면 Claude Code에 plugin install과 symlink가 동시에 잡혀 **동일 skill이 중복 노출된다** (2026-04-25 실제 사고). `docs/ops/skills-cli.md` 참조.
  - 이 저장소의 Codex install은 **Codex 전용**(`~/.codex/skills`)이다. 이 형태를 유지하려면 marketplace에서 직접 복사하고 `.installed-ref`를 수동 기록한다.
  - Codex CLI는 세션 시작 시 skill 목록을 캐싱하므로 동기화 후 **세션 재시작** 필요.
  ```bash
  # 1. 동기화 상태 확인
  #    Codex 전용 install(~/.codex/skills) + universal install(~/.agents/skills) 모두 탐색
  #    .installed-ref가 있는 디렉터리만 골라내 stale/누락 폴더와 install scope를 함께 노출
  for base in ~/.codex/skills ~/.agents/skills; do
    [ -d "$base" ] || continue
    scope=$([ "$base" = ~/.agents/skills ] && echo "universal" || echo "codex")
    for d in "$base"/*/; do
      [ -f "$d/.installed-ref" ] || continue
      printf "%-15s %s [%s]\n" "$(basename "$d"):" "$(cat "$d/.installed-ref")" "$scope"
    done
  done

  # 2. marketplace를 최신 main으로 pull 후 직접 복사 + .installed-ref 기록
  MKT=${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins/marketplaces/git-claw
  git -C "$MKT" pull
  SHA=$(git -C "$MKT" rev-parse HEAD)
  for s in commit pr issue review-reply code-review handoff explain-diff micro-world visual-doc eli5; do
    rm -rf ~/.codex/skills/$s
    cp -R "$MKT/skills/$s" ~/.codex/skills/$s
    echo "$SHA" > ~/.codex/skills/$s/.installed-ref
  done

  # 3. 검증: 파일이 실제로 갱신됐는지(silent no-op 아닌지) + universal 오염 없는지
  diff -q "$MKT/skills/code-review/SKILL.md" ~/.codex/skills/code-review/SKILL.md
  ls "${CLAUDE_CONFIG_DIR:-$HOME/.claude}/skills/"   # git-claw skill이 보이면 universal install로 오염된 것
  ```
- 스킬 rename / 제거 시 stale 폴더 cleanup:
  - `npx skills` CLI는 rename, 제거를 추적하지 않아 옛 디렉터리(`~/.codex/skills/{old-name}/`)가 잔존한다. 사용자에게 동일 description의 skill이 두 개로 보이거나, 옛 이름으로 노출되는 문제가 발생함.
  - rename/제거 PR의 CHANGELOG 항목에 반드시 다음 cleanup 안내를 포함한다 (install 형태를 바꾸지 않는 직접 삭제를 우선한다):
    ```bash
    rm -rf ~/.codex/skills/{old-name}        # 권장
    npx skills remove {old-name} -g          # 대안. 단, 설치 scope과 일치해야 함
    ```
  - Claude Code plugin 쪽은 cache가 marketplace 트리를 그대로 미러링하므로 rename 시 자동 정리됨 (별도 cleanup 불필요).
