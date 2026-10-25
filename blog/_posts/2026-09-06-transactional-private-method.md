---
title: "private 메서드에 @Transactional을 걸면 안 되는 이유"
date: 2026-09-06
categories: [Java/Spring]
---

`@Transactional`을 private 메서드에 걸어도 컴파일은 되고, 앱도 정상적으로 뜬다.
그런데 실제로는 트랜잭션이 전혀 적용되지 않는다 — 예외도, 경고도 없이 조용히
무시된다는 게 이 문제를 위험하게 만든다. 왜 이런 일이 생기는지 프록시 동작 원리부터
따라가 본다.

## 프록시 기반 AOP — @Transactional의 실제 동작 방식

스프링의 `@Transactional`은 기본적으로 **프록시**로 구현된다. 컨테이너는 `@Transactional`이
붙은 빈을 그대로 등록하지 않고, 그 빈을 감싸는 프록시 객체를 대신 등록한다. 외부에서
그 빈을 호출하면 실제로는 프록시가 먼저 호출을 받고, 트랜잭션을 시작한 뒤 원본
메서드를 호출하고, 성공하면 커밋, 예외가 터지면(기본은 unchecked 예외 기준) 롤백한다.

프록시를 만드는 방식은 두 가지다.

- **JDK 동적 프록시**: 대상 빈이 인터페이스를 구현하고 있으면, 그 인터페이스를 구현하는
  프록시 객체를 런타임에 생성한다.
- **CGLIB**: 인터페이스가 없으면, 대상 클래스를 **상속**하는 서브클래스를 만들어서
  프록시로 쓴다.

두 방식 다 공통점이 있다 — 프록시가 원본 메서드를 "가로채려면" 그 메서드를
오버라이드하거나(CGLIB) 인터페이스로 위임할 수 있어야(JDK 프록시) 한다. 즉 프록시가
개입할 수 있는 지점은 **오버라이드 가능한 메서드**로 한정된다.

## private 메서드가 원천적으로 안 되는 이유

`private` 메서드는 애초에 오버라이드가 불가능하다 (자바 언어 규칙상 상속되지도
않는다). CGLIB 서브클래스든 뭐든, private 메서드를 가로챌 방법 자체가 없다 — 어드바이스를
끼워 넣을 지점이 없는 거다.

여기서 더 중요한 포인트가 하나 있다. `public` 메서드의 "self-invocation 문제"와
비교해보면 왜 private가 구조적으로 더 나쁜지 보인다.

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        // ... 검증 로직
        saveWithTransaction(order); // (1) self-invocation
    }

    @Transactional
    public void saveWithTransaction(Order order) {
        // 저장 로직
    }
}
```

`saveWithTransaction`이 `public`이라도, `placeOrder` 안에서 `this.saveWithTransaction()`으로
부르면 트랜잭션이 안 걸린다. 이유는 (1)의 호출이 **프록시를 거치지 않고 원본 객체의
`this`를 그대로 쓰기 때문**이다. 외부에서 컨테이너가 주입한 프록시를 통해
`saveWithTransaction`을 직접 호출하면(예: 다른 빈이 `orderService.saveWithTransaction()`을
호출) 트랜잭션이 정상적으로 걸린다. 즉 public self-invocation 문제는 **호출 경로에
따라 걸리기도 하고 안 걸리기도 하는, 상황 의존적인 버그**다.

`private` 메서드는 이 스펙트럼의 극단이다. 자바 언어 자체가 클래스 밖에서 private
메서드를 호출하는 걸 막기 때문에, 그 메서드로 들어가는 경로는 **오직 같은 클래스
내부의 self-invocation 하나뿐**이다. 외부에서 프록시를 통해 도달할 수 있는 경로가
아예 존재하지 않는다. 그러니 "가끔은 걸린다"가 성립할 여지가 없고, **항상, 구조적으로
무효**다.

## 조용히 무시된다는 게 진짜 문제

스프링은 이 상황에서 컴파일 에러도, 앱 기동 실패도, 런타임 예외도 던지지 않는다.
애노테이션은 그냥 아무 효과 없이 무시된다. (버전에 따라 로그에 경고가 남을 수도
있지만, 이건 확정적으로 기대할 수 있는 동작은 아니다.) 그래서 실무에서 흔히
이런 식으로 걸린다.

```java
@Service
public class PaymentService {

