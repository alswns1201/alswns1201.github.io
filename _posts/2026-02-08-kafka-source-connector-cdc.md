---
title: "Kafka Connect로 DB 변경사항 스트리밍하기: JDBC Source vs CDC"
date: 2026-02-08
categories: [Kafka]
tags: [kafka, database, cdc]
---

DB에 쌓인 데이터를 Kafka로 실시간에 가깝게 흘려보내고 싶을 때, 매번 폴링 코드를
직접 짜는 대신 **Kafka Connect**를 쓰면 커넥터 설정만으로 이 파이프라인을 구성할 수
있다. 여기서는 가장 기본적인 JDBC Source Connector로 시작해서, 이게 왜 한계가
있는지, 그리고 CDC(Change Data Capture) 방식이 그 한계를 어떻게 다르게 푸는지를
짚는다.

## Kafka Connect 띄우기

기존에 떠 있는 Kafka 브로커에, Kafka Connect 엔진과 DB 커넥터 플러그인이 포함된
별도 컨테이너를 연결하는 구조다.

```yaml
services:
  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.5.0
    network_mode: "container:kafka-broker"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: 'localhost:9092'
      CONNECT_GROUP_ID: compose-connect-group
      CONNECT_CONFIG_STORAGE_TOPIC: docker-connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: docker-connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: docker-connect-status
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
```

Kafka Connect가 뜨면 내부적으로 `docker-connect-configs`, `docker-connect-offsets`,
`docker-connect-status` 같은 관리용 토픽이 자동 생성된다 — Connect 자신의 상태
(어떤 커넥터가 어디까지 읽었는지)를 Kafka 안에 저장하는 구조다.

JDBC 커넥터를 설치한 뒤, MySQL과 통신하려면 MySQL 전용 드라이버(`.jar`)를 커넥터
폴더에 추가로 넣어줘야 한다 — 커넥터 자체는 프로토콜 어댑터일 뿐, DB별 드라이버는
별도로 필요하다.

## JDBC Source Connector 설정

```json
{
  "name": "my-mariadb-source-connector",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:mysql://host.docker.internal:3306/mydb?allowPublicKeyRetrieval=true&useSSL=false",
    "mode": "incrementing",
    "incrementing.column.name": "id",
    "table.whitelist": "users",
    "topic.prefix": "my_topic_",
    "tasks.max": "1"
  }
}
```

- `mode: incrementing`은 지정한 컬럼(보통 PK)의 값이 이전에 읽은 최댓값보다 큰
  행만 골라 읽는 방식이다 — 내부적으로 `SELECT * FROM users WHERE id > ?`를
  주기적으로 반복하는 것과 같다.
- `table.whitelist`는 실수로 전체 테이블을 다 읽지 않도록 명시적으로 대상을
  제한하는 안전장치다.

## incrementing 모드의 근본적인 한계

여기가 JDBC Source Connector의 핵심 문제다. `incrementing` 모드는 **새로 INSERT된
행만 감지**한다. 기존 행이 UPDATE되거나 DELETE되는 건 이 방식으로는 절대 잡히지
않는다 — PK 값이 이전 최댓값보다 커지는 이벤트가 아니기 때문이다. "최신 상태를
정확히 반영해야 하는" 파이프라인에 이 모드를 그대로 쓰면, 수정/삭제된 데이터가
Kafka 쪽에서는 계속 과거 상태로 남아있는 정합성 문제가 생긴다.

## CDC(Debezium)가 다른 이유

| 비교 항목 | JDBC Source Connector (일반) | Debezium (CDC 방식) |
|---|---|---|
| 방식 | 일정 시간마다 SELECT 쿼리를 반복 실행 | DB의 로그(Binlog 등)를 직접 읽음 |
| DB 부하 | 계속 쿼리를 날리므로 부하가 있음 | 로그만 읽으므로 부하가 거의 없음 |
| DELETE 감지 | 불가능 (행이 지워지면 쿼리에 안 잡힘) | 가능 (로그에 삭제 기록이 남음) |
| 데이터 추적 | 현재 상태만 알 수 있음 | 수정 전/후 데이터를 모두 알 수 있음 |

Debezium 같은 CDC 도구는 DB가 복제(replication)를 위해 이미 기록하는 로그(MySQL
binlog 등)를 읽는다. 이건 DB 입장에서 보면 "새로운 쿼리 부하"가 아니라 "이미 하고
있던 일을 하나 더 구독하는 것"에 가깝다 — 그래서 INSERT/UPDATE/DELETE를 전부,
DB에 추가 부하를 주지 않고 잡아낼 수 있다.

## 선택 기준

- **JDBC Source Connector**: 배치성 데이터 로드, 새로 추가되는(INSERT) 데이터만
  관심 있는 경우, 또는 보안 정책상 DB 로그 접근 권한을 줄 수 없는 경우.
- **CDC(Debezium 등)**: 실시간 데이터 동기화, 서비스 간 데이터 복제, 삭제 이벤트
  처리가 중요한 경우. 대신 DB 로그 접근 권한과 CDC 도구 자체의 운영 부담(스키마
  변경 대응, 초기 스냅샷 등)이 추가로 따라온다.

"CDC가 항상 더 좋다"가 아니라, **DELETE를 감지해야 하는가, DB 로그에 접근 가능한
운영 환경인가**가 실질적인 갈림길이다.
