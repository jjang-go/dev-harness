---
name: setup
description: 현재 프로젝트에 dev-harness 규칙 세트(브랜치 전략, 커밋·README·PR·이슈 규칙, 비밀값 커밋 방지, CLAUDE.md, ARCHITECTURE.md, Obsidian 지식 볼트)를 초기 세팅한다. "하네스 세팅", "하네스 초기화", "harness setup" 요청 시 사용.
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
| `T/githooks/pre-commit` | `.githooks/pre-commit` (실행 권한 부여) |
| `T/githooks/secret-files` | `.githooks/secret-files` |
| `T/gitleaks.toml` | `.gitleaks.toml` |
| `T/rules/secret-scan.md` | `.claude/rules/secret-scan.md` |

`secret-files`(비밀 파일 이름 목록)와 `.gitleaks.toml`(비밀값 규칙)은 팀이 규칙과 예외를 더해 쓰는 파일이다. 이미 있으면 "충돌 처리"를 따르되, 덮어쓰면 팀이 추가한 규칙이 사라진다는 점을 함께 알린다. `pre-commit`은 자기와 같은 디렉터리의 `secret-files`를 읽으므로 둘은 항상 같은 디렉터리에 둔다.

복사한 뒤 아래 git 설정을 한다. 이미 다른 값이 설정돼 있으면 값을 보여 주고 바꿀지 묻는다.

- `git config commit.template .gitmessage`
- `git config core.hooksPath .githooks`: 내용 없는 커밋 소제목을 막는 `commit-msg`와 비밀값 커밋을 막는 `pre-commit`을 켠다.
  - `core.hooksPath`가 이미 다른 디렉터리(예: `.husky`)를 가리키면 바꾸지 않는다. 두 훅 각각에 대해, 그 디렉터리에 같은 이름의 훅이 없으면 그곳에 복사할지, 있으면 기존 훅에 호출을 덧붙일지 묻는다. `pre-commit`을 그곳에 복사하면 `secret-files`도 함께 복사하고, 호출을 덧붙이면 `.githooks/pre-commit`을 부르므로 목록은 `.githooks/`에 그대로 둔다.
  - `core.hooksPath`가 없고 `.git/hooks/`에 `.sample`이 아닌 훅이 있으면, 설정하는 순간 그 훅들이 꺼진다. 목록을 보여 주고 `.githooks/`로 옮길지 묻는다. 옮긴 훅은 커밋하면 팀 전체에 공유된다는 점도 함께 알린다(원래는 각자의 로컬에만 있던 훅이다).
  - 연결한 뒤에는 git이 실제로 부르는 훅(`core.hooksPath` 아래 훅)을 직접 실행해 확인한다. 기존 훅 중간에 `exit`가 있으면 덧붙인 호출이 실행되지 않을 수 있기 때문이다.
    - `commit-msg`: 임시 메시지 파일로 실행한다. 빈 소제목이 있는 메시지는 거부되고 정상 메시지는 통과해야 한다.
    - `pre-commit`: 대상 저장소의 스테이징을 건드리지 않도록 `mktemp -d`로 만든 임시 저장소에서 실행한다. 훅·`secret-files`·`.gitleaks.toml`을 복사하고, 아래를 하나씩 스테이징해 실행한다. 내용 있는 `.env`(예: `A=1`)와 빈 `.env`는 거부되고, 평범한 파일만 있으면 통과해야 한다. 빈 `.env`는 파일 이름 검사가 gitleaks와 별개로 동작하는지 확인하는 경우다. 확인이 끝나면 임시 저장소를 지운다.
  - `core.hooksPath`는 저장소마다 각자 설정해야 하는 로컬 설정이다. 마무리 보고에서 다른 팀원도 같은 명령을 실행해야 한다고 알린다.

비밀값 검사를 마저 갖춘다.

- **gitleaks**: 설치 방법과 최소 버전은 `T/rules/secret-scan.md`의 "gitleaks 설치" 절을 따른다. 팀원도 같은 절을 보고 설치하므로 방법을 여기에 따로 적지 않는다.
  1. `gitleaks version`으로 확인한다. 8.25 이상의 버전 번호면 다음으로 간다.
  2. 없으면 비밀 파일 검사는 그대로 동작하지만, 비밀값은 훅의 최소 내장 패턴으로만 검사하고 `.gitleaks.toml`이 적용되지 않는다고 설명한다. 8.25보다 낮거나 버전 번호가 아닌 문구가 나오면(배포판 패키지 빌드) 훅이 실행 오류로 모든 커밋을 막는다고 설명한다. `command -v gitleaks`로 지금 쓰이는 경로도 보여 준다.
  3. 환경을 파악해 위 절의 표에서 방법을 고른다. `uname -s`(Darwin/Linux/MINGW·MSYS), Linux면 `/etc/os-release`의 `ID`·`ID_LIKE`, 그리고 `command -v brew`·`pacman`·`dnf`·`scoop`·`choco`로 쓸 수 있는 패키지 관리자를 본다. 표에서 위에 있는 방법이 먼저다.
  4. 고른 방법과 이유(예: Ubuntu라서 apt 대신 공식 바이너리), 실행할 명령을 보여 주고 `설치 (Recommended)` / `설치 안 함`을 묻는다. `sudo`가 필요한 명령이면 그렇다고 알린다.
  5. 설치가 실패하면 다른 방법으로 자동으로 바꾸지 말고 오류를 보고한 뒤 다음으로 넘어간다. 사용자가 승인한 것은 보여 준 명령뿐이기 때문이다.
  6. 설치한 뒤 `gitleaks version`으로 다시 확인한다. 예전 버전이 계속 잡히면 `command -v gitleaks` 경로를 보여 주고 `PATH` 순서나 예전 패키지 제거를 안내한다. 공식 바이너리로 설치했는데 찾지 못하면 `~/.local/bin`을 `PATH`에 넣으라고 안내한다.
