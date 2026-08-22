---
title: "Spring Cloud Gateway 기본기: MVC 모드 vs WebFlux 모드"
date: 2026-01-06
categories: [개발 고민/설계]
---

*(원제 "[인프런] Spring Cloud gateway". 원문의 핵심이 "두 가지 구현 방식이 있고
그 차이가 뭔지"였는데 제목엔 안 드러나서 바꿨습니다.)*

MSA에서 API Gateway는 클라이언트가 여러 마이크로서비스로 흩어진 엔드포인트를 하나의
진입점으로 접근할 수 있게 해준다. Spring Cloud Gateway는 이걸 구현하는 두 가지
방식을 제공하는데, 어떤 걸 고르느냐가 나머지 아키텍처 선택에도 영향을 준다.

## 1) MVC 기반 Gateway

```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-gateway-server-webmvc'
```

```yaml
spring:
  cloud:
    gateway:
      mvc:
        routes:
          - id: first-service
            uri: http://localhost:8081/
            predicates:
              - Path=/first-service/**
```

전통적인 서블릿 기반 Spring MVC 위에서 동작한다. 요청 하나당 스레드 하나를 점유하는
동기 모델이다.

## 2) WebFlux 기반 Gateway

```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-gateway-server-webflux'
```

```yaml
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8761/eureka

spring:
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: first-service
              uri: http://localhost:8081/
              predicates:
                - Path=/first-service/**
```

논블로킹 이벤트 루프 기반이다. Gateway는 구조상 클라이언트 요청을 받아 그대로
백엔드 서비스로 넘기고 응답을 다시 클라이언트로 돌려주는 **중계 역할**이 대부분이라,
이 요청-대기-응답 흐름에서 스레드를 계속 붙잡고 있을 필요가 없다. 그래서 실무에서는
WebFlux 기반이 기본 선택지가 되는 경우가 많다 — Gateway는 모든 트래픽이 거쳐가는
단일 지점이라, 여기서 스레드가 병목이 되면 그 뒤에 있는 모든 서비스가 함께 영향을
받는다.

## 왜 이 선택이 백엔드 서비스 선택과는 다른가

일반 백엔드 서비스에서는 MVC(동기)냐 WebFlux(비동기)냐를 팀의 러닝 커브나 기존
코드베이스와의 정합성을 보고 고를 여지가 크다. 하지만 Gateway는 성격이 다르다 —
비즈니스 로직이 거의 없고 순수하게 "받아서 넘기는" 역할이 큰 비중을 차지하기 때문에,
논블로킹의 이점(적은 스레드로 많은 동시 연결 처리)을 온전히 누릴 수 있는 지점이다.
반대로 Gateway 안에서 무거운 동기 로직(예: 외부 인증 서버에 블로킹 호출)을 넣게
되면 WebFlux를 쓰는 의미가 상당 부분 희석된다 — 이벤트 루프 스레드에서 블로킹 호출을
하면 그 스레드에 몰린 다른 모든 요청이 함께 지연된다.

## Filter로 요청/응답 가로채기

```java
@Configuration
public class FilterConfig {

    @Bean
    public RouteLocator getRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route(r -> r.path("/first-service/**")
                        .filters(f -> f.addRequestHeader("f-request", "1st-request-header-by-java")
                                .addResponseHeader("f-response", "1st-response-header-from-java"))
                        .uri("http://localhost:8081"))
                .build();
    }
}
```

Filter는 라우팅되는 모든 요청/응답에 공통 로직을 끼워넣는 지점이다. 인증 토큰 검증,
공통 헤더 주입, 로깅 같은 걸 여기서 처리하면 각 마이크로서비스가 이 로직을 중복
구현하지 않아도 된다. 다만 여기에 로직을 얼마나 태울지는 트레이드오프가 있다 — Gateway
필터에 비즈니스 판단이 섞이기 시작하면, Gateway가 사실상 숨겨진 비즈니스 로직 계층이
되어버려서 각 서비스만 보고는 전체 동작을 이해할 수 없게 된다. Gateway 필터는 "횡단
관심사(cross-cutting concern)"에 한정하는 게 유지보수 관점에서 안전하다.

## 정리

- Gateway는 성격상(중계 역할이 대부분) WebFlux 기반의 논블로킹 이점을 특히 잘 활용할
  수 있는 지점이다.
- 다만 필터 안에 블로킹 호출을 넣으면 그 이점이 사라진다는 점을 항상 염두에 둬야 한다.
- Filter는 인증, 공통 헤더, 로깅처럼 횡단 관심사에 한정하는 것이 좋다 — 비즈니스
  판단을 Gateway로 옮기기 시작하면 서비스별 책임 경계가 흐려진다.
