# TIL 아카이빙 프로세스

## 전체 흐름

```
claude.ai 대화 → "정리해줘" → 파일 다운로드 → ~/personal/til-vault/_inbox/ 에 넣기 → Claude Code에서 정리 프롬프트 실행
```

---

## 1단계 프롬프트: 대화 정리 (claude.ai에서 사용)

대화 끝에 아래 프롬프트를 붙여넣으세요.

```
이 대화를 TIL로 정리해줘. 다음 규칙을 지켜줘:

## 마크다운 문서
- 파일명: YYYY-MM-DD-주제-kebab-case.md
- 아래 구조로 작성:
  ---
  title: "제목"
  date: YYYY-MM-DD
  category: "카테고리명"
  tags: [관련태그들]
  widgets: [위젯파일명들]
  ---
  ## 핵심 질문
  (내 원래 질문을 명확하게 정돈)

  ## TL;DR
  (한줄 요약)

  ## 상세 설명
  (핵심 내용. 코드 스니펫 포함)

  ## 실무 연결
  (실무에 어떻게 적용되는지)

  ## 위젯
  (있으면 파일명과 설명 목록)

- 카테고리: vault의 기존 폴더 구조를 참고하되, 맞는 게 없으면 새 카테고리를 제안해도 됨
  (기본: db-internals, kafka-distributed, jvm-kotlin, spring-jpa, infra, architecture, ai-tooling, misc)
- 하위 카테고리가 필요하면 서브폴더로 표현 (예: infra/k8s, db-internals/mysql)
- 내가 러프하게 질문한 것도 깔끔하게 정돈해서 작성
- 대화에서 다룬 주제가 여러 개면 주제별로 파일을 분리

## HTML 위젯
- 이 대화에서 만든 인터랙티브 위젯을 전부 개별 .html 파일로 추출
- 파일명: 마크다운과 매칭되는 이름 (예: btree-visualizer.html)
- 반드시 standalone으로 실행 가능해야 함 (외부 CDN 의존은 OK, 로컬 의존은 안됨)
- CSS/JS 전부 인라인으로 포함
- 위젯이 여러 개면 각각 별도 파일

## 출력
- 모든 파일을 다운로드 가능하게 만들어줘
- 파일 목록을 마지막에 정리해서 보여줘:
  📁 출력 파일:
  - [md파일명] → category: xxx
  - [html파일명] → widget for: xxx
```

---

## 2단계 프롬프트: vault 정리정돈 (Claude Code에서 사용)

다운로드한 파일들을 `~/personal/til-vault/_inbox/`에 넣은 후,
vault 루트에서 아래 프롬프트로 Claude Code를 실행하세요.

```
~/personal/til-vault/_inbox/ 에 새 TIL 파일들이 있어.
이 vault를 정리해줘. 다음 규칙을 따라:

## 1. _inbox 파일 분석
- _inbox/ 안의 .md 파일들의 frontmatter에서 category를 읽어
- .html 위젯 파일도 함께 확인

## 2. 중복/유사 체크
- 기존 vault의 각 카테고리 폴더를 스캔
- 새 파일과 기존 파일의 title, tags, 내용을 비교
- 중복 판단 기준:
  - 같은 주제를 다루는 기존 문서가 있으면 → APPEND 후보
  - APPEND 후 문서가 2~3페이지를 초과하면 → SPLIT 후보 (세부 주제별 파일 분리)
  - 완전히 동일한 내용이면 → SKIP 후보
  - 새로운 내용이면 → MOVE 후보

## 3. 실행 전 확인
- 아래 형식으로 계획을 먼저 보여줘:
  📋 정리 계획:
  - [파일명] → MOVE to /카테고리/ (신규)
  - [파일명] → APPEND to /카테고리/기존파일.md (유사 문서 존재: 기존파일명)
  - [파일명] → SPLIT: 기존파일.md → 세부파일1.md + 세부파일2.md (내용 과다)
  - [파일명] → SKIP (중복: 기존파일명)
  - [위젯파일명] → MOVE to /_widgets/
- 내가 확인하면 실행

## 4. 실행
- MOVE: 해당 카테고리 폴더로 이동
- APPEND: 기존 문서에 자연스럽게 내용 추가 (중복 없이 이어지도록).
  기존 frontmatter의 tags에 새 태그 병합.
- SPLIT: 기존 문서 + 새 내용 합쳤을 때 너무 길면 (2~3페이지 초과) 세부 카테고리로 파일 분리.
  예) session-management.md가 너무 길어지면 → session-lifecycle.md + distributed-session.md + spring-session-redis.md
- SKIP: _inbox에서 삭제
- .html 위젯: _widgets/ 폴더로 이동
- .md 안에 위젯을 iframe으로 임베드 (../_widgets/파일명.html)

## 5. 완료 보고
  ✅ 정리 완료:
  - 신규 N개, 병합 N개, 분리 N개, 스킵 N개
  - 위젯 N개 이동
  - 변경된 파일 목록
```

---

## 폴더 구조 참고

```
til-vault/
├── _inbox/              ← 1단계 산출물을 여기에 넣기
├── _templates/
├── _widgets/            ← HTML 위젯 보관
├── db-internals/
├── kafka-distributed/
├── jvm-kotlin/
├── spring-jpa/
├── infra/
├── architecture/
├── ai-tooling/
└── misc/
```

## 카테고리 가이드

초기 카테고리는 아래와 같지만, 언제든 추가/세분화 가능.
새 카테고리가 필요하면 2단계에서 폴더를 만들고 이 테이블에 추가할 것.

| 카테고리 | 포함 주제 |
|---------|----------|
| db-internals | B+Tree, InnoDB, Index, MySQL, 쿼리 최적화 |
| kafka-distributed | Kafka, Spark, 분산시스템, 합의 알고리즘 |
| jvm-kotlin | JVM, GC, Kotlin, 코루틴, GraalVM |
| spring-jpa | Spring Boot, JPA/Hibernate, Security |
| infra | K8s, Docker, Valkey/Redis, CI/CD |
| architecture | 설계 패턴, 시스템 아키텍처, DDD |
| ai-tooling | Claude, MCP, AI IDE, 프롬프팅, RAG |
| network-security | 쿠키, 세션, JWT, OAuth2, OIDC, SSO, CORS, CSRF |
| misc | 위 카테고리에 안 맞는 것들 |

서브폴더 예시: `infra/k8s/`, `db-internals/mysql/`, `architecture/ddd/`
