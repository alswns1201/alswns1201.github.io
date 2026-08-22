---
title: "제네릭은 왜 필요하고, 왜 런타임엔 사라지는가 — Function<T,R>과 타입 소거"
date: 2026-08-17
categories: [Java/Spring]
---

제네릭이 없던 시절 자바 컬렉션은 전부 `Object`를 담았다.

```java
List list = new ArrayList();
list.add("hello");
list.add(42); // 컴파일 에러 없음 — 뭐든 넣을 수 있었다

String s = (String) list.get(1); // 런타임에 ClassCastException
```

`list.get(1)`이 실제로는 `Integer`인데 `String`으로 캐스팅해버리면, 이 코드는
컴파일은 통과하지만 **실행하는 순간** 터진다. 제네릭은 이런 캐스팅 실수를
컴파일 타임으로 끌어올리기 위해 나왔다 — "이 컬렉션엔 `String`만 들어간다"는
걸 타입 시스템에 선언해두면, 컴파일러가 다른 타입이 섞이는 걸 애초에 막아준다.

```java
List<String> list = new ArrayList<>();
list.add("hello");
list.add(42); // 컴파일 에러 — 여기서 바로 잡힌다
```

## 제네릭 인터페이스: `Function<T, R>`

표준 라이브러리의 `Function<T, R>`이 대표적인 제네릭 인터페이스다.

```java
public interface Function<T, R> {
    R apply(T t);
}
```

