---
title: "세션 관리: 라이프사이클부터 분산 환경 전략까지"
date: 2026-03-31
category: network-security
tags: [session, Redis, Valkey, sticky-session, session-clustering, Spring-Session, 분산세션]
related: [cookie-deep-dive, jwt-token]
---

# 세션 관리

## 세션이란?

HTTP가 stateless이므로 "이 요청이 아까 그 사용자의 요청이다"를 서버가 식별하기 위한 **서버 측 상태 관리 개념**. 쿠키는 세션 ID를 전달하는 운반체이고, 세션은 서버에 저장된 실제 데이터.

## 세션 라이프사이클

### 생성
1. 서버가 UUID 기반 세션 ID 생성
2. 메모리(ConcurrentHashMap 등)에 빈 세션 객체 생성
3. `Set-Cookie: JSESSIONID=abc123`으로 브라우저에 전달

### 인증 정보 저장
```java
session.setAttribute("user", "혁순");
session.setAttribute("role", "ADMIN");
```
세션 자체는 로그인 전에 이미 생성됨. 로그인은 세션에 인증 정보를 채우는 과정.

### 조회
매 요청마다 `Cookie: JSESSIONID=abc123` → 서버가 세션 저장소에서 조회 → 사용자 식별.
요청이 올 때마다 TTL(기본 30분) 갱신.

### 폐기
- 로그아웃: `session.invalidate()` + `Set-Cookie: JSESSIONID=; Max-Age=0`
- 타임아웃: 30분간 요청 없으면 백그라운드 스레드가 정리
- 서버 재시작: 메모리의 세션 전부 유실 → **분산 환경 문제의 시작점**

## 분산 환경에서의 세션 문제

서버가 여러 대일 때, 사용자가 Pod-1에서 로그인 후 다음 요청이 Pod-2로 가면 세션을 못 찾음 → 로그인이 풀리는 현상.

### 해결 전략 비교

| 전략 | 방식 | 장점 | 단점 |
|------|------|------|------|
| **No strategy** | 아무것도 안 함 | - | 요청마다 랜덤 로그아웃 |
| **Sticky session** | LB가 사용자를 특정 Pod에 고정 | 구현 간단 | Pod 장애 시 세션 유실, 트래픽 불균형 |
| **Session clustering** | Pod 간 세션 복제 | 어느 Pod든 세션 있음 | 메모리 Pod수×세션수, 네트워크 n(n-1) 동기화 비용, 소규모만 가능 |
| **외부 세션 스토어 (Redis)** | Pod는 stateless, Redis에 세션 저장 | Pod 장애에 안전, 스케일 아웃 자유 | Redis 의존성, 네트워크 레이턴시 |

### Session clustering 세부

- Tomcat DeltaManager: all-to-all 복제. 세션 변경 시 모든 Pod에 전송.
- Tomcat BackupManager: primary-backup. 하나의 백업 Pod에만 복제.
- 전송 방식: UDP 멀티캐스트(클라우드에서 대부분 미지원) 또는 TCP 유니캐스트.
- Kubernetes에서 문제: Pod IP가 유동적, 멀티캐스트 불가, Pod 증가 시 동기화 트래픽 급증.

<iframe src="../_widgets/distributed-session-strategies.html" width="100%" height="480" frameborder="0"></iframe>

## Spring Session + Redis/Valkey

### 핵심 원리

`@EnableRedisHttpSession` → `SessionRepositoryFilter`가 모든 요청을 가로채서 `HttpServletRequest.getSession()`을 Redis 기반 구현체로 교체. 애플리케이션 코드 수정 불필요.

### Redis 데이터 구조

```
KEY:   spring:session:sessions:abc123  (HASH type)
FIELD: sessionAttr:user     → "혁순"
FIELD: sessionAttr:role     → "ADMIN"
FIELD: creationTime         → 1711872000000
FIELD: maxInactiveInterval  → 1800
FIELD: lastAccessedTime     → 1711872000000
TTL:   EXPIRE ... 1800
```

### 요청 흐름

1. 브라우저가 `Cookie: SESSION=abc123` 전송
2. `SessionRepositoryFilter`가 쿠키에서 세션 ID 추출
3. `RedisIndexedSessionRepository.findById("abc123")`
4. Redis: `HGETALL spring:session:sessions:abc123`
5. `lastAccessedTime` 갱신 + `EXPIRE` 연장
6. `HttpServletRequest.getSession()` → Redis 기반 객체 반환

### 쿠키 설정

```java
@Bean
public CookieSerializer cookieSerializer() {
    DefaultCookieSerializer s = new DefaultCookieSerializer();
    s.setCookieName("SESSION");  // JSESSIONID가 아님
    s.setUseHttpOnlyCookie(true);
    s.setUseSecureCookie(true);
    s.setSameSite("Lax");
    return s;
}
```

### 세션 폐기

- 로그아웃: `session.invalidate()` → Redis `DEL spring:session:sessions:abc123`
- 타임아웃: Redis `EXPIRE`가 자동 처리 → 서버가 별도로 정리할 필요 없음

### Kubernetes 환경에서의 이점

- Pod는 세션을 메모리에 안 들고 있으므로 완전한 stateless
- Pod 장애/재시작/스케일 아웃해도 세션 안전
- 어느 Pod로 요청이 가든 같은 Redis를 보므로 일관성 보장
