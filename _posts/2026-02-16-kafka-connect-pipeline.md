---
title: "Kafka Connect로 DB 파이프라인 구성하기 — Source/Sink Connector와 CDC"
date: 2026-02-16
categories: [Kafka]
tags: [kafka, database, cdc, msa]
---

MSA에서 서비스마다 자기 DB를 따로 두던 구조(예: 각 서비스가 H2를 각자 물고 있는 실습
환경)를 걷어내고, 메시지 큐를 거쳐 하나의 데이터베이스로 모으는 구조로 바꿀 때 선택지가
갈린다 — 애플리케이션 코드에서 직접 JDBC로 쓸 것인가, 아니면 **Kafka Connect**에
저장을 맡길 것인가. DB에 쌓인 데이터를 Kafka로 흘려보내는 쪽도 마찬가지다: 매번 폴링
코드를 직접 짜는 대신 커넥터 설정만으로 파이프라인을 구성할 수 있다.

이 글은 Kafka Connect의 Source/Sink Connector로 DB ↔ Kafka 파이프라인을 구성하는
방법과, 그 과정에서 실제로 신경 써야 하는 지점(incrementing 모드의 한계, upsert/삭제
전파, 스키마 자동 변경 권한)을 정리한다.

## Source Connector vs Sink Connector

- **Source Connector**: DB의 변화를 감지해서 Kafka 토픽으로 실어 나른다 (DB → Kafka)
- **Sink Connector**: Kafka 토픽에 쌓인 데이터를 읽어서 타겟 DB에 자동으로 넣어준다
  (Kafka → DB)

두 커넥터를 이어 붙이면 전체 흐름은 이렇다.

1. **Source DB** — 원본 데이터
2. **Source Connector** — DB 변화를 감지해 Kafka 토픽으로 전송
3. **Kafka Topic** — 중간 저장소
4. **Sink Connector** — 토픽을 구독하다 새 메시지가 오면 Target DB에 반영
5. **Target DB** — 최종 목적지

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

## Source Connector: JDBC 폴링과 그 한계

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

여기가 JDBC Source Connector의 핵심 문제다. `incrementing` 모드는 **새로 INSERT된
행만 감지**한다. 기존 행이 UPDATE되거나 DELETE되는 건 이 방식으로는 절대 잡히지
않는다 — PK 값이 이전 최댓값보다 커지는 이벤트가 아니기 때문이다. "최신 상태를
정확히 반영해야 하는" 파이프라인에 이 모드를 그대로 쓰면, 수정/삭제된 데이터가
Kafka 쪽에서는 계속 과거 상태로 남아있는 정합성 문제가 생긴다.

## 왜 애플리케이션이 직접 저장하지 않고 Connect를 거치는가

기존에는 엔티티를 통해 애플리케이션이 직접 DB에 쓰는 구조였다.

```java
OrderDto createdOrder = orderService.createOrder(orderDto);
```

이걸 "Producer 역할만 하고, 저장은 Kafka Connect에 맡기는" 구조로 바꾸면 얻는 것과
잃는 것이 분명하다.

**얻는 것**: 애플리케이션 코드가 DB 커넥션/트랜잭션을 직접 관리하지 않아도 된다.
저장 대상 DB를 바꾸거나 여러 개로 늘려야 할 때 Connector 설정만 바꾸면 된다.
서비스 코드는 "이벤트를 발행한다"는 책임만 지고, "어디에 어떻게 영속화할지"는
분리된다.

**잃는 것**: 저장 성공 여부를 애플리케이션이 즉시 알 수 없다. 기존 방식은 `save()`가
실패하면 그 자리에서 트랜잭션이 롤백되고 호출자가 바로 안다. Connector를 거치면
저장은 비동기이고, 실패해도 애플리케이션 응답에는 이미 반영되지 않는다 — 저장 실패를
감지하려면 Connector의 상태(status)를 별도로 모니터링해야 한다.

## Sink Connector: 메시지 구조와 등록

Kafka Connect의 JDBC Sink Connector가 메시지를 테이블 컬럼으로 매핑하려면, 메시지
자체에 스키마 정보가 있어야 한다.

```java
@Data @AllArgsConstructor
public class KafkaOrderDto implements Serializable {
    private Schema schema;
    private Payload payload;
}
```

프로듀서가 이 형태로 직렬화해서 보내면, Connect는 `schema`를 보고 어떤 타입의
어떤 필드인지 파악해서 타겟 테이블에 upsert한다.

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

