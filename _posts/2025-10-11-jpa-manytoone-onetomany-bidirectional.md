---
title: "JPA @ManyToOne과 @OneToMany, 그리고 양방향 매핑의 진짜 함정"
date: 2025-10-11
categories: [개발일지, SPRINGBOOT]
tags: [spring-boot, jpa]
---

1:N 관계를 `@ManyToOne`/`@OneToMany`로 매핑하는 법 자체는 금방 외운다. 진짜 실수는
그 다음부터 나온다 — **연관관계의 주인을 헷갈리는 것**과 **N+1을 뒤늦게 발견하는 것**.
Crew(팀)-Member(회원) 예시로 이 둘을 짚는다.

## N 쪽이 외래 키를 갖는다: `@ManyToOne`

```java
@Entity
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "crew_id")
    private Crew crew; // Member는 하나의 Crew에 속한다
}
```

`@JoinColumn(name = "crew_id")`가 실제로 `member` 테이블에 `crew_id` 외래 키
컬럼을 만든다. `fetch = FetchType.LAZY`는 `member.getCrew()`를 실제로 호출하는
시점까지 조회를 미룬다는 뜻이고, `@ManyToOne`의 기본값이 원래 `EAGER`이기 때문에
이렇게 명시적으로 LAZY로 바꿔주는 게 관례다 — 매번 Member를 조회할 때마다 Crew까지
즉시 조인해서 가져올 이유가 없는 경우가 대부분이라서다.

## 1 쪽은 주인이 아니다: `@OneToMany`

```java
@Entity
public class Crew {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "crew", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Member> members = new ArrayList<>();

    public void addMember(Member member) {
        members.add(member);
        member.setCrew(this);
    }

    public void removeMember(Member member) {
        members.remove(member);
        member.setCrew(null);
    }
}
```

여기서 가장 중요한 한 줄은 `mappedBy = "crew"`다. **외래 키를 실제로 관리하는 쪽이
연관관계의 주인**이고, 이 예시에서는 `Member.crew`가 주인이다. `Crew` 쪽은
`mappedBy`로 "나는 주인이 아니고, 저쪽(`Member.crew`)이 이미 관계를 관리하고
있다"고 선언할 뿐이다 — 그래서 `crew` 테이블에는 `member_id` 같은 외래 키 컬럼이
생기지 않는다.

이 구분을 놓치면 실제로 어떤 문제가 생기는가: `Crew` 쪽 컬렉션에만 `member`를
추가하고 `member.setCrew(this)`를 빼먹으면, DB에는 아무 변화가 없다. JPA는
연관관계의 주인이 아닌 쪽(`Crew.members`)의 상태 변화를 무시하기 때문이다. 이게
바로 `addMember`/`removeMember` 같은 편의 메서드가 필요한 이유 — 양쪽 참조를
항상 같이 맞춰줘야 "주인이 아닌 쪽만 바꿔서 아무 일도 안 일어나는" 실수를 막을 수
있다.

`cascade = CascadeType.ALL`과 `orphanRemoval = true`도 같은 맥락이다. `Crew`를
저장/삭제할 때 그에 속한 `Member`들도 함께 저장/삭제되게 하고(`cascade`), 컬렉션에서
`Member`를 제거하면 그 즉시 DB에서도 삭제되게(`orphanRemoval`) 만든다 — 즉
"Member의 생명주기가 Crew에 완전히 종속된다"는 도메인 규칙을 코드로 강제하는
설정이다. 반대로 Member가 독립적으로 존재할 수 있는 도메인(예: 팀을 나가도 회원
데이터 자체는 남아야 하는 경우)이라면 이 두 옵션을 그대로 쓰면 안 된다.

## N+1: 리스트에 접근하는 순간 쿼리가 터진다

```java
@Transactional
public Crew getCrewWithMembers(Long crewId) {
    Crew crew = crewRepository.findById(crewId)
                      .orElseThrow(() -> new IllegalArgumentException("Crew not found"));

    for (Member member : crew.getMembers()) { // 이 지점에서 조회 쿼리 발생
        System.out.println(" - " + member.getName());
    }
    return crew;
}
```

`crew.getMembers()`를 호출하는 순간, LAZY 설정 때문에 그 Crew에 속한 Member들을
가져오는 쿼리가 별도로 나간다. 문제는 이게 Crew 하나에서 끝나지 않는다는 것 —
여러 Crew를 순회하면서 매번 `getMembers()`를 호출하면, Crew 조회 1번 + Member
조회 N번(Crew 개수만큼)이 발생한다. 이게 N+1 문제다.

## 해결책 세 가지와 각각의 트레이드오프

**1. Fetch Join** — 가장 확실하지만 컬렉션에 걸면 페이징이 깨진다.

```java
@Query("select distinct c from Crew c join fetch c.members")
List<Crew> findAllWithMembersFetchJoin();
```

`DISTINCT`가 필요한 이유는 SQL 레벨의 JOIN이 Crew 하나당 Member 개수만큼 행을
만들어내기 때문이다(카티전 곱) — DISTINCT 없이 그대로 두면 같은 Crew 객체가
Member 수만큼 중복으로 리스트에 담긴다. 그리고 이 카티전 곱 자체가 문제다:
`List<Member>` 같은 컬렉션을 Fetch Join하면 페이징(`LIMIT`/`OFFSET`)이 DB가
아니라 애플리케이션 메모리에서 처리되어, 사실상 페이징이 안 된다고 봐야 한다.

**2. `@EntityGraph`** — Fetch Join과 실질적으로 같은 문제를 안고 간다.

```java
public interface CrewRepository extends JpaRepository<Crew, Long> {
    @EntityGraph(attributePaths = "members")
    List<Crew> findAll();
}
```

JPQL을 직접 안 써도 되는 편의성이 장점이지만, 내부 동작은 Fetch Join과 비슷하므로
컬렉션에 적용할 때는 위와 동일한 페이징 제약이 그대로 따라온다.

**3. `@BatchSize`** — 페이징과 공존 가능한 대신, N+1이 완전히 사라지진 않는다.

```java
@OneToMany(mappedBy = "crew", cascade = CascadeType.ALL, orphanRemoval = true)
@BatchSize(size = 100)
private List<Member> members = new ArrayList<>();
```

N번의 개별 쿼리를 `IN` 절로 묶어서 최대 100개씩 가져온다 — N+1이 N/100+1 정도로
줄어드는 것이지 1번으로 줄지는 않는다. 대신 Crew 목록 자체는 정상적으로 페이징
쿼리로 가져올 수 있다는 게 핵심 이점이다.

## 선택 기준

- Crew 목록에 페이징이 필요하다 → `@BatchSize`. 페이징이 걸리는 목록 화면 대부분이
  여기 해당한다.
- 페이징이 필요 없고, 특정 Crew 하나의 Member 전체를 한 번에 봐야 한다(상세 화면) →
  Fetch Join 또는 `@EntityGraph`.

## 정리

`@ManyToOne`/`@OneToMany`를 매핑하는 문법보다 중요한 건 두 가지다 — **연관관계의
주인이 누구인지 정확히 알고 양방향 편의 메서드로 항상 양쪽을 같이 맞추는 것**,
그리고 **컬렉션에 LAZY로 접근하는 순간 N+1이 터질 수 있다는 걸 전제로 페이징 필요
여부에 따라 해결책을 고르는 것**이다.
