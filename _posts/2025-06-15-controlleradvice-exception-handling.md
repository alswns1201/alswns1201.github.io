---
title: "try-catch 지옥에서 벗어나기: @ControllerAdvice로 우아하게 예외 처리하기"
date: 2025-06-15
categories: [개발일지, 예외처리]
tags: [spring-boot, architecture]
---

서비스 계층에서 예외를 직접 try-catch로 잡아 처리하는 코드는 언뜻 안전해 보이지만,
사실은 문제가 있다. 왜 그런지부터 짚고, 대안이 왜 더 나은지 설명한다.

## try-catch를 서비스에서 직접 잡으면 안 되는 이유

```java
// 나쁜 예: Service 계층에서 모든 것을 처리하려는 경우
@Service
public class CrewService {
    public void createCrew(CrewCreateDto dto) {
        try {
            if (crewRepository.existsByName(dto.getName())) {
                throw new IllegalStateException("이미 존재하는 크루 이름입니다.");
            }
            crewRepository.save(newCrew);
        } catch (IllegalStateException e) {
            log.error("크루 생성 실패: {}", e.getMessage());
            // 사용자에게 보낼 메시지를 직접 반환하거나, null을 반환...
            // 여기서 뭘 해야 할지 애매해집니다. 서비스가 HTTP 응답을 고민하기 시작합니다.
        }
    }
}
```

문제는 세 가지다.

- **책임의 모호함**: 서비스의 책임은 "비즈니스 로직 수행"인데, catch 블록 안에서
  "사용자에게 뭐라고 보여줄지"까지 고민하게 된다. 이건 HTTP 계층의 책임이지
  비즈니스 계층의 책임이 아니다.
- **코드 오염**: 메서드마다 try-catch가 반복되면 진짜 비즈니스 로직이 예외 처리
  코드에 파묻힌다.
- **트랜잭션이 조용히 깨진다**: 이게 가장 위험하다. 서비스 안에서 예외를 catch해
  삼켜버리면, 스프링의 `@Transactional`은 그 메서드가 정상 종료했다고 판단해
  **롤백하지 않는다.** 즉 "예외를 잡아서 로깅만 했을 뿐인데" DB에는 일부만 반영된
  상태로 커밋되는 사고가 날 수 있다. 예외를 서비스 안에서 삼키는 순간 트랜잭션의
  안전망이 사라진다는 걸 알아야 한다.

## 계층별로 역할을 나눈다

- **Service**: 예외를 만들어서 던지기만 한다.
- **Controller**: 아무것도 하지 않는다. 그냥 통과시킨다.
- **`@ControllerAdvice`**: 던져진 모든 예외를 한 곳에서 받아 처리한다.

### 1) Service — 커스텀 예외를 정의하고 던지기만

```java
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;
}

@Getter
@RequiredArgsConstructor
public enum ErrorCode {
    CREW_NOT_FOUND(HttpStatus.NOT_FOUND, "해당 크루를 찾을 수 없습니다."),
    CREW_NAME_DUPLICATED(HttpStatus.CONFLICT, "이미 존재하는 크루 이름입니다.");

    private final HttpStatus status;
    private final String message;
}

@Service
public class AuthService {
    @Transactional
    public User registerOrLoginUser(...) {
        crewRepository.findByName(crewNameToCreate).ifPresent(c -> {
            throw new BusinessException(ErrorCode.CREW_NAME_DUPLICATED);
        });
    }
}
```

`ErrorCode`를 enum으로 미리 정의해두는 이유는, 예외 상황과 그에 대응하는 HTTP
상태 코드·메시지를 한 곳에서 관리하기 위해서다. 새로운 실패 케이스가 생기면
enum에 상수 하나만 추가하면 되고, 서비스 코드는 예외를 어떻게 표현할지 고민할
필요 없이 이미 정의된 `ErrorCode`를 골라 던지기만 하면 된다.

### 2) Controller — 예외를 그냥 통과시킨다

```java
@RestController
public class AuthController {
    @PostMapping("/login/{provider}")
    public ResponseEntity<String> socialLogin(...) {
        String jwtToken = authService.login(requestDto);
        return ResponseEntity.ok(jwtToken);
    }
}
```

서비스에서 예외가 발생하면 이 메서드는 그 즉시 중단되고 예외는 바깥으로 전파된다.
컨트롤러가 예외 처리에 관여하지 않는다는 게 핵심이다.

### 3) `@RestControllerAdvice` — 중앙에서 한 번에 처리

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException ex) {
        log.error("비즈니스 예외 발생: {}", ex.getMessage());
        ErrorCode errorCode = ex.getErrorCode();
        ErrorResponse response = new ErrorResponse(errorCode.name(), errorCode.getMessage());
        return new ResponseEntity<>(response, errorCode.getStatus());
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception ex) {
        log.error("처리되지 않은 예외 발생", ex);
        ErrorResponse response = new ErrorResponse("INTERNAL_SERVER_ERROR", "서버 내부 오류입니다.");
        return new ResponseEntity<>(response, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

`BusinessException`을 잡는 핸들러와, 그 외 모든 예외(NPE 등 예상 못 한 것)를
잡는 fallback 핸들러를 분리해 둔 것도 의도적이다. 전자는 "우리가 예상한 실패"라서
사용자에게 구체적인 메시지를 줄 수 있고, 후자는 "예상 못 한 실패"라서 내부 정보를
노출하지 않고 일반화된 500 응답만 준다. 이 구분이 없으면 스택트레이스나 내부
구현 디테일이 그대로 클라이언트에 노출되는 보안 문제로 이어질 수 있다.

이 구조의 진짜 이점은 "예외 처리 로직이 한 곳에 모인다"는 것 자체보다,
**서비스 계층이 트랜잭션 롤백 문제로부터 자유로워진다**는 데 있다 — 서비스는
예외를 삼키지 않고 던지기만 하므로, `@Transactional`이 예외를 정상적으로 감지해
롤백을 수행한다.

## 번외: `enum.name()`

```java
ErrorCode.CREW_NOT_FOUND.name(); // "CREW_NOT_FOUND"
```

모든 enum은 `java.lang.Enum`을 상속하므로 별도 구현 없이 `name()`을 쓸 수 있다.
API 응답의 `code` 필드처럼 enum 상수 자체를 문자열 식별자로 노출할 때 유용하다.

## 정리

1. **Service**: 비즈니스 예외 발생 시 Custom Exception을 던지기만 한다.
2. **Controller**: 예외를 무시하고 통과시킨다.
3. **`@ControllerAdvice`**: 모든 예외를 중앙에서 받아 일관된 에러 응답으로 변환한다.

이 구조가 지키는 가장 중요한 불변식은 "서비스 계층에서 예외를 절대 삼키지
않는다"는 것 — 그래야 트랜잭션 롤백과 에러 응답 일관성이 둘 다 보장된다.
