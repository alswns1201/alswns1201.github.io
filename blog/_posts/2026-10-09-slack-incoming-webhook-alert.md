---
title: "Spring에서 Slack 채널로 알림 보내기 — Incoming Webhook"
date: 2026-10-09
categories: [Java/Spring]
---

결제처럼 중요한 로직이 실패하면 Slack 채널로 알림을 보내고 싶었다.
Slack 연동 글을 찾아보면 Bot 토큰, Bolt 프레임워크, 슬래시 커맨드까지 나와서 복잡해 보이는데,
**정해진 채널에 알림만 보내는 거라면 Incoming Webhook 하나면 충분하다.**

## 방법 고르기

| 방식 | 필요한 것 | 언제 |
|---|---|---|
| **Incoming Webhook** | 웹훅 URL 하나 | 정해진 채널에 알림만 보낼 때 ← 이번 글 |
| Bot 토큰 + `chat.postMessage` | `xoxb-` 토큰, `chat:write` 권한 | 채널을 코드에서 골라 보내거나, DM·메시지 수정/삭제가 필요할 때 |
| Bolt 프레임워크 | 토큰 + 이벤트 수신 서버 | 슬래시 커맨드, 버튼 클릭처럼 Slack에서 우리 서버로 요청이 들어올 때 |

웹훅은 URL 하나로 채널 하나에 메시지를 보낸다. 대신 보낼 채널은 URL을 만들 때 정해지고, 코드에서 바꿀 수 없다.
채널이 여러 개면 웹훅 URL을 채널마다 하나씩 만들면 된다.

## 1. Slack 앱 만들고 웹훅 URL 받기

