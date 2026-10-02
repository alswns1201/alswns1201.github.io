---
title: "Redis Pub/Sub과 Stream: 차이, 실무 활용처, Spring 예제 코드"
date: 2026-10-01
categories: [Redis]
---

*(Redis 자체의 기본 개념과 Spring 연동은 [Spring Boot와 Redis 기본 개념](/posts/spring-redis-basics/) 글에서
다뤘다. 이 글은 그중 "메시징" 기능인 Pub/Sub과 Stream에 집중한다.)*

Redis를 캐시로만 쓰다가 "서버 여러 대에 이벤트를 뿌려야 하는데 Kafka까지 띄우긴 부담스럽다"는
상황을 만나면 자연스럽게 Redis의 메시징 기능을 찾게 된다. 그런데 Redis에는 메시징 기능이 두
가지 있다 — **Pub/Sub**과 **Stream**. 이름만 보면 비슷해 보이지만 보장하는 것이 완전히 다르고,
잘못 고르면 "메시지가 가끔 사라진다" 같은 재현하기 어려운 장애로 이어진다.

한 줄로 요약하면 이렇다.

- **Pub/Sub** — 지금 듣고 있는 사람에게만 방송한다. 저장하지 않는다. (라디오)
- **Stream** — 로그에 append하고, 소비자가 어디까지 읽었는지 추적한다. (Kafka의 축소판)

## 1. Pub/Sub

### 동작 방식

```bash
# 터미널 A — 구독
127.0.0.1:6379> SUBSCRIBE chat:room:1
1) "subscribe"
2) "chat:room:1"
3) (integer) 1

# 터미널 B — 발행
127.0.0.1:6379> PUBLISH chat:room:1 "hello"
(integer) 1          # ← 이 메시지를 받은 구독자 수

# 패턴 구독도 가능
127.0.0.1:6379> PSUBSCRIBE chat:room:*
```

`PUBLISH`는 채널을 구독 중인 클라이언트 연결에 메시지를 그대로 밀어넣고 끝난다. Redis는
메시지를 어디에도 저장하지 않는다. 그래서 다음과 같은 성질이 생긴다.

| 성질 | 의미 |
|---|---|
| **At-most-once** | 구독자가 그 순간 연결되어 있지 않으면 메시지는 영영 사라진다. 재시도/재전송 없음. |
| **ACK 없음** | 구독자가 메시지를 받고 처리하다 죽어도 Redis는 모른다. |
| **히스토리 없음** | 나중에 접속한 구독자는 과거 메시지를 볼 수 없다. |
| **Fan-out** | 같은 채널의 모든 구독자가 같은 메시지를 받는다 (부하 분산 X, 방송 O). |
| **매우 빠름** | 저장/추적 비용이 없어서 지연이 매우 낮다. |

`PUBLISH`의 반환값이 "받은 구독자 수"라는 점은 디버깅할 때 유용하다. 0이 나오면 지금 아무도
듣고 있지 않다는 뜻이고, 그 메시지는 이미 버려진 것이다.

### 운영에서 알아둘 점

- **구독 모드 연결은 전용이다.** RESP2 프로토콜에서는 `SUBSCRIBE`를 호출한 연결로
  `(P|S)SUBSCRIBE`, `(P|S)UNSUBSCRIBE`, `PING`, `QUIT` 외의 명령을 보낼 수 없다. 그래서
  Spring Data Redis도 구독용 연결을 별도로 잡는다 (RESP3에서는 이 제약이 풀렸다).
- **느린 구독자는 끊긴다.** Redis는 구독자에게 보낼 메시지를 클라이언트 출력 버퍼에 쌓는데,
  기본 설정 `client-output-buffer-limit pubsub 32mb 8mb 60`을 넘으면(32MB 즉시, 또는 8MB를
  60초 이상 유지) 그 연결을 강제로 끊는다. 끊긴 동안의 메시지는 당연히 유실된다.
