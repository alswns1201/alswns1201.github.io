---
title: "SQS + Lambda + DynamoDB로 앱 푸시 보내기"
date: 2026-10-17
categories: [AWS]
---

앱 푸시(FCM)를 **API 서버에서 바로 보내지 않고**, SQS에 넣어 두면 Lambda가 꺼내서 보내는 구조로 만들었다.
보낸 결과(통계)와 사용자별 알림함은 DynamoDB에 저장하고, 어드민 화면과 앱은 DynamoDB에서 조회한다.
전체 구조와 각 단계에서 무엇을 하는지 정리해 둔다.

> 실무에서 만든 구조를 일반화해서 적었다. 서비스·큐·테이블·함수·API 이름은 모두 예시 이름이고, 코드는 구조를 보여 주기 위한 예시 코드다.
>
> 실습 코드: [github.com/alswns1201/sqs-lambda-push](https://github.com/alswns1201/sqs-lambda-push) — 커밋 하나가 한 단계라 `git log --reverse`로 따라가면 된다.

## 전체 구조

**발송(Producer) → SQS → Lambda(Consumer) → FCM / DynamoDB, 조회는 DynamoDB에서**

![앱 푸시 발송 흐름: Producer → SQS → Lambda → FCM, 결과는 DynamoDB](/assets/img/posts/aws-sqs-lambda-app-push.png)

구성 요소별 역할은 이렇다.

| 구성 | 역할 |
|---|---|
| **어드민** | 앱 사용자 목록에서 전체 / 선택 푸시를 등록한다 |
| **API 서버 (Producer)** | 대상 사용자를 조회해서 SQS에 메시지로 넣는다. 통계 row를 먼저 만든다 |
| **배치 서버 (Producer)** | 특정 조건을 만족한 사용자에게 자동으로 보내는 푸시. 사용자 1명 = 메시지 1건 |
| **SQS** | 보낼 푸시를 쌓아 두는 큐 |
| **Lambda (Consumer)** | 큐 메시지를 받아 FCM으로 실제 발송하고, 알림함·통계를 DynamoDB에 기록한다 |
| **DynamoDB** | `push_statistics`(발송 건별 통계), `push_history`(사용자별 알림함) |

## 왜 API 서버에서 바로 보내지 않았나

전체 푸시는 대상이 수만 명이 될 수 있다. API 서버가 FCM을 직접 호출하면

- 요청 하나가 **수만 명분의 FCM 호출을 다 끝낼 때까지** 붙잡혀 있고,
- 중간에 서버가 재시작되거나 FCM이 잠깐 실패하면 **어디까지 보냈는지** 알기 어렵다.
- 푸시를 보내는 동안 API 서버의 스레드·커넥션을 같이 써서 다른 요청에도 영향을 준다.

그래서 **API 서버는 "누구에게 보낼지"만 정해서 큐에 넣고 바로 응답**하고,
실제 발송은 큐를 소비하는 Lambda가 맡게 나눴다.

- **SQS**가 보낼 일을 들고 있으니 발송 쪽이 느려도 API는 영향을 받지 않는다.
- Lambda 처리가 실패하면 메시지가 큐에 다시 보여서 **재시도**된다.
- 메시지가 많으면 Lambda가 **여러 개 동시에 떠서** 나눠 처리한다.
- 관리자가 보내는 푸시와 배치가 보내는 자동 푸시가 **같은 큐·같은 Lambda를 공유**한다. 발송 로직이 한 곳에만 있다.

## 1. Producer — API 서버에서 SQS로 넣기

어드민이 전체 푸시를 등록하면 API 서버는 이렇게 처리한다.

1. **통계 row 생성** — `push_statistics`에 이번 발송 건의 row를 카운트 0으로 만든다.
2. **대상 조회** — 사용자를 **5,000명씩** 나눠 조회하고, **마케팅 수신 동의**한 사용자만 남긴다.
3. **메시지 만들기** — 사용자 **300명을 메시지 1건**으로 묶는다.
4. **SQS 전송** — 메시지를 **10건씩 `sendMessageBatch`**로 보낸다.
5. **totalUsers 보정** — 필터링이 끝나야 실제 대상 수를 알 수 있으니, 마지막에 통계 row의 `totalUsers`를 채운다.

숫자를 이렇게 잡은 이유:

- **5,000명씩 조회** — 전체 사용자를 한 번에 메모리에 올리지 않기 위해서다.
- **300명 = 메시지 1건** — 사용자 1명을 메시지 1건으로 만들면 메시지 수와 Lambda 호출이 너무 많아진다. 반대로 너무 많이 묶으면 SQS 메시지 크기 제한에 걸리고, 실패 시 재시도 단위도 커진다. 그 사이로 잡았다. FCM `send_each`가 한 번에 받을 수 있는 메시지 수(500)보다도 작다.
- **10건씩 `sendMessageBatch`** — SQS 배치 전송은 **한 번에 최대 10건**이 한도다. 1건씩 보내는 것보다 API 호출 수가 1/10로 준다.

SQS도 SES 글에서 쓴 것과 같은 **AWS SDK v2 BOM** 아래에 모듈만 추가한다.

```kotlin
implementation(platform("software.amazon.awssdk:bom:2.25.70"))
implementation("software.amazon.awssdk:sqs")
implementation("software.amazon.awssdk:dynamodb")
```

```java
@Service
@RequiredArgsConstructor
public class NotificationPushService {

	private static final int PAGE_SIZE = 5_000;
	private static final int USERS_PER_MESSAGE = 300;
	private static final int SQS_BATCH_SIZE = 10;

	private final SqsClient sqsClient;
	private final ObjectMapper objectMapper;
	private final UserRepository userRepository;
	private final NotificationStatisticsRepository statisticsRepository;

	@Value("${aws.sqs.push-queue-url}")
	private String queueUrl;   // 큐 URL은 배포 환경변수로 주입

	public void pushToAll(PushRequest request) {
		long sentAt = System.currentTimeMillis();
		statisticsRepository.create(sentAt, request);          // ① 카운트 0으로 생성

		int totalUsers = 0;
		for (List<User> page : userRepository.pages(PAGE_SIZE)) {
			List<User> targets = page.stream()
					.filter(User::isMarketingAgreed)            // ② 마케팅 동의 필터
					.toList();
			totalUsers += targets.size();

			List<SendMessageBatchRequestEntry> entries =
					Lists.partition(targets, USERS_PER_MESSAGE).stream()   // ③ 300명 = 1건
							.map(users -> toEntry(sentAt, request, users))
							.toList();

			for (List<SendMessageBatchRequestEntry> batch : Lists.partition(entries, SQS_BATCH_SIZE)) {
				sqsClient.sendMessageBatch(b -> b.queueUrl(queueUrl).entries(batch));   // ④ 10건씩
			}
		}

		statisticsRepository.updateTotalUsers(sentAt, totalUsers);   // ⑤ totalUsers 보정
	}
}
```

- `sendMessageBatch`는 일부 메시지만 실패할 수 있다. 응답의 **`failed()` 목록을 확인**해서 실패한 건은 로그를 남기거나 다시 보낸다. 예외가 안 났다고 전부 성공한 게 아니다.
- 큐 URL은 코드에 적지 않고 **배포 환경변수**로 받는다. 개발/운영 큐를 환경별로 바꾸기 쉽다.

**배치 서버**(조건 기반 자동 푸시)도 같은 큐에 넣는다. 이쪽은 대상이 한 명씩 정해지므로 **사용자 1명 = 메시지 1건**이고, 통계 row 생성·보정도 배치가 직접 한다.

## 2. Consumer — Lambda에서 FCM으로 보내기

Lambda는 **SQS 이벤트 소스 매핑(Event Source Mapping)**으로 큐에 연결했다.
큐에 메시지가 들어오면 AWS가 알아서 Lambda를 호출해 주므로, 큐를 폴링하는 코드를 직접 짤 필요가 없다.

메시지 1건(최대 300명)마다 Lambda가 하는 일:

1. **pushToken 유무로 분리** — 토큰이 없는 사용자(앱 삭제, 알림 미허용 등)는 FCM으로 보내지 않는다.
2. **FCM `send_each` 발송** — 토큰이 있는 사용자에게 한 번에 보낸다.
3. **history 저장** — 토큰 유무와 관계없이 **대상 전원**의 알림함에 `status=new`로 넣는다. 푸시를 못 받아도 앱 알림함에서는 볼 수 있게.
4. **statistics ADD** — `success / failure / noToken / processed` 카운트를 더한다.

```python
import json, time
import boto3
import firebase_admin
from firebase_admin import messaging

firebase_admin.initialize_app()          # 서비스 계정 키는 환경변수/Secrets Manager로
dynamodb = boto3.resource("dynamodb")
history = dynamodb.Table("push_history")
statistics = dynamodb.Table("push_statistics")


def handler(event, context):
    for record in event["Records"]:
        body = json.loads(record["body"])
        users = body["users"]

        # ① pushToken 유무로 분리
        with_token = [u for u in users if u.get("pushToken")]
        no_token = len(users) - len(with_token)

        # ② FCM 발송
        success = failure = 0
        if with_token:
            messages = [
                messaging.Message(
                    token=u["pushToken"],
                    notification=messaging.Notification(title=body["title"], body=body["message"]),
                    data=body.get("data", {}),
                )
                for u in with_token
            ]
            result = messaging.send_each(messages)
            success, failure = result.success_count, result.failure_count

        # ③ history 저장 (전체 대상, status=new)
        sent_at = int(time.time() * 1000)
        with history.batch_writer() as writer:
            for u in users:
                writer.put_item(Item={
                    "userId": u["userId"],
                    "notificationId": f'{body["sentAt"]}#{u["userId"]}',
                    "sentAt": sent_at,
                    "title": body["title"],
                    "message": body["message"],
                    "data": body.get("data", {}),
                    "status": "new",
                })

        # ④ statistics ADD
        statistics.update_item(
            Key={"pk": "STATS", "sentAt": body["sentAt"]},
            UpdateExpression="ADD success :s, failure :f, noToken :n, processedUsers :p",
            ExpressionAttributeValues={":s": success, ":f": failure, ":n": no_token, ":p": len(users)},
        )
```

- `send_each`는 **메시지마다 성공/실패를 따로** 돌려준다. 토큰 하나가 만료됐다고 300명 전체가 실패하지 않는다.
- 통계는 **`UpdateExpression`의 `ADD`**로 더한다. 메시지가 많으면 Lambda가 동시에 여러 개 떠서 같은 통계 row를 건드리는데,
  "읽고 → 더하고 → 쓰기"를 하면 서로 덮어써서 숫자가 빠진다. `ADD`는 DynamoDB가 원자적으로 더해 주므로 동시에 실행돼도 안전하다.
- Lambda가 예외로 끝나면 메시지가 큐로 돌아가 **다시 처리**된다. 이때 FCM 발송과 통계 ADD가 **한 번 더** 일어날 수 있다(SQS는 최소 한 번 전달).
  재시도 횟수를 정하고 넘으면 **DLQ(Dead Letter Queue)**로 빼 두는 설정을 같이 해 둔다.

## 3. DynamoDB 테이블 설계

DynamoDB는 **어떻게 조회할지를 먼저 정하고 키를 설계**한다. 파티션 키는 정확히 일치하는 값으로만 찾을 수 있고, 범위 조회·정렬은 정렬 키(sort key)로만 된다.

### push_statistics — 발송 1건 = row 1개

| 속성 | 설명 |
|---|---|
| **PK** `pk` | 항상 `"STATS"` 고정 |
| **SK** `sentAt` | 발송 시각(ms) |
| `totalUsers`, `success`, `failure`, `noToken`, `processedUsers` … | 통계 카운트 |

어드민 화면은 "발송 이력을 최신순으로 목록 보기"만 하면 된다.
그래서 **파티션 키를 고정값 하나로 두고, 정렬 키를 발송 시각**으로 잡았다. `pk = "STATS"`로 Query하고 정렬 키 역순으로 읽으면 바로 최신순 목록이 나온다.
(발송 건수가 아주 많아지면 한 파티션에 몰리는 구조라 월별 등으로 파티션 키를 나눠야 하지만, 관리자 발송 이력 정도의 양에서는 이 단순한 구조로 충분하다.)

- `processedUsers`가 `totalUsers`에 도달하면 해당 발송이 끝난 것이다.
- 어드민의 푸시 발송 이력 화면은 이 목록만 보여 준다. 사용자별 상세는 없다.

### push_history — 사용자별 알림함

| 속성 | 설명 |
|---|---|
| **PK** `userId` | 사용자 |
| **SK** `notificationId` | 알림 ID |
| **GSI** `userId-sentAt-index` | PK `userId`, SK `sentAt` |
| `title`, `message`, `data` | 알림 내용 |
| `status` | `new` / `unread` / `read` |

앱 알림함 API는 **"내 알림을 최근 30일, 최신순으로"** 보여 준다.
기본 키의 정렬 키는 `notificationId`라 시간 범위 조회가 안 되므로, **`sentAt`을 정렬 키로 하는 GSI**를 추가했다.

```java
QueryRequest request = QueryRequest.builder()
		.tableName("push_history")
		.indexName("userId-sentAt-index")
		.keyConditionExpression("userId = :userId AND sentAt >= :from")
		.expressionAttributeValues(Map.of(
				":userId", AttributeValue.fromS(userId),
				":from", AttributeValue.fromN(String.valueOf(thirtyDaysAgoMillis))))
		.scanIndexForward(false)   // 최신순
		.build();
```

- GSI는 테이블을 만든 뒤에도 추가할 수 있다(LSI는 테이블 생성 시에만 가능).
- 알림함에서는 **읽음 / 전체 읽음** 처리로 `status`를 바꾸고, **`new-count`** API로 새 알림 개수를 내려 준다.

## 정리

| 단계 | 하는 일 | 포인트 |
|---|---|---|
| Producer (API) | 5,000명씩 조회 → 마케팅 동의 필터 → 300명 = 1건 → 10건씩 `sendMessageBatch` | 통계 row 먼저 생성, 마지막에 `totalUsers` 보정 |
| Producer (배치) | 조건 기반 자동 푸시, 1명 = 1건 | 같은 큐·같은 Lambda 공유 |
| SQS | 보낼 푸시를 쌓아 둠 | API와 발송을 분리, 실패 시 재시도 |
| Lambda | 토큰 분리 → FCM `send_each` → history put → statistics `ADD` | 동시 실행돼도 `ADD`로 원자적 집계 |
| DynamoDB | 통계(최신순 목록), 알림함(사용자별 최근 30일) | 조회 패턴에 맞춰 PK/SK·GSI 설계 |

> API 서버는 "누구에게 보낼지"만 정해 SQS에 넣고, 실제 발송은 SQS가 트리거하는 Lambda가 FCM으로 한다.
> 결과는 DynamoDB에 남기고, 어드민·앱은 DynamoDB만 조회한다.

## 참고

- [\[AWS\] DynamoDB란](https://yoo11052.tistory.com/174)
- [AWS SES로 안내 메일 보내기 — AWS SDK v2 SesClient 적용](/posts/aws-ses-notice-mail/)
