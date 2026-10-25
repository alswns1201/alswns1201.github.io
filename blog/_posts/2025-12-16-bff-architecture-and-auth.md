---
title: "BFF(Backend for Frontend): 언제 도입하고, 인증은 어떻게 설계하는가"
date: 2025-12-16
categories: [개발 고민/설계]
---

프론트엔드가 백엔드 API를 직접 호출하는 구조는 프로젝트 초기엔 별문제가 없다. 문제는
클라이언트가 여러 개(웹, iOS, Android)로 늘어나고, 화면 하나에 필요한 데이터가 여러
API에 흩어져 있을 때부터 시작된다. BFF는 이 지점에서 등장하는 패턴이다.

이 글은 BFF가 언제 필요하고 언제 과한지부터 시작해서, BFF를 인증 계층으로 쓸 때
실제로 부딪히는 두 가지 질문 — **토큰을 어디에 저장할 것인가**, **저장한 토큰을
어떻게 "진짜로" 검증할 것인가** — 를 순서대로 다룬다.

## BFF가 푸는 문제

전통적인 구조에서는 하나의 백엔드 API 서버가 모든 클라이언트의 요청을 처리한다. BFF
패턴에서는 각 프론트엔드(웹, iOS, Android 등)가 자신만을 위한 전담 백엔드를 갖는다 —
프론트엔드와 실제 백엔드(MSA, 레거시 API 등) 사이의 중개자 역할이다.

### 시나리오 1: 여러 API를 조합해야 화면 하나가 완성될 때

상품 상세 페이지 하나를 그리는 데 상품 정보, 리뷰, 쿠폰, 추천 상품 API가 각각
필요하다고 하자. BFF가 없으면 프론트엔드가 4개 API를 직접 호출하고, 응답을 기다리고,
클라이언트 측에서 조합해야 한다 — 네트워크 요청이 늘고 프론트엔드 로직이 복잡해진다.

BFF는 `/view/product-detail/{id}` 하나의 요청만 받아서 내부적으로 4개 API를 조합해
화면에 필요한 형태로 응답한다.

### 시나리오 2: 민감 정보를 걸러내야 할 때

백엔드가 반환하는 사용자 객체에 `internalAdminNote` 같은 내부용 필드가 섞여 있다면,
이걸 그대로 클라이언트에 흘려보내는 건 보안 사고다. BFF가 중간에서 필요한 필드만
추려서 전달하면 이 문제가 원천 차단된다.

### 시나리오 3~4: 무거운 연산, 외부 API 키 관리

필터링/정렬 같은 무거운 연산을 저사양 클라이언트에서 처리하게 두지 않고 BFF가
대신 수행하거나, 외부 Open API 키를 클라이언트에 노출하지 않고 BFF 서버에서만
보관하는 것도 BFF의 전형적인 역할이다.

## 실전: Next.js(BFF) + Spring Boot 로그인 예제

```
[사용자 브라우저] ↔ [Next.js 서버 (BFF)] ↔ [Spring Boot 서버 (백엔드)]
```

Spring Boot는 인증 로직만 처리하고 순수한 JWT 문자열을 반환한다 — 웹 환경(쿠키,
브라우저 등)을 전혀 신경 쓰지 않는다.

```java
@PostMapping("/login/{provider}")
public ResponseEntity<String> login(/*...*/) {
    String jwtToken = authService.createJwtToken(user);
    return ResponseEntity.ok(jwtToken); // 오직 토큰 문자열만
}
```

Next.js Route Handler(BFF)가 이 순수한 토큰을 받아 웹 환경에 맞게 가공한다.

```ts
export async function POST(request: NextRequest) {
  const jwtToken = await backendResponse.text();

  const response = NextResponse.json({ message: '로그인 성공' });
  response.cookies.set('accessToken', jwtToken, {
    httpOnly: true,   // JS에서 접근 불가 (XSS 방어)
    path: '/',
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
  });

  return response;
}
```

이 설계의 핵심은 **JWT가 JavaScript 코드에 노출될 가능성을 아예 없앤다**는 것과,
**백엔드가 웹이라는 특정 클라이언트 환경에 대한 의존성을 갖지 않는다**는 것이다.
백엔드는 플랫폼 독립적인 순수 비즈니스 로직에만 집중하고, BFF가 클라이언트별 경험
(보안, 데이터 포맷)을 책임진다.

## 그런데, 항상 BFF가 정답은 아니다

여기까지만 보면 BFF가 만능처럼 보이지만, 공짜가 아니다.

- **네트워크 홉이 하나 늘어난다.** 브라우저 → BFF → 백엔드 구조는 브라우저 →
  백엔드 직접 호출보다 지연 시간이 늘어난다. 지연에 민감한 서비스라면 이 비용을
  먼저 재봐야 한다.
- **운영해야 할 서버가 하나 더 생긴다.** BFF도 배포, 모니터링, 장애 대응이 필요한
  독립 서비스다. 클라이언트가 하나뿐이고 팀 규모가 작다면, BFF를 도입하는 순간
  "관리해야 할 컴포넌트 수"가 늘어나는 비용이 얻는 이득보다 클 수 있다.
- **클라이언트가 여러 개면 BFF도 여러 개가 될 수 있다.** 웹용 BFF, 모바일용 BFF를
  따로 두면 각 BFF에 비슷한 조합/가공 로직이 중복될 위험이 있다 — 이 중복을
  줄이려고 공통 로직을 또 다른 계층으로 뽑다 보면 구조가 다시 복잡해진다.