- **`.gitignore`**: `.githooks/secret-files`의 패턴 줄(주석 제외) 중 `.gitignore`에 없는 줄을 보여 주고 추가할지 묻는다(`추가 (Recommended)` / `추가 안 함`). 비밀 파일이 애초에 스테이징되지 않게 하려는 것이다. 추가하면 `# 비밀 파일 (dev-harness)` 주석 아래에 빠진 줄만 파일 끝에 덧붙이고, `.gitignore`가 없으면 만든다. 이미 추적 중인 파일은 `.gitignore`로 빠지지 않으므로, `git ls-files`에 해당 패턴의 파일이 있으면 목록을 보여 주고 직접 확인하라고 알린다.

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
  5. `CLAUDE.local.md`를 만들었다면 `.gitignore`에 `CLAUDE.local.md` 줄이 있는지 확인하고, 없으면 추가한다(`.gitignore`가 없으면 만든다). 개인 메모가 커밋되지 않게 하려는 것이다.

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

### ⑥ codegraph

codegraph는 코드의 심볼·호출 관계·의존성을 로컬 그래프(`.codegraph/codegraph.db`)로 만들고, Claude가 MCP 도구로 그 그래프를 조회하게 한다. 파일을 하나씩 열어 찾는 대신 그래프에서 바로 답을 얻으므로 도구 호출·토큰·시간이 크게 줄어든다. 모두 로컬에서 동작하고 API 키가 필요 없다.

⑤-b까지 끝낸 뒤 진행한다. CLI 층 → 연결 층 → 프로젝트 층 순서로 본다. CLI 층과 연결 층은 기기 전역을 바꾸므로 반드시 묻고 나서 실행한다. 어느 층에서든 사용자가 거절하면 그 뒤 층은 건너뛰고 ⑦로 간다. 연결 없이 인덱스만 만들면 Claude가 그래프를 쓰지 못하기 때문이다.

1. **CLI 층**: `command -v codegraph`로 확인한다. 있으면 다음 층으로 간다.
   - 없으면 아래를 설명한 뒤 `설치 (Recommended)` / `설치 안 함`을 묻는다.
     - 코드베이스를 그래프로 만들어 Claude가 구조를 바로 조회하게 하는 도구다.
     - 도구 호출이 약 88%, 토큰이 약 62%, 작업 시간이 약 53% 줄어든다.
     - 파일이 바뀌면 그래프가 자동으로 갱신되고, 코드는 기기 밖으로 나가지 않는다.
     - 익명 사용 통계가 기본으로 켜져 있다. `codegraph telemetry off`로 끌 수 있다.
   - 설치하면 `command -v npm`이 있을 때 `npm i -g @colbymchenry/codegraph`를, 없을 때 `curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh`를 실행한다. 실행할 명령을 질문에 함께 보여 준다. 설치가 실패하면(예: npm 권한 오류) 다른 방법으로 자동으로 바꾸지 말고 오류를 보고한 뒤 ⑥을 멈추고 ⑦로 간다. 사용자가 승인한 것은 보여 준 명령뿐이기 때문이다.
   - 설치한 뒤 `command -v codegraph`로 다시 확인한다. curl 방식은 새 셸에서만 PATH에 잡힐 수 있으므로, 찾지 못하면 새 터미널을 열고 setup을 다시 실행하라고 안내하고 ⑥을 멈추고 ⑦로 간다.
2. **연결 층**: `claude mcp get codegraph`로 Claude Code에 연결됐는지 본다. 종료 코드가 0이면 연결된 것이므로 다음 층으로 간다(범위와 무관하게 찾는다).
   - 없으면 아래 전역 변경을 모두 알리고 연결할지 묻는다.
     - `~/.claude.json`: codegraph MCP 서버(user 범위)
     - `~/.claude/settings.json`: `mcp__codegraph__*` 자동 허용 권한, 프롬프트마다 실행되는 `UserPromptSubmit` 훅(`codegraph prompt-hook`)
     - `~/.claude/CLAUDE.md`: `<!-- CODEGRAPH_START -->` 표식으로 감싼 사용 안내 섹션. `.codegraph/`가 있는 저장소에서만 codegraph를 쓰라는 내용이다.
     - 되돌리려면 `codegraph uninstall --target=claude --location=global --keep-cli`를 실행한다.
   - 연결하면 `codegraph install --target=claude --location=global --yes`를 실행한다. `--location=local`은 쓰지 않는다. 프로젝트 `CLAUDE.md`에 섹션이 덧붙어 ④에서 설치한 파일이 원본과 달라지기 때문이다.
3. **프로젝트 층**: 저장소 루트에서 `codegraph status --json`을 실행해 `initialized`가 `true`면 이미 세팅된 것이므로 건너뛴다. 명령이 실패하거나 JSON을 읽을 수 없으면 세팅되지 않은 것으로 본다.
   - 개발 프로젝트인지 판정한다. 루트에 매니페스트(`package.json`, `pyproject.toml`, `requirements.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`, `build.gradle.kts`, `Gemfile`, `composer.json`, `Package.swift`, `pubspec.yaml`, `CMakeLists.txt`, `*.csproj`, `*.sln`)가 있거나, `git ls-files`에 소스 파일(`.ts`, `.js`, `.py`, `.go`, `.rs`, `.java`, `.kt`, `.cs`, `.php`, `.rb`, `.swift`, `.c`, `.cpp`, `.scala`, `.dart`, `.vue`, `.svelte` 등)이 있으면 개발 프로젝트로 본다.
   - 개발 프로젝트로 판정되면 감지한 근거를 보여 주고 `세팅 (Recommended)` / `세팅 안 함`으로 적극 권장한다. 이 프로젝트에서 Claude가 코드를 찾을 때마다 위 이점을 얻는다는 점을 강조한다.
   - 판정되지 않아도 건너뛰지 않는다. 코드가 아직 없거나 감지 기준 밖일 수 있기 때문이다. "개발 프로젝트라면 세팅을 권장한다"고 설명하고 `개발 프로젝트다, 세팅한다` / `개발 프로젝트가 아니다(건너뛰기)`를 한 질문으로 묻는다.
   - 세팅하면 저장소 루트에서 `codegraph init --yes`를 실행하고, `codegraph status --json`으로 `initialized: true`와 파일·노드 수를 확인한다.
   - `init`은 `.codegraph/`만 만든다. 그 안의 자체 `.gitignore`가 데이터베이스를 제외하므로 루트 `.gitignore`는 건드리지 않는다.

