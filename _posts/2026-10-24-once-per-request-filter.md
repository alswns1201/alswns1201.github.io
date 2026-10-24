---
title: "Spring OncePerRequestFilter 정리: 왜 쓰고, 언제 다시 실행되나"
date: 2026-10-24 12:00:00 +0900
categories: [Java/Spring]
---

요청마다 클라이언트 IP, 추적 ID를 로그에 남기는 필터를 만들면서 `Filter` 대신 `OncePerRequestFilter`를 썼다.
이름 그대로 **"요청 하나에 한 번만"** 실행되는 필터인데, 왜 이게 필요한지와
에러·비동기 요청에서는 어떻게 동작하는지 정리해 둔다.

> 실무에서 쓴 내용을 일반화해서 적었다. 클래스 이름과 MDC 키는 예시다.

## 필터가 한 요청에서 여러 번 실행될 수 있다

서블릿 필터는 **디스패치(dispatch)** 단위로 실행된다.
보통은 클라이언트 요청이 들어올 때 한 번(`REQUEST`)이지만, 같은 요청 안에서 서블릿 컨테이너가 내부적으로 다시 디스패치할 때가 있다.

| DispatcherType | 언제 |
|---|---|
| `REQUEST` | 클라이언트 요청이 처음 들어올 때 |
| `FORWARD` | 서버 안에서 `RequestDispatcher.forward()`로 다른 경로로 넘길 때 |
| `INCLUDE` | 다른 리소스의 결과를 포함할 때 |
| `ERROR` | 예외나 `sendError()`로 **에러 페이지(`/error`)**로 넘어갈 때 |
| `ASYNC` | 비동기 처리(`Callable`, `DeferredResult` 등)가 끝나고 결과를 쓰러 다시 들어올 때 |

필터가 여러 디스패치 타입에 등록되어 있으면 **같은 요청인데 필터가 두세 번 실행**된다.

```
컨트롤러에서 예외 발생
→ REQUEST 디스패치: 필터 실행 → 컨트롤러 → 예외
→ ERROR 디스패치(/error): 필터 또 실행 → BasicErrorController
```

로그를 남기는 필터면 로그가 두 번 찍히고, 인증·카운트를 하는 필터면 **두 번 인증하거나 두 번 센다.**
IP별 요청 횟수를 세는 필터였다면 에러가 난 요청은 2회로 카운트된다.

## OncePerRequestFilter가 하는 일

`OncePerRequestFilter`는 **요청 속성(attribute)에 "이미 실행했음" 표시**를 남겨서 같은 요청에서 다시 실행되지 않게 한다.
핵심 동작을 간단히 줄이면 이렇다.

```java
// OncePerRequestFilter.doFilter() 를 단순화한 흐름
String attr = getFilterName() + ".FILTERED";

if (skipDispatch(request) || shouldNotFilter(request)) {
	chain.doFilter(request, response);          // 이번 디스패치는 건너뜀
} else if (request.getAttribute(attr) != null) {
	chain.doFilter(request, response);          // 이미 실행했으면 건너뜀
} else {
	request.setAttribute(attr, Boolean.TRUE);
	try {
		doFilterInternal(request, response, chain);   // 우리가 구현하는 부분
	} finally {
		request.removeAttribute(attr);
	}
}
```

그래서 직접 만들 때는 `doFilter()`가 아니라 **`doFilterInternal()`**을 구현한다.
파라미터가 처음부터 `HttpServletRequest`, `HttpServletResponse`라서 캐스팅할 필요도 없다.

### 오버라이드할 수 있는 메서드

| 메서드 | 기본값 | 의미 |
|---|---|---|
| `doFilterInternal()` | (필수 구현) | 실제 필터 로직 |
| `shouldNotFilter(request)` | `false` | `true`를 돌려주면 이 요청은 필터를 건너뜀 (경로 제외 등) |
| `shouldNotFilterAsyncDispatch()` | `true` | `ASYNC` 디스패치에서는 실행 안 함 |
| `shouldNotFilterErrorDispatch()` | `true` | `ERROR` 디스패치에서는 실행 안 함 |

