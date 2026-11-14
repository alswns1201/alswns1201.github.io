---
title: "SSE로 브라우저 팝업에 처리 결과 알려 주기 — 내부 웹훅, 이벤트 이름 계약, 커밋 후 발송, 서버 여러 대"
date: 2026-11-14 10:00:00 +0900
categories: [개발 고민/설계]
---
{% raw %}

PC 화면에 QR을 띄우고, 사용자는 **휴대폰 앱**으로 QR을 찍어 인증이나 결제를 진행하는 흐름이 있다. QR 로그인, QR 결제가 대표적이다.
이때 PC 화면은 앱에서 무슨 일이 일어났는지 스스로 알 수 없다. 누군가 "끝났어, 결과 화면으로 넘어가"라고 알려 줘야 한다.

이 글은 그 "알려 주기"를 **SSE(Server-Sent Events)** 로 구성할 때의 구조와, 실제로 잘 터지는 지점 네 가지를 정리한다.

> 글에 나오는 서비스·경로·이벤트 이름은 모두 예시다. 코드는 구조를 보여 주기 위한 예시 코드다.

## 선택지: 폴링, 롱 폴링, WebSocket, SSE

| 방식 | 동작 | 장점 | 단점 |
|---|---|---|---|
| 폴링 | 브라우저가 N초마다 상태 조회 | 가장 단순, 서버 상태 없음 | 간격만큼 늦고, 대부분의 요청이 "변화 없음" |
| 롱 폴링 | 변화가 생길 때까지 응답을 잡아 둠 | 폴링보다 빠름 | 응답마다 다시 연결, 구현이 의외로 번거롭다 |
| WebSocket | 양방향 연결 | 양방향·저지연 | 별도 프로토콜, 프록시·로드밸런서 설정, 이 용도엔 과함 |
| **SSE** | 서버 → 브라우저 단방향 스트림 (`text/event-stream`) | 그냥 HTTP, 브라우저 `EventSource`가 **자동 재연결** | 단방향, 텍스트만, HTTP/1.1에서는 도메인당 연결 수 제한 |

팝업이 할 일은 "서버가 신호를 주면 상태를 다시 본다"뿐이다. 브라우저에서 서버로 보낼 것은 일반 HTTP 요청으로 충분하니 양방향이 필요 없다. 그래서 SSE가 잘 맞는다.

## 구조: 외부에 열린 서버가 연결을 잡고, 내부 서버는 웹훅으로 신호만 보낸다

```
[휴대폰 앱] ──(인증·결제 요청)──▶ [내부 API 서버] ── 상태 변경 + 커밋
                                          │
                                          │ ① 내부 웹훅: "거래 T 상태 바뀜" (시크릿 헤더)
                                          ▼
[PC 브라우저 팝업] ◀──② SSE 이벤트── [외부용 게이트웨이 서버]
        │
        └──③ 상태 조회 (거래 전용 토큰) ──▶ 게이트웨이 ──▶ 내부 API
```

이렇게 나누는 이유는 두 가지다.

1. **내부 API 서버를 브라우저에 직접 노출하지 않는다.** 팝업은 외부 사용자의 브라우저에서 돈다. 브라우저가 오래 붙잡는 연결(SSE)을 내부 서버가 직접 받으면, 내부 서버를 외부에 열어야 하고 연결 관리 부담도 내부 서버가 진다. 외부 진입점인 게이트웨이가 연결을 맡고, 내부 서버는 서버 간 호출(웹훅)로 신호만 보낸다.
2. **이벤트에는 데이터를 싣지 않고 "다시 조회하라"는 신호만 보낸다 (notify-then-fetch).**
   - 이벤트를 하나 놓치거나 순서가 바뀌어도, 다음 조회가 **항상 최신 상태**를 가져오므로 결과가 틀리지 않는다.
   - 권한 검사는 조회 API 한 곳에서만 하면 된다. 이벤트에 금액이나 개인정보를 실으면 SSE 채널까지 보호 대상이 된다.
   - 이벤트 형식이 바뀌어도 조회 API 응답만 지키면 된다.

