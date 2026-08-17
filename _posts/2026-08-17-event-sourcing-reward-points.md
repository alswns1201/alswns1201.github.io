---
title: "이벤트 소싱: '새 이벤트 없이 상태가 바뀌는' 도메인이 진짜 필요로 하는 이유"
date: 2026-08-17
categories: [아키텍쳐 설계 관련 글]
tags: [database, architecture, spring-boot]
---

이벤트 소싱은 "현재 상태를 저장하는 대신, 상태를 만들어낸 사건을 순서대로 저장한다"는
아이디어다. 상태가 필요하면 그 사건들을 처음부터 재생(replay)해서 계산한다. 개념
자체는 간단하지만, 이게 실제로 왜 필요한지는 **"현재 상태가 이벤트를 단순히 합산해서는
안 나오는" 도메인**을 봐야 감이 잡힌다. [유효기간이 있는 적립금(포인트)](https://github.com/alswns1201/event-sourcing-practice)이
딱 그런 예다.

## 규칙은 단순하다

- 포인트는 적립될 때마다 배치로 쌓이고, 각 배치는 적립일 기준 유효기간이 지나면
  자동으로 소멸한다.
- 사용(redeem)은 오래된 배치부터 먼저 차감한다(FIFO).

## 핵심: "지금 얼마나 쓸 수 있는가"의 답이 사건 없이도 바뀐다

이 도메인이 특별한 이유는 이거다: **아무 사건도 일어나지 않았는데, 시간이 지나는
것만으로 정답이 바뀐다.** 어제는 1500이었는데 오늘은 배치 하나가 만료돼서 700일 수
있다 — 그 사이에 적립도 사용도 없었는데도.

```bash
curl -X POST localhost:8080/api/rewards/u1/earn -d '{"amount":1000,"validForSeconds":2592000}' # 30일짜리
curl -X POST localhost:8080/api/rewards/u1/earn -d '{"amount":500,"validForSeconds":2}'         # 2초짜리
curl localhost:8080/api/rewards/u1/balance
# → {"usableBalance":1500, ...}

# 3초 대기 — 이 사이에 커맨드는 단 하나도 실행하지 않는다

curl localhost:8080/api/rewards/u1/balance
# → {"usableBalance":1000, ...}
```

`balance` 같은 단일 컬럼으로는 이걸 표현할 방법이 없다. "얼마가 남았는지"뿐 아니라
"언제 적립된 얼마가 아직 안 쓰였는지"를 알아야만 "지금 유효한 개수"를 계산할 수 있고,
결국 각 배치의 이력을 들고 있어야 한다. 이게 이 도메인이 이벤트(또는 최소한 날짜가
찍힌 원장)를 요구하는 이유다.

## 이벤트: "결정"을 기록한다, 나중에 재계산하지 않는다

```java
public record PointsEarnedEvent(String batchId, long amount, Instant earnedAt, Instant expiresAt)
        implements RewardEvent { }

public record PointsRedeemedEvent(long amount, List<BatchConsumption> consumptions, Instant redeemedAt)
        implements RewardEvent { }
```

`PointsRedeemedEvent`는 "얼마를 뺐다"만 남기지 않고, **어느 배치에서 얼마씩 뺐는지**
(`consumptions`)까지 기록한다. 어떤 배치가 소진 대상으로 유효했는지는 그 커맨드가
실행되던 "그 순간의 지금"에 달려 있기 때문이다. 이 결정을 기록해두지 않고 리플레이할
때마다 "현재 시각 기준으로" FIFO를 다시 계산하게 만들었다면, 몇 년 뒤 모든 배치가
만료된 뒤에 리플레이할 때 실제 있었던 일과 다른 결과가 나올 수 있다. 그래서 이 결정은
**한 번 내려서 이벤트에 그대로 박아두고, 리플레이는 그 결정을 재현하기만 한다.**

애그리거트 쪽에서 "지금 쓸 수 있는 잔액"은 필드가 아니라 매번 계산되는 값이다.

```java
public long usableBalance(Instant asOf) {
    return batches.values().stream()
            .filter(batch -> batch.expiresAt.isAfter(asOf))
            .mapToLong(batch -> batch.remaining)
            .sum();
}
```

같은 이벤트 로그를 두고 `asOf`만 다르게 넣으면 다른 답이 나온다 — 값을 하나 들고
있다가 그대로 돌려주는 게 아니라, 매번 "이 시점 기준으로" 다시 계산한다.

## 스냅샷: 스칼라 두 개가 아니라 리스트 전체

