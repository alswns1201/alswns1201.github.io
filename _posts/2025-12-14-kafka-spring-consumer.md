---
title: "Kafka로 메시지 큐 다루기 (4) — Spring Boot 컨슈머 연동"
date: 2025-12-14
categories: [Kafka]
tags: [kafka, spring-boot]
---

[1편](Kafka 서버 설치)에서 EC2에 Kafka를 올리고, [3편](프로듀서 연동)에서 이메일
발송 이벤트를 발행하는 프로듀서를 붙였다. 이번엔 그 이벤트를 소비하는 컨슈머 차례다.

## 설정값이 뜻하는 것

```properties
server.port=0
spring.kafka.bootstrap-servers=[AWS EC2 서버 IP:PORT]
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.auto-offset-reset=earliest
```

각 옵션은 그냥 붙여넣을 값이 아니라 실제로 동작을 좌우한다.

- `auto-offset-reset=earliest`: 컨슈머 그룹이 커밋해둔 오프셋이 없을 때(처음 붙거나,
  커밋 기록이 만료됐을 때) 토픽의 맨 처음부터 읽을지(`earliest`), 지금 시점 이후 것만
  읽을지(`latest`)를 결정한다. 로컬 테스트에서는 지금까지 쌓인 메시지를 다 보고 싶으니
  `earliest`가 편하지만, 운영에서 신규 컨슈머 그룹을 `earliest`로 붙이면 과거 데이터를
  전부 재처리하게 되어 부작용(중복 이메일 발송 같은)이 생길 수 있다.
- `key/value-deserializer`: Kafka는 메시지를 바이트 배열로만 다룬다. 역직렬화기는
  "이 바이트를 어떻게 객체로 되돌릴지"를 정하는 계층이고, 프로듀서 쪽 직렬화기와
  반드시 짝이 맞아야 한다 — 여기서는 문자열로 보내고 문자열로 받은 뒤 애플리케이션
  코드(`EmailSendMessage.fromJson`)에서 JSON 파싱을 한 번 더 하는 구조다.

## `@KafkaListener`

```java
@Slf4j
@Service
public class EmailSendConsumer {

    @KafkaListener(topics = "email.send", groupId = "eamil-send-group")
    public void consumer(String message) {
        log.info("kafka message {} ", message);
        EmailSendMessage emailSendMessage = EmailSendMessage.fromJson(message);
        log.info("success");
    }
}
```

`groupId`가 컨슈머 그룹을 결정한다. 같은 `groupId`를 가진 인스턴스가 여러 개면 Kafka는
토픽의 파티션을 그 인스턴스들에게 나눠준다 — 즉 컨슈머를 늘리는 것만으로 처리량을
수평 확장할 수 있다(단, 파티션 수가 상한이다. 파티션이 3개면 4번째 인스턴스는 놀게 된다).

## 이 구성이 프로덕션에서는 왜 위험한가

이 코드는 로컬 실습용으로는 충분하지만, 그대로 운영에 올리면 문제가 생긴다.

**1) 자동 커밋(auto-commit)의 함정**

설정에 명시하지 않으면 `spring.kafka.consumer.enable-auto-commit`은 기본적으로
활성화되어 있고, 컨슈머는 메시지를 "처리했든 안 했든" 주기적으로 오프셋을 커밋한다.
위 코드에서 `EmailSendMessage.fromJson(message)`가 예외를 던지면 어떻게 될까 —
이미 오프셋이 커밋된 뒤라면 그 메시지는 유실된 것과 같다. 최소한 이 메시지 유형에서는
`enable-auto-commit=false`로 두고, 처리에 성공했을 때만 수동으로
`Acknowledgment.acknowledge()`를 호출하는 편이 안전하다.

**2) 실패 시 재시도/DLQ 전략 부재**

지금 코드는 예외가 나면 그냥 로그만 남기고 끝난다. 처리 실패가 일시적인 문제(DB
커넥션 순간 끊김 등)인지 영구적인 문제(메시지 포맷이 애초에 잘못됨)인지 구분하지
않으면, 재시도 로직 없이는 그 메시지는 그냥 사라진다. `DefaultErrorHandler` +
`DeadLetterPublishingRecoverer`로 N번 재시도 후 실패하면 별도 DLQ 토픽으로 보내는
구성이 실무 기본값에 가깝다.

**3) 멱등성(idempotency) 문제**

Kafka의 기본 전달 보장은 "적어도 한 번(at-least-once)"이다 — 즉 같은 메시지가
두 번 이상 도착할 수 있다(컨슈머가 처리 후 커밋 직전에 죽는 경우 등). 이메일
발송처럼 부작용이 있는 처리를 하는 컨슈머라면, 메시지 자체에 멱등키를 두고
"이미 처리한 메시지인지" 확인하는 로직이 없으면 중복 발송이 실제로 일어난다.

## 정리

이 시리즈에서 만든 프로듀서-컨슈머 구성은 Kafka의 기본 흐름을 이해하기엔 충분하지만,
`enable-auto-commit`, 에러 핸들링, 멱등성이라는 세 가지를 붙이지 않은 상태로는
프로덕션에 그대로 못 쓴다. 다음에 Kafka를 다시 만지게 되면 이 세 가지부터 채우고
시작할 것.
