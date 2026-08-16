---
title: "JPA 값 객체(Value Object)와 N+1 — @BatchSize vs Fetch Join, 언제 무엇을 쓸까"
date: 2025-05-18
---

*(원문: "Spring Boot JPA, 값 객체 가이드". 원문 뒷부분의 N+1 해결책 비교를 "왜 그 기준으로
고르는가" 중심으로 재구성했습니다 — 검토 부탁드려요)*

## 값 객체란: 식별자가 없다는 것의 의미

JPA에서 값 객체(Value Object)는 `@Id`가 없는 객체, 즉 **단독으로 조회/수정/삭제될
수 없고 항상 엔티티에 종속되는 객체**다. 상품(엔티티)에 딸린 상품 이미지가 좋은 예다 —
이미지 하나만 따로 조회하거나 삭제할 일이 없고, 항상 "어떤 상품의 이미지"로만 의미를
가진다. 이 "식별자 없음"이라는 성질은 DDD에서 말하는 값 객체 개념과도 일치한다 —
동일성이 아니라 속성값 자체로 비교되는 객체.

```java
@Embeddable
public class ProductImage implements Comparable<ProductImage> {
    private int idx;
    private String fileName;
}

@Entity
public class ProductEntity {
    @Id
    private Long pno;

    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(name = "tbl_product_images", joinColumns = @JoinColumn(name = "pno"))
    private SortedSet<ProductImage> images = new TreeSet<>();
}
```

`@Embeddable`은 "이 클래스는 엔티티가 아니라 다른 엔티티의 속성값으로 쓰인다"는 선언이고,
`@ElementCollection`은 그 값 객체를 컬렉션으로 담을 때 쓴다. `SortedSet`을 쓰려면
`Comparable`을 구현해야 하는데(자동 정렬을 위해), 이걸 안 하면 컬렉션에 넣는 순서가
곧 저장 순서가 되어버려서 이미지 순서 같은 걸 제어할 수 없게 된다.

## 값 객체 컬렉션은 왜 지연 로딩이 기본인가

값 객체 컬렉션(`@ElementCollection`)은 기본이 `FetchType.LAZY`다. 상품 1000개를
조회하는데 매번 이미지까지 즉시 로딩하면, 이미지가 필요 없는 목록 화면에서도
불필요한 조인/쿼리 비용을 치르게 된다. 문제는 이 지연 로딩이 **N+1 문제**를
만든다는 것 — 상품 1000개를 조회한 뒤, 이미지가 필요해지는 순간 상품 개수만큼
추가 쿼리(N번)가 나간다.

## N+1을 해결하는 두 가지 방법과 그 트레이드오프

**방법 1: `@BatchSize`**

```java
@ElementCollection(fetch = FetchType.LAZY)
@BatchSize(size = 100)
private SortedSet<ProductImage> images = new TreeSet<>();
```

N번의 개별 쿼리 대신, `IN` 절로 묶어서 최대 `size`개씩 한 번에 가져온다. 즉 N+1이
N/100+1 정도로 줄어드는 것이지, 완전히 1번으로 줄지는 않는다. 이 방식의 장점은
**페이징과 충돌하지 않는다**는 것 — 상위 엔티티(상품)는 정상적으로 페이징 쿼리로
가져오고, 하위 컬렉션(이미지)만 배치로 나중에 채운다.

**방법 2: Fetch Join**

```java
query.leftJoin(productEntity.images, productImage).fetchJoin();
```

한 번의 쿼리로 끝난다는 게 장점이지만, 대가가 크다 — **컬렉션을 Fetch Join하면
페이징이 불가능해진다.** SQL 레벨에서 LEFT JOIN을 하면 상품 하나에 이미지가 여러
개일 경우 행이 뻥튀기되는데(카티전 곱), JPA는 이 뻥튀기된 결과를 메모리에서 전부
가져온 뒤에야 페이징을 적용하게 된다 — 즉 `LIMIT`이 DB가 아니라 애플리케이션
메모리에서 걸리는 셈이라, 실질적으로 페이징이 안 된다고 봐야 한다.

## 선택 기준

- **최상위 엔티티 수가 적고, 각각이 참조하는 하위 컬렉션이 많다** → `@BatchSize`.
  페이징이 필요한 목록 화면 대부분이 여기에 해당한다.
- **연관 데이터가 이미 필터링되어 적고, 페이징이 필요 없다** → Fetch Join.
  예: 상세 화면에서 특정 상품 하나의 이미지를 전부 가져올 때.

경험적으로도(예: "구멍가게 코딩단"에서 언급되는 것처럼) 페이징이 조금이라도 걸리는
목록성 조회에서는 Fetch Join보다 `@BatchSize`나 별도 쿼리 분리가 안전하다는 의견이
많다 — 쿼리 한 번 더 나가는 비용보다, 페이징이 깨지는 리스크가 더 크기 때문이다.

## `@EntityGraph`는 또 다른 도구

`@EntityGraph(attributePaths = {"images"})`는 JPQL 없이도 특정 연관관계를 즉시
로딩하도록 지정할 수 있는 방법이다. 다만 이것도 내부적으로는 Fetch Join과 비슷하게
동작하므로, 컬렉션을 대상으로 쓸 때는 위와 같은 페이징 제약을 그대로 안고 간다 —
단일 연관관계(값 객체가 컬렉션이 아니라 단일 객체인 경우)에는 페이징 문제 없이
안전하게 쓸 수 있다.
