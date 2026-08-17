---
title: "이벤트 소싱: 상태 대신 사건을 저장한다는 것의 실제 의미"
date: 2026-08-17
categories: [아키텍쳐 설계 관련 글]
tags: [database, architecture, spring-boot]
---

통장 잔액이 100만 원이라고 하자. 그런데 이 100만 원이 어디서 왔는지 설명할 수
있는가? 월급 300만 원이 들어오고, 200만 원을 카드값으로 냈다는 걸 통장 앱이
말해주는 이유는 은행이 "잔액"이 아니라 "입출금 내역"을 저장하기 때문이다. 이
차이 — **현재 상태만 남길 것인가, 상태를 만들어낸 사건을 전부 남길 것인가** —
가 이벤트 소싱을 이해하는 전부다.

## 방식 1: 상태 저장 (CRUD)

지금 대부분의 서비스가 쓰는 방식이다. 테이블에는 현재 시점의 최신 데이터만
있고, 값이 바뀌면 그 자리에서 덮어쓴다.

```java
@Service
@Transactional
public class AccountService {
    public void deposit(Long id, int amount) {
        Account account = accountRepository.findById(id).orElseThrow();
        account.setBalance(account.getBalance() + amount);
    }
}
```

`UPDATE` 한 줄로 잔액을 바꾸는 순간, "바뀌기 전엔 얼마였는지"는 사라진다. 구현이
직관적이고 조회 성능도 좋지만, 과거 데이터를 조회할 수 없고 "왜 이 값이 됐는지"를
나중에 추적할 방법이 없다.

## 방식 2: 이벤트 소싱

이벤트 소싱은 현재 상태를 저장하지 않는다. 대신 일어난 사건을 순서대로,
**추가만 가능한(append-only)** 로그에 쌓는다. 현재 상태가 필요하면 그 사건들을
처음부터 순서대로 재생(replay)해서 계산한다.

```java
class DepositEvent { public int amount; }
class WithdrawEvent { public int amount; }

public void saveEvent(Long aggregateId, Object event) {
    eventStore.append(aggregateId, event);
}

public Account getAccount(Long id) {
    Account account = new Account();
    List<Object> events = eventStore.loadEvents(id);

    for (Object event : events) {
        if (event instanceof DepositEvent)
            account.balance += ((DepositEvent) event).amount;
        else if (event instanceof WithdrawEvent)
            account.balance -= ((WithdrawEvent) event).amount;
    }
    return account;
}
```

`UPDATE`도 `DELETE`도 없다. 잔액이 바뀌는 게 아니라, "1000원이 입금됐다"는
사건이 새로 하나 추가될 뿐이다. 이 구조 덕분에 CRUD로는 못 하는 게 가능해진다:
언제든 이벤트 로그를 처음부터 다시 재생하면, 그 시점의 상태를 정확히 그대로
재구성할 수 있다 — 완벽한 감사 로그이자, 과거 임의 시점으로 돌아가는 타임머신이다.

### "이벤트가 100만 개면 매번 다 재생하나?" — 스냅샷

당연히 드는 의문이다. 계좌 하나에 이벤트가 수십만 개 쌓이면, 잔액 하나 조회하는데
수십만 번의 연산이 필요해진다. 해결책은 스냅샷이다 — 일정 주기(예: 이벤트 100개
마다)로 그 시점까지의 계산 결과를 저장해두고, 조회할 때는 가장 가까운 스냅샷부터
시작해서 그 이후 이벤트만 재생한다. 이벤트 전체가 아니라 "스냅샷 이후 최대 100개"만
재생하면 되니 조회 비용이 이벤트 총량과 무관하게 일정해진다.

### CQRS는 선택이 아니라 짝꿍이다

이벤트 로그는 "지금 잔액이 얼마인가"를 묻기에 좋은 구조가 아니다 — 매번 재생해야
답이 나온다. 그래서 이벤트 소싱은 거의 항상 **CQRS**(Command Query Responsibility
Segregation)와 함께 쓰인다: 쓰기는 이벤트 로그에, 읽기는 조회에 최적화된 별도
테이블(읽기 모델)에 하고, 이벤트가 추가될 때마다 그 읽기 모델을 갱신해둔다. 이벤트
소싱을 쓰기로 했다면 CQRS는 옵션이 아니라 사실상 필수 짝꿍이다.

