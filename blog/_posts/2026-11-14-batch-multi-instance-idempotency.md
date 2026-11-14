---
title: "배치 서버가 여러 대일 때 같은 알림을 두 번 보내지 않기 — 분산 락 대신 DB UNIQUE 키를 고른 이유"
date: 2026-11-14 12:00:00 +0900
categories: [개발 고민/설계]
---
{% raw %}

"결제한 사용자에게 안내 푸시를 한 번 보낸다", "가입하고 하루 동안 카드를 등록하지 않은 사용자에게 안내를 보낸다" 같은 알림을 배치로 만든다고 하자.
배치는 1분마다 돌면서 조건에 맞는 사용자를 찾아 보낸다. 서버가 한 대일 때는 간단하다. 문제는 배치 서버가 **두 대 이상**이 되는 순간이다.

- 장애 대비로 두 대를 띄웠다.
- 무중단 배포 중에는 새 버전과 이전 버전이 **잠깐 같이** 돈다.
- 오토 스케일링으로 늘어났다.

그러면 같은 1분짜리 작업이 동시에 두 번 실행되고, 같은 사용자에게 알림이 두 번 간다. 이 글은 이걸 막는 방법들을 비교하고, **"한 대만 돌게 하기"보다 "몇 대가 돌아도 한 번만 보내지게 하기"** 를 고른 이유를 정리한다.

> 글에 나오는 테이블·컬럼·키 형식은 모두 예시다. 코드는 구조를 보여 주기 위한 예시 코드다.

## 막아야 하는 것 세 가지

1. **동시 실행 중복**: 인스턴스 A와 B가 같은 분에 같은 사용자를 찾아 둘 다 보낸다.
2. **겹친 실행 중복**: 한 번의 실행이 1분을 넘겨, 다음 실행과 같은 데이터를 처리한다(한 대여도 생긴다).
3. **누락**: 배치가 몇 분 멈췄거나 배포 중이었던 동안의 이벤트를 아무도 처리하지 않는다.

1·2번은 "중복", 3번은 "누락"이다. **누락을 막으려고 넉넉하게 다시 훑으면 중복이 늘고, 중복을 막으려고 딱 맞게 훑으면 누락이 생긴다.** 둘을 같이 풀어야 한다.

## 선택지 비교

### 1) 한 대만 실행되게 한다 — 분산 락 / ShedLock

실행 직전에 락을 잡고, 잡은 인스턴스만 실행한다.

```java
@Scheduled(cron = "0 * * * * *")
@SchedulerLock(name = "campaignNotify", lockAtMostFor = "50s", lockAtLeastFor = "10s")
public void run() { ... }
```

- 장점: 기존 코드를 거의 안 바꾼다.
- 한계:
  - `lockAtMostFor`(락 최대 유지 시간)보다 오래 걸리면 락이 풀리고 **다음 실행이 겹친다.** 너무 길게 잡으면 인스턴스가 죽었을 때 그 시간 동안 아무도 못 돈다.
  - 서버마다 시계가 다르면 락 판단이 어긋난다.
  - 락은 "동시에 실행되는 것"만 막는다. **이미 보낸 사용자에게 다음 실행이 또 보내는 것**은 별도로 막아야 한다. 결국 "보냈는지" 기록이 따로 필요하다.

락의 세부 함정은 [Redisson 분산 락 글](/posts/redisson-distributed-lock/)에 정리했다.

### 2) Redis에 "보냄" 표시를 남긴다 — SETNX

```java
Boolean first = redis.opsForValue().setIfAbsent("sent:rule:7:user:42", "1", Duration.ofDays(30));
if (Boolean.TRUE.equals(first)) send(...);
```

- 장점: 빠르고, 여러 인스턴스가 경쟁해도 한 명만 `true`를 받는다.
- 한계:
  - **TTL이 끝나면 다시 보낸다.** "사용자당 평생 1회"를 TTL로 표현할 수 없다. TTL 없이 두면 Redis 메모리에 영원히 쌓인다.
  - **발송 기록(DB)과 저장소가 다르다.** Redis에는 표시했는데 DB 저장이 실패하거나, 그 반대가 되면 둘이 어긋난다. 두 저장소를 한 트랜잭션으로 묶을 수 없다.
  - Redis 복제는 비동기라 페일오버 순간 최근 표시가 사라질 수 있다. 사라진 만큼 다시 보낸다.
  - 운영자가 "이 사용자에게 언제 무엇을 보냈나"를 보려면 결국 DB 기록이 필요하다.

### 3) DB UNIQUE 키로 "한 번만 기록되게" 한다 — 멱등 키

발송 기록 테이블에 **멱등 키(idempotency key)** 컬럼을 두고 UNIQUE 제약을 건다. 보내기 전에 기록을 먼저 넣고, **넣기에 성공한 쪽만 보낸다.**

