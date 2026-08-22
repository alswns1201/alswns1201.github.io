---
title: "Kafka 프로듀서·컨슈머를 Spring Boot에 연동하기 — 기본 설정 다음에 봐야 할 것들"
date: 2025-12-14
categories: [Kafka]
---

Kafka에 메시지를 넣고 받는 것 자체는 몇 줄이면 끝난다. 진짜 판단이 필요한 지점은
"이걸 프로덕션에서 그대로 써도 되는가"다. 이메일 발송 이벤트를 예로 프로듀서와
컨슈머를 각각 붙여보고, 최소 설정 그대로 운영에 올리면 왜 위험한지를 정리한다.

## 프로듀서: 최소 설정

```groovy
implementation 'org.springframework.kafka:spring-kafka'
implementation 'com.fasterxml.jackson.core:jackson-databind'
```

```properties
spring.kafka.bootstrap-servers=[브로커 주소]
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
```

```java
@Service
@RequiredArgsConstructor
public class EmailService {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public void sendEmail(SendEmailRequestDto dto) {
        EmailSendMessage message = new EmailSendMessage(dto.from(), dto.to(), dto.subject(), dto.body());
        this.kafkaTemplate.send("email.send", toJsonString(message));
    }

    private String toJsonString(Object o) {
        ObjectMapper objectMapper = new ObjectMapper();
        try {
            return objectMapper.writeValueAsString(o);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Json 직렬화 실패");
        }
    }
}
```

이 코드는 동작은 한다. 하지만 프로덕션 기준으로 보면 세 가지가 걸린다.

### 1) 메시지에 키가 없다 — 순서 보장을 포기한 것과 같다

`kafkaTemplate.send("email.send", jsonString)`처럼 키 없이 보내면, 파티셔너가
라운드로빈으로(또는 배치 단위로) 파티션을 정한다. Kafka는 **파티션 내부**에서만
순서를 보장하므로, 키가 없으면 같은 수신자에게 가는 이메일 이벤트들이 서로 다른
파티션에 흩어질 수 있고, 그러면 어떤 순서로 처리될지 보장할 수 없다. "이 유저의
이벤트는 발생한 순서대로 처리돼야 한다"는 요구사항이 조금이라도 있다면, 그 유저를
식별하는 값(수신자 이메일, 유저 ID 등)을 키로 넣어야 한다 — 같은 키는 항상 같은
파티션으로 가는 게 Kafka 파티셔닝의 기본 규칙이기 때문이다.

```java
kafkaTemplate.send("email.send", dto.to(), toJsonString(message));
```

### 2) `ObjectMapper`를 매 호출마다 새로 만든다

`ObjectMapper`는 생성 비용이 작지 않고, 무엇보다 **스레드 세이프해서 재사용하도록
설계된 객체**다. 요청마다 `new ObjectMapper()`를 하는 건 불필요한 객체 생성 비용을
매번 지불하는 것과 같다. Bean으로 등록해서 재사용하거나, 아예 `JsonSerializer`를
Kafka 프로듀서 설정에 등록해서 직렬화를 프레임워크에 맡기는 편이 낫다.

```properties
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
```

이러면 서비스 코드에서 `toJsonString()` 같은 수동 직렬화 로직 자체가 사라진다.

### 3) 전송 실패를 무시하고 있다

`kafkaTemplate.send(...)`는 비동기로 동작하고 `CompletableFuture`를 반환한다. 반환값을
그냥 버리면, 브로커가 일시적으로 응답이 없거나 전송이 실패해도 애플리케이션은 그 사실을
전혀 모른다 — 로그도 안 남고 예외도 안 던져진다. 최소한 콜백을 붙여서 실패를 로깅하거나,
`acks` 설정(`acks=all`이면 모든 ISR 복제본의 ack을 받을 때까지 기다린다)을 명시적으로
정해야 "메시지가 실제로 안전하게 저장됐는가"에 대한 신뢰 수준을 스스로 알 수 있다.

```java
kafkaTemplate.send("email.send", dto.to(), toJsonString(message))
    .whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("이메일 이벤트 전송 실패: to={}", dto.to(), ex);
        }
    });
```

프로듀서 연동 자체는 몇 분이면 끝나지만, "이 메시지들이 순서대로 처리돼야 하는가",
"전송 실패를 감지할 방법이 있는가", "직렬화 객체를 매번 새로 만들고 있지 않은가" —
이 세 가지를 확인하지 않은 최소 설정은 프로토타입 단계에서만 안전하다.

## 컨슈머: 발행된 이벤트 소비하기

같은 `email.send` 토픽을 소비하는 컨슈머 차례다.

### 설정값이 뜻하는 것

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

### `@KafkaListener`

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

### 이 구성이 프로덕션에서는 왜 위험한가

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

프로듀서-컨슈머 구성은 Kafka의 기본 흐름을 이해하기엔 충분하지만, 그대로
프로덕션에 올리기 전에 확인해야 할 게 각각 세 가지씩 있다.

- **프로듀서**: 메시지 키로 순서를 보장하는가, 직렬화 객체를 재사용하는가, 전송
  실패를 감지하는가.
- **컨슈머**: `enable-auto-commit`을 그대로 두지 않았는가, 재시도/DLQ 전략이
  있는가, 중복 처리에 대비한 멱등성이 있는가.

다음에 Kafka를 다시 만지게 되면 이 여섯 가지부터 채우고 시작할 것.
