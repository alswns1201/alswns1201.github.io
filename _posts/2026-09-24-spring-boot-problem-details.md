---
title: "Spring Boot가 지원하는 Problem Details(RFC 7807/9457) 제대로 쓰기"
date: 2026-09-24
categories: [Java/Spring]
---

REST API 에러 응답 포맷은 팀마다, 심지어 같은 프로젝트 안에서도 컨트롤러마다 제각각인
경우가 많다. `{error, message}`, `{code, msg}`, `{status, reason}`... 클라이언트
입장에서는 API마다 다른 에러 스키마를 파싱해야 하는 게 곤혹스럽다. Spring Framework 6
(Spring Boot 3)부터는 이 문제를 표준 스펙으로 풀 수 있는 `ProblemDetail`을 지원한다.

## RFC 7807 / RFC 9457 — 에러 응답의 표준 스키마

Problem Details는 HTTP API의 에러 응답을 위한 IETF 표준(RFC 7807, 2023년에 RFC 9457로
개정)이다. `application/problem+json` 미디어 타입으로, 다음 필드를 정의한다.

```json
{
  "type": "https://example.com/errors/insufficient-funds",
  "title": "Insufficient Funds",
  "status": 400,
  "detail": "계좌 잔액이 부족합니다",
  "instance": "/accounts/12345/transactions"
}
```

- `type` — 문제 유형을 식별하는 URI. 실제로 접근 가능한 문서일 필요는 없지만, 클라이언트가
  에러 종류를 문자열 비교가 아니라 URI로 프로그래밍적으로 구분할 수 있게 해준다.
- `title` — 사람이 읽는 요약. **같은 `type`이면 항상 같은 `title`**이어야 한다는 게
  스펙의 핵심 규칙이다 (요청마다 달라지는 건 `detail` 쪽 책임).
- `status` — HTTP 상태 코드. 응답 헤더의 상태 코드와 일치해야 한다.
- `detail` — 이번 요청에서 실제로 무슨 일이 있었는지에 대한 구체적 설명.
- `instance` — 문제가 발생한 구체적 리소스의 URI.
- 이 다섯 필드 외의 키는 전부 확장 필드(extension member)로 취급된다 — 표준이 필드
  이름만 정의하고, 그 외 정보는 자유롭게 얹을 수 있게 열어둔 구조다.

`type`/`title`이 "이 에러의 분류"를, `detail`/`instance`가 "이번 발생 건의 구체적
정보"를 담당한다는 역할 분리가 이 스펙의 설계 의도다. 커스텀 포맷을 쓰던 팀이 흔히
놓치는 지점이 여기인데, `title`에 매번 다른 문자열을 넣거나 `detail`에 고정 문자열을
넣으면 스펙의 의도를 반쯤 무시하는 셈이 된다.

## Spring의 구현: ProblemDetail과 ErrorResponse

Spring이 이 스펙을 지원하는 핵심 타입은 두 개다.

- **`ProblemDetail`** — RFC 필드를 담는 클래스. `ProblemDetail.forStatus(...)`,
  `ProblemDetail.forStatusAndDetail(...)` 같은 정적 팩토리로 생성한다.
- **`ErrorResponse`** — "이 예외/응답은 `ProblemDetail`로 변환 가능하다"를 나타내는
  인터페이스. `ResponseStatusException`을 비롯해 Spring MVC의 표준 예외들이 이미 이
  인터페이스를 구현하고 있다.

이 둘의 관계가 이해의 핵심이다. `ProblemDetail`은 순수한 데이터 컨테이너고,
`ErrorResponse`는 "나는 ProblemDetail로 변환될 수 있다"는 계약이다. 그래서 커스텀
예외를 `ErrorResponseException`(둘을 잇는 기본 구현체)을 상속해서 만들면, 별도의
`@ExceptionHandler` 없이도 Spring이 알아서 ProblemDetail 응답으로 바꿔준다.

## 활성화

```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

이 설정을 켜면 Spring MVC의 기본 예외 처리 체인(`DefaultHandlerExceptionResolver`)이
`MethodArgumentNotValidException`, `HttpMessageNotReadableException`,
`NoHandlerFoundException` 같은 표준 예외들을 자동으로 ProblemDetail 포맷으로
직렬화한다. 이 설정을 켜지 않으면 예전 방식의 기본 에러 응답(`/error` 엔드포인트가
만드는 `{timestamp, status, error, path}` 형태)이 그대로 유지된다 — 즉 하위 호환을
위해 기본값은 꺼져 있는 옵션이다.

## 사용 패턴 세 가지

### 1. 컨트롤러에서 즉석으로 던지기

```java
@GetMapping("/accounts/{id}")
public Account getAccount(@PathVariable Long id) {
    return accountRepository.findById(id)
        .orElseThrow(() -> new ResponseStatusException(
            HttpStatus.NOT_FOUND, "계좌를 찾을 수 없습니다"));
}
```

`ResponseStatusException`이 이미 `ErrorResponse`를 구현하므로, `problemdetails.enabled`가
켜져 있으면 별도 코드 없이 표준 포맷으로 응답된다.

### 2. `@ExceptionHandler`에서 ProblemDetail을 직접 구성

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InsufficientFundsException.class)
    public ProblemDetail handleInsufficientFunds(InsufficientFundsException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, ex.getMessage());
        problem.setTitle("Insufficient Funds");
        problem.setType(URI.create("https://example.com/errors/insufficient-funds"));
        problem.setProperty("balance", ex.getBalance()); // 확장 필드
        return problem;
    }
}
```

