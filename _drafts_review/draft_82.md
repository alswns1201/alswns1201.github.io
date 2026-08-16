---
title: "Spring Boot 메트릭을 Prometheus·Grafana로 모니터링하기"
date: 2026-03-28
categories: [아키텍쳐 설계 관련 글, 그라파나 (Grafana)]
---

*(원문 제목 "Prometheus 와 Grafana"는 내용과 범위가 안 맞아서 구체화했습니다. 참고로 38번
글도 같은 제목·같은 주제("Prometheus & Grafana")로 되어 있어 중복 가능성이 있습니다 —
100/104번처럼 병합이 필요한지 확인 부탁드려요. 일단 82번 내용만 기준으로 재작성했습니다)*

애플리케이션에 장애가 나기 전까지는 로그만 봐도 충분하다고 느끼기 쉽다. 문제는 "지금
CPU가 튀고 있는지", "이 배포 이후로 응답 시간이 늘었는지" 같은 질문에 로그만으로는
답하기 어렵다는 점이다. 시계열 지표(metric)와 로그는 서로 다른 질문에 답하는 도구다.
Prometheus + Grafana는 그중 지표 쪽을 담당하는 가장 표준적인 조합이다.

## 왜 Pull 방식인가

Prometheus는 애플리케이션이 지표를 밀어 넣는(push) 게 아니라, Prometheus 서버가
주기적으로 애플리케이션의 엔드포인트를 긁어가는(pull/scrape) 구조다.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "prometheus, health, info"
  metrics:
    tags:
      application: ${spring.application.name}
```

Pull 방식을 쓰는 이유는 크게 두 가지다. 첫째, **모니터링 대상이 죽었는지 판단하기 쉽다**
— push 방식이면 "데이터가 안 온다"가 "앱이 죽었다"인지 "네트워크가 끊겼다"인지 "앱이
바빠서 못 보냈다"인지 구분이 안 되는데, pull은 Prometheus가 스크랩에 실패한 시점 자체가
곧 장애 신호다. 둘째, **수집 주기를 중앙에서 통제할 수 있다** — 앱마다 push 주기가 다르면
지표 해상도가 들쭉날쭉해지는데, pull은 `scrape_interval` 하나로 전체를 통일한다.

단점도 있다. 배치 작업처럼 실행 시간이 짧아 스크랩 시점에 이미 죽어있는 프로세스는
Prometheus가 관측할 수 없다 — 이런 워크로드는 Pushgateway 같은 별도 장치가 필요하다.

```yaml
scrape_configs:
  - job_name: "user-service"
    metrics_path: "/user-service/actuator/prometheus"
    static_configs:
      - targets: ["host.docker.internal:8000"]
```

**주의할 점 — `/actuator/prometheus`는 기본적으로 인증 없이 열린다.** `include`에
`prometheus`를 추가하는 순간 내부 지표(스레드풀 상태, DB 커넥션 풀, 커스텀 비즈니스
지표까지)가 외부에 그대로 노출될 수 있다. 사내망으로 접근을 제한하거나 Spring
Security로 해당 경로만 별도 인증을 걸어야 한다 — 이걸 놓치는 게 실무에서 꽤 흔한
사고 포인트다.

## 태그 설계가 곧 카디널리티 설계다

`metrics.tags.application`처럼 태그를 붙이는 건 대시보드에서 서비스별로 지표를
구분하기 위해서인데, 태그 값의 종류가 너무 다양하면(예: 사용자 ID를 태그로 넣는 것)
Prometheus 내부에 저장되는 시계열 개수가 폭발적으로 늘어난다 — 이를 **카디널리티
폭발(cardinality explosion)**이라 부른다. 태그는 "종류가 유한하고 적은" 값(서비스명,
HTTP 메서드, 상태 코드 그룹 등)에만 붙여야 한다.

## Grafana 연동: 직접 만들지 말고 가져다 쓴다

```yaml
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

Grafana에서 데이터 소스로 Prometheus를 등록할 때 **`localhost`가 아니라 도커
네트워크상의 서비스명(`http://prometheus:9090`)**을 써야 한다는 게 흔히 걸리는
함정이다 — Grafana 컨테이너 입장에서 `localhost`는 자기 자신이지 Prometheus
컨테이너가 아니기 때문이다. 컨테이너 간 통신은 항상 서비스명 기준이라는 걸
기억해두면 이런 종류의 실수를 줄일 수 있다.

대시보드는 직접 처음부터 설계하기보다 **Grafana Labs Hub**에서 커뮤니티가 만든
템플릿(예: JVM/Spring Boot용 4701번)을 Import ID로 불러오는 게 효율적이다. 표준
지표(JVM 힙, GC, 스레드, HTTP 요청 레이턴시)는 이미 잘 정리된 템플릿이 있고, 굳이
바퀴를 다시 만들 이유가 없다 — 커스텀 비즈니스 지표만 별도 패널로 추가하는 쪽이
현실적이다.
