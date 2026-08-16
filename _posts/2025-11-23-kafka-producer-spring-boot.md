---
title: "Kafka 프로듀서를 Spring Boot에 연동하기 — 기본 설정 다음에 봐야 할 것들"
date: 2025-11-23
categories: [개발일지, Kafka]
tags: [kafka, spring-boot]
---

Kafka에 메시지를 넣는 것 자체는 몇 줄이면 끝난다. 진짜 판단이 필요한 지점은 "이걸
프로덕션에서 그대로 써도 되는가"다.

## 최소 설정

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

## 1) 메시지에 키가 없다 — 순서 보장을 포기한 것과 같다

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

## 2) `ObjectMapper`를 매 호출마다 새로 만든다

`ObjectMapper`는 생성 비용이 작지 않고, 무엇보다 **스레드 세이프해서 재사용하도록
설계된 객체**다. 요청마다 `new ObjectMapper()`를 하는 건 불필요한 객체 생성 비용을
매번 지불하는 것과 같다. Bean으로 등록해서 재사용하거나, 아예 `JsonSerializer`를
Kafka 프로듀서 설정에 등록해서 직렬화를 프레임워크에 맡기는 편이 낫다.

```properties
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
```

이러면 서비스 코드에서 `toJsonString()` 같은 수동 직렬화 로직 자체가 사라진다.

## 3) 전송 실패를 무시하고 있다

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

## 정리

프로듀서 연동 자체는 몇 분이면 끝나지만, "이 메시지들이 순서대로 처리돼야 하는가",
"전송 실패를 감지할 방법이 있는가", "직렬화 객체를 매번 새로 만들고 있지 않은가" —
이 세 가지를 확인하지 않은 최소 설정은 프로토타입 단계에서만 안전하다.
