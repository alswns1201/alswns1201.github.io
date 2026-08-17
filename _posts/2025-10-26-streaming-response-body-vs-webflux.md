---
title: "StreamingResponseBody vs WebFlux Flux: 서버 스트리밍 두 가지 방식"
date: 2025-10-26
categories: [Spring Boot]
tags: [spring-boot, reactive, architecture]
---

주식 시세, 로그 스트리밍, 실시간 알림처럼 서버에서 클라이언트로 데이터를 계속 흘려보내야
할 때, Spring에는 철학이 다른 두 가지 도구가 있다. 어느 쪽을 쓸지는 "동시 연결 수"와
"팀이 감당할 수 있는 코드 복잡도"의 문제다.

## StreamingResponseBody — 내가 직접 민다

```java
@GetMapping(value = "/live")
public StreamingResponseBody streamStockPrices(HttpServletResponse response) {
    response.setContentType("text/event-stream");
    return outputStream -> {
        for (int i = 0; i < 20; i++) {
            double price = 100 + new Random().nextDouble() * 10;
            outputStream.write(("Stock Price: " + price + "\n").getBytes());
            outputStream.flush();
            Thread.sleep(1000);
        }
    };
}
```

서블릿 기반(Spring MVC)의 방식이다. **한 요청당 스레드 하나가 붙어서, 그 스레드가
`OutputStream`에 직접 쓰고 flush 타이밍까지 관리한다.** 명령형 — 내가 언제 무엇을 보낼지
직접 코드로 통제한다.

## WebFlux Flux — 흐름을 정의하면 시스템이 민다

```java
@GetMapping(value = "/live", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamStockPrice() {
    return Flux.interval(Duration.ofSeconds(1))
            .take(20)
            .map(i -> "Stock Price: " + (100 + new Random().nextDouble() * 10));
}
```

`Flux`는 데이터를 미리 만들어두지 않고 **흐름 자체를 선언**한다. Netty 기반 논블로킹
이벤트 루프로 동작하고, 구독자가 있을 때만 실제로 데이터가 흐른다.

## 진짜 차이는 스레드 모델이 아니라 백프레셔다

표면적으로는 "스레드 1개 vs 이벤트 루프"가 차이처럼 보이지만, 실무에 영향을 주는 진짜
차이는 **클라이언트가 느릴 때 무슨 일이 일어나는가**다.

- `StreamingResponseBody`는 클라이언트가 데이터를 천천히 읽으면 `outputStream.write()`가
  블로킹되고, 그동안 그 요청을 처리하던 서블릿 스레드는 계속 점유된 채로 묶인다. 동시
  연결이 늘어나면 스레드풀이 그만큼 소모된다 — 스레드풀 크기가 곧 동시 스트리밍 가능
  연결 수의 상한이 된다.
- `Flux`는 Reactive Streams 스펙의 **백프레셔(backpressure)**를 지원한다. 구독자가 처리
  속도를 조절(`request(n)`)할 수 있고, 발행자는 그 요청량만큼만 만들어 보낸다. 스레드를
  점유하지 않고 이벤트 루프가 여러 스트림을 동시에 다중화한다.

즉 "동시에 몇 개의 느린 클라이언트를 감당해야 하는가"가 선택 기준의 핵심이다. 동시
연결이 수십 개 수준이면 스레드 점유 비용은 무시할 만하지만, 수천 개 이상으로 늘어나면
서블릿 방식은 스레드풀 고갈로 막힌다.

## 그럼에도 WebFlux를 쓰지 않는 게 나은 경우

Reactive가 항상 우월한 건 아니다.

- **기존 코드베이스가 이미 MVC(블로킹) 기반**이라면, 리액티브로 전환하는 비용(JPA →
  R2DBC, 블로킹 라이브러리 대체, 팀의 러닝커브)이 얻는 이득보다 클 수 있다.
- **디버깅 난이도**: 블로킹 코드는 스택 트레이스가 호출 흐름을 그대로 보여주지만, 리액티브
  체인은 콜백이 여러 스레드/이벤트 루프를 오가며 실행되기 때문에 스택 트레이스만 보고
  문제를 추적하기가 훨씬 어렵다. `Hooks.onOperatorDebug()` 같은 도구 없이는 원인 파악이
  오래 걸린다.
- **동시 스트리밍 연결 수가 애초에 적다**면(내부 관리자 대시보드 수준) 스레드풀 고갈
  걱정 자체가 없으므로, 굳이 복잡도를 감수할 이유가 없다.

## 정리

| 구분 | StreamingResponseBody | WebFlux (Flux) |
|---|---|---|
| 실행 모델 | 서블릿 (Blocking I/O) | 논블로킹 (Reactive) |
| 스레드 관리 | 요청당 스레드 1개, 느린 클라이언트가 스레드 점유 | 이벤트 루프, 백프레셔로 흐름 제어 |
| 코드 스타일 | 명령형 | 선언형 |
| 확장 한계 | 스레드풀 크기 | 이벤트 루프 처리량 |
| 적합한 상황 | 동시 연결 적음, 기존 MVC 코드베이스 | 대규모 동시 스트리밍, SSE/WebSocket |

두 방식 모두 "서버에서 클라이언트로 실시간 전송"이라는 목적은 같지만, 선택 기준은
결국 **예상 동시 연결 규모와 팀이 감당할 복잡도**다.