### ⑦ fluent-korean 출력 스타일

fluent-korean은 Claude가 조사·어미를 빠뜨리지 않고 의미가 분명한 한국어를 쓰게 하는 출력 스타일 플러그인이다. 스타일이 두 가지다.

- `fluent-korean:fluent-korean`: Claude Code의 코딩 지침을 유지한다. 코드를 다루는 프로젝트용이다.
- `fluent-korean:fluent-korean-not-coding`: 코딩 지침을 뺀다. 코드를 고치지 않는 작업용이다.

전역(`~/.claude/settings.json`)에는 not-coding을 두고, 개발 프로젝트에서만 프로젝트 설정으로 코딩용을 덮어쓰는 구성을 권장한다. 설정은 local → project → user 순으로 우선하므로 프로젝트 값이 전역 값을 이긴다. 개발 프로젝트에 전역 not-coding이 그대로 적용되면 코딩 지침이 빠진 채 코드를 고치게 된다.

`outputStyle` 값은 `<플러그인>:<스타일>` 형식이고 대소문자를 구분한다. 스타일 이름과 정확히 맞지 않으면 경고 없이 기본 스타일이 적용되므로 위 두 값을 그대로 쓴다.

⑥까지 끝낸 뒤 진행한다. 플러그인 층 → 전역 층 → 프로젝트 층 순서로 본다. 플러그인 층과 전역 층은 기기 전역을 바꾸므로 반드시 묻고 나서 실행한다. 설정 파일을 바꿀 때는 다른 키를 건드리지 않고 `outputStyle`만 추가하거나 바꾼다. 파일이 없으면 `outputStyle` 하나만 담아 만든다. 파일이 있는데 JSON으로 읽을 수 없으면 고치거나 새로 쓰지 말고 그렇다고 보고한 뒤 그 파일은 건너뛴다. 새로 쓰면 다른 설정이 사라지기 때문이다.

1. **플러그인 층**: `claude plugin list --json`에서 `id`가 `fluent-korean@`으로 시작하는 항목을 찾는다.
   - 없으면 위 설명과 함께 `설치 (Recommended)` / `설치 안 함`을 묻는다. 스타일 지침이 시스템 프롬프트에 붙어 입력 토큰이 늘어난다는 점도 알린다. 설치하면 `claude plugin marketplace list --json`에 `name`이 `fluent-korean`인 마켓이 없을 때만 `claude plugin marketplace add snflkd/fluent-korean`을 실행하고, 이어서 `claude plugin install fluent-korean@fluent-korean --scope user`를 실행한다. 실행할 명령을 질문에 함께 보여 준다.
   - 있는데 `enabled`가 `false`면 `claude plugin enable <id>`로 켤지 묻는다.
   - 설치·활성화를 거절하거나 명령이 실패하면 오류를 보고하고 ⑦을 멈춘다. 플러그인 없이 스타일 값만 넣으면 기본 스타일이 적용될 뿐이기 때문이다.
2. **전역 층**: `~/.claude/settings.json`의 `outputStyle`을 읽는다.
   - `fluent-korean:fluent-korean-not-coding`이면 다음 층으로 간다.
   - 없거나 다른 값이면 현재 값을 보여 주고 `not-coding으로 설정 (Recommended)` / `그대로 두기`를 묻는다. 다른 값이 있었다면, 바꾸면 모든 프로젝트의 기본 응답 스타일이 바뀐다고 알린다.
3. **프로젝트 층**: 개발 프로젝트인지 정한다.
   - ⑥ 프로젝트 층에서 사용자가 개발 프로젝트 여부에 답했거나 `세팅`을 골랐으면 그 결과를 쓴다. codegraph 인덱스가 이미 있고 노드 수(`nodeCount`)가 0보다 컸던 경우도 개발 프로젝트로 본다. 같은 질문을 두 번 하지 않기 위해서다. 자동 감지만 하고 사용자가 `세팅 안 함`을 고른 경우는 확인받은 것이 아니므로 다음 줄을 따른다.
   - 그렇지 않으면 ⑥ 프로젝트 층의 기준(매니페스트·소스 파일)으로 감지한 근거를 보여 주고 `개발 프로젝트다` / `개발 프로젝트가 아니다`를 묻는다. 감지되면 `개발 프로젝트다 (Recommended)`를 첫 선택지로 두고, 감지되지 않으면 권장안 없이 묻는다.
   - 개발 프로젝트가 아니면 프로젝트 설정을 바꾸지 않는다. 전역 값이 적용된다.
   - 개발 프로젝트이면 `.claude/settings.local.json`, `.claude/settings.json` 순으로 처음 나오는 `outputStyle`을 본다. 이 값이 이 프로젝트에 실제로 적용된다. 이미 `fluent-korean:fluent-korean`이면 어느 파일인지 알리고 이 층을 끝낸다.
   - 아니면 두 파일의 현재 `outputStyle`을 보여 주고 `fluent-korean:fluent-korean`을 어디에 넣을지 묻는다.
     - `local (Recommended)`: `.claude/settings.local.json`. 나에게만 적용되고 커밋되지 않는다.
     - `project`: `.claude/settings.json`. 커밋하면 팀 전체에 적용된다. 플러그인이 없는 팀원에게는 경고 없이 기본 스타일이 적용된다.
     - `설정 안 함`
   - `project`를 골랐는데 `.claude/settings.local.json`에 다른 `outputStyle`이 있으면 local 값이 project 값을 가린다고 알리고, local 값을 지울지 묻는다.
   - `local`을 골랐으면 `git check-ignore -q .claude/settings.local.json`으로 커밋에서 빠지는지 확인한다. 빠지지 않으면 `.gitignore`에 `.claude/settings.local.json` 줄을 추가할지 묻는다(`.gitignore`가 없으면 만든다). 개인 설정이 커밋되지 않게 하려는 것이다. 이미 추적 중인 파일은 `.gitignore`로 빠지지 않으므로, `git ls-files`에 이 파일이 있으면 그렇다고 알리고 직접 확인하라고 안내한다.
