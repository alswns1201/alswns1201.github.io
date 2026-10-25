---
title: "@DataJpaTest는 서비스 테스트용이 아니다 — new로 서비스를 만들면 생기는 문제"
date: 2026-09-06
categories: [테스트]
---

코드 리뷰에서 `@DataJpaTest` 안에 서비스를 `new`로 직접 생성해서 쓰는 코드를 봤다.
테스트는 통과했지만, 이 방식이 왜 위험한지 — `@DataJpaTest`가 정확히 뭘 검증하도록
설계됐는지부터 짚어야 설명이 된다.

## `@DataJpaTest`가 하는 일

- **JPA 관련 컴포넌트만** 로드하는 테스트 슬라이스다: `@Entity` 스캔, Spring Data
  JPA 리포지토리, `TestEntityManager`.
- 기본적으로 실제 DB가 아니라 **임베디드 DB(H2 등)로 자동 교체**된다.
- **`@Service`, `@Component`, `@Controller`는 스캔 대상이 아니다.** 컨텍스트를
  가볍게 유지해서 빠르게 돌리는 게 목적이라, 리포지토리 계층 바깥의 빈은 애초에
  올라오지 않는다.
- 각 테스트는 기본적으로 트랜잭션으로 감싸져 있고, 끝나면 자동 롤백된다.

즉 설계 의도 자체가 "리포지토리가 실제 DB 앞에서 맞게 동작하는가"를 좁게 검증하는
슬라이스 테스트다. 서비스 로직 검증은 이 슬라이스의 책임이 아니다.

## 그런데도 서비스를 테스트하려고 하면 — `new`로 생성하는 안티패턴

`@DataJpaTest`는 서비스를 빈으로 안 올려주니, 서비스 로직까지 같이 확인하고
싶어지면 이런 코드가 나온다.

```java
@DataJpaTest
class OrderServiceTest {

    @Autowired
    OrderRepository orderRepository; // 이건 진짜 빈 (H2 기반)

    OrderService orderService;

    @BeforeEach
    void setUp() {
        orderService = new OrderService(orderRepository); // 문제 지점
    }
}
```

`new`로 만든 `OrderService`는 **Spring이 관리하는 빈이 아니다.** 컨테이너를 거치지
않았으니 프록시로 감쌀 기회 자체가 없다.

- `OrderService` 메서드에 `@Transactional`이 붙어 있어도 **조용히 무시된다.**
  프록시가 없으니 트랜잭션 어드바이스가 개입할 지점 자체가 없다.
- `@Validated`, `@Async` 등 프록시 기반으로 동작하는 다른 기능도 전부 죽는다.
- 필드 주입(`@Autowired` 필드)을 쓰는 의존성이 있다면 `new`로는 채워지지 않아
  `NullPointerException`이 날 수 있다 (생성자 주입만 쓰면 회피 가능).

## 이게 왜 특히 위험한가 — 테스트는 통과하는데 실제로는 검증이 안 됨

`@DataJpaTest` 자체가 테스트 메서드를 트랜잭션으로 감싸고 끝나면 롤백해준다.
그래서 서비스의 `@Transactional`이 무효여도 **데이터는 어차피 테스트 프레임워크가
롤백해주고, 겉보기엔 테스트가 문제없이 통과한다.** 하지만 서비스가
`@Transactional(propagation = REQUIRES_NEW)`나 `rollbackFor`로 특정 예외 상황에서
부분 커밋/롤백을 의도한 로직이라면, 그 동작 자체가 테스트에서 전혀 검증되지 않은
채 통과해버린다. 프로덕션에서는 실제 프록시가 걸린 빈이 동작하니, 테스트와 다르게
행동할 수 있다 — "테스트는 초록불인데 운영에서 롤백이 안 됐다"는 사고가 여기서
나온다.

## 리뷰 코멘트 예시

이런 코드를 리뷰할 때는 "뭘 바꿔야 하는지"뿐 아니라 "왜 지금 방식이 위험한지"까지
같이 적어야 리뷰이가 다음에 같은 패턴을 스스로 피할 수 있다.