이벤트가 수천 개 쌓이면 매번 처음부터 재생하는 비용이 부담스러워진다. 그래서
일정 주기마다 그 시점까지의 계산 결과를 스냅샷으로 저장해두고, 다음번엔 그 스냅샷
이후 이벤트만 재생한다. 다만 이 도메인에서는 애그리거트의 상태 자체가 "배치 리스트"라서,
스냅샷도 숫자 하나가 아니라 그 리스트 전체를 들고 있어야 한다.

```java
public record RewardSnapshot(String accountId, long version, List<BatchState> batches) { }
```

## CQRS 읽기 모델: 캐시된 숫자 하나로는 안 된다

쓰기(이벤트 로그)와 읽기(조회용 모델)를 분리하는 CQRS는 이벤트 소싱과 거의 항상
같이 쓰인다 — 이벤트 로그 자체는 "지금 잔액이 얼마인가"를 답하기에 좋은 구조가
아니기 때문이다. 그런데 여기서 "커맨드가 끝날 때마다 잔액 숫자 하나를 캐시해둔다"는
흔한 방식을 그대로 쓰면 문제가 생긴다 — 배치가 조용히 만료될 때마다 그 캐시된 숫자가
소리 없이 stale해진다. 갱신을 트리거할 커맨드 자체가 없기 때문이다.

그래서 읽기 모델을 "계정당 숫자 하나"가 아니라 "계정당 배치별 한 줄"로 만든다.

```java
@GetMapping("/api/rewards/{accountId}/balance")
public BalanceView balance(@PathVariable String accountId, @RequestParam(required = false) Instant asOf) {
    Instant effectiveAsOf = asOf != null ? asOf : Instant.now();
    long usable = summaryRepository.findByAccountId(accountId).stream()
            .filter(row -> row.getExpiresAt().isAfter(effectiveAsOf))
            .mapToLong(RewardBatchSummaryEntity::getRemaining)
            .sum();
    return new BalanceView(accountId, usable, effectiveAsOf);
}
```

여전히 원본 이벤트 로그 전체를 리플레이하는 것보다는 훨씬 싸다 — 계정당 활성 배치
수만큼만 훑으면 되니, 이벤트가 몇만 개 쌓여도 이 쿼리 비용은 안 늘어난다. 다만 이
읽기 모델은 **"지금 또는 미래" 질문에만 정확하다.** 이미 다 써버린 배치나 그 시점
이후에 새로 적립된 배치까지는 반영이 안 되기 때문에, 과거 시점을 물어보면 틀린
답을 준다.

그래서 과거 조회는 완전히 다른 경로를 쓴다 — 원본 로그를 그 시점까지만 리플레이하는,
느리지만 항상 정확한 경로다.

```java
@GetMapping("/api/rewards/{accountId}/historical-balance")
public RewardView historicalBalance(@PathVariable String accountId, @RequestParam Instant asOf) {
    List<RewardEvent> eventsUpToThatPoint = eventStore.loadHistory(accountId).stream()
            .filter(record -> !record.occurredAt().isAfter(asOf))
            .map(StoredEventRecord::event)
            .toList();
    RewardAccount account = RewardAccount.empty(accountId).replay(eventsUpToThatPoint);
    return new RewardView(accountId, account.usableBalance(asOf), asOf, account.version());
}
```

빠른 경로(읽기 모델)와 느린 경로(원본 리플레이) 중 뭘 쓸지는 "지금/미래를 묻는가,
과거를 묻는가"로 갈린다 — 이 구분 자체가 CQRS 읽기 모델의 한계를 정확히 보여준다.

## 이런 도메인인지 어떻게 판단하는가

기준은 하나로 좁혀진다: **"현재 상태"가 단순 필드 하나로 표현 가능한가, 아니면
"지금까지 무슨 일이 있었는지" 전체를 알아야만 계산 가능한가.** 은행 잔액처럼
입금 합계에서 출금 합계를 빼기만 하면 되는 값은 원장 테이블 + balance 컬럼으로
충분하다. 반면 이 적립금 예제처럼 상태가 시간에 따라, 새 이벤트 없이도 변하는
경우는 이벤트(또는 최소한 날짜가 찍힌 이력)를 들고 있는 것 자체가 요구사항이다.
돈과 관련되어 있거나, 회계·법적 감사 요건이 있거나, 상태 변화 자체가 비즈니스
핵심인 경우가 대체로 여기 해당한다 — 그 외에는 이벤트 소싱이 요구하는 복잡도
(리플레이, 스냅샷, 별도 읽기 모델)를 감당할 이유가 없다.

실습 코드 전체는 [event-sourcing-practice](https://github.com/alswns1201/event-sourcing-practice)
레포에 있다.
