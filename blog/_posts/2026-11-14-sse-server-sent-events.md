---
title: "SSE(Server-Sent Events) 정리 — 프로토콜, EventSource, WebSocket과의 차이, 운영에서 조심할 점"
date: 2026-11-14 10:00:00 +0900
categories: [네트워크]
redirect_from:
  - /posts/sse-popup-status-push/
---
{% raw %}

서버에서 브라우저로 "무언가 바뀌었다"를 바로 알려 줘야 할 때가 있다. 알림 배지, 진행률, 주문 상태, 다른 기기에서 끝난 인증 결과 같은 것들이다.
HTTP는 기본적으로 **클라이언트가 물어야 서버가 답하는** 구조라서, 서버가 먼저 말을 거는 방법이 따로 필요하다. 그중 가장 단순한 표준이 **SSE(Server-Sent Events)** 다.

이 글은 SSE가 무엇이고 어떻게 동작하는지, WebSocket과는 무엇이 다른지, 실제로 쓸 때 어디서 막히는지를 정리한다.

> 코드는 개념을 보여 주기 위한 예시 코드이고, 경로·이벤트 이름은 모두 예시다.

## 서버가 먼저 알려 주는 방법들

| 방식 | 동작 | 장점 | 단점 |
|---|---|---|---|
| 폴링 | 클라이언트가 N초마다 요청 | 가장 단순 | 간격만큼 늦고, 대부분의 요청이 "변화 없음" |
| 롱 폴링 | 변화가 생길 때까지 서버가 응답을 붙잡음 | 폴링보다 빠름 | 응답마다 다시 연결, 타임아웃·중복 처리가 번거롭다 |
| **SSE** | 응답 하나를 끝내지 않고 이벤트를 계속 흘려보냄 | 그냥 HTTP, 브라우저가 **자동 재연결** | 서버 → 클라이언트 **단방향**, 텍스트만 |
| WebSocket | 별도 프로토콜로 업그레이드한 양방향 연결 | 양방향·저지연·바이너리 | 프로토콜·인프라 설정이 더 필요 |

SSE는 "요청 1번, 응답은 끝나지 않는 스트림"이다. 새로운 프로토콜이 아니라 **HTTP 응답을 길게 유지하면서 정해진 텍스트 형식으로 조금씩 쓰는 것**뿐이다.

## 프로토콜: `text/event-stream`

서버는 응답 헤더를 이렇게 보낸다.

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

본문은 줄 단위 텍스트다. **빈 줄 하나가 이벤트 하나의 끝**이다.

```
: 주석. 클라이언트는 무시한다 (연결 유지용 heartbeat로 쓴다)

event: order-updated
id: 1024
data: {"orderId": 7, "status": "SHIPPED"}

data: 이름 없는 이벤트
data: data 줄이 여러 개면 줄바꿈으로 이어 붙는다

retry: 5000

```

| 필드 | 의미 |
|---|---|
| `event` | 이벤트 이름. 없으면 기본 이름 `message` |
| `data` | 내용. 여러 줄이면 `\n`으로 합쳐진다 |
| `id` | 이벤트 ID. 브라우저가 기억했다가 재연결 때 `Last-Event-ID` 헤더로 보낸다 |
| `retry` | 재연결 대기 시간(ms) |
| `:`로 시작 | 주석. 무시된다 |

인코딩은 항상 UTF-8이고, 바이너리는 보낼 수 없다(필요하면 Base64 등으로 텍스트화한다).

## 브라우저: `EventSource`

```javascript
const es = new EventSource('/orders/7/events', { withCredentials: true });

es.onopen = () => console.log('연결됨');                 // 재연결 때도 다시 호출된다
es.onmessage = (e) => console.log('이름 없는 이벤트', e.data);
es.addEventListener('order-updated', (e) => {
  const payload = JSON.parse(e.data);
  console.log(payload.status, e.lastEventId);
});
es.onerror = () => console.log('끊김, 브라우저가 알아서 재연결 시도');

// 더 이상 필요 없으면 직접 닫는다 (닫지 않으면 계속 재연결한다)
// es.close();
```

여기서 가장 많이 헷갈리는 규칙이 있다.

- `onmessage`는 **이름이 없는 이벤트**(`message`)만 받는다. 서버가 `event: order-updated`로 보내면 `onmessage`에는 오지 않는다.
- `addEventListener('order-updated')`는 **이름이 정확히 같을 때만** 호출된다.
- 이름이 맞지 않는 이벤트는 **에러 없이 버려진다.** 콘솔에도 아무것도 안 찍힌다.

그 밖에 알아 둘 점:

