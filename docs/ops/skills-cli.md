# Skills CLI 운영 가이드

> 이 문서는 `AGENTS.md`의 작업별 문서 표에서 연결된다. `npx skills` 명령을 실행하기 전에 읽는다.

`npx skills`(upstream `vercel-labs/skills`) 운영 시 알려진 함정과 안전 규칙. 이 가이드는 실제 사고(2026-04-25, universal install로 의도치 않게 전환되어 Claude Code에서 plugin/skill 중복 노출 발생) 회고에서 도출됨.

## 운영 원칙

- **기존 install 형태(scope, target, copy/symlink)를 사용자 동의 없이 변경하지 않는다.** update/cleanup은 기존 설치 형태를 그대로 유지하는 방식으로만 진행한다.
- update/remove/add 명령의 옵션 차이로 install 위치가 바뀔 수 있다. 옵션을 명시하지 않은 채 실행하지 말고, 변경되는 디렉터리/symlink target을 사전에 확인한다.
- "guide"/"안내"/"정리"는 **설명 요청**이지 실행 요청이 아니다. 실행이 필요하면 사용자에게 명시적 승인을 받는다.

## 알려진 함정

- **`npx skills update`의 silent no-op**: copy-mode로 설치된 skill에 대해 update가 "Updated"로 표시되지만 실제 파일은 변경되지 않는 경우가 있다. timestamp(`ls -la`)와 `diff -q`로 실제 갱신 여부를 검증해야 한다.
- **`npx skills add ... --all`의 universal store 부작용**: default가 `~/.agents/skills/`(universal store)에 source를 두고 모든 host agent(Claude Code, Codex, Cursor 등)에 symlink하는 형태. Claude Code에서 plugin install된 동일 skill이 있으면 `~/.claude/skills/{name}` symlink와 plugin install이 동시에 잡혀 중복 노출됨.
- **빌트인 `/review`와 `git-claw:review-reply`의 트리거 충돌**: Claude Code 빌트인 `/review`(PR 리뷰)가 트리거를 가져갈 수 있어 `review-reply`가 의도된 시점에 호출되지 않을 수 있다. 명시적으로 `/git-claw:review-reply`처럼 plugin prefix를 사용하면 충돌을 피할 수 있다.

## Install scope 비교

| 명령 | source 위치 | 영향받는 agent |
|------|------------|---------------|
| `npx skills add chanmuzi/git-claw` (no `-g`) | 현재 프로젝트 내부 | 현재 프로젝트의 agents |
| `npx skills add chanmuzi/git-claw -g --all` | `~/.agents/skills/{name}` (universal) | **모든 host agent**에 symlink (Claude Code 포함) |
| 직접 복사 (아래 복원 레시피) | `~/.codex/skills/{name}` | Codex만 |

> `--all`은 `--skill '*' --agent '*' -y` shorthand. `-a Codex`로 agent를 좁혀도 Skills CLI 버전에 따라 universal로 가는 경우가 있으므로, **Codex 전용 install이 필요하면 직접 복사 방식을 사용한다.**

## 복원 레시피 (Codex 전용 install로 복구)

universal install로 잘못 전환된 경우 또는 update가 동작하지 않는 경우:

```bash
# 1. git-claw 관련 스킬 제거 (universal source + 해당 agent symlink 정리)
npx skills remove -g commit pr issue review-reply code-review handoff explain-diff micro-world visual-doc eli5 -y

# 2. marketplace를 최신 main으로 pull
MKT=${CLAUDE_CONFIG_DIR:-$HOME/.claude}/plugins/marketplaces/git-claw
git -C "$MKT" pull

# 3. marketplace → ~/.codex/skills/ 직접 복사 + .installed-ref 수동 기록
SHA=$(git -C "$MKT" rev-parse HEAD)
for s in commit pr issue review-reply code-review handoff explain-diff micro-world visual-doc eli5; do
  rm -rf ~/.codex/skills/$s
  cp -R "$MKT/skills/$s" ~/.codex/skills/$s
  echo "$SHA" > ~/.codex/skills/$s/.installed-ref
done

# 4. 검증: Claude Code 쪽 중복 없는지 확인
ls "${CLAUDE_CONFIG_DIR:-$HOME/.claude}/skills/"   # context7 같은 universal-only skill만 보여야 함

# 5. (사용자 직접) Codex CLI 세션 재시작
```