4. 바꾼 설정 파일을 다시 읽어 `outputStyle` 값이 들어갔고 JSON이 깨지지 않았는지 확인한다.

### ⑧ claude-obsidian 지식 볼트

claude-obsidian(MIT 라이선스)은 Obsidian 볼트를 출처가 남는 위키로 관리하는 플러그인이다. `/claude-obsidian:save`(대화 결과 저장), `/claude-obsidian:wiki-ingest`(자료 정리), `/claude-obsidian:wiki-query`(볼트 검색), `/claude-obsidian:autoresearch`(웹 조사) 같은 스킬을 제공한다. 볼트를 바꾸는 작업은 모두 미리 보기를 거쳐 사용자가 승인한 트랜잭션으로만 적용된다. 대화를 자동으로 기록하거나 정해진 시각에 웹을 조사하는 기능은 의도적으로 없다. 검토를 거치지 않은 내용과 비밀값이 볼트에 쌓이지 않게 하기 위해서다.

여러 프로젝트가 볼트 하나를 함께 쓰는 구성을 권장한다. 그래서 플러그인과 볼트 경로는 기기 전역에 두고, 프로젝트는 태그와 허브 페이지로 구분한다.

⑦까지 끝낸 뒤 진행한다. 플러그인 층 → Python 층 → 볼트 경로 층 → 초기화 층 → 프로젝트 층 순서로 본다. 프로젝트 층을 뺀 나머지는 기기 전역을 바꾸므로 반드시 묻고 나서 실행한다.

1. **플러그인 층**: `claude plugin list --json`에서 `id`가 `claude-obsidian@`으로 시작하고, `scope`가 `user`이거나 `projectPath`가 현재 저장소 루트와 같은 항목을 찾는다. 이 목록에는 다른 프로젝트에만 설치된 항목도 나오므로, 그런 항목은 이 프로젝트에서 쓸 수 없는 것으로 본다. 여러 개가 남으면 `user` 범위를 쓴다.
   - 없으면 위 설명과 함께 아래를 알리고 `설치 (Recommended)` / `설치 안 함`을 묻는다.
     - 모든 프로젝트에서 같은 볼트를 쓰도록 user 범위(전역)로 설치한다.
     - Python 3.11 이상이 필요하다. 다음 층에서 확인한다.
     - 세션 시작 훅은 기본적으로 아무것도 출력하지 않는다. 응답이 끝날 때마다 실행되는 Stop 훅은 볼트의 트랜잭션이나 잠금에 이상이 있을 때만 경고한다. Python이 부족하면 이 훅이 매 응답 끝에 오류를 낼 수 있다.
   - 설치하면 `claude plugin marketplace list --json`에 `name`이 `agricidaniel-claude-obsidian`인 마켓이 없을 때만 `claude plugin marketplace add AgriciDaniel/claude-obsidian`을 실행하고, 이어서 `claude plugin install claude-obsidian@agricidaniel-claude-obsidian --scope user`를 실행한다. 실행할 명령을 질문에 함께 보여 준다.
   - 찾은 항목의 `enabled`가 `false`면 `claude plugin enable <id>`로 켤지 묻는다.
   - 설치·활성화를 거절하거나 명령이 실패하면 오류를 보고하고 ⑧을 멈춘다. 플러그인 없이는 나머지 층이 쓸모없기 때문이다.
   - 새로 설치했든 원래 있었든, 쓸 항목의 `installPath`를 기록한다(설치했다면 `claude plugin list --json`을 다시 읽는다). 초기화 층에서 쓰는 CLI는 `<installPath>/scripts/claude-obsidian.py`다. 아래에서는 이 경로를 `CORE`라고 부른다.