## 실습: 원문의 의사코드를 실제로 돌아가게 만들면

원문 코드는 개념을 보여주는 의사코드에 가깝다. `eventStore.append(aggregateId,
event)` 한 줄 뒤에 숨어 있는 질문들 — "두 요청이 동시에 오면?", "읽기 모델은
누가 언제 갱신하는가?" — 을 실제로 풀어보려고 Spring Boot + JPA로 똑같은 은행
계좌 예제를 [만들어봤다](https://github.com/alswns1201/event-sourcing-practice).

### 동시성: append만 있다고 안전한 게 아니다

"추가만 한다"는 게 동시성 문제를 저절로 없애주지는 않는다. 계좌 하나에 두 요청이
거의 동시에 들어와서, 둘 다 "현재 버전은 5"라고 읽고 각자 6번째 이벤트를 추가하려
하면 어떻게 될까? 둘 다 성공해버리면 6번째 자리에 이벤트 두 개가 겹쳐 쓰인다.

이걸 막으려면 이벤트를 저장할 때 "내가 읽었던 버전"을 같이 넘기고, 그 버전이
여전히 최신인지 DB가 확인하게 만들어야 한다.

```java
@Entity
@Table(name = "stored_event",
       uniqueConstraints = @UniqueConstraint(columnNames = {"aggregate_id", "sequence_number"}))
public class StoredEventEntity { ... }
```

`(aggregate_id, sequence_number)`에 유니크 제약을 걸어두면, 두 트랜잭션이 동시에
같은 계좌의 같은 순번으로 쓰려고 할 때 DB가 하나만 받아주고 나머지는 제약 위반으로
튕겨낸다. "먼저 버전을 확인하고 나서 삽입한다"는 애플리케이션 코드로는 확인과
삽입 사이에 경쟁 조건(race window)이 남지만, 유니크 제약은 DB 레벨에서 원자적으로
막아준다. 이 위반을 잡아서 "누군가 먼저 다녀갔다"는 의미의 예외로 바꿔주면 된다.

```java
try {
    repository.saveAllAndFlush(rows);
} catch (DataIntegrityViolationException e) {
    throw new ConcurrencyConflictException(aggregateId, expectedVersion);
}
```

### 시간여행 조회

스냅샷 없이 이벤트를 특정 버전까지만 재생하면, 그 시점의 상태를 정확히 재구성할
수 있다.

```java
@GetMapping("/api/accounts/{accountId}/at")
public AccountView asOfVersion(@PathVariable String accountId, @RequestParam long version) {
    List<BankAccountEvent> eventsUpToThatPoint = eventStore.loadUpTo(accountId, version);
    BankAccount account = BankAccount.empty(accountId).replay(eventsUpToThatPoint);
    return new AccountView(accountId, account.balance(), account.version());
}
```

실제로 입금 1000원 → 출금 300원 → 입금 5000원 순서로 커맨드를 세 번 보낸 뒤
`GET /api/accounts/acc-1/at?version=2`를 호출하면, 현재 잔액은 5700원인데도 이
API는 700원을 돌려준다 — "두 번째 사건이 일어난 직후엔 얼마였는가"라는, CRUD
테이블 하나로는 애초에 답할 방법이 없는 질문에 답하는 것이다.

## 그래서 언제 쓰나

이벤트 소싱은 **오버엔지니어링이 될 확률이 99%**다. 게시판, 회원 정보처럼 "지금
값이 뭔지"만 중요한 단순 데이터에 쓰면 코드만 몇 배로 복잡해지고 얻는 게 없다.
반대로 돈과 관련되어 있거나(결제, 정산), 회계·법적 감사 요건이 있거나, 상태
변화 자체가 비즈니스의 핵심(버전 관리 시스템처럼)인 경우에는 "왜 이 상태가
됐는가"를 설명할 수 있어야 한다는 요구사항 자체가 이벤트 소싱을 정당화한다.
조회 성능 저하(그래서 CQRS가 필수가 된다), 구현 난이도, 이벤트 스키마가 바뀔 때의
마이그레이션 복잡도 — 이 비용을 감수할 만한 이유가 있는지가 기준이다.

실습 코드 전체는 [event-sourcing-practice](https://github.com/alswns1201/event-sourcing-practice)
레포에 있다.