- **자동 재연결.** 연결이 끊기면 브라우저가 `retry` 간격으로 다시 연결한다. 마지막으로 받은 `id`가 있으면 `Last-Event-ID` 헤더에 실어 보내므로, 서버가 그 뒤의 이벤트만 다시 보낼 수 있다.
- **요청 헤더를 직접 붙일 수 없다.** `EventSource`는 `Authorization` 같은 커스텀 헤더를 지원하지 않는다. 인증은 보통 쿠키(`withCredentials`)로 하고, 쿼리 파라미터에 토큰을 넣는다면 접속 로그에 남지 않도록 마스킹한다. 헤더가 꼭 필요하면 `fetch` + `ReadableStream`으로 직접 파싱하는 방법도 있다.
- **GET만 된다.** 본문을 보내는 요청으로는 열 수 없다.
- 서버가 `204 No Content`로 응답하면 브라우저는 재연결을 멈춘다. "더 보낼 게 없다"를 알리는 방법으로 쓸 수 있다.

## WebSocket과 무엇이 다른가

| | SSE | WebSocket |
|---|---|---|
| 방향 | 서버 → 클라이언트 | 양방향 |
| 프로토콜 | 일반 HTTP 응답 | `Upgrade`로 전환한 별도 프로토콜 |
| 데이터 | UTF-8 텍스트 | 텍스트·바이너리 |
| 재연결 | 브라우저 내장 (`Last-Event-ID`) | 직접 구현 |
| 인프라 | 기존 HTTP 경로·인증·로드밸런서를 그대로 사용 | 업그레이드 지원, 유휴 타임아웃 설정 필요 |
| 어울리는 곳 | 알림, 진행률, 상태 변경, 로그 tail, LLM 응답 스트리밍 | 채팅, 게임, 협업 편집처럼 양쪽이 자주 말하는 곳 |

클라이언트가 서버에 보낼 일이 가끔이면 그건 **일반 HTTP 요청으로** 보내면 된다. "서버가 알리고, 클라이언트는 필요할 때 요청한다"로 정리되면 SSE로 충분하다.

## 서버 구현 (Spring)

### Spring MVC: `SseEmitter`

```java
@RestController
@RequiredArgsConstructor
public class OrderEventController {

    private final EmitterRegistry registry;   // orderId → 연결 목록 (메모리)

    @GetMapping(value = "/orders/{orderId}/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribe(@PathVariable Long orderId) {
        SseEmitter emitter = new SseEmitter(Duration.ofMinutes(10).toMillis());
        registry.add(orderId, emitter);
        emitter.onCompletion(() -> registry.remove(orderId, emitter));
        emitter.onTimeout(() -> registry.remove(orderId, emitter));
        emitter.onError(e -> registry.remove(orderId, emitter));
        return emitter;
    }
}

// 상태가 바뀌는 곳에서
registry.forEach(orderId, emitter -> emitter.send(
        SseEmitter.event().name("order-updated").id(String.valueOf(eventId)).data(payload)));
```

- 컨트롤러가 `SseEmitter`를 반환하면 요청 스레드는 바로 반환된다(Servlet 비동기). 연결 하나가 스레드를 계속 잡지는 않는다.
- 보내다가 `IOException`이 나면 클라이언트가 떠난 것이니 목록에서 지운다. 지우지 않으면 메모리에 쌓인다.

### WebFlux: `Flux<ServerSentEvent>`

```java
@GetMapping(value = "/orders/{orderId}/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> subscribe(@PathVariable Long orderId) {
    return orderEvents.stream(orderId)
            .map(e -> ServerSentEvent.builder(e.json()).event("order-updated").id(e.id()).build());
}
```

두 방식의 차이(직접 밀기 vs 흐름 정의, 백프레셔)는 [StreamingResponseBody vs WebFlux 글](/posts/streaming-response-body-vs-webflux/)에 정리했다.

## 이벤트에 무엇을 실을까: 데이터 vs "다시 조회하라" 신호

이벤트에 바뀐 데이터를 통째로 실을 수도 있고, **"바뀌었으니 다시 조회하라"는 신호만** 보낼 수도 있다(notify-then-fetch).

| | 데이터를 싣기 | 신호만 보내기 |
|---|---|---|
| 요청 수 | 적다 | 이벤트마다 조회 1번 |
| 이벤트 유실·순서 뒤바뀜 | 화면이 틀릴 수 있다 | 다음 조회가 항상 최신이라 틀리지 않는다 |
| 권한·민감정보 | SSE 채널도 보호 대상 | 조회 API에서만 검사 |
| 형식 변경 | 이벤트 형식까지 맞춰야 함 | 조회 API만 유지 |

