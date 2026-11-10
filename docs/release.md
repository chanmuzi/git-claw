# 릴리스 프로세스

> 이 문서는 `AGENTS.md`의 작업별 문서 표에서 연결된다. 릴리스할 때 읽는다.

## 릴리스 시점

릴리스는 사용자가 수동으로 결정한다. Feature PR에는 version bump를 포함하지 않는다.
릴리스하고 싶을 때 `chore: release vX.Y.Z` 커밋을 생성하고 태그를 단다.

## Semver 기준

마지막 태그 이후의 커밋을 분석하여 bump 타입을 결정한다:
- 새 스킬 추가, 주요 기능 변경 (`feat`) → **minor** (예: 1.7.1 → 1.8.0)
- 버그 수정 (`fix`) → **patch** (예: 1.7.1 → 1.7.2)
- 하위호환 깨지는 변경 → **major** (예: 1.x → 2.0.0)
- `docs`/`style`/`chore`만 있으면 → 릴리스 불필요

## 릴리스 절차

```bash
# 1. version bump 커밋
#    marketplace.json version + 변경된 SKILL.md version 동기화
git commit -m "chore: release vX.Y.Z"

# 2. 태그 생성 및 푸시
git tag vX.Y.Z && git push origin main vX.Y.Z

# 3. GitHub Release 생성: 여기까지 해야 릴리스가 끝난다
#    CHANGELOG.md의 해당 버전 섹션 본문을 릴리스 노트 파일로 저장한 뒤 사용
gh release create vX.Y.Z --title "vX.Y.Z" --notes-file <릴리스 노트 파일>
gh release list --limit 5   # 방금 릴리스한 버전이 Latest로 표시되는지 확인
```

**태그 푸시(2번)에서 멈추면 릴리스가 아니다.** GitHub Releases 페이지에 해당 버전이 나타나지 않아 사용자가 배포 여부를 확인할 수 없고, 이전 버전이 Latest로 남는다. 실제로 태그만 푸시된 채 Release가 누락된 사례가 반복됐다 (v1.8.0, v1.10.0~v1.12.0).

릴리스 노트는 `--generate-notes`(커밋 목록 자동 생성) 대신 `CHANGELOG.md`의 해당 버전 섹션 본문을 사용한다. 이 저장소는 CHANGELOG를 상세히 관리하므로 커밋 목록보다 정보량이 많다.

## 릴리스에 포함되는 파일

1. `.claude-plugin/marketplace.json`: `metadata.version` bump
2. `skills/*/SKILL.md`: frontmatter `version` 동기화
3. `CHANGELOG.md`: Change Log 항목 추가
4. `README.md`, `README.ko.md`: 필요 시 업데이트 (새 기능/변경된 동작)
