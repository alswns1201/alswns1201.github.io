---
title: "Prometheus + Grafana로 Spring Boot 모니터링 구축하기"
date: 2026-03-28
categories: [인프라]
tags: [spring-boot, monitoring, docker, msa]
---

Actuator가 노출하는 지표가 정확히 뭘 말해주는지 눈으로 확인하고 싶어서 시작한 것이
출발점이었다 — 스레드 풀을 고정 크기로 만들어놓고, 실제로 그 개수만큼만 쓰이는지,
문제가 생기면 어떻게 보이는지 시각적으로 보고 싶었다. 그 실험이 나중에는 여러 서비스를
한 대시보드에서 관찰하는 구성으로 자연스럽게 커졌다. 이 글은 그 순서 그대로 정리한다:
**단일 서비스 관찰 → 다중 서비스(MSA) 관찰**.

## 왜 로그만으로는 부족한가

로그는 "무슨 일이 있었는지"는 알려주지만 "지금 상태가 정상 범위인지"는 알려주지 않는다.
스레드 풀이 몇 개 활성화되어 있는지, CPU 사용률이 튀는 시점이 언제인지를 로그를 grep해서
알아내는 건 사실상 불가능하다. Prometheus는 애플리케이션이 노출하는 수치 지표를 주기적으로
수집(scrape)하고, Grafana는 그걸 시계열 그래프로 보여준다 — "지금 이상한가"를 눈으로
바로 판단할 수 있게 해주는 도구다.

## 1) 단일 서비스: 최소 구성으로 먼저 눈으로 확인

### 의존성 추가

```gradle
implementation 'org.springframework.boot:spring-boot-starter-actuator'
implementation 'io.micrometer:micrometer-registry-prometheus'
```

`actuator`는 애플리케이션 내부 상태를 HTTP 엔드포인트로 노출하는 역할이고,
`micrometer-registry-prometheus`는 그 내부 지표를 Prometheus가 읽을 수 있는 포맷으로
변환해주는 역할이다. 이 둘의 역할이 나뉘어 있다는 걸 알아두면, 나중에 "왜 actuator만
켰는데 prometheus 엔드포인트가 안 나오지" 같은 문제를 바로 짚을 수 있다.

### Prometheus가 어디를 긁어갈지 정의

```yaml
global:
  scrape_interval: 15s   # 15초마다 수집

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']
```

`host.docker.internal`이 핵심이다. Prometheus는 Docker 컨테이너 안에서 돌고, 지표를
노출하는 Spring Boot 앱은 호스트에서 돈다 — 컨테이너 입장에서 "호스트 머신"을 가리키는
특수 주소가 이거다. 이걸 `localhost`로 잘못 적으면 컨테이너 자기 자신을 가리키게 되어
연결이 안 된다. 로컬 환경에서 Prometheus 연동이 안 될 때 가장 먼저 의심할 지점이다.

```bash
docker run -d -p 9090:9090 \
  -v /c/project/batch/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

### Grafana에 데이터 소스 연결, 그리고 확인

```bash
docker run -d -p 3000:3000 grafana/grafana
```

`localhost:3000` (기본 계정 admin/admin) → Configuration → Data Sources → Prometheus →
URL에 `http://host.docker.internal:9090` 입력 → Save & Test.

여기까지 되면 `jvm_memory_bytes_used`, `process_cpu_usage`, `jvm_threads_live` 같은
지표를 대시보드에 그래프로 그릴 수 있다. 실제로 `Executors.newFixedThreadPool(4)`로
스레드 풀을 4개로 고정해두고 API를 호출해보면, `jvm_threads_live`가 정확히 그 개수만큼만
움직이는 걸 눈으로 확인할 수 있다 — 문서로 읽은 "고정 크기 풀"이라는 개념이 실제로
지켜지는지 검증하는 가장 직관적인 방법이다.

## 2) 여러 서비스로 확장: MSA 환경에서의 차이