이벤트가 자주 오고 내용이 작으면(진행률, 로그) 데이터를 싣고, **상태가 중요하고 이벤트가 드물면**(결제·인증 결과) 신호만 보내는 쪽이 안전하다.

## 운영에서 조심할 점

1. **프록시 버퍼링.** Nginx 같은 프록시가 응답을 모았다가 보내면 이벤트가 늦게 도착한다. `X-Accel-Buffering: no` 응답 헤더나 `proxy_buffering off`로 끈다. 압축(gzip)도 버퍼링을 일으킬 수 있다.
2. **유휴 타임아웃.** 로드밸런서·프록시는 일정 시간 데이터가 없으면 연결을 끊는다. 그보다 짧은 간격으로 주석 줄(`: ping`)을 보낸다.
3. **연결 수 제한.** HTTP/1.1에서 브라우저는 도메인당 동시 연결이 6개 정도다. 탭을 여러 개 열면 SSE가 연결을 다 차지해 다른 요청이 막힐 수 있다. HTTP/2에서는 한 연결에 여러 스트림이 실려 사라지는 문제다.
4. **서버가 여러 대일 때.** 연결 목록은 **그 서버의 메모리**에 있다. 이벤트가 연결이 없는 서버에서 발생하면 아무에게도 안 간다. Redis Pub/Sub 같은 브로커로 모든 서버에 뿌리고, 각 서버는 자기에게 있는 연결에만 보낸다([Redis Pub/Sub과 Stream 비교](/posts/redis-pubsub-vs-stream/)).
5. **커밋 전에 보내지 않는다.** 트랜잭션 안에서 이벤트를 보내면, 클라이언트가 조회할 때 아직 커밋 전이라 옛날 상태를 볼 수 있다. `@TransactionalEventListener(phase = AFTER_COMMIT)`로 커밋 뒤에 보낸다.
6. **놓친 이벤트 대비.** 재연결 사이에 이벤트가 지나갈 수 있다. `id`와 `Last-Event-ID`로 다시 보내 주거나, 연결(`onopen`) 직후 한 번 조회한다. 정말 중요한 화면이면 느린 폴링을 안전망으로 같이 둔다.

## 실무 예시: 다른 기기에서 끝난 일을 브라우저 화면에 알리기

PC 화면에 QR을 띄우고 사용자가 **휴대폰 앱**으로 인증이나 결제를 끝내는 흐름이 있다. PC 화면은 앱에서 무슨 일이 일어났는지 모르니, 서버가 알려 줘야 한다.

```
[앱] → 내부 서버: 처리 완료, 커밋
내부 서버 → 외부용 서버: 내부 웹훅 "거래 T 바뀜"
외부용 서버 → [PC 화면]: SSE 이벤트
[PC 화면] → 상태 조회 → 결과 화면으로 이동
```

내부 서버를 브라우저에 직접 노출하지 않으려고 외부용 서버가 SSE 연결을 맡고, 이벤트는 "다시 조회하라" 신호만 보내는 구조다.
이런 구조에서 실제로 겪기 쉬운 문제가 앞의 이름 규칙이다. **보내는 쪽 이벤트 이름과 화면이 듣는 이름이 어긋나면**, 연결도 정상이고 서버 로그도 깨끗한데 화면만 대기 상태에 멈춘다. 다음 처리가 "화면의 조회"에 걸려 있으면 처리 자체가 멈춘다. 그래서 이벤트 이름은 한 곳에서 정의해 양쪽이 같이 쓰고, 성공뿐 아니라 **실패로 끝나는 경로에서도 이벤트를 보내는지** 확인해야 한다.

## 정리

> SSE는 **HTTP 응답 하나를 열어 둔 채 `text/event-stream` 형식으로 이벤트를 흘려보내는** 단방향 푸시다. 브라우저 `EventSource`가 재연결과 `Last-Event-ID`를 알아서 처리해 주고, 기존 HTTP 인프라를 그대로 쓸 수 있다.
> 이름 있는 이벤트는 `addEventListener`로만 받고 이름이 틀리면 조용히 버려진다는 점, 커스텀 헤더를 못 붙인다는 점, 프록시 버퍼링·유휴 타임아웃·연결 수 제한·다중 서버를 조심하면 된다. 양방향이 꼭 필요한 게 아니라면 WebSocket보다 먼저 검토할 만하다.

## 참고

- [MDN — Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [MDN — EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)
- [HTML Living Standard — Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [Spring Framework — SSE (Spring MVC)](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html#mvc-ann-async-sse)
- [Spring Framework — Transaction-bound Events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)
{% endraw %}