2. **Python 층**: 플러그인의 훅과 CLI는 `PATH`에서 처음 찾은 `python3`로 실행된다. 이 셸의 `PATH`는 Claude Code가 훅을 실행할 때의 `PATH`와 같다고 보고 확인한다.
   1. `command -v python3`와 `python3 --version`으로 경로와 버전을 얻는다. `which -a python3`로 `PATH`에 있는 다른 `python3`도 함께 기록한다.
   2. 버전 지원 상태를 조회한다. `curl -fsSL --max-time 10 https://endoflife.date/api/python.json`의 각 항목은 `cycle`(예: `3.13`), `latest`(그 계열의 최신 패치), `eol`(지원 종료일)을 담는다. 오늘 날짜는 `date +%F`로 얻는다. 아래 순서로 판정한다.
      - 계열이 3.11보다 낮다 → **부족**: 플러그인이 동작하지 않는다.
      - `eol`이 지났다 → **지원 종료**: 보안 수정을 더 받지 못한다.
      - 버전이 `latest`보다 낮다 → **패치 누락**: 보안 수정은 패치 버전으로 나오므로, 이미 공개된 취약점이 고쳐지지 않았을 수 있다.
      - 모두 아니면 **정상**이다. 이 경로를 `PY`로 기록하고 다음 층으로 간다.
      - 조회에 실패하면 취약 여부는 판단할 수 없다고 알리고 3.11 이상인지만 본다.
   3. 지금 `python3`가 정상이 아니면 `which -a python3`의 나머지 후보도 같은 기준으로 판정한다. 정상인 후보가 있으면 새로 설치하지 않는다. 그 후보의 경로와 판정을 보여 주고 6번의 PATH 수정을 제안한다. 이미 쓸 수 있는 Python이 `PATH` 뒤쪽에 있을 뿐인데 새로 설치하면 Python만 하나 더 늘고 문제는 그대로 남기 때문이다. 정상인 후보가 없을 때만 4번부터 진행한다.
   4. 설치할 버전을 정한다.
      - 패치 누락이면 같은 계열의 `latest`로 올린다. 계열이 바뀌지 않아 다른 프로젝트에 주는 영향이 가장 작다.
      - 부족·지원 종료이면 `eol`이 지나지 않은 3.11 이상 계열 중 두 번째로 높은 계열의 `latest`로 정한다. 그런 계열이 하나뿐이면 그 계열을 쓴다. 가장 높은 계열은 막 나와서 관리 도구에 아직 없을 수 있기 때문이다.
      - 조회에 실패했다면 버전을 정할 수 없으므로 설치를 제안하지 않는다. 3.11 이상이며 지원 중인 계열의 최신 패치를 설치하라고 안내하고 넘어간다.
   5. 쓸 관리 도구를 고른다. `command -v pyenv`·`mise`·`asdf`·`uv`·`brew`로 설치된 도구를 보고, 지금 `python3`를 어느 도구가 관리하는지 판단한다.
      - 먼저 `command -v python3`가 돌려준 경로 그대로에 `.pyenv/shims`, `mise/shims`, `.asdf/shims`가 있는지 본다. 있으면 그 도구다. shim은 관리 도구 실행 파일로 이어지는 링크일 수 있어서, 링크를 따라가면 도구를 알아볼 수 없게 되기 때문이다.
      - shim이 아니면 `realpath "$(command -v python3)"`로 얻은 실제 경로를 아래 표에 위에서부터 대어 보고, 처음 맞는 행으로 판단한다. 링크를 따라가야 Homebrew나 uv가 만든 링크를 구별할 수 있기 때문이다.

      | 실제 경로에 포함된 것 | 관리 주체 |
      | --- | --- |
      | `.pyenv/` | pyenv |
      | `mise/` | mise |
      | `.asdf/` | asdf |
      | `uv/python/` | uv |
      | `/Cellar/` 또는 `/opt/homebrew/` | Homebrew |
      | `/Library/Frameworks/Python.framework/` | python.org 설치 파일 |
      | `/usr/bin/`, `/Library/Developer/CommandLineTools/`, `/Applications/Xcode.app/` | 시스템 Python |

      지금 `python3`를 관리하는 도구를 먼저 쓴다. 이미 쓰는 도구로 버전을 바꿔야 PATH와 설정이 서로 충돌하지 않기 때문이다. 시스템 Python, python.org 설치, 판단할 수 없는 경로이면 설치된 도구 중 pyenv, mise, asdf, uv, Homebrew 순서로 고른다. 시스템 Python 자체는 고치지 않는다. 운영 체제가 관리하는 파일이기 때문이다.

      | 도구 | 명령 |
      | --- | --- |
      | pyenv | `pyenv install <버전>` 후 `pyenv global <버전>`. `pyenv install --list`에 버전이 없으면 정의 목록이 낡은 것이므로 먼저 `brew upgrade pyenv`(Homebrew 설치) 또는 `pyenv update`를 제안한다. 빌드에 필요한 의존성이 없으면 설치가 실패할 수 있다고 알린다. |
      | mise | `mise use -g python@<버전>` |
      | asdf | `asdf plugin list`에 `python`이 없으면 먼저 `asdf plugin add python`. 이어서 `asdf install python <버전>`, 0.16 이상이면 `asdf set -u python <버전>`, 그 미만이면 `asdf global python <버전>` (`asdf --version`으로 확인) |
      | uv | `uv python install <버전> --default` (`~/.local/bin`에 `python3`를 만든다). `uv python install --help`에 `--default`가 없는 오래된 uv면 `uv self update`(Homebrew 설치는 `brew upgrade uv`)를 먼저 제안한다. |
      | Homebrew | 그 계열이 이미 설치돼 있으면 `brew upgrade python@<계열>`, 아니면 `brew install python@<계열>`. 버전을 패치 단위로 고를 수 없어 Homebrew에 반영된 최신 패치가 설치되므로, 설치 뒤에도 패치 누락이 남을 수 있다. 또 기본 계열이 아니면 `python3` 이름을 연결하지 않으므로, 설치 뒤 `python3`가 바뀌지 않으면 7번 안내를 따른다. |

      어느 도구도 없으면 설치하지 않고 python.org 설치 파일이나 위 도구 중 하나를 쓰라고 안내한 뒤 넘어간다.
   6. 현재 경로·버전, 판정과 근거(지원 종료일, 최신 패치), 설치할 버전과 고른 이유, 관리 도구와 고른 이유, 실행할 명령을 보여 주고 묻는다. 기기 전역의 기본 Python이 바뀌어 다른 프로젝트에도 영향이 간다는 점도 알린다. 부족이면 `설치 (Recommended)` / `설치 안 함`, 지원 종료·패치 누락이면 `업데이트 (Recommended)` / `그대로 두기`로 묻는다. 설치가 실패하면 다른 도구로 자동으로 바꾸지 말고 오류를 보고한다. 사용자가 승인한 것은 보여 준 명령뿐이기 때문이다.
   7. 설치한 뒤 `command -v python3`와 `python3 --version`을 다시 본다. 정상이면 그 경로를 `PY`로 기록한다. 버전이 그대로면 설치한 Python의 경로를 찾아 `PY`로 기록하고(예: uv는 `~/.local/bin/python3`, Homebrew는 `/opt/homebrew/bin/python3.<계열>`) 원인을 찾는다.
      - 저장소에 `.python-version`이나 `.tool-versions`가 있으면 그 파일이 전역 설정을 가린다. 파일 내용을 보여 주고, 이 프로젝트의 Python 버전이 정해져 있다는 뜻이므로 사용자가 직접 판단하게 한다.
      - 그렇지 않으면 아래 PATH 수정을 제안한다.
   8. **PATH 수정**: 3번이나 7번에서 정상인 Python이 `PATH` 뒤쪽에 있거나 `PATH`에 없을 때 실행한다.
      - `which -a python3`로 지금 순서를 보여 준다. 셸 설정에 추가할 줄은 `export PATH="<PY가 있는 디렉터리>:$PATH"`다. 홈 디렉터리는 `$HOME`으로 쓴다.
      - `basename "$SHELL"`이 `zsh`면 `~/.zshrc`, `bash`면 `~/.bashrc` 끝에 덧붙인다. 끝에 두어야 앞쪽에서 `PATH`를 바꾸는 다른 줄보다 우선한다. 그 밖의 셸은 줄만 보여 주고 직접 넣으라고 안내한다.
      - 이 디렉터리에 `python3` 말고 다른 실행 파일이 있으면 목록을 보여 준다. 같은 이름의 시스템 명령보다 먼저 잡히게 되기 때문이다.
      - 추가할 파일과 줄을 보여 주고 `추가 (Recommended)` / `직접 고치기`를 묻는다. 추가했으면 `zsh -i -c 'command -v python3; python3 --version'`(bash면 `bash -i -c`)로 새 셸에서 바뀌었는지 확인한다.
      - Claude Code는 실행될 때의 `PATH`를 훅에 넘기므로, 새 터미널에서 Claude Code를 다시 시작해야 반영된다고 알린다. 이번 setup 안에서는 아래 초기화 층이 `PY`를 직접 쓰므로 계속 진행할 수 있다.
   - 끝까지 `PY`를 정하지 못했으면(거절, 실패, 정상인 Python 없음) 다음 층은 진행하되 초기화 층은 건너뛴다. CLI를 실행할 수 없기 때문이다. 이 경우 플러그인을 꺼 둘지 묻는다(`끄기 (Recommended)` / `켜 두기`). 켜 두면 Stop 훅이 모든 세션에서 응답마다 오류를 내기 때문이다. 끄면 `claude plugin disable claude-obsidian@agricidaniel-claude-obsidian`을 실행하고, Python을 고친 뒤 `claude plugin enable`로 다시 켜거나 setup을 다시 실행하라고 안내한다.
