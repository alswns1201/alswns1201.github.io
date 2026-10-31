---
title: "CORS와 SameSite는 브라우저가 지키는 규칙이다"
date: 2026-10-31 10:00:00 +0900
categories: [프론트엔드]
---

CORS 에러가 나면 "서버가 요청을 막았다"고 생각하기 쉽다. SameSite도 "서버가 쿠키를 지켜 준다"고 오해하기 쉽다.
둘 다 사실은 **서버는 규칙을 응답 헤더로 알려 주기만 하고, 지키는 건 브라우저**다.
이걸 놓치면 "CORS를 걸었으니 외부에서 API를 못 부른다", "SameSite를 걸었으니 CSRF는 끝났다" 같은 잘못된 결론이 나온다.

## 먼저: origin과 site는 다르다

두 규칙은 기준이 다르다. CORS는 **origin**을 보고, SameSite는 **site**를 본다.

| 구분 | 기준 | `https://www.example.com` 과 `https://api.example.com` |
|---|---|---|
| origin | scheme + host + port 가 모두 같아야 같음 | **다른 origin** (host가 다름) |
| site | scheme + 등록 가능한 도메인(eTLD+1) 이 같으면 같음 | **같은 site** (`example.com`) |

- `http://example.com` 과 `https://example.com`은 scheme이 달라서 cross-origin이고, 지금 브라우저 기준(schemeful same-site)으로는 cross-site이기도 하다.
- `eTLD`는 Public Suffix List 기준이다. `github.io`는 public suffix라서 `a.github.io`와 `b.github.io`는 **서로 다른 site**다.
  GitHub Pages에 올린 블로그끼리 쿠키를 같은 site로 취급받지 못하는 이유다.

이 차이 때문에 `www.example.com` → `api.example.com` 요청은 **CORS 검사는 받지만 SameSite에는 걸리지 않는다.**

## CORS: 서버는 응답에 허락을 적고, 브라우저가 판단한다

브라우저에는 기본으로 **동일 출처 정책(SOP)**이 있다. 다른 origin의 응답을 JavaScript가 읽지 못하게 막는 규칙이다.
CORS는 이 제한을 **서버가 응답 헤더로 풀어 주는 방법**이다.

```
브라우저(JS: www.example.com)                 서버(api.example.com)
  fetch('https://api.example.com/me') ───────▶  요청 처리 (실제로 실행됨)
                                     ◀───────  200 OK
                                               Access-Control-Allow-Origin: ???
  브라우저가 헤더 확인
   - 허용된 origin이면 → JS에 응답 전달
   - 아니면           → JS에는 CORS 에러, 응답은 버림
```

핵심은 **요청은 서버에 이미 도착해서 처리됐다**는 점이다. 막힌 건 "JS가 응답을 읽는 것"이다.

### 단순 요청과 preflight

모든 요청이 그대로 나가는 건 아니다. 브라우저는 요청을 두 종류로 나눈다.

| 종류 | 조건 | 동작 |
|---|---|---|
| 단순 요청 | `GET`/`HEAD`/`POST` + 기본 헤더만 + `Content-Type`이 `text/plain`, `application/x-www-form-urlencoded`, `multipart/form-data` 중 하나 | 바로 보낸다. 응답을 읽을지만 나중에 판단 |
| preflight 대상 | 그 밖의 경우 (`application/json`, `Authorization` 헤더, `PUT`/`DELETE` 등) | 먼저 `OPTIONS`로 허락을 묻고, 허락받으면 실제 요청을 보낸다 |

```http
OPTIONS /orders HTTP/1.1
Origin: https://www.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type, authorization

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: content-type, authorization
Access-Control-Max-Age: 3600
```

JSON API는 대부분 preflight 대상이라 "실패하면 실제 요청이 안 나간다"고 느끼기 쉽다.
하지만 **단순 요청은 CORS가 실패해도 서버에서 실행된다.** form으로 보내는 `POST`가 여기에 해당한다.

### 그래서 CORS는 접근 제어가 아니다

- `curl`, Postman, 서버 간 호출은 CORS를 전혀 신경 쓰지 않는다. CORS는 **브라우저 안의 JS**에만 적용된다.
- 단순 요청은 CORS와 상관없이 서버에서 실행되므로, **CORS는 CSRF 방어도 아니다.**
- "우리 프론트에서만 API를 부르게 하고 싶다"는 요구는 CORS로 해결되지 않는다. 인증·인가로 해결해야 한다.

CORS가 막아 주는 건 하나다. **사용자 브라우저에 로그인된 상태(쿠키)를 이용해, 다른 사이트의 JS가 응답 내용을 훔쳐 읽는 것.**

