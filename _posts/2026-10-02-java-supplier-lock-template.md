---
title: "Supplier<T>로 '할 일'을 넘기기 — 분산 락 템플릿 예제"
date: 2026-10-02
categories: [Java/Spring]
---

선불 지갑 실습에서 Redisson 분산 락을 붙이면서 `executeWithLock(walletId, () -> ...)` 형태의
메서드를 만들었다. 여기서 쓴 `Supplier<T>`가 무엇이고, 왜 이 자리에 딱 맞는지 정리한다.

## Supplier<T>란

`java.util.function`에 있는 표준 함수형 인터페이스로, **인자 없이 호출하면 `T`를 돌려주는 함수**를 담는다.

```java
@FunctionalInterface
public interface Supplier<T> {
	T get();
}
```

## 왜 필요했나 — "락 안에서" 실행해야 하는 작업

충전·결제·취소는 모두 아래 순서로 실행돼야 한다.

```
락 획득 → [트랜잭션 시작 → 처리 → 커밋] → 락 해제
```

앞뒤의 락 처리는 같고 가운데 작업만 다르다. 그래서 가운데 작업을 **값이 아니라 함수로** 넘겨받고,
락을 잡은 뒤에 `task.get()`으로 실행한다.

```java
public <T> T executeWithLock(Long walletId, Supplier<T> task) {
	RLock lock = redissonClient.getLock("wallet:lock:" + walletId);
	boolean acquired = false;
	try {
		acquired = lock.tryLock(waitMillis, TimeUnit.MILLISECONDS);
		if (!acquired) {
			throw new WalletException(ErrorCode.LOCK_TIMEOUT);
		}
		return task.get();  // 이 시점에 비로소 실제 작업이 실행된다
	} catch (InterruptedException e) {
		Thread.currentThread().interrupt();
		throw new WalletException(ErrorCode.LOCK_TIMEOUT);
	} finally {
		if (acquired && lock.isHeldByCurrentThread()) {
			lock.unlock();
		}
	}
}
```

호출하는 쪽(Facade)은 이렇게 쓴다.

```java
public TransactionResponse charge(Long walletId, long amount) {
	return lockManager.executeWithLock(walletId, () -> walletService.charge(walletId, amount));
}
```

`walletService.charge()`는 `@Transactional`이라 프록시를 거쳐 트랜잭션을 열고, **커밋까지 끝낸 뒤에야
리턴**한다. 그래서 `task.get()`이 끝난 다음 `finally`에서 락을 풀면 자연스럽게 "락이 트랜잭션 전체를
감싸는" 구조가 된다.

## 핵심: 람다를 넘기는 시점에는 실행되지 않는다

```java
// ❌ 값을 넘김: charge()가 먼저 실행되고 결과만 넘어간다 → 락 밖에서 실행됨
executeWithLock(walletId, walletService.charge(walletId, amount));

// ✅ 람다를 넘김: "charge를 호출하는 방법"만 넘어간다 → 락 안에서 task.get() 할 때 실행됨
executeWithLock(walletId, () -> walletService.charge(walletId, amount));
```

(첫 번째는 사실 타입이 맞지 않아 컴파일도 안 되지만, "언제 실행되느냐"의 차이를 보여주는 예로 적었다.)

## 람다가 Supplier<TransactionResponse>가 되는 과정

```java
lockManager.executeWithLock(walletId, () -> walletService.charge(walletId, amount));
```

1. 두 번째 매개변수 타입이 `Supplier<T>`이므로 람다는 `Supplier`로 해석된다.
2. 람다 본문이 `TransactionResponse`를 리턴하므로 `T = TransactionResponse`로 추론된다.

풀어 쓰면 아래와 같다.

```java
Supplier<TransactionResponse> task = () -> walletService.charge(walletId, amount);
TransactionResponse result = lockManager.executeWithLock(walletId, task);
```

람다 모양과 `get()`의 대응은 이렇다.

```
() -> walletService.charge(walletId, amount)
│      └─ get()이 리턴할 값
└─ get()은 매개변수가 없으니 괄호가 비어 있다
```

`walletId`, `amount`는 람다의 매개변수가 아니라 **바깥 메서드의 값을 캡처한 것**이다.
그래서 괄호가 비어 있어도 서비스에 값을 넘길 수 있다. 캡처한 변수는 effectively final이어야 한다.

## 제네릭 `<T>`로 열어 둔 이유

지금은 충전·결제·취소 모두 `TransactionResponse`를 돌려주니 `Supplier<TransactionResponse>`로
고정해도 동작한다. 하지만 락 매니저는 "지갑 단위 락"이라는 범용 도구라서, 작업의 리턴 타입을 몰라도
되게 `<T>`로 열어 뒀다. 다른 타입을 돌려주는 작업을 감싸도 락 코드는 바꿀 필요가 없다.

## 다른 선택지와 비교

| 방식 | 단점 |
|---|---|
| 메서드마다 `try { lock } finally { unlock }` 복붙 | 같은 코드가 3번 반복되고, 하나만 실수해도 락이 안 풀린다 |
| `Runnable` | 리턴값이 없어 결과를 돌려줄 수 없다 |
| `Callable<T>` | 리턴은 되지만 `throws Exception` 때문에 호출부마다 체크 예외 처리가 필요하다 |
| **`Supplier<T>`** | 리턴값이 있고 체크 예외가 없다. 런타임 예외만 던지는 서비스에 잘 맞는다 |

작업 중 예외가 나도(예: 잔액 부족) `finally`에서 락은 반드시 풀리고, 예외는 그대로 위로 전파된다.

## 정리

- `() -> 서비스 호출`은 "서비스 결과 타입을 돌려주는 `Supplier`"라고 보면 된다.
- 람다를 넘기면 **실행 시점을 받는 쪽이 정한다** — 그래서 락 안에서 실행시킬 수 있다.
- 공통 처리(락)는 템플릿이, 가운데 할 일은 람다가 맡는 "템플릿 + 콜백" 패턴이다.
  Spring의 `TransactionTemplate.execute(status -> ...)`도 같은 방식이다.