3. **볼트 경로 층**: `~/.claude/settings.json`의 `env.CLAUDE_OBSIDIAN_VAULT`를 읽는다. 플러그인은 이 환경 변수를 프로젝트의 `.claude-obsidian.json`보다 먼저 보므로, 여기에 경로를 두면 모든 프로젝트가 같은 볼트를 쓴다.
   - 이 층에서 다루는 모든 경로는, 디렉터리가 있으면 `ls -A "<경로>"`로 실제로 읽을 수 있는지도 본다. 디렉터리가 있는지만 보면 macOS 개인정보 보호가 막은 폴더를 놓치기 때문이다. `Operation not permitted`가 나오면 Claude Code의 샌드박스가 아니라 macOS가 이 터미널 앱의 접근을 막은 것이다. 이런 경로는 볼트 작업이 모두 실패하므로, 시스템 설정 → 개인정보 보호 및 보안 → 전체 디스크 접근 권한(또는 파일 및 폴더 → iCloud Drive)에서 Claude Code를 실행하는 터미널 앱을 허용하고 터미널을 완전히 종료한 뒤 다시 열라고 안내한다.
   - 경로에 `Mobile Documents`, `CloudStorage`, `Dropbox`, `OneDrive`, `Google Drive`가 들어 있으면 동기화 폴더라고 알린다. claude-obsidian은 동기화 폴더에서의 안전성을 보장하지 않으므로, 동기화가 끝난 뒤 한 기기에서만 쓰라고 함께 알린다.
   - 값이 있고 그 디렉터리를 읽을 수 있으면 경로를 알리고 다음 층으로 간다. 읽을 수 없으면 위 안내와 함께 `권한을 고친 뒤 setup 다시 실행 (Recommended)` / `다른 경로 고르기`를 묻는다. 앞의 것을 고르면 값은 그대로 두고 초기화 층을 건너뛴다.
   - 값이 있는데 디렉터리가 없으면 그렇다고 알리고 `이 경로에 새로 만들기 (Recommended)`와 아래 후보·새 경로를 함께 보여 준다. 새로 만들기를 고르면 값은 그대로 두고 초기화 층에서 만든다.
   - 값이 없으면 Obsidian의 볼트 목록에서 후보를 찾는다. 목록 파일은 macOS `~/Library/Application Support/obsidian/obsidian.json`, Linux `~/.config/obsidian/obsidian.json`, Windows `%APPDATA%\obsidian\obsidian.json`이고, `vaults` 아래 각 항목의 `path`가 볼트 경로다. 실제로 있는 디렉터리만 후보로 삼는다.
   - 후보를 선택지로 보여 주고(가장 최근 `ts` 순으로 최대 3개) 새 경로는 직접 입력하게 한다. 읽을 수 없는 후보와 동기화 폴더 후보는 설명에 그렇다고 적는다. `(Recommended)`는 읽을 수 있는 후보 중 가장 최근 것에만 붙이고, 그런 후보가 없으면 붙이지 않는다. 읽을 수 없는 경로를 고르면 위 안내를 하고, 경로는 기록하되 초기화 층은 건너뛴다. 후보가 없으면 새 경로를 입력받는다(예: `~/Documents/Obsidian/knowledge`). 새 경로는 아직 없는 디렉터리여도 된다. 초기화 층에서 만든다.
   - 입력한 경로의 `~`는 홈 디렉터리로 풀어 절대 경로로 바꾼다. 경로가 지금 프로젝트의 git 저장소 안에 있으면 노트가 프로젝트에 커밋될 수 있다고 알리고 다시 묻는다.
   - 고른 경로를 `env.CLAUDE_OBSIDIAN_VAULT`에 넣는다. ⑦과 같이 다른 키는 건드리지 않는다. 파일이 없으면 `env` 하나만 담아 만들고, `env`가 없으면 추가한다. 파일이 JSON으로 읽을 수 없으면 고치지 말고 보고한 뒤 경로를 정하지 않은 것으로 본다. 바꾼 뒤 파일을 다시 읽어 값이 들어갔고 JSON이 깨지지 않았는지 확인한다. 아래를 함께 알린다.
     - 환경 변수는 새 세션부터 적용된다.
     - 특정 프로젝트만 다른 볼트를 쓰려면 그 프로젝트의 `.claude/settings.local.json`에서 `env.CLAUDE_OBSIDIAN_VAULT`를 덮어쓴다.
     - 세션 시작 때 볼트의 최근 맥락(`wiki/hot.md`)을 주입하는 `CLAUDE_OBSIDIAN_SESSION_CONTEXT`는 설정하지 않는다. 볼트 하나를 여러 프로젝트가 쓰므로 다른 프로젝트의 맥락이 섞이기 때문이다.
   - 이 프로젝트의 `.claude/settings.local.json`이나 `.claude/settings.json`에 다른 `env.CLAUDE_OBSIDIAN_VAULT`가 있으면, 이 프로젝트에서는 그 값이 전역 값을 가린다고 알린다.
   - 경로를 정하지 않으면 초기화 층을 건너뛴다.
