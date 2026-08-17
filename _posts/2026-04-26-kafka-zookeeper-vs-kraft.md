---
title: "Kafka는 왜 Zookeeper를 버렸는가 — Zookeeper 모드 vs KRaft 모드"
date: 2026-04-26
categories: [Kafka]
tags: [kafka, msa, architecture]
---

Kafka를 처음 띄워보면 이상한 점이 하나 있다 — Kafka 혼자 뜨는 게 아니라 Zookeeper라는
낯선 프로그램을 먼저 띄워야 한다. 최신 버전에서는 이 과정이 사라졌다. 무엇을 하던
프로그램이었고, 왜 없어질 수 있었는지를 브로커의 역할부터 순서대로 짚어본다.

## 브로커가 하는 일부터: Zookeeper를 이해하려면 먼저 필요하다

브로커는 단순히 메시지가 지나가는 통로가 아니라, 데이터를 실제로 들고 있는 저장소다.

- **메시지 저장**: 프로듀서가 보낸 메시지를 로컬 디스크에 파일로 저장한다.
- **메시지 전달**: 컨슈머 요청이 오면 저장된 메시지를 꺼내 보낸다.
- **분산 저장**: 하나의 토픽을 여러 **파티션**으로 쪼개서, 여러 브로커에 나눠 담는다 —
  이게 Kafka가 병렬 처리를 얻는 방식이다.

여기에 **복제(Replication)**가 붙는다. 브로커 한 대가 죽어도 데이터가 사라지면 안 되므로
같은 데이터를 다른 브로커에도 복사해둔다. 이때 읽기/쓰기를 담당하는 파티션이
**리더(Leader)**, 그걸 복제해서 대기하는 게 **팔로워(Follower)**다.

문제는 여기서 생긴다. **"지금 어떤 브로커가 살아있는가", "어떤 파티션이 리더인가",
"장애가 나면 누가 새 리더가 되는가"** — 이 결정을 누가, 어떻게 내리는가? 브로커가 여러
대인데 다들 자기 마음대로 결정하면 클러스터 전체가 일관성을 잃는다. 이 조율 역할을
맡았던 게 Zookeeper였다.

## Zookeeper가 하던 일

- **브로커 상태 감시(Heartbeat)**: 브로커가 뜨면 "살아있다"고 알리고, 연결이 끊기면 다른
  브로커들에게 "죽었다"고 전파한다.
- **컨트롤러 선출**: 여러 브로커 중 하나를 "반장"(컨트롤러)으로 뽑는다. 컨트롤러는 죽은
  브로커의 리더 파티션을 다른 살아있는 브로커에게 재할당하는 역할을 한다.
- **메타데이터 저장**: 토픽/파티션 개수, 복제본이 어느 브로커에 있는지, 각 파티션의
  현재 리더가 누구인지 — 클러스터 운영에 필요한 명부 전체를 쥐고 있다.

정리하면 Kafka는 "데이터 창고", Zookeeper는 그 창고들을 관리하는 "관리 사무소"였다.

## Zookeeper 모드의 구조적 비용

| 구분 | Zookeeper 모드 | KRaft 모드 |
|---|---|---|
| 필수 구성 요소 | 카프카 + 주키퍼 | 카프카 단독 |
| 메타데이터 저장 위치 | 주키퍼 전용 디렉토리 | 카프카 내부 로그(`@metadata` 토픽) |
| 실행 순서 | 주키퍼가 완전히 뜬 뒤에 카프카 실행 (의존성 필수) | 카프카만 띄우면 끝 |
| 복잡도 | 두 종류의 분산 시스템을 따로 관리 | 카프카 하나만 관리 |
| 확장성 | 파티션이 많아지면 주키퍼 자체가 병목이 됨 | 훨씬 많은 파티션을 안정적으로 처리 |

가장 큰 비용은 "관리해야 할 분산 시스템이 두 개"라는 점이다. Zookeeper 자체도 별도로
장애 대응, 버전 관리, 튜닝이 필요한 시스템이고, 게다가 파티션 수가 늘어날수록
Zookeeper가 들고 있는 메타데이터 양도 커져서 그 자체가 확장성의 한계가 됐다.

## KRaft: 그 관리 사무소를 Kafka 안으로 흡수

2.8부터 도입되고 3.x부터 본격화된 **KRaft(Kafka Raft)**는 Raft 합의 알고리즘을 Kafka
브로커 내부에 직접 구현해서, Zookeeper가 하던 일(브로커 감시, 컨트롤러 선출, 메타데이터
관리)을 전부 Kafka 자신이 수행하게 만든 것이다. 메타데이터는 이제 별도 시스템이 아니라
Kafka의 내부 로그(`@metadata`라는 특수 토픽)에 저장된다 — 즉 Kafka가 자기 자신의
메타데이터를 자기 자신의 저장 메커니즘(로그)으로 관리하게 된 셈이다.

KRaft 모드에는 Zookeeper 모드에 없던 단계가 하나 추가되는데, 최초 실행 시 **클러스터
ID를 발급받아 포맷**하는 과정이다 — "이 브로커들은 하나의 팀"이라는 걸 정의하는 단계로,
Docker 이미지에서는 대개 자동 처리되지만 왜 필요한지는 알아둘 만하다: Zookeeper가
하던 "클러스터 정체성 관리" 역할의 일부가 이 초기화 단계로 옮겨온 것이다.

## 두 모드를 실제로 띄워보면 드러나는 차이

**Zookeeper 모드** — 두 컨테이너를 순서대로:

```yaml
services:
  zookeeper:
    image: bitnami/zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes

  kafka:
    image: bitnami/kafka:latest
    ports:
      - "9092:9092"
    environment:
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_ANONYMOUS_LOGIN=yes
    depends_on:
      - zookeeper   # 주키퍼가 먼저 떠야만 실행됨
```

**KRaft 모드** — Kafka 하나:

```yaml
services:
  kafka:
    image: bitnami/kafka:latest
    ports:
      - "9092:9092"
    environment:
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_NODE_ID=1
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@kafka:9093
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
```

여기서 `PROCESS_ROLES=controller,broker`가 KRaft의 핵심을 그대로 보여준다 — 브로커
프로세스 자신이 "데이터도 저장하고(broker), 컨트롤러 역할도 한다(controller)"고 선언하는
것. `CONTROLLER_QUORUM_VOTERS`는 예전에 Zookeeper 클러스터가 하던 합의 투표를 이제
Kafka 브로커들끼리(여기서는 자기 자신) 직접 하겠다는 설정이다. `depends_on` 같은
의존성 설정이 사라진 게 겉으로 보이는 차이지만, 실제로는 "합의와 메타데이터 관리를
누가 하는가"라는 아키텍처 자체가 바뀐 결과다.

## 정리

- Zookeeper는 Kafka 브로커들의 생사 감시, 컨트롤러 선출, 메타데이터 저장을 담당하던
  **외부 조율자**였다.
- KRaft는 이 조율 로직을 Raft 알고리즘으로 Kafka 내부에 구현해서, **별도 시스템 없이
  Kafka 스스로 조율**하게 만들었다.
- 실무적으로 체감되는 차이는 운영 대상 시스템이 하나로 줄어든다는 것, 그리고 파티션
  규모가 커져도 메타데이터 관리가 병목이 되지 않는다는 것이다.
- 새 프로젝트에서 Kafka를 붙인다면 KRaft 모드가 기본 선택지다. Zookeeper 모드를
  마주친다면 대개 오래된 인프라를 유지보수하는 상황일 가능성이 높다.
