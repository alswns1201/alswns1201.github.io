---
title: "BFF(Backend for Frontend) 아키텍처, 언제 필요하고 언제 과한가"
date: 2025-06-23
categories: [개발 고민/설계]
---

프론트엔드가 백엔드 API를 직접 호출하는 구조는 프로젝트 초기엔 별문제가 없다. 문제는
클라이언트가 여러 개(웹, iOS, Android)로 늘어나고, 화면 하나에 필요한 데이터가 여러
API에 흩어져 있을 때부터 시작된다. BFF는 이 지점에서 등장하는 패턴이다.

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

## 판단 기준

BFF는 "관심사의 분리"라는 원칙을 지키는 설계지만, 그 원칙이 필요한 복잡도에
도달했을 때만 이득이 비용을 넘어선다. 클라이언트가 하나뿐이고 화면-API가 1:1에
가깝다면 굳이 도입할 이유가 없다. 반대로 여러 클라이언트가 각기 다른 데이터 형태를
요구하고, 민감 정보 필터링이나 외부 API 키 관리가 반복적으로 필요하다면 그때부터
BFF의 이득이 명확해진다.
