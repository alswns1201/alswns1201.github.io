---
title: "Spring Security FilterChainProxy: SecurityFilterChain 여러 개는 어떻게 하나의 필터가 되나"
date: 2026-10-31 11:00:00 +0900
categories: [Java/Spring]
---

경로마다 인증 방식이 다르면 `SecurityFilterChain` 빈을 여러 개 만들어 나눈다.
그런데 서블릿 컨테이너 입장에서 보면 체인 개수만큼 필터가 등록된 게 아니다.
**Spring Security는 체인을 전부 모아 `FilterChainProxy`라는 필터 하나에 담고, 요청마다 그중 하나만 골라 실행한다.**
이 구조를 알면 "왜 이 경로에 인증이 안 걸리지?", "왜 이 체인이 안 타지?" 같은 문제를 빨리 찾을 수 있다.

> 체인 이름과 경로는 설명을 위한 예시다.

## 전체 흐름

```
[서블릿 컨테이너(톰캣) 필터 체인]
  CharacterEncodingFilter
  ...
  DelegatingFilterProxy("springSecurityFilterChain")   ← 컨테이너에 등록된 건 이것 하나
        │  위임
        ▼
  FilterChainProxy (스프링 빈)
        │  요청 경로를 보고 체인을 위에서부터 비교, 처음 맞는 하나만 선택
        ├─ publicChain   (/public/**, /actuator/health)
        ├─ adminChain    (/admin/**)
        ├─ apiChain      (/api/**)
        └─ fallbackChain  (나머지 전부)
              │
              ▼ 선택된 체인의 필터들만 순서대로 실행
           SecurityContextHolderFilter → CorsFilter → ... → AuthorizationFilter
              │
              ▼ 끝나면 원래 서블릿 필터 체인으로 복귀
  (다음 서블릿 필터) → DispatcherServlet
```

등장인물은 셋이다.

| 이름 | 정체 | 하는 일 |
|---|---|---|
| `DelegatingFilterProxy` | 서블릿 필터 (컨테이너에 등록) | 스프링 빈인 `FilterChainProxy`를 찾아서 일을 넘긴다 |
| `FilterChainProxy` | 스프링 빈 (`springSecurityFilterChain`) | 등록된 `SecurityFilterChain` 목록에서 요청에 맞는 체인 하나를 골라 실행 |
| `SecurityFilterChain` | 스프링 빈 (우리가 만드는 것) | `RequestMatcher` 하나 + `Filter` 목록 |

## 왜 프록시를 두 번 거치나

### DelegatingFilterProxy: 서블릿 컨테이너와 스프링의 다리

톰캣은 서블릿 스펙의 `Filter`만 안다. 스프링 컨텍스트의 빈은 모른다.
게다가 톰캣이 필터를 등록하는 시점에는 스프링 빈이 아직 다 만들어지지 않았을 수도 있다.

그래서 컨테이너에는 `DelegatingFilterProxy`라는 껍데기만 등록하고, 첫 요청이 올 때 스프링 컨텍스트에서
`springSecurityFilterChain`이라는 이름의 빈을 찾아 위임한다(지연 조회).
Spring Boot에서는 `SecurityFilterAutoConfiguration`이 이 등록을 해 주고, 순서는 기본 `-100`이다.
그래서 대부분의 직접 만든 필터보다 앞, Spring의 요청 래핑 필터들보다는 뒤에 온다.

### FilterChainProxy: 여러 체인 중 하나를 고르는 곳

`SecurityFilterChain` 인터페이스는 단순하다.

```java
public interface SecurityFilterChain {
	boolean matches(HttpServletRequest request);   // 이 체인이 이 요청을 맡는가
	List<Filter> getFilters();                     // 맡는다면 실행할 필터 목록
}
```

`FilterChainProxy`는 이 목록을 들고 있다가 요청이 오면 이렇게 동작한다.

