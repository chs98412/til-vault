---
title: "OAuth2 / OIDC / Keycloak SSO 완전 정리"
date: 2026-03-31
category: network-security
tags: [OAuth2, OIDC, Keycloak, SSO, authorization-code-flow, PKCE, ID-token, access-token, 인증, 인가, Discovery]
related: [cookie-deep-dive, session-management, jwt-token]
---

# OAuth2 / OIDC / Keycloak SSO

## 인증(Authentication) vs 인가(Authorization)

| | 인증 (Authentication) | 인가 (Authorization) |
|--|---------------------|---------------------|
| **질문** | "너 누구야?" | "너 뭐 할 수 있어?" |
| **결과물** | ID token (OIDC) | access token (OAuth2) |
| **예시** | Keycloak 로그인 → "이 사람은 혁순" | 혁순은 ADMIN → 상품 가격 수정 가능 |
| **프로토콜** | OIDC | OAuth2 |

- 인증 없이 인가는 의미 없음 (누군지 모르는데 권한 줄 수 없음)
- OAuth2는 원래 인가만 담당. 인증이 필요해서 OIDC가 OAuth2 위에 추가됨.

<iframe src="../_widgets/authn-vs-authz.html" width="100%" height="480" frameborder="0"></iframe>

## OAuth2가 해결하는 문제

OAuth 이전: 써드파티 앱이 사용자 데이터에 접근하려면 사용자의 비밀번호를 직접 받아야 했음 → 비밀번호 유출 시 모든 권한이 노출.

OAuth2: **비밀번호를 공유하지 않고, 필요한 권한만 위임**. `scope=photos.readonly`로 사진 읽기만 허용, Gmail/Drive에는 접근 불가.

## OAuth2 4가지 주체

| 주체 | 역할 | 예시 |
|------|------|------|
| Resource Owner | 데이터의 주인 (사용자) | 사용자 |
| Client | 데이터에 접근하고 싶은 앱 | 사진 인쇄 서비스, service-a.com |
| Authorization Server | 권한 발급 담당 | Google, Keycloak |
| Resource Server | 실제 데이터 보유 | Google Photos API, 백오피스 API |

## Authorization Code Flow (+ PKCE)

가장 권장되는 OAuth2 흐름. 각 단계에 보안상 이유가 있다.

### 흐름

```
1. Client → Auth Server: GET /authorize
   ?response_type=code
   &client_id=service-a
   &redirect_uri=https://service-a.com/callback
   &scope=openid profile email
   &code_challenge=SHA256(code_verifier)  ← PKCE
   &state=xyz789                          ← CSRF 방어

2. 사용자가 Auth Server에서 직접 로그인 + 권한 동의
   → Client는 비밀번호를 절대 보지 못함

3. Auth Server → Browser: 302 Redirect
   → service-a.com/callback?code=AUTH_CODE&state=xyz789
   (access token이 아닌 일회용 code를 URL로 전달)

4. Client Server → Auth Server: POST /token (백채널)
   code=AUTH_CODE
   client_id=service-a
   client_secret=s3cr3t      ← Client 인증
   code_verifier=random()     ← PKCE 검증

5. Auth Server → Client Server: 토큰 응답
   { access_token, id_token, refresh_token }
```

### 각 단계의 설계 이유

| 요소 | 이유 |
|------|------|
| `code`를 먼저 발급 | access token을 URL에 노출하지 않기 위해 (URL은 브라우저 히스토리, 로그, Referer 헤더에 노출 가능) |
| `state` 파라미터 | CSRF 방어. 콜백 시 일치 여부 확인 |
| `client_secret` | Client가 진짜 해당 앱인지 인증 (서버에만 저장) |
| PKCE `code_verifier/challenge` | authorization code 탈취 시에도 토큰 발급 방지. `code_verifier`를 모르면 code로 토큰 교환 불가 |
| 백채널 토큰 교환 | 브라우저를 거치지 않는 서버 간 통신으로 토큰 노출 방지 |

<iframe src="../_widgets/oauth2-oidc-flow.html" width="100%" height="520" frameborder="0"></iframe>

## OIDC가 OAuth2에 추가하는 것

OAuth2와 거의 동일한 흐름에 두 가지 추가:

1. **scope에 `openid` 추가**: 이게 있어야 OIDC 흐름
2. **토큰 응답에 ID token (JWT) 포함**

### ID token 구조

```json
{
  "iss": "https://keycloak.company.com/realms/nhn-commerce",
  "sub": "user-uuid-1234",     // 사용자 고유 ID
  "aud": "service-a",          // 대상 Client
  "exp": 1711875600,
  "iat": 1711872000,
  "name": "혁순",              // profile scope
  "email": "hs@company.com",   // email scope
  "nonce": "abc123"            // replay 방어
}
```

### access token vs id token

