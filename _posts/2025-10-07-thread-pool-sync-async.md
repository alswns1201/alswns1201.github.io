---
title: "동기/비동기, 블로킹/논블로킹 — Spring에서 별도 Thread Pool을 두는 이유"
date: 2025-10-07
categories: [개발일지, 동기 비동기]
tags: [java, spring-boot, redis]
---

"동기/비동기"와 "블로킹/논블로킹"은 자주 같은 뜻처럼 섞여 쓰이지만 서로 다른 축이다.
**동기/비동기**는 "호출한 작업의 결과를 언제 알게 되는가"(즉시 vs 나중에 콜백/Future로)의
문제고, **블로킹/논블로킹**은 "결과를 기다리는 동안 현재 스레드가 멈추는가"의 문제다.
이 둘을 구분해야 왜 Spring에서 별도 스레드 풀을 두고, 왜 `CompletableFuture`를
어떻게 체이닝하느냐에 따라 결과가 완전히 달라지는지 이해가 된다.

## Tomcat 스레드 하나가 왜 그렇게 귀한가

Spring Boot가 기본으로 쓰는 Tomcat은 요청마다 스레드 풀에서 스레드를 하나 꺼내
컨트롤러 메서드를 실행한다. 이 스레드의 목표는 단순하다 — **최대한 빨리 응답하고
풀로 돌아가서 다음 요청을 받을 준비를 하는 것.** 이메일 발송처럼 오래 걸리는 작업을
이 스레드에서 동기적으로 처리하면, 그 스레드는 작업이 끝날 때까지 다른 요청을 받지
못한다. 동시 요청이 몰리면 Tomcat 스레드 풀 전체가 고갈(thread starvation)되고,
전혀 무관한 다른 API까지 응답이 느려진다.

해법은 오래 걸리는 작업을 **별도의 전담 스레드 풀**로 떼어내는 것이다.

```java
@Bean(name = "EmailExecutor")
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);   // 상시 대기 스레드 수
    executor.setMaxPoolSize(10);   // 최대 스레드 수
    executor.setQueueCapacity(25); // 큐 용량
    executor.setThreadNamePrefix("EmailAsync-");
    executor.initialize();
    return executor;
}
```

## 이 설정에서 흔히 놓치는 것: 큐가 가득 차면 무슨 일이 생기는가

`corePoolSize`, `maxPoolSize`만 보고 "5명이 일하다 바쁘면 10명까지 늘어나겠지"라고
생각하기 쉬운데, 정확한 동작 순서는 이렇다: 요청이 들어오면 ①core 스레드가 비어있으면
바로 실행, ②core가 다 차 있으면 큐에 대기, ③**큐(25개)까지 다 차야만** max까지
스레드를 늘린다. 즉 큐가 넉넉하면 스레드는 core 이상으로 잘 늘어나지 않는다 —
`maxPoolSize`는 생각보다 훨씬 늦게 발동하는 안전장치에 가깝다.

그리고 큐도 max 스레드도 다 찼을 때 새 작업이 들어오면 어떻게 될까? `ThreadPoolTaskExecutor`의
기본 거부 정책은 `AbortPolicy` — **예외를 던지고 작업을 그냥 버린다.** 원본 코드처럼
이메일 발송 실패가 사용자에게 중요하지 않다면 괜찮지만, 결제·주문 처리처럼 유실되면
안 되는 작업이라면 거부 정책을 `CallerRunsPolicy`(호출한 스레드가 직접 처리해서
자연스럽게 속도를 늦춤)로 바꾸거나, 아예 큐 앞단에 메시지 큐(Kafka 등)를 두는 설계로
가야 한다. 스레드 풀 설정에서 크기보다 더 자주 사고를 내는 게 이 거부 정책 설정
누락이다.

## CompletableFuture: join()이 왜 다시 블로킹을 만드는가

```java
CompletableFuture<String> future1 = emailService.sendEmail(email.get(0));
CompletableFuture<String> future2 = emailService.sendEmail(email.get(1));
CompletableFuture<String> future3 = emailService.sendEmail(email.get(2));

List<String> results = Stream.of(future1, future2, future3)
        .map(CompletableFuture::join) // 여기서 현재 스레드가 블로킹된다
        .collect(Collectors.toList());
```

