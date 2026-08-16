---
title: "Spring Boot + Prometheus + Grafana로 스레드 풀을 눈으로 확인하기"
date: 2025-09-13
categories: [아키텍쳐 설계 관련 글, 그라파나 (Grafana)]
---

*(원문은 설치·설정 단계별 캡처 위주의 실습 메모였는데, "왜 이 조합을 쓰는가"와
"메트릭을 뭘 봐야 하는가"를 추가해서 다시 썼습니다 — 검토 부탁드려요)*

로그만으로는 "지금 스레드 풀이 몇 개나 쓰이고 있는지", "요청이 몰릴 때 큐가 쌓이고
있는지"를 알기 어렵다. 로그는 이벤트 단위 기록이라, 시계열로 추세를 보려면 결국
grep과 스크립트를 짜야 한다. Prometheus + Grafana를 붙이는 이유는 이 관찰을
쿼리 가능한 시계열 데이터로 바꾸기 위해서다.

## 왜 pull 방식인가

Prometheus는 애플리케이션이 메트릭을 밀어넣는(push) 게 아니라, Prometheus가
주기적으로 애플리케이션의 `/actuator/prometheus` 엔드포인트를 긁어간다(pull, scrape).
이 방식의 장점은 애플리케이션이 모니터링 시스템의 존재를 몰라도 된다는 것 — Micrometer가
표준 포맷으로 메트릭을 노출해두기만 하면, Prometheus 쪽 설정(`scrape_interval`,
`targets`)만 바꿔서 수집 대상을 조정할 수 있다. 반대로 애플리케이션이 죽어서 스크레이프가
실패하면 그 자체가 "서비스 다운"을 감지하는 신호가 된다 — push 방식이었다면 애플리케이션이
죽었을 때 아무 신호도 오지 않아 구분이 안 된다.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']
```

Docker 환경에서 컨테이너 안의 Prometheus가 호스트에서 도는 Spring Boot 앱에 접근하려면
`host.docker.internal`을 써야 한다 — 컨테이너 입장에서 `localhost`는 자기 자신이지
호스트가 아니기 때문이다.

## 무엇을 봐야 하는가: 스레드 풀 관찰의 핵심은 "포화 여부"

의존성은 단순하다.

```
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'io.micrometer:micrometer-registry-prometheus'
```

Actuator + Micrometer 조합이 왜 필요한지: Actuator는 애플리케이션 내부 상태(헬스,
메트릭, 빈 정보)를 HTTP로 노출하는 표준 프레임워크고, Micrometer는 그 메트릭을
Prometheus/Datadog/CloudWatch 등 여러 벤더 포맷으로 변환해주는 추상화 계층이다.
즉 Micrometer 덕분에 코드를 바꾸지 않고도 모니터링 백엔드를 교체할 수 있다.

스레드 풀 관련 메트릭 중 실무에서 의미 있는 건 세 가지다.

- `jvm_threads_live` — 현재 실행 중인 스레드 수
- `jvm_threads_peak` — 지금까지 관측된 최대 스레드 수
- `process_cpu_usage` — 전체 프로세스의 CPU 사용률

단순히 "스레드 몇 개 떠 있나"를 보는 것보다 중요한 건, **고정 크기 풀을 설정했을 때
실제로 그 크기를 넘지 않는지, 그리고 큐가 쌓이고 있는지**다. 예를 들어
`Executors.newFixedThreadPool(4)`로 4개 스레드만 쓰도록 제한했다면, 부하가 늘어도
`jvm_threads_live`가 4를 넘지 않아야 정상이다. 만약 예상보다 스레드가 늘어난다면
어딘가 다른 풀(Tomcat 기본 풀 등)에서 요청을 처리하고 있다는 뜻이므로, 어느 풀에서
얼마나 스레드를 쓰는지를 태그(예: `pool` 라벨)로 구분해서 봐야 한다.

## 이 구성으로 확인할 수 있는 것, 확인할 수 없는 것

이 구성(Actuator + Micrometer + Prometheus + Grafana)은 "지금 시스템이 어떤 상태인가"를
보여주는 데는 강력하지만, "왜 그런 상태가 됐는가"까지는 알려주지 않는다. 스레드 수가
갑자기 튀었을 때 그게 어떤 요청 때문인지 추적하려면 분산 트레이싱(Zipkin 등)이나
로그의 요청 ID 상관관계가 별도로 필요하다. 즉 메트릭(지금 몇 개인가)과 트레이싱(왜
이렇게 됐는가)은 서로 다른 질문에 답하는 도구라는 걸 구분해두는 게 좋다.

## 접속

- `http://localhost:9090` — Prometheus (직접 쿼리하며 메트릭 존재 여부 확인용)
- `http://localhost:3000` — Grafana (대시보드로 시각화, 기본 계정 admin/admin)

Grafana에서 데이터 소스로 Prometheus URL(`http://host.docker.internal:9090`)을 등록하고,
`jvm_threads_live` 같은 쿼리로 패널을 만들면 시간에 따른 스레드 사용 추이를 눈으로
확인할 수 있다.
