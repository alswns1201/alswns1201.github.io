---
title: "@Bean, @Component vs @Configuration: 빈 등록 방식의 차이와 함정"
date: 2025-05-23
categories: [Java/Spring]
---

스프링 컨테이너는 빈(Bean)의 생명주기를 관리하고 필요한 곳에 주입해준다. 문제는 빈을
"어떻게 컨테이너에 등록시키는가"이고, 여기에 @Component와 @Bean/@Configuration이라는
두 가지 서로 다른 방식이 존재한다. 둘의 차이를 아는 것보다 중요한 건, **왜 두 가지
방식이 따로 필요한가**다.

## @Component — "이 클래스는 내가 관리하는 컴포넌트다"

```java
@Component
public class MyComponent { }

@Service // 내부적으로 @Component를 포함
public class MyService { }
```

클래스패스 스캔 대상이 되어 자동으로 빈으로 등록된다. @Service, @Repository,
@Controller는 모두 @Component의 특수화(stereotype)일 뿐이다.

**한계**: 이 방식은 **내가 소스 코드를 가진 클래스**에만 쓸 수 있다. 외부 라이브러리
클래스에 `@Component`를 붙일 수는 없다 — 소스를 수정할 권한이 없기 때문이다.

## @Configuration + @Bean — "내가 직접 만들어서 등록한다"

```java
@Configuration
public class CodefConfig {

    @Bean
    public EasyCodef makeEasyCodef() {
        EasyCodef codef = new EasyCodef();
        // 초기화 로직
        return codef;
    }
}
```

메소드가 반환하는 객체를 빈으로 등록한다. 외부 라이브러리 클래스를 빈으로 만들거나,
생성 과정에 조건/커스텀 로직이 필요할 때 쓴다.

## 두 방식을 가르는 진짜 기준

"내 클래스냐 아니냐"가 표면적인 기준이라면, 좀 더 실무적인 기준은 이거다:
**빈 생성 로직에 조건 분기나 외부 설정값이 필요한가?** @Component는 클래스에
어노테이션 하나 붙이는 것 외에 생성 과정에 개입할 여지가 없다. 반면 @Bean 메소드는
평범한 자바 메소드이므로, 조건에 따라 다른 구현체를 반환하거나(팩토리 패턴),
`@Value`로 주입받은 설정값으로 초기화 로직을 분기하는 게 자유롭다.

## 놓치기 쉬운 함정: @Configuration의 CGLIB 프록시와 싱글톤 보장

여기서부터가 원문에 없는, 실무에서 실제로 자주 걸리는 지점이다. 한 `@Configuration`
클래스 안에서 다른 `@Bean` 메소드를 호출하면 어떻게 될까?

```java
@Configuration
public class AppConfig {

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // ServiceB를 직접 메소드 호출로 얻음
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

얼핏 보면 `serviceB()`를 호출할 때마다 새 인스턴스가 생성될 것 같지만, 실제로는
**컨테이너가 관리하는 싱글톤 `ServiceB` 빈이 반환된다.** 이게 가능한 이유는
`@Configuration` 클래스가 스프링에 의해 **CGLIB 프록시로 감싸지기** 때문이다.
프록시가 `serviceB()` 호출을 가로채서 "이미 등록된 빈이 있으면 그걸 반환하고, 없으면
생성 후 등록"하는 식으로 동작한다.

**여기서 실제로 사고가 나는 지점**: `@Configuration(proxyBeanMethods = false)`로
설정하거나, 애초에 `@Component`(즉 `@Configuration`이 아닌 일반 설정 클래스처럼
동작하는 "lite mode")를 쓰면 이 프록시가 적용되지 않는다. 이 상태에서 위 코드처럼
`serviceB()`를 직접 호출하면 **매번 새 인스턴스가 생성**되고, 싱글톤을 기대했던
다른 빈들과 상태가 어긋난다. 스프링 부트 최신 버전은 성능상의 이유로
`proxyBeanMethods = false`를 권장하는 흐름인데, 이 설정을 켜둔 채로 빈 간 메소드
호출 패턴을 그대로 쓰면 조용히 버그가 생긴다 — 컴파일도 되고 당장 에러도 안 나서
발견하기 더 어렵다.

**안전한 패턴**: 빈 간 의존성이 필요하면 메소드 호출 대신 파라미터로 받는다.

```java
@Bean
public ServiceA serviceA(ServiceB serviceB) { // 컨테이너가 주입
    return new ServiceA(serviceB);
}
```

이렇게 하면 `proxyBeanMethods` 설정과 무관하게 항상 컨테이너가 관리하는 빈을
정확히 주입받는다.

## 정리

| | @Component | @Configuration + @Bean |
|---|---|---|
| 대상 | 내 소스 코드 클래스 | 외부 라이브러리 포함 모든 객체 |
| 등록 방식 | 클래스패스 스캔 (선언적) | 메소드 실행 (명령형) |
| 생성 로직 개입 | 불가 | 자유로움 (조건부, 팩토리 등) |
| 빈 간 의존성 | 생성자/필드 주입 | 파라미터로 받는 것을 권장 (메소드 직접 호출은 프록시 설정에 따라 함정) |

두 방식 중 뭘 쓸지 고민하기 전에, "이 객체의 생성 과정에 내가 개입해야 하는가"를
먼저 묻는 게 실질적인 판단 기준이다.
