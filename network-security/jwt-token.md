---
title: "JWT 토큰: 구조, 서명 검증, Refresh Token 전략"
date: 2026-03-31
category: network-security
tags: [JWT, access-token, refresh-token, 서명검증, stateless, HMAC, RS256, token-rotation]
related: [session-management, oauth2-oidc-sso]
---

# JWT 토큰

## 세션 vs JWT 트레이드오프

| | 세션 (stateful) | JWT (stateless) |
|--|----------------|-----------------|
| **상태 저장** | 서버(Redis)에 저장 | 토큰 자체에 포함 |
| **매 요청 시** | Redis 조회 필요 | CPU 서명 검증만 (외부 저장소 호출 없음) |
| **즉시 무효화** | `session.invalidate()` → Redis 삭제 → 즉시 효과 | 불가능. 만료까지 유효 |
| **스케일링** | Redis가 SPOF 가능성 | 외부 저장소 불필요, 수평 확장 용이 |
| **마이크로서비스** | 서비스 간 Redis 공유 필요 | 토큰을 그대로 전달하면 됨 |

## JWT 구조

`header.payload.signature` 세 부분을 점(.)으로 연결한 문자열.

### Header
```json
{ "alg": "HS256", "typ": "JWT" }
```
서명 알고리즘 정보.

### Payload
```json
{ "user": "혁순", "role": "ADMIN", "exp": 1711872000 }
```
실제 데이터(claims). **Base64 인코딩일 뿐 암호화가 아님** → 누구나 디코딩 가능.

### Signature
```
HMACSHA256(base64(header) + "." + base64(payload), SECRET_KEY)
```
서버만 아는 SECRET_KEY로 생성. **무결성 보장**이 목적.

## 서명 검증 원리

1. 서버가 토큰을 받으면 SECRET_KEY로 서명을 재계산
2. 재계산된 서명 vs 토큰의 서명 비교
3. 일치 → 변조 없음 확인 → DB/Redis 조회 없이 인증 완료

**변조 방어**: Payload를 1글자만 바꿔도 서명이 완전히 달라짐. 공격자가 새 서명을 만들려면 SECRET_KEY가 필요하지만 모르므로 불가능.

**탈취 방어 불가**: 토큰을 통째로 복사하면 서명이 완벽하게 유효. 서버는 탈취된 토큰인지 구분 불가. → JWT의 근본적 한계.

## Access Token + Refresh Token 패턴

### 왜 필요한가

JWT는 즉시 무효화가 불가능하므로, Access Token TTL을 짧게(15분) 설정하고 Refresh Token으로 갱신하는 전략.

### 두 토큰의 역할

| | Access Token | Refresh Token |
|--|-------------|---------------|
| **형태** | JWT (stateless) | 랜덤 문자열 또는 JWT |
| **TTL** | 짧음 (15분) | 김 (7일) |
| **저장 위치** | 클라이언트 | **Redis에 저장** (stateful) |
| **용도** | API 요청 인증 | Access Token 재발급 |
| **Redis 조회** | 없음 (서명 검증만) | 있음 (갱신 시에만) |

### 흐름

1. 일반 API 요청: Access Token만 사용. 서명 검증만 → Redis 부하 제로
2. Access Token 만료: 401 응답
3. 클라이언트가 Refresh Token으로 `/auth/refresh` 호출
4. 서버가 Redis에서 Refresh Token 유효성 확인
5. 새 Access Token + 새 Refresh Token 발급 (rotation)
6. 기존 Refresh Token은 Redis에서 삭제

### Redis 부하 비교

- 세션 방식: 매 요청마다 Redis 조회 (초당 1만 건 → 1만 번)
- Access + Refresh: 15분에 1번만 Redis 조회 → **수백 배 감소**

### Refresh Token을 왜 Redis에 저장하는가?

서명만으로 검증하면 강제 무효화 불가능. Redis에 저장하면:
- 삭제 → 즉시 무효화
- rotation 시 이전 토큰을 "사용됨"으로 표시
- **Reuse detection**: 이미 사용된 Refresh Token으로 요청 오면 → 탈취 감지 → 해당 사용자 전체 토큰 무효화

### Refresh Token Rotation과 탈취 대응

- 공격자가 먼저 Refresh Token 사용 → 새 토큰 획득 → 정상 사용자의 Refresh 시도 시 reuse detection 발동 → 전체 무효화
- 정상 사용자가 먼저 사용 → rotation됨 → 공격자의 기존 Refresh Token은 "이미 사용됨" → reuse detection → 전체 무효화
- 어느 쪽이든 결국 감지됨. 단, 공격자가 먼저 쓰면 그 사이에 활동 가능 → **완벽한 방어가 아닌 피해 최소화 전략**
- 1차 방어선은 HttpOnly 쿠키로 XSS 탈취 자체를 방지하는 것

<iframe src="../_widgets/jwt-signature-verification.html" width="100%" height="520" frameborder="0"></iframe>

## 로컬스토리지 vs HttpOnly 쿠키에 토큰 저장

| | 로컬스토리지 | HttpOnly 쿠키 |
|--|------------|--------------|
| XSS 시 | 토큰 탈취 가능 → 영구 재사용 | 값 자체 접근 불가 |
| CSRF | 면역 (자동 전송 안 됨) | 취약 (SameSite + CSRF token으로 방어) |
| 실무 권장 | 간편하지만 보안 취약 | 보안에 민감한 서비스에서 권장 |

보안 우선이면 HttpOnly 쿠키, 커머스 플랫폼 환경에서는 쿠키 기반이 더 적절.