- **클러스터에서는 Sharded Pub/Sub을 고려한다.** 클러스터 모드에서 일반 `PUBLISH`는 모든
  노드로 브로드캐스트되기 때문에 노드가 많을수록 클러스터 버스 트래픽이 커진다. Redis 7.0부터
  `SPUBLISH`/`SSUBSCRIBE`를 쓰면 채널이 키처럼 슬롯에 매핑되어 해당 샤드 안에서만 전파된다.

### 실무 활용처

공통점은 **"메시지 하나쯤 놓쳐도 시스템이 스스로 복구되거나, 놓친 게 큰 문제가 아닌 경우"**다.

1. **로컬 캐시 무효화 (가장 흔한 용도)**
   서버마다 Caffeine 같은 로컬 캐시를 두고, 데이터가 바뀌면 "이 키 지워"를 모든 서버에
   방송한다. 메시지를 놓쳐도 로컬 캐시에 TTL을 짧게 걸어두면 결국 정합성이 맞춰진다.
2. **WebSocket/SSE 다중 서버 브로드캐스트**
   채팅·알림 서버가 여러 대일 때, 사용자 A는 1번 서버에, 사용자 B는 2번 서버에 소켓이
   붙어 있다. 1번 서버가 받은 메시지를 Redis Pub/Sub으로 뿌리면 각 서버가 자기에게 붙은
   소켓으로 전달한다. 채팅 원문은 DB에 따로 저장하고, Pub/Sub은 "실시간 전달" 역할만 한다.
3. **설정/피처 플래그 변경 전파**
   관리자가 설정을 바꾸면 "리로드해" 신호만 방송하고, 실제 값은 각 서버가 DB나 Redis 키에서
   다시 읽는다. 신호를 놓쳐도 주기적 리로드가 보완한다.
4. **Keyspace Notification**
   `notify-keyspace-events Ex` 설정 후 `__keyevent@0__:expired` 채널을 구독하면 키 만료
   이벤트를 받을 수 있다. 다만 이것도 Pub/Sub이라 유실될 수 있고, 만료 이벤트는 키가
   "실제로 삭제될 때" 발생해서 TTL 시점과 정확히 일치하지 않는다. **결제 만료 처리 같은
   중요한 로직의 트리거로 쓰면 안 된다.**

### Spring 예제: 다중 서버 로컬 캐시 무효화

```java
@Configuration
public class RedisPubSubConfig {

    public static final String CACHE_INVALIDATE_CHANNEL = "cache:invalidate";

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory,
            CacheInvalidationListener listener) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(listener, new ChannelTopic(CACHE_INVALIDATE_CHANNEL));
        return container;
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class CacheInvalidationListener implements MessageListener {

    private final Cache<String, Product> productLocalCache; // Caffeine

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String key = new String(message.getBody(), StandardCharsets.UTF_8);
        productLocalCache.invalidate(key);
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;
    private final StringRedisTemplate redisTemplate;

    @Transactional
    public void updatePrice(Long productId, long price) {
        Product product = productRepository.findById(productId).orElseThrow();
        product.changePrice(price);
        // 커밋 이후에 방송해야, 다른 서버가 캐시를 지우고 다시 읽을 때 새 값을 읽는다
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                redisTemplate.convertAndSend(
                        RedisPubSubConfig.CACHE_INVALIDATE_CHANNEL, "product:" + productId);
            }
        });
    }
}
```

포인트는 **트랜잭션 커밋 이후에 발행**하는 것이다. 커밋 전에 발행하면 다른 서버가 캐시를 지우고
DB를 다시 읽는 시점에 아직 옛 값이 보여서, 옛 값이 다시 캐시에 들어가는 경쟁 상태가 생긴다.
(`@TransactionalEventListener(phase = AFTER_COMMIT)`로 같은 효과를 낼 수도 있다.)

## 2. Stream

### 동작 방식

