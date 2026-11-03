---
name: setup
description: 현재 프로젝트에 dev-harness 규칙 세트(브랜치 전략, 커밋·README·PR·이슈 규칙, CLAUDE.md, ARCHITECTURE.md)를 초기 세팅한다. "하네스 세팅", "하네스 초기화", "harness setup" 요청 시 사용.
disable-model-invocation: true
---

# dev-harness setup

현재 프로젝트에 하네스를 설치한다. 아래 순서대로 진행하고, 선택이 필요한 지점에서는 항상 `AskUserQuestion`으로 묻는다. 권장안이 있으면 첫 번째 선택지에 `(Recommended)`를 붙인다.

- 템플릿 원본은 `${CLAUDE_PLUGIN_ROOT}/templates/`다. 아래 `T/`는 이 경로를 줄여 쓴 것이다.
- 대상은 현재 작업 디렉터리의 git 저장소 루트다. 저장소가 아니면 중단하고 `git init`을 먼저 안내한다.
- 기존 파일을 말없이 덮어쓰지 않는다. 대상 경로에 파일이 이미 있으면 아래 "충돌 처리"를 따른다.
- 설치가 끝나도 커밋하지 않는다. 변경 목록만 보고한다.

## 충돌 처리

- 내용이 템플릿과 같으면 건너뛴다.
- 내용이 다르면 파일 이름과 차이를 보여 주고 `덮어쓰기` / `건너뛰기`를 묻는다. 건너뛴 파일은 마지막 보고에 남긴다.

## 순서

### ② 플랫폼

1. `git remote get-url origin`으로 호스트를 확인한다. `github.com`이면 GitHub, `gitlab`이 들어 있으면 GitLab을 권장안으로 둔다. 판단할 수 없으면 권장안 없이 묻는다.
2. `GitHub` / `GitLab` 중 고르게 한다. 선택값은 ⑤-b에서 쓴다.

### ⑤-a 공통 설치 (선택과 무관한 항목)

| 원본 | 설치 위치 |
| --- | --- |
| `T/gitmessage` | `.gitmessage` |
| `T/rules/commit-message.md` | `.claude/rules/commit-message.md` |
| `T/rules/readme-guidelines.md` | `.claude/rules/readme-guidelines.md` |
| `T/githooks/commit-msg` | `.githooks/commit-msg` (실행 권한 부여) |

복사한 뒤 아래 git 설정을 한다. 이미 다른 값이 설정돼 있으면 값을 보여 주고 바꿀지 묻는다.

- `git config commit.template .gitmessage`
- `git config core.hooksPath .githooks`: 내용 없는 커밋 소제목을 막는 훅을 켠다.
  - `core.hooksPath`가 이미 다른 디렉터리(예: `.husky`)를 가리키면 바꾸지 않는다. 그 디렉터리에 `commit-msg`가 없으면 훅을 그곳에 복사할지, `commit-msg`가 있으면 기존 훅에 호출을 덧붙일지 묻는다.
  - `core.hooksPath`가 없고 `.git/hooks/`에 `.sample`이 아닌 훅이 있으면, 설정하는 순간 그 훅들이 꺼진다. 목록을 보여 주고 `.githooks/`로 옮길지 묻는다.
  - `core.hooksPath`는 저장소마다 각자 설정해야 하는 로컬 설정이다. 마무리 보고에서 다른 팀원도 같은 명령을 실행해야 한다고 알린다.

### ① 브랜치 전략

1. `.claude/rules/branch-strategy.md`가 이미 있으면 첫 줄의 `<!-- branch-strategy: ... -->` 표식으로 현재 전략을 알려 주고, 바꿀지 먼저 묻는다. 유지하면 이 단계를 건너뛴다.
2. 아래 4개 중 고르게 한다. 각 선택지에 해당 `T/branching/<이름>/branch-strategy.md`의 첫 문단을 요약해 설명으로 붙인다.
   - `gitflow`: 릴리스 주기가 있고 여러 버전을 관리하는 프로젝트
   - `githubflow`: `main` 하나와 짧은 브랜치·PR, 지속적 배포
   - `gitlabflow`: 환경 브랜치형(`main` → `pre-production` → `production`)
   - `trunk-based`: 트렁크 하나에 자주 통합
3. 선택값은 ⑤-b에서 `.claude/rules/branch-strategy.md`로 설치한다. 첫 줄 표식은 지우지 않는다. `doctor`가 이 표식을 읽는다.

### ④ CLAUDE.md

프로젝트 루트의 `CLAUDE.md` 상태에 따라 나눈다. 선택값(설치 방식)은 ⑤-b에서 실행한다.

