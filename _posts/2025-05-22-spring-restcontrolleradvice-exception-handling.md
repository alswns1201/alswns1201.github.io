---
title: "Spring 예외 처리: @ExceptionHandler와 @RestControllerAdvice 제대로 나누기"
date: 2025-05-22
categories: [개발일지, 예외처리]
tags: [spring-boot, exception-handling, java]
---

예외 처리를 "어디서 잡을 것인가"는 생각보다 설계 질문이다. 컨트롤러 안에서 잡을지,
전역으로 잡을지, 서비스 계층에서 잡을지에 따라 코드 구조 자체가 달라진다.

## Error와 Exception은 다른 층위의 문제다

`java.lang.Error`의 하위 클래스(`OutOfMemoryError` 등)는 JVM이 던지는 것이고,
애플리케이션 코드로 복구할 수 없는 상황을 뜻한다. 그래서 이건 "잡아서 처리"하는
대상이 아니다 — 잡아봤자 할 수 있는 게 없다. 예외 처리 논의는 항상 `Exception`
계열에 한정된다.

체크 예외(Checked)와 언체크 예외(RuntimeException, Unchecked)의 구분도 마찬가지로
"복구 가능성"의 문제다. 체크 예외는 파일 I/O, DB 연결처럼 **외부 시스템과의 상호작용에서
발생할 수 있는, 호출자가 미리 대비해야 하는 상황**이라 컴파일러가 처리를 강제한다.
언체크 예외는 잘못된 인자, null 참조처럼 **코드 결함**에 가깝다 — 컴파일러가 강제하지
않는 이유는, 애초에 이건 "처리"가 아니라 "수정"의 대상이기 때문이다.

실무에서 커스텀 예외를 `RuntimeException`을 상속해서 만드는 경우가 많은데, 이건
"이 예외는 호출자가 매번 catch를 강제당하지 않아도 된다"는 설계 선택이다. 대신 그만큼
어디서 이 예외가 터질 수 있는지를 문서나 명명 규칙으로 드러내야 한다 — 컴파일러가
안 잡아주는 걸 사람이 잡아야 하기 때문이다.

## 예외는 어디서 터질 수 있는가: Filter와 DispatcherServlet의 경계

Spring MVC 요청은 `Filter` → `DispatcherServlet` → `Controller/Service/Repository`
순으로 흐른다. 이 경계가 중요한 이유는, **`@ExceptionHandler`와 `@ControllerAdvice`는
`DispatcherServlet` 내부에서 발생한 예외만 잡는다**는 것이다. `Filter` 단계(예: 인증
필터, CORS 필터)에서 던진 예외는 `DispatcherServlet`에 도달하기 전에 발생하므로
`@RestControllerAdvice`로 처리되지 않는다. 이 경계를 모르면 "왜 전역 예외 핸들러를
만들었는데 이 예외는 안 잡히지?"라는 상황을 겪는다 — Filter 단의 예외는 Filter 안에서
직접 try-catch로 응답을 만들거나, `HandlerExceptionResolver`를 별도로 연결해야 한다.

## 지역 처리 vs 전역 처리: 우선순위 규칙

```java
@RestController
public class UserController {
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<?> handleEntityNotFound(EntityNotFoundException exception) {
        Map<String, String> errors = Map.of("message", "Entity Not Found in UserController");
        return new ResponseEntity<>(errors, HttpStatus.NOT_FOUND);
    }
}
```

컨트롤러 내부에 이렇게 지역 핸들러를 두면, 같은 예외에 대해 전역 `@RestControllerAdvice`가
있어도 **지역 핸들러가 우선 적용**된다. 이 우선순위가 실무에서 중요한 이유는, 대부분의
예외는 전역으로 통일된 응답 포맷을 쓰되, 특정 컨트롤러만 다른 응답이 필요할 때(예:
파일 업로드 API는 에러 메시지에 특정 필드가 더 필요하다든가) 지역 핸들러로 예외적으로
오버라이드할 수 있다는 뜻이기 때문이다. 반대로, 이 우선순위를 모르고 지역 핸들러를
남발하면 "왜 전역 규칙이 안 먹히지"라는 디버깅 시간을 낭비하게 된다.