| | access token | id token |
|--|-------------|----------|
| **용도** | API 호출 권한 (인가) | 사용자 식별 (인증) |
| **대상** | Resource Server (API) | Client (서비스) |
| **API 호출에 사용** | O | X (사용하면 안 됨) |

## OIDC Discovery

OIDC 스펙에 따라 모든 호환 프로바이더는 `/.well-known/openid-configuration`에 Discovery 문서를 제공해야 한다.

```
GET https://keycloak.company.com/realms/nhn-commerce/.well-known/openid-configuration
```

응답에 포함되는 핵심 필드:
- `issuer`: Provider 고유 식별자
- `authorization_endpoint`: 로그인/동의 URL
- `token_endpoint`: code → token 교환 URL
- `userinfo_endpoint`: 사용자 정보 조회 URL
- `jwks_uri`: ID token 서명 검증용 공개키 URL
- `scopes_supported`, `response_types_supported`, `grant_types_supported`

Spring Security에서 `issuer-uri`만 설정하면 나머지를 자동으로 가져오는 게 이 Discovery 덕분.

## 크로스도메인 SSO와 OIDC

<iframe src="../_widgets/subdomain-cookie-sso.html" width="100%" height="480" frameborder="0"></iframe>

### 왜 쿠키로 SSO가 안 되는가 (도메인이 다를 때)

서비스들이 `service-a.com`, `service-b.com`처럼 도메인이 완전히 다르면:
- `Domain=.company.com` 쿠키 공유 불가 (쿠키 체이닝 불가)
- `SameSite=None`은 서드파티 쿠키 차단 추세로 불안정
- `SameSite=None`이 되더라도 auth 쿠키가 service-a.com으로 가는 요청에 붙는 게 아님 → service-a.com에 자체 세션이 없음

### OIDC가 해결하는 방식

1. service-a.com 접속 → 세션 없음 → Keycloak으로 리다이렉트
2. Keycloak에서 로그인 (자기 도메인 쿠키로 세션 유지)
3. authorization code를 URL 파라미터로 전달 (쿠키 아님!)
4. service-a.com이 code로 토큰 교환 (서버 간 백채널)
5. ID token으로 사용자 식별 → **자기 도메인에 자체 세션 쿠키 발급**
6. 이후 service-a.com 내에서는 자체 세션으로 동작 → Keycloak 의존 없음

"쿠키 체이닝이 불가능해서 쿠키를 안 쓴다" = 서비스 간 인증 전달에 쿠키를 안 쓰는 것. 각 서비스 내부에서는 여전히 쿠키로 세션 유지.

<iframe src="../_widgets/oidc-sso-flow-simulation.html" width="100%" height="520" frameborder="0"></iframe>

## Keycloak 핵심 개념

### Core

| 개념 | 설명 |
|------|------|
| **Realm** | 최상위 격리 단위. 독립된 사용자, 클라이언트, 역할, 인증 설정 |
| **Client** | Keycloak에 인증/인가를 요청하는 애플리케이션. client_id, client_secret, redirect_uri 등 설정 |
| **User / Group** | Realm 내 사용자. Group으로 묶어서 Role 일괄 할당 가능 |
| **Realm Role** | 전체 Realm에서 유효. 예: ADMIN, USER |
| **Client Role** | 특정 Client에서만 유효. 예: backoffice의 price-editor |

### Role이 access token에 들어가는 구조

```json
{
  "realm_access": { "roles": ["ADMIN"] },
  "resource_access": {
    "backoffice": { "roles": ["price-editor"] }
  }
}
```

Spring Security에서: `@PreAuthorize("hasRole('price-editor')")`

### Authentication 기능

- **Authentication Flow**: Admin Console에서 로그인 흐름 시각적 설계 (Password → OTP → 조건부 2FA)
- **Identity Provider**: Google, GitHub 등 Social Login / 다른 Keycloak 인스턴스 연동
- **User Federation**: LDAP/Active Directory 연동 (동기화 또는 실시간 조회)
- **Session Management**: Admin Console에서 세션 조회/강제 로그아웃, SSO Session Max/Idle 설정

### Spring Security 연동

```yaml
# Client 모드 (로그인 처리)
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: service-a
            client-secret: s3cr3t
            scope: openid,profile,email
        provider:
          keycloak:
            issuer-uri: https://keycloak.company.com/realms/nhn-commerce

# Resource Server 모드 (API 토큰 검증)
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.company.com/realms/nhn-commerce
```

Resource Server 모드에서는 `jwks_uri`의 공개키로 JWT 서명을 로컬에서 검증 → 매 요청마다 Keycloak에 네트워크 호출 불필요.

### OAuth2/OIDC 스펙 준수

- OAuth2: RFC 6749 (핵심), RFC 7636 (PKCE), RFC 6750 (Bearer Token)
- OIDC: OpenID Connect Core 1.0, Discovery 1.0
- Keycloak, Google, Auth0, Okta 모두 같은 규약 → 클라이언트 구현이 프로바이더에 독립적
