---
title: "Redisson으로 분산 락 다루기: tryLock의 함정"
date: 2025-10-07
categories: [대용량 고민]
---

단일 서버라면 `synchronized`나 `ReentrantLock`으로 동시성 문제를 막을 수 있다. 하지만
서버가 여러 대(여러 JVM)로 늘어나면 이 락은 각 JVM 안에서만 유효해서 무력해진다.
같은 이메일로 동시에 두 대의 서버가 요청을 처리하는 걸 막으려면, **락 자체를 여러
서버가 공유하는 곳(Redis)에 둬야 한다** — 이게 분산 락이 필요한 이유다.

## 비동기 처리와 락을 함께 쓰는 이유

이메일 발송처럼 시간이 걸리는 작업을 동기로 처리하면 두 가지 문제가 생긴다.

1. **사용자 경험 저하**: 응답을 2초씩 기다려야 한다.
2. **서버 자원 낭비**: 그 2초 동안 요청을 처리한 스레드가 블로킹된 채 아무 일도 못
   한다. 동시 사용자가 늘면 이 블로킹이 누적되어 서버 전체 응답이 느려진다.

그래서 `@Async`로 별도 스레드에 위임하는데, 비동기로 넘긴 순간 "같은 이메일로 중복
발송되지 않게 막는" 책임이 새로 생긴다. 여기서 Redisson의 분산 락이 들어온다.

```java
@Async("EmailExecutor")
public CompletableFuture<String> sendEmail(String email) {
    String lockKey = "lock:email:" + email;
    RLock lock = redissonClient.getLock(lockKey);

    try {
        if (lock.tryLock(0, 10, TimeUnit.SECONDS)) {
            try {
                // 실제 이메일 발송 로직
            } finally {
                if (lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        } else {
            // 이미 처리 중 → 스킵
        }
    } catch (Exception e) {
        // ...
    }
    return CompletableFuture.completedFuture(resultMessage);
}
```

## tryLock의 두 숫자가 하는 일

`tryLock(waitTime, leaseTime, unit)`에서 두 값의 역할이 다르다.

- **waitTime (0초)**: 락을 못 잡으면 기다리지 않고 즉시 포기한다. "이미 처리 중이면
  그냥 스킵"이라는 정책을 여기서 표현한다. `lock()`을 썼다면 락이 풀릴 때까지
  무한정 대기했을 것이다 — 이건 큐잉이 필요한 작업에는 맞지만, 중복 요청을 그냥
  버려도 되는 작업(이메일 재전송 방지)에는 과하다.
- **leaseTime (10초)**: 락을 잡은 스레드가 죽거나 응답 없이 멈춰도, 10초가 지나면
  Redis가 자동으로 락을 풀어준다.

## leaseTime이 만드는 진짜 함정

`leaseTime`은 데드락을 막아주지만, 새로운 위험을 만든다: **실제 작업이 leaseTime보다
오래 걸리면, 작업이 끝나기 전에 락이 자동으로 풀려버린다.** 그러면 다른 서버가 같은
락을 잡고 같은 작업을 동시에 시작할 수 있다 — 분산 락을 걸었는데도 중복 실행이
일어나는 것이다. 위 코드에서 이메일 발송이 10초를 넘기면 정확히 이 문제가 발생한다.

Redisson은 이 문제를 **워치독(Watchdog)**으로 완화한다. `leaseTime`을 지정하지 않고
`lock()`을 호출하면, Redisson이 내부적으로 락을 보유한 스레드가 살아있는 동안
주기적으로(기본 10초마다) 락의 TTL을 자동 연장해준다. 즉 작업이 얼마나 걸릴지
불확실하다면, leaseTime을 직접 정하는 것보다 워치독에 맡기는 게 더 안전하다 — 단,
이 경우엔 애플리케이션이 비정상 종료됐을 때 락이 즉시 풀리지 않고 워치독 타임아웃까지
기다려야 한다는 트레이드오프가 생긴다.

## unlock을 반드시 소유자 스레드에서 확인해야 하는 이유

```java
if (lock.isHeldByCurrentThread()) {
    lock.unlock();
}
```

이 체크가 없으면, leaseTime이 지나 락이 이미 자동 해제된 뒤에 뒤늦게 `unlock()`을
호출했을 때 **다른 스레드/서버가 그 사이에 새로 획득한 락을 실수로 풀어버릴 수 있다.**
분산 락에서 "내가 확실히 이 락의 소유자인가"를 확인하지 않고 unlock을 호출하는 건
흔한 실수 지점이다.

## Redisson의 핵심: 단일 Redis 인스턴스를 여러 JVM이 바라본다

```java
RLock lock = redissonClient.getLock("lock:email:user1@example.com");
```

서로 다른 서버(다른 JVM)에서 호출해도 모두 Redis에 있는 동일한 락 키를 참조한다 —
이게 로컬 락과 결정적으로 다른 지점이다. 다만 이 구조는 동시에 **Redis 자체가 단일
장애점(SPOF)이 된다는 뜻**이기도 하다. Redis가 죽거나 네트워크가 끊기면 분산 락
전체가 무력화된다. 정말 엄격한 정합성이 필요한 도메인(결제, 재고 차감처럼 락이
풀렸을 때 손실이 큰 경우)이라면, 단일 Redis 노드보다 Redlock 알고리즘이나 DB 수준의
락(`SELECT ... FOR UPDATE`)과의 트레이드오프까지 검토할 가치가 있다 — Redisson의
기본 락은 "빠르고 대부분의 경우 충분히 안전"한 수준이지, 절대적인 정합성을 보장하는
것은 아니다.

## 정리

- `waitTime`/`leaseTime`은 각각 "얼마나 기다릴지"와 "얼마나 들고 있을지"를 결정한다 —
  둘 다 도메인 정책(재시도 허용 여부, 예상 작업 시간)에 맞춰 골라야 한다.
- leaseTime을 짧게 잡으면 작업이 끝나기 전에 락이 풀려 중복 실행될 수 있다 — 확신이
  없다면 워치독에 맡기는 편이 안전하다.
- unlock 전에 `isHeldByCurrentThread()`를 확인하지 않으면 남의 락을 풀어버릴 수 있다.
- Redisson 락은 Redis를 단일 장애점으로 만든다 — 정합성이 절대적으로 중요한 도메인엔
  추가 검토가 필요하다.
