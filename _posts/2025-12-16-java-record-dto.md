---
title: "Spring Boot에서 record를 DTO로 쓰기: 왜 잘 맞고, 왜 Entity에는 못 쓰는가"
date: 2025-12-16
categories: [Spring Boot]
tags: [java, spring-boot, jpa]
---

DTO를 만들 때마다 반복되던 보일러플레이트가 있다.

```java
public class UserDto {
    private final Long id;
    private final String name;

    public UserDto(Long id, String name) {
        this.id = id;
        this.name = name;
    }

    public Long getId() { return id; }
    public String getName() { return name; }

    @Override
    public boolean equals(Object o) { /* ... */ }
    @Override
    public int hashCode() { /* ... */ }
    @Override
    public String toString() { /* ... */ }
}
```

이 코드가 지키려는 규칙 — 모든 필드는 `final`, 생성자로만 초기화, setter 없음,
`equals`/`hashCode`/`toString` 오버라이딩 — 은 사실 "좋은 DTO는 불변 값 객체여야
한다"는 원칙을 수작업으로 흉내 낸 것이다. Java 16의 `record`는 이 원칙을 언어
차원에서 강제한다.

```java
public record User(Long id, String name) {}
```

이 한 줄이 위 클래스 전체와 동등하다 — `final` 필드, 생성자, 각 필드의 getter
(`id()`, `name()`), `equals`/`hashCode`/`toString`이 전부 자동 생성된다.

## record가 DTO에 잘 맞는 이유: 컴파일러가 원칙을 강제한다

DTO의 역할은 "레이어 사이에서 데이터를 옮기는 것"뿐이어야 한다는 데는 대부분
동의하지만, 실제 코드에서는 DTO에 슬금슬금 로직이 스며드는 일이 흔하다 — setter가
있으면 누군가는 결국 DTO를 받아서 값을 바꿔치기하는 코드를 짜게 되고, 그러다 보면
DTO가 상태를 가진 객체처럼 취급되기 시작한다. `record`는 애초에 setter를 만들
방법이 없고 상속도 불가능해서, 이런 침식이 구조적으로 일어날 수 없다. "이렇게
짜지 말자"는 컨벤션이 아니라 "이렇게 못 짠다"는 컴파일 제약으로 바뀌는 것 —
코드 리뷰에서 매번 지적하던 걸 언어가 대신해주는 셈이다.

## 놓치기 쉬운 디테일: 검증은 compact constructor에서

`record`가 생성자를 자동으로 만들어준다고 해서 검증 로직을 포기할 필요는 없다.
compact constructor로 필드 대입 전에 값을 점검할 수 있다.

```java
public record CreateOrderRequest(String productId, int quantity) {
    public CreateOrderRequest {
        if (quantity <= 0) {
            throw new IllegalArgumentException("quantity must be positive");
        }
    }
}
```

이렇게 하면 `new CreateOrderRequest(...)`가 생성되는 모든 경로에서 검증이
빠짐없이 실행된다 — 별도 setter가 없으므로 "검증을 거치지 않고 값을 바꾸는 우회로"
자체가 존재하지 않는다.

## 왜 Entity나 Domain 객체에는 못 쓰는가

원문이 "Entity나 Domain까지 침범하면 독이 된다"고만 짚고 넘어간 부분을 짚어보면,
이건 스타일 문제가 아니라 **구조적으로 불가능한 조합**이다.

- JPA는 지연 로딩(LAZY) 시 실제 엔티티 대신 프록시 객체를 반환한다. 이 프록시는
  런타임에 엔티티 클래스를 상속한 서브클래스를 생성하는 방식(CGLIB/ByteBuddy)으로
  만들어지는데, `record`는 암묵적으로 `final`이라 상속 자체가 불가능하다.
- JPA는 리플렉션으로 필드를 채우기 위해 기본적으로 파라미터 없는 생성자를 요구한다.
  `record`는 캐노니컬 생성자만 제공하고 무인자 생성자를 만들 방법이 없다.
- Dirty checking(변경 감지)은 필드 값이 바뀔 수 있다는 전제로 동작한다. `record`는
  모든 필드가 `final`이라 애초에 변경이 불가능하다.

세 가지 모두 "record가 불변을 강제한다"는 바로 그 특성에서 나오는 제약이다 —
DTO에서 장점이었던 바로 그 성질이 ORM 엔티티에서는 성립 자체를 막는다.

## Lombok `@Value`와 비교하면

Lombok의 `@Value`도 불변 DTO를 만들어주지만 방식이 다르다 — 컴파일 타임에
코드를 생성하는 어노테이션 프로세서다. `record`는 언어 자체의 문법이라 IDE의
디버거/바이트코드 뷰어가 별도 플러그인 없이도 온전히 이해하고, Lombok 버전 호환성
문제나 어노테이션 프로세서 설정 이슈에서 자유롭다. 신규 프로젝트에서 Java 17
이상을 쓸 수 있다면 DTO는 Lombok보다 `record`를 기본으로 검토할 이유가 여기 있다.

## 정리

- `record`는 Controller ↔ Service ↔ Response 구간의 순수 데이터 전달 객체에
  최적이다 — 원치 않는 상태 변경을 컴파일 타임에 막아준다.
- 검증이 필요하면 compact constructor에 넣는다. setter로 우회할 방법이 없으므로
  검증 누락 자체가 구조적으로 어렵다.
- Entity/Domain에는 쓸 수 없다 — 지연 로딩 프록시, 무인자 생성자, dirty checking
  모두 "필드가 바뀔 수 있어야 한다"는 전제 위에 서 있는데, record는 그 전제를
  정면으로 부정하기 때문이다.
