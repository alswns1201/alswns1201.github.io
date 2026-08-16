---
title: "BFF에서 httpOnly JWT를 미들웨어가 '진짜' 검증하게 만들기"
date: 2025-06-24
categories: [아키텍쳐 설계 관련 글, BFF]
tags: [spring-security, jwt, bff]
---

httpOnly 쿠키로 JWT를 저장하면 자바스크립트가 토큰에 접근할 수 없으니 XSS로 토큰이
털릴 위험은 막힌다. 그런데 여기서 끝내면 반쪽짜리 방어다. "쿠키가 존재한다"는 것과
"그 안의 토큰이 아직 유효하다"는 건 전혀 다른 질문이기 때문이다.

## 반쪽짜리 해결책: 존재 여부만 확인하는 미들웨어

```ts
// middleware.ts (초기 버전)
export function middleware(request: NextRequest) {
  const accessToken = request.cookies.get('accessToken')?.value;
  if (!accessToken && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  return NextResponse.next();
}
```

이 코드는 "쿠키가 없으면 막는다"만 할 수 있다. 다음 상황들을 전혀 구분하지 못한다:

- 토큰이 만료됐다면?
- 토큰 payload가 위조됐다면?
- 서버가 이 사용자를 강제로 로그아웃시켰다면?

이 경우 사용자는 무효한 토큰을 가진 채 페이지에 진입하고, 이후 API 호출이 줄줄이
401로 실패하면서 화면이 깨진다. "쿠키 존재 확인"은 인증이 아니라 **인증이 있는 척**만
하는 것이다.

## 해법: 토큰을 발급한 서버만이 유효성을 보증할 수 있다

검증 흐름을 다시 설계한다.

1. Spring에 토큰 검증 전용 엔드포인트(`/api/auth/validate`)를 만든다.
2. Next.js Middleware는 보호된 경로 요청을 가로챌 때마다 이 API를 호출한다.
3. Spring이 쿠키의 JWT를 검증하고 성공/실패를 응답한다.
4. Middleware는 응답에 따라 요청을 통과시키거나 로그인 페이지로 리다이렉트한다.

```java
@GetMapping("/validate")
public ResponseEntity<?> validateToken(
        @CookieValue(name = "accessToken", required = false) String accessToken
) {
    if (accessToken == null || !jwtUtil.validateToken(accessToken)) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("Invalid or expired token");
    }
    String username = jwtUtil.getUsernameFromToken(accessToken);
    return ResponseEntity.ok().body(Map.of("username", username, "isValid", true));
}
```

이 엔드포인트는 Security 설정에서 `permitAll()`이어야 한다 — 인증을 수행하는
엔드포인트 자체가 인증을 요구하면 순환이 생긴다.

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/login", "/api/auth/validate").permitAll()
    .anyRequest().authenticated()
);
```

```ts
// middleware.ts (최종)
const response = await fetch(`${API_BASE_URL}/api/auth/validate`, {
  headers: { 'Cookie': `accessToken=${accessToken}` }
});

if (response.ok) return NextResponse.next();
return NextResponse.redirect(new URL('/login', request.url));
```

## 놓치기 쉬운 두 가지

**1) 검증 엔드포인트가 새로운 공격 표면이 된다.** `permitAll()`로 열려 있고 요청마다
호출되는 엔드포인트이므로, 별도의 rate limiting을 걸지 않으면 이 엔드포인트 자체가
무차별 대입(브루트포스)이나 DoS의 대상이 될 수 있다. 인증을 지키기 위해 연 문이
새로운 취약점이 되는 셈이라 이 엔드포인트는 반드시 별도의 요청 제한을 걸어야 한다.

**2) 페이지 요청마다 네트워크 왕복이 하나씩 추가된다.** Middleware가 보호된 페이지에
진입할 때마다 Spring 서버를 호출하므로, 사용자 입장에서는 페이지 이동마다 지연이 하나
더 생긴다. 이건 이 아키텍처가 의도적으로 감수하는 트레이드오프다 — JWT의 원래 장점은
서명만 검증하면 서버 상태 조회 없이 로컬에서 유효성을 판단할 수 있다는 것인데, 여기서는
그 장점을 포기하고 매번 네트워크로 확인한다. 왜 그런가 하면, 로컬 서명 검증만으로는
"서버가 강제로 로그아웃시켰다"는 사실을 알 수 없기 때문이다 — 서명은 여전히 유효하므로
로컬 검증은 통과해버린다. 즉시 무효화(강제 로그아웃, 계정 정지 등)가 정말 필요하다면
이 네트워크 왕복은 대가로 치를 수밖에 없다. 반대로 즉시 무효화가 중요하지 않은
서비스라면, 매 요청 검증 대신 공개키로 서명만 로컬에서 확인하고 만료 시간을 짧게
가져가는 순수 stateless 방식이 지연 시간 면에서 더 유리하다 — 이건 세션 기반 인증과
JWT 기반 인증의 근본적인 트레이드오프(즉시성 vs 확장성)가 BFF 레이어에서도 똑같이
반복되는 것이다.

## 결론

JWT 인증 아키텍처의 전체 흐름:

1. 로그인 시 Next.js(BFF)가 Spring API로부터 JWT를 받아 httpOnly 쿠키에 담아 저장 (XSS 방어)
2. 사용자가 보호된 페이지로 이동
3. Middleware가 요청을 가로채 쿠키를 확인
4. Middleware가 Spring의 `/api/auth/validate`로 실시간 검증 요청
5. Spring이 서명·만료·유효성을 검사해 응답
6. Middleware가 응답에 따라 통과 또는 리다이렉트

이 구조의 핵심은 "검증은 발급자만 할 수 있다"는 원칙과, 그 대가로 감수하는 요청당
네트워크 왕복 비용을 의식적으로 받아들이는 것이다.