`@Async`로 이메일 발송 자체는 별도 스레드에서 논블로킹으로 돌아간다. 그런데
`.join()`을 호출하는 순간, **컨트롤러 요청을 처리하던 Tomcat 스레드가 결과가 나올
때까지 멈춰버린다.** 결국 이메일 발송은 비동기였지만, 그 결과를 기다리는 방식은
동기·블로킹이 되어버려서 처음에 별도 스레드 풀을 둔 의미가 반쯤 사라진다.

```java
@GetMapping("/send-emails-nonblocking")
public CompletableFuture<List<String>> sendEmailsNonBlocking(@RequestParam("emails") List<String> email) {
    CompletableFuture<String> future1 = emailService.sendEmail(email.get(0));
    CompletableFuture<String> future2 = emailService.sendEmail(email.get(1));
    CompletableFuture<String> future3 = emailService.sendEmail(email.get(2));

    return CompletableFuture.allOf(future1, future2, future3)
            .thenApply(v -> Stream.of(future1, future2, future3)
                    .map(CompletableFuture::join) // allOf가 끝난 뒤라 즉시 반환
                    .collect(Collectors.toList()));
}
```

`thenApply`로 바꾸면 Tomcat 스레드는 콜백 등록만 하고 즉시 풀로 돌아간다.
컨트롤러가 `CompletableFuture<List<String>>`를 반환하는 게 핵심인데, Spring MVC가
이 반환 타입을 인식해서 서블릿 비동기 처리(`DeferredResult` 메커니즘)로 자동
전환해주기 때문에 가능한 일이다 — 즉 `@Async`만으로는 부족하고, **컨트롤러의
반환 타입 자체가 비동기를 지원하는 형태여야** 진짜로 스레드가 자유로워진다.

## 분산 락이 왜 `synchronized`가 아니라 Redisson인가

이메일 예시 코드에서 `RLock`(Redisson)으로 락을 거는 이유도 같은 맥락이다.
`synchronized`나 `ReentrantLock`은 **같은 JVM 안에서만** 유효하다. 서버가
한 대뿐이면 문제없지만, 인스턴스를 2대 이상 띄우는 순간 각 인스턴스가 자기 JVM
안에서만 락을 걸기 때문에 서로 다른 인스턴스에서 동시에 같은 이메일을 중복 발송하는
걸 못 막는다. Redisson의 `RLock`은 Redis라는 공유 저장소를 락 상태의 단일 진실
공급원으로 써서, 여러 인스턴스에 걸친 락(분산 락)을 가능하게 한다.

```java
if (lock.tryLock(0, 10, TimeUnit.SECONDS)) { // 0초 대기, 획득 시 10초 후 자동 해제
    ...
}
```

`tryLock(waitTime, leaseTime, unit)`에서 `waitTime=0`은 "락을 못 얻으면 기다리지
않고 바로 포기"라는 뜻이고, `leaseTime=10`은 "락을 쥔 프로세스가 죽어도 10초 뒤엔
자동으로 풀린다"는 안전장치다. `leaseTime`을 안 걸면, 락을 잡은 인스턴스가 크래시
났을 때 그 락이 영원히 안 풀리는 데드락 상태가 될 수 있다 — 분산 락에서 자동 해제
시간을 반드시 설정해야 하는 이유가 여기 있다.

## 정리

- 동기/비동기는 결과를 언제 받는가, 블로킹/논블로킹은 기다리는 동안 스레드가
  멈추는가 — 두 축을 구분해야 `@Async` + `.join()` 조합이 왜 절반만 비동기인지
  이해된다.
- 스레드 풀은 크기보다 **큐가 찼을 때의 거부 정책**을 먼저 정해야 한다.
- 컨트롤러가 `CompletableFuture`를 반환해야 Spring MVC가 진짜로 스레드를 풀어준다.
- 로컬 락(`synchronized`)은 다중 인스턴스 환경에서 무력하다 — 분산 락은
  `leaseTime` 없이 쓰면 데드락 위험을 그대로 옮겨온다.
