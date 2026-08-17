---
title: "\"이거 당연히 성공하는 거 아니야?\" — 단위 테스트가 실제로 증명하는 것"
date: 2025-06-14
categories: [테스트]
tags: [testing, tdd, spring-boot]
---

단위 테스트를 처음 짜본 사람이 자주 하는 질문이 있다. "Mock으로 리턴값을 다 정해놓고
테스트하면, 그거 당연히 성공하는 거 아닌가요?" 틀린 말은 아니다 — 실제로 그 테스트는
"당연히" 통과하도록 짜여 있다. 그런데 그 "당연함"을 확인하는 것 자체가 단위 테스트의
진짜 목적이라는 걸 이해하면 관점이 달라진다.

## 단위 테스트를 리허설로 생각하면

단위 테스트를 "주인공 배우의 독백 연기 리허설"에 비유해볼 수 있다.

- **테스트 대상(Unit)**: 오늘 리허설할 주인공 배우 (예: `AuthService`)
- **테스트의 목표**: 주인공이 대사·동선·감정 표현(비즈니스 로직)을 각본대로 정확히
  수행하는지 확인
- **의존성(Dependencies)**: 주인공과 상호작용하는 조연 배우들 (`UserRepository`,
  `JwtTokenProvider` 등)

리허설의 목적은 주인공의 연기에만 집중하는 것이다. 조연이 즉흥적으로 다른 대사를
치면 주인공의 연기를 제대로 평가할 수 없다. 그래서 조연 대신 **우리가 시키는 대로만
정해진 대사를 읊는 대역(Mock)**을 세운다. Mockito 같은 모킹 라이브러리를 쓰는 이유가
바로 이것이다 — 의존성의 변동성을 제거하고, 오직 테스트 대상의 로직만 관찰 가능한
상태로 만드는 것.

```java
@ExtendWith(MockitoExtension.class)
public class AuthServiceTest {

    @InjectMocks
    private AuthService authService;

    @Mock private UserRepository userRepository;
    @Mock private JwtTokenProvider jwtTokenProvider;
    @Mock private OAuthProviderFactory providerFactory;
    @Mock private OAuthProvider mockOAuthProvider;

    @Test
    @DisplayName("신규 유저가 크루장으로 가입하는 시나리오")
    void login_new_user_as_leader() {
        // given — 대역 배우들에게 대본을 나눠준다
        when(providerFactory.getProvider(SocialType.KAKAO)).thenReturn(mockOAuthProvider);
        when(mockOAuthProvider.getAccessToken(any())).thenReturn("test-access-token");
        when(userRepository.findByEmail("newuser@example.com")).thenReturn(Optional.empty());
        when(userRepository.save(any(User.class))).thenAnswer(i -> i.getArgument(0));

        // when — 실제 연기 시작
        String resultToken = authService.login(requestDto);

        // then
        assertThat(resultToken).isEqualTo("fake-jwt-token");
    }
}
```

`when(대역.메서드).thenReturn(값)`은 "이 상황이면 이렇게 답해라"는 대본이다. 대본대로
답하는 대역들 사이에서 주인공이 시나리오대로 흘러가는 건 당연하다 — 그런데 그 "당연함"이
왜 가치가 있을까?

## "당연함"이 증명하는 세 가지

1. **로직 흐름에 대한 보증.** 테스트가 통과했다는 건 "AuthService가 예상한 순서대로
   `findByEmail`을 호출하고, 그 결과에 따라 `save`를 호출한 뒤, 마지막으로
   `generateToken`을 호출하는구나"라는 내부 로직의 흐름이 실제로 그렇게 짜여 있음을
   증명한다. `Mockito.verify()`로 "정말 `save`가 호출됐는가"를 직접 검증하면 이
   보증이 더 명확해진다.
2. **미래를 위한 안전망.** 6개월 뒤 이 코드를 리팩터링한다고 하자. 수정 후 이 테스트가
   여전히 통과한다면, 기존의 핵심 로직 흐름을 깨지 않았다는 자신감을 곧바로 얻는다.
3. **살아있는 명세서.** 이 테스트 코드 자체가 `AuthService.login()`이 어떤 조건에서
   어떤 의존성을 호출하고 어떤 결과를 내야 하는지 보여주는 가장 정확한 문서가 된다 —
   주석이나 위키 문서와 달리, 코드가 바뀌면 테스트도 실패하므로 실제 동작과
   어긋날 수가 없다.

## 이 방식의 한계도 있다

Mock으로 통제된 환경은 강력하지만 공짜가 아니다. Mock의 동작이 실제 `UserRepository`
구현과 어긋나면(예: 실제로는 `findByEmail`이 대소문자를 구분하지 않는데 Mock 설정은
구분한다고 가정했다면), 단위 테스트는 초록불인데 실제 운영에서는 다르게 동작하는
상황이 생긴다. 즉 단위 테스트는 **"내가 짠 로직이 내가 의도한 대로 각 의존성을
호출하는가"**를 보증할 뿐, **"그 의존성들이 실제로 그렇게 동작하는가"**는 보증하지
않는다. 후자를 확인하려면 실제 DB/Bean을 쓰는 통합 테스트가 별도로 필요하다 — 단위
테스트가 통합 테스트를 대체할 수 없는 이유가 여기 있다.

## 정리

단위 테스트는 성공/실패를 가리는 도구가 아니라, 통제된 환경 속에서 테스트 대상이
의도한 설계대로 정확히 동작하는지를 검증하고 그 설계를 문서화하는 장치다. "당연히
성공하는 테스트"가 무의미해 보인다면, 그 당연함이 실제로는 로직 흐름에 대한 보증이라는
사실을 놓친 것이다. 다만 이 보증은 Mock으로 대체된 의존성의 실제 동작까지는 책임지지
않는다는 것도 함께 기억해야 한다.
