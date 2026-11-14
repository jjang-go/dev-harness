<!-- obsidian-vault: <프로젝트> -->
# Obsidian 볼트 기록 규칙

여러 프로젝트가 claude-obsidian 볼트 하나를 함께 쓴다. 볼트의 폴더는 노트 유형(sources, entities, concepts, questions, sessions)으로만 나뉘므로, 어느 프로젝트의 노트인지는 태그와 허브 페이지로 구분한다. 이 규칙이 없으면 프로젝트 사이에 노트가 섞여 나중에 프로젝트별로 찾을 수 없다.

이 프로젝트의 이름은 `<프로젝트>`다.

## 노트를 만들거나 고칠 때

claude-obsidian으로 save, ingest, autoresearch 병합을 할 때 아래를 같은 트랜잭션에 넣는다.

- 이 프로젝트에서 나온 노트의 frontmatter `tags`에 `project/<프로젝트>`를 넣는다. 여러 프로젝트에 공통인 개념 노트에 태그를 더할 때는 기존 태그를 지우지 않고 덧붙인다.
- 프로젝트 허브 `wiki/entities/<프로젝트>.md`에 새 노트 링크를 추가한다. 허브가 이 프로젝트 노트의 목록 역할을 하기 때문이다. 링크는 `[[wiki/concepts/노트|노트]]`처럼 볼트 경로를 포함해 쓴다. claude-obsidian은 카탈로그에서 경로를 포함한 링크만 연결로 인정한다.
- save와 ingest에서는 허브를 번들의 `workflow.catalogs`에 넣는다. autoresearch 병합은 이 검사를 거치지 않으므로 허브 링크만 같은 번들에서 갱신한다.
- 허브가 아직 없으면 처음 저장할 때 함께 만든다. 허브의 `type`은 `entity`, `tags`에는 `project/<프로젝트>`를 넣는다. 새로 만든 허브는 활성 카탈로그에서 링크되어야 하므로 `wiki/index.md`에 `[[wiki/entities/<프로젝트>|<프로젝트>]]` 링크를 추가하고, save와 ingest에서는 `wiki/index.md`도 `workflow.catalogs`에 함께 넣는다.
- 노트의 본문, 소제목, `title` 값, 파일 이름은 한국어로 쓴다. 영어 자료를 ingest할 때도 요약과 설명은 한국어로 쓰고, 인용문만 원문을 유지한다. 볼트 하나에 여러 언어가 섞이면 검색과 링크가 흩어지기 때문이다.
- 다만 claude-obsidian이 검사하는 부분은 영어로 둔다. 속성 이름(`type`, `title`, `status`, `created`, `updated`, `tags` 등), 유형·상태 값(`concept`, `evergreen` 등), `wiki/log.md` 항목 제목 형식(`## [YYYY-MM-DD] save`)을 바꾸면 트랜잭션 검사가 실패한다.
- `wiki/index.md`에는 프로젝트 허브와 여러 프로젝트에 공통인 노트만 올린다. 프로젝트 노트는 허브에만 올려 색인이 한없이 커지지 않게 한다.

## 하지 않는 것

- 대화 전체를 자동으로 저장하지 않는다. 사용자가 고른 내용만 저장한다.
- 볼트 파일을 Write·Edit 도구로 직접 고치지 않는다. claude-obsidian의 트랜잭션으로만 바꾼다.
- 비밀값, 토큰, 내부 접속 정보는 노트에 옮기지 않는다.
