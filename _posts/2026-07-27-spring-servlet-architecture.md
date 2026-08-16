---
title: "Spring Boot 서블릿 아키텍처: Filter Chain부터 DispatcherServlet까지"
date: 2026-07-27
categories: [개발일지, SPRINGBOOT]
tags: [spring-boot, architecture]
---

Spring Boot에서 인증 실패, 인코딩 깨짐, 특정 요청만 이상하게 처리되는 버그를
쫓다 보면 결국 "이게 Filter에서 처리돼야 하나, Interceptor에서 처리돼야 하나"
라는 질문에 부딪힌다. 이 질문에 답하려면 하나의 HTTP 요청이 실제로 어떤 계층을
거쳐 Controller까지 도달하는지부터 정확히 알아야 한다.

## 전체 요청 처리 흐름

1. 클라이언트가 HTTP 요청을 보낸다.
2. 서블릿 컨테이너에 등록된 Filter Chain이 요청을 순차적으로 통과시킨다.
3. Filter Chain을 통과한 요청은 프론트 컨트롤러인 DispatcherServlet에 도달한다.
4. DispatcherServlet은 HandlerMapping을 통해 요청 URL에 매핑된 핸들러를 찾는다.
5. HandlerAdapter가 해당 핸들러를 실행 가능한 형태로 호출한다.
6. Controller가 실행되어 비즈니스 로직(Service 계층)을 호출한다.
7. 반환된 결과는 ViewResolver 또는 HttpMessageConverter를 거쳐 응답 형태로 변환된다.
8. 최종 응답이 클라이언트에 반환된다.

## Filter Chain의 동작 원리

Filter Chain은 Spring의 ApplicationContext에 속한 개념이 아니라, Tomcat 같은
서블릿 컨테이너 레벨의 인프라다. **DispatcherServlet보다 앞단에서 동작하며,
요청이 서블릿까지 도달할지를 결정한다.**

각 필터는 `doFilter(request, response, chain)` 메서드 안에서 전처리 로직을
실행한 뒤 `chain.doFilter(request, response)`를 호출해 다음 필터로 제어를
넘긴다. 이 호출이 재귀적으로 쌓이기 때문에, 마지막 필터가 서블릿을 호출하고
나면 콜스택을 타고 역순으로 되돌아오면서 각 필터의 후처리 코드가 실행된다.
결과적으로 요청은 바깥에서 안쪽으로, 응답은 안쪽에서 바깥으로 처리되는 구조가
된다.

이 구조에서 알아둘 점:

- Filter는 서블릿 컨테이너 레벨에서 동작하기 때문에 Spring 빈이 아니어도
  되며, `@Controller`가 매핑되지 않은 정적 리소스 요청에도 적용된다. Spring
  Boot에서는 `FilterRegistrationBean`을 통해 필터를 Spring 빈으로 등록해
  관리할 수 있다.
- 실행 순서는 `FilterRegistrationBean`의 `setOrder()`나 `@Order` 어노테이션으로
  지정한다. 숫자가 낮을수록 먼저 실행된다.
- 필터 안에서 `HttpServletRequestWrapper` 또는 `HttpServletResponseWrapper`로
  요청/응답을 감싸서 다음 단계로 전달하면, 이후 단계는 원본 대신 래핑된 객체를
  사용하게 된다. 요청 바디를 여러 번 읽어야 하는 로깅 필터 등에서 이 방식이
  쓰인다.

**Spring Security의 인증도 이 Filter 계층에서 일어난다.** `SecurityFilterChain`이
DispatcherServlet보다 먼저 실행되기 때문에, 인증/인가 실패는 Controller에
진입하기도 전에 걸러진다 — JWT 검증 필터를 만들 때 "왜 이걸 Interceptor가
아니라 Filter로 구현하나"라는 질문의 답이 여기 있다. 인증되지 않은 요청은
아예 DispatcherServlet에 도달하지도 못하게 막는 게 목적이라면, Interceptor는
이미 DispatcherServlet 이후 단계라 그 목적에 맞지 않는다.

## DispatcherServlet 내부 동작

DispatcherServlet은 내부적으로 `doDispatch()` 메서드에서 다음 순서로 요청을
처리한다.

- `getHandler()`: 등록된 HandlerMapping 구현체(예: `RequestMappingHandlerMapping`)를
  순회하며 요청 URL에 매핑되는 핸들러와 인터셉터 목록을 묶은
  `HandlerExecutionChain`을 반환한다.