BFF는 "관심사의 분리"라는 원칙을 지키는 설계지만, 그 원칙이 필요한 복잡도에
도달했을 때만 이득이 비용을 넘어선다. 클라이언트가 하나뿐이고 화면-API가 1:1에
가깝다면 굳이 도입할 이유가 없다. 반대로 여러 클라이언트가 각기 다른 데이터 형태를
요구하고, 민감 정보 필터링이나 외부 API 키 관리가 반복적으로 필요하다면 그때부터
BFF의 이득이 명확해진다.

BFF를 인증 계층으로 쓰기로 했다면, 다음 질문은 자연스럽게 "그럼 토큰을 어디에
저장할 것인가"다.

## 왜 BFF에서는 AccessToken을 쿠키에 담는가

JWT 기반 인증을 설계하다 보면 항상 나오는 질문이 있다.

> AccessToken과 RefreshToken이 있는데, 왜 어떤 서비스는 AccessToken을 쿠키(HttpOnly)에
> 넣고, 어떤 곳은 절대 쿠키에 안 넣을까?

- **AccessToken**: API 접근 권한 증명, 짧은 만료 시간(보통 5~15분)
- **RefreshToken**: AccessToken 재발급용, 상대적으로 긴 만료 시간

역할은 SPA든 BFF든 동일하다. 차이는 **브라우저가 이 토큰을 직접 다루느냐**에 있다.

### 일반적인 SPA + API 구조

```
[Browser SPA] → Authorization: Bearer AccessToken → [Backend API]
```

프론트엔드가 API 서버와 직접 통신한다.

- AccessToken: **JS 메모리**(변수, 상태)
- RefreshToken: **HttpOnly 쿠키**

왜 AccessToken을 쿠키에 넣지 않는가:

**(1) CSRF 공격 위험**: 쿠키는 브라우저가 요청마다 자동으로 전송한다. AccessToken이
쿠키에 있으면, 사용자가 악성 사이트에 접속했을 때도 브라우저가 자동으로 쿠키를
포함해 요청을 보내고, 서버는 이를 인증된 요청으로 오인할 수 있다. SPA 구조에서
AccessToken을 쿠키에 두면 CSRF 방어가 필수가 되는데, 이 부담 때문에 보통 헤더에
명시적으로 붙이는 쪽을 택한다.

**(2) 프론트엔드가 인증 흐름을 직접 제어해야 함**: Authorization 헤더 구성, 토큰
만료 감지, 401 발생 시 재발급 — 이 흐름 전체를 프론트가 책임진다. AccessToken이
프론트가 직접 다뤄야 하는 자산이라면 쿠키보다 JS 메모리가 더 적합하다.

### BFF 구조

```
[Browser] --(쿠키)--> [BFF 서버] --(Authorization 헤더)--> [Backend API]
```

핵심은 **브라우저가 더 이상 AccessToken을 직접 다루지 않는다**는 것이다.

- 브라우저: 쿠키만 전송, 토큰의 존재를 모름
- BFF: 쿠키에서 토큰 추출 → 내부 API 호출 시 Authorization 헤더로 변환 → 인증/인가의
  책임 주체

토큰은 이제 **서버 자산**이 된다.

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

| 구조 | AccessToken | RefreshToken |
|---|---|---|
| 순수 SPA + 다수 API | 메모리 | HttpOnly 쿠키 |
| BFF | HttpOnly 쿠키 | 서버 저장 또는 강화된 쿠키 |

자주 나오는 오해:

- ❌ **"BFF니까 AccessToken을 쿠키에 넣으면 무조건 안전하다"** — CSRF 방어(SameSite,
  CSRF 토큰)가 없으면 그대로 취약하다.
- ❌ **"쿠키에 넣으면 JWT를 쓰는 의미가 없다"** — JWT의 가치는 서버 확장성과
  무상태성이지, 전달 매체(쿠키냐 헤더냐)와는 별개 문제다.
- ⭕ **"BFF 환경에서는 AccessToken을 쿠키에 담는 게 합리적이다"** — 단, **CSRF 방어를
  전제로 할 때**다.

BFF는 단순히 API 중계 서버가 아니라, **인증 책임을 브라우저에서 서버로 이동시키는
구조적 선택**이다. 다만 쿠키에 담는 순간 "쿠키에 토큰이 있다"는 사실 자체를 인증
성공으로 착각하기 쉬운데, 이건 반쪽짜리 방어다.

## httpOnly 쿠키만으로는 부족하다 — 미들웨어가 진짜 검증하게 만들기

httpOnly 쿠키로 JWT를 저장하면 자바스크립트가 토큰에 접근할 수 없으니 XSS로 토큰이
털릴 위험은 막힌다. 그런데 여기서 끝내면 반쪽짜리 방어다. "쿠키가 존재한다"는 것과
"그 안의 토큰이 아직 유효하다"는 건 전혀 다른 질문이기 때문이다.

### 반쪽짜리 해결책: 존재 여부만 확인하는 미들웨어

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

### 해법: 토큰을 발급한 서버만이 유효성을 보증할 수 있다

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

### 놓치기 쉬운 두 가지

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

## 정리

1. 로그인 시 Next.js(BFF)가 Spring API로부터 JWT를 받아 httpOnly 쿠키에 담아 저장 (XSS 방어)
2. 사용자가 보호된 페이지로 이동
3. Middleware가 요청을 가로채 쿠키를 확인
4. Middleware가 Spring의 `/api/auth/validate`로 실시간 검증 요청
5. Spring이 서명·만료·유효성을 검사해 응답
6. Middleware가 응답에 따라 통과 또는 리다이렉트

BFF는 "관심사의 분리"가 필요한 복잡도에 도달했을 때 이득이 비용을 넘어서는
패턴이고, 그 위에 인증을 얹을 땐 "검증은 발급자만 할 수 있다"는 원칙과 그 대가로
감수하는 요청당 네트워크 왕복 비용을 의식적으로 받아들여야 한다.
