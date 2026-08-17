---
title: "Spring Cloud Bus + RabbitMQ(AMQP) 개념과 실습"
date: 2026-01-10
categories: [아키텍처]
tags: [msa, spring-boot, kafka, architecture]
---

Config Server를 쓰는 마이크로서비스 환경을 생각해보자. 설정을 바꾸면 Config
Server는 최신 값을 제공하지만, 각 서비스는 `/actuator/refresh`를 **직접 호출해야**
반영된다. 서비스가 3개면 3번, 30개면 30번 — 서비스 수만큼 호출 지점이 늘어난다.
Spring Cloud Bus는 이 문제를 "이벤트를 메시지로 흘려서 모든 서비스에 동기화한다"는
아이디어로 푼다.

## 문제: N번 호출해야 하는 refresh

```
POST /order-service/actuator/refresh
POST /member-service/actuator/refresh
POST /payment-service/actuator/refresh
```

서비스가 늘어날수록 이 목록도 늘어난다. 배포 자동화 스크립트에 서비스 목록을
하드코딩해야 한다는 뜻이고, 새 서비스가 추가될 때마다 이 스크립트도 같이 고쳐야
한다.

## 해법: 브로드캐스트

```
Config Server → (Refresh Event) → RabbitMQ (Exchange)
                                        ├─ order-service
                                        ├─ member-service
                                        └─ payment-service
```

`POST /actuator/busrefresh`를 **딱 한 번만** 호출하면 이벤트가 메시지 브로커를
통해 모든 서비스에 브로드캐스트되고, 각 서비스가 이벤트를 수신해 자동으로
refresh한다. 서비스가 늘어나도 호출 지점은 하나로 고정된다 — 서비스 개수와
운영 복잡도가 분리되는 것이 이 설계의 핵심 이득이다.

## RabbitMQ(AMQP)의 역할

Spring Cloud Bus는 메시지 브로커가 반드시 필요하다. AMQP(Advanced Message
Queuing Protocol)는 메시지 기반 비동기 통신 표준이고, RabbitMQ는 `Producer →
Exchange → Queue → Consumer` 구조로 동작한다.

- **Producer**: Config Server
- **Exchange**: Spring Cloud Bus 전용 Exchange
- **Queue**: 각 마이크로서비스별 Queue
- **Consumer**: 각 서비스 인스턴스

서비스 인스턴스가 뜰 때마다 자기 Queue를 Exchange에 바인딩하므로, 서비스 수가
늘어나도 별도 설정 없이 자동으로 확장된다.

## 설정

```yaml
# Config Server
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your/config-repo
  rabbitmq:
    host: localhost
    port: 5672
management:
  endpoints:
    web:
      exposure:
        include: busrefresh
```

```yaml
# order-service (각 마이크로서비스)
spring:
  config:
    import: "optional:configserver:http://localhost:8888"
  rabbitmq:
    host: localhost
    port: 5672
management:
  endpoints:
    web:
      exposure:
        include: refresh
```

`POST http://localhost:8888/actuator/busrefresh` 한 번이면 RabbitMQ에 이벤트가
발행되고, 모든 서비스가 수신해 `/actuator/refresh`를 자동 실행한다.

## 원문에 없던 것: 이 브로드캐스트가 조용히 실패하는 경우

여기서 짚어야 할 게 있다 — **브로드캐스트가 항상 성공한다는 보장은 없다.**
이벤트가 발행되는 시점에 재배포 중이거나 일시적으로 죽어 있던 서비스 인스턴스는
그 이벤트를 놓친다. RabbitMQ Queue 자체는 인스턴스가 없어도 메시지를 잠깐
버퍼링할 수 있지만, 인스턴스가 완전히 새로 뜨면(재시작) 이전 Queue 바인딩이
사라지고 새 Queue로 다시 바인딩되므로, **재시작 도중 발행된 refresh 이벤트는
유실될 수 있다.**

이게 왜 위험하냐면, "설정이 반영됐는지 확인하는 절차" 없이 이 구조를 신뢰하면
일부 인스턴스만 구설정으로 남아있는 상태를 눈치채기 어렵기 때문이다. 실무에서는
보통 두 가지로 보완한다:

1. **배포 시점에 최신 설정으로 기동**: 애초에 서비스가 시작할 때 Config
   Server에서 최신 설정을 받아오므로(`spring.config.import`), 재시작된
   인스턴스는 refresh 이벤트를 놓쳐도 최신 설정으로 뜬다 — 유실 위험이 있는 건
   "이미 떠 있던" 인스턴스뿐이다.
2. **모니터링/헬스체크로 설정 버전 확인**: `/actuator/env`나 커스텀 엔드포인트로
   각 인스턴스가 실제로 어떤 설정 버전을 쓰고 있는지 주기적으로 확인하는 절차를
   별도로 둔다.

## Kafka를 쓸 수도 있다

RabbitMQ 대신 `spring-cloud-starter-bus-kafka`로 Kafka를 브로커로 쓸 수도 있다.
이미 Kafka를 다른 용도(이벤트 소싱, 로그 스트리밍)로 운영 중이라면 새 브로커를
추가로 띄우지 않고 기존 Kafka 클러스터에 설정 브로드캐스트용 토픽 하나만 추가하는
쪽이 운영 부담이 적다. 반대로 이 목적 하나만을 위해서라면, 관리 콘솔이 직관적이고
가벼운 RabbitMQ 쪽이 초기 구성이 더 간단하다.

## 정리

| 항목 | 설명 |
|---|---|
| Spring Cloud Config | 설정 중앙화 |
| Spring Cloud Bus | 설정 변경 이벤트 브로드캐스트 |
| RabbitMQ(AMQP) | 이벤트 전달 채널 |
| busrefresh | 단 한 번의 호출로 전체 갱신 |

Spring Cloud Bus + Config Server는 배포와는 분리된, "설정 변경을 자동으로
반영하기 위한 운영 자동화 레이어"다. 다만 브로드캐스트가 100% 도달을 보장하는
건 아니라는 전제를 깔고, 재시작 시 최신 설정을 받아오는 경로를 항상 확보해둬야
한다.