`setProperty()`로 붙인 값은 JSON 직렬화 시 최상위 필드로 펼쳐진다. 유효성 검증
실패처럼 필드별 에러 목록이 필요한 경우, 여기에 `errors: [{field, message}, ...]`
같은 배열을 얹는 패턴이 실무에서 흔히 쓰인다.

### 3. `ResponseEntityExceptionHandler` 상속으로 전역 표준화

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ProblemDetail handleNotFound(EntityNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }
}
```

`ResponseEntityExceptionHandler`를 상속하면, Spring MVC가 던지는 표준 예외들
(`MethodArgumentNotValidException`, `HttpMediaTypeNotSupportedException` 등)에 대한
처리 메서드를 부모가 이미 구현해서 제공한다. 이 메서드들을 오버라이드해서 커스터마이징할
수도 있다. 직접 정의한 예외만 핸들러를 추가하고, 프레임워크가 던지는 예외는 부모가
알아서 ProblemDetail로 통일해주므로, `@ExceptionHandler`를 예외 타입마다 일일이
새로 만들 필요가 줄어든다.

## `@RestControllerAdvice` + `@ExceptionHandler` 방식과 뭐가 다른가

기존에도 `@RestControllerAdvice`로 전역 예외 처리를 표준화하는 건 가능했다 (관련해서
[이전 글](/posts/spring-restcontrolleradvice-exception-handling/)에서 다룬 적 있다).
Problem Details가 더해주는 건 "그 표준화의 결과물 스키마 자체를 표준화"하는 것이다.

- 기존 방식: 팀이 직접 응답 DTO(`ErrorResponseDto` 같은)를 설계하고, 모든
  `@ExceptionHandler`가 그 DTO를 반환하도록 통일한다. 스키마는 팀 재량.
- Problem Details: 스키마 자체가 IETF 표준이고, `application/problem+json` 미디어
  타입으로 응답하므로 클라이언트(특히 여러 조직의 API를 소비하는 클라이언트)가 별도
  문서 없이도 필드 의미를 예측할 수 있다.

즉 "전역에서 처리한다"는 원칙은 그대로고, 그 결과물의 형태를 사내 컨벤션이 아니라
업계 표준에 맞추는 선택지가 하나 생긴 것이다.

## 실무에서 챙길 점

- **`type` URI는 안정적인 식별자로 관리**한다. 실제 문서 페이지로 연결되면 좋지만,
  최소한 같은 에러 종류에 대해 URI가 바뀌지 않아야 클라이언트가 안심하고 분기할 수 있다.
- **`detail`에 내부 구현 정보를 흘리지 않는다.** 스택 트레이스나 SQL 에러 메시지를
  그대로 `detail`에 넣으면 정보 노출 문제가 된다 — 사용자 대상 메시지와 내부 로그는
  분리해야 한다.
- **Content negotiation은 자동**이다. 클라이언트가 `Accept: application/problem+json`을
  명시하지 않아도 Spring이 알아서 이 미디어 타입으로 응답한다.
- **기존 API와의 마이그레이션은 점진적으로.** 이미 커스텀 에러 포맷을 쓰고 있는 API를
  한 번에 바꾸면 프론트엔드/클라이언트 계약이 깨진다. 신규 API부터 적용하거나, 버전
  분기(`/v2`)를 통해 전환하는 편이 안전하다.
- Spring Security 쪽 인증/인가 예외도 최근 버전으로 갈수록 ProblemDetail 통합이
  점점 확대되는 추세라, 인증 실패 응답까지 같은 포맷으로 묶을 수 있는 범위가 넓어지고
  있다.

## 정리

| | 기존 커스텀 에러 DTO | Spring `ProblemDetail` |
|---|---|---|
| 스키마 정의 주체 | 팀/프로젝트 재량 | IETF 표준(RFC 7807/9457) |
| 미디어 타입 | 보통 `application/json` | `application/problem+json` |
| 프레임워크 예외 자동 변환 | 직접 `@ExceptionHandler` 작성 필요 | `problemdetails.enabled`만으로 표준 예외 자동 변환 |
| 확장 필드 | DTO에 필드 추가 | `setProperty()`로 자유롭게 확장 |
| 클라이언트 상호운용성 | API마다 별도 학습 필요 | 표준 스키마라 여러 API에 재사용 가능 |

`ProblemDetail`이 해결하는 건 "예외를 어디서 잡을 것인가"의 문제가 아니라 "잡은
다음 그 결과를 어떤 모양으로 내려줄 것인가"의 문제다. 전역 처리 구조(`@RestControllerAdvice`,
지역/전역 우선순위 등)는 그대로 두고, 그 위에 응답 스키마만 표준으로 교체한다고
생각하면 기존 예외 처리 설계와 자연스럽게 이어진다.