몇 가지는 "편해서 켠다"가 아니라 **의도적으로 판단하고 켜야 하는** 옵션이다.

- **`auto.create` / `auto.evolve`**: 토픽 스키마를 기준으로 테이블을 자동
  생성/변경한다. 프로토타입에서는 편하지만, 프로덕션에서는 위험한 설정이다 — 이
  말은 곧 **데이터 파이프라인이 프로덕션 DB에 DDL을 실행할 권한을 갖는다**는
  뜻이다. Source 쪽 스키마가 실수로 바뀌면 그 변화가 그대로 Target DB의 테이블
  구조 변경으로 전파된다. 프로듀서 팀과 DB 스키마 오너가 다르면 이 옵션은 사실상
  "누구나 프로덕션 스키마를 바꿀 수 있다"는 구멍이 된다. 대부분의 프로덕션
  운영에서는 이 옵션을 끄고, 스키마 변경을 명시적인 마이그레이션 절차로 관리하는
  걸 권장한다.
- **`insert.mode: upsert` + `pk.mode` / `pk.fields`**: `insert` 모드는 항상 새 행을
  추가하려 하므로 같은 메시지가 두 번 도착하면 PK 충돌로 실패한다. `upsert`는 PK가
  이미 있으면 갱신하는 방식이라, **같은 메시지를 여러 번 처리해도 결과가 같다
  (멱등성)**. Kafka Connect는 기본적으로 at-least-once(적어도 한 번은 전달) 전달
  방식이라 재시도로 인한 중복 전달이 일어날 수 있는데, `upsert` + PK 지정 조합이
  바로 이 중복을 안전하게 흡수하는 장치다. `insert` 모드로 두면 재시도가 발생하는
  순간 파이프라인이 통째로 멈춘다. 단, `pk.fields`로 지정한 필드가 프로듀서 쪽에서
  유일성이 보장되지 않으면 upsert가 조용히 다른 데이터를 덮어쓴다.
- **`delete.enabled: false`**: Source에서 행이 삭제돼도 Target에는 반영되지
  않는다는 뜻이다. 삭제까지 동기화하려면 Kafka의 tombstone 메시지(value가 null인
  메시지)를 활용해야 하는데, 이건 Source Connector 쪽에서 삭제를 tombstone으로
  내보내도록 별도 설정이 필요하다. `delete.enabled: false`인 파이프라인은 사실상
  "추가/수정만 동기화, 삭제는 안 됨"이라는 걸 팀 전체가 인지하고 있어야 한다 —
  안 그러면 Source에서 지운 데이터가 Target에 계속 남아있는 걸 나중에야 발견하게
  된다.

## 상태 확인과 삭제

```
GET  http://localhost:8083/connectors/my-sink-connect/status
DELETE http://localhost:8083/connectors/my-sink-connect
```

`status`로 커넥터와 태스크의 `RUNNING` 여부를 확인할 수 있다. 태스크가 실패 상태로
멈춰 있으면 그 시점부터 이후 메시지가 전혀 반영되지 않으므로, 이 상태를 모니터링하지
않으면 동기화가 조용히 멈춘 채로 방치될 수 있다.

## JDBC 폴링의 근본적 한계와 CDC(Debezium)

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

- **JDBC Source/Sink Connector**: 배치성 데이터 로드, 새로 추가되는(INSERT)
  데이터만 관심 있는 경우, 보안 정책상 DB 로그 접근 권한을 줄 수 없는 경우, 또는
  "일단 동기화 파이프라인을 빠르게 구성해보고 싶다"는 단계.
- **CDC(Debezium 등)**: 실시간 데이터 동기화, 서비스 간 데이터 복제, 삭제 이벤트
  처리와 트랜잭션 순서 보장이 중요한 경우. 대신 DB 로그 접근 권한과 CDC 도구 자체의
  운영 부담(스키마 변경 대응, 초기 스냅샷 등)이 추가로 따라온다.

"CDC가 항상 더 좋다"가 아니라, **DELETE를 감지해야 하는가, DB 로그에 접근 가능한
운영 환경인가**가 실질적인 갈림길이다. 그리고 `auto.evolve`, `delete.enabled`,
`pk.mode` 같은 Connect 옵션은 편의 기능이 아니라 **스키마 변경 권한과 삭제 전파
여부를 결정하는 운영 정책**이므로, 켜기 전에 누가 책임지는 설정인지부터 정해야 한다.
