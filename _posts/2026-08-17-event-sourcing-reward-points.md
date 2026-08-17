---
title: "이벤트 소싱: '새 이벤트 없이 상태가 바뀌는' 도메인이 진짜 필요로 하는 이유"
date: 2026-08-17
categories: [아키텍쳐 설계 관련 글]
tags: [database, architecture, spring-boot]
---

[jinwookoh님의 이벤트 소싱 패턴 글](https://jinwookoh.tistory.com/251)을 읽고, 거기서
다룬 은행 계좌 예제로 이벤트 소싱의 동작 방식(append-only 저장, replay, 스냅샷, CQRS)을
Spring Boot로 직접 만들어봤다. 그런데 만들고 나서 다시 들여다보니 이상한 점이 있었다 —
**은행 잔액은 그냥 입금 합계에서 출금 합계를 뺀 값**이다. 원장(거래 내역) 테이블 하나에
`balance` 컬럼 하나만 얹으면 감사 추적도 되고 조회도 빠르다. 이벤트 로그, 리플레이,
스냅샷, 별도 CQRS 읽기 모델까지 갖출 이유가 딱히 없었다 — **이벤트 소싱을 보여주기
위한 이벤트 소싱**을 만든 셈이었다.

그래서 도메인을 바꿨다. 이번엔 반대로 **CRUD로는 애초에 정답을 낼 수 없는** 문제를
찾아서 다시 만들었다.

## "새 이벤트 없이 상태가 바뀐다"는 것

바뀐 도메인은 **유효기간이 있는 적립금(포인트)**이다. 규칙은 단순하다.

- 포인트는 적립될 때마다 배치로 쌓이고, 각 배치는 적립일 기준 유효기간이 지나면
  자동으로 소멸한다.
- 사용(redeem)은 오래된 배치부터 먼저 차감한다(FIFO).

이 도메인의 핵심은 이거다: **"지금 쓸 수 있는 포인트가 몇 개인가"라는 질문의 답이,
아무 사건도 일어나지 않았는데 시간이 지나는 것만으로 바뀐다.** 어제는 1500이었는데
오늘은 배치 하나가 만료돼서 700일 수 있다 — 그 사이에 입금도 출금도 없었는데도.

실제로 커맨드를 몇 번만 실행해보면 바로 보인다.

```bash
curl -X POST localhost:8080/api/rewards/u1/earn -d '{"amount":1000,"validForSeconds":2592000}' # 30일짜리
curl -X POST localhost:8080/api/rewards/u1/earn -d '{"amount":500,"validForSeconds":2}'         # 2초짜리
curl localhost:8080/api/rewards/u1/balance
# → {"usableBalance":1500, ...}

# 3초 대기 — 이 사이에 커맨드는 단 하나도 실행하지 않는다

curl localhost:8080/api/rewards/u1/balance
# → {"usableBalance":1000, ...}
```

`balance` 컬럼 하나로는 이걸 표현할 방법이 없다. "얼마가 남았는지"뿐 아니라 "언제
적립된 얼마가 아직 안 쓰였는지"를 알아야만 "지금 유효한 개수"를 계산할 수 있다 —
결국 각 배치의 이력을 들고 있어야 한다는 뜻이고, 이게 이 도메인이 이벤트(또는 최소한
날짜가 찍힌 원장)를 진짜로 요구하는 이유다.

## 이벤트: "결정"을 기록한다, 나중에 재계산하지 않는다

```java
public record PointsEarnedEvent(String batchId, long amount, Instant earnedAt, Instant expiresAt)
        implements RewardEvent { }

public record PointsRedeemedEvent(long amount, List<BatchConsumption> consumptions, Instant redeemedAt)
        implements RewardEvent { }
```

`PointsRedeemedEvent`는 "얼마를 뺐다"만 남기지 않고, **어느 배치에서 얼마씩 뺐는지**
(`consumptions`)까지 기록한다. 이유는 간단하다 — 어떤 배치가 소진 대상으로 유효했는지는
그 커맨드가 실행되던 "그 순간의 지금"에 달려 있다. 이 결정을 기록해두지 않고, 리플레이할
때마다 "현재 시각 기준으로" FIFO를 다시 계산하게 만들었다면, 몇 년 뒤 모든 배치가 만료된
뒤에 리플레이할 때 실제 있었던 일과 다른 결과가 나올 수 있다. 그래서 이 결정은 **한 번
내려서 이벤트에 그대로 박아두고, 리플레이는 그 결정을 재현하기만 한다.**

애그리거트 쪽에서 "지금 쓸 수 있는 잔액"은 필드가 아니라 매번 계산되는 값이다.

```java
public long usableBalance(Instant asOf) {
    return batches.values().stream()
            .filter(batch -> batch.expiresAt.isAfter(asOf))
            .mapToLong(batch -> batch.remaining)
            .sum();
}
```

같은 이벤트 로그를 두고 `asOf`만 다르게 넣으면 다른 답이 나온다 — 은행 계좌의
`balance` 필드처럼 "값 하나를 들고 있다가 돌려주는" 게 아니라, 매번 "이 시점 기준으로
계산"한다.

## 스냅샷: 스칼라 두 개가 아니라 리스트 전체

은행 계좌였다면 스냅샷은 `{balance, version}` 정도면 충분했다. 여기서는 애그리거트의
상태 자체가 배치 리스트이기 때문에, 스냅샷도 그 리스트 전체를 들고 있어야 한다.

```java
public record RewardSnapshot(String accountId, long version, List<BatchState> batches) { }
```

스냅샷 하나가 단순 숫자 몇 개가 아니라 JSON 블롭이 될 수 있다는 걸, 이 도메인을
만들면서 처음 체감했다 — 애그리거트 상태가 복잡해질수록 스냅샷도 같이 복잡해진다는
당연한 사실이지만, 은행 계좌 예제로는 절대 안 보이던 부분이다.

## 읽기 모델: 캐시된 숫자 하나로는 안 된다

은행 계좌 버전에서는 커맨드가 끝날 때마다 `account_summary` 테이블에 잔액 숫자
하나를 캐시해뒀다. 이 도메인에 똑같은 방식을 그대로 쓰면 문제가 생긴다 — 배치가
조용히 만료될 때마다 그 캐시된 숫자가 소리 없이 stale해진다. 갱신을 트리거할 커맨드
자체가 없기 때문이다.

그래서 읽기 모델을 "계정당 숫자 하나"가 아니라 "계정당 배치별 한 줄"로 바꿨다.

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
수만큼만 훑으면 되니까, 이벤트가 몇만 개 쌓여도 이 쿼리 비용은 안 늘어난다. 다만
"캐시된 숫자 하나 그대로 반환"보다는 필터링 연산이 하나 더 붙는다는 차이는 있다.

그리고 이 읽기 모델은 **"지금 또는 미래" 질문에만 정확하다.** 이미 다 써버린 배치나,
그 시점 이후에 새로 적립된 배치까지는 반영이 안 되기 때문에, 과거 시점을 물어보면
틀린 답을 준다. 그래서 과거 조회는 완전히 다른 경로를 하나 더 팠다.

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

느리지만(원본 로그를 그 시점까지 통째로 리플레이한다) 언제를 묻든 정확하다. 빠른 경로
(읽기 모델)와 느린 경로(원본 리플레이) 중 뭘 쓸지는 "지금/미래를 묻는가, 과거를
묻는가"로 갈린다 — 이 구분 자체가 CQRS 읽기 모델의 한계를 보여주는 지점이라
일부러 남겨뒀다.

## 그래서 이게 "진짜" 이벤트 소싱 문제인가

은행 계좌 버전을 만들고 나서 스스로에게 물었던 질문 — "이거 억지로 갖다 붙인 거
아닌가?" — 을 이번 버전에도 똑같이 던져봤다. 이번엔 답이 다르다: 잔액을 컬럼
하나로 표현할 수 없다는 것 자체가, 이 도메인이 애초에 "지금까지 무슨 일이 있었는지"를
들고 있어야만 정답을 낼 수 있다는 뜻이다. 원문의 결론("이벤트 소싱은 오버엔지니어링이
될 확률 99%, 돈과 관련되거나 변화 분석이 핵심인 경우만")을 다시 빌리면, 은행 계좌는
그 1%에 안 들어가고 이 적립금 예제는 들어간다.

실습 코드 전체는 [event-sourcing-practice](https://github.com/alswns1201/event-sourcing-practice)
레포에 있다.