## 전역 처리: @RestControllerAdvice

```java
@RestControllerAdvice
@Log4j2
public class APIControllerAdvice {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<?> handleArgsException(MethodArgumentNotValidException exception) {
        Map<String, Object> errors = new HashMap<>();
        exception.getBindingResult().getFieldErrors().forEach(fieldError ->
            errors.put(fieldError.getField(), fieldError.getDefaultMessage()));
        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }
}
```

`@RestControllerAdvice`는 `@ControllerAdvice` + `@ResponseBody`다. 그래서
`@Controller` 선언이 따로 필요 없다. 둘의 실질적 차이는 응답 형태다:

- `@RestControllerAdvice` — JSON을 반환하는 REST API용. 메서드 반환값이 그대로
  응답 바디가 된다.
- `@ControllerAdvice` — HTML 뷰를 반환하는 전통적인 MVC 컨트롤러용. 반환값이
  뷰 이름으로 해석된다.

REST API 프로젝트에서 `@ControllerAdvice`를 잘못 쓰면, 에러 응답이 JSON이 아니라
뷰 이름을 찾다가 실패하는 이상한 결과로 이어진다 — 에러 상황에서 또 다른 에러가
발생하는 셈이다.

## 어디서 예외를 잡아야 하는가: 서비스 vs 컨트롤러

```java
public TodoDTO read(Long mno) {
    Optional<TodoDTO> result = todoRepository.getDTO(mno);
    TodoDTO todoDTO = result.orElseThrow(() -> new EntityNotFoundException("todo not found"));
    return todoDTO;
}
```

이 예시처럼 "존재하지 않으면 예외를 던진다"는 서비스 계층에서 하는 게 자연스럽다 —
컨트롤러는 이 데이터가 어디서 왔는지, 왜 없을 수 있는지 알 필요가 없다. 원칙을 정리하면:

- **예외는 발생 가능성이 있는 가장 가까운 곳에서 판단**하되(여기가 "존재하지 않는다"를
  가장 잘 아는 곳이므로), **처리(응답 변환)는 전역에서** 한다. 판단과 처리를 분리하는
  것이 핵심이다.
- 컨트롤러에서 서비스의 예외를 다시 try-catch로 감싸는 건 대개 관심사 침범이다 —
  컨트롤러가 서비스 내부 실패 사유까지 알아야 할 이유가 없다.
- `@ControllerAdvice`는 **예상치 못했거나, 여러 컨트롤러에 공통되는** 예외에 쓴다.
  특정 API에만 있는 특이 케이스까지 전역으로 몰아넣으면 전역 핸들러가 점점
  if-else 덩어리가 된다.

## 실무에서 자주 놓치는 지점: 예외 타입의 세분화

`MethodArgumentNotValidException`과 `MethodArgumentTypeMismatchException`처럼
Spring이 이미 세분화해서 던져주는 예외를 뭉뚱그려 하나의 `Exception.class` 핸들러로
잡으면, 클라이언트에게 "정확히 뭐가 잘못됐는지"를 전달할 수 없다. 유효성 검증 실패
(400, 필드별 메시지)와 인증 실패(401)와 서버 내부 오류(500)는 근본적으로 다른
클라이언트 대응을 요구하므로, 예외 타입을 세분화해서 매핑하는 만큼 API 사용성이
좋아진다. 반대로 너무 잘게 쪼개면 `@ExceptionHandler` 메서드가 수십 개로 늘어나는데,
이 균형점은 "클라이언트가 이 에러를 보고 다르게 행동해야 하는가"를 기준으로 잡는
게 실용적이다 — 클라이언트 대응이 같다면 같은 핸들러로 묶어도 된다.
