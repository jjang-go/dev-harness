# dev-harness

프로젝트마다 직접 만든 규칙 세트(하네스)를 초기 세팅해주는 Claude Code 플러그인.

## 설치

마켓플레이스를 등록하고 플러그인을 설치한다.

```
/plugin marketplace add jjang-go/dev-harness
/plugin install dev-harness@tykim-dev-harness
```

## 사용

| 명령 | 하는 일 |
| --- | --- |
| `/dev-harness:setup` | 현재 프로젝트에 하네스를 설치한다. 플랫폼, 브랜치 전략, CLAUDE.md, ARCHITECTURE.md 설치 여부를 차례로 묻는다. |
| `/dev-harness:doctor` | 설치 상태를 점검하고 문제를 보고한다. 파일은 고치지 않는다. |

`setup`은 기존 파일을 말없이 덮어쓰지 않고, 설치가 끝나도 커밋하지 않는다.

### setup이 설치하는 것

| 항목 | 설치 위치 |
| --- | --- |
| 커밋 메시지 템플릿 | `.gitmessage` (`git config commit.template`도 설정) |
| 커밋 메시지·README 작성 규칙 | `.claude/rules/commit-message.md`, `.claude/rules/readme-guidelines.md` |
| PR·이슈 템플릿과 작성 규칙 | `.github/` 또는 `.gitlab/`, `.claude/rules/pr-issue-guidelines.md` |
| 기능 명세 템플릿 | `docs/specs/TEMPLATE.md` |
| 브랜치 전략 규칙 (4개 중 선택) | `.claude/rules/branch-strategy.md` |
| 코딩 지침 | `CLAUDE.md` |
| 아키텍처 문서 (선택) | `ARCHITECTURE.md`, `.claude/rules/architecture-doc.md` |
| 서드파티 고지 (선택) | `THIRD_PARTY_NOTICES.md` |

## 구조

```
.claude-plugin/
  plugin.json        # 플러그인 매니페스트
  marketplace.json   # 마켓플레이스 등록용
skills/
  setup/             # 하네스 설치 스킬
  doctor/            # 설치 상태 점검 스킬
agents/              # 서브에이전트 정의
hooks/hooks.json     # 훅 설정
templates/           # 프로젝트로 복사될 템플릿과 규칙
  rules/             #   .claude/rules/ 로 설치되는 규칙
  branching/         #   브랜치 전략별 규칙 (gitflow, githubflow, gitlabflow, trunk-based)
  platforms/         #   GitHub·GitLab PR/이슈 템플릿
  docs/              #   ARCHITECTURE.md, 기능 명세 템플릿
  gitmessage         #   커밋 메시지 템플릿
  CLAUDE.md          #   코딩 지침
```

## 로컬 테스트

```
claude --plugin-dir .
claude plugin validate .
```

## 브랜치 전략

이 저장소는 Gitflow를 쓴다. 전체 규칙은 `templates/branching/gitflow/branch-strategy.md`와 같다.

- `main`: 릴리스된 코드만 둔다. 모든 커밋에 버전 태그를 붙인다.
- `develop`: 다음 릴리스를 위한 통합 브랜치
- `feature/*`, `fix/*` 등: `develop`에서 만들어 `develop`으로 병합한다.
- `release/<버전>`: `develop`에서 만든다. 버그 수정, 버전 갱신, 문서만 넣고 `main`에 병합한 뒤 `develop`에도 병합한다.
- `hotfix/<버전>`: `main`에서 만든다. `main`과 `develop` 양쪽에 병합한다.
- 모든 병합은 `--no-ff`로 한다.
