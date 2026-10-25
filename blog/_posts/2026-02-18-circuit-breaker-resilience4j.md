---
title: "CircuitBreaker로 장애 전파 막기 - Resilience4j"
date: 2026-02-18
categories: [개발 고민/설계]
---

MSA에서 서비스 A가 서비스 B를 호출하는데 B가 죽었다고 해보자. A가 계속 B를 호출하며
응답을 기다리면, A의 스레드/커넥션이 하나씩 그 대기 상태에 묶인다. 시간이 지나면
A마저 요청을 처리할 여력이 없어지고, 결국 B 하나의 장애가 A까지 끌고 내려간다 — 이게
**장애 전파(cascading failure)**다. CircuitBreaker는 이걸 막기 위해 "장애가 발생하는
서비스에 반복적으로 호출이 가지 않도록 차단"하는 장치다.

Netflix Hystrix가 더 이상 유지보수되지 않게 되면서, 지금은 **Resilience4j**가 사실상
표준 선택지다.

## 적용

```xml
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

```java
@FeignClient(name = "order-service", configuration = FeignErrorDecoder.class)
public interface OrderServiceClient {
    @GetMapping("/order-service/{userId}/orders")
    List<ResponseOrder> getOrders(@PathVariable String userId);
}
```

```java
CircuitBreaker circuitBreaker = circuitBreakerFactory.create("circuitbreaker");
List<ResponseOrder> orderList = circuitBreaker.run(
        () -> orderServiceClient.getOrders(userId),
        throwable -> new ArrayList<>()  // fallback
);
```

핵심은 두 번째 인자인 **fallback**이다. `order-service`가 죽어도 `user-service`는
빈 리스트를 반환하며 계속 정상 응답한다 — 완전한 기능은 아니지만, 최소한 죽지는 않는다.
이게 "graceful degradation"이라는 개념이다: 전체가 죽는 것보다 일부 기능이 저하된
채로라도 살아있는 게 낫다는 설계 철학이다.

## 커스텀 설정

```java
@Bean
public Customizer<Resilience4JCircuitBreakerFactory> globalCustomConfiguration() {
    CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
            .failureRateThreshold(4)                      // 실패율 임계치
            .waitDurationInOpenState(Duration.ofMillis(1000)) // OPEN 유지 시간
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(2)
            .build();

    TimeLimiterConfig timeLimiterConfig = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(4))
            .build();

    return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .timeLimiterConfig(timeLimiterConfig)
            .circuitBreakerConfig(circuitBreakerConfig)
            .build());
}
```

## 세 가지 상태를 이해해야 설정값이 이해된다

CircuitBreaker는 이름 그대로 회로 차단기처럼 세 상태를 오간다.

- **CLOSED**: 정상 상태. 요청이 그대로 통과하고, 실패율을 계속 집계한다.
- **OPEN**: 실패율이 `failureRateThreshold`를 넘으면 전환. 이 상태에서는 실제 호출을
  아예 시도하지 않고 즉시 fallback으로 응답한다 — 죽어있는 서비스에 계속 요청을
  던져서 상황을 악화시키지 않기 위해서다.
- **HALF_OPEN**: `waitDurationInOpenState`가 지나면 전환. 제한된 수의 요청을
  "시험적으로" 흘려보내서 서비스가 회복됐는지 확인한다. 성공률이 다시 기준을 넘으면
  CLOSED로, 아니면 다시 OPEN으로 돌아간다.

위 설정에서 `slidingWindowSize(2)`, `failureRateThreshold(4)`처럼 작은 값을 쓰면
빠르게 반응하지만, 반대로 일시적인 네트워크 튐 한두 번에도 회로가 바로 열려버릴 수
있다. 이 값들은 "얼마나 민감하게 반응할 것인가"의 트레이드오프이지, 정답이 정해진
숫자가 아니다 — 트래픽 패턴과 백엔드 서비스의 정상 실패율(타임아웃, 일시적 오류)을
보고 튜닝해야 한다.

## Fallback을 설계할 때 진짜 고민해야 할 것

예시처럼 빈 리스트를 반환하는 게 항상 안전한 fallback은 아니다. "주문 목록이 비어
있음"과 "주문 서비스 장애로 못 가져옴"을 클라이언트가 구분할 수 없다면, 사용자는
실제로는 주문이 있는데 없다고 오해할 수 있다. 진짜 신중한 fallback 설계는 단순히
예외를 안 던지는 데서 끝나지 않고, **호출한 쪽이 이게 "정상적으로 비어있음"인지
"장애로 인한 대체 응답"인지 구분할 수 있는 형태**(예: 별도 상태 필드, 캐시된 이전
데이터 반환)까지 고려해야 한다.

## 정리

- CircuitBreaker의 목적은 장애를 막는 게 아니라 **장애가 전파되지 않게** 하는 것이다.
- CLOSED → OPEN → HALF_OPEN 상태 전이를 이해해야 설정값들이 뭘 조절하는지 감이 잡힌다.
- fallback은 "예외를 안 던지는 것"이 아니라 "호출자가 장애 상황임을 구분할 수 있게"
  설계하는 게 진짜 어려운 부분이다.
