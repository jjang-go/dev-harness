---
name: doctor
description: 현재 프로젝트에 설치된 dev-harness 상태를 점검하고 문제를 보고한다. "하네스 점검", "하네스 진단", "harness doctor" 요청 시 사용.
allowed-tools: Read Glob Grep Bash(git config *) Bash(git rev-parse *) Bash(git remote *) Bash(date *) Bash(ls *) Bash(command -v codegraph) Bash(claude mcp get codegraph) Bash(codegraph status --json) Bash(command -v gitleaks) Bash(gitleaks version) Bash(claude plugin list --json) Bash(git ls-files *) Bash(command -v python3) Bash(python3 --version) Bash(which -a python3) Bash(curl -fsSL --max-time 10 https://endoflife.date/api/python.json) Bash(git check-ignore *)
---

# dev-harness doctor

현재 프로젝트의 하네스 설치 상태를 점검한다. **읽기 전용**이다. 파일을 고치지 않고 결과만 보고한다.

- 템플릿 원본은 `${CLAUDE_PLUGIN_ROOT}/templates/`다. 아래 `T/`는 이 경로를 줄여 쓴 것이다.
- 점검 대상은 현재 작업 디렉터리의 git 저장소 루트다. 저장소가 아니면 그렇다고 보고하고 중단한다.
- 각 항목은 `OK` / `WARN` / `FAIL` / `INFO` 중 하나로 판정한다.
  - `FAIL`: 하네스가 동작하지 않는 상태 (표식 없음, 필수 파일 없음, 설정 불일치)
  - `WARN`: 동작하지만 확인이 필요한 상태 (원본과 다름, 낡음)
  - `INFO`: 선택 설치라 없어도 정상인 상태
- 문제가 있으면 항목마다 고치는 방법을 한 줄로 붙인다. 설치가 빠진 경우는 `/dev-harness:setup`을 안내한다.
- 고치는 방법은 대상 프로젝트 안에서 바로 할 수 있는 행동으로 쓴다. 플러그인 내부 경로(`templates/...`)는 대상 프로젝트에 없으므로 가리키지 않는다. 템플릿 내용이 필요하면 필요한 부분(예: 섹션 제목 목록)을 보고에 직접 보여 준다.

## 원본 비교

설치된 파일과 원본의 대응은 setup 스킬의 설치 표(`${CLAUDE_PLUGIN_ROOT}/skills/setup/SKILL.md`)를 따른다. 대응을 여기에 따로 적지 않는다. 표가 바뀌면 점검도 함께 따라가게 하기 위해서다.

- 설치된 파일은 모두 원본과 비교한다. 같으면 `OK`, 다르면 `WARN`으로 차이를 한두 줄로 요약한다.
- 다르다는 것은 팀이 의도해서 고쳤거나, 플러그인이 갱신되어 설치본이 낡았다는 뜻이다. 어느 쪽인지 단정하지 말고 둘 다 가능하다고 적는다. 원본을 받으려면 `/dev-harness:setup`을 다시 실행하면 된다(다른 파일은 덮어쓰기 전에 묻는다).
- 채워 쓰도록 만든 파일인 `ARCHITECTURE.md`와, 팀이 규칙·예외를 더해 쓰는 `secret-files`·`.gitleaks.toml`은 비교하지 않는다. 뒤의 둘은 8번에서 핵심 설정만 본다.

## 점검 항목

### 1. 커밋 템플릿

- `git config --get commit.template`이 `.gitmessage`인지 본다. 다르거나 없으면 `FAIL`.
- 루트 `.gitmessage`가 없으면 `FAIL`.
- `git config --get core.hooksPath`가 가리키는 디렉터리에 실행 가능한 `commit-msg`가 있는지 본다. 설정이 없거나 훅이 없으면 `WARN`(빈 소제목을 막지 못함). 실행 권한이 없으면 `FAIL`(훅이 조용히 무시됨).

### 2. 브랜치 전략

- `.claude/rules/branch-strategy.md`가 없으면 `FAIL`.
- 첫 줄이 `<!-- branch-strategy: <이름> -->` 형태이고 `<이름>`이 `gitflow` / `githubflow` / `gitlabflow` / `trunk-based` 중 하나인지 본다. 아니면 `FAIL`(표식을 지웠거나 알 수 없는 전략). 표식이 유효하면 전략 이름을 `OK`에 적고, `T/branching/<이름>/branch-strategy.md`와 비교한다.

### 3. CLAUDE.md

- 루트 `CLAUDE.md`가 없으면 `FAIL`.
- 첫 줄이 `<!-- claude-md: karpathy-guidelines -->`가 아니면 `WARN`(하네스가 설치한 CLAUDE.md가 아님).

### 4. ARCHITECTURE.md

- 루트 `ARCHITECTURE.md`와 `.claude/rules/architecture-doc.md`가 **둘 다 없으면** `INFO`(선택 설치). 이 항목의 나머지는 건너뛴다.
- 한쪽만 있으면 `WARN`(짝이 맞지 않음).
- `마지막 점검: YYYY-MM-DD` 줄을 찾는다. 없거나 `YYYY-MM-DD` 그대로이면 `WARN`(템플릿 상태). 날짜가 1년 넘게 지났으면 `WARN`. 오늘 날짜는 `date +%F`로 얻는다.
- `## 코드맵` 섹션의 `###` 제목과 본문 백틱 이름(`` `이름` ``)을 모은다. 각 이름을 `Grep`/`Glob`으로 저장소에서 찾고, 하나도 없는 이름을 `WARN`으로 나열한다. `<`로 시작하는 이름은 템플릿 자리표시자이므로 `WARN`(채워지지 않음)으로 한 번만 보고한다.

### 5. 규칙과 템플릿 존재

아래 파일이 모두 있는지 본다. 없으면 `FAIL`.

- `.claude/rules/commit-message.md`
- `.claude/rules/readme-guidelines.md`
- `.claude/rules/pr-issue-guidelines.md`
- `.claude/rules/secret-scan.md`
- `docs/specs/TEMPLATE.md`

플랫폼 파일은 `.github/` 또는 `.gitlab/` 중 어느 쪽이 있는지로 판단한다.

- `.github/`가 있으면 `PULL_REQUEST_TEMPLATE.md`, `ISSUE_TEMPLATE/bug_report.yml`, `ISSUE_TEMPLATE/feature_request.yml`이 있는지 본다.
- `.gitlab/`이 있으면 `merge_request_templates/Default.md`, `issue_templates/Bug.md`, `issue_templates/Feature.md`가 있는지 본다.
- 둘 다 없으면 `FAIL`. 둘 다 있으면 `INFO`(양쪽 설치됨).
- `git remote get-url origin`의 호스트와 설치된 플랫폼이 어긋나면 `WARN`.

### 6. 서드파티 고지

- 루트 `THIRD_PARTY_NOTICES.md`가 있는지 본다. 없으면 `INFO`(내부 프로젝트라면 정상. 외부에 공개·배포하는 프로젝트이고 CLAUDE.md가 Karpathy 지침이면 `WARN`).

### 7. codegraph

선택 설치다. setup ⑥의 세 층을 차례로 보고, 층마다 한 줄로 보고하며 각각 합계에 센다.

- CLI: `command -v codegraph`가 없으면 `INFO`(설치 안 함). 이 항목의 나머지는 건너뛴다.
- 연결: `claude mcp get codegraph`의 종료 코드가 0이 아니면 `WARN`(Claude가 그래프를 쓰지 못함). 명령 자체를 실행할 수 없으면 `INFO`(확인 불가). 해결: `/dev-harness:setup`을 다시 실행하면 전역 변경 내용을 설명하고 연결한다.
- 인덱스: `codegraph status --json`의 `initialized`가 `false`면 `INFO`(이 프로젝트는 인덱싱 안 함). 해결: `codegraph init`.
  - 인덱싱돼 있으면 `fileCount`와 `lastIndexed`를 `OK`에 적는다.
  - `nodeCount`가 0이면 `WARN`(심볼이 없어 그래프가 비어 있음). `fileCount`는 YAML·설정 파일도 세므로 기준으로 쓰지 않는다. 해결: 소스 코드가 생긴 뒤 `codegraph index --force`로 다시 인덱싱하고, 개발 프로젝트가 아니면 `codegraph uninit`으로 지운다.
  - `index.reindexRecommended`가 `true`면 `WARN`. 해결: `codegraph index --force`.

### 8. 비밀값 커밋 방지

층마다 한 줄로 보고하며 각각 합계에 센다.

- 훅: `git config --get core.hooksPath`가 가리키는 디렉터리에 실행 가능한 `pre-commit`이 있는지 본다. 설정이 없거나 훅이 없으면 `FAIL`(비밀값 커밋을 막지 못함). 실행 권한이 없으면 `FAIL`(훅이 조용히 무시됨).
- 비밀 파일 목록: 하네스 `pre-commit`과 같은 디렉터리에서 `secret-files`를 찾는다. `core.hooksPath`가 `.githooks`가 아니고 그 아래 `pre-commit`이 `.githooks/pre-commit`을 부른다면(setup이 기존 훅에 호출을 덧붙인 경우) `.githooks/`에서 찾는다. 없으면 `FAIL`(훅이 비밀 파일 이름 검사를 건너뜀). 있으면 `.env` 줄이 있는지 보고, 없으면 `WARN`(가장 흔한 비밀 파일을 막지 못함. 팀이 의도했을 수도 있다).
- 비밀값 설정: 루트 `.gitleaks.toml`이 없으면 `INFO`(gitleaks 기본 규칙만 적용됨). 있는데 `useDefault = true`가 없으면 `WARN`(gitleaks 기본 규칙이 꺼지고 이 파일의 규칙만 적용됨. 팀이 의도했을 수도 있다).
- gitleaks: `gitleaks version`을 본다. 해결 방법은 모두 `.claude/rules/secret-scan.md`의 "gitleaks 설치" 절을 따르라는 것이다.
  - 없으면 `WARN`(비밀값을 최소 내장 패턴으로만 검사하고 `.gitleaks.toml`이 적용되지 않음).
  - 8.25보다 낮거나 버전 번호가 아닌 문구가 나오면 `FAIL`(훅이 실행 오류로 모든 커밋을 막음). `command -v gitleaks` 경로를 함께 적는다.
  - 8.25 이상이면 버전을 `OK`에 적는다.
- `.gitignore`: `secret-files`의 패턴 줄(주석 제외) 중 `.gitignore`에 없는 줄이 있으면 `INFO`로 나열한다(선택 설치. 훅이 커밋은 막는다).

### 9. fluent-korean 출력 스타일

선택 설치다. 하네스 동작을 막지 않는 개인 환경 설정이므로 `FAIL`은 쓰지 않는다. setup ⑦의 세 층을 차례로 보고, 층마다 한 줄로 보고하며 각각 합계에 센다. 권장 값은 전역 `fluent-korean:fluent-korean-not-coding`, 개발 프로젝트 `fluent-korean:fluent-korean`이다.

- 아래 세 파일(`~/.claude/settings.json`, `.claude/settings.local.json`, `.claude/settings.json`) 중 JSON으로 읽을 수 없는 파일이 있으면 해당 층을 `WARN`(Claude Code가 이 파일의 설정을 적용하지 못함)으로 보고한다.
- 플러그인: `claude plugin list --json`에서 `id`가 `fluent-korean@`으로 시작하는 항목을 찾는다. 명령 자체를 실행할 수 없으면 `INFO`(확인 불가), 항목이 없으면 `INFO`(설치 안 함)로 보고하고 이 항목의 나머지는 건너뛴다. `enabled`가 `false`면 `WARN`(스타일을 골라도 적용되지 않음). 해결: `claude plugin enable <id>`.
- 전역: `~/.claude/settings.json`의 `outputStyle`을 본다.
  - `fluent-korean:fluent-korean-not-coding`이면 `OK`.
  - 없으면 `WARN`(fluent-korean이 적용되지 않음). 다른 값이면 값을 적어 `WARN`(사용자가 의도했을 수도 있다).
  - 해결: `/dev-harness:setup`을 다시 실행하거나 `"outputStyle": "fluent-korean:fluent-korean-not-coding"`을 넣는다.
- 프로젝트: `.claude/settings.local.json`, `.claude/settings.json` 순으로 처음 나오는 `outputStyle`이 이 프로젝트에 실제로 적용되는 값이다. 없으면 전역 값이 적용된다. doctor는 묻지 않으므로 개발 프로젝트 여부는 setup ⑥ 프로젝트 층의 기준(매니페스트·소스 파일)으로 감지한다. 7번에서 codegraph 인덱스가 있고 `nodeCount`가 0보다 크면 이것도 개발 프로젝트로 본다. setup과 판정을 맞추기 위해서다.
  - 개발 프로젝트로 감지되고 적용 값이 `fluent-korean:fluent-korean`이면 어느 파일에서 왔는지 적어 `OK`. 아니면 `WARN`(코딩 지침이 빠진 스타일이나 기본 스타일이 적용됨. setup에서 개발 프로젝트가 아니라고 답했다면 무시해도 된다). 해결: `.claude/settings.local.json`에 `"outputStyle": "fluent-korean:fluent-korean"`을 넣는다.
  - 감지되지 않으면 적용 값을 적어 `INFO`(개발 프로젝트가 아니면 전역 값이 맞다. 코드가 감지 기준 밖에 있을 수도 있다).
- 위 세 파일 중 `outputStyle`이 대소문자를 무시하고 `fluent-korean`을 포함하는데 두 권장 값 중 어느 것과도 정확히 같지 않으면(대소문자, `fluent-korean:` 접두어 누락 등) 그 파일이 속한 층(전역 또는 프로젝트)의 줄을 `WARN`으로 하고 이유를 덧붙인다. 줄을 따로 만들거나 따로 세지 않는다. 이름이 정확히 맞지 않으면 경고 없이 기본 스타일이 적용되기 때문이다.

### 10. claude-obsidian 지식 볼트

선택 설치다. setup ⑧의 층을 차례로 보고, 층마다 한 줄로 보고하며 각각 합계에 센다. 설치하지 않았으면 `INFO`로 끝나지만, 설치했는데 동작하지 않거나 점검할 수 없는 상태(Python 부족, 볼트 경로의 디렉터리 없음, 규칙 표식 손상)는 7·9번과 달리 `FAIL`로 본다. 앞의 둘은 모든 볼트 작업을 실패하게 하고, 표식이 손상되면 프로젝트 구분 규칙을 점검할 수 없어 노트가 섞여도 알아차리지 못하기 때문이다.

- 플러그인: `claude plugin list --json`에서 `id`가 `claude-obsidian@`으로 시작하고, `scope`가 `user`이거나 `projectPath`가 현재 저장소 루트와 같은 항목을 찾는다. 다른 프로젝트에만 설치된 항목은 이 프로젝트에서 쓸 수 없으므로 세지 않는다. 여러 개가 남으면 `user` 범위를 본다. 명령 자체를 실행할 수 없으면 `INFO`(확인 불가), 항목이 없으면 `INFO`(설치 안 함)로 보고하고 이 항목의 나머지는 건너뛴다. `enabled`가 `false`면 `WARN`(스킬과 훅이 동작하지 않음). 해결: `claude plugin enable <id>`. 남은 항목이 project·local 범위뿐이면 `INFO`(이 프로젝트에서만 쓸 수 있음. 여러 프로젝트에서 같은 볼트를 쓰려면 user 범위로 설치).
- Python: `command -v python3`와 `python3 --version`을 본다. 플러그인의 훅과 CLI가 이 `python3`로 실행된다.
  - 없거나 3.11보다 낮으면 `FAIL`(플러그인 스킬과 훅이 실행되지 않음). `which -a python3` 결과를 함께 적는다. 해결: `/dev-harness:setup`을 다시 실행하면 관리 도구를 파악해 설치를 제안한다. 더 높은 버전이 이미 있으면 그 디렉터리를 셸 설정에서 `PATH` 앞쪽에 두고 Claude Code를 다시 시작한다.
  - `curl -fsSL --max-time 10 https://endoflife.date/api/python.json`에서 같은 계열(`cycle`)을 찾는다. 오늘 날짜(`date +%F`)가 `eol`을 지났으면 `WARN`(지원 종료로 보안 수정을 받지 못함), 버전이 `latest`보다 낮으면 `WARN`(보안 패치가 빠졌을 수 있음). 두 경우 모두 `eol`과 `latest`를 적는다. 조회에 실패하면 버전만 적어 `INFO`(취약 여부 확인 불가).
  - 그 밖에는 버전과 경로를 `OK`에 적는다.
- 볼트 경로: `.claude/settings.local.json`, `.claude/settings.json`, `~/.claude/settings.json` 순으로 처음 나오는 `env.CLAUDE_OBSIDIAN_VAULT`가 이 프로젝트에 실제로 적용되는 값이다. 어느 파일에서 왔는지 함께 적는다. JSON으로 읽을 수 없는 파일이 있으면 `WARN`(Claude Code가 이 파일의 설정을 적용하지 못함).
  - 값이 없으면 `WARN`(스킬이 현재 디렉터리와 그 위에서만 볼트를 찾으므로 다른 프로젝트에서는 볼트를 찾지 못함). 해결: `/dev-harness:setup`을 다시 실행하거나 `~/.claude/settings.json`의 `env`에 `"CLAUDE_OBSIDIAN_VAULT": "<볼트 절대 경로>"`를 넣는다.
  - 값의 디렉터리가 없으면 `FAIL`(볼트를 찾지 못해 모든 작업이 중단됨).
  - 프로젝트 파일에서 온 값이면 `INFO`(이 프로젝트만 다른 볼트를 씀. 의도했을 수도 있다).
  - 위 세 파일 중 `env.CLAUDE_OBSIDIAN_SESSION_CONTEXT`가 `"1"`인 파일이 있으면 같은 줄에 `WARN`으로 덧붙인다(다른 프로젝트의 최근 맥락이 세션에 섞임).
- 볼트 상태: 볼트 경로의 디렉터리가 있을 때만 본다. 경로에 `.claude-obsidian.json`과 `wiki/`가 모두 있으면 `OK`. `wiki/`만 있으면 `WARN`(예전 형식. 출처 원장과 볼트 설정이 없음). `wiki/`가 없으면 `WARN`(초기화되지 않음). 두 경우 모두 해결: `/dev-harness:setup`을 다시 실행하면 미리 보기를 보여 주고 이전하거나 초기화한다.
- 프로젝트 규칙: `.claude/rules/obsidian-vault.md`가 없으면 `INFO`(이 프로젝트 노트에 태그와 허브가 붙지 않아 다른 프로젝트 노트와 섞임). 이 항목의 나머지는 건너뛴다.
  - 첫 줄이 `<!-- obsidian-vault: <이름> -->` 형태이고 `<이름>`이 영문자·숫자·`-`·`_`로만 되어 있는지 본다. 아니면 `FAIL`(표식을 지웠거나 이름이 태그로 쓸 수 없음). 표식이 유효하면 이름을 `OK`에 적고, `T/rules/obsidian-vault.md`의 `<프로젝트>`를 이 이름으로 바꾼 내용과 비교한다.
  - `git check-ignore -q .claude/rules/obsidian-vault.md`가 실패하면 `WARN`(개인 볼트 규칙이 팀 저장소에 커밋될 수 있음). 해결: `git rev-parse --git-path info/exclude`가 가리키는 파일에 `/.claude/rules/obsidian-vault.md` 줄을 추가한다. `git ls-files`에 이미 있으면 추적 중이라 제외되지 않는다고 함께 적는다.

## 보고 형식

1. 항목별로 한 줄씩 `판정 항목 — 근거`를 나열한다. 원본 비교 결과는 해당 항목 아래에 붙인다.
2. 마지막에 `FAIL n건, WARN n건, OK n건, INFO n건`으로 합계를 적는다.
3. `FAIL`이 있으면 먼저 고칠 순서를 제안한다.