Redis 5.0에 추가된 Stream은 **append-only 로그** 자료구조다. 메시지가 키에 저장되고, 각 항목은
`<밀리초 타임스탬프>-<시퀀스>` 형태의 ID를 갖는다.

```bash
# 추가 (* = ID 자동 생성), MAXLEN ~ 로 대략 10만 개만 유지
127.0.0.1:6379> XADD stream:order MAXLEN ~ 100000 * orderId 1001 status PAID
"1759280400000-0"

# 범위 조회 — 저장되어 있으니 과거 메시지도 읽을 수 있다
127.0.0.1:6379> XRANGE stream:order - +

# 단순 소비 (Pub/Sub처럼 새 메시지를 기다림, $ = 지금 이후)
127.0.0.1:6379> XREAD BLOCK 5000 STREAMS stream:order $
```

여기까지는 "저장되는 Pub/Sub" 정도인데, Stream의 진짜 핵심은 **Consumer Group**이다.

### Consumer Group: 부하 분산 + ACK + 재처리

```bash
# 그룹 생성 ($ = 지금 이후 메시지부터, MKSTREAM = 스트림 없으면 생성)
XGROUP CREATE stream:order notification-group $ MKSTREAM

# consumer-1이 그룹 이름으로 읽기 (> = 아직 아무에게도 전달 안 된 새 메시지)
XREADGROUP GROUP notification-group consumer-1 COUNT 10 BLOCK 2000 STREAMS stream:order >

# 처리 완료 후 ACK
XACK stream:order notification-group 1759280400000-0

# ACK 안 된 메시지 목록 (누가 들고 있는지, 얼마나 됐는지, 몇 번 전달됐는지)
XPENDING stream:order notification-group - + 10

# 60초 이상 처리 안 된 메시지를 consumer-2가 가져오기 (Redis 6.2+)
XAUTOCLAIM stream:order notification-group consumer-2 60000 0-0 COUNT 10
```

Consumer Group의 동작을 정리하면 이렇다.

- **같은 그룹 안에서는 메시지가 한 consumer에게만 간다** (부하 분산). 서버 3대가 같은 그룹으로
  읽으면 메시지가 나눠서 처리된다.
- **다른 그룹은 같은 메시지를 각자 전부 받는다** (fan-out). "알림 그룹", "포인트 적립 그룹"이
  같은 주문 스트림을 독립적으로 소비할 수 있다. Kafka의 consumer group과 같은 개념이다.
- 전달된 메시지는 ACK 전까지 그룹의 **PEL(Pending Entries List)**에 남는다. consumer가 처리
  중에 죽으면 메시지는 PEL에 남아 있고, `XAUTOCLAIM`/`XCLAIM`으로 다른 consumer가 가져가서
  재처리할 수 있다.
- consumer가 재시작했을 때 `>` 대신 `0`으로 `XREADGROUP`을 호출하면 **자기 PEL에 남은 메시지**
  (받았지만 ACK 못 한 것)를 다시 받는다.

즉 Stream은 **At-least-once**를 보장한다. 반대로 말하면 **같은 메시지가 두 번 처리될 수 있으므로
consumer 로직은 반드시 멱등하게** 만들어야 한다 (주문 ID 기준 중복 체크, 유니크 제약 등).

### 운영에서 알아둘 점

- **길이 제한은 직접 걸어야 한다.** Stream은 ACK 여부와 관계없이 메시지를 지우지 않는다.
  `XADD ... MAXLEN ~ N`이나 `XTRIM ... MINID ~ <id>`(6.2+)로 잘라내지 않으면 메모리가 계속
  늘어난다. `~`(approximate)를 붙이면 내부 노드 단위로 잘라서 훨씬 효율적이다.
- **트리밍은 ACK를 신경 쓰지 않는다.** 아직 처리되지 않은 메시지도 MAXLEN에 걸리면 지워진다.
  consumer가 오래 멈춰 있을 수 있다면 MAXLEN을 넉넉히 잡거나 consumer lag을 모니터링해야 한다.
