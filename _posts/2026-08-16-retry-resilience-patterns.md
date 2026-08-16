---
title: "재시도와 복원력 패턴: Timeout → Retry → CircuitBreaker → Fallback 조합 순서"
date: 2026-08-16
categories: [개발일지, SPRINGBOOT]
tags: [spring-boot, resilience4j, architecture]
---

분산 환경에서 외부 호출은 언젠가 반드시 실패한다. 네트워크가 순간적으로
끊기기도 하고, 상대 서버가 과부하에 걸리기도 하고, 일시적인 장애가 나기도
한다. 중요한 건 "실패가 일어나느냐"가 아니라 "실패했을 때 어떻게 다루느냐"다.
**복원력(resilience)**이란 시스템 일부가 실패하더라도 전체가 무너지지 않게
하는 능력을 말하고, 여기엔 대표적으로 네 가지 안전장치가 쓰인다.

| 패턴 | 역할 |
|---|---|
| Retry | 일시적 실패를 다시 시도해서 회복 |
| CircuitBreaker | 장애가 번지는 걸 끊어 빠르게 실패시킴 |
| Fallback | 실패했을 때 대체 응답을 제공 |
| Bulkhead | 장애의 영향 범위를 격리 |

`CircuitBreaker` 단독 적용은 [이전 글](/posts/circuit-breaker-resilience4j/)에서
다뤘으니, 이번 글은 나머지 패턴들과 — 특히 이 패턴들을 **어떤 순서로 조합해야
하는지**에 초점을 맞춘다.

## 모든 실패를 재시도하면 안 된다

재시도를 걸기 전에 먼저 실패를 두 종류로 구분해야 한다.

- **일시적 실패(transient failure)**: 네트워크 순단, 타임아웃, 서버가 잠시
  과부하 상태라 뱉는 503 같은 응답. 잠시 후 다시 하면 성공할 가능성이 있으므로
  재시도가 의미 있다.
- **영구적 실패(permanent failure)**: 잘못된 요청(400)이나 인증 실패(401)
  같은 것. 몇 번을 다시 보내도 결과는 똑같다 — 요청 자체가 틀렸기 때문이다.

영구적 실패를 재시도하는 건 시간과 자원을 낭비할 뿐 아니라, 이미 힘든 상대
서버에 불필요한 부하까지 더한다. 그래서 재시도를 걸 때는 **어떤 예외 타입에
걸 것인지**를 반드시 명시해야 한다.

```java
@Retryable(
    retryFor = TransientException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2)
)
public PaymentResult processPayment(PaymentRequest request) {
    return paymentClient.charge(request);
}

@Recover
public PaymentResult recover(TransientException e, PaymentRequest request) {
    return PaymentResult.pending(); // 세 번 다 실패하면 보류 상태로 반환
}
```

`retryFor`에 일시적 예외만 지정해 영구적 실패는 대상에서 제외했고, 세 번 모두
실패하면 `@Recover` 메서드가 호출되어 예외를 그대로 던지는 대신 대안(결제 보류
상태)을 반환한다. 재시도가 실패로 끝났을 때 무조건 예외를 터뜨리지 않고 이렇게
대안을 마련해두는 것이 좋은 설계다.

여기서 잘 알려지지 않은 함정이 하나 있다. 클래스에 `@Recover` 메서드가 하나라도
있으면, `retryFor`에 없는 예외 — 즉 애초에 재시도 대상이 아닌 영구적 실패 —
까지도 전부 recovery 탐색을 거친다. 이때 그 예외 타입을 받는 `@Recover`가 없으면,
원래 예외 대신 `ExhaustedRetryException("Cannot locate recovery method")`가 던져져서
원인이 무엇이었는지 알기 어려워진다. 그래서 영구적 실패용으로도 "받아서 그대로
다시 던지는" `@Recover`를 별도로 하나 둬야 한다.

```java
@Recover
public PaymentResult recoverInvalidPayment(InvalidPaymentException e, PaymentRequest request) {
    throw e; // 복구가 아니라, 원래 예외가 가려지지 않게 그대로 흘려보내는 용도
}
```

## 지수 백오프와 지터

재시도 간격을 어떻게 둘지도 중요하다. 위 코드의 `multiplier = 2`처럼 1초 →
2초 → 4초로 간격을 점점 두 배씩 늘리는 걸 **지수 백오프(exponential
backoff)**라 한다. 호출이 실패했다는 건 보통 상대 서버가 과부하 상태라는
신호이므로, 실패하자마자 바로 다시 찌르면 회복을 방해한다. 간격을 점점 늘려
상대가 숨 돌릴 시간을 주는 것이다.

