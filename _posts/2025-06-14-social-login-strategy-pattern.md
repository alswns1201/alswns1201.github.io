---
title: "전략 패턴으로 카카오·네이버·구글 소셜 로그인 유연하게 구현하기"
date: 2025-06-14
categories: [개발일지, SPRINGBOOT]
tags: [spring-boot, architecture, spring-security]
---

소셜 로그인을 카카오 하나만 구현한다면 `KakaoLoginService` 하나로 충분하다. 문제는
여기에 네이버, 구글이 추가되는 순간이다 — 비슷한 듯 다른 로직이 반복되고,
`AuthService`나 컨트롤러의 분기문(if-else, switch)이 플랫폼이 늘어날 때마다
길어진다. 이건 OCP(개방-폐쇄 원칙) 위반이고, 유지보수를 서서히 지옥으로 만든다.

## 설계의 축: 변하는 것과 변하지 않는 것을 분리한다

- **변하는 것**: 각 소셜 플랫폼과의 통신 방식, 응답 데이터 형식
- **변하지 않는 것**: 우리 서비스의 회원가입/로그인 처리 로직, JWT 발급 로직

이 "변하는 것"을 전략(Strategy)으로 캡슐화하는 게 핵심이다.

## 1) 공통 계약: OAuthProvider 인터페이스

```java
public interface OAuthProvider {
    SocialType getProviderType();
    String getAccessToken(String authorizationCode);
    OAuthUserProfile getUserProfile(String accessToken);
}
```

각 플랫폼의 제각각인 유저 정보 응답은 `OAuthUserProfile`이라는 표준화된 DTO로
변환해서 반환한다. 이 표준화가 없으면 이후 로직(회원가입, JWT 발급)이 플랫폼별로
다시 갈라지게 된다 — 인터페이스를 나누는 것 못지않게, **인터페이스가 반환하는
데이터 형태를 통일하는 것**이 이 설계의 진짜 핵심이다.

## 2) 플랫폼별 구현체

```java
@Component
public class KakaoOAuthProvider implements OAuthProvider {
    @Override
    public SocialType getProviderType() { return SocialType.KAKAO; }
    // getAccessToken, getUserProfile 구현
}
```

`NaverOAuthProvider`, `GoogleOAuthProvider`도 동일한 구조로 구현한다.

## 3) 전략을 찾아주는 팩토리

```java
@Component
public class OAuthProviderFactory {

    private final Map<SocialType, OAuthProvider> providers;

    public OAuthProviderFactory(List<OAuthProvider> providerList) {
        this.providers = providerList.stream().collect(
            Collectors.toUnmodifiableMap(OAuthProvider::getProviderType, Function.identity())
        );
    }

    public OAuthProvider getProvider(SocialType socialType) {
        return providers.get(socialType);
    }
}
```

**여기서 왜 `List<OAuthProvider>`로 주입받는지가 이 설계의 진짜 트릭이다.** 스프링은
같은 타입의 빈이 여러 개 있으면 단일 주입(`@Autowired OAuthProvider provider`)
시점에 "어떤 빈을 줄지 모호하다"며 에러를 던진다(`NoUniqueBeanDefinitionException`).
그런데 `List<OAuthProvider>`로 받으면 **해당 인터페이스의 모든 구현체를 하나도
빠짐없이 리스트로 모아서 주입**해준다 — 모호성 문제 자체가 성립하지 않는 주입
방식이다. 생성자에서 이 리스트를 `SocialType`을 키로 하는 Map으로 변환해두면,
새 플랫폼이 추가돼도 팩토리 코드는 단 한 줄도 안 바뀐다.

## 4) 지휘자: AuthService

```java
@Service
@RequiredArgsConstructor
public class AuthService {

    private final OAuthProviderFactory providerFactory;

    @Transactional
    public String login(OAuthLoginRequestDto req) {
        OAuthProvider provider = providerFactory.getProvider(req.getSocialType());
        String accessToken = provider.getAccessToken(req.getAuthorizationCode());
        OAuthUserProfile userProfile = provider.getUserProfile(accessToken);

        User user = registerOrLoginUser(userProfile, req);
        return createJwtToken(user);
    }
}
```

`if (provider == "kakao")` 같은 분기문이 사라진다. 애플 로그인을 추가해야 한다면
`AppleOAuthProvider` 구현체 하나만 만들고 `@Component`를 붙이면 끝난다 — 기존
코드는 한 줄도 안 바뀐다.

## 원문에 없던 것: 이 설계가 안 잡아주는 보안 문제

전략 패턴이 잡아주는 건 **코드 구조**다. 소셜 로그인 자체의 보안 문제는 이 설계와
별개로 여전히 신경 써야 한다.

- **state 파라미터(CSRF 방지)**: 프론트엔드가 사용자를 소셜 플랫폼 인증 페이지로
  보낼 때, 예측 불가능한 랜덤 `state` 값을 함께 보내고 콜백에서 이 값이 일치하는지
  검증해야 한다. 이게 없으면 공격자가 자신의 인가 코드를 피해자 세션에 흘려넣는
  방식의 CSRF 공격이 가능해진다. `OAuthProvider` 인터페이스 어디에도 이 검증
  책임이 명시돼 있지 않다는 점이 이 설계의 사각지대다 — 별도로 필터나
  `AuthService` 진입 시점에 넣어줘야 한다.
- **redirect_uri 검증**: 소셜 플랫폼 콘솔에 등록된 redirect_uri와 실제 요청의
  redirect_uri가 정확히 일치하는지는 플랫폼 쪽에서 검증하지만, 여러 환경(로컬,
  스테이징, 프로덕션)의 URI를 전부 등록해두는 실수를 하면 검증이 느슨해질 수
  있다.
- **발급된 JWT를 어디에 저장할지**: 이 설계는 JWT를 "발급"하는 데까지만 책임진다.
  발급된 토큰을 프론트엔드가 어디에 저장하느냐(localStorage vs httpOnly 쿠키)는
  XSS 취약점 노출 여부를 가르는 완전히 별개의 결정이다.

## 정리

전략 패턴이 여기서 하는 일은 "플랫폼이 늘어나도 기존 코드를 안 건드리게" 만드는
것이지, 인증 자체를 안전하게 만들어주는 게 아니다. 구조적 확장성과 보안은 서로
다른 축의 문제이고, 이 설계를 도입했다고 후자가 자동으로 해결됐다고 착각하면 안
된다.
