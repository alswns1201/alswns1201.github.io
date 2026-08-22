---
title: "JPA 도메인 설계, 요구사항부터 테스트까지의 사고 흐름"
date: 2025-07-01
categories: [Java/Spring]
---

JPA 문법(`@ManyToOne`, `@Embeddable` 같은 어노테이션)을 아는 것과, 요구사항을 받았을 때
"어떤 테이블이 필요하고 어떻게 관계를 맺을지"를 스스로 판단하는 것은 다른 능력이다.
여기서는 "회원이 크루 일정을 등록한다"는 요구사항 하나를 붙잡고, 요구사항 분석부터
테스트 코드까지 실제로 어떤 순서로 사고가 흘러가는지 정리한다.

## 1) 요구사항을 먼저 문장으로 쪼갠다

코드를 짜기 전에 요구사항을 명확한 문장 단위로 분해한다.

- 사용자가 일정을 추가한다
- 일정의 유형은 개인/정기로 나뉜다
- 입력값은 제목, 날짜(시간), 장소, 설명
- 등록자는 자동으로 일정의 책임자가 된다
- 등록과 동시에 등록자는 참석자가 된다

이 단계를 건너뛰고 바로 엔티티부터 설계하면, "책임자"와 "참석자"가 같은 사람인데도
서로 다른 두 개념이라는 걸 놓치기 쉽다 — 실제로 아래 도메인 모델에서 이 둘은
`Event.owner`와 별도의 `EventAttendee` 레코드로, 개념적으로 명확히 분리된다.

## 2) 요구사항에서 도메인(테이블)을 뽑아낸다

문장에 나온 명사와 동사를 보면서 필요한 테이블을 판단한다: `User`(사용자),
`Event`(일정), `EventAttendee`(일정 참석자). 여기서 판단 기준은 "이 개념이 독립적인
생명주기를 가지는가"다 — 참석자 기록은 일정과 사용자 둘 다에 의존하는 별도 개체이므로
값 객체가 아니라 엔티티로 뽑는다(누가 언제 참석 확정했는지 등 추가 속성이 붙을 수
있기 때문).

```java
@Entity
public class Event {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String title;

    @Embedded
    private Location location;

    private String description;
    private LocalDateTime eventDateTime;

    @Enumerated(EnumType.STRING)
    private EventType eventType;

    @ManyToOne(fetch = FetchType.LAZY) // 일정은 하나의 책임자(User)에 속한다
    private User owner;
}
```

`@ManyToOne`을 붙일 때 항상 "이 엔티티는 저 엔티티에 속해있다"는 문장으로 읽는 습관이
방향을 헷갈리지 않게 해준다 — `Event`가 `owner`를 `@ManyToOne`으로 가진다는 건
"여러 일정이 한 명의 책임자에 속할 수 있다"는 뜻이다.

```java
@Entity
public class EventAttendee {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @ManyToOne(fetch = FetchType.LAZY) // 참석자는 하나의 이벤트에 속한다
    private Event event;

    @ManyToOne(fetch = FetchType.LAZY) // 참석자는 하나의 사용자에 속한다
    private User user;
}
```

`EventAttendee`가 `Event`와 `User` 양쪽을 각각 `@ManyToOne`으로 들고 있는 구조는
전형적인 N:M 관계의 중간 테이블 패턴이다 — 한 이벤트에 여러 참석자가 있을 수 있고,
한 사용자가 여러 이벤트에 참석자로 등록될 수 있다.

## 3) Repository는 필요한 조회부터 최소한으로

```java
public interface EventAttendeeRepository extends JpaRepository<EventAttendee, String> {
    boolean existByEventAndUser(Event event, User user);
}

public interface EventRepository extends JpaRepository<Event, String> {
}
```

`existByEventAndUser`처럼 단순 존재 여부를 확인하는 쿼리는 메소드 쿼리로 충분하다.
Repository를 처음부터 다 채우지 않고, 서비스 로직을 짜다가 "이 조회가 필요하다"고
판단될 때 하나씩 추가하는 게 실제 개발 순서에 가깝다.

## 4) 서비스 로직: 검증 → 저장 → 연관 저장 순서

```java
@Transactional
public String createEvent(CreateEventRequest createEventRequest) {
    User user = userService.findUserByEmail(createEventRequest.getUserEmail());

    if (createEventRequest.getEventDateTime() == null
            || createEventRequest.getEventDateTime().isBefore(LocalDateTime.now())) {
        throw new BusinessException(ErrorCode.INVALID_EVENT_DATE);
    }
    if (createEventRequest.getTitle() == null || createEventRequest.getTitle().isBlank()
            || createEventRequest.getLocation() == null) {
        throw new BusinessException(ErrorCode.INVALID_INPUT_VALUE);
    }

    Event event = Event.builder()
            .title(createEventRequest.getTitle())
            .description(createEventRequest.getDescription())
            .owner(user)
            .eventDateTime(createEventRequest.getEventDateTime())
            .location(createEventRequest.getLocation())
            .eventType(createEventRequest.getEventType())
            .build();
    Event saved = eventRepository.save(event);

    boolean alreadyExists = eventAttendeeRepository.existByEventAndUser(saved, user);
    if (alreadyExists) {
        throw new BusinessException(ErrorCode.CREW_USER_DUPLICATED);
    }

    EventAttendee eventAttendee = EventAttendee.builder()
            .event(event)
            .user(user)
            .build();
    eventAttendeeRepository.save(eventAttendee);

    return saved.getOwner().getName();
}
```