### 게이트웨이: 연결 보관과 이벤트 전달 (Spring MVC)

```java
@RestController
@RequiredArgsConstructor
public class PopupEventController {

    private final EmitterRegistry registry;

    // ② 팝업이 연결하는 SSE 엔드포인트 (거래 전용 토큰으로 인증된 요청만 들어온다고 가정)
    @GetMapping(value = "/popup/transactions/{txId}/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribe(@PathVariable String txId) {
        SseEmitter emitter = new SseEmitter(Duration.ofMinutes(10).toMillis());
        registry.add(txId, emitter);
        emitter.onCompletion(() -> registry.remove(txId, emitter));
        emitter.onTimeout(() -> registry.remove(txId, emitter));
        emitter.onError(e -> registry.remove(txId, emitter));
        return emitter;
    }
}

@RestController
@RequiredArgsConstructor
class InternalWebhookController {

    private final EmitterRegistry registry;
    private final WebhookProperties props;

    // ① 내부 API 서버가 부르는 웹훅
    @PostMapping("/internal/webhook/events")
    public ResponseEntity<Void> receive(@RequestHeader(value = "X-Webhook-Secret", required = false) String secret,
                                        @RequestBody WebhookEvent event) {
        // 설정이 비어 있으면 통과시키는 게 아니라 거부한다 (fail-closed)
        if (!StringUtils.hasText(props.secret()) || !MessageDigest.isEqual(
                props.secret().getBytes(UTF_8), String.valueOf(secret).getBytes(UTF_8))) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        registry.send(event.txId(), event.type());   // 이벤트 이름 = type
        return ResponseEntity.accepted().build();
    }
}
```

```java
@Component
class EmitterRegistry {

    private final Map<String, Set<SseEmitter>> emitters = new ConcurrentHashMap<>();

    void add(String txId, SseEmitter e) {
        emitters.computeIfAbsent(txId, k -> ConcurrentHashMap.newKeySet()).add(e);
    }

    void remove(String txId, SseEmitter e) {
        emitters.computeIfPresent(txId, (k, set) -> { set.remove(e); return set.isEmpty() ? null : set; });
    }

    void send(String txId, String eventName) {
        for (SseEmitter e : emitters.getOrDefault(txId, Set.of())) {
            try {
                e.send(SseEmitter.event().name(eventName).data("{}"));
            } catch (IOException ex) {
                remove(txId, e);   // 끊긴 연결 정리
            }
        }
    }
}
```

웹훅 경로는 **로드밸런서나 WAF에서 외부 접근 자체를 막는 것**까지 같이 해야 한다. 시크릿 검사는 두 번째 방어선이다.

### 팝업: 이벤트를 받으면 조회한다

```javascript
const es = new EventSource(`/popup/transactions/${txId}/events`, { withCredentials: true });

es.addEventListener('STATUS_CHANGED', () => refreshStatus());

// 연결(재연결 포함) 직후 한 번은 무조건 조회 — 끊겨 있던 동안의 이벤트를 놓쳤을 수 있다
es.onopen = () => refreshStatus();

async function refreshStatus() {
  const res = await fetch(`/popup/transactions/${txId}/status`, { credentials: 'include' });
  const { status } = await res.json();
  if (status === 'DONE' || status === 'FAILED') {
    es.close();
    location.replace(`/popup/result?tx=${txId}`);
  }
}
```

## 터지는 지점 1: 이벤트 이름 계약 — 틀리면 **아무 에러 없이** 무시된다

SSE 이벤트는 `event:` 필드로 이름을 가진다.

```
event: STATUS_CHANGED
data: {}

```

브라우저 쪽 규칙이 중요하다.

- `es.onmessage`는 **이름이 없는 이벤트(기본 `message`)만** 받는다. 이름을 붙여 보내면 `onmessage`로는 안 온다.
- `addEventListener('STATUS_CHANGED', ...)`는 **이름이 정확히 같을 때만** 호출된다.
- 이름이 다르면 브라우저는 그냥 버린다. 콘솔 에러도, 서버 에러도 없다.