- **영속성은 Redis 설정을 따른다.** Stream도 결국 메모리에 있는 키다. AOF/RDB 설정에 따라
  재시작 시 최근 데이터가 유실될 수 있고, 복제가 비동기라서 failover 순간에도 유실 가능성이 있다.
  Kafka처럼 "디스크에 복제 커밋된 뒤 ACK"를 보장하지는 않는다.
- **계속 실패하는 메시지(poison message) 처리.** `XPENDING`이 알려주는 전달 횟수(delivery
  count)가 일정 횟수를 넘으면 별도 DLQ 스트림으로 옮기고 ACK하는 로직을 직접 만들어야 한다.
  Redis가 자동으로 해주지 않는다.
- **모니터링**: `XINFO GROUPS stream:order`로 그룹별 pending 수와 `lag`(7.0+)을 확인할 수 있다.

### 실무 활용처

공통점은 **"유실되면 안 되지만, Kafka를 따로 운영할 만큼의 규모/요구사항은 아닌 경우"**다.

1. **도메인 이벤트 기반 후처리**
   주문 완료 → 알림 발송, 포인트 적립, 통계 집계. 주문 API는 `XADD`만 하고 바로 응답하고,
   후처리는 각 consumer group이 독립적으로 수행한다. 알림 서버가 잠깐 죽어도 재시작 후 이어서
   처리한다.
2. **비동기 작업 큐**
   이미지 리사이징, 엑셀 생성, 외부 API 호출처럼 오래 걸리는 작업을 큐에 넣고 워커 여러 대가
   나눠 처리한다. 예전에는 `LPUSH`/`BRPOP` List로 많이 구현했지만, List는 꺼낸 순간 사라져서
   워커가 죽으면 작업이 유실된다. Stream은 PEL 덕분에 이 문제가 없다.
3. **Outbox 릴레이의 경량 대체/중계**
   DB Outbox 테이블을 폴링해서 Stream에 넣고, 여러 서비스가 각자의 그룹으로 소비하는 구성.
4. **활동 로그·이벤트 수집, 실시간 피드**
   사용자 행동 로그, IoT 센서 값처럼 시간순으로 쌓이고 "최근 N개" 조회가 필요한 데이터.
   ID가 타임스탬프 기반이라 `XRANGE`로 시간 범위 조회가 자연스럽다.
5. **채팅 메시지 히스토리 + 실시간 전달**
   방마다 스트림을 두면 "최근 50개 메시지 로드(`XREVRANGE ... COUNT 50`)"와 "새 메시지
   대기(`XREAD BLOCK`)"를 같은 자료구조로 해결할 수 있다.

### Spring 예제: 주문 이벤트 처리

**발행 (Producer)**

```java
@Component
@RequiredArgsConstructor
public class OrderEventPublisher {

    public static final String ORDER_STREAM = "stream:order";
    private static final long MAX_LEN = 100_000;

    private final StringRedisTemplate redisTemplate;

    public RecordId publishPaid(Long orderId, Long userId, long amount) {
        MapRecord<String, String, String> record = StreamRecords.newRecord()
                .in(ORDER_STREAM)
                .ofMap(Map.of(
                        "type", "ORDER_PAID",
                        "orderId", orderId.toString(),
                        "userId", userId.toString(),
                        "amount", String.valueOf(amount)));

        RecordId id = redisTemplate.opsForStream().add(record);
        redisTemplate.opsForStream().trim(ORDER_STREAM, MAX_LEN, true); // MAXLEN ~
        return id;
    }
}
```

**소비 (Consumer Group)**