```java
// FilterChainProxy를 단순화한 흐름
private List<Filter> getFilters(HttpServletRequest request) {
	for (SecurityFilterChain chain : this.filterChains) {   // @Order 순서대로
		if (chain.matches(request)) {
			return chain.getFilters();                       // 처음 맞는 체인 하나만
		}
	}
	return null;
}

private void doFilterInternal(ServletRequest req, ServletResponse res, FilterChain chain) {
	HttpServletRequest request = firewall.getFirewalledRequest((HttpServletRequest) req);  // 경로 정규화·검증
	List<Filter> filters = getFilters(request);
	if (filters == null || filters.isEmpty()) {
		chain.doFilter(request, res);        // 맞는 체인이 없으면 보안 필터 없이 그냥 통과
		return;
	}
	new VirtualFilterChain(chain, filters).doFilter(request, res);   // 선택한 필터들 실행 후 원래 체인으로 복귀
}
```

이 코드에서 중요한 점이 세 가지 나온다.

1. **처음 맞는 체인 하나만 실행된다.** 두 체인에 모두 맞아도 뒤쪽 체인은 실행되지 않는다.
2. **맞는 체인이 없으면 보안 필터 없이 통과한다.** 인증도 인가도 안 걸린다.
3. 체인을 고르기 전에 **`HttpFirewall`(기본 `StrictHttpFirewall`)이 경로를 먼저 검사**한다.
   `;`, `//`, 인코딩된 `/`(`%2F`), `../` 같은 값이 들어간 요청을 거절해서, 경로를 비틀어 매처를 우회하는 공격을 막는다.

실제 `FilterChainProxy`는 요청이 끝나면 `SecurityContextHolder`도 비운다.
스레드 풀에서 스레드가 재사용될 때 이전 사용자의 인증 정보가 남지 않게 하는 처리다.

## 여러 체인이 하나로 모이는 과정

`@EnableWebSecurity`가 가져오는 `WebSecurityConfiguration`이 컨텍스트의 `SecurityFilterChain` 빈을 **전부 `List`로 주입받는다.**
이 `List`는 `@Order` 순서로 정렬된다. 그다음 `WebSecurity`가 이 목록으로 `FilterChainProxy`를 만들어
`springSecurityFilterChain`이라는 이름의 빈으로 등록한다.

즉 체인을 몇 개 만들든 결과는 **`FilterChainProxy` 빈 하나**이고, 서블릿 컨테이너에 등록되는 것도 `DelegatingFilterProxy` 하나다.

