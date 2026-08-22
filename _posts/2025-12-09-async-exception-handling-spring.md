---
title: "@Async 예외가 조용히 사라지는 이유와 AsyncUncaughtExceptionHandler"
date: 2025-12-09
categories: [Java/Spring]
---

`@Async`로 비동기 작업을 돌리다 보면 한 번쯤 겪는다 — 분명 예외가 터졌는데
`@ControllerAdvice`에도 안 잡히고, try-catch도 못 잡고, 로그에도 안 남는다. 이건 버그가
아니라 정상 동작이다. 이유를 알아야 제대로 된 예외 처리 전략을 세울 수 있다.

## 왜 비동기 예외는 기본 흐름에서 안 잡히는가

`@Async` 메서드는 스프링이 관리하는 별도의 `TaskExecutor` 스레드풀에서 실행된다. 호출한
쪽(컨트롤러/서비스) 스레드와 완전히 분리되어 있기 때문에:

- `@ControllerAdvice`로 예외가 전달되지 않는다 (그건 요청을 처리하던 스레드의 예외만 잡는다)
- 호출부의 try-catch로도 안 잡힌다 (다른 스레드에서 던진 예외라서)
- AOP 로깅 로직에도 안 걸릴 수 있다
- 결과적으로 예외가 **조용히 삼켜진다**

그래서 비동기 메서드를 쓰기로 한 순간, 별도의 예외 처리 경로를 의도적으로 만들어둬야 한다.

## 반환 타입에 따라 처리 방법이 갈린다

- `void` 반환 비동기 메서드의 예외 → `AsyncUncaughtExceptionHandler`
- `CompletableFuture` 반환 비동기 메서드의 예외 → `.exceptionally()`, `.handle()` 등
  Future 체인 안에서 처리

여기서는 실무에서 더 자주 놓치는 `void` 기반을 기준으로 설명한다.

## 1) 예외 처리기 구현

```java
@Slf4j
public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {
    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        log.error("[Async Exception] method={} params={}", method.getName(), params, ex);
    }
}
```

이 핸들러는 `void` 반환 비동기 메서드에서 발생한 미처리 예외를 전부 여기로 모아준다.
로그를 남기든, 알림을 보내든, APM에 커스텀 이벤트로 쏘든 여기가 "모으는 지점"이다.

## 2) 등록

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("async-exec-");
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }
}
```

`getAsyncUncaughtExceptionHandler()`를 등록하지 않으면 핸들러가 없는 것과 같다 —
스프링 기본 구현체는 그냥 로그 한 줄 찍고 끝이라, 알림/모니터링 연계가 안 된다.

## try-catch로 잡으면 핸들러는 무력화된다

```java
@Async
public void runAsyncJob() {
    try {
        // ...
    } catch (Exception e) {
        log.error("failed", e); // 여기서 끝 — AsyncUncaughtExceptionHandler로 안 감
    }
}
```

비동기 메서드 내부에서 예외를 잡아버리면 당연히 전파되지 않고, `AsyncUncaughtExceptionHandler`까지
도달하지 않는다. 실무 기준을 정리하면:

| 상황 | 예외 흐름 |
|---|---|
| try-catch로 내부에서 처리 | 핸들러로 안 감 |
| try-catch 없이 던짐 | `AsyncUncaughtExceptionHandler`가 처리 |

**복구 가능한 예외**(재시도 가능, 기본값으로 대체 가능한 경우)는 내부에서 잡아서 처리하고,
**예상치 못한 예외**는 일부러 던져서 핸들러로 통합 처리하는 쪽이 관리하기 쉽다. 모든 걸
try-catch로 감싸면 에러가 코드 여기저기 흩어져서 오히려 추적이 어려워진다.

## CompletableFuture라고 더 안전한 건 아니다

`void` 대신 `CompletableFuture`를 쓰면 예외 제어가 더 정교해지는 건 맞지만, 함정이 하나
더 있다. `CompletableFuture`는 `.exceptionally()`나 `.handle()`을 명시적으로 걸어두지
않으면, 그리고 결과를 `.get()`으로 꺼내보지도 않으면 **예외가 아예 관찰되지 않고 조용히
사라진다** — `void` 기반은 최소한 `AsyncUncaughtExceptionHandler`라는 보장된 도착지가
있지만, `CompletableFuture`는 체인을 제대로 안 걸면 그 보장조차 없다. "Future 기반이
더 정교하다"는 말이 "더 안전하다"는 뜻은 아니라는 걸 구분해야 한다 — 정교함은 곧 놓치기
쉬운 지점이 늘어난다는 뜻이기도 하다.

## 실무 팁

1. 비동기 로직에서는 **예측 가능한 예외만** try-catch로 잡는다. 남발하면 추적이 어려워진다.
2. 예외 처리기(`AsyncUncaughtExceptionHandler`) 안에서는 **로깅/알림/모니터링만** 한다 —
   여기서 비즈니스 로직을 처리하려 들면 예외가 예외를 낳는 상황이 생긴다.
3. 중요한 비동기 로직은 `void`보다 `CompletableFuture` 기반으로 설계하되, `.exceptionally()`를
   반드시 체인에 건다.

## 정리

비동기 예외는 호출 스레드의 예외 처리 흐름과 완전히 단절되어 있다. `AsyncUncaughtExceptionHandler`로
`void` 기반 예외의 도착지를 명시적으로 만들어주지 않으면, 예외는 로그에도 안 남고 그냥
사라진다. 반대로 `CompletableFuture`를 쓴다고 안심할 게 아니라, 체인을 제대로 걸었는지를
매번 확인해야 한다.