```java
@Slf4j
@Configuration
@RequiredArgsConstructor
public class OrderStreamConsumerConfig {

    private static final String GROUP = "notification-group";

    private final RedisConnectionFactory connectionFactory;
    private final StringRedisTemplate redisTemplate;
    private final OrderNotificationListener listener;

    @Value("${HOSTNAME:local}")  // 인스턴스마다 consumer 이름이 달라야 한다
    private String consumerName;

    @Bean(initMethod = "start", destroyMethod = "stop")
    public StreamMessageListenerContainer<String, MapRecord<String, String, String>> orderStreamContainer() {
        createGroupIfAbsent();

        var options = StreamMessageListenerContainer.StreamMessageListenerContainerOptions
                .builder()
                .pollTimeout(Duration.ofSeconds(2))  // XREADGROUP BLOCK 2000
                .batchSize(10)                       // COUNT 10
                .errorHandler(e -> log.error("stream poll error", e))
                .build();

        var container = StreamMessageListenerContainer.create(connectionFactory, options);

        var request = StreamMessageListenerContainer.StreamReadRequest
                .builder(StreamOffset.create(OrderEventPublisher.ORDER_STREAM, ReadOffset.lastConsumed())) // ">"
                .consumer(Consumer.from(GROUP, consumerName))
                .autoAcknowledge(false)        // 처리 성공 후 직접 ACK
                .cancelOnError(e -> false)     // 기본값은 에러 시 구독 취소 → 리스너가 조용히 멈춘다
                .build();

        container.register(request, listener);
        return container;
    }

    private void createGroupIfAbsent() {
        try {
            redisTemplate.execute((RedisCallback<String>) conn -> conn.streamCommands().xGroupCreate(
                    OrderEventPublisher.ORDER_STREAM.getBytes(StandardCharsets.UTF_8),
                    GROUP,
                    ReadOffset.from("0"),
                    true)); // MKSTREAM
        } catch (RedisSystemException e) {
            // 이미 그룹이 있으면 BUSYGROUP 에러 → 무시
            if (!String.valueOf(e.getRootCause().getMessage()).contains("BUSYGROUP")) {
                throw e;
            }
        }
    }
}
```

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class OrderNotificationListener
        implements StreamListener<String, MapRecord<String, String, String>> {

    private static final String GROUP = "notification-group";

    private final StringRedisTemplate redisTemplate;
    private final NotificationService notificationService;

    @Override
    public void onMessage(MapRecord<String, String, String> record) {
        Map<String, String> body = record.getValue();
        try {
            // 멱등 처리: 같은 orderId로 이미 발송했다면 내부에서 skip
            notificationService.sendPaidNotification(
                    Long.valueOf(body.get("orderId")),
                    Long.valueOf(body.get("userId")));

            redisTemplate.opsForStream().acknowledge(record.getStream(), GROUP, record.getId());
        } catch (Exception e) {
            // ACK하지 않으면 PEL에 남아 재처리 대상이 된다
            log.warn("order notification failed. id={}", record.getId(), e);
        }
    }
}
```

**재처리 (죽은 consumer의 메시지 회수 + DLQ)**

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class OrderPendingReclaimer {

    private static final String STREAM = OrderEventPublisher.ORDER_STREAM;
    private static final String DLQ_STREAM = "stream:order:dlq";
    private static final String GROUP = "notification-group";
    private static final Duration MIN_IDLE = Duration.ofMinutes(1);
    private static final long MAX_DELIVERY = 5;

    private final StringRedisTemplate redisTemplate;
    private final OrderNotificationListener listener;

    @Value("${HOSTNAME:local}")
    private String consumerName;

    @Scheduled(fixedDelay = 30_000)
    public void reclaim() {
        StreamOperations<String, String, String> ops = redisTemplate.opsForStream();
        PendingMessages pending = ops.pending(STREAM, GROUP, Range.unbounded(), 100);

        for (PendingMessage pm : pending) {
            if (pm.getElapsedTimeSinceLastDelivery().compareTo(MIN_IDLE) < 0) {
                continue; // 아직 처리 중일 수 있음
            }

            List<MapRecord<String, String, String>> claimed =
                    ops.claim(STREAM, GROUP, consumerName, MIN_IDLE, pm.getId());
            if (claimed.isEmpty()) {
                continue; // 다른 인스턴스가 먼저 가져갔거나, 트리밍으로 이미 삭제됨
            }
            MapRecord<String, String, String> record = claimed.get(0);

            if (pm.getTotalDeliveryCount() >= MAX_DELIVERY) {
                // poison message → DLQ로 옮기고 원본은 ACK
                ops.add(StreamRecords.newRecord().in(DLQ_STREAM).ofMap(record.getValue()));
                ops.acknowledge(STREAM, GROUP, record.getId());
                log.error("moved to DLQ. id={}", record.getId());
                continue;
            }
            listener.onMessage(record);
        }
    }
}
```