> `@DataJpaTest`는 리포지토리 계층만 검증하도록 설계된 슬라이스 테스트라
> `@Service` 빈이 컨텍스트에 로드되지 않습니다. 그래서 서비스를 `new`로 직접
> 생성하면 Spring이 관리하는 프록시가 아니게 되어 `@Transactional` 등 AOP 기반
> 동작이 조용히 무시된 채 테스트만 통과하는 문제가 있습니다. `@Import(OrderService.class)`로
> 서비스를 실제 빈으로 등록해 `@Autowired`로 주입받거나, 여러 계층을 함께
> 검증해야 한다면 `@SpringBootTest`를 쓰는 게 좋을 것 같습니다.

여기서 `@Autowired`만 단독으로 제안하면 안 된다 — `@DataJpaTest`는 `@Service`를
스캔하지 않으므로, `@Import`로 명시적으로 빈을 끌어오지 않은 채 `@Autowired`만
붙이면 `NoSuchBeanDefinitionException`이 난다. `@Import`와 `@Autowired`는 항상
같이 가야 하는 조합이다.

```java
@DataJpaTest
@Import(OrderService.class) // 스프링이 진짜 빈으로 등록 + 프록시 생성
class OrderServiceTest {

    @Autowired
    OrderService orderService; // 이제 진짜 관리되는 빈, @Transactional도 정상 적용
}
```

## 그래서 서비스는 뭘로 테스트해야 하나

무겁고 가벼움 순으로 4단계로 나뉜다.

1. **순수 단위 테스트 (기본값)**: 리포지토리를 Mockito로 mock 처리. 스프링
   컨텍스트를 아예 안 띄우니 가장 빠르고, 대부분의 서비스 분기/조건 로직은
   이걸로 충분하다.

   ```java
   @ExtendWith(MockitoExtension.class)
   class OrderServiceTest {
       @Mock OrderRepository orderRepository;
       @InjectMocks OrderService orderService;

       @Test
       void 재고가_없으면_예외를_던진다() {
           given(orderRepository.findById(1L)).willReturn(Optional.empty());
           assertThatThrownBy(() -> orderService.placeOrder(1L))
               .isInstanceOf(BusinessException.class);
       }
   }
   ```

2. **`@DataJpaTest` + `@Import`**: 서비스 로직과 실제 DB 상호작용을 함께 보되,
   컨트롤러/웹 계층 등 나머지는 안 띄우는 중간 지점.
3. **`@SpringBootTest`**: 여러 빈이 실제로 조합돼 동작하는지, 트랜잭션 전파가
   여러 계층에 걸쳐 실제로 맞물리는지 확인해야 할 때. 컨텍스트 전체를 띄우는
   만큼 느리다.

실무에서는 "가능하면 mock 기반 단위 테스트로 커버 → 여러 계층이 얽힌 시나리오만
`@SpringBootTest`로" 순서가 합리적이다. `@SpringBootTest`를 서비스 테스트의
기본값으로 삼으면 테스트 스위트 전체가 느려지는 문제로 돌아온다.

## 정리

| 검증 대상 | 도구 | 실제 DB | Spring 컨텍스트 | 서비스가 프록시로 동작 |
|---|---|---|---|---|
| 서비스 로직만 | 단위 테스트 (Mockito) | 안 씀 | 안 띄움 | 해당 없음 (mock) |
| 리포지토리 쿼리 | `@DataJpaTest` | 씀 (임베디드) | 리포지토리만 | 해당 없음 |
| 서비스 + DB, 좁게 | `@DataJpaTest` + `@Import` | 씀 (임베디드) | 지정한 빈만 | 정상 동작 |
| 전체 배선 | `@SpringBootTest` | 씀 | 전체 | 정상 동작 |

`@DataJpaTest` 안에서 서비스를 `new`로 만드는 건 이 표의 어느 칸에도 속하지 않는
잘못된 조합이다 — 리포지토리 슬라이스의 가벼움과 서비스 테스트라는 목적을 동시에
가지려다, 정작 서비스가 의존하는 AOP 동작(`@Transactional`)만 조용히 빠뜨리게
된다. 테스트하려는 계층에 맞는 도구를 쓰는 것, 그리고 슬라이스를 벗어난 빈이
필요하면 `@Import`로 명시적으로 끌어오는 것 — 이 두 가지만 지키면 피할 수 있는
문제다.
