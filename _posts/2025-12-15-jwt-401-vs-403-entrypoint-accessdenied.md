---
title: "JWT 인증에서 401과 403을 정확히 나누는 법 — EntryPoint vs AccessDeniedHandler"
date: 2025-12-15
categories: [개발일지, SpringSecurity]
tags: [spring-security, jwt]
---

JWT 기반 인증/인가를 Spring Security로 구현하다 보면 **401과 403을 어떻게 나눌 것인가**에서
막히는 경우가 많다. 이 구분이 흐트러지면 프론트의 에러 분기(로그인 만료 vs 권한 없음)가
꼬이고, 로그로 원인을 추적하기도 어려워진다. `JwtAuthenticationEntryPoint`와
`JwtAccessDeniedHandler`가 각각 언제 호출되고 무엇을 책임져야 하는지 정리한다.

## 401과 403의 역할부터 정리

| 상태코드 | 의미 |
|---|---|
| 401 Unauthorized | 인증 실패 (Authentication 실패) |
| 403 Forbidden | 인가 실패 (Authorization 실패) |

JWT 기준으로 풀면:

- **401 (인증 실패)**: 토큰 없음, 토큰 만료, 서명 불일치, 형식이 잘못된 토큰 —
  즉 "너 누구야?"에 답을 못 하는 상태
- **403 (인가 실패)**: 토큰은 유효하고 사용자는 인증됐지만, 해당 API에 접근할
  권한(Role)이 없는 상태 — "너인 건 알겠는데, 여기 들어올 권한은 없어"

이 둘의 근본적인 차이는 **인증(Authentication) 객체가 SecurityContext에 존재하는가**다.
401은 애초에 그 객체가 없거나 무효한 상태고, 403은 그 객체가 이미 성립된 뒤에
권한 체크에서 걸리는 상태다 — 그래서 Spring Security는 이 둘을 서로 다른 예외
타입(`AuthenticationException` vs `AccessDeniedException`)과 서로 다른
핸들러로 분리해서 처리하도록 설계돼 있다.

## `JwtAuthenticationEntryPoint`: 인증 자체가 실패했을 때

`AuthenticationEntryPoint`는 Security Filter Chain에서 `AuthenticationException`이
발생했을 때 호출된다 — 아직 인증이 성립되지 않은 상태에서 실패하는 모든 경우다
(Authorization 헤더 없음, Bearer 토큰 없음, JWT 만료, JWT 파싱 실패).

```java
@Component
public class JwtAuthenticationEntryPointHandler implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request,
                          HttpServletResponse response,
                          AuthenticationException authException) throws IOException {

        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json;charset=UTF-8");

        if (authException instanceof ExpiredTokenException) {
            response.getWriter().write("{\"code\":\"401\", \"message\":\"token expired\"}");
        } else if (authException instanceof InvalidTokenException) {
            response.getWriter().write("{\"code\":\"401\", \"message\":\"invalid token\"}");
        } else {
            response.getWriter().write("{\"code\":\"401\", \"message\":\"authentication failed\"}");
        }
    }
}
```

**핵심 원칙: 여기서는 무조건 401만 내려야 한다.** 만료된 토큰인지 위조된 토큰인지
같은 세부 원인은 메시지나 내부 코드로만 구분하고, HTTP 상태 코드 자체는 절대
바꾸지 않는다. 원인별로 상태 코드를 다르게 내려버리면 "401은 인증 실패"라는
클라이언트 쪽의 단순한 분기 규칙이 깨진다.

## `JwtAccessDeniedHandler`: 인증은 됐는데 권한이 없을 때

`AccessDeniedHandler`는 SecurityContext에 `Authentication`이 이미 존재하는
상태에서, `@PreAuthorize`나 `hasRole`, 경로 매칭 조건을 통과하지 못했을 때
호출된다.

```java
@Component
public class JwtAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request,
                        HttpServletResponse response,
                        AccessDeniedException accessDeniedException) throws IOException {

        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"code\":\"403\", \"message\":\"access denied\"}");
    }
}
```

이 핸들러는 딱 하나만 책임진다 — 403 반환. 인증 상태는 이미 확정된 뒤이므로
"누구인지"를 다시 판단할 필요가 없다는 게 EntryPoint와의 근본적인 차이다.

## SecurityConfig에서 연결

```java
http
    .exceptionHandling()
    .authenticationEntryPoint(jwtAuthenticationEntryPointHandler)
    .accessDeniedHandler(jwtAccessDeniedHandler);
```

이렇게 설정해두면 인증 실패는 EntryPoint로, 인가 실패는 AccessDeniedHandler로
자동 분기된다 — 단, 이 자동 분기가 성립하려면 한 가지 전제가 있다.

## 왜 JWT 필터 안에서 예외를 직접 잡으면 안 되는가

JWT를 파싱하고 검증하는 커스텀 필터는 `AuthenticationEntryPoint`보다 앞단에서
동작한다. 여기서 흔히 하는 실수가, 필터 안에서 만료/위조 예외를 `try-catch`로
직접 잡아서 즉시 `response.sendError(401)` 같은 걸 호출해버리는 것이다. 이렇게
하면 동작은 하지만, `JwtAuthenticationEntryPoint`가 아예 호출되지 않는 우회
경로가 생긴다 — 응답 형식이 EntryPoint가 만드는 JSON과 달라지고, 세부 원인 구분
로직(만료/위조 분기)을 필터 안에도 중복으로 짜야 한다.

올바른 방식은 필터 안에서 `ExpiredTokenException`, `InvalidTokenException` 같은
`AuthenticationException`의 서브타입을 **그대로 던지고 잡지 않는 것**이다. Spring
Security의 필터 체인이 이 예외를 가로채서 `AuthenticationEntryPoint.commence()`를
호출하도록 위임하면, 응답 형식과 상태 코드 처리가 한 곳(`EntryPointHandler`)으로
모인다. "예외를 어디서 잡을 것인가"를 필터가 아니라 EntryPoint로 일원화하는 게
핵심이다.

## 실무에서 자주 터지는 실수

- **EntryPoint에서 403을 내려버리는 경우**: 토큰 없음이나 만료를 403으로
  처리하면, 프론트는 "로그인이 만료됐다"와 "권한이 없다"를 구분할 수 없게 된다 —
  전자는 재로그인 유도, 후자는 다른 화면으로 리다이렉트처럼 서로 다른 UX 분기가
  필요한데, 상태 코드가 같으면 이 분기 자체가 불가능해진다.
- **인증/인가 예외를 커스텀 예외 하나로 뭉뚱그리는 경우**: 결과적으로 항상 같은
  상태 코드만 나오게 되고, 로그에서도 "인증 문제인지 권한 문제인지"를 구분할 수
  없어 장애 대응이 느려진다.

## 정리

- 401은 `AuthenticationException` 계열 → `AuthenticationEntryPoint` → 무조건 401.
- 403은 `AccessDeniedException` → `AccessDeniedHandler` → 무조건 403.
- JWT 필터 안에서는 예외를 잡지 않고 그대로 던져서, 상태 코드 결정을 Security의
  표준 예외 처리 경로(EntryPoint/AccessDeniedHandler)에 맡긴다.

이 원칙이 지켜지면 클라이언트는 상태 코드만으로 분기할 수 있고, 서버는 로그만으로
원인을 추적할 수 있다 — 둘 중 하나라도 흔들리면 그 대가는 디버깅 시간으로 돌아온다.
