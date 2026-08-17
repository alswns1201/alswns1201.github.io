---
title: "Kafka Connect JDBC Sink Connector로 DB 간 데이터 동기화하기"
date: 2026-02-14
categories: [Kafka]
tags: [kafka, database, devops]
---

Source DB의 변경을 다른 DB에 반영해야 할 때, 애플리케이션 코드로 직접 동기화 로직을
짜는 대신 Kafka Connect의 Sink Connector로 선언적으로 처리할 수 있다. 이건 사실상
경량 CDC(Change Data Capture) 파이프라인이다.

## 전체 데이터 흐름

1. **Source DB** — 원본 데이터
2. **Source Connector** — DB 변화를 감지해 Kafka 토픽으로 전송
3. **Kafka Topic** — 중간 저장소
4. **Sink Connector** — 토픽을 구독하다 새 메시지가 오면 Target DB에 반영
5. **Target DB** — 최종 목적지

## Sink Connector 등록

```json
{
  "name": "my-mariadb-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "my_topic_users",
    "connection.url": "jdbc:mysql://host.docker.internal:3306/mydb",
    "connection.user": "spring",
    "connection.password": "spring123",
    "auto.create": "true",
    "auto.evolve": "true",
    "delete.enabled": "false",
    "insert.mode": "upsert",
    "pk.mode": "record_value",
    "pk.fields": "id",
    "table.name.format": "users_sink",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter"
  }
}
```

`kafka-connect` 컨테이너(포트 8083)의 `/connectors`에 POST로 등록한다.

## 옵션 하나하나가 실제로 무슨 위험을 지고 있는가

**`auto.create` / `auto.evolve`**: 토픽 스키마를 기준으로 테이블을 자동 생성/변경한다.
프로토타입에서는 편하지만, 프로덕션에서는 위험한 설정이다 — 이 말은 곧 **데이터
파이프라인이 프로덕션 DB에 DDL을 실행할 권한을 갖는다**는 뜻이다. Source 쪽 스키마가
실수로 바뀌면 그 변화가 그대로 Target DB의 테이블 구조 변경으로 전파된다. 대부분의
프로덕션 운영에서는 이 옵션을 끄고, 스키마 변경을 명시적인 마이그레이션 절차로
관리하는 걸 권장한다.

**`insert.mode: upsert` + `pk.mode` / `pk.fields`**: `insert` 모드는 항상 새 행을
추가하려 하므로 같은 메시지가 두 번 도착하면 PK 충돌로 실패한다. `upsert`는 PK가
이미 있으면 갱신하는 방식이라, **같은 메시지를 여러 번 처리해도 결과가 같다(멱등성)**.
Kafka Connect는 기본적으로 at-least-once(적어도 한 번은 전달) 전달 방식이라 재시도로
인한 중복 전달이 일어날 수 있는데, `upsert` + PK 지정 조합이 바로 이 중복을 안전하게
흡수하는 장치다. `insert` 모드로 두면 재시도가 발생하는 순간 파이프라인이 통째로
멈춘다.

**`delete.enabled: false`**: Source에서 행이 삭제돼도 Target에는 반영되지 않는다는
뜻이다. 삭제까지 동기화하려면 Kafka의 tombstone 메시지(value가 null인 메시지)를
활용해야 하는데, 이건 Source Connector 쪽에서 삭제를 tombstone으로 내보내도록
별도 설정이 필요하다. `delete.enabled: false`인 파이프라인은 사실상 "추가/수정만
동기화, 삭제는 안 됨"이라는 걸 팀 전체가 인지하고 있어야 한다 — 안 그러면 Source에서
지운 데이터가 Target에 계속 남아있는 걸 나중에야 발견하게 된다.

## 상태 확인과 삭제

```
GET  http://localhost:8083/connectors/my-sink-connect/status
DELETE http://localhost:8083/connectors/my-sink-connect
```

`status`로 커넥터와 태스크의 `RUNNING` 여부를 확인할 수 있다. 태스크가 실패 상태로
멈춰 있으면 그 시점부터 이후 메시지가 전혀 반영되지 않으므로, 이 상태를 모니터링하지
않으면 동기화가 조용히 멈춘 채로 방치될 수 있다.

## 이 방식의 한계와 대안

폴링/토픽 기반의 JDBC Connector는 구성이 간단하다는 장점이 있지만, 진짜 CDC(로그
기반, 트랜잭션 순서 보장, 삭제 포함 전체 변경 이력 캡처)가 필요한 프로덕션
환경에서는 **Debezium** 같은 로그 기반 CDC 도구가 더 적합하다. Debezium은 DB의
트랜잭션 로그(MySQL binlog 등)를 직접 읽기 때문에 폴링 주기에 따른 지연이 없고,
삭제를 포함한 모든 변경을 순서 보장과 함께 캡처한다. JDBC Sink/Source Connector
조합은 "일단 동기화 파이프라인을 빠르게 구성해보고 싶다"는 단계에는 적합하지만,
그 이상의 정합성 요구사항이 생기면 로그 기반 CDC로 옮겨가는 걸 검토해야 한다.
