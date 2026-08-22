---
title: "JPA 엔티티와 값 객체 — 무엇이 다르고, 언제 무엇을 쓸까"
date: 2025-06-04
categories: [Java/Spring]
---

JPA로 도메인을 설계하다 보면 일단 다 `@Entity`로 만들고 싶은 유혹이 생긴다. 하지만 모든 걸
엔티티로 만들면 불필요한 테이블과 식별자가 늘어나고, 정작 지켜야 할 불변성은 지켜지지 않는다.
엔티티와 값 객체(Value Object)를 가르는 기준을 명확히 하면 이 문제가 정리된다.

## 판단 기준: 독립적인 생명주기가 있는가

```
독립적으로 조회/수정할 일이 있나?
        ↙               ↘
       YES               NO
        ↓                 ↓
   @Entity            @Embeddable
  (ManyToOne)      (ElementCollection)
```

**주문상품(OrderItem)**은 취소·수정·단독 조회가 필요하고 다른 엔티티(Item)를 참조한다 → 엔티티.
**주소(Address)**는 Member에 종속된 단순 값이고 그 자체로 조회/수정할 일이 없다 → 값 객체.

## 값 객체를 쓰는 이유: 식별자가 아니라 값으로 비교한다

엔티티는 식별자(Id)로 동일성을 판단하지만, 값 객체는 내부 필드 값 전체로 동일성을 판단한다.
그래서 값 객체는 **불변(Immutable)**으로 설계하는 게 원칙이다 — 생성 후 상태가 바뀌지 않으므로
예상치 못한 부작용이 없고, 여러 엔티티가 값 객체를 참조로 공유해도 안전하다.

```java
@Embeddable
public class Address {
    private String city;
    private String street;
    private String zipcode;

    protected Address() {}

    public Address(String city, String street, String zipcode) {
        if (city == null || city.isEmpty()) {
            throw new IllegalArgumentException("City cannot be empty.");
        }
        this.city = city;
        this.street = street;
        this.zipcode = zipcode;
    }
    // getter만 있고 setter 없음 — 불변

    @Override
    public boolean equals(Object o) { /* city, street, zipcode 전부 비교 */ }
    @Override
    public int hashCode() { /* 동일 */ }
}
```

값을 바꾸고 싶으면 필드를 수정하는 게 아니라 **새 값 객체로 통째로 교체**한다.

```java
public void setHomeAddress(Address homeAddress) {
    this.homeAddress = homeAddress; // 기존 객체 mutate 아님, 교체
}
```

**여기서 흔히 나는 실수**: `@Embeddable`을 불변으로 만들어놓고 실수로 setter를 추가하면,
값 객체가 여러 엔티티에서 참조를 공유하고 있을 때(특히 컬렉션에 넣어뒀거나 캐시에 올라간
경우) 한쪽에서의 변경이 다른 쪽에도 은근슬쩍 반영되는 버그로 이어진다. 값 객체에 setter를
넣고 싶어지는 순간이 "사실 이건 엔티티여야 하는 게 아닌가"를 다시 물어볼 신호다.

또 하나, **equals/hashCode를 빠뜨리면** 값 객체를 `Set`이나 `Map` 키로 쓸 때 같은 값인데도
다른 객체로 취급되는 버그가 생긴다. 기본 `Object.equals()`는 참조 비교라서, 값 객체의
"값으로 비교한다"는 정의 자체가 깨진다.

## @Embedded vs @ElementCollection

| | 기능 | DB 저장 방식 | equals/hashCode |
|---|---|---|---|
| `@Embedded` | 단일 값 객체를 엔티티에 포함 | 부모 테이블의 컬럼으로 (별도 테이블 X) | 필요 |
| `@ElementCollection` | 값 객체의 컬렉션 | 별도 테이블, 부모 PK를 FK로 | 필요 |

```java
@Entity
public class Product {
    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(name = "product_images", joinColumns = @JoinColumn(name = "product_id"))
    @OrderColumn(name = "image_index") // 순서 보장용 (선택)
    private List<ProductImage> images = new ArrayList<>();
}
```

`@CollectionTable`을 생략하면 `부모엔티티명_필드명`으로 테이블명이 자동 생성된다 — 명시적으로
이름을 안 주면 나중에 스키마를 눈으로 훑을 때 테이블명만으로 어떤 엔티티의 컬렉션인지 유추해야
하는 불편이 생기니, 실무에서는 대체로 명시하는 편이 낫다.

## 그럼 @OneToMany/@ManyToOne은 언제?

`@Embeddable`은 다른 엔티티를 참조할 수 없다. **OrderItem**처럼 다른 엔티티(Item)를 참조하거나
자체 비즈니스 로직(`cancel()`)이 있다면 반드시 `@Entity` + `@ManyToOne`이어야 한다.

### 실무 관례: 연관관계는 FK를 가진 쪽에서만

```java
// Order 쪽에는 @OneToMany 컬렉션을 두지 않는다
@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order; // 이것만으로 충분
}
```

`@OneToMany` 양방향 매핑을 굳이 안 쓰는 이유는, 필요하면 쿼리로 가져오면 되는데 양방향은
코드 복잡도와 N+1 문제 같은 성능 비용만 늘리기 때문이다.

### N:M은 `@ManyToMany` 대신 중간 엔티티로 직접 분해

`@ManyToMany`가 자동 생성하는 중간 테이블에는 추가 컬럼을 넣을 수 없고 직접 제어도 안 된다.
"주문 수량", "주문 시점 가격" 같은 부가 정보는 결국 필요해지기 마련이라, 처음부터
`OrderItem`처럼 중간 엔티티로 설계해두는 게 나중에 마이그레이션하는 것보다 싸다.

## 정리

| 상황 | 선택 |
|---|---|
| 단순 값, 독립 조회 불필요, 다른 엔티티 참조 없음 | `@Embeddable` (+ `@Embedded`/`@ElementCollection`) |
| 독립 조회/수정 필요, 다른 엔티티 참조, 비즈니스 로직 존재 | `@Entity` + `@ManyToOne` |
| 1:N | FK를 가진 쪽에서 `@ManyToOne` 단방향만 |
| N:M | 중간 엔티티로 분해 후 `@ManyToOne` × 2 |

결국 모든 선택의 출발점은 하나다 — **이 객체가 독립적인 생명주기를 가지는가.**