- **없음** → `T/CLAUDE.md`를 `CLAUDE.md`로 설치한다.
- **첫 줄이 `<!-- claude-md: karpathy-guidelines -->`** → 이미 하네스 CLAUDE.md다. 내용이 템플릿과 같으면 건너뛰고, 다르면 "충돌 처리"를 따른다.
- **그 밖의 기존 CLAUDE.md**
  1. 기존 내용을 `.claude/` 구조에 맞게 나눠 옮기자고 제안한다(`이전` / `거절`). 제안에는 내용을 읽고 파악한 분류안을 함께 보여 준다.
     - 주제별 지침 → `.claude/rules/<주제>.md` (특정 경로에만 해당하면 frontmatter `paths`로 제한)
     - 개인 선호·로컬 환경 내용만 → `CLAUDE.local.md` (gitignore 대상이므로 팀 지침은 넣지 않는다)
  2. `이전`이면 분류안대로 파일을 만든다.
  3. `거절`이면 기존 내용을 통째로 `.claude/rules/project-guidelines.md`로 옮긴다. 개인 내용만 `CLAUDE.local.md`로 분리한다.
  4. 어느 쪽이든 옮긴 내용이 빠짐없이 새 파일에 들어갔는지 확인한 뒤에만 루트 `CLAUDE.md`를 `T/CLAUDE.md`로 바꾼다. 확인하기 전에는 원본을 지우지 않는다.

### ③ ARCHITECTURE.md

1. 설치할지 묻는다. 기본은 설치다(`설치 (Recommended)` / `설치 안 함`).
2. 설치하면 ⑤-b에서 아래를 복사한다.

| 원본 | 설치 위치 |
| --- | --- |
| `T/docs/ARCHITECTURE.md` | `ARCHITECTURE.md` (루트) |
| `T/rules/architecture-doc.md` | `.claude/rules/architecture-doc.md` |

3. 루트에 `ARCHITECTURE.md`가 이미 있으면 파일은 건드리지 않고 규칙만 설치한다.

### 서드파티 고지

④에서 `T/CLAUDE.md`를 설치하기로 했을 때만 묻는다. 아래를 설명한 뒤 `설치` / `설치 안 함`을 고르게 한다.

- 설치되는 CLAUDE.md는 Karpathy 스타일 지침(MIT 라이선스)을 그대로 포함한다.
- 라이선스는 저작권 고지를 함께 배포하도록 요구한다. 대상 프로젝트를 외부에 공개하거나 배포한다면 `THIRD_PARTY_NOTICES.md`를 함께 두는 것이 안전하다.
- 내부에서만 쓰는 프로젝트라면 설치하지 않아도 된다. 설치된 CLAUDE.md 맨 위의 짧은 고지 주석은 그대로 남는다.

`설치`이면 ⑤-b에서 `T/docs/THIRD_PARTY_NOTICES.md`를 루트 `THIRD_PARTY_NOTICES.md`로 복사한다. 플러그인 루트의 `THIRD_PARTY_NOTICES.md`는 플러그인 자체용이므로 복사하지 않는다.

### ⑤-b 선택값 기반 설치

위에서 모은 선택값대로 한꺼번에 복사한다.

| 조건 | 원본 | 설치 위치 |
| --- | --- | --- |
| GitHub | `T/platforms/github/.github/` | `.github/` |
| GitLab | `T/platforms/gitlab/.gitlab/` | `.gitlab/` |
| 공통 | `T/rules/pr-issue-guidelines.md` | `.claude/rules/pr-issue-guidelines.md` |
| 공통 | `T/docs/specs/TEMPLATE.md` | `docs/specs/TEMPLATE.md` |
| ① 선택 | `T/branching/<전략>/branch-strategy.md` | `.claude/rules/branch-strategy.md` |
| ④ 선택 | `T/CLAUDE.md` 또는 이전 결과 | `CLAUDE.md` 등 |
| ③ 선택 | ARCHITECTURE.md와 규칙 | 위 ③ 표 |
| 고지 선택 | `T/docs/THIRD_PARTY_NOTICES.md` | `THIRD_PARTY_NOTICES.md` |

필요한 상위 디렉터리는 만들고, `.DS_Store`는 복사하지 않는다.

## 마무리

1. `git status --short`로 바뀐 파일을 보여 주고, 설치한 것과 건너뛴 것을 나눠 보고한다.
2. `ARCHITECTURE.md`를 설치했다면 조감도·코드맵을 채워야 하고 맨 위 `마지막 점검` 날짜도 직접 적어야 한다고 알린다.
3. `/dev-harness:doctor`로 설치 상태를 점검할 수 있다고 안내한다.