서비스가 하나 늘어나는 순간 구성이 달라진다. 단일 서비스일 때는 target 하나만 등록하면
됐지만, MSA 환경에서는 서비스마다 다른 `job_name`과 `metrics_path`를 등록해야 한다.

```yaml
scrape_configs:
  - job_name: "user-service"
    metrics_path: "/user-service/actuator/prometheus"
    static_configs:
      - targets: ["host.docker.internal:8000"]

  - job_name: "order-service"
    metrics_path: "/order-service/actuator/prometheus"
    static_configs:
      - targets: ["host.docker.internal:8000"]
```

여기서 실수하기 쉬운 지점: 서비스가 게이트웨이 뒤에 있으면 `metrics_path`가
`/actuator/prometheus`가 아니라 게이트웨이 라우팅 규칙에 맞춘 경로
(`/user-service/actuator/prometheus`)가 돼야 한다. 이걸 놓치면 Prometheus 로그에는
"타겟은 정상인데 데이터가 없다"는 애매한 상태로 나타나서 디버깅이 오래 걸린다 —
`curl`로 그 경로를 직접 쳐서 응답이 오는지부터 확인하는 게 빠르다.

### 매번 대시보드를 새로 그리지 않는다

서비스가 늘어나면 매번 패널을 손으로 만드는 대신, Grafana Labs가 공개한 커뮤니티
템플릿을 임포트하는 게 현실적이다. JVM 기반 Spring Boot 모니터링용으로 널리 쓰이는
템플릿(예: ID 4701)을 가져오면 힙 메모리, GC, 스레드, HTTP 요청 레이턴시 같은 지표가
이미 패널로 구성되어 있다 — 바퀴를 다시 만들 이유가 없다. **직접 만드는 패널은 그
서비스 고유의 비즈니스 지표(주문 처리량, 실패율 같은)로 한정**하는 게 유지보수 측면에서
합리적이다.

```
Dashboards → New → Import → grafana.com Dashboard ID 입력 → 데이터 소스로
방금 만든 Prometheus 선택 → Import
```

### docker-compose로 묶어서 관리

컨테이너를 하나씩 `docker run`으로 띄우는 건 서비스가 둘 이상이 되는 순간 번거로워진다.
`docker-compose`로 묶어두면 재현 가능하고, 팀원과 그대로 공유할 수 있다.

```yaml
version: "3"
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    restart: always

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    restart: always
```

한 가지 주의할 점: `docker-compose` 환경에서는 Grafana가 Prometheus를 찾을 때
`localhost:9090`이 아니라 **compose 네트워크 안에서의 서비스 이름**(`http://prometheus:9090`)을
써야 한다. 로컬에서 `docker run`으로 따로 띄웠을 때 썼던 `host.docker.internal` 방식과
헷갈리기 쉬운데, docker-compose는 컨테이너들을 같은 네트워크에 올려주기 때문에 서비스
이름 자체가 곧 호스트명이 된다 — 어떤 방식으로 띄웠는지에 따라 접속 주소를 바꿔야 한다는
걸 기억해두면 된다.

## 정리

- 단일 서비스에서 먼저 "지표가 실제로 코드의 동작과 일치하는가"를 눈으로 검증한 뒤,
  여러 서비스로 확장하는 순서가 안전하다.
- MSA로 갈수록 각 서비스의 `metrics_path`를 정확히 맞추는 게 관건이고, 실패는 대부분
  "타겟은 잡히는데 데이터가 없다"는 형태로 나타나므로 `curl`로 직접 확인하는 습관이 필요하다.
- 대시보드는 커뮤니티 템플릿(JVM/Spring Boot 표준 지표)으로 시작하고, 직접 만드는 건
  비즈니스 지표에만 집중한다.
- `docker run`으로 개별 실행할 때와 `docker-compose`로 묶을 때 Prometheus/Grafana가
  서로를 찾는 주소 체계가 다르다는 걸 인지하고 있어야 한다.