문제는 지수 백오프만으로는 부족한 경우가 있다는 것. 수천 개의 요청이 동시에
실패했다고 하면, 모두가 똑같이 "1초 후 재시도"를 따를 경우 그 수천 개가
정확히 같은 시점에 서버로 다시 몰려간다. 회복하려던 서버가 이 동시 폭주에
다시 쓰러진다 — 이걸 **재시도 폭풍(retry storm)**이라 부른다. 지수 백오프를
쓰더라도 모두가 같은 간격으로 늘리면 여전히 같은 시점에 몰리는 건 마찬가지다.

해결책은 **지터(jitter)**다. 재시도 시각에 무작위 지연을 살짝 더해서, 누구는
1.1초 후, 누구는 1.4초 후처럼 흩어지게 만든다. 그러면 부하가 시간에 걸쳐
고르게 분산된다. 그래서 실무에서 지수 백오프를 쓸 때는 거의 항상 지터를
함께 적용한다.

```java
@Retryable(
    retryFor = TransientException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2, random = true)
)
```

## 멱등성 없는 작업을 재시도하면 생기는 사고

재시도를 아무 작업에나 걸면 부작용이 생긴다. 예를 들어 결제 요청에 "안전하게
하겠다"며 무조건 재시도를 걸었다고 해보자. 그런데 같은 결제가 세 번 일어나서
고객에게 세 번 청구되는 사고가 날 수 있다.

원인은 이렇다: 첫 번째 결제는 서버에서 이미 성공했는데, 그 **응답**이
네트워크에서 유실됐다. 호출한 쪽에서는 이걸 실패로 보고 재시도했고, 서버는
그걸 완전히 새로운 결제 요청으로 처리했다. 조회처럼 여러 번 실행해도 결과가
같은 작업은 재시도해도 무방하지만, 결제나 주문 생성처럼 부작용이 있는 작업은
재시도가 곧 중복 실행이 된다.

해결책은 두 가지다.

1. **멱등성이 보장된 연산만** 안전하게 재시도한다 (조회 등).
2. 결제처럼 멱등하지 않은 연산에는 **멱등 키(idempotency key)**를 부여해서,
   서버가 같은 키로 들어온 중복 요청을 걸러내도록 설계한다.

## 재시도의 전제 조건: 타임아웃

재시도가 동작하려면 반드시 먼저 갖춰야 할 게 있다. 바로 **타임아웃**이다.
응답을 무한정 기다리고 있으면 실패인지 아닌지 알 수 없어서 재시도할 기회조차
오지 않는다. 더 큰 문제는, 타임아웃이 없으면 느린 호출 하나가 스레드를 계속
점유하고 이게 쌓이면 스레드 풀 고갈로 시스템 전체가 마비된다는 점이다.

타임아웃은 두 종류로 나눠 설정해야 한다.

- **연결 타임아웃(connect timeout)**: 상대 서버와 연결을 맺는 데까지 기다리는
  최대 시간.
- **읽기 타임아웃(read timeout)**: 연결된 후 응답을 기다리는 최대 시간.

둘 다 짧게 설정해야 실패를 빠르게 감지하고 재시도로 넘어갈 수 있다. 그리고
한 가지 더 — 재시도를 여러 번 하는 전체 시간(재시도 예산)이, 이 호출을
호출한 상위 요청의 타임아웃을 넘으면 안 된다. 예를 들어 API 게이트웨이가
5초 안에 응답을 기다린다면, 그 안쪽에서 벌어지는 타임아웃 + 재시도의 합이
5초를 넘지 않도록 예산을 맞춰야 한다.

## 폴백을 설계할 때 지켜야 할 원칙

재시도도, 서킷 브레이커도 실패하면 마지막에 남는 건 **폴백(fallback)**이다.
무조건 에러를 던지는 대신 서비스가 어떻게든 굴러가게 만드는 차선책이다.
대표적인 방식은 세 가지다.

- **기본값 반환**: 개인화 추천 목록을 못 불러오면 누구에게나 보여줄 인기 상품
  목록을 대신 보여준다.
- **캐시된 값**: 직전에 조회해둔 결과를 임시로 제공한다.
- **우아한 축소(graceful degradation)**: 부가 기능만 끄고 핵심 기능은
  유지한다.

