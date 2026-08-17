---
title: "Spring AOP로 Controller/Service 공통 로깅 구현하기"
date: 2025-10-29
categories: [Spring Boot]
tags: [spring-boot, monitoring, architecture]
---

서비스가 커질수록 Controller나 Service 단에서 "요청이 들어왔는가", "정상적으로
리턴됐는가", "예외가 발생했는가" 같은 공통 관심사(Cross-Cutting Concern)를 반복해서
작성하게 된다. 이걸 매번 직접 코드로 박아넣으면 유지보수성이 떨어지고, 실수로
어딘가에 로그를 빠뜨리기도 쉽다. **AOP(Aspect-Oriented Programming)**는 이 반복을
한 곳으로 모으는 도구다.

## AOP가 실제로 하는 일: 프록시

AOP를 "마법처럼 코드 사이에 끼어드는 것"으로 이해하면 나중에 디버깅할 때 헤맨다.
실제로는 스프링이 대상 빈을 감싸는 **프록시 객체**를 만들고, 그 프록시가 원본 메서드
호출 전후로 Advice를 끼워 넣는 구조다. 이걸 알아야 두 가지 실무 함정을 피할 수 있다.

- **자기 호출(self-invocation) 문제**: 같은 클래스 내부에서 `this.otherMethod()`로
  호출하면 프록시를 거치지 않으므로 AOP가 적용되지 않는다. Controller 안에서
  `@RestController` 클래스의 다른 public 메서드를 내부적으로 호출하는 구조를 짜면
  로그가 안 찍히는 원인이 대부분 이거다.
- **인터페이스 유무에 따른 프록시 방식**: 대상 빈이 인터페이스를 구현하면 JDK 동적
  프록시, 아니면 CGLIB(클래스 상속) 프록시를 쓴다. `final` 클래스/메서드는 CGLIB로
  프록시를 만들 수 없어 AOP가 조용히 무시된다.

## 용어 정리

| 용어 | 의미 |
|---|---|
| Aspect | 공통 관심사를 모듈화한 클래스 (`@Aspect`) |
| Join Point | AOP가 적용될 수 있는 지점 (메서드 실행 등) |
| Pointcut | 실제로 적용할 메서드를 고르는 조건식 |
| Advice | Before/AfterReturning/AfterThrowing/Around 시점에 실행되는 코드 |

## Controller 단 공통 로그

```java
@Aspect
@Component
public class LoggingAspect {

    private static final Logger logger = LoggerFactory.getLogger(LoggingAspect.class);

    @Before("@within(org.springframework.web.bind.annotation.RestController)")
    public void logBefore(JoinPoint joinPoint) {
        logger.info("[ENTER] {}.{}()",
                joinPoint.getSignature().getDeclaringTypeName(),
                joinPoint.getSignature().getName());
    }

    @AfterReturning(pointcut = "@within(org.springframework.web.bind.annotation.RestController)", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        logger.info("[RETURN] {}.{}() => {}",
                joinPoint.getSignature().getDeclaringTypeName(),
                joinPoint.getSignature().getName(), result);
    }

    @AfterThrowing(pointcut = "@within(org.springframework.web.bind.annotation.RestController)", throwing = "ex")
    public void logAfterThrowing(JoinPoint joinPoint, Throwable ex) {
        logger.error("[EXCEPTION] {}.{}() => {}",
                joinPoint.getSignature().getDeclaringTypeName(),
                joinPoint.getSignature().getName(), ex.getMessage(), ex);
    }
}
```

`@Before` + `@AfterReturning` + `@AfterThrowing`을 조합으로 쓴 이유는 각 Advice가
"관찰만" 하고 흐름을 바꾸지 않기 때문이다. `@AfterThrowing`은 예외를 잡아먹지 않고
그대로 다시 던진다 — 로그를 남기되 실제 예외 처리(`@ControllerAdvice` 등)는 그대로
동작해야 하므로, 이 지점에서 예외를 삼키면 안 된다.

