---
title: "localStorage와 sessionStorage, 그리고 둘 다 인증 토큰을 두면 안 되는 이유"
date: 2025-06-16
categories: [프론트엔드]
tags: [frontend, security]
---

localStorage와 sessionStorage는 API가 거의 동일해서 (`setItem`, `getItem`,
`removeItem`, `clear`) 차이를 "지속 기간"으로만 외우기 쉽다. 하지만 실무에서 이
둘을 고르는 진짜 기준은 지속 기간이 아니라 **"이 데이터가 노출되면 얼마나 위험한가"**다.

## 기본 차이: 지속성과 범위

| 특징 | localStorage | sessionStorage | Cookie |
|---|---|---|---|
| 지속성 | 명시적 삭제 전까지 영구 | 탭/창이 닫히면 소멸 | 만료 기간 설정 |
| 접근 범위 | 동일 origin의 모든 탭/창 | 동일 origin이라도 현재 탭만 | 동일 origin (경로 설정 가능) |
| 용량 | 5~10MB | 5~10MB | 약 4KB |
| 서버 전송 | 안 됨 (클라이언트 전용) | 안 됨 | HTTP 요청마다 자동 전송 |
| XSS 취약성 | 취약 (JS로 직접 접근 가능) | 취약 | HttpOnly면 JS 접근 차단 |

용량과 API가 압도적으로 좋아 보이는 localStorage/sessionStorage가 쿠키를 완전히
대체하지 못하는 이유가 표의 마지막 줄이다.

## 왜 인증 토큰을 localStorage에 두면 안 되는가

localStorage는 자바스크립트로 직접 읽고 쓸 수 있다는 게 장점이자 동시에 가장 큰
약점이다. 페이지에 XSS(악성 스크립트 삽입) 취약점이 하나라도 있으면, 공격자의
스크립트는 `document.cookie`(HttpOnly가 아닌 경우)뿐 아니라 `localStorage`도
아무 제약 없이 통째로 읽어갈 수 있다. JWT 같은 인증 토큰을 localStorage에
저장했다면, 그 순간 XSS 취약점 하나가 곧바로 계정 탈취로 이어진다.

`HttpOnly` 쿠키는 이 문제를 구조적으로 막는다. `HttpOnly` 플래그가 붙은 쿠키는
자바스크립트의 `document.cookie`로 아예 접근이 안 되기 때문에, XSS가 터져도
스크립트가 토큰을 훔쳐갈 방법이 없다. 이게 "인증 토큰은 HttpOnly 쿠키에 두라"는
조언의 근거다.

## 그런데 "HttpOnly 쿠키가 정답"도 반쪽짜리 답이다

HttpOnly로 XSS 문제는 막았지만, 쿠키는 브라우저가 매 요청마다 **자동으로**
서버에 실어 보낸다. 이 자동 전송 특성이 이번엔 CSRF(Cross-Site Request Forgery)
공격의 통로가 된다 — 사용자가 로그인된 상태로 악성 사이트를 열면, 그 사이트가 만든
요청에도 브라우저가 쿠키를 자동으로 붙여서 보내버린다.

그래서 실제로는 `HttpOnly` + `SameSite` 속성 조합, 그리고 필요하면 별도의
CSRF 토큰까지 함께 써야 안전하다. "localStorage는 XSS에 취약하니 쿠키로
바꾸면 끝"이 아니라, 저장 위치를 바꾸는 순간 새로운 공격 표면(CSRF)이 열린다는
것까지 같이 이해해야 한다.

## 실무에서 흔히 쓰는 절충: Access Token은 메모리, Refresh Token은 HttpOnly 쿠키

한 가지 방식이 여러 SPA에서 쓰인다 — 수명이 짧은 Access Token은 자바스크립트
변수(메모리)에만 두고 새로고침하면 사라지게 하고, 수명이 긴 Refresh Token만
`HttpOnly` + `Secure` + `SameSite=Strict` 쿠키에 둔다. Access Token이
메모리에만 있으면 XSS가 터져도 훔쳐갈 수 있는 건 "그 순간의 짧은 토큰" 하나뿐이고,
페이지를 새로고침하면 이미 사라진 상태다. Refresh Token은 애초에 자바스크립트가
접근할 수 없으니 XSS로도 못 훔친다.

## localStorage/sessionStorage가 여전히 적합한 경우

인증 토큰을 빼면 이 둘은 여전히 유용하다:

- **localStorage**: 다크모드, 언어 설정처럼 노출돼도 위험하지 않고 영구적으로
  유지되어야 하는 사용자 설정.
- **sessionStorage**: 다단계 폼의 중간 입력값처럼, 새로고침엔 살아남아야 하지만
  탭을 닫으면 사라져도 되는 일시적 데이터.

한 가지 덜 알려진 기능은 `storage` 이벤트다 — 같은 origin의 다른 탭에서
localStorage 값이 바뀌면 `window.addEventListener('storage', ...)`로 감지할 수
있다. 로그아웃을 한 탭에서 하면 다른 탭들도 동기화해서 로그아웃 처리하는 식의
탭 간 상태 동기화에 별도 라이브러리 없이 쓸 수 있는 기본 브라우저 기능이다.

## 정리

- 지속 기간이 아니라 **"노출되면 얼마나 위험한 데이터인가"**로 저장 위치를 고른다.
- 인증 토큰: localStorage 금지. Access Token은 메모리, Refresh Token은
  HttpOnly+Secure+SameSite 쿠키가 현실적인 절충안.
- HttpOnly 쿠키로 XSS는 막아도 CSRF는 별도로 막아야 한다 — 저장 위치를 바꾸는 건
  공격 표면을 옮기는 것이지 없애는 게 아니다.
- 그 외의 비민감 데이터(설정값, 임시 폼 데이터)에는 localStorage/sessionStorage가
  여전히 가장 간단한 선택이다.
