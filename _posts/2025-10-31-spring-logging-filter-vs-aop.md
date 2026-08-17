---
title: "Spring에서 공통 로그 처리: Filter vs AOP"
date: 2025-10-31
categories: [Spring Boot]
tags: [spring-boot, monitoring, architecture]
---

요청(Request) 단위로 IP, 헤더, URI, 파라미터를 공통으로 로깅해야 한다는 요구사항이
들어왔다고 하자. Spring에서는 **Filter**와 **AOP** 두 가지로 접근할 수 있는데,
어느 쪽을 쓸지는 "로그를 어느 시점에, 무엇을 기준으로 남기고 싶은가"에 달려 있다.

## Filter — 서블릿 요청/응답 사이클의 입구와 출구

```java
@Component
public class RequestLoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        if (request instanceof HttpServletRequest httpRequest) {
            String ip = request.getRemoteAddr();
            String method = httpRequest.getMethod();
            String uri = httpRequest.getRequestURI();
            // 헤더/파라미터를 JSON으로 직렬화해 로깅
        }
        chain.doFilter(request, response);
    }
}
```

**장점**: 모든 요청에 대해 일괄 적용된다 — API가 추가/삭제돼도 Filter 코드는
그대로다. IP, Method, URI, Header에 자연스럽게 접근할 수 있다.

**단점**: Spring Bean 내부의 메서드 단위 로직까지는 볼 수 없다. 요청 Body를
읽으려면 `InputStream`을 감싸는(wrapping) 별도 작업이 필요하다.

## AOP — Bean 메서드 호출 시점에 개입

```java
@Aspect
@Component
public class RequestLoggingAspect {

    @Before("within(@org.springframework.web.bind.annotation.RestController *)")
    public void logBeforeController(JoinPoint joinPoint) {
        // RequestContextHolder로 현재 요청 정보를 가져와 로깅
    }
}
```

**장점**: 특정 서비스 메서드, 트랜잭션, 예외 발생 지점처럼 세밀한 비즈니스 로직
단위로 로깅할 수 있다. 메서드 인자와 리턴값도 함께 기록 가능하다.

**단점**: HTTP 요청 전체를 포괄적으로 잡기 어렵다. `RequestContextHolder`로
요청 정보를 가져와야 하는데, 이건 어디까지나 "현재 스레드에 요청 컨텍스트가
바인딩되어 있다"는 전제가 성립할 때만 동작한다.

## 원문에 없던 함정: AOP만으로는 잡히지 않는 요청들

여기가 중요한데, AOP 포인트컷을 `@RestController` 메서드로 잡으면 **컨트롤러
메서드에 도달하지 못한 요청은 애초에 로그가 안 남는다.** 예를 들어:

- 잘못된 URL로 들어와 404가 나는 요청 (컨트롤러 매핑 자체가 없음)
- 인증 필터에서 401로 즉시 거부되는 요청 (컨트롤러 진입 전에 차단)
- 요청 파싱 단계에서 실패하는 malformed 요청

이런 케이스는 보안 감사나 장애 원인 분석 시 오히려 **가장 궁금한 요청들**인 경우가
많다. AOP만 걸어두면 이 요청들이 로그에 아예 존재하지 않는다는 사각지대가 생긴다.
반면 Filter는 서블릿 컨테이너 진입 지점에 있어서 이런 요청도 잡아낸다 — 이게
"요청 단위 공통 로그라면 Filter가 정석"이라는 결론의 실질적인 근거다.

## 실무 팁: traceId와 MDC

Filter/AOP 어느 쪽을 쓰든, 로그를 남기는 것 자체보다 중요한 건 **여러 로그 줄을
하나의 요청으로 묶어서 추적**할 수 있어야 한다는 점이다. 여기서 쓰는 게 SLF4J의
**MDC(Mapped Diagnostic Context)**다.

```java
@Component
public class TraceIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        String traceId = UUID.randomUUID().toString();
        MDC.put("traceId", traceId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("traceId"); // 스레드 재사용 대비 반드시 정리
        }
    }
}
```

`MDC.put()`으로 넣어둔 값은 이후 같은 스레드에서 실행되는 모든 로그 라인에 자동으로
붙는다 (로그 패턴에 `%X{traceId}` 추가). 이렇게 하면 하나의 요청이 Filter, 서비스
계층, AOP를 거치며 남긴 로그를 traceId 하나로 전부 묶어서 추적할 수 있다.
**`finally`에서 `MDC.remove()`를 빼먹으면**, 스레드 풀 환경에서 다음 요청이 같은
스레드를 재사용할 때 이전 요청의 traceId가 새 요청 로그에 섞여 들어가는 사고가
난다 — 흔히 겪는 실수다.

## 정리

- 요청 단위 공통 로그(모든 요청에 대해 IP/Method/URI 기록)는 **Filter**가 자연스럽다.
  API가 늘어나도 유지보수 부담이 없고, 컨트롤러에 도달하지 못한 요청도 잡는다.
- 비즈니스 로직 단위(특정 서비스 메서드, 성능 측정, 예외 로그)는 **AOP**가 유리하다.
- 민감정보(비밀번호, 토큰)는 로깅 전에 반드시 마스킹한다 — Header/Parameter를
  통째로 JSON 직렬화하면 이런 값이 그대로 로그에 남을 수 있다.
- 실무에서는 두 방식을 함께 쓰는 경우가 많다: Filter로 traceId를 발급해 MDC에
  넣고, AOP는 그 traceId가 이미 있다는 전제로 비즈니스 로직 단위 로그를 남긴다.
