---
title: "Kafka Connect로 애플리케이션 코드 없이 DB에 데이터 적재하기"
date: 2026-02-16
categories: [개발일지, Kafka]
tags: [kafka, msa, database]
---

MSA에서 서비스마다 자기 DB를 따로 두던 구조(예: 각 서비스가 H2를 각자 물고 있는 실습
환경)를 걷어내고, 메시지 큐를 거쳐 하나의 데이터베이스로 모으는 구조로 바꿀 때 선택지가
갈린다 — 애플리케이션 코드에서 직접 JDBC로 쓸 것인가, 아니면 **Kafka Connect**에
저장을 맡길 것인가.

## Source Connector vs Sink Connector

- **Source Connector**: DB의 변화를 감지해서 Kafka 토픽으로 실어 나른다 (DB → Kafka)
- **Sink Connector**: Kafka 토픽에 쌓인 데이터를 읽어서 타겟 DB에 자동으로 넣어준다
  (Kafka → DB)

이 글은 Sink Connector로 주문(order) 데이터를 MariaDB에 적재하는 흐름을 다룬다.

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

## 메시지 구조: schema + payload

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

## Sink Connector 설정에서 실제로 위험한 옵션들

```json
{
  "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
  "topics": "orders",
  "connection.url": "jdbc:mysql://host.docker.internal:3306/mydb",
  "auto.create": "true",
  "auto.evolve": "true",
  "delete.enabled": "false",
  "insert.mode": "upsert",
  "pk.mode": "record_value",
  "pk.fields": "id"
}
```

몇 가지는 "편해서 켠다"가 아니라 **의도적으로 판단하고 켜야 하는** 옵션이다.

- **`auto.evolve: true`**: 메시지 스키마가 바뀌면 타겟 테이블 스키마를 Connector가
  자동으로 ALTER한다. 개발 단계에서는 편하지만, 프로덕션에서는 프로듀서 쪽 스키마
  변경이 검토 없이 그대로 DB 스키마 변경으로 이어진다는 뜻이다 — 프로듀서 팀과
  DB 스키마 오너가 다르면 이 옵션은 사실상 "누구나 프로덕션 스키마를 바꿀 수 있다"는
  구멍이 된다.
- **`delete.enabled: false`**: Kafka 토픽에서 삭제(tombstone) 메시지가 와도 타겟
  테이블에서 삭제하지 않는다는 뜻이다. 즉 이 파이프라인은 **soft-delete나 삭제
  이벤트가 전파되지 않는 구조**다 — 소스에서 데이터를 지워도 싱크 테이블에는 계속
  남아있을 수 있다는 걸 감안해야 한다.
- **`pk.mode: record_value`**: 메시지 payload 안의 필드(`pk.fields: "id"`)를 타겟
  테이블의 PK로 쓴다는 뜻이다. 이 필드가 프로듀서 쪽에서 유일성이 보장되지 않으면
  upsert가 조용히 데이터를 덮어쓴다.

## 정리

- Source Connector는 DB 변화를 Kafka로, Sink Connector는 Kafka 데이터를 DB로 옮긴다.
- 애플리케이션이 직접 저장하는 대신 Connect에 맡기면 책임은 분리되지만, 저장 성공
  여부를 즉시 알 수 없다는 비동기성의 대가를 치른다.
- `auto.evolve`, `delete.enabled`, `pk.mode` 같은 옵션은 편의 기능이 아니라 **스키마
  변경 권한과 삭제 전파 여부를 결정하는 운영 정책**이므로, 켜기 전에 누가 책임지는
  설정인지부터 정해야 한다.