`XCLAIM`은 `MIN_IDLE` 조건을 Redis 쪽에서 다시 검사하기 때문에, 여러 인스턴스가 동시에
reclaim을 돌려도 같은 메시지를 둘이 동시에 가져가지 않는다 (먼저 claim한 쪽이 idle 시간을
0으로 리셋하므로 뒤의 claim은 빈 결과를 받는다).

## 3. 한눈에 비교

| 항목 | Pub/Sub | Stream | (참고) Kafka |
|---|---|---|---|
| 저장 | X | O (메모리, MAXLEN으로 제한) | O (디스크, retention) |
| 전달 보장 | At-most-once | At-least-once (ACK + PEL) | At-least-once / Exactly-once 옵션 |
| 늦게 붙은 소비자 | 과거 메시지 못 받음 | ID 지정으로 과거부터 읽기 가능 | offset 지정으로 재처리 가능 |
| 부하 분산 | X (모두에게 방송) | Consumer Group | Consumer Group + Partition |
| 순서 | 채널 내 순서 | 스트림 내 순서 | 파티션 내 순서 |
| 처리량/보관 규모 | 매우 빠름 | 메모리 크기가 한계 | 디스크 기반이라 대용량/장기 보관 |
| 운영 부담 | Redis만 있으면 됨 | Redis만 있으면 됨 | 별도 클러스터 운영 |

### 무엇을 고를까

- **놓쳐도 되는 실시간 신호** (캐시 무효화, 실시간 브로드캐스트, 리로드 신호) → **Pub/Sub**
- **놓치면 안 되는 작업/이벤트이고, 규모가 Redis 메모리로 감당되는 수준** → **Stream**
- **대용량, 장기 보관, 재처리(replay)가 중요, 여러 팀이 이벤트를 공유** → **Kafka**

실무에서는 둘을 섞어 쓰는 경우도 많다. 예를 들어 채팅은 메시지를 Stream(또는 DB)에 저장해서
유실을 막고, 다른 서버에 붙은 사용자에게 "새 메시지 왔다"는 실시간 전달만 Pub/Sub으로 한다.
Pub/Sub 신호를 놓친 클라이언트는 재접속 시 Stream에서 마지막으로 받은 ID 이후를 읽어오면 된다.

## 정리

- Pub/Sub은 **방송**이다. 빠르고 단순하지만 저장도 ACK도 없어서 "유실돼도 괜찮은" 곳에만 쓴다.
- Stream은 **로그 + 소비 추적**이다. Consumer Group, PEL, `XACK`/`XAUTOCLAIM`으로 At-least-once를
  보장하지만, 그만큼 **멱등 처리, 트리밍, 재처리/DLQ 로직은 직접 챙겨야** 한다.
- Spring에서는 Pub/Sub은 `RedisMessageListenerContainer`, Stream은
  `StreamMessageListenerContainer`로 다룬다. 특히 Stream 컨테이너의 `cancelOnError` 기본 동작
  (에러 시 구독 취소)은 운영에서 "리스너가 조용히 멈추는" 원인이 되기 쉬우니 꼭 확인하자.