- `getHandlerAdapter()`: 핸들러 타입에 맞는 HandlerAdapter를 찾는다. 어노테이션
  기반 컨트롤러의 경우 대부분 `RequestMappingHandlerAdapter`가 사용된다.
- `preHandle()`: HandlerExecutionChain에 등록된 인터셉터들의 사전 처리 로직이
  실행된다. `preHandle()`이 false를 반환하면 이후 체인 전체가 중단된다.
- `adapter.handle()`: 실제 컨트롤러 메서드를 리플렉션으로 호출하고, 반환값을
  ModelAndView로 변환한다. `@ResponseBody`인 경우 이 과정에서
  `HttpMessageConverter`가 바로 응답 본문을 작성하므로 View 처리 단계는
  생략된다.
- `postHandle()`: 컨트롤러 실행 후 인터셉터의 사후 처리 로직이 실행된다.
- `processDispatchResult()`: 예외가 없으면 ViewResolver로 View를 찾아
  렌더링한다. 예외가 발생했다면 이 시점에 등록된 `HandlerExceptionResolver`
  체인이 먼저 예외를 처리해 ModelAndView로 변환한다.
- `afterCompletion()`: 인터셉터의 최종 콜백이 실행된다.

## Filter vs Interceptor: 뭘 어디에 둘 것인가

두 계층의 위치 차이가 그대로 책임 분리 기준이 된다.

- **Filter**: 서블릿 컨테이너 레벨. Spring 컨텍스트 정보(어떤 컨트롤러가
  호출될지, `@Auth` 같은 커스텀 어노테이션 등)에 접근할 수 없다. 인증,
  인코딩 설정, CORS처럼 **컨트롤러가 뭔지와 무관하게 모든 요청에 동일하게
  적용돼야 하는 관심사**에 적합하다.
- **Interceptor**: DispatcherServlet 이후, Spring MVC 컨텍스트 안에서 동작하므로
  `HandlerMethod`(어떤 컨트롤러 메서드가 호출될지)에 접근할 수 있다. 특정
  어노테이션이 붙은 메서드에만 로직을 적용하고 싶을 때, 또는 Spring 빈을
  자유롭게 주입받아야 하는 로직에 적합하다.

인증처럼 "컨트롤러에 도달하기 전에 아예 막아야 하는" 것은 Filter, 로깅처럼
"어떤 컨트롤러가 호출됐는지 알아야 의미 있는"것은 Interceptor나 AOP로 가는
식으로 나누면 이후 유지보수에서 "이 로직이 왜 여기 있지"라는 질문이 줄어든다.

## WebApplicationContext 구조

Spring MVC에서는 전통적으로 Root WebApplicationContext와 Servlet
WebApplicationContext를 구분해왔다.

- Root WebApplicationContext는 `ContextLoaderListener`가 로드하며, Service나
  Repository처럼 웹 계층과 무관한 공통 빈을 담는다.
- Servlet WebApplicationContext는 DispatcherServlet이 로드하며, Controller,
  HandlerMapping, ViewResolver, Interceptor 등 웹 계층 빈만 담는다.
- 두 컨텍스트는 부모-자식 관계로, Servlet WebApplicationContext가 Root
  WebApplicationContext를 부모로 참조한다.

이 구조는 XML 기반의 전통적인 Spring MVC 설정에서 두드러지는 개념이다. Spring
Boot는 기본적으로 단일 ApplicationContext를 사용한다. `@SpringBootApplication`이
Controller, Service, Repository, HandlerMapping 등을 모두 하나의 컨텍스트에
등록하기 때문에, 별도로 멀티 DispatcherServlet 구성을 하지 않는 이상 Root와
Servlet 컨텍스트의 분리는 실질적으로 존재하지 않는다.

## 정리

Filter는 서블릿 컨테이너 레벨에서 요청이 DispatcherServlet까지 도달할지를
결정하는 관문이고, DispatcherServlet은 ApplicationContext 안에서
HandlerMapping과 HandlerAdapter를 통해 요청을 Controller로 연결하는 프론트
컨트롤러 역할을 한다. 이 두 계층의 책임 범위를 구분해서 이해하면, 인증/인코딩처럼
컨트롤러 진입 전에 처리해야 하는 공통 관심사는 Filter로, 특정 핸들러 실행
전후에 필요한 로직은 HandlerInterceptor로 나누어 설계하는 기준이 명확해진다.