### 쿠키를 같이 보낼 때 (credentials)

다른 origin으로 쿠키를 보내고 응답까지 읽으려면 세 쪽이 모두 맞아야 한다.

```js
fetch('https://api.example.com/me', { credentials: 'include' });   // 1. 클라이언트가 명시
```

```http
Access-Control-Allow-Origin: https://www.example.com   # 2. *가 아니라 정확한 origin
Access-Control-Allow-Credentials: true                 #    + credentials 허용
Vary: Origin                                           #    origin마다 응답이 다르므로 캐시 분리
```

3. 그리고 쿠키 자체가 그 요청에 실릴 수 있어야 한다. 이건 아래 SameSite가 정한다.

Spring에서 `allowedOrigins("*")`와 `allowCredentials(true)`를 같이 쓰면 시작할 때 `IllegalArgumentException`이 난다.
스펙상 credentials 요청에는 `*`를 쓸 수 없기 때문이다. 여러 origin이 필요하면 `allowedOriginPatterns`를 쓴다.

> 흔한 실수: 요청의 `Origin` 헤더 값을 그대로 `Access-Control-Allow-Origin`에 되돌려 주는 것.
> `*`보다 더 위험하다. credentials까지 허용되면 **아무 사이트나 로그인된 사용자의 응답을 읽을 수 있다.**
> 허용 목록과 비교해서 맞을 때만 내려야 한다.

### Spring Security와 같이 쓸 때

preflight `OPTIONS` 요청에는 `Authorization` 헤더도 쿠키도 없다. 브라우저가 붙이지 않기 때문이다.
그래서 CORS 처리가 인증보다 뒤에 있으면 **preflight가 401로 떨어지고**, 브라우저에는 CORS 에러로 보인다.

```java
@Bean
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
	http.cors(Customizer.withDefaults())   // CorsConfigurationSource 빈을 사용, 인증 필터보다 앞에서 처리
	    .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
	return http.build();
}

@Bean
CorsConfigurationSource corsConfigurationSource() {
	CorsConfiguration config = new CorsConfiguration();
	config.setAllowedOrigins(List.of("https://www.example.com"));   // 예시 도메인
	config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
	config.setAllowedHeaders(List.of("Content-Type", "Authorization"));
	config.setAllowCredentials(true);
	config.setMaxAge(3600L);   // preflight 결과 캐시. 매 요청마다 OPTIONS가 나가는 걸 줄인다

	UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
	source.registerCorsConfiguration("/**", config);
	return source;
}
```

`WebMvcConfigurer.addCorsMappings()`만 설정하면 MVC 단계에서 처리되므로, 그보다 앞에 있는 Security 필터에서 막힌 응답에는 CORS 헤더가 안 붙는다.
Security를 쓴다면 `http.cors()`로 Security 안에서 처리하는 게 안전하다.

### "CORS 에러"가 진짜 원인이 아닐 때

서버에서 401, 403, 500이 났는데 **그 에러 응답에 CORS 헤더가 없으면**, 브라우저 콘솔에는 CORS 에러만 보인다.
JS는 응답 상태 코드도 읽을 수 없다. CORS 에러가 나면 먼저 **네트워크 탭에서 실제 상태 코드**를 확인한다.
의외로 CORS 설정이 아니라 인증 실패나 서버 에러인 경우가 많다.

## SameSite: 쿠키를 실을지 브라우저가 정한다

SameSite는 쿠키 속성이다. **cross-site 요청에 이 쿠키를 붙일지**를 브라우저가 이 값으로 정한다.

```http
Set-Cookie: SESSION=abc123; Path=/; HttpOnly; Secure; SameSite=Lax
```

| 값 | cross-site 요청에 쿠키를 실을 때 |
|---|---|
| `Strict` | 절대 안 실음. 다른 사이트의 링크를 눌러 들어와도 첫 요청엔 쿠키가 없어서 로그아웃 상태처럼 보인다 |
| `Lax` | 최상위 이동(링크 클릭, 주소창 이동)의 `GET`에만 실음. `iframe`·`fetch`·`img`·form `POST`에는 안 실음 |
| `None` | 항상 실음. 반드시 `Secure`와 같이 써야 한다 (아니면 브라우저가 쿠키를 거부) |

- SameSite를 안 쓰면 Chrome 등은 **`Lax`로 취급**한다. 예전 브라우저는 `None`처럼 동작했다.
  Chrome은 SameSite를 지정하지 않은 쿠키에 한해, 쿠키가 만들어진 뒤 2분 동안은 최상위 `POST`에도 쿠키를 실어 준다(SSO 호환용 예외).
  그래서 **SameSite는 명시적으로 쓰는 게 낫다.**
