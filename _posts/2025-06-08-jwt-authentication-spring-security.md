---
title: "Spring Security + JWT 토큰 기반 인증 구축하기"
date: 2025-06-08
categories: [Java/Spring]
---

세션 대신 JWT를 쓰는 이유는 "요즘 다 그렇게 하니까"가 아니라 구체적인 트레이드오프
때문이다. 이 트레이드오프를 이해하지 못하고 도입하면, JWT의 단점만 떠안고 장점은
못 누리는 구성이 되기 쉽다.

## 왜 세션이 아니라 토큰인가

세션 방식은 서버가 인증 상태를 메모리나 DB에 저장한다(Stateful). 구현은 간단하지만,
서버를 여러 대로 확장(Scale-out)하면 "어느 서버가 이 세션을 가지고 있는가" 문제가
생긴다 — 로드밸런서가 다른 서버로 요청을 보내면 세션을 못 찾는다. 이걸 Sticky
Session이나 세션 클러스터링(Redis 등 외부 세션 저장소)으로 해결하는데, 결국 별도
인프라가 필요해진다.

토큰 방식은 서버가 상태를 안 가진다(Stateless) — 토큰 자체에 필요한 정보가 담겨
있고, 서버는 서명만 검증하면 된다. 그래서 서버를 몇 대로 늘려도 별도 세션 동기화가
필요 없다. 대가는 **서버가 토큰을 강제로 무효화할 방법이 마땅치 않다**는 것이다 —
세션은 서버가 저장소에서 지우면 즉시 끝나지만, JWT는 만료 시간이 될 때까지 유효하다.
이 비대칭을 이해하지 못하면 "로그아웃했는데 토큰이 왜 아직 유효하지"라는 질문에
답할 수 없다.

## JwtTokenProvider — 토큰의 생성/검증을 한 곳에 모으는 이유

```java
public String generateToken(Authentication authentication) {
    String authorities = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.joining(","));

    long now = (new Date()).getTime();
    Date accessTokenExpiresIn = new Date(now + 86400000); // 1일

    return Jwts.builder()
            .setSubject(authentication.getName())
            .claim("auth", authorities)
            .setExpiration(accessTokenExpiresIn)
            .signWith(key, SignatureAlgorithm.HS256)
            .compact();
}
```

토큰 생성/검증 로직을 한 클래스에 모으는 이유는 단순 편의가 아니라, **서명 키를
다루는 지점을 하나로 좁히기 위해서**다. 서명 키가 여러 클래스에 흩어져 있으면 키
교체(rotation) 한 번 하려 해도 코드 전체를 뒤져야 한다.

여기서 눈여겨볼 설계 선택이 하나 있다: 권한 정보를 `authorities` 문자열(콤마 구분)로
토큰에 직접 넣었다. 이건 매 요청마다 DB에서 권한을 다시 조회하지 않아도 된다는
장점이 있지만, **권한이 바뀌면 토큰이 만료되기 전까지 반영이 안 된다**는 단점을
동반한다. 관리자가 한 유저의 권한을 박탈해도, 그 유저가 이미 발급받은 토큰은 만료
시간까지 계속 유효하다 — 이게 바로 앞서 말한 "무효화가 안 된다" 문제의 구체적인
사례다. 권한 변경이 즉시 반영되어야 하는 서비스라면, 토큰에 권한을 굳이 담지 않고
매 요청마다 DB/캐시에서 조회하는 편이 안전하다.

## JwtAuthenticationFilter — 왜 필터 단계에서 처리하는가

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
        throws ServletException, IOException {
    String token = resolveToken(request);
    if (token != null && jwtTokenProvider.validateToken(token)) {
        Authentication authentication = jwtTokenProvider.getAuthentication(token);
        SecurityContextHolder.getContext().setAuthentication(authentication);
    }
    filterChain.doFilter(request, response);
}
```

인증을 컨트롤러가 아니라 필터에서 처리하는 이유는, 컨트롤러에 도달하기 *전에*
인증 여부를 결정해야 하기 때문이다. 컨트롤러 안에서 인증을 체크하면 모든 컨트롤러
메서드마다 같은 코드를 반복해야 하고, 실수로 하나라도 빠뜨리면 그 엔드포인트만
인증 없이 뚫린다. 필터는 요청 경로에 무관하게 공통으로 적용되므로 이런 누락을
구조적으로 막는다.

## SecurityConfig — 필터 순서가 왜 중요한가

```java
.addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider), UsernamePasswordAuthenticationFilter.class);
```

`formLogin`을 비활성화했기 때문에 `UsernamePasswordAuthenticationFilter`는 실제로는
동작하지 않는다. 그런데도 이 필터를 기준점으로 삼아 그 앞에 JWT 필터를 넣는 이유는,
Spring Security의 필터 체인이 **등록 순서대로 실행**되기 때문이다. Security의 기본
필터 체인 안에는 이미 정해진 순서가 있고, `addFilterBefore`로 "이 필터보다는 앞에서
실행되어야 한다"는 상대적 위치를 지정하는 게 절대 인덱스를 지정하는 것보다 안전하다
— Spring Security 버전이 올라가면서 기본 필터 체인 구성이 바뀌어도 상대적 순서는
유지되기 때문이다.

## 이 구조에서 빠진 것: Refresh Token

여기 구현에는 Access Token 하나만 있고 만료되면 재로그인해야 한다. 실서비스에서는
보통 Access Token(짧은 수명, 예: 30분~1시간)과 Refresh Token(긴 수명, httpOnly
쿠키)을 분리해서, Access Token이 만료되면 Refresh Token으로 재발급받는 구조를
쓴다 — 그래야 "탈취되어도 피해가 짧게 끝나는 토큰"과 "탈취되면 안 되니 접근을
엄격히 제한하는 토큰"을 분리할 수 있다. Access Token 하나에 1일짜리 수명을 주는
건 탈취 시 노출 창이 그만큼 길다는 뜻이라, 실제로는 짧게 잡고 재발급 흐름을 따로
두는 쪽이 일반적이다.

## 정리

- 세션 vs JWT의 핵심은 Stateful/Stateless 트레이드오프이지, 유행이 아니다.
- JWT의 근본적 약점은 **즉시 무효화가 안 된다**는 것 — 권한을 토큰에 담을지 말지도
  이 약점과 직결된 설계 결정이다.
- 인증을 필터 단에서 처리하는 이유는 컨트롤러별 누락을 구조적으로 막기 위해서다.
- Access/Refresh 분리 없이 장수명 Access Token 하나만 쓰는 구성은 탈취 시 피해
  범위가 크다 — 실서비스로 갈수록 이 부분을 보완해야 한다.
