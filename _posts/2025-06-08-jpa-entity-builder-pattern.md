---
title: "빌더 패턴으로 JPA 엔터티 안전하게 생성하기"
date: 2025-06-08
categories: [Java/Spring]
---

*(원제 "Builder 패턴을 적용한 엔터티". 원문의 핵심은 "연관관계 중간 엔터티에서 필수 값을
생성자 레벨 빌더로 강제하기"인데, 제목만 봐서는 그게 안 드러나서 바꿨습니다.)*

빌더 패턴 자체는 새로울 게 없다. 진짜 흥미로운 지점은 **JPA 엔터티, 그중에서도 연관관계
중간 엔터티에 빌더를 어떻게 적용하느냐**에 있다.

## 빌더가 푸는 두 가지 문제

생성자 파라미터가 늘어나면 두 가지 방식 중 하나로 가게 된다.

**점층적 생성자(Telescoping Constructor)** — 파라미터 조합별로 생성자를 여러 개 두는
방식. 파라미터 개수가 늘수록 생성자가 기하급수적으로 늘고, `new User(email, pw, null,
phone)`처럼 어떤 인자가 뭔지 호출부만 보고는 알기 어려워진다.

**자바빈즈 패턴(setter)** — 기본 생성자로 만든 뒤 setter로 채우는 방식. 문제는 두
가지다. 첫째, setter 호출이 끝나기 전까지 객체가 불완전한 상태로 존재할 수 있다(일관성
붕괴). 둘째, setter가 열려 있는 한 불변 객체를 만들 수 없다.

빌더는 `builder().field(value).field(value).build()` 형태로 이 둘을 동시에 해결한다
— 어떤 필드에 뭐가 들어가는지 호출부에서 명확하고, `build()` 시점에 한 번에 생성되므로
중간 상태가 노출되지 않는다.

## Lombok @Builder + JPA 엔터티

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED) // (1)
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String title;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String content;

    private String author;

    @Builder // (2)
    public Post(String title, String content, String author) {
        this.title = title;
        this.content = content;
        this.author = author;
    }
}
```

여기서 `@NoArgsConstructor(access = AccessLevel.PROTECTED)`가 왜 필요한지가 핵심이다.
JPA는 DB에서 엔터티를 조회할 때 리플렉션으로 객체를 생성하기 때문에 기본 생성자가
반드시 있어야 한다. 문제는 이걸 `public`으로 열어두면, 개발자가 실수로 `new Post()`처럼
불완전한 객체를 만들 수 있다는 것이다. `PROTECTED`로 좁혀두면 JPA 스펙(같은 패키지/상속
관계에서는 접근 가능해야 함)은 만족시키면서, 애플리케이션 코드에서의 오용은 막을 수
있다.

## 진짜 핵심: 클래스가 아니라 생성자에 @Builder 붙이기

`Member`가 `Post`에 좋아요를 누르는 `PostLike` 중간 엔터티를 생각해보자. `member`와
`post`는 이 엔터티의 존재 이유 그 자체라서 **둘 다 없으면 안 되는 필수 값**이다. 이걸
어떻게 강제할까?

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class PostLike {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", nullable = false)
    private Member member;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    private Post post;

    @Builder // 클래스가 아니라 이 생성자에만 적용
    public PostLike(Member member, Post post) {
        this.member = member;
        this.post = post;
    }
}
```

`@Builder`를 클래스 레벨이 아니라 `(Member member, Post post)` 생성자에 직접 붙였다.
이렇게 하면 Lombok은 **그 생성자에 정의된 파라미터만 받는 빌더**를 만든다. 결과적으로:

```java
PostLike like = PostLike.builder()
        .member(findMember)
        .post(findPost)
        .build();

// PostLike.builder().build();  ← member, post 없이 빌드 시도하면 컴파일 시점에 걸림
```

`@Builder`를 클래스에 붙였다면 `PostLike.builder().build()`처럼 `member`와 `post`가
둘 다 `null`인 상태로도 컴파일이 통과했을 것이다 — DB 제약(`nullable = false`)에서야
막히지, 컴파일 타임에는 못 잡는다. 생성자 레벨 `@Builder`는 이걸 **컴파일 타임 강제**로
끌어올린다. 필수 연관관계가 있는 중간 엔터티(좋아요, 팔로우, 참여 같은 N:M 조인
엔터티)에서 특히 값어치가 있는 패턴이다 — 이런 엔터티는 정의상 두 참조가 다 있어야만
의미가 성립하기 때문이다.

## 언제 이 패턴이 과한가

모든 엔터티에 이렇게까지 할 필요는 없다. 선택적(optional) 필드가 많은 엔터티에서
생성자 레벨 빌더를 억지로 쓰면, 오히려 "왜 이 필드는 빌더에 없지?"라는 혼란만 늘어난다.
이 패턴은 **"이 필드들이 없으면 객체 자체가 의미가 없다"는 게 명확한 경우**, 특히
연관관계의 존재 이유가 되는 FK들에 한정해서 쓰는 게 맞다.
