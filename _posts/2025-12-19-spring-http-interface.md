---
title: "Spring HttpInterface: 선언형 HTTP 클라이언트가 실제로 해결하는 문제"
date: 2025-12-19
categories: [개발일지, SPRINGBOOT]
tags: [spring-boot, api-design]
---

Spring Boot 3 / Spring 6부터 등장한 **HttpInterface**는 외부 HTTP API 호출 방식을
바꿔놓았다. `RestTemplate`이나 `WebClient` 코드를 서비스 안에 직접 작성하는 대신,
**자바 인터페이스 + 애노테이션 선언만으로 HTTP 클라이언트를 만들 수 있다.**

## 한 줄 요약

> HttpInterface는 외부 API 호출을 자바 인터페이스로 추상화하고, Spring이 이를 실제
> HTTP 호출로 변환해주는 선언형 HTTP 클라이언트다.

Feign과 매우 유사하지만, **Spring Core에서 공식 제공**한다는 점이 가장 큰 차이다.

## 왜 이 방식이 나왔을까

WebClient를 직접 쓰면 세 가지 문제가 반복된다: 코드가 장황해지고, HTTP 호출 코드가
서비스 로직과 섞이고, API 스펙이 코드로 명확히 드러나지 않는다. Spring 팀의 발상은
단순하다 — "컨트롤러는 애노테이션으로 선언형 정의가 가능한데, 왜 HTTP **클라이언트**는
항상 직접 구현해야 하지?" 그 결과가 HttpInterface다.

## 핵심 개념: Controller와 정반대 역할

모양은 Controller와 비슷하지만 역할은 반대다.

| 구분 | Controller | HttpInterface |
|---|---|---|
| 역할 | 요청을 받음 | 요청을 보냄 |
| 대상 | 클라이언트 | 외부 서버 |
| 구현 | 직접 구현 | Spring이 자동 생성 |

즉 **요청을 처리하는 계약이 아니라 요청을 보내는 계약**이다.

```java
@HttpExchange(url = "https://api.example.com")
public interface ExternalUserClient {

    @GetExchange("/users/{id}")
    UserResponse getUser(@PathVariable Long id);
}
```

이 인터페이스에는 구현 클래스도, `@Service`도, 로직도 없다. 그런데 이 메서드를
호출하면 실제 HTTP 요청이 발생한다 — Spring이 **런타임에 프록시 구현체를 자동
생성**하기 때문이다.

## 실제 통신은 누가 하나: HttpInterface는 선언만 담당

실제 통신은 `WebClient`(기본) 또는 `RestClient`(Spring 6.1+)가 수행한다.

```
[HttpInterface] → [Spring Proxy] → [WebClient / RestClient] → [외부 API]
```

이 계층 구조를 이해하면 "HttpInterface가 마법처럼 통신을 대체한다"는 오해가 풀린다 —
실제로는 여전히 WebClient가 통신하고, HttpInterface는 그 위에 선언형 껍데기를
씌운 것뿐이다. 그래서 아래에서 볼 것처럼 타임아웃, 재시도, 커넥션 풀 같은 실제
통신 동작은 여전히 하부의 WebClient/RestClient 설정에 달려 있다.

## 실무 흐름 3단계

**1) API를 인터페이스로 선언** — 스펙만 정의한다.

```java
@HttpExchange(url = "https://api.example.com")
public interface ExternalUserClient {

    @GetExchange("/users/{id}")
    UserResponse getUser(@PathVariable Long id);
}
```

**2) 인터페이스를 실제 클라이언트로 변환**

```java
@Configuration
public class HttpClientConfig {

    @Bean
    ExternalUserClient externalUserClient(WebClient.Builder builder) {
        WebClient webClient = builder.build();

        HttpServiceProxyFactory factory =
                HttpServiceProxyFactory
                        .builder(WebClientAdapter.forClient(webClient))
                        .build();

        return factory.createClient(ExternalUserClient.class);
    }
}
```

`HttpServiceProxyFactory`가 인터페이스를 분석해서 애노테이션 기반으로 HTTP 요청을
구성하고 동적 프록시를 생성한다. 여기서 `webClient` 빌더에 타임아웃, 로깅 필터,
공통 헤더 같은 걸 설정해두면 이 인터페이스를 쓰는 모든 API 호출에 일괄 적용된다 —
즉 인터페이스 자체는 스펙만 담당하고, 실제 통신 정책은 이 Bean 설정 단계에 모인다.

**3) 서비스에서는 그냥 메서드 호출**

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final ExternalUserClient externalUserClient;

    public UserResponse getUser(Long id) {
        return externalUserClient.getUser(id);
    }
}
```

서비스 입장에서는 이 객체가 HTTP 클라이언트인지조차 알 필요가 없다 — **메서드
호출 = HTTP 요청**이라는 구조가 완성된다.

## 이 구조가 실제로 주는 것과 놓치기 쉬운 것

장점은 명확하다: HTTP 호출 코드와 비즈니스 로직이 분리되고, 외부 API 스펙이
인터페이스로 문서화되며, 테스트 시 인터페이스를 Mock으로 대체하기 쉽다.

다만 이 인터페이스 자체는 **에러 처리와 재시도 전략을 아무것도 대신해주지 않는다.**
외부 API가 5xx를 던지거나 타임아웃이 나는 상황을 어떻게 처리할지는 여전히
`WebClient.Builder`에 `ExchangeFilterFunction`으로 재시도/에러 핸들링 로직을
직접 붙이거나, 인터페이스 메서드 자체를 `Mono`/`Flux` 반환형으로 바꿔서 리액티브
연산자로 처리해야 한다. 이 부분이 Feign + Spring Cloud 조합(서킷 브레이커,
로드밸런싱, 재시도가 선언적으로 통합된)과 비교했을 때 HttpInterface가 상대적으로
더 챙겨야 할 것이 많은 지점이다.

## Feign과의 차이

| 항목 | Feign | HttpInterface |
|---|---|---|
| 제공 주체 | Netflix / Spring Cloud | Spring Core |
| 설정 | 간단 | 약간의 설정 필요 |
| 기반 | 동기 중심 | WebClient 기반 |
| 서킷 브레이커/로드밸런싱 | Spring Cloud 생태계와 통합 기본 제공 | 직접 구성 필요 |

**판단 기준**: 이미 Spring Cloud(Eureka, 서킷 브레이커 등)로 MSA 인프라를 갖추고
있고 서비스 디스커버리와 연동된 호출이 필요하다면 Feign이 여전히 자연스럽다.
반면 특정 외부 SaaS API 하나를 호출하는 정도의, MSA 생태계와 무관한 단순 연동이라면
별도 의존성 없이 Spring Core만으로 되는 HttpInterface 쪽이 더 가볍다. "Spring Boot 3
이상이면 무조건 HttpInterface"라고 단정하기보다, 이미 Spring Cloud 스택 위에서
움직이는 프로젝트인지가 실질적인 갈림길이다.

## 결론

HttpInterface는 외부 API 호출을 선언형으로 표준화하고 서비스 로직을 순수하게
유지하게 해준다. 다만 이건 "호출 방식"을 표준화한 것이지 "통신 정책"(재시도, 타임아웃,
서킷 브레이커)까지 대신 해주는 게 아니라는 걸 알고 도입하는 게, 나중에 장애 대응
시나리오를 짤 때 헤매지 않는 길이다.
