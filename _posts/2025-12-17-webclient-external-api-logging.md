---
title: "Spring WebClient로 외부 API 로그 분리하기"
date: 2025-12-17
categories: [개발일지, 로깅]
tags: [spring-boot, logging, webclient]
---

외부 API 연동이 들어간 순간부터 로그 전략은 바뀌어야 한다. Controller 필터에서
찍히는 로그(요청 URL, 처리 시간, 응답 코드)만으로는 **외부 API 장애, 지연, 재시도
상황을 절대 파악할 수 없다.** Spring WebClient를 쓴다면 외부 API 로그는
`ExchangeFilterFunction` 레벨에서 관리하는 게 사실상 표준이다.

## 왜 Controller 로그만으로는 부족한가

Controller 로그는 "우리 API가 호출됐다"는 사실만 알려준다. 외부 API 연동이
들어오면 다음 정보가 추가로 필요해진다: 외부 API 호출 여부, 실제 외부 요청 URL,
응답 상태 코드, 타임아웃 발생 여부, 재시도 횟수, 실패 원인(네트워크/5xx/timeout).
이건 Controller 필터에서는 절대 알 수 없는 정보다 — 외부 호출 계층(WebClient)
안에서만 관측 가능하기 때문이다.

## `ExchangeFilterFunction`이란?

WebClient 요청/응답 흐름에 끼어드는 Interceptor다. 요청 전, 응답 후, 예외 발생 시
모두 가로챌 수 있다. 개념적으로는 `RestTemplate`의 `ClientHttpRequestInterceptor`와
같은 역할이다.

```java
public interface ExchangeFilterFunction {
    Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next);
}
```

## 적용 구조

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
                .filter(logRequest())
                .filter(logResponse())
                .build();
    }
}
```

이렇게 하면 이 WebClient를 사용하는 모든 외부 API 호출에 로그가 자동 적용된다.

```java
private ExchangeFilterFunction logRequest() {
    return ExchangeFilterFunction.ofRequestProcessor(request -> {
        log.info("[External-Request] method={} url={}", request.method(), request.url());
        return Mono.just(request);
    });
}
```

요청 바디는 찍지 않는다 — 개인정보, 카드 정보, 토큰 유출 위험 때문이다.

```java
private ExchangeFilterFunction logResponse() {
    return ExchangeFilterFunction.ofResponseProcessor(response -> {
        log.info("[External-Response] status={}", response.statusCode());
        return Mono.just(response);
    });
}
```

## 재시도 로그는 반드시 별도로

외부 API에서 가장 중요한 운영 지표는 재시도 횟수다.

```java
.retryWhen(
    Retry.fixedDelay(3, Duration.ofSeconds(1))
        .doBeforeRetry(signal ->
            log.warn("[External-Retry] retryCount={} reason={}",
                signal.totalRetries() + 1,
                signal.failure().getClass().getSimpleName())
        )
)
```

이 로그가 없으면 외부 API가 느린 건지, 내부에서 재시도를 반복하는 건지 운영에서
전혀 구분할 수 없다.

## traceId 연동 — 그리고 WebFlux에서 이게 생각보다 까다로운 이유

Controller에서 생성한 traceId를 외부 API 로그에서도 같이 찍어야 하나의 요청 흐름을
추적할 수 있다. 보통은 MDC를 쓴다.

```java
log.info("[External-Request] traceId={} url={}", MDC.get("traceId"), request.url());
```

여기서 실무에서 자주 걸리는 함정이 있다. **MDC는 `ThreadLocal` 기반이고, WebClient는
리액티브(WebFlux) 스택 위에서 동작한다.** 리액티브 파이프라인은 하나의 요청을
처리하는 도중에도 스레드가 여러 번 바뀐다(스레드 호핑) — Controller에서 요청을 받은
스레드와, WebClient가 실제로 필터를 실행하는 스레드가 다를 수 있다는 뜻이다. `ThreadLocal`은
스레드에 묶여 있으므로, 스레드가 바뀌는 순간 `MDC.get("traceId")`는 값을 잃고 `null`을
반환할 수 있다.

동기(서블릿) 스택에서는 요청 하나가 스레드 하나에 고정되므로 이 문제가 안 보이지만,
WebFlux 환경에서 MDC를 그대로 쓰면 traceId가 로그에서 간헐적으로 빠지는 버그로
나타난다. 이 문제를 제대로 풀려면 `ThreadLocal` 대신 Reactor의 `Context`(리액티브
체인을 타고 전파되는 값 저장소)에 traceId를 담고, Micrometer Context Propagation
같은 라이브러리로 MDC와 Reactor Context를 동기화해줘야 한다 — 리액티브 스택에서
로그 추적을 도입할 때 가장 먼저 검증해봐야 할 지점이다.

## 로그 레벨 전략

| 상황 | 로그 레벨 |
|---|---|
| 외부 요청 시작 | INFO |
| 정상 응답 | INFO |
| 재시도 발생 | WARN |
| timeout / 실패 | ERROR |

이 기준으로 안 나누면 로그가 너무 많아지거나, 장애 로그가 그 안에 묻힌다.

## 절대 찍지 말아야 할 것

요청/응답 바디 전체, `Authorization` 헤더, 카드번호·주민번호·토큰. 외부 API 로그는
**"상태 확인용"**이지 **"데이터 확인용"**이 아니다.

## 정리

- Controller 로그와 외부 API 로그는 목적이 다르다 — 외부 API 로그는 WebClient
  레벨에서 관리한다.
- `ExchangeFilterFunction`이 표준적인 해결책이지만, retry/timeout 로그는 반드시
  별도로 남긴다.
- traceId 연동 시, WebFlux 환경에서는 `ThreadLocal` 기반 MDC가 스레드 호핑 때문에
  깨질 수 있다는 걸 먼저 검증해야 한다.

외부 API 로그를 분리하지 않으면 장애가 났을 때 "감"으로 디버깅하게 된다. 분리해두면
로그만 보고 원인을 바로 알 수 있다.