> 위 코드에서 표시를 `finally`에서 지우는 부분을 보면, **REQUEST 처리가 끝난 뒤 따로 들어오는 ERROR·ASYNC 디스패치는 표시만으로는 막을 수 없다.**
> 그래서 이 두 경우는 `shouldNotFilterErrorDispatch()`, `shouldNotFilterAsyncDispatch()`로 따로 정한다.
> 기본값이 둘 다 `true`라서, 아무것도 안 바꾸면 **REQUEST 디스패치에서 한 번만** 실행된다.

## Spring Boot에서 등록될 때의 차이

Spring Boot에서 필터를 `@Component`로 등록하면 **어떤 디스패치 타입에 걸리는지가 필터 종류에 따라 다르다.**

- 일반 `Filter` → **`REQUEST`에만** 등록
- `OncePerRequestFilter` → **모든 디스패치 타입**에 등록 (대신 위의 규칙으로 스스로 건너뜀)

즉 Spring Boot 기본 설정에서는 일반 `Filter`도 에러 디스패치에서 다시 실행되지는 않는다.
그래도 `OncePerRequestFilter`를 쓰는 이유는

1. `FilterRegistrationBean`으로 디스패치 타입을 바꾸거나, 외부 톰캣/다른 컨테이너에 배포해도 **"한 번만"이 보장**된다.
2. `shouldNotFilterErrorDispatch()`처럼 **에러·비동기 디스패치에서 실행할지 명시적으로 고를 수 있다.**
3. `shouldNotFilter()`로 제외 경로를 깔끔하게 뺄 수 있고, `HttpServletRequest`를 바로 받는다.
4. Spring Security, Spring Web의 내장 필터(`CharacterEncodingFilter`, `ForwardedHeaderFilter` 등)도 같은 방식이라 **일관성**이 있다.

## 예시: 클라이언트 IP와 추적 ID를 MDC에 넣는 필터

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestLoggingFilter extends OncePerRequestFilter {

	private static final String TRACE_ID = "traceId";
	private static final String CLIENT_IP = "clientIp";

	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
			FilterChain chain) throws ServletException, IOException {
		MDC.put(TRACE_ID, UUID.randomUUID().toString().substring(0, 8));
		MDC.put(CLIENT_IP, request.getRemoteAddr());   // native 설정으로 이미 실제 클라이언트 IP
		try {
			chain.doFilter(request, response);
		} finally {
			MDC.clear();                                // 스레드 재사용 대비, 반드시 정리
		}
	}

	@Override
	protected boolean shouldNotFilter(HttpServletRequest request) {
		return request.getRequestURI().startsWith("/actuator");   // 헬스체크는 제외
	}
}
```

```xml
<!-- logback: 로그 패턴에 MDC 값 출력 -->
<pattern>%d{HH:mm:ss.SSS} [%X{traceId}] [%X{clientIp}] %-5level %logger{36} - %msg%n</pattern>
```

- **`finally`에서 MDC를 비운다.** 톰캣은 스레드 풀을 쓰기 때문에 안 지우면 다음 요청에 이전 요청의 값이 남는다.
- `request.getRemoteAddr()`는 `server.forward-headers-strategy: native` 설정이 되어 있으면 ALB 뒤에서도 실제 클라이언트 IP다.
  톰캣 `RemoteIpValve`가 Spring 필터보다 먼저 실행되기 때문이다. ([X-Forwarded-For와 getRemoteAddr() 차이](/posts/x-forwarded-for-vs-remote-addr/) 참고)
- ALB 헬스체크는 몇 초마다 들어오니 `shouldNotFilter()`로 빼서 로그를 줄였다.

### 에러 로그에 추적 ID가 빠지는 문제

위 필터로 운영해 보면, 컨트롤러에서 예외가 터졌을 때 **`/error` 처리 중에 찍힌 로그에는 traceId가 비어 있다.**

```
REQUEST 디스패치: MDC 설정 → 컨트롤러 예외 → finally에서 MDC.clear()
ERROR 디스패치:   필터 건너뜀(shouldNotFilterErrorDispatch = true) → MDC 비어 있음
```

에러 처리 쪽 로그에도 같은 traceId를 남기고 싶다면 두 가지 방법이 있다.

1. **`@RestControllerAdvice`로 예외를 처리한다.** 예외가 `DispatcherServlet` 안에서 처리되어 ERROR 디스패치 자체가 생기지 않고, REQUEST 디스패치 안(MDC가 살아 있을 때)에서 응답이 나간다. 보통은 이 방법이면 충분하다.
2. ERROR 디스패치에서도 필터를 실행하도록 바꾼다. 이때는 traceId를 **요청 속성에 저장해 두고 재사용**해야 같은 값이 이어진다.

```java
@Override
protected boolean shouldNotFilterErrorDispatch() {
	return false;   // ERROR 디스패치에서도 실행
}