    public void processPayment(Payment payment) {
        validate(payment);
        chargeAndRecord(payment); // 트랜잭션이 필요한 로직인데...
    }

    @Transactional
    private void chargeAndRecord(Payment payment) {
        paymentRepository.save(payment);
        ledgerRepository.record(payment);
        // 여기서 예외가 나도 위 save()가 롤백되지 않는다
    }
}
```

"트랜잭션 안에서 여러 저장 로직을 묶고 싶다"는 의도로 private 헬퍼 메서드를 만들고
거기에 `@Transactional`을 붙이는 건 자연스러운 시도처럼 보이지만, 정확히 반대
효과를 낸다. 개발 중 정상 케이스만 테스트하면 이 버그는 드러나지 않고, 운영에서
중간에 예외가 나야 "왜 부분 커밋이 됐지"로 발견된다.

## 해결 방법과 트레이드오프

- **별도 빈으로 분리**: `chargeAndRecord`를 `public`으로 바꾸고 별도의 빈
  (`PaymentTransactionExecutor` 같은)으로 빼서, 호출하는 쪽이 그 빈을 주입받아
  프록시를 통해 호출하게 한다. 가장 정석적이고 스프링 관용구에 맞는 방법이지만,
  클래스 수가 늘어난다.
- **`AopContext.currentProxy()`**: 같은 클래스 안에서 `((OrderService)
  AopContext.currentProxy()).saveWithTransaction()`처럼 현재 프록시를 얻어와
  self-invocation을 프록시 경유 호출로 바꾼다. 클래스 분리 없이 해결되지만,
  `exposeProxy = true` 설정이 필요하고 AOP 프록시 존재를 코드가 직접 알아야 해서
  결합도가 올라간다.
- **AspectJ 컴파일/로드타임 위빙**: `@EnableTransactionManagement(mode =
  AdviceMode.ASPECTJ)`처럼 프록시가 아니라 바이트코드 자체를 위빙하는 방식을 쓰면
  private 메서드에도 `@Transactional`이 실제로 적용된다. 근본적인 해결책이지만
  빌드 설정이 복잡해지고, 실무에서 프록시 방식 대신 이걸 쓰는 경우는 드물다.

가장 실용적인 선택은 첫 번째다 — private 메서드에 트랜잭션을 걸고 싶다는 요구
자체가 "이 로직은 사실 별도 책임으로 분리돼야 한다"는 신호로 보는 게 낫다.

## 정리

| | public self-invocation | private + @Transactional |
|---|---|---|
| 트랜잭션 적용 여부 | 호출 경로에 따라 다름 (상황 의존적) | 항상 무효 (구조적) |
| 외부에서 프록시 경유 호출 가능? | 가능 | 애초에 불가능 (언어 레벨 제약) |
| 원인 | 프록시가 아닌 raw `this`로 호출 | 오버라이드 불가 → 프록시 개입 지점 자체가 없음 |
| 발견 난이도 | 특정 호출 경로에서만 재현 | 항상 재현되지만 예외 없이 조용히 무시됨 |

`@Transactional`이 프록시 기반이라는 사실 하나만 기억하면, private 메서드 문제와
self-invocation 문제가 결국 같은 원인(프록시를 거치지 않는 호출)에서 나온
두 가지 증상이라는 게 보인다. 트랜잭션이 필요한 로직은 항상 "프록시를 통해 도달
가능한 public 메서드"에 둬야 한다는 원칙 하나로 정리된다.
