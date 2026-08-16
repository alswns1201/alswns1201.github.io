---
title: "비관적 락 vs 낙관적 락: '좋아요' 카운트로 보는 세 가지 구현"
date: 2026-05-26
categories: [인프런]
tags: [database, spring-boot, architecture]
---

두 락 방식의 차이는 결국 하나의 질문으로 요약된다 — **"충돌이 일어날 것이라고
가정하는가, 아닌가"**.

| | 비관적 락 | 낙관적 락 |
|---|---|---|
| 가정 | 충돌이 자주 일어난다 | 충돌이 거의 없다 |
| 방식 | DB 레벨에서 미리 잠금 | 버전으로 충돌 감지 후 재시도 |
| 단점 | 성능 저하 (대기) | 충돌 시 재시도 비용 |

'좋아요' 카운트 증가라는 동일한 요구사항을 세 가지 방식으로 구현해보면 이 차이가
분명해진다.

## 방식 1 — UPDATE 원자 연산 (사실상 비관적 락)

```java
@Transactional
public void likePessimisticLock1(Long articleId, Long userId) {
    articleLikeRepository.save(ArticleLike.create(snowflake.nextId(), articleId, userId));

    int result = articleLikeCountRepository.increase(articleId);
    if (result == 0) {
        // UPDATE된 행 없음 → 최초 요청이므로 INSERT
        articleLikeCountRepository.save(ArticleLikeCount.init(articleId, 1L));
    }
}
```

`UPDATE like_count = like_count + 1` 쿼리 자체가 원자적으로 실행되므로, DB가
해당 행에 **Row Lock**을 자동으로 걸어 동시 요청을 순차 처리한다. 별도로 잠금을
명시하지 않아도 되는, 가장 단순한 방식.

## 방식 2 — SELECT FOR UPDATE

```java
@Transactional
public void likePessimisticLock2(Long articleId, Long userId) {
    articleLikeRepository.save(...);

    ArticleLikeCount count = articleLikeCountRepository
        .findLockedByArticleId(articleId) // SELECT ... FOR UPDATE
        .orElseGet(() -> ArticleLikeCount.init(articleId, 0L));

    count.increase();
    articleLikeCountRepository.save(count);
}
```

조회 시점에 명시적으로 잠금을 걸어서, 읽기부터 쓰기까지 전 과정을 잠금 아래 둔다.
다른 트랜잭션은 이 행에 대한 `SELECT FOR UPDATE`를 요청한 순간부터 대기한다.
방식 1과 달리 **읽은 값을 기반으로 복잡한 로직을 수행한 뒤 저장**해야 하는
경우(단순 증가가 아니라 조건부 로직이 섞인 경우)에 필요해진다.

## 방식 3 — 낙관적 락

```java
@Transactional
public void likeOptimisticLock(Long articleId, Long userId) {
    articleLikeRepository.save(...);

    ArticleLikeCount count = articleLikeCountRepository
        .findById(articleId) // 일반 SELECT, 잠금 없음
        .orElseGet(() -> ArticleLikeCount.init(articleId, 0L));

    count.increase();
    articleLikeCountRepository.save(count);
    // 내부: UPDATE ... WHERE version = 기존버전
    // 버전 불일치 → OptimisticLockException!
}
```

잠금 없이 조회하고, 저장 시점에 `@Version` 필드로 충돌을 감지한다. 트랜잭션 A가
`version=1`을 읽고 수정해서 `version=2`로 커밋하면, 그사이 같은 행을 읽었던
트랜잭션 B는 `WHERE version = 1`이 더 이상 맞지 않아 `OptimisticLockException`을
던지며 실패한다.

## 원문에서 빠진 것: 낙관적 락은 재시도 로직 없이는 반쪽짜리다

여기서 짚어야 할 지점이 있다. 위 낙관적 락 코드는 **충돌을 감지만 하고 처리하지
않는다.** `OptimisticLockException`이 던져지면 그대로 요청이 실패한다. 실제
프로덕션에서 쓰려면 반드시 재시도 래퍼가 필요하다.

```java
@Retryable(
    retryFor = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 50)
)
public void likeWithRetry(Long articleId, Long userId) {
    likeOptimisticLock(articleId, userId);
}
```

재시도 로직이 빠진 낙관적 락은 "충돌 시 사용자에게 에러를 그대로 보여주는" 구현일
뿐이다. 이걸 감안하면 낙관적 락의 실제 비용은 "버전 컬럼 하나"가 아니라 "재시도
로직까지 포함한 구현 복잡도"라는 걸 알 수 있다.

## 세 방식 중 뭘 골라야 하는가

| 방식 | 잠금 시점 | 적합한 상황 |
|---|---|---|
| UPDATE 원자 연산 | UPDATE 실행 시 | 단순 카운터 증감처럼 로직이 없는 경우 |
| SELECT FOR UPDATE | SELECT 시점 | 조회한 값 기반으로 복잡한 로직 후 저장해야 하는 경우 |
| 낙관적 락 | 커밋 시점 | 충돌이 드물게 일어나는 리소스, 재시도 로직 구현 가능할 때 |

'좋아요' 카운트처럼 **동시에 매우 많은 트랜잭션이 같은 행을 두드리는** 핫 로우
(hot row)는 오히려 비관적 락(방식 1)이 유리한 경우가 많다 — 낙관적 락은 충돌이
잦으면 재시도가 계속 실패하며 쌓이는 "재시도 폭풍(retry storm)"이 발생할 수
있고, 이건 대기하는 비관적 락보다 오히려 더 나쁜 처리량을 낼 수 있다. 반대로
충돌이 드물게 일어나는 리소스(예: 사용자가 자기 프로필을 수정하는 경우)라면
낙관적 락이 잠금 대기 없이 훨씬 가볍게 동작한다.

**SELECT FOR UPDATE를 쓸 때 추가로 고려할 것**: 여러 개의 행을 서로 다른 순서로
잠그는 트랜잭션들이 동시에 존재하면 데드락이 발생할 수 있다. 예를 들어 트랜잭션
A가 행 1 → 행 2 순서로, 트랜잭션 B가 행 2 → 행 1 순서로 잠그려 하면 서로를
기다리며 영원히 멈춘다. 잠금 순서를 일관되게(예: 항상 ID가 작은 행부터) 유지하는
게 이 문제를 피하는 기본 원칙이다.
