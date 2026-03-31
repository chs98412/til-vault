---
title: "B+Tree 삽입 시 페이지 분할 메커니즘"
date: 2026-03-28
category: "db-internals"
tags: [til, db, innodb, btree, deep-dive]
chat_url: "https://claude.ai/chat/xxx"
---

# B+Tree 삽입 시 페이지 분할 메커니즘

## 핵심 질문
> InnoDB의 B+Tree에서 리프 노드가 꽉 찼을 때 페이지 분할은 정확히 어떤 순서로 일어나는가? 부모 노드까지 연쇄적으로 분할되는 경우는?

## TL;DR
- 리프 페이지가 가득 차면 새 페이지를 할당하고 키의 절반을 이동시킨 뒤, 중간 키를 부모 내부 노드로 승격(promote)한다. 부모도 꽉 차면 재귀적으로 분할이 전파된다.

## 상세 설명

### 배경/맥락
Valkey → MySQL 동기화 배치에서 대량 INSERT 시 성능 저하가 발생. 인덱스 페이지 분할 빈도가 원인인지 확인하기 위해 B+Tree 분할 메커니즘을 정확히 이해할 필요가 있었음.

### 핵심 내용
1. **리프 노드 분할**: 페이지 내 레코드가 `MERGE_THRESHOLD`(기본 50%)를 넘기면 분할
2. **키 승격**: 분할 시 중간 키가 부모 내부 노드로 올라감
3. **연쇄 분할**: 부모 노드도 꽉 차 있으면 동일 로직이 루트까지 전파
4. **루트 분할**: 루트가 분할되면 트리 높이가 1 증가 (유일하게 높이가 늘어나는 케이스)

### 코드/예시
```sql
-- 페이지 분할 모니터링
SHOW GLOBAL STATUS LIKE 'Innodb_page_splits';
-- 인덱스별 분할 통계
SELECT * FROM sys.schema_index_statistics WHERE table_schema = 'my_db';
```

## 인터랙티브 위젯
> 🎮 [B+Tree 삽입 시각화 열기](../_widgets/btree-insert-visualizer.html)

## 실무 연결
- 벌크 INSERT 시 `ORDER BY PK` 정렬 후 삽입하면 순차 분할로 효율적
- 랜덤 UUID PK는 랜덤 분할을 유발하므로 auto_increment나 ULID 권장
- `innodb_fill_factor` 설정으로 페이지 여유 공간 조절 가능

## 관련 문서
- [[innodb-buffer-pool-동작원리]]
- [[index-selectivity-최적화]]

## 원본 대화
- [Claude 대화 링크](https://claude.ai/chat/xxx)
