---
title: "MSA에서 traceId가 끊기는 이유 — Zipkin 분산 트레이싱과 Feign의 함정"
date: 2026-02-22
categories: [개발 고민/설계]
---

로그만 보고 장애를 추적하기 어려운 이유는 단일 요청이 여러 서비스를 거치기
때문이다. `user-service`가 `order-service`를 호출하고, 그게 다시
`payment-service`를 호출하는 흐름에서 각 서비스가 자기 로그만 남기면, 하나의
사용자 요청이 어디서부터 어디까지 걸쳐 있었는지 로그를 이어붙일 방법이 없다.
**분산 트레이싱**은 요청 하나에 고유한 `traceId`를 부여하고, 그 요청이 거쳐가는
모든 서비스가 같은 `traceId`를 로그와 함께 남기게 해서 이 문제를 푼다. Zipkin은
그렇게 수집된 트레이스를 저장하고 시각화해주는 도구다.

```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

## Spring Boot 3부터 트레이싱 스택이 바뀌었다

Spring Boot 2.x 시절에는 `spring-cloud-starter-sleuth` + `spring-cloud-starter-zipkin`
두 개만 추가하면 `Sleuth → Brave → Zipkin` 구조가 자동으로 잡혔다. 그런데 **Spring
Cloud Sleuth는 Spring Cloud 2022.0 이상 릴리즈 트레인에서 제거됐다** — Micrometer
Tracing으로 대체됐기 때문이다. Spring Boot 3.x 기준으로는 이렇게 바뀐다.

```
Micrometer Tracing → Brave → Zipkin
```

`micrometer-tracing-bridge-brave`가 예전 Sleuth가 하던 "브리지" 역할을 대신한다 —
Micrometer Tracing API 호출을 Brave 구현체로 연결해주고, Brave가 실제로 Zipkin에
span을 전송한다.

```xml
<!-- Micrometer Tracing + Brave -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<!-- Zipkin 전송 -->
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

설정 키도 `sleuth.*`에서 `management.tracing.*`로 옮겨갔다.

```yaml
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0   # 예전 sleuth.sampler.probability 대체
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans   # 반드시 /api/v2/spans까지 포함
```

`sampling.probability: 1.0`은 모든 요청을 100% 수집한다는 뜻이다. 트래픽이 많은
운영 환경에서는 이 값을 낮춰서 트레이싱 오버헤드와 저장 비용을 조절한다.

## 실제로 붙여보니 traceId가 서비스마다 달랐다

설정만 맞추면 끝일 줄 알았는데, `user-service`에서 `order-service`를 호출해 주문
정보를 조회해보니 **`order-service`가 남긴 traceId와 `user-service`가 남긴
traceId가 서로 달랐다.** 같은 사용자 요청인데도 Zipkin에서는 두 개의 별개
트레이스로 잡혔다는 뜻이다 — 트레이싱을 붙인 의미가 없어지는 상황이다.

원인은 서비스 간 호출에 쓰던 **OpenFeign이 트레이스 컨텍스트를 전파하지 않는다**는
데 있었다. `management.tracing`과 `zipkin` 익스포터 설정은 "이 서비스가 트레이스를
생성하고 전송한다"까지만 담당하고, "나가는 HTTP 요청에 현재 traceId를 헤더로 실어
보낸다"는 건 별도의 계약이다. Feign은 기본적으로 이 계약을 몰라서, 호출할 때마다
새 트레이스를 시작해버린다.

해결책은 `feign-micrometer` 의존성을 추가하는 것이었다.

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-micrometer</artifactId>
</dependency>
```

Spring Boot 3 + Micrometer Tracing 환경에서 Feign이 트레이스를 전파하려면, 내부적으로
`MicrometerObservationCapability`가 Feign 클라이언트에 등록되어야 한다. 이 모듈이
없으면 Feign 호출 → Zipkin 익스포터 자체는 정상 동작하지만, **Feign이 나가는 요청에
trace 헤더를 실어주지 않아서 호출받는 쪽이 새 traceId를 발급**해버린다. 겉으로는
"Zipkin에 데이터가 들어오니 트레이싱이 되고 있다"고 착각하기 쉬운 상태라, traceId를
직접 비교해보지 않으면 놓치기 쉬운 문제였다.

의존성을 추가한 뒤 다시 확인하니 `user-service`와 `order-service`가 같은 traceId를
공유했고, Zipkin에서 하나의 요청이 두 서비스를 거치는 흐름을 하나의 트레이스로 볼 수
있었다.

## 정리

- Spring Boot 3부터는 Sleuth 대신 `Micrometer Tracing(+Brave) → Zipkin` 구조를 쓴다.
  설정 프리픽스도 `sleuth.*` → `management.tracing.*`로 바뀌었다.
- 트레이싱 익스포터 설정이 됐다고 해서 서비스 간 호출에서 trace 컨텍스트가 자동으로
  전파되는 게 아니다 — 클라이언트 라이브러리(Feign 등)가 그 계약을 별도로 지원해야
  한다.
- OpenFeign을 쓴다면 `feign-micrometer`를 추가해야 나가는 요청에 traceId가 실린다.
  이게 빠지면 Zipkin에 데이터는 쌓이지만 서비스 경계마다 트레이스가 끊긴, 반쪽짜리
  트레이싱이 된다.
- 트레이싱이 제대로 붙었는지 확인하는 가장 확실한 방법은 대시보드가 아니라, 호출을
  주고받는 두 서비스의 로그에서 traceId를 직접 비교해보는 것이다.