// doFilterInternal 안에서
String traceId = (String) request.getAttribute(TRACE_ID);
if (traceId == null) {
	traceId = UUID.randomUUID().toString().substring(0, 8);
	request.setAttribute(TRACE_ID, traceId);   // ERROR 디스패치에서 같은 값 사용
}
MDC.put(TRACE_ID, traceId);
```

### 비동기 요청에서는

`Callable`, `DeferredResult`, `WebAsyncTask` 같은 비동기 요청에서는

- `doFilterInternal()`은 **비동기 작업이 시작되면 바로 끝난다.** 결과는 나중에 다른 스레드에서 ASYNC 디스패치로 쓰인다.
- 기본값(`shouldNotFilterAsyncDispatch() = true`)이면 ASYNC 디스패치에서는 필터가 다시 실행되지 않는다.
- 그래서 "응답까지 걸린 시간"을 재는 필터라면 비동기 요청에서는 **작업 시작까지의 시간만** 재게 된다.
  응답 기준으로 재야 한다면 `isAsyncStarted(request)`로 확인해서 비동기 리스너를 붙이거나, ASYNC 디스패치에서도 실행되게 바꿔야 한다.

## Spring Security와 같이 쓸 때 주의

`OncePerRequestFilter`를 `@Component`로 만들고 Security 설정에도 `addFilterBefore()`로 넣으면,
**서블릿 컨테이너 필터로 한 번, Security 필터 체인 안에서 한 번** 두 군데에 등록된다.

```java
http.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
// jwtAuthFilter가 @Component이기도 하면 서블릿 필터로도 자동 등록됨
```

같은 빈이라 "이미 실행했음" 표시 덕분에 보통 두 번 실행되지는 않지만,
**먼저 실행되는 쪽이 의도와 다른 위치**(Security 체인 밖)가 될 수 있다. Security 체인에서만 쓰려면 서블릿 등록을 끈다.

```java
@Bean
public FilterRegistrationBean<JwtAuthFilter> jwtAuthFilterRegistration(JwtAuthFilter filter) {
	FilterRegistrationBean<JwtAuthFilter> registration = new FilterRegistrationBean<>(filter);
	registration.setEnabled(false);   // 서블릿 컨테이너에는 등록하지 않음
	return registration;
}
```

또는 처음부터 `@Component`를 붙이지 않고 Security 설정에서 `new`로 만들어 넣는다.

## 한 줄 정리

> 서블릿 필터는 FORWARD·ERROR·ASYNC 같은 내부 디스패치 때 같은 요청에서 다시 실행될 수 있다.
> `OncePerRequestFilter`는 요청 속성에 표시를 남겨 **요청당 한 번만** 실행되게 하고,
> 에러·비동기 디스패치에서 실행할지는 `shouldNotFilterErrorDispatch()`, `shouldNotFilterAsyncDispatch()`로 정한다(기본은 둘 다 실행 안 함).
> 로직은 `doFilterInternal()`에, 제외 경로는 `shouldNotFilter()`에 둔다.