- `Lax`는 "GET에는 쿠키가 실린다"는 뜻이다. **GET으로 상태를 바꾸는 API**(예: `GET /logout`, `GET /delete?id=1`)가 있으면 `Lax`로는 막히지 않는다.

Spring에서 쿠키를 직접 만들 때와 세션 쿠키 설정은 이렇게 한다.

```java
ResponseCookie cookie = ResponseCookie.from("refresh_token", token)
		.httpOnly(true)
		.secure(true)
		.sameSite("Lax")
		.path("/")
		.maxAge(Duration.ofDays(14))
		.build();
response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
```

```yaml
server:
  servlet:
    session:
      cookie:
        same-site: lax
        secure: true
        http-only: true
```

### SameSite만으로 CSRF가 끝나지 않는 이유

1. **같은 site의 다른 서브도메인은 막지 못한다.** `blog.example.com`에 XSS가 있거나 서브도메인을 탈취당하면, 거기서 `api.example.com`으로 보내는 요청은 same-site라서 쿠키가 실린다.
2. `Lax`는 최상위 `GET`을 허용한다. GET으로 상태를 바꾸는 API가 있으면 뚫린다.
3. 브라우저마다 지원 수준이 다르고, 오래된 브라우저는 SameSite를 무시한다.

그래서 쿠키로 인증하는 서비스라면 SameSite는 **1차 방어선**으로 두고, 상태를 바꾸는 요청에는 CSRF 토큰이나 `Origin` 헤더 검증을 같이 둔다.
반대로 `Authorization: Bearer` 헤더로 인증하면 브라우저가 자동으로 붙여 주는 값이 없으니 CSRF 걱정은 줄어든다(대신 토큰 저장 위치가 XSS 문제로 바뀐다. [localStorage vs sessionStorage](/posts/localstorage-vs-sessionstorage/) 참고).

## 둘을 같이 놓고 보면

| | CORS | SameSite |
|---|---|---|
| 서버가 하는 일 | 응답에 `Access-Control-Allow-*` 헤더를 적는다 | `Set-Cookie`에 속성을 적는다 |
| 브라우저가 하는 일 | 다른 origin의 응답을 **JS가 읽게 할지** 정한다 | cross-site 요청에 **쿠키를 실을지** 정한다 |
| 기준 | origin (scheme + host + port) | site (scheme + eTLD+1) |
| 막는 것 | 다른 사이트가 사용자 권한으로 받은 응답을 **읽는 것** | 다른 사이트가 사용자 쿠키로 요청을 **보내는 것** |
| 막지 못하는 것 | 요청이 서버에 도착하는 것, curl·서버 간 호출 | 같은 site의 서브도메인, Lax의 GET, curl |

"응답을 읽는 것"과 "요청을 보내는 것"은 다른 문제다. CSRF는 응답을 읽을 필요가 없는 공격이라 CORS로는 못 막고, SameSite·CSRF 토큰의 영역이다.

### 프론트와 API를 다른 도메인에 두면

`www.example.com` + `api.example.com`처럼 **같은 site 안에 두면** 쿠키 인증이 단순하다. SameSite=Lax 쿠키가 그대로 실리고, CORS만 맞추면 된다.

반대로 `my-front.vercel.app` + `my-api.example.com`처럼 **site가 다르면** 쿠키가 third-party 쿠키가 된다.
`SameSite=None; Secure`가 필요하고, Safari·Firefox처럼 third-party 쿠키를 기본으로 막는 브라우저에서는 그래도 안 실릴 수 있다.
이럴 때는 프론트 서버가 API를 대신 호출하는 BFF 구조나 같은 도메인 아래 프록시를 두는 쪽을 검토한다. ([BFF 구조와 인증](/posts/bff-architecture-and-auth/) 참고)

## 한 줄 정리

> CORS와 SameSite는 서버가 응답 헤더로 규칙을 알려 주고 **브라우저가 지키는** 규칙이다.
> CORS는 "다른 origin의 응답을 JS가 읽어도 되는가", SameSite는 "cross-site 요청에 쿠키를 실어도 되는가"를 정한다.
> 둘 다 브라우저 밖(curl, 서버 간 호출)에는 아무 효과가 없으니, **API 접근 제어는 인증·인가로**, CSRF는 SameSite + CSRF 토큰/Origin 검증으로 따로 챙긴다.