그래서 보내는 쪽이 `TX_STATUS`로 보내고 팝업이 `STATUS_CHANGED`만 듣고 있으면, **연결도 잘 되어 있고 웹훅도 200인데 화면만 멈춰 있는** 장애가 된다. 중간 서버가 내부 서버의 `type` 값을 그대로 이벤트 이름으로 쓰는 구조라면, 내부 서버에서 enum 하나 이름을 바꾼 것만으로 이런 일이 생긴다.

더 나쁜 경우는 **다음 단계가 조회에 걸려 있을 때**다. "팝업이 상태를 조회하는 시점에 다음 처리(예: 최종 승인)를 진행한다"는 구조라면, 조회가 안 일어나니 처리 자체가 멈춘다.

대응:

- 이벤트 이름을 **한 곳에서 정의**하고(공유 상수나 계약 문서), 양쪽이 그걸 쓴다.
- 중간 서버가 내부 값을 그대로 넘기지 말고, **허용된 이벤트 이름 목록**으로 매핑한다. 목록에 없는 값이 오면 경고 로그를 남긴다.
- "이 이벤트를 보내면 팝업이 조회를 한다"를 확인하는 **E2E 테스트**를 하나 둔다. 단위 테스트로는 양쪽이 각자 맞다고 통과해 버린다.

## 터지는 지점 2: 커밋 전에 보내면 팝업이 옛날 상태를 본다

상태를 바꾸는 트랜잭션 안에서 바로 이벤트를 보내면, 팝업이 조회할 때 **아직 커밋되지 않아** 이전 상태가 보일 수 있다. 팝업은 "변화 없음"으로 판단하고 다음 이벤트를 기다리는데, 다음 이벤트는 오지 않는다.

```java
@Transactional
public void fail(String txId, FailReason reason) {
    tx.markFailed(reason);
    events.publishEvent(new StatusChanged(txId));   // 여기서 바로 웹훅을 부르지 않는다
}

@Component
class StatusChangedNotifier {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(StatusChanged e) {
        webhookClient.notify(e.txId(), "STATUS_CHANGED");   // 커밋이 끝난 뒤에만 나간다
    }
}
```

- `AFTER_COMMIT`이면 롤백된 변경에 대해서는 이벤트가 나가지 않는다는 장점도 있다.
- 웹훅 호출이 느리면 요청 스레드를 붙잡으니 `@Async`나 별도 실행기로 넘기는 것도 고려한다. 그때는 **실패가 로그에만 남지 않도록** 알림이나 재시도를 붙인다. 이벤트 전달 실패는 사용자 화면이 멈추는 문제라서 조용히 묻히면 안 된다.

그리고 **상태가 바뀌는 모든 경로**에서 이벤트를 보내는지 확인한다. 성공 경로만 챙기고 중간 단계 실패(외부 심사 거절, 인증 실패 등)에서 이벤트를 빼먹으면, 실패했는데 팝업은 계속 대기 화면이다. 상태 전이를 한 메서드로 모으고 그 안에서 이벤트를 발행하면 빠뜨리기 어렵다.

## 터지는 지점 3: 서버가 여러 대면 웹훅이 "연결이 없는 서버"에 도착한다

`SseEmitter`는 **그 서버의 메모리**에 있다. 게이트웨이가 두 대라면,

```
팝업 SSE 연결 ──▶ 게이트웨이 A (emitter 보관)
내부 웹훅     ──▶ 게이트웨이 B (emitter 없음) → 아무 일도 안 일어남
```

로드밸런서가 웹훅을 어느 서버로 보낼지 모르니 확률적으로 이벤트가 사라진다. 한 대일 때는 절대 재현되지 않는다는 점이 특히 고약하다.

해결책은 보통 두 가지다.

| 방법 | 내용 | 주의 |
|---|---|---|
| **브로커로 전체에 뿌리기** | 웹훅을 받은 서버가 Redis Pub/Sub 등에 발행 → 모든 게이트웨이가 구독해서 자기한테 있는 emitter에만 전송 | 구독자가 없을 때 메시지는 사라진다(Pub/Sub). notify-then-fetch라 괜찮다 |
| 스티키 세션 | 같은 거래의 요청을 같은 서버로 보냄 | 웹훅은 서버 간 호출이라 브라우저 쿠키 기반 스티키가 안 먹는다. 배포·스케일 인 때 깨진다 |

