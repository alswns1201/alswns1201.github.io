---
title: "Spring Cloud OpenFeign으로 외부 API 호출하기 — 인터셉터와 에러 디코더가 진짜 핵심인 이유"
date: 2026-01-11
categories: [아키텍처]
tags: [msa, spring-cloud, api-design]
---

*(같은 "선언형 HTTP 클라이언트"를 다루는 [Spring HttpInterface](/posts/spring-http-interface/) 글과
겹치는 부분은 짧게 넘기고, Feign에만 있는 인증/에러 처리 구조에 집중했다)*

RestTemplate으로 외부 API를 직접 호출하면 URL이 코드에 흩어지고, 인증 헤더 같은
공통 설정이 호출부마다 중복된다. Feign은 이 문제를 "인터페이스 선언"으로 해결한다 —
Spring의 HttpInterface와 접근 자체는 비슷하지만, Feign은 **Spring Cloud 생태계
(서비스 디스커버리, 로드밸런싱)와 통합된 인터셉터/에러 디코더 체계**를 갖추고
있다는 게 실무에서 갈리는 지점이다.

## 기본 구조: URL만 알면 메서드 호출처럼

```java
@FeignClient(
    name = "userApiClient",
    url = "${external.api.user}"
)
public interface UserApiClient {

    @GetMapping("/users/{id}")
    UserResponse getUser(@PathVariable("id") Long id);
}
```

구현 클래스는 없다. `@EnableFeignClients`가 켜져 있으면 Spring이 런타임에 프록시를
만들어준다. 서비스는 이게 HTTP 클라이언트인지 몰라도 된다.

```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserApiClient userApiClient;

    public UserResponse getUser(Long id) {
        return userApiClient.getUser(id);
    }
}
```

## 공통 헤더/인증: RequestInterceptor로 한 곳에 모은다

실무에서 진짜 중요한 부분은 여기부터다. 매 API 호출마다 인증 토큰, 추적 ID(trace
ID) 같은 공통 헤더를 넣어야 한다면, 이걸 각 인터페이스 메서드마다 파라미터로
받게 만들면 호출부마다 실수로 빠뜨리기 쉽다.

```java
@Configuration
public class FeignConfig {

    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            requestTemplate.header("Content-Type", "application/json");
            requestTemplate.header("Authorization", "Bearer TOKEN");
        };
    }
}

@FeignClient(
    name = "secureApi",
    url = "${external.api.secure}",
    configuration = FeignConfig.class
)
public interface SecureApiClient {
}
```

`RequestInterceptor`는 **이 Feign 클라이언트로 나가는 모든 요청에 자동으로**
적용된다. 실무에서는 토큰을 하드코딩하지 않고, 요청 스레드의 `SecurityContext`나
`RequestContextHolder`에서 현재 사용자의 토큰을 꺼내 헤더에 실어 보내는 식으로
구현한다 — 이렇게 하면 개별 API 메서드는 인증에 대해 전혀 몰라도 되고, 헤더를
빠뜨리는 실수 자체가 구조적으로 불가능해진다.

## 예외 처리: 상태 코드별로 다른 예외를 던지게 만든다

기본 상태로는 외부 API가 4xx/5xx를 반환하면 Feign이 일반적인 예외를 던진다.
문제는 이것만으로는 "왜 실패했는지"를 호출부에서 구분할 수 없다는 것 —
`ErrorDecoder`로 상태 코드를 우리 도메인의 예외로 변환해야 한다.

```java
@Component
public class FeignErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        return new RuntimeException("외부 API 호출 실패: " + response.status());
    }
}
```

이 예시는 상태 코드를 그대로 메시지에 박아 넣는 최소 구현이다. 실무에서는 여기서
한 단계 더 나아가 상태 코드별로 다른 예외 타입을 던지게 만든다 — 예를 들어 404는
`ExternalResourceNotFoundException`, 401/403은 `ExternalAuthException`, 5xx는
`ExternalServerException`처럼 구분해두면, 호출하는 서비스 코드에서 "이 실패가
재시도할 가치가 있는지(5xx는 대체로 재시도 가능, 4xx는 대체로 불가능)"를 예외
타입만으로 판단할 수 있다. `ErrorDecoder`를 구현하지 않고 넘어가면, 이런 판단을
호출부마다 `response.status()`를 직접 파싱해서 반복하게 된다.

## Timeout은 반드시 명시적으로

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 3000
        readTimeout: 5000
```

타임아웃을 설정하지 않으면 기본값에 의존하게 되는데, 외부 API가 응답을 지연시키는
상황에서 이 기본값이 얼마나 관대한지에 따라 우리 서비스의 스레드가 그만큼 오래
묶여 있게 된다. 외부 시스템 장애가 우리 서비스의 스레드 풀 고갈로 전이되는 대표적인
경로가 이 타임아웃 미설정이다.

## 설계 팁: Feign은 "외부 시스템 경계"에만 쓴다

```
user-api-client
payment-api-client
order-api-client
```

API 도메인별로 Feign 인터페이스를 분리하는 이유는, 특정 외부 시스템의 스펙 변경이
다른 외부 연동에 영향을 주지 않게 하기 위해서다. 그리고 원칙 하나: **Feign은 외부
API/타 시스템 호출에만 쓰고, 내부 서비스 간 호출에는 쓰지 않는다.** 내부 MSA
서비스 간 호출은 서비스 디스커버리(Eureka)와 로드밸런서가 개입하는 별도 경로로
가져가는 게 맞다 — Feign을 내부 호출에도 무분별하게 쓰면, "이게 외부 연동인지
내부 연동인지"를 코드만 보고 구분하기 어려워진다.

## Feign vs HttpInterface, 다시 정리

RestTemplate과 비교하면 Feign이 가독성·테스트 용이성·유지보수 모든 면에서 낫다는
건 자명하다. 진짜 선택은 Feign과 HttpInterface 사이에서 갈린다 — 이미 Spring
Cloud 스택(서비스 디스커버리, 서킷 브레이커)을 쓰는 MSA 환경이라면 그 생태계와
자연스럽게 통합되는 Feign이 낫고, 특정 외부 SaaS 하나만 호출하는 단순한 경우라면
별도 의존성이 필요 없는 HttpInterface가 더 가볍다.

## 한 줄 정리

Feign Client는 "외부 API를 인터페이스로 다루게 해주는 HTTP 추상화 도구"다 — 다만
그 가치는 인터페이스 선언 자체보다, `RequestInterceptor`로 인증을 구조적으로
빠뜨릴 수 없게 만들고 `ErrorDecoder`로 실패 원인을 도메인 예외로 번역하는 데서
나온다.