폴백은 사용자가 장애를 덜 체감하게 하는 마지막 방어선이지만, 꼭 지켜야 할
원칙이 있다: **폴백이 잘못된 데이터를 주면 안 된다.** 결제 확인에 실패했는데
폴백으로 결제 성공을 위장하는 건 절대 안 된다. 폴백은 항상 안전한 방향으로만
동작해야 한다 — [CircuitBreaker 글](/posts/circuit-breaker-resilience4j/)에서
다룬 "호출자가 정상 응답과 대체 응답을 구분할 수 있어야 한다"는 원칙과 같은
맥락이다.

## 벌크헤드: 장애 범위를 구획으로 나누기

마지막 패턴은 **벌크헤드(bulkhead)**다. 배는 내부가 여러 격벽으로 나뉘어
있어서, 한 구획에 구멍이 나도 다른 구획은 멀쩡해 배 전체가 가라앉지 않는다.
소프트웨어에서도 똑같이 장애의 영향 범위를 구획으로 나누는 패턴이 벌크헤드다.

격리가 없으면 느린 외부 호출 하나가 전체 스레드 풀을 다 점유해서, 그
의존성 하나의 장애가 서비스 전면 장애로 번진다. 벌크헤드를 적용하면 외부
호출마다 전용 스레드 풀이나 동시 호출 수를 따로 제한한다. 그러면 그 호출이
아무리 느려져도 자기에게 할당된 전용 풀만 소진할 뿐, 나머지 기능은 정상
동작한다. 앞서 언급한 스레드 풀 고갈을 막는 핵심 장치이기도 하다.

```java
@Bulkhead(name = "paymentService", type = Bulkhead.Type.THREADPOOL)
@Retryable(retryFor = TransientException.class, maxAttempts = 3)
public PaymentResult processPayment(PaymentRequest request) {
    return paymentClient.charge(request);
}
```

## 조합 순서가 왜 중요한가

이 네 가지 패턴은 따로따로 쓰는 게 아니라 겹겹이 조합해서 쓴다. 그리고 그
**순서**가 중요하다. 일반적인 적용 순서는 다음과 같다.

```
Timeout → Retry → CircuitBreaker → Fallback
(바깥)                              (안쪽)
```

1. 가장 먼저 **타임아웃**으로 빨리 실패해야 다음 단계로 넘어갈 수 있다.
2. **재시도**로 일시적 실패를 흡수한다.
3. 그래도 반복해서 실패하면 **서킷 브레이커**가 차단한다.
4. 최종적으로 실패하면 **폴백**이 대체 응답을 준다.

여기서 꼭 기억해야 할 핵심은, **서킷 브레이커가 재시도를 감싸야 한다**는
점이다. 즉 바깥쪽이 서킷 브레이커, 그 안쪽에 재시도가 들어가야 재시도까지
통째로 묶어서 차단할 수 있다.

만약 순서가 반대로 — 재시도가 서킷 브레이커를 감싸면 — 서킷이 열려서
차단된 상태인데도 재시도 로직이 계속 호출을 시도하게 되어, 서킷 브레이커로
빠르게 실패시키려던 의도 자체가 무의미해진다.

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
@Retryable(retryFor = TransientException.class, maxAttempts = 3, backoff = @Backoff(delay = 1000, multiplier = 2, random = true))
public PaymentResult processPayment(PaymentRequest request) {
    return paymentClient.charge(request);
}

public PaymentResult fallback(PaymentRequest request, Throwable t) {
    return PaymentResult.pending();
}
```

## 정리

- 복원력은 일부가 실패해도 전체가 무너지지 않게 하는 능력이다.
- 재시도는 일시적 실패에만 걸고, 영구적 실패는 대상에서 뺀다.
- 지수 백오프로 재시도 간격을 늘리고, 지터를 더해 재시도 폭풍을 막는다.
- 재시도는 멱등한 연산에만 걸거나, 멱등 키로 중복 실행을 막은 뒤 적용한다.
- 타임아웃 없는 재시도는 위험하다 — 타임아웃이 재시도의 전제 조건이다.
- 폴백은 실패를 감춰서는 안 되고, 항상 안전한 방향으로만 대체 응답을 준다.
- 벌크헤드로 의존성별 자원을 격리해 장애 범위를 한 기능으로 제한한다.
- 조합 순서는 Timeout → Retry → CircuitBreaker → Fallback이며, 서킷
  브레이커가 재시도를 바깥에서 감싸야 차단이 실제로 의미를 가진다.

여기서 다룬 패턴들(Retry, 지수 백오프+지터, 멱등성, CircuitBreaker, TimeLimiter,
Bulkhead, 그리고 이들의 조합 순서)을 Spring Boot + Resilience4j로 직접 구현하고
테스트로 검증해본 코드는
[resilience4j-retry-practice](https://github.com/alswns1201/resilience4j-retry-practice)
레포에 있다.