검증을 저장보다 먼저 하는 이유는 단순하다 — 잘못된 입력값으로 DB에 쓰기 작업을
시도하고 나서 실패하는 것보다, 쓰기 전에 걸러내는 게 불필요한 쿼리를 줄인다.
`@Transactional`이 메서드 전체를 감싸고 있으므로, `Event` 저장 후
`EventAttendee` 저장 사이에 예외가 나도(예: 중복 참석자 체크 실패) 앞서 저장한
`Event`까지 롤백된다 — 이게 두 테이블에 걸친 저장을 하나의 메서드 안에 묶어두는
이유다.

`Location`은 `@Embeddable`로 뽑았다 — 위도/경도/주소 같은 필드들이 항상
`Event`에 종속되어 존재하고, 독립적으로 조회되거나 삭제될 일이 없기 때문이다.

## 5) 예외는 "화면에서 막을 수 없는 것"부터 생각한다

프론트에서도 필수값 검증을 하겠지만, 서버 쪽 예외 처리는 별개로 필요하다 — 클라이언트
검증은 우회 가능하기 때문이다. 이 요구사항에서 실제로 나올 수 있는 예외 세 가지:
날짜 누락/과거 날짜, 제목·위치 누락, 이미 참석 중인 사용자의 중복 등록. 기존에
정의해둔 `ErrorCode` enum에 상수를 추가하는 방식으로 확장했다 — 새 예외 상황이
생길 때마다 enum에 항목을 더하는 패턴은 [별도 글](/posts/controlleradvice-exception-handling/)에서
다룬 예외 처리 구조와 동일하다.

## 6) 테스트: "이 서비스 메서드 하나"만 검증한다

```java
@ExtendWith(MockitoExtension.class)
public class EventServiceTest {

    @InjectMocks
    private EventService eventService;
    @Mock private UserService userService;
    @Mock private EventRepository eventRepository;
    @Mock private EventAttendeeRepository eventAttendeeRepository;

    @Test
    void 이벤트_등록() {
        CreateEventRequest createEventRequest = new CreateEventRequest();
        createEventRequest.setEventType(EventType.PERSONAL);
        createEventRequest.setEventDateTime(LocalDateTime.now().plusDays(1));
        createEventRequest.setTitle("내일 일정 테스트");
        createEventRequest.setUserEmail("test@example.com");
        createEventRequest.setLocation(new Location("스타벅스 강남점", "서울특별시 강남구", 37.49, 127.02));

        User user = User.builder().id("u1").email("test@example.com").name("김러너").build();
        Event event = Event.builder().id("e1").title(createEventRequest.getTitle())
                .owner(user).build();

        Mockito.when(userService.findUserByEmail(createEventRequest.getUserEmail())).thenReturn(user);
        Mockito.when(eventRepository.save(Mockito.any(Event.class))).thenReturn(event);
        Mockito.when(eventAttendeeRepository.existByEventAndUser(Mockito.any(), Mockito.any())).thenReturn(false);

        String result = eventService.createEvent(createEventRequest);

        assertThat(result).isEqualTo("김러너");
    }
}
```

`@Mock`으로 감싼 대상은 `UserService`, `EventRepository`, `EventAttendeeRepository`
셋 다다 — 검증 대상이 `eventService.createEvent()`라는 로직 하나이지, 그 로직이
의존하는 DB나 다른 서비스의 실제 동작이 아니기 때문이다. 이렇게 의존성을 전부
Mock으로 격리하면 "이 메서드가 주어진 입력에 대해 올바른 순서로 호출하고 올바른
값을 반환하는가"만 순수하게 검증할 수 있다.

## 정리: 이 사고 흐름이 왜 이 순서인가

요구사항 → 도메인 → Repository → Service → Controller → 예외 → 테스트 순서로
진행한 이유는, 각 단계가 앞 단계의 결과에 의존하기 때문이다. 도메인 모델이 정해지지
않으면 Repository의 조회 메서드를 뭘로 짤지 알 수 없고, Service 로직이 없으면
어떤 예외가 실제로 발생할 수 있는지 알 수 없다. 순서를 거꾸로 가면(예: 테스트부터
작성) TDD 관점에서는 유효하지만, 도메인 자체가 불확실한 초기 설계 단계에서는
요구사항을 먼저 도메인으로 변환하는 이 흐름이 더 안정적이다.