```java
// 웹훅 수신: registry.send(...)로 직접 보내지 않고 발행만 한다
redisTemplate.convertAndSend("popup-events", event.txId() + "|" + event.type());

// 모든 게이트웨이 인스턴스가 구독
@Bean
RedisMessageListenerContainer container(RedisConnectionFactory cf, EmitterRegistry registry) {
    var c = new RedisMessageListenerContainer();
    c.setConnectionFactory(cf);
    c.addMessageListener((msg, pattern) -> {
        String[] p = new String(msg.getBody(), UTF_8).split("\\|", 2);
        registry.send(p[0], p[1]);           // 내 메모리에 연결이 있을 때만 실제로 전송됨
    }, new ChannelTopic("popup-events"));
    return c;
}
```

Pub/Sub이 메시지를 저장하지 않는다는 점은 [Redis Pub/Sub과 Stream 비교](/posts/redis-pubsub-vs-stream/)에 정리했다. 여기서는 이벤트가 "조회 신호"일 뿐이고 팝업이 재연결 시 한 번 조회하므로, 유실을 감수할 수 있어 Pub/Sub으로 충분하다.

## 터지는 지점 4: 중간 장비와 브라우저 제약

- **유휴 연결 타임아웃.** 로드밸런서·프록시는 일정 시간 데이터가 없으면 연결을 끊는다. 주기적으로 주석 줄(`: ping`)을 보내 연결을 살려 둔다. `EventSource`는 끊기면 재연결하므로, 재연결 직후 조회(`onopen`)가 있으면 사용자에게는 티가 나지 않는다.
- **프록시 버퍼링.** Nginx 같은 프록시가 응답을 모았다가 보내면 이벤트가 늦게 도착한다. `X-Accel-Buffering: no` 응답 헤더나 `proxy_buffering off`를 쓴다.
- **연결 수.** HTTP/1.1에서 브라우저는 도메인당 동시 연결이 6개 정도로 제한된다. 같은 도메인에 탭을 여러 개 열면 SSE가 연결을 다 차지할 수 있다. HTTP/2면 한 연결에 여러 스트림을 싣기 때문에 이 문제가 사라진다.
- **서버 자원.** Servlet 비동기 요청이라 연결 하나가 스레드를 계속 잡지는 않지만, emitter는 메모리에 남는다. 타임아웃을 거래 유효 시간에 맞추고, 결과 화면으로 넘어가면 팝업이 `close()`한다.
- **안전망.** 그래도 이벤트가 끝까지 안 올 수 있다. 팝업에 **느린 폴링(예: 수십 초 간격)** 을 함께 두면, SSE가 실패해도 "한참 늦게라도 넘어가는" 상태가 된다.

## 정리

> 다른 기기에서 일어난 일을 브라우저 팝업에 알려야 할 때 SSE는 단순하고 잘 맞는다. 외부 진입 서버가 SSE 연결을 잡고, 내부 서버는 웹훅으로 "다시 조회하라"는 신호만 보내면 내부 서버를 노출하지 않으면서 유실에도 강한 구조가 된다.
> 다만 **이벤트 이름이 틀리면 조용히 무시되고**, **커밋 전에 보내면 옛날 상태를 보고**, **서버가 여러 대면 연결 없는 서버로 신호가 가고**, **실패 경로에서 빼먹으면 대기 화면에 갇힌다**. 이름은 한 곳에서 정의하고, `AFTER_COMMIT`으로 보내고, 브로커로 모든 인스턴스에 뿌리고, 재연결·느린 폴링으로 마지막 안전망을 둔다.

## 참고

- [MDN — Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [HTML Living Standard — Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [Spring Framework — SseEmitter](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html#mvc-ann-async-sse)
- [Spring Framework — Transaction-bound Events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)
{% endraw %}
