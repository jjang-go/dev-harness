# dev-harness

프로젝트마다 직접 만든 규칙 세트(하네스)를 초기 세팅해주는 Claude Code 플러그인.

## 구조

```
.claude-plugin/
  plugin.json        # 플러그인 매니페스트
  marketplace.json   # 마켓플레이스 등록용
skills/init/         # 하네스 초기화 스킬
agents/              # 서브에이전트 정의
hooks/hooks.json     # 훅 설정
templates/rules/     # 프로젝트로 복사될 규칙 템플릿
```

## 로컬 테스트

```
claude --plugin-dir .
```

## 브랜치 전략

- `main`: 릴리스
- `develop`: 개발 (feature 브랜치는 develop에서 파생)