4. **초기화 층**: 볼트 경로 층에서 정한 경로를 쓴다. 이 세션에는 아직 환경 변수가 없으므로 경로를 인자로 넘긴다. CLI는 `python3` 대신 Python 층에서 정한 `PY`로 실행한다. PATH 수정은 Claude Code를 다시 시작해야 반영되므로, `python3`를 쓰면 이번 실행에서는 여전히 낮은 버전이 잡힐 수 있기 때문이다. 아래 명령은 모두 같은 흐름을 따른다.
   - 먼저 `--apply` 없이 미리 보기를 실행한다. 출력 JSON의 `status`가 `noop`이면 바꿀 것이 없으므로 다음 층으로 간다. `dry-run`이면 `changed_paths`(만들거나 바꿀 파일)를 보여 주고 적용할지 묻는다.
   - 적용은 같은 명령에 `--generated-at <미리 보기의 generated_at> --approved-plan-sha256 <미리 보기의 approved_plan_sha256> --apply`를 붙여 실행한다. 승인 해시는 시각에 묶여 있으므로 미리 보기의 `generated_at`을 그대로 넘겨야 한다.
   - `--force`는 쓰지 않는다. 기존 파일을 바꾸게 되기 때문이다. 오류가 나면 그대로 보고하고 다음 층으로 간다.
   - 적용한 뒤 `"$PY" "$CORE" doctor --vault <경로>`로 볼트를 확인한다.

   경로 상태에 따라 명령을 고른다.

   - `.claude-obsidian.json`과 `wiki/`가 모두 있으면 이미 초기화된 볼트다. 알리고 다음 층으로 간다.
   - `wiki/`는 있는데 `.claude-obsidian.json`이 없으면 예전 형식의 볼트다. 출처 원장과 볼트 설정을 더하는 이전이 필요하다고 설명하고 `이전 (Recommended)` / `이전 안 함`을 묻는다. 이전 명령은 `"$PY" "$CORE" migrate --vault <경로>`다. 기존 노트는 바꾸지 않는다.
   - 경로가 없거나 빈 디렉터리이면 `init`을 쓴다. 명령은 `"$PY" "$CORE" init <경로>`다. `init`은 경로의 상위 디렉터리가 있어야 하므로, 없으면 `mkdir -p <상위 디렉터리>`를 실행할지 함께 묻는다.
   - 그 밖에(기존 Obsidian 볼트나 내용이 있는 폴더) `adopt`를 쓴다. 명령은 `"$PY" "$CORE" adopt <경로>`다. `init`은 기존 파일과 겹치면 멈추지만 `adopt`는 내용이 다른 기존 파일을 건너뛰기 때문이다.
   - `init`·`adopt`는 묻기 전에 무엇이 생기는지 설명하고 `초기화 (Recommended)` / `초기화 안 함`을 묻는다. 생기는 것은 `wiki/` 아래 색인·로그·요약·개요 페이지와 출처·주장 원장(`wiki/meta/ledgers/`), `inbox/`, `.raw/`, `.claude-obsidian.json`, `.gitignore`, 그리고 Obsidian 설정 파일(`.obsidian/` 아래 앱·화면·그래프 설정과 색상 스니펫)이다. 기존 볼트에서는 같은 이름의 파일을 바꾸지 않는다.
