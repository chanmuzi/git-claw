# 기록 문서 점검

> 이 문서는 `AGENTS.md`의 작업별 문서 표에서 연결된다. 기록 문서가 길어졌거나 찾기 어렵다고 느낄 때, 또는 사용자가
> 문서 구조 점검을 요청할 때 읽는다. 정해진 주기는 없다. 필요할 때 한 번 돌리고 끝낸다.

## 재는 것

저장소 루트에서 실행한다. 출력이 없으면 통과다.

```bash
# 1. AGENTS.md 크기: 32,768바이트를 넘으면 Codex가 뒤쪽을 읽지 못한다
[ "$(wc -c < AGENTS.md)" -le 32768 ] || echo "AGENTS.md가 32KiB를 넘음: $(wc -c < AGENTS.md)"

# 2. 깨진 경로: 문서가 가리키는 docs/ 파일이 실제로 있는지
for f in $(grep -oh 'docs/[A-Za-z0-9_./-]*\.\(md\|html\)' AGENTS.md docs/*.md docs/ops/*.md docs/decisions/README.md | sort -u); do
  [ -e "$f" ] || echo "없는 파일을 가리킴: $f"
done

# 3. 목록에 없는 결정 문서
for f in docs/decisions/*.md; do
  b=$(basename "$f"); [ "$b" = README.md ] && continue
  grep -q "$b" docs/decisions/README.md || echo "결정 문서 목록에 없음: $b"
done

# 4. 목록 표의 긴 행 (경고): 300자를 넘으면 값이나 이력이 행에 들어온 것이다
for f in docs/shared-logic.md docs/decisions/README.md; do
  LC_ALL=C.UTF-8 awk -v f="$f" '/^\| / && length($0) > 300 {print "긴 행: " f ":" NR " (" length($0) "자)"}' "$f"
done
```

1~3은 고친다. 4는 읽어 보고 판단한다.

크기 추이를 보고 싶을 때:

```bash
wc -c AGENTS.md CHANGELOG.md docs/*.md docs/ops/*.md docs/decisions/*.md | sort -n | tail -8
```

## 읽고 판단하는 것

숫자로 잡히지 않는 것은 아래 질문으로 본다. 해당하는 것만 고친다.

- **`AGENTS.md`에 특정 작업에서만 필요한 글이 다시 들어왔는가.** 들어왔으면 `docs/` 문서로 옮기고 작업별 문서 표에 줄을 더한다.
- **작업별 문서 표의 왼쪽 칸이 "언제 여는가"로 쓰여 있는가.** "무엇이 들어 있는가"로 쓴 줄은 에이전트가 자기 작업과 대조하지 못한다.
- **두 번째로 읽는 문서가 불어나고 있는가.** `AGENTS.md`를 줄이면 그다음에 읽는 목록 파일(`docs/shared-logic.md`, `docs/decisions/README.md`)에 요약과 수치가 붙기 시작한다. `AGENTS.md`의 크기 확인은 이 파일들을 지켜 주지 않는다.
- **같은 글이 두 곳 이상에 있는가.** 개정 내용은 결정 문서에만 길게 쓰고, 다른 곳은 한 줄로 쓰고 가리킨다.
- **결정 문서에 개정이 5건 넘게 쌓였는가.** `docs/decisions/README.md`의 규칙대로 현재 상태 문서와 `-history` 문서로 나눈다.
- **파일 이름이 내용과 어긋나는가.** 개명은 참조를 모두 고쳐야 하므로 이득이 분명할 때만 한다. 목록의 "다루는 것" 칸을 고치는 것으로 충분한 경우가 많다.

## 하지 않는 것

- 검사를 늘리지 않는다. 위 네 가지로 잡히지 않는 문제는 규칙을 더하기 전에 실제로 두 번 이상 생겼는지 본다.
- 주기 점검이나 자동 실행을 두지 않는다. 문서 구조를 고치는 PR을 올리기 전에 한 번 돌린다.
- 줄 수로 길이를 제한하지 않는다. 한 줄이 길어질 뿐이다. 길이는 바이트나 글자 수로 적는다.

## 근거

- Codex의 32KiB 상한과 이번 정리의 배경: docs/decisions/2026-11-doc-routing.md
- Claude Code 메모리 문서: `CLAUDE.md`는 파일당 200줄 이하 권장, `@` import는 context 비용을 줄이지 않음 (code.claude.com/docs/en/memory)
- 결정 기록 관례: 뒤집힌 결정은 지우지 않고 대체 표시 (Nygard의 ADR, MADR)
- 목록 파일과 기록 파일을 나누고 모순, 낡은 주장, 가리키는 곳 없는 문서를 점검하는 방식 (Karpathy의 LLM wiki)