**주의할 점 두 가지**

1. `@AfterReturning`에서 `result`를 그대로 로그로 찍으면, 응답 DTO에 비밀번호·토큰
   같은 민감정보가 섞여 있을 때 그대로 로그에 남는다. 운영 환경에서는 민감 필드를
   마스킹하거나 별도 로깅 DTO를 쓰는 게 안전하다.
2. 모든 Controller 메서드에 로그를 찍는 건 트래픽이 커지면 로그 볼륨 자체가
   부담된다. 헬스체크·정적 리소스처럼 자주 호출되는 엔드포인트는 포인트컷에서
   제외하거나 로그 레벨을 낮추는 걸 고려해야 한다.

## Service 단 세밀 제어: 커스텀 어노테이션

Controller는 "들어왔다/나갔다"만 공통으로 찍으면 충분하지만, Service 레이어는
메서드마다 반환값을 찍을지, 요청 ID를 연동할지가 다르다. 커스텀 어노테이션으로
선택적으로 적용한다.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface MethodLogging {
    boolean isLoggingValue() default false;
}
```

```java
@Aspect
@Component
@Slf4j
public class MethodLoggingAspect {

    @Around("@annotation(com.example.common.aop.MethodLogging) && @annotation(aop)")
    public Object methodLoggingAround(ProceedingJoinPoint joinPoint, MethodLogging aop) throws Throwable {

        String requestId = null;

        try {
            HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
            HttpServletResponse response = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getResponse();
            requestId = response.getHeader("X-B3-TraceId");
            log.info("Req ID:[{}] URL:[{}] Params:[{}]",
                    requestId, request.getRequestURI(),
                    Arrays.stream(joinPoint.getArgs())
                            .map(String::valueOf)
                            .collect(Collectors.joining(", ")));
        } catch (Exception e) {
            log.warn("Failed to extract trace info: {}", e.getMessage());
        }

        try {
            Object proceed = joinPoint.proceed();

            if (aop.isLoggingValue()) {
                log.info("Res ID:[{}] Value:[{}]", requestId, proceed);
            }

            return proceed;
        } catch (Exception e) {
            log.error("Error after ID:[{}] message:[{}]", requestId, e.getMessage());
            throw e;
        }
    }
}
```

`@Around`를 쓴 이유는 Before/AfterReturning/AfterThrowing 세 Advice를 조합할
필요 없이 메서드 실행 전체(진입·반환·예외)를 한 곳에서 제어할 수 있어서다. 대신
`joinPoint.proceed()`를 반드시 호출하고 그 결과를 반환해야 한다는 책임이 개발자에게
넘어온다 — 실수로 `proceed()` 결과를 버리고 다른 값을 리턴하면 실제 비즈니스 로직
결과가 조용히 무시된다.

`RequestContextHolder`로 HTTP 요청 정보에 접근하는 부분은 웹 요청 스레드 안에서만
동작한다. `@Async`로 비동기 실행되는 메서드나 배치 스레드에서 같은 Aspect를 재사용하면
`RequestContextHolder.getRequestAttributes()`가 `null`을 반환해 `NullPointerException`이
날 수 있다 — 그래서 이 코드에서도 `try-catch`로 감싸고 실패해도 로깅 자체는 계속
진행되도록 만들어 놨다.

```java
@MethodLogging(isLoggingValue = true)
public UserResponse getUserInfo(Long userId) {
    return userService.findById(userId);
}
```

## 정리

Controller에는 공통 진입 로그를, Service에는 어노테이션 기반 선택적 로그를 —
계층마다 로깅 전략을 다르게 가져가는 이유는 계층마다 "무엇을 알고 싶은가"가 다르기
때문이다. Controller는 트래픽 전체의 흐름을, Service는 특정 로직의 입출력을 본다.
같은 AOP라도 포인트컷 범위와 Advice 종류를 계층 목적에 맞춰 고르는 게 핵심이다.
