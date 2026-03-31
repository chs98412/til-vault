---
title: "쿠키 완전 정복: 속성, 보안, 크로스도메인 한계"
date: 2026-03-31
category: network-security
tags: [cookie, SameSite, HttpOnly, Secure, CSRF, XSS, 쿠키체이닝, Domain, Path]
related: [session-management, jwt-token, oauth2-oidc-sso]
---

# 쿠키 완전 정복

## 쿠키란?

HTTP는 stateless 프로토콜이라 요청 간 상태를 공유하지 않는다. 쿠키는 이 한계를 극복하기 위해 **브라우저에 key-value 데이터를 저장하고, 매 요청마다 서버에 자동 전송**하는 HTTP 헤더 기반 메커니즘이다.

- 서버 → 브라우저: `Set-Cookie` 응답 헤더로 쿠키 설정
- 브라우저 → 서버: `Cookie` 요청 헤더에 자동 첨부

## 쿠키 vs 로컬스토리지

|  | 쿠키 | 로컬스토리지 |
|--|------|-------------|
| **설계 목적** | 서버와의 HTTP 통신 상태 유지 | 클라이언트 측 범용 저장소 |
| **자동 전송** | 매 요청마다 브라우저가 자동으로 서버에 보냄 | 전송 안 됨. JS로 직접 꺼내서 보내야 함 |
| **접근 제어** | `HttpOnly`로 JS 접근 차단 가능 | JS에서 항상 접근 가능 |
| **격리 단위** | Domain + Path 기반 (서브도메인 공유 가능) | Origin(프로토콜+도메인+포트) 단위 완전 격리 |
| **보안 속성** | `Secure`, `SameSite`, `HttpOnly`, `Domain`, `Path` | 없음 |

핵심 차이: 쿠키는 HTTP 통신을 위해 보안 장치가 촘촘하게 설계된 반면, 로컬스토리지는 범용 저장소라 보안 장치가 없다.

## Set-Cookie 속성 총정리

### SameSite

cross-site 요청에서 쿠키 전송 여부를 결정한다.

| 값 | cross-site sub-request (fetch, iframe) | cross-site top-level navigation (link) |
|----|---------------------------------------|---------------------------------------|
| `Strict` | X | X |
| `Lax` (기본값) | X | O (GET only) |
| `None` | O (`Secure` 필수) | O |

- 2020년경 Chrome이 기본값을 `None` → `Lax`로 변경. Firefox 96(2022.01)부터 따라감.
- 변경 이유: **CSRF 방어**. cross-site POST 요청에 쿠키 자동 첨부를 막아 공격을 원천 차단.

### Domain

```
Set-Cookie: SESSION=abc; Domain=.company.com
```

- `Domain=.company.com` 설정 시 `a.company.com`, `b.company.com` 등 모든 서브도메인에 전달.
- 미설정 시 쿠키를 발급한 정확한 호스트에만 전달.
- **쿠키 체이닝**: 같은 상위 도메인 내 서브도메인 간 쿠키를 공유하는 것. 도메인이 완전히 다르면 불가능.

### Secure

- `Secure` 설정 시 HTTPS 요청에서만 쿠키 전송.
- `SameSite=None`을 쓰려면 `Secure` 필수.

### HttpOnly

- `HttpOnly` 설정 시 `document.cookie`로 JS 접근 불가.
- XSS 공격으로 쿠키 값 탈취 방지. 세션 쿠키에는 거의 필수.

### Path, Expires, Max-Age

- `Path=/admin`: 해당 경로 하위 요청에만 쿠키 전달
- `Expires`: 특정 날짜까지 유지
- `Max-Age`: 초 단위 수명. 둘 다 없으면 session cookie(브라우저 종료 시 삭제)

### __Host- / __Secure- prefix

- `__Host-SESSION=abc`: 브라우저가 `Secure` 필수, `Path=/` 필수, `Domain` 설정 불가를 강제
- 상위 도메인에서의 쿠키 주입(Cookie Tossing) 방어

## Origin vs Site

**Origin** = scheme + host + port (`https://a.example.com:443`)
- 하나라도 다르면 cross-origin
- 로컬스토리지, CORS가 이 기준으로 동작

**Site** = eTLD+1 (`example.com`)
- `SameSite` 쿠키가 이 기준으로 판단
- `a.example.com`과 `b.example.com`은 cross-origin이지만 same-site

**eTLD (effective TLD)**: Public Suffix List에 등록된 도메인.
- `.com` → TLD이자 eTLD
- `github.io`, `co.kr` → Public Suffix List에 등록된 eTLD
- `alice.github.io`와 `bob.github.io`는 cross-site (eTLD+1이 다름)

> [위젯: SameSite=None 쿠키 문제 시뮬레이션](../_widgets/cookie-samesite-none-problems.html)

## SameSite=None과 서드파티 쿠키의 한계

`SameSite=None; Secure`로 설정하면 cross-site 요청에도 쿠키가 전달되지만:
- Safari ITP: 서드파티 쿠키 기본 차단
- Firefox ETP: 서드파티 쿠키 제한
- Chrome: Privacy Sandbox로 서드파티 쿠키 폐지 방향

차단 이유: 광고 네트워크가 서드파티 쿠키로 크로스사이트 사용자 추적에 악용 → 프라이버시 규제(GDPR, CCPA) 대응.

SSO에서도 영향: `SameSite=None`으로 SSO 쿠키를 공유하는 방식은 브라우저 정책에 취약 → OIDC 토큰 기반으로 대체.

## 쿠키 기반 보안 공격과 방어

> [위젯: CSRF 공격 & 방어 시뮬레이션](../_widgets/csrf-attack-defense.html)

### CSRF (Cross-Site Request Forgery)

공격 원리: 사용자가 bank.com에 로그인된 상태에서 evil.com 접속 → evil.com의 hidden form이 bank.com/transfer로 POST → 쿠키가 자동 첨부되어 송금 실행.

방어:
1. **SameSite=Lax** (브라우저 레벨): cross-site POST에 쿠키 안 붙임
2. **CSRF token** (서버 레벨): 서버가 form에 랜덤 토큰을 hidden field로 삽입 → 요청 시 토큰 검증 → 공격자는 토큰을 모르므로 실패

### XSS와 쿠키/로컬스토리지

- 로컬스토리지: XSS 시 `localStorage.getItem('token')`으로 토큰 탈취 → 공격자 서버로 전송 → 영구 재사용 가능
- `HttpOnly` 쿠키: XSS가 있어도 `document.cookie`로 접근 불가 → 값 자체를 빼낼 수 없음

### CORS

- SameSite: 요청에 쿠키를 붙일지 (요청 시점)
- CSRF token: 요청의 출처가 우리 페이지인지 (서버 검증)
- CORS: 응답을 JS에 노출할지 (응답 시점)

세 메커니즘은 다른 단계를 보호한다. 실무에서는 세 가지를 조합해야 완전한 방어.

## 실무 체크리스트

- [ ] 세션 쿠키: `HttpOnly; Secure; SameSite=Lax` 설정
- [ ] cross-site 통신 필요 시: `SameSite=None; Secure` + CORS 설정
- [ ] 가능하면 `__Host-` prefix 사용
- [ ] 서브도메인 공유 필요 시: `Domain=` 명시적 설정
- [ ] 세션 쿠키 `Max-Age` / `Expires` 적절한지 확인
- [ ] Safari/Firefox 서드파티 쿠키 차단 시 기능 정상 동작 확인