`T`는 입력 타입, `R`은 반환 타입 — 실제 타입은 쓰는 쪽에서 채워 넣는다. 이벤트
소싱 실습([event-sourcing-practice](https://github.com/alswns1201/event-sourcing-practice))에서
쓴 예를 보면:

```java
private RewardView handle(String accountId, Instant now, Function<RewardAccount, RewardEvent> decide) {
    RewardAccount account = repository.load(accountId);
    RewardEvent event = decide.apply(account);
    ...
}

public RewardView earn(String accountId, long amount, Duration validFor) {
    Instant now = Instant.now();
    return handle(accountId, now, account -> account.decideEarn(amount, now, now.plus(validFor)));
}
```

`Function<RewardAccount, RewardEvent>`라고 선언하는 순간 "이 함수는 반드시
`RewardAccount`를 받아서 `RewardEvent`를 돌려줘야 한다"는 게 타입으로
고정된다. 만약 `decide.apply(account)`의 결과를 실수로 `String` 변수에 담으려
하면, 캐스팅이고 뭐고 할 것도 없이 그 자리에서 컴파일 에러가 난다. 제네릭이
없었다면 `Function` 대신 `Object apply(Object)` 시그니처를 쓰고, 호출부마다
`(RewardEvent) decide.apply(account)`처럼 캐스팅을 손으로 넣어야 했을
것이다 — 캐스팅 타입을 잘못 적어도 컴파일러는 못 잡고, 역시 런타임에야 터진다.

## 그런데 왜 `List<String>.class`는 안 될까 — 타입 소거

제네릭은 **컴파일 타임에만 존재**하고, 컴파일된 바이트코드에는 타입 파라미터
정보가 지워진다. 이걸 **타입 소거(type erasure)**라고 한다. 자바가 제네릭을
이렇게 설계한 이유는 하위 호환성 때문이다 — 제네릭은 Java 5에서 처음 도입됐는데,
그 이전에 컴파일된 수많은 코드(그리고 JVM 자체)가 제네릭을 모른 채로 동작하고
있었다. 런타임 타입 정보까지 유지하려면 바이트코드 포맷 자체를 바꿔야 했고,
그러면 기존 코드와의 호환이 깨진다. 그래서 자바는 "컴파일러가 타입 검사는
철저히 하되, 컴파일이 끝나면 제네릭 타입 정보는 지우고 옛날처럼 `Object`
기반 바이트코드로 만든다"는 절충을 택했다.

```java
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();

System.out.println(strings.getClass() == ints.getClass()); // true
```

런타임에는 `List<String>`과 `List<Integer>`가 **완전히 같은 클래스**다 —
`java.util.ArrayList`. `<String>`, `<Integer>`라는 정보는 컴파일이 끝나는
순간 사라졌다.

## 실전에서 부딪히는 지점: `Map<String, Object>.class`

Elasticsearch 클라이언트를 typesafe하게 바꾸던 [실습](/posts/elasticsearch-java-typesafe-client/)에서
정확히 이 문제를 만났다. 검색 결과를 `Map<String, Object>`로 받고 싶어서 이렇게
썼더니 컴파일이 안 됐다.

```java
Class<Map<String, Object>> clazz = Map.class; // 컴파일 에러
```

`Map<String, Object>.class`라는 표현 자체가 자바 문법에 존재하지 않는다.
`.class` 리터럴이 참조하는 `Class` 객체는 **런타임에 존재하는 타입**을
가리켜야 하는데, 타입 소거 때문에 런타임엔 `Map<String, Object>`라는 타입
자체가 없다 — `Map`(raw type)만 있을 뿐이다. 그러니 컴파일러 입장에서는
"존재하지도 않는 타입의 `.class`를 달라"는 요청이라 애초에 표현할 방법이
없다.

```
컴파일 타임: Map<String, Object>
런타임:      Map   (제네릭 정보 소거됨)
```

우회 방법은 `Class<?>`를 한 번 거쳐서 강제로 캐스팅하는 것이다.

```java
@SuppressWarnings("unchecked")
public List<Map<String, Object>> search(String index, String field, String keyword) {
    SearchResponse<Map<String, Object>> response = esClient.search(s -> s
                    .index(index)
                    .query(q -> q.match(m -> m.field(field).query(keyword))),
            (Class<Map<String, Object>>) (Class<?>) Map.class  // 강제 캐스팅
    );
    return response.hits().hits().stream().map(Hit::source).collect(Collectors.toList());
}
```

`Class<?>`를 경유하는 이유는, `Class<Map>`(raw)에서 `Class<Map<String, Object>>`로
바로 캐스팅하면 컴파일러가 "이 둘은 관련 없는 타입"이라며 거부하기 때문이다.
와일드카드 `Class<?>`는 "뭐가 됐든 어떤 `Class`"를 뜻하므로, 여기를 경유하면
컴파일러의 타입 검사를 억지로 통과시킬 수 있다. 대신 `@SuppressWarnings("unchecked")`가
붙는다 — 컴파일러가 "이 캐스팅이 실제로 안전한지 나는 증명할 수 없다"고
경고하는 걸, 개발자가 "그래도 이건 의도적으로 안전하다는 걸 알고 쓴다"고
명시적으로 억누르는 것이다. 이 어노테이션을 넓은 범위에 무분별하게 붙이면
진짜 타입 오류까지 같이 숨어버리므로, 원인이 명확한 지점에만 최소 범위로
써야 한다.

## 타입 소거가 실무에 남기는 흔적들

- **`instanceof`에 제네릭 타입을 못 쓴다**: `if (obj instanceof List<String>)`은
  컴파일 에러다. 런타임엔 `List<String>`이라는 타입 자체가 없으니 검사할
  방법이 없다 — `if (obj instanceof List<?>)`까지만 가능하다.
- **제네릭 배열을 못 만든다**: `new T[10]`이 안 되는 이유도 같다. 배열은
  런타임에 자기 원소 타입을 기억해야 하는데(공변성 검사를 위해), 소거된
  `T`로는 그 정보를 담을 방법이 없다.
- **오버로딩이 막힌다**: `void method(List<String> a)`와
  `void method(List<Integer> a)`를 같은 클래스에 오버로드할 수 없다. 소거
  후엔 둘 다 `method(List a)`로 완전히 같은 시그니처가 되기 때문이다.

## 정리

- 제네릭의 목적은 캐스팅 실수를 런타임 `ClassCastException`이 아니라 컴파일
  타임 에러로 끌어올리는 것이다.
- `Function<T, R>`처럼 표준 라이브러리 인터페이스도 제네릭이라서, 쓰는 쪽에서
  구체 타입을 지정하는 순간 그 시그니처가 컴파일러 수준에서 강제된다.
- 제네릭 타입 정보는 하위 호환성 때문에 컴파일 이후 소거된다 — 런타임에는
  존재하지 않는다.
- 이 때문에 `Map<String, Object>.class`, `instanceof List<String>`, `new T[]`
  같은 코드가 애초에 불가능하다. `Class<?>` 경유 캐스팅 + 범위를 좁힌
  `@SuppressWarnings("unchecked")`가 현실적인 우회다.