5. **프로젝트 층**: 이 프로젝트의 노트를 구분하는 규칙을 설치한다. 볼트의 폴더는 노트 유형으로만 나뉘므로, 규칙이 없으면 프로젝트 사이에 노트가 섞인다. 이 규칙은 내 볼트를 위한 개인 설정이므로 저장소에 커밋하지 않는다.
   1. 프로젝트 이름을 정한다. `.claude/rules/obsidian-vault.md`가 이미 있으면 첫 줄 `<!-- obsidian-vault: <이름> -->` 표식의 이름을 알리고 유지할지 묻는다. 없으면 `basename "$(git rev-parse --show-toplevel)"`에서 영문자·숫자·`-`·`_`가 아닌 문자를 `-`로 바꾼 값을 기본값으로 `<기본값> (Recommended)`를 보여 주고, 다른 이름은 직접 입력하게 한다. 바꾼 결과가 `-`와 `_`뿐이면 기본값 없이 입력받는다.
   2. 이름은 영문자, 숫자, `-`, `_`만 쓴다. Obsidian 태그와 파일 이름에 그대로 쓰이기 때문이다. 입력한 이름에 다른 문자가 있으면 다시 묻는다.
   3. 볼트 경로를 알고 있고 `<볼트>/wiki/entities/<이름>.md`가 이미 있으면 첫 몇 줄을 보여 주고 같은 프로젝트인지 묻는다. 디렉터리 이름이 같은 다른 저장소의 허브일 수 있기 때문이다. 다른 프로젝트라면 새 이름을 받는다.
   4. `T/rules/obsidian-vault.md`의 `<프로젝트>`를 모두 이름으로 바꿔 `.claude/rules/obsidian-vault.md`로 설치한다. 첫 줄 표식은 지우지 않는다. `doctor`가 이 표식을 읽는다. 파일이 이미 있으면 이름을 바꾼 템플릿과 비교해 "충돌 처리"를 따른다.
   5. `git rev-parse --git-path info/exclude`로 얻은 파일에 `/.claude/rules/obsidian-vault.md` 줄이 없으면 추가한다. 앞의 `/`는 저장소 루트의 그 파일만 가리키게 한다. 파일이나 상위 `info/` 디렉터리가 없으면 만든다. `.gitignore`와 달리 이 파일은 커밋되지 않으므로, 팀 저장소에 흔적을 남기지 않고 이 기기에서만 커밋을 막는다. 추가한 뒤 `git check-ignore -q .claude/rules/obsidian-vault.md`로 확인한다. `git ls-files`에 이미 이 파일이 있으면 제외 설정이 적용되지 않는다고 알리고 직접 확인하라고 안내한다.

## 마무리

1. `git status --short`로 바뀐 파일을 보여 주고, 설치한 것과 건너뛴 것을 나눠 보고한다.
2. 비밀값 검사를 보고한다. 다른 팀원도 각자 `.claude/rules/secret-scan.md`의 "gitleaks 설치"를 따라 설치해야 `.gitleaks.toml`이 적용된다고 알린다. 설치하지 않은 팀원은 커밋할 때마다 훅이 같은 안내를 보여 준다.
3. `ARCHITECTURE.md`를 설치했다면 조감도·코드맵을 채워야 하고 맨 위 `마지막 점검` 날짜도 직접 적어야 한다고 알린다.
4. ⑥에서 한 일을 보고한다.
   - Claude Code를 새로 연결했다면 Claude Code를 다시 시작해야 codegraph MCP 서버가 로드된다고 알린다.
   - 프로젝트를 세팅했다면 커밋할 파일은 `.codegraph/.gitignore` 하나뿐이고, 인덱스 데이터베이스는 공유되지 않으므로 다른 팀원도 각자 `codegraph init`을 실행해야 인덱스가 생긴다고 알린다.
5. ⑦에서 바꾼 설정 파일과 값을 보고한다. 출력 스타일은 새 세션부터 적용되고, `/output-style`로 현재 스타일을 확인할 수 있다고 알린다. 출력 스타일은 메인 대화에만 적용되고 서브에이전트에는 적용되지 않는다는 점도 알린다.
6. ⑧에서 한 일을 층별로 보고한다.
   - 플러그인을 설치했거나 볼트 경로를 바꿨다면 Claude Code를 다시 시작해야 스킬과 환경 변수가 적용된다고 알린다.
   - Python 층에서 셸 설정에 PATH 줄을 추가했다면, 새 터미널에서 Claude Code를 다시 시작해야 훅이 새 Python으로 실행된다고 알린다.
   - Python이 부족한 상태로 남았다면 플러그인이 동작하지 않는다는 점을 다시 알린다. 플러그인을 껐다면 다시 켜는 방법도 알린다.
   - 볼트 경로를 정했는데 그 디렉터리가 아직 없으면(초기화를 거절했거나 실패한 경우) 그렇다고 알린다. 이 상태에서는 볼트 작업이 모두 실패하고 doctor도 `FAIL`로 보고하므로, setup을 다시 실행해 초기화해야 한다.
   - 프로젝트 규칙은 커밋되지 않으므로, 다른 기기에서 이 저장소를 받으면 setup을 다시 실행해야 한다고 알린다.
   - 저장은 자동으로 되지 않으며, 남기고 싶은 결과가 나오면 `/claude-obsidian:save`를 직접 실행한다고 안내한다.
7. `/dev-harness:doctor`로 설치 상태를 점검할 수 있다고 안내한다.