```sql
CREATE TABLE notification_instance (
    id               BIGSERIAL PRIMARY KEY,
    rule_id          BIGINT       NOT NULL,
    user_id          BIGINT       NOT NULL,
    idempotency_key  VARCHAR(200) NOT NULL UNIQUE,
    status           VARCHAR(20)  NOT NULL,   -- PENDING → SENT / CANCELLED
    scheduled_at     TIMESTAMPTZ  NOT NULL,
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

```sql
INSERT INTO notification_instance (rule_id, user_id, idempotency_key, status, scheduled_at)
VALUES (:ruleId, :userId, :key, 'PENDING', :scheduledAt)
ON CONFLICT (idempotency_key) DO NOTHING
RETURNING id;
```

- `RETURNING`으로 행이 돌아오면 내가 처음 넣은 것이니 진행하고, 아무것도 안 돌아오면 이미 누군가 넣은 것이니 건너뛴다.
- 인스턴스 A와 B가 동시에 같은 키를 넣어도 **DB가 하나만 통과시킨다.** 몇 대가 돌든, 몇 번을 다시 훑든 결과는 같다.
- 기록과 중복 판단이 **같은 테이블, 같은 트랜잭션**이라 어긋날 일이 없고, TTL 없이 영구적이며, 그대로 발송 이력이 된다.

> PostgreSQL에서는 UNIQUE 위반 예외를 잡아서 무시하는 방식(`try { insert } catch (DuplicateKeyException e)`)을 쓰면 안 된다. 예외가 나는 순간 **그 트랜잭션 전체가 실패 상태**가 되어 이후 쿼리가 모두 거부된다. `ON CONFLICT DO NOTHING`을 쓰거나, 꼭 예외로 처리해야 하면 SAVEPOINT를 둔다.

### 키를 정책에서 만든다

"한 번"의 기준은 알림마다 다르다. 키 형식으로 그 기준을 표현한다.

| 발송 정책 | 키 | 의미 |
|---|---|---|
| 사용자당 1회 | `rule:{ruleId}:user:{userId}` | 이 알림은 한 사람에게 평생 한 번 |
| 대상 건당 1회 | `rule:{ruleId}:ref:{결제ID}` | 결제할 때마다 한 번씩 |
| 제한 없음 | `rule:{ruleId}:user:{userId}:at:{이벤트 발생 시각}` | 같은 이벤트를 두 번 처리하는 것만 막음 |

"제한 없음"이어도 키를 두는 이유는 3번 문제(다시 훑기) 때문이다. **같은 이벤트를 두 번 보는 건 막아야** 하므로 이벤트 발생 시각이나 이벤트 ID를 키에 넣는다.

## 누락 막기: 되돌아보기 구간 + 멱등 키

"마지막 실행 시각 이후"만 훑는 방식은 깔끔해 보이지만 두 가지 이유로 놓친다.

1. 배치가 멈춘 동안의 이벤트는 "마지막 실행 시각"을 어떻게 저장하느냐에 따라 건너뛰어진다. 여러 인스턴스가 그 값을 동시에 갱신하면 더 꼬인다.
2. **늦게 커밋되는 데이터.** 결제가 10:00:30에 `created_at`을 찍었지만 트랜잭션이 10:01:05에 커밋됐다면, 10:01:00 실행은 그 행을 못 보고, 다음 실행은 "10:01:00 이후"만 보니 영원히 놓친다.

그래서 매번 **넉넉하게 뒤로 돌아가서** 다시 훑는다.

```
기준 시각 = max(지금 - 되돌아보기 시간, 알림을 켠 시각)
대상 = 기준 시각 이후 발생한 이벤트
```

- 되돌아보기 시간(예: 1시간)은 "배치가 멈출 수 있는 최대 시간 + 커밋 지연"보다 길게 잡는다.
- 같은 이벤트를 60번 다시 보게 되지만, **멱등 키 덕분에 처음 한 번만 기록되고 나머지는 `DO NOTHING`** 이다. 되돌아보기가 가능한 이유가 바로 멱등 키다.
- `max(..., 알림을 켠 시각)`은 **알림을 켜기 전 이벤트에는 보내지 않기** 위해서다. 이걸 빼면 새 알림을 켜는 순간 지난 한 시간의 대상에게 한꺼번에 나간다. 껐다가 다시 켤 때도 마찬가지다.
- 되돌아보기 구간이 길수록 쿼리 비용이 늘어나니 이벤트 시각 컬럼에 인덱스를 둔다.

## 보내는 단계: 기록과 발송을 나눈다

기록을 넣는 것과 실제로 보내는 것을 같은 실행에서 바로 하면, 기록은 됐는데 발송 직전에 서버가 죽은 건이 `PENDING`으로 남는다. 또 "밤에는 아침 8시로 미루기" 같은 예약도 필요하다. 그래서 발송은 **`PENDING`이고 `scheduled_at`이 지난 행을 가져가서 보내는** 단계로 분리한다.

여기서도 인스턴스 여러 대가 같은 행을 가져가면 안 된다. `FOR UPDATE SKIP LOCKED`로 **서로 다른 행을 나눠 가지게** 한다.

```sql
WITH picked AS (
    SELECT id FROM notification_instance
    WHERE status = 'PENDING' AND scheduled_at <= now()
    ORDER BY scheduled_at
    LIMIT 300
    FOR UPDATE SKIP LOCKED
)
UPDATE notification_instance n
SET status = 'SENDING'
FROM picked
WHERE n.id = picked.id
RETURNING n.*;
```

- 다른 인스턴스가 잠근 행은 기다리지 않고 건너뛴다. 락 경합 없이 일을 나눠 갖는다.
- 상태를 `SENDING`으로 바꾼 뒤 커밋하고, 큐(SQS 등)에 넣은 다음 `SENT`로 바꾼다. 오래 `SENDING`에 머문 행은 별도 정리 작업이 되살린다.
- **보내기 직전에 알림이 아직 켜져 있는지 다시 확인**한다. 예약해 둔 사이 관리자가 끄거나 지웠으면 `CANCELLED`로 바꾸고 보내지 않는다.

## 아래 단계도 "최소 한 번"이다

큐(SQS)는 기본적으로 **최소 한 번 전달**이다. 같은 메시지가 두 번 소비될 수 있고, 푸시 발송 서비스(FCM 등)는 중복을 걸러 주지 않는다. 배치에서 아무리 한 번만 넣어도 소비자 쪽에서 두 번 보낼 수 있다는 뜻이다.

- 소비자가 저장하는 알림함·결과 기록도 `(알림 ID, 사용자 ID)` 같은 키로 **덮어쓰기**가 되게 한다. 두 번 처리돼도 행은 하나다.
- 통계를 누적(`ADD`)한다면 재처리 시 두 번 더해질 수 있다는 점을 알고 쓴다. 정확해야 하면 결과 기록을 세어서 통계를 만든다.
- 큐 메시지에 알림 ID를 넣어 두면 "어디까지 보냈는지" 추적이 쉬워진다. 비동기 발송 구조는 [SQS + Lambda 푸시 글](/posts/aws-sqs-lambda-app-push/)에 정리했다.

## 한눈에 비교

| | 분산 락 / ShedLock | Redis SETNX 표시 | DB UNIQUE 멱등 키 |
|---|---|---|---|
| 동시 실행 중복 | 막음 (락 유지 시간 안에서) | 막음 | 막음 |
| 다음 실행의 재발송 | 별도 기록 필요 | TTL 동안만 막음 | 영구히 막음 |
| 되돌아보기(넉넉히 다시 훑기) | 별도 기록 필요 | TTL 안에서만 | 그대로 가능 |
| 발송 기록과 정합성 | 별개 | 저장소가 달라 어긋날 수 있음 | 같은 트랜잭션 |
| 추가 인프라 | 락 저장소 | Redis | 없음 (기존 DB) |
| 비용 | 실행마다 락 1번 | 대상마다 Redis 1번 | 대상마다 INSERT 1번 |

락은 "실행을 한 번으로", 멱등 키는 "**결과를 한 번으로**" 만든다. 알림에서 지켜야 하는 건 결과다. 그래서 몇 대가 돌든, 몇 번 다시 훑든 결과가 같아지는 멱등 키를 기본으로 두고, 필요하면 락은 **불필요한 중복 조회를 줄이는 최적화**로 덧붙인다.

## 흔한 실수

- **"조회 → 없으면 INSERT"** 로 중복을 검사한다. 두 인스턴스가 동시에 "없음"을 보고 둘 다 넣는다. 검사는 반드시 UNIQUE 제약에 맡긴다.
- **UNIQUE 위반 예외를 잡고 계속 진행한다**(PostgreSQL). 트랜잭션이 이미 실패 상태라 다음 쿼리가 터진다.
- **알림을 켠 시각을 기준에 넣지 않는다.** 켜자마자 과거 대상에게 한꺼번에 발송된다.
- **발송 직전 재확인을 하지 않는다.** 꺼진 알림이 예약분만큼 계속 나간다.
- **배포 중 두 버전이 같이 도는 걸 잊는다.** "어차피 한 대"라고 생각한 배치도 배포 순간에는 두 대다.

## 정리

> 배치 서버가 여러 대면 같은 작업이 동시에, 혹은 겹쳐서 실행된다. 분산 락은 실행을 하나로 줄이지만 "이미 보낸 사람에게 다시 보내기"와 "누락"은 따로 풀어야 하고, Redis 표시는 TTL과 저장소 불일치 문제가 남는다.
> 발송 기록 테이블에 **정책별 멱등 키 + UNIQUE 제약**을 두면, 몇 대가 돌든 결과는 한 번이다. 이 덕분에 **되돌아보기 구간으로 넉넉하게 다시 훑어 누락을 막을 수 있고**, 발송 단계는 `FOR UPDATE SKIP LOCKED`로 인스턴스끼리 나눠 갖는다. 큐 아래 단계도 최소 한 번 전달이라는 점까지 생각해서 저장은 덮어쓰기로 만든다.

## 참고

- [PostgreSQL — INSERT … ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)
- [PostgreSQL — SELECT … FOR UPDATE SKIP LOCKED](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE)
- [ShedLock](https://github.com/lukas-krecan/ShedLock)
- [Amazon SQS — At-least-once delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html)
{% endraw %}
