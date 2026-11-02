---
name: init
description: 현재 프로젝트에 dev-harness 규칙 세트를 초기 세팅한다. "하네스 초기화", "harness init" 요청 시 사용.
---

# dev-harness init

현재 프로젝트에 하네스를 초기 세팅한다.

## 절차

1. 프로젝트 루트와 기존 `CLAUDE.md`, `.claude/` 존재 여부를 확인한다.
2. `${CLAUDE_PLUGIN_ROOT}/templates/` 의 템플릿을 대상 프로젝트로 복사한다. 기존 파일은 덮어쓰지 않고 사용자에게 확인한다.
3. 프로젝트 특성(언어/프레임워크)에 맞게 규칙을 조정한다.
4. 생성된 파일 목록을 보고한다.

## 템플릿 위치

- `templates/rules/` — 프로젝트에 복사될 규칙 파일

> TODO: 실제 규칙 내용은 templates/ 에 추가하면서 이 절차를 구체화한다.
