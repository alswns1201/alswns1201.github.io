---
title: "BFF 환경에서 AccessToken을 쿠키에 담는 이유"
date: 2025-12-16
categories: [아키텍처]
tags: [architecture, jwt, frontend]
---

JWT 기반 인증을 설계하다 보면 항상 나오는 질문이 있다.

> AccessToken과 RefreshToken이 있는데, 왜 어떤 서비스는 AccessToken을 쿠키(HttpOnly)에
> 넣고, 어떤 곳은 절대 쿠키에 안 넣을까?

이 글에서는 **일반적인 SPA 구조**와 **BFF(Backend For Frontend) 구조**를 기준으로
AccessToken 저장 방식의 차이를 보안 모델 관점에서 정리한다. 단순한 패턴 나열이
아니라, 왜 그렇게 설계하는지에 초점을 둔다.

## JWT 인증 구조 기본 정리

- **AccessToken**: API 접근 권한 증명, 짧은 만료 시간(보통 5~15분)
- **RefreshToken**: AccessToken 재발급용, 상대적으로 긴 만료 시간

역할은 SPA든 BFF든 동일하다. 차이는 **브라우저가 이 토큰을 직접 다루느냐**에 있다.

## 일반적인 SPA + API 구조

```
[Browser SPA] → Authorization: Bearer AccessToken → [Backend API]
```

프론트엔드가 API 서버와 직접 통신한다.

- AccessToken: **JS 메모리**(변수, 상태)
- RefreshToken: **HttpOnly 쿠키**

### 왜 AccessToken을 쿠키에 넣지 않는가

**(1) CSRF 공격 위험**: 쿠키는 브라우저가 요청마다 자동으로 전송한다. AccessToken이
쿠키에 있으면, 사용자가 악성 사이트에 접속했을 때도 브라우저가 자동으로 쿠키를
포함해 요청을 보내고, 서버는 이를 인증된 요청으로 오인할 수 있다. SPA 구조에서
AccessToken을 쿠키에 두면 CSRF 방어가 필수가 되는데, 이 부담 때문에 보통 헤더에
명시적으로 붙이는 쪽을 택한다.

**(2) 프론트엔드가 인증 흐름을 직접 제어해야 함**: Authorization 헤더 구성, 토큰
만료 감지, 401 발생 시 재발급 — 이 흐름 전체를 프론트가 책임진다. AccessToken이
프론트가 직접 다뤄야 하는 자산이라면 쿠키보다 JS 메모리가 더 적합하다.

## BFF 구조

```
[Browser] --(쿠키)--> [BFF 서버] --(Authorization 헤더)--> [Backend API]
```

핵심은 **브라우저가 더 이상 AccessToken을 직접 다루지 않는다**는 것이다.

- 브라우저: 쿠키만 전송, 토큰의 존재를 모름
- BFF: 쿠키에서 토큰 추출 → 내부 API 호출 시 Authorization 헤더로 변환 → 인증/인가의
  책임 주체

토큰은 이제 **서버 자산**이 된다.

## 왜 BFF에서는 쿠키에 담는가

**XSS 위협 모델이 달라진다.** SPA 구조에서 XSS가 터지면 JS 메모리에 있는
AccessToken이 그대로 탈취되어 즉시 API 호출에 쓰일 수 있다. BFF 구조에서
AccessToken은 HttpOnly 쿠키에 있어 JS에서 접근 자체가 불가능하다 — XSS로 토큰을
직접 훔칠 수 없다.

**CSRF 방어를 한 곳에서 처리할 수 있다.** BFF에서 토큰을 쿠키에 넣는다고 CSRF
문제가 사라지는 건 아니다. 다만 SameSite 설정, CSRF 토큰 검증, Origin/Referer
검증을 **BFF 한 곳에서만** 구현하면 된다 — 여러 API 서버를 각각 보호해야 하는
SPA 구조보다 관리 지점이 단순해진다.

**프론트엔드가 단순해진다.** `fetch("/api/user")`만 호출하면 된다. Authorization
헤더 구성, 토큰 만료 처리, 인증 로직이 전부 사라지고 보안·인증 책임이 서버로
넘어간다.

## RefreshToken은 어디에 두는가

| 토큰 | 저장 위치 |
|---|---|
| AccessToken | HttpOnly 쿠키 |
| RefreshToken | HttpOnly 쿠키 (더 엄격하게는 서버 저장) |

보안 요구사항이 높은 경우(금융, 마이데이터 계열)에는 RefreshToken을 서버(DB/Redis)에만
저장하고, 쿠키에는 세션 식별자만 남기는 방식도 자주 쓰인다.

## 자주 나오는 오해

- ❌ **"BFF니까 AccessToken을 쿠키에 넣으면 무조건 안전하다"** — CSRF 방어(SameSite,
  CSRF 토큰)가 없으면 그대로 취약하다.
- ❌ **"쿠키에 넣으면 JWT를 쓰는 의미가 없다"** — JWT의 가치는 서버 확장성과
  무상태성이지, 전달 매체(쿠키냐 헤더냐)와는 별개 문제다.
- ⭕ **"BFF 환경에서는 AccessToken을 쿠키에 담는 게 합리적이다"** — 단, **CSRF 방어를
  전제로 할 때**다.

## 결론

| 구조 | AccessToken | RefreshToken |
|---|---|---|
| 순수 SPA + 다수 API | 메모리 | HttpOnly 쿠키 |
| BFF | HttpOnly 쿠키 | 서버 저장 또는 강화된 쿠키 |

BFF는 단순히 API 중계 서버가 아니라, **인증 책임을 브라우저에서 서버로 이동시키는
구조적 선택**이다. 이 관점에서 보면 BFF 환경에서 AccessToken을 쿠키에 담는 설계는
유행이 아니라 보안 모델에 따른 자연스러운 결과다. 다만 이 구조를 택하는 순간 BFF
서버 자체가 인증의 핵심 방어선이 되므로, CSRF 방어를 빠뜨리면 SPA 구조보다 오히려
더 취약해질 수 있다는 점은 분명히 해둬야 한다.
