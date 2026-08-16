---
title: "Spring Data Envers로 엔티티 변경 이력 관리하기"
date: 2025-10-08
categories: [개발일지, SPRINGBOOT]
tags: [spring-boot, jpa, database]
---

"누가 언제 이 값을 바꿨는가"를 추적해야 하는 요구사항은 감사(audit) 로그, 데이터
복구, 특정 시점 조회처럼 흔하게 등장한다. 이걸 매번 별도 이력 테이블과 INSERT 트리거
로직으로 직접 짜는 대신, Hibernate Envers를 Spring Data와 통합한 **Spring Data Envers**를
쓰면 애노테이션만으로 해결할 수 있다.

## 동작 원리: 그림자 테이블

`@Audited`를 붙인 엔티티는 매 변경마다 원본 테이블과 별도로 `{table}_aud`라는
그림자(shadow) 테이블에 그 시점의 스냅샷이 함께 기록된다. 리비전 번호와 시각을 담는
`REVINFO` 테이블도 함께 생긴다. 이 모든 게 Hibernate의 이벤트 리스너에 후킹되어
자동으로 일어나기 때문에, 애플리케이션 코드에는 트랜잭션 로직이 전혀 추가되지 않는다.

```java
@Entity
@Audited
public class Book {
    @Id @GeneratedValue
    private int id;
    private String name;
    private int pages;
}

public interface BookRepository
        extends RevisionRepository<Book, Integer, Integer>, JpaRepository<Book, Integer> {
}
```

`RevisionRepository<Book, Integer, Integer>`의 두 `Integer`는 각각 엔티티 ID 타입과
리비전 ID(리비전 번호) 타입이다. 이 인터페이스를 상속하는 것만으로 기본 CRUD에
이력 조회 기능이 추가된다.

```java
// 마지막 변경 이력
Optional<Revision<Integer, Book>> lastRevision = repository.findLastChangeRevision(id);

// 전체 이력
Revisions<Integer, Book> revisions = repository.findRevisions(id);

// 특정 리비전 시점의 상태
Optional<Revision<Integer, Book>> revision = repository.findRevision(id, revisionNumber);
```

`book_aud` 테이블에는 매 변경마다 이런 식으로 쌓인다.

| id | rev | revtype | name | pages |
|---|---|---|---|---|
| 1 | 1 | 0 (ADD) | Spring in Action | 350 |
| 1 | 2 | 1 (MOD) | Spring in Action | 400 |
| 2 | 3 | 0 (ADD) | Spring | 444 |
| 1 | 4 | 2 (DEL) | NULL | NULL |

`revtype`이 생성(0)/수정(1)/삭제(2)를 구분하고, 삭제된 리비전은 `name`, `pages`가
NULL로 남는다 — 삭제됐다는 사실 자체가 이력이기 때문이다.

## 대가: 이게 공짜가 아니다

편리함 뒤에는 세 가지 비용이 있다.

**쓰기 성능 저하.** `@Audited` 엔티티는 매 INSERT/UPDATE/DELETE마다 원본 테이블 쓰기에
더해 `_aud` 테이블 쓰기가 하나 더 발생한다. 쓰기가 잦은 테이블(예: 실시간 카운터,
초당 수백 건씩 갱신되는 상태값)에 무분별하게 `@Audited`를 붙이면 눈에 띄는 성능
저하로 이어진다. 이력이 진짜 필요한 엔티티에만 선택적으로 붙여야 한다.

**저장 공간이 계속 커진다.** `_aud` 테이블은 삭제 없이 계속 쌓이기만 하는 구조라서,
시간이 지날수록 원본 테이블보다 훨씬 커진다. 별도의 보관 정책(오래된 리비전을
아카이빙하거나 삭제하는) 없이는 무한정 자란다.

**GDPR류 요구사항과 충돌한다.** "이 사용자의 데이터를 완전히 삭제해달라"는 요청이
오면, 원본 테이블에서 행을 지워도 `_aud` 테이블에는 과거 상태가 그대로 남아 있다.
Envers의 존재 목적 자체가 "지워진 데이터도 이력으로 남기는 것"이라서, 완전 삭제
요구사항과 정면으로 부딪힌다 — 이런 요구사항이 있는 도메인이라면 Envers를 쓰기 전에
이력 보관 기간과 파기 정책을 먼저 설계해야 한다.

## 언제 Envers 대신 다른 걸 써야 하는가

- **높은 쓰기 처리량이 필요한 테이블**: Envers의 동기적 이중 쓰기 비용을 감당하기
  어렵다면, 비동기 이벤트 발행(도메인 이벤트 → 별도 컨슈머가 이력 테이블에 기록) 쪽이
  쓰기 경로에 미치는 영향이 적다.
- **이력 자체가 비즈니스 로직의 핵심인 도메인** (주문 상태 전이, 결제 이력 등): 단순
  스냅샷 기록으로는 부족하고, 애초에 이벤트 소싱(event sourcing)처럼 상태 변화 자체를
  1급 데이터로 다루는 설계가 더 잘 맞는다. Envers는 "기존 CRUD 위에 감사 기능을
  얹는" 용도에 최적화되어 있지, 이력이 곧 도메인 모델인 경우를 위한 도구가 아니다.

## 정리

Spring Data Envers는 "이력 추적이 필요하지만 그게 도메인의 핵심은 아닌" 엔티티에
가장 잘 맞는다 — 적은 코드로 감사 요구사항을 해결할 수 있지만, 쓰기 성능·저장
공간·데이터 파기 정책이라는 세 가지 비용을 미리 계산에 넣어야 한다.