## 예시: 경로별로 체인을 나누기

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

	@Bean
	@Order(1)
	SecurityFilterChain publicChain(HttpSecurity http) throws Exception {
		http.securityMatcher("/public/**", "/actuator/health")   // 이 체인이 맡을 요청
		    .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
		    .csrf(AbstractHttpConfigurer::disable);
		return http.build();
	}

	@Bean
	@Order(2)
	SecurityFilterChain adminChain(HttpSecurity http) throws Exception {
		http.securityMatcher("/admin/**")
		    .authorizeHttpRequests(auth -> auth.anyRequest().hasRole("ADMIN"))
		    .formLogin(Customizer.withDefaults());               // 관리자는 세션 + 로그인 폼
		return http.build();
	}

	@Bean
	@Order(3)
	SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
		http.securityMatcher("/api/**")
		    .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
		    .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
		    .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()));   // API는 JWT
		return http.build();
	}

	@Bean
	@Order(Ordered.LOWEST_PRECEDENCE)
	SecurityFilterChain fallbackChain(HttpSecurity http) throws Exception {
		// securityMatcher 없음 = 모든 요청. 반드시 마지막에 둔다
		http.authorizeHttpRequests(auth -> auth.anyRequest().denyAll());
		return http.build();
	}
}
```

체인마다 들어가는 필터가 다르다. `publicChain`에는 로그인 관련 필터가 없고, `adminChain`에는 `UsernamePasswordAuthenticationFilter`가,
`apiChain`에는 `BearerTokenAuthenticationFilter`가 들어간다. **경로마다 인증 방식이 다를 때** 체인을 나누는 이유가 이것이다.
한 체인 안에서 `if`로 갈라 처리하는 것보다 각 체인의 설정이 단순해진다.

## 자주 하는 실수

### 1. `securityMatcher`와 `requestMatchers`를 헷갈린다

| 설정 | 의미 |
|---|---|
| `http.securityMatcher("/api/**")` | **이 체인이 이 요청을 맡을지** 정한다 (`FilterChainProxy`가 체인을 고를 때 사용) |
| `authorizeHttpRequests(a -> a.requestMatchers("/api/admin/**").hasRole(...))` | 체인이 선택된 **뒤에**, 그 안에서 어떤 권한이 필요한지 정한다 |

`securityMatcher` 없이 `requestMatchers`만 쓰면 그 체인은 **모든 요청을 맡는 체인**이 된다.

### 2. 모든 요청을 맡는 체인이 앞에 있다

`securityMatcher`가 없는 체인(= any request)이 앞 순서에 있으면 뒤의 체인은 영원히 선택되지 않는다.
Spring Security 6는 이 경우 시작할 때 아래처럼 예외를 던져 알려 준다.

```
A filter chain that matches any request [...] has already been configured,
which means that this filter chain [...] will never get invoked.
```

`@Order`를 빼먹으면 순서가 의도와 다르게 잡힐 수 있으니, 체인이 여러 개면 **모든 체인에 `@Order`를 명시**한다.

### 3. 마지막 체인이 없다

모든 체인에 `securityMatcher`가 있으면, 어디에도 맞지 않는 경로(새로 추가한 컨트롤러 등)는 **보안 필터 없이 통과**한다.
마지막에 `fallbackChain`처럼 나머지를 전부 받는 체인을 두고 `denyAll()`이나 `authenticated()`로 닫아 두면,
새 경로를 추가하고 체인 설정을 잊어도 **기본이 거부**가 된다.

### 4. `permitAll()`과 `web.ignoring()`을 같은 걸로 생각한다

| | `permitAll()` | `WebSecurityCustomizer`의 `ignoring()` |
|---|---|---|
| 보안 필터 실행 | 실행됨 (인가에서 허용만 함) | **필터 목록이 빈 체인**이 되어 아무것도 실행 안 됨 |
| 보안 헤더, CORS, `SecurityContext` | 적용됨 | 적용 안 됨 |

`ignoring()`은 정적 리소스처럼 정말 보안 처리가 필요 없는 경로에만 쓴다. Spring Security도 `permitAll()` 쓰기를 권장하고, `ignoring()`을 쓰면 시작할 때 경고 로그를 남긴다.

### 5. 직접 만든 필터가 엉뚱한 위치에서 실행된다

`OncePerRequestFilter`를 `@Component`로 만들고 `addFilterBefore()`로 체인에도 넣으면,
**서블릿 컨테이너 필터로 한 번, `FilterChainProxy` 안에서 한 번** 두 군데에 등록된다.
특정 체인에서만 쓰려던 필터가 모든 요청에서 실행될 수 있다. ([OncePerRequestFilter 정리](/posts/once-per-request-filter/) 참고)

## 어떤 체인을 탔는지 확인하는 법

```yaml
logging:
  level:
    org.springframework.security: TRACE
```

요청마다 어떤 체인이 선택됐고 어떤 필터가 몇 번째로 실행됐는지 로그로 나온다.

```
Securing GET /api/orders
Invoking SecurityContextHolderFilter (2/12)
...
Invoking AuthorizationFilter (12/12)
Secured GET /api/orders
```

시작 로그(`DEBUG`)에는 체인마다 어떤 필터가 들어 있는지 `Will secure ... with filters: [...]` 형태로 찍힌다.
"이 경로에 인증이 안 걸린다"는 문의가 오면 먼저 이 로그로 **어떤 체인을 탔는지**부터 확인한다.

## 한 줄 정리

> 서블릿 컨테이너에는 `DelegatingFilterProxy` 하나만 등록되고, 이게 스프링 빈 `FilterChainProxy`에 위임한다.
> `FilterChainProxy`는 모든 `SecurityFilterChain`을 `@Order` 순서로 들고 있다가 **처음 맞는 체인 하나만** 실행한다.
> 그래서 체인 순서, `securityMatcher`, 나머지를 받는 마지막 체인이 보안 설정의 핵심이다.
