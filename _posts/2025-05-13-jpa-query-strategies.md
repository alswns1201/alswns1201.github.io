---
title: "JPA 데이터 조회 전략 4가지: 언제 무엇을 써야 하는가"
date: 2025-05-13
categories: [Spring Boot]
---

Spring Data JPA로 데이터를 조회하는 방법은 하나가 아니다. 기본 메소드, 쿼리 메소드,
JPQL/Native Query, QueryDSL — 네 가지가 공존한다. 문제는 이 중 뭘 쓸지가 아니라,
**넷 다 프로젝트 안에서 같이 쓰이게 된다**는 점이다. 그래서 중요한 건 각 방법의 사용법이
아니라 "이 조회는 어떤 방법이어야 하는가"를 판단하는 기준이다.

## 1) JpaRepository 기본 메소드 — 별다른 판단이 필요 없을 때

```java
public interface TodoRepository extends JpaRepository<TodoEntity, Long> {
    // save(), findById(), findAll(), deleteById() 등 기본 제공
}
```

`findAll(Pageable)`처럼 조건 없이 전체를 페이징만 하면 되는 경우가 여기 해당한다.
고민할 필요가 없다는 게 이 계층의 존재 이유다 — 조건이 하나라도 붙는 순간 다음 단계로
넘어가야 한다.

## 2) 쿼리 메소드 — 조건이 1~2개, 그리고 "이 조건은 앞으로도 안 바뀐다"일 때

```java
Page<TodoEntity> findByTitleContaining(String keyword, Pageable pageable);
List<TodoEntity> findByAuthorAndPriorityGreaterThanEqual(String author, Integer minPriority);
```

메소드 이름만으로 JPQL을 생성해주는 기능. 장점은 가독성, 단점은 조건이 늘어날수록
메소드 이름이 감당이 안 된다는 것이다. `findByAuthorAndPriorityGreaterThanEqualAndStatusNot(...)`
같은 이름을 실제로 마주치면 이 방식의 한계를 체감하게 된다.

**판단 기준**: 조건 조합이 고정적이고 2개를 넘지 않을 때만 쓴다. "나중에 조건이
추가될 수도 있다"는 낌새가 조금이라도 있으면 처음부터 QueryDSL로 가는 게 낫다 —
쿼리 메소드에서 QueryDSL로 리팩터링하는 비용이 생각보다 크다.

## 3) JPQL / Native Query — 고정된 복잡한 쿼리 하나에

```java
@Query("SELECT t FROM TodoEntity t JOIN FETCH t.member m WHERE m.username = :username")
List<TodoEntity> findTodosByUsernameWithMember(@Param("username") String username);
```

JPQL은 엔티티 객체를 대상으로 하는 쿼리 언어라서, 테이블이 아니라 매핑된 필드 기준으로
작성한다. `JOIN FETCH`가 특히 중요한데, 이게 없으면 연관 엔티티는 지연 로딩(LAZY)
설정에 따라 N+1 문제를 일으킨다 — 리스트를 순회하면서 `member.getUsername()`을 호출할
때마다 추가 쿼리가 나가는 식이다. JPQL로 미리 fetch join을 걸어주는 게 이 문제를 막는
가장 직접적인 방법이다.

Native Query는 여기서 한 단계 더 나아가 실제 DB 방언(dialect)에 종속된 SQL을 직접
쓰는 것이다. DB 벤더를 바꿀 계획이 있다면 피해야 하고, 그렇지 않다면 JPQL로 표현이
안 되는 DB 전용 함수(윈도우 함수, 특정 인덱스 힌트 등)를 써야 할 때만 쓴다.

**판단 기준**: 조건이 고정적이지만 조인/서브쿼리/집계가 섞여서 쿼리 메소드로는
표현이 안 될 때. 단, 이 쿼리에 나중에 동적 조건이 붙을 가능성이 있다면 처음부터
QueryDSL로 시작하는 게 낫다 — `@Query` 문자열에 조건부 WHERE절을 넣으려면 결국
스트링 조립이나 SpEL을 쓰게 되는데, 둘 다 컴파일 타임에 오류를 잡아주지 못한다.

## 4) QueryDSL — 조건이 동적이거나, 쿼리가 복잡해질 가능성이 있을 때

```java
public List<Member> findByName(String name) {
    QMember member = QMember.member;
    return queryFactory.selectFrom(member)
            .where(member.name.eq(name))
            .fetch();
}
```

핵심 장점은 **자바 코드로 쿼리를 작성해서 컴파일 타임에 오류를 잡는다**는 것과,
동적 쿼리를 조립하기 쉽다는 것이다. `BooleanBuilder`나 `where(조건1, 조건2, ...)`
형태로 null인 조건을 자동으로 무시하게 만들 수 있어서, "검색 조건이 있을 때만
필터링"하는 흔한 요구사항에 특히 잘 맞는다.

```java
List<MemberTeamDto> content = queryFactory
        .select(new QMemberTeamDto(member.id, member.username, member.age, team.id, team.name))
        .from(member)
        .leftJoin(member.team, team)
        .where(
            usernameEq(condition.getUsername()),
            teamNameEq(condition.getTeamName()),
            ageGoe(condition.getAgeGoe()),
            ageLoe(condition.getAgeLoe())
        )
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();
```

여기서 `usernameEq`, `teamNameEq` 같은 메소드가 조건값이 null이면 `null`을 반환해서
`where`절에서 자동으로 빠지게 하는 패턴이다. 이게 QueryDSL을 쓰는 진짜 이유에 가깝다 —
JPQL 문자열로 이런 동적 조건을 짜려면 if문으로 쿼리 문자열을 이어붙여야 하고, 실수하기
쉽다.

**주의할 점 두 가지**:
- `JPAQueryFactory`는 자체 페이징 기능이 없다. `offset()`/`limit()`을 직접 계산하거나
  `PageableExecutionUtils.getPage()`로 count 쿼리를 분리해서 처리해야 한다. 여기서
  흔한 실수가, 전체 개수를 구하는 count 쿼리에도 fetch join을 그대로 남겨서 불필요하게
  느려지는 것이다 — count 쿼리에는 fetch join이 필요 없다.
- `QuerydslRepositorySupport`를 상속하는 방식은 지양하는 게 낫다. 이 방식은
  `JpaRepository`의 기본 CRUD 메소드를 포기해야 하고, 내부적으로 쓰는 `EntityManager`가
  Spring의 `@Transactional` 관리와 충돌할 수 있다. `JPAQueryFactory`를 Bean으로
  등록해서 커스텀 레포지토리에 주입하는 방식이 훨씬 예측 가능하다.

## 정리: 선택 기준

| 상황 | 선택 |
|---|---|
| 조건 없음, 전체 조회 | 기본 메소드 |
| 고정 조건 1~2개 | 쿼리 메소드 |
| 고정된 복잡한 쿼리 (조인/집계) | JPQL |
| DB 전용 기능이 꼭 필요 | Native Query |
| 조건이 동적으로 조합됨 | QueryDSL |

네 가지를 순서대로 배우는 것보다, 지금 짜는 쿼리가 "조건이 앞으로 바뀔 가능성이
있는가"를 먼저 묻는 게 실제로 유용한 판단 기준이다.
