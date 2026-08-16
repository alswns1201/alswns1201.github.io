---
title: "API 호출 발전 과정 그리고, RestAPI(with Spring)"
date: 2025-05-20
categories: [개발일지, 데이터 통신]
tags: [spring-boot, api-design, java]
---

Java/Spring에서 외부 API를 호출하는 방법은 `HttpURLConnection` → `HttpClient` →
`RestTemplate` → `WebClient` 순으로 발전해왔다. 단순히 "최신 게 최고"가 아니라, 각
단계가 이전 방식의 어떤 한계를 풀기 위해 나왔는지를 보면 지금 어떤 걸 골라야 하는지도
같이 보인다.

## HttpURLConnection — 표준 라이브러리, 그리고 그 대가

```java
URL url = new URL("https://www.example.com");
HttpURLConnection con = (HttpURLConnection) url.openConnection();
con.setRequestMethod("GET");

int status = con.getResponseCode();
if (status == HttpURLConnection.HTTP_OK) {
    BufferedReader in = new BufferedReader(new InputStreamReader(con.getInputStream()));
    // ...
}
```

외부 의존성이 필요 없다는 게 유일한 장점이다. 대신 요청 본문 작성, 에러 처리, 커넥션
관리를 전부 수동으로 해야 하고, 비동기 처리가 아예 지원되지 않는다. 지금 이 방식을
새로 쓸 이유는 거의 없다 — 다만 "왜 다음 단계들이 필요했는가"를 이해하는 기준점으로는
유용하다.

## HttpClient — 저수준이지만 유연한 선택지

Apache HttpClient(5.x부터는 `HttpClient` 인터페이스 + `HttpClientBuilder`)는 HTTP
프로토콜 레벨에서 더 세밀한 제어를 제공한다. 5.x부터는 `CompletableFuture`로 비동기
요청도 가능하다.

```java
CompletableFuture<HashMap<String, Object>> future = CompletableFuture.supplyAsync(() -> {
    // 외부 API를 비동기로 처리
    return codef.requestProduct(api, easycodefservicetype, newParam);
}, threadPool);
```

여기서 주의할 점: `HttpClient`는 JSON 직렬화/역직렬화를 직접 제공하지 않는다. Jackson
같은 라이브러리를 별도로 붙여야 한다. 즉 "저수준 제어가 필요한가"가 이 계층을 선택하는
기준이지, 편의성이 기준이 아니다 — 커넥션 풀 세부 튜닝이나 프로토콜 레벨 옵션이 필요
없다면 굳이 이 계층까지 내려갈 이유가 없다.

## RestTemplate — Spring 표준이었지만 태생적 한계가 있다

```java
HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
factory.setConnectTimeout(connTimeout);
factory.setReadTimeout(readTimeout);

RestTemplate restTemplate = new RestTemplate(factory);
ResponseEntity<String> result = restTemplate.exchange(uri.toString(), method, requestEntity, String.class);
```

Jackson을 기본 내장해서 요청/응답을 Java 객체로 자동 변환해준다는 게 `HttpClient`
대비 장점이다. 하지만 결정적인 한계가 있다 — **Spring 3.0부터 5.0까지 동기 방식으로만
동작한다.** 요청을 보내고 응답을 기다리는 동안 스레드가 블로킹된다. 외부 API 호출이
많은 서비스에서는 이게 곧 스레드 풀 고갈로 이어질 수 있다 — 동시 요청이 늘어날수록
블로킹된 스레드가 쌓이고, 결국 새 요청을 처리할 스레드가 없어진다.

## WebClient — 논블로킹이 기본값이 된 이유

```java
Mono<ApiResponseDto> apiResponseDtoMono = webClient.method(method).uri(url).bodyValue(param)
        .retrieve()
        .bodyToMono(ApiResponseDto.class);

apiResponseDtoMono.subscribe(result -> { /* 비동기 콜백 */ });
```

`WebClient`는 Reactive Streams 기반이라 요청을 보내고 스레드를 반환한 뒤, 응답이
오면 콜백으로 처리한다. 스레드가 블로킹되지 않으므로 적은 스레드로 훨씬 많은 동시
요청을 처리할 수 있다. 동기 처리가 필요하면 `block()`을 쓸 수도 있지만, 그럴 거면
애초에 `RestTemplate`을 쓰는 것과 큰 차이가 없어진다 — `WebClient`를 쓰는 의미는
논블로킹을 실제로 활용할 때 생긴다.

## 헷갈리기 쉬운 지점: WebClient와 Flux(WebFlux)의 관계

`WebClient`는 HTTP 클라이언트이고, `Flux`는 Reactive Streams의 Publisher다. 층이
다르다 — `WebClient`는 "요청을 보내는 우체부", `Flux`는 "여러 데이터 흐름을 처리하는
우체국 시스템"에 가깝다. `WebClient`는 기본적으로 단일 요청/응답이고, 여러 요청을
스트림으로 묶어 처리하려면 `Flux`가 그 위에 필요하다. 즉 `WebClient`가 HTTP 통신을
담당하고, `Flux`는 그 결과로 나온 데이터 흐름을 변환·조합하는 역할을 한다 — 이 둘을
같은 층위의 대안으로 오해하면 "WebClient vs WebFlux 뭘 써야 하나" 같은 잘못된 질문을
하게 된다.

| 항목 | CompletableFuture (HttpClient) | WebFlux (WebClient) |
|---|---|---|
| 스레드 관리 | 스레드 풀 기반 | 이벤트 루프 기반 |
| 스레드 수 | 많음 | 적음 |
| 블로킹 여부 | 블로킹 | Non-blocking |
| 자원 효율 | 낮음 | 높음 |
| 확장성 | 제한적 | 높음 |

## 정리: 무엇을 언제

- 외부 라이브러리 없이 최소한의 호출만 필요 → `HttpURLConnection` (사실상 거의 안 씀)
- 프로토콜 레벨 세밀 제어가 필요 → `HttpClient`
- 레거시 프로젝트라 이미 `RestTemplate`을 쓰고 있고, 동시성 요구가 낮음 → 굳이 급하게
  바꿀 필요는 없지만 신규 코드에는 권장하지 않음
- 신규 프로젝트, 특히 외부 API 호출이 잦거나 동시 요청이 많은 서비스 → `WebClient`

Spring 공식 문서도 5.0 이후로는 `RestTemplate` 대신 `WebClient`를 권장한다 — 다만
이건 "새 프로젝트라면"이라는 전제가 붙는다. 이미 `RestTemplate`으로 안정적으로 돌아가는
서비스를 동시성 문제가 실제로 발생하지도 않았는데 마이그레이션하는 건 별개의 판단이다.
