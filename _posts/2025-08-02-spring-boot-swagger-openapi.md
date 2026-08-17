---
title: "Spring Boot에서 Swagger(springdoc-openapi)로 API 문서 자동화하기"
date: 2025-08-02
categories: [Spring Boot]
tags: [spring-boot, api-design, documentation]
---

API 문서를 손으로 쓰고 유지하는 건 코드가 바뀔 때마다 같이 안 바뀌기 마련이라
금방 신뢰를 잃는다. Swagger(정확히는 springdoc-openapi)를 붙이면 코드에서 문서를
뽑아내므로, 적어도 "코드와 문서가 따로 논다"는 문제는 구조적으로 줄어든다.

## 의존성 하나로 시작되는 이유

```gradle
dependencies { implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.1.0' }
```

이 의존성만 추가하면 별도 설정 없이 `/swagger-ui/index.html`에서 API 문서가
보인다. 이게 가능한 이유는 springdoc이 런타임에 Spring MVC의 `RequestMappingHandlerMapping`을
스캔해서 컨트롤러의 매핑 정보(경로, 메서드, 파라미터, 응답 타입)를 자동으로
OpenAPI 스펙으로 변환해주기 때문이다 — 즉 문서가 "코드에서 자동으로 파생된
결과물"이지, 별도로 관리하는 산출물이 아니다.

## 어노테이션으로 세부 정보 채우기

```java
@RestController
@RequestMapping("/api/users")
@Tag(name = "User API", description = "사용자 관련 API")
public class UserController {
    @GetMapping("/{id}")
    @Operation(summary = "사용자 조회", description = "ID로 사용자 정보를 조회한다.")
    public User getUser(@PathVariable Long id) {
        return userService.getUserById(id);
    }
}
```

리플렉션만으로는 "이 API가 뭘 하는지"라는 의도는 못 뽑아낸다. `@Operation`,
`@Tag` 같은 어노테이션이 그 빈틈을 채운다. 여기서 실무 판단 포인트가 하나 있다 —
어노테이션을 얼마나 자세히 채울지는 **이 문서를 누가 보는가**에 따라 달라진다.
내부 개발자용이면 최소한(요약 한 줄)으로 충분하고, 외부 파트너사에 API를 제공하는
계약 문서로 쓴다면 요청/응답 예시, 에러 코드까지 채워야 문서로서 값어치가 있다.

## 놓치기 쉬운 지점: Response DTO를 그대로 노출하는 문제

springdoc은 컨트롤러 메서드의 반환 타입을 그대로 스캔해서 응답 스키마를 만든다.
문제는 반환 타입으로 **엔티티를 그대로** 쓰면, 엔티티에 있는 내부 필드(예: 비밀번호
해시, 감사 로그용 컬럼)까지 문서에 노출된다는 것이다. 이건 단순히 "문서가 지저분하다"
수준을 넘어서, 실제 API 응답도 엔티티를 그대로 직렬화하고 있을 가능성이 높다는
신호이기도 하다 — 응답 전용 DTO를 분리하지 않았다는 뜻이기 때문이다. Swagger 문서를
검토하는 과정에서 이런 설계 누락을 발견하는 경우가 실제로 많다.

## 인증이 걸린 API 문서화

기본 설정만으로는 Swagger UI에 인증 헤더를 넣을 방법이 없다. `SecurityScheme`을
별도로 등록해야 UI에서 "Authorize" 버튼으로 토큰을 넣고 테스트할 수 있다.

```java
@Configuration
public class SwaggerConfig {
    @Bean
    public OpenAPI openAPI() {
        String schemeName = "bearerAuth";
        return new OpenAPI()
                .addSecurityItem(new SecurityRequirement().addList(schemeName))
                .components(new Components().addSecuritySchemes(schemeName,
                        new SecurityScheme()
                                .name(schemeName)
                                .type(SecurityScheme.Type.HTTP)
                                .scheme("bearer")
                                .bearerFormat("JWT")));
    }
}
```

이걸 빠뜨리면, JWT 인증이 걸린 엔드포인트를 Swagger UI에서 테스트할 때마다
매번 별도 도구(Postman 등)로 토큰을 받아와야 해서 문서의 실용성이 크게 떨어진다.

## Swagger UI를 운영 환경에 그대로 노출하면 안 되는 이유

`/swagger-ui/index.html`이 인증 없이 열려 있으면, 이 서비스의 전체 API 목록과
파라미터 구조가 외부에 그대로 드러난다 — 공격자 입장에서는 정찰(recon) 단계를
생략시켜주는 셈이다. 운영 환경에서는 다음 중 하나를 반드시 적용해야 한다.

- `springdoc.swagger-ui.enabled=false`로 운영 프로필에서 아예 끄기
- 또는 Swagger 관련 경로를 Security 설정에서 별도 인증/IP 제한으로 보호하기

개발 편의를 위해 열어둔 문서 UI가 그대로 운영까지 따라가는 실수는 생각보다 흔하다.

## 정리

- springdoc-openapi는 "코드와 문서가 따로 관리되는 문제"를 코드에서 문서를
  파생시키는 방식으로 해결한다.
- 어노테이션 상세도는 문서의 독자(내부 개발자 vs 외부 파트너)에 맞춰 조절한다.
- 엔티티를 응답 타입으로 그대로 쓰면 Swagger 문서에 내부 필드가 새어나간다 —
  이건 DTO 분리 누락의 신호로 읽을 수 있다.
- 인증이 걸린 API는 `SecurityScheme`을 등록해야 문서에서 바로 테스트할 수 있다.
- 운영 환경에서 Swagger UI를 그대로 열어두는 건 API 표면을 공개 정찰 대상으로
  만드는 것과 같다 — 반드시 끄거나 보호할 것.
