---
name: doctor
description: 현재 프로젝트에 설치된 dev-harness 상태를 점검하고 문제를 보고한다. "하네스 점검", "하네스 진단", "harness doctor" 요청 시 사용.
allowed-tools: Read Glob Grep Bash(git config *) Bash(git rev-parse *) Bash(git remote *) Bash(date *) Bash(ls *)
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

## 원본 비교

설치된 파일과 원본의 대응은 setup 스킬의 설치 표(`${CLAUDE_PLUGIN_ROOT}/skills/setup/SKILL.md`)를 따른다. 대응을 여기에 따로 적지 않는다. 표가 바뀌면 점검도 함께 따라가게 하기 위해서다.

- 설치된 파일은 모두 원본과 비교한다. 같으면 `OK`, 다르면 `WARN`으로 차이를 한두 줄로 요약한다.
- 다르다는 것은 팀이 의도해서 고쳤거나, 플러그인이 갱신되어 설치본이 낡았다는 뜻이다. 어느 쪽인지 단정하지 말고 둘 다 가능하다고 적는다. 원본을 받으려면 `/dev-harness:setup`을 다시 실행하면 된다(다른 파일은 덮어쓰기 전에 묻는다).
- 채워 쓰도록 만든 파일인 `ARCHITECTURE.md`는 비교하지 않는다.

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
- `docs/specs/TEMPLATE.md`

플랫폼 파일은 `.github/` 또는 `.gitlab/` 중 어느 쪽이 있는지로 판단한다.

- `.github/`가 있으면 `PULL_REQUEST_TEMPLATE.md`, `ISSUE_TEMPLATE/bug_report.yml`, `ISSUE_TEMPLATE/feature_request.yml`이 있는지 본다.
- `.gitlab/`이 있으면 `merge_request_templates/Default.md`, `issue_templates/Bug.md`, `issue_templates/Feature.md`가 있는지 본다.
- 둘 다 없으면 `FAIL`. 둘 다 있으면 `INFO`(양쪽 설치됨).
- `git remote get-url origin`의 호스트와 설치된 플랫폼이 어긋나면 `WARN`.

### 6. 서드파티 고지

- 루트 `THIRD_PARTY_NOTICES.md`가 있는지 본다. 없으면 `INFO`(내부 프로젝트라면 정상. 외부에 공개·배포하는 프로젝트이고 CLAUDE.md가 Karpathy 지침이면 `WARN`).

## 보고 형식

1. 항목별로 한 줄씩 `판정 항목 — 근거`를 나열한다. 원본 비교 결과는 해당 항목 아래에 붙인다.
2. 마지막에 `FAIL n건, WARN n건, OK n건, INFO n건`으로 합계를 적는다.
3. `FAIL`이 있으면 먼저 고칠 순서를 제안한다.