1. [https://api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From scratch**
2. 앱 이름(예: `order-alert`)과 워크스페이스 선택 → **Create App**
3. 왼쪽 메뉴 **Incoming Webhooks** → **Activate Incoming Webhooks**를 On
4. 아래 **Add New Webhook** → 알림 받을 채널 선택 → **허용(Allow)**
5. 생성된 URL 복사

```
https://hooks.slack.com/services/T0000000/B0000000/XXXXXXXXXXXXXXXXXXXXXXXX
```

> 이 URL만 있으면 **누구나 그 채널에 메시지를 보낼 수 있다.** 비밀번호처럼 다뤄야 한다.
> 코드나 Git에 그대로 넣지 말고 환경변수로 뺀다.

## 2. curl로 먼저 확인

코드를 짜기 전에 URL이 동작하는지 터미널에서 확인한다.

```bash
curl -X POST -H 'Content-Type: application/json' \
  -d '{"text":"안녕하세요, 테스트 알림입니다."}' \
  https://hooks.slack.com/services/T0000000/B0000000/XXXXXXXX
```

성공하면 응답으로 `ok`가 오고, 채널에 메시지가 뜬다.
결국 하는 일은 **JSON 하나를 POST로 보내는 것**이 전부다. Spring에서도 이걸 그대로 하면 된다.

## 3. Spring에서 보내기

별도 Slack 라이브러리 없이 Spring의 `RestClient`(Spring Boot 3.2+)로 보낸다.

### 설정

```yaml
# application.yml
slack:
  webhook-url: ${SLACK_WEBHOOK_URL}   # 실제 값은 환경변수로
```

### SlackNotifier

```java
@Slf4j
@Component
public class SlackNotifier {

	private final RestClient restClient;
	private final String webhookUrl;

	public SlackNotifier(RestClient.Builder builder,
						 @Value("${slack.webhook-url}") String webhookUrl) {
		this.restClient = builder.build();
		this.webhookUrl = webhookUrl;
	}

	public void send(String text) {
		try {
			restClient.post()
					.uri(webhookUrl)
					.contentType(MediaType.APPLICATION_JSON)
					.body(Map.of("text", text))
					.retrieve()
					.toBodilessEntity();
		} catch (Exception e) {
			log.warn("Slack 알림 전송 실패: {}", e.getMessage());   // 알림 실패로 본 로직이 깨지면 안 됨
		}
	}
}
```

- 보내는 JSON은 `{"text": "..."}` 하나다.
- **알림이 실패해도 예외를 밖으로 던지지 않는다.** Slack이 잠깐 안 된다고 주문이 실패하면 안 되니까, 로그만 남긴다.

## 4. 실제 로직에서 쓰기 — 실패했을 때만 알림

알림은 **결제가 실패했을 때만** 보낸다. 성공 알림까지 보내면 채널이 금방 묻혀서 정작 실패를 놓친다.

```java
@Service
@RequiredArgsConstructor
public class PaymentFacade {

	private final PaymentService paymentService;   // @Transactional 로직
	private final SlackNotifier slackNotifier;

	public void pay(Long orderId, long amount) {
		try {
			paymentService.pay(orderId, amount);
		} catch (Exception e) {
			slackNotifier.send("🚨 결제 실패: 주문 %d, 금액 %,d원, 사유 %s"
					.formatted(orderId, amount, e.getMessage()));
			throw e;                                   // 실패 처리는 원래대로 진행
		}
	}
}
```

- **성공하면 아무것도 안 보낸다.** catch에 들어왔을 때만 알림이 나간다.
- 알림을 보낸 뒤 **예외는 다시 던진다.** 알림은 "알려 주는 것"일 뿐, 실패 응답·롤백 같은 원래 처리를 바꾸면 안 된다.
- try/catch를 **트랜잭션 밖(Facade)**에 둔다. `@Transactional` 메서드 안에서 잡으면 커밋 단계에서 나는 실패(DB 제약 조건 위반 등)는 못 잡는다.
  밖에서 잡으면 트랜잭션이 롤백까지 끝난, **최종적으로 실패한 경우**만 알림이 간다.
- 비동기로 빼지 않고 그냥 보낸다. 실패할 때만 호출되니 Slack 응답을 기다리는 시간은 실패한 요청에만 붙는다.
  대신 Slack이 응답하지 않을 때 오래 붙잡히지 않도록 **타임아웃은 짧게** 걸어 둔다.

```java
public SlackNotifier(RestClient.Builder builder,
					 @Value("${slack.webhook-url}") String webhookUrl) {
	SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
	factory.setConnectTimeout(Duration.ofSeconds(2));
	factory.setReadTimeout(Duration.ofSeconds(3));

	this.restClient = builder.requestFactory(factory).build();
	this.webhookUrl = webhookUrl;
}
```

## 5. 메시지 꾸미기 (선택)

`text`만으로도 충분하지만, 알림이 많아지면 **Block Kit**으로 보기 좋게 만들 수 있다.

```json
{
  "text": "결제 실패 알림",
  "blocks": [
    { "type": "header", "text": { "type": "plain_text", "text": "🚨 결제 실패" } },
    { "type": "section", "fields": [
        { "type": "mrkdwn", "text": "*주문번호*\n12345" },
        { "type": "mrkdwn", "text": "*금액*\n15,000원" }
    ]}
  ]
}
```

- `blocks`를 쓸 때도 `text`는 같이 넣는다. 모바일 푸시 알림 문구로 쓰인다.
- 레이아웃은 [Block Kit Builder](https://app.slack.com/block-kit-builder)에서 눈으로 보면서 만들고 JSON을 복사해 오면 편하다.
- 자바에서는 `Map`/`List`로 같은 구조를 만들어 `body()`에 넣으면 된다.

## 주의할 점

- **웹훅 URL은 비밀값.** Git에 올라가면 Slack이 감지해서 URL을 무효화할 수 있고, 그 전에 누군가 스팸을 보낼 수도 있다.
- **짧은 시간에 너무 많이 보내지 않는다.** 웹훅에도 전송 제한이 있어서 에러 로그마다 알림을 보내면 막힌다(429). 중요한 이벤트만 보내거나 모아서 보낸다.
- 보낸 메시지를 **수정·삭제하거나 채널을 코드에서 바꿔야 하면** 웹훅으로는 안 된다. 그때 Bot 토큰 + `chat.postMessage`로 넘어간다.

## 한 줄 정리

> 정해진 채널에 알림만 보낼 거라면 Slack 앱에서 Incoming Webhook URL을 받아 JSON을 POST하면 끝이다.
> 실제 로직에서는 트랜잭션 밖에서 실패를 잡았을 때만 알림을 보내고, 알림이 실패해도 본 로직은 깨지지 않게 한다.
