---
title: "단위 테스트 vs 통합 테스트, 언제 무엇을 써야 할까? (feat. AssertJ)"
date: 2025-06-15
categories: [테스트]
---

테스트를 "많이 짜면 좋다"는 건 다들 안다. 문제는 어떤 테스트를 얼마나 짜야 하냐는
것이다. 모든 걸 통합 테스트로 짜면 느려서 CI가 죽고, 모든 걸 단위 테스트로만 짜면
"각 부품은 잘 만들었는데 조립하니 안 돌아가는" 사고를 놓친다. 이 둘의 역할을
정확히 나누는 게 핵심이다.

## 테스트 피라미드에서 각 층이 잡는 버그가 다르다

- **단위 테스트**: 가장 넓은 기반. 클래스/메서드 하나가 설계대로 동작하는지, 즉
  "이 부품 자체가 맞게 만들어졌는가"를 검증한다.
- **통합 테스트**: 여러 부품을 실제로 조립했을 때 상호작용이 맞는지 — DB 연동,
  트랜잭션, 컴포넌트 간 협력을 검증한다.
- **E2E 테스트**: 사용자 관점에서 전체 흐름을 검증한다. 가장 느리고 비싸다.

| 구분 | 단위 테스트 | 통합 테스트 |
|---|---|---|
| 목표 | 부품이 설계대로 만들어졌나 | 부품을 조립하니 제대로 작동하나 |
| 환경 | Mock으로 주변 통제 | 실제 Spring 컨테이너 + 실제 DB(인메모리) |
| 검증 대상 | 클래스 내부 로직, 분기 | 컴포넌트 간 상호작용, DB 연동, 트랜잭션 |
| 속도 | 매우 빠름 | 느림 |
| 안정성 | 외부 요인 없음 | DB/설정 상태에 영향받을 수 있음 |

이 표에서 중요한 건 속도/안정성 트레이드오프다. 단위 테스트가 빠르고 안정적인
이유는 정확히 "무엇을 검증하지 않는가" 덕분이다 — 실제 DB, 실제 Bean 조립을
모두 배제하고 순수 로직만 본다. 그래서 단위 테스트를 아무리 많이 짜도
"Repository 쿼리 메서드 이름을 잘못 지었다"거나 "@Transactional이 실제로
롤백되는가" 같은 문제는 절대 못 잡는다. 그건 통합 테스트의 영역이다.

## AssertJ에서 `import static`을 쓰는 이유

```java
import static org.assertj.core.api.Assertions.assertThat;
```

`assertThat`은 `Assertions` 클래스의 static 메서드다. static import를 하면
`Assertions.assertThat()` 대신 `assertThat()`으로 바로 쓸 수 있다. 단순히 타이핑을
줄이는 게 아니라, 테스트 코드가 "나는 ~라고 단언한다(assert that)"처럼 자연어에
가깝게 읽히게 만드는 게 진짜 이유다. 테스트 코드는 실행되는 코드이기 이전에
**읽히는 문서**다 — 실패했을 때 무엇을 기대했는지가 코드에서 바로 읽혀야 디버깅이
빠르다.

## 통합 테스트: `@SpringBootTest` + `@Transactional`

```java
@SpringBootTest
@Transactional // 각 테스트 후 DB 롤백을 위해 필수!
class AuthServiceIntegrationTest {

    @Autowired private AuthService authService;
    @Autowired private UserRepository userRepository;
    @Autowired private CrewRepository crewRepository;
    @Autowired private CrewMemberRepository crewMemberRepository;

    @Test
    @DisplayName("신규 유저가 크루장으로 가입 시, 모든 데이터가 DB에 올바르게 저장된다.")
    void registerOrLoginUser_integration_test() {
        OAuthUserProfile fakeProfile = new OAuthUserProfile();
        fakeProfile.setEmail("integration-test@example.com");
        fakeProfile.setProviderId("kakao-12345");

        OAuthLoginRequestDto requestDto = new OAuthLoginRequestDto();
        requestDto.setName("통합테스터");
        requestDto.setCrewName("통합테스트크루");
        requestDto.setCrewMemberRole(CrewMemberRole.CREW_LEADER);

        User resultUser = authService.registerOrLoginUser(fakeProfile, requestDto);

        User foundUser = userRepository.findById(resultUser.getId()).orElseThrow();
        assertThat(foundUser.getEmail()).isEqualTo("integration-test@example.com");

        Crew foundCrew = crewRepository.findByName("통합테스트크루").orElseThrow();
        assertThat(foundCrew).isNotNull();

        CrewMember foundMember = crewMemberRepository.findByUserAndCrew(foundUser, foundCrew).orElseThrow();
        assertThat(foundMember.getRole()).isEqualTo(CrewMemberRole.CREW_LEADER);

        List<CrewMember> members = crewMemberRepository.findAllByCrew(foundCrew);
        assertThat(members).hasSize(1);
    }
}
```

`@Transactional`을 테스트 클래스에 붙이면 각 테스트가 끝날 때 변경 사항이 자동
롤백된다. 이게 왜 필수인가 — 롤백이 없으면 테스트가 실행될 때마다 DB에 데이터가
누적되고, 이전 테스트의 잔여 데이터가 다음 테스트의 결과에 영향을 줘서 테스트
간 격리가 깨진다. "테스트는 항상 같은 순서로, 같은 결과로 통과해야 한다"는
전제를 지키는 장치가 바로 이 롤백이다.

여기서 외부 API 호출이 필요한 `OAuthProvider` 같은 컴포넌트는 여전히 Mocking하는
게 맞다 — 통합 테스트라고 모든 걸 진짜로 붙이는 게 아니라, **우리가 검증하고 싶은
경계(여기서는 DB까지의 흐름)만 진짜로 붙이고 나머지는 격리**하는 것이 핵심이다.
이 경계를 잘못 잡으면(예: 외부 API까지 진짜로 호출) 테스트가 네트워크 상태에
좌우되는 flaky 테스트가 된다.

## 언제 무엇을 쓸까

- **단위 테스트를 기본으로.** 새 클래스/메서드를 만들면 성공/실패/예외 케이스를
  커버하는 단위 테스트부터 작성한다. 빠르고 안정적이며, 무엇보다 "테스트하기
  쉬운 설계"를 강제하는 효과가 있다 — Mock으로 고립시키기 어려운 클래스는 대개
  책임이 너무 많다는 신호다.
- **통합 테스트는 '연결점'을 검증할 때.** Controller→Service→Repository→DB
  전체 흐름이 중요한 핵심 기능(회원가입, 주문), `@Transactional`의 실제 동작,
  외부 설정/메시지 큐/외부 API 연동.

두 테스트는 대체 관계가 아니라 보완 관계다. 단위 테스트로 부품의 품질을,
통합 테스트로 조립 후의 안정성을 각각 다른 층에서 보장한다.
