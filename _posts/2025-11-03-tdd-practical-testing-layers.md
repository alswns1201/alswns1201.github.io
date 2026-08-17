---
title: "TDD, Service만이 아니라 Controller와 외부 API까지 테스트로 감싸는 법"
date: 2025-11-03
categories: [테스트]
tags: [testing, tdd, spring-boot]
---

TDD를 "테스트를 먼저 짜고 그걸 만족시키는 코드를 쓴다"는 원론으로만 알고 있으면
실제 적용에서 막힌다. 현실적인 순서는 조금 다르다 — 기본 골격(Service, Repository
등)을 먼저 만든 뒤, **필요한 부분을 테스트로 채워나가는 것**에 가깝다. 여기서
중요한 건 "테스트를 어느 계층에, 무엇을 가짜로 만들어서 짜야 하는가"다. Service
단위 테스트는 이미 익숙하다는 전제로, 그다음 두 계층 — Controller와 외부 API
연동 — 을 어떻게 테스트로 감싸는지 정리한다.

## Service 계층: Mockito로 의존성을 통제한다

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderService orderService;

    @Test
    void 주문_생성_테스트() {
        // given
        Order order = new Order("A001", 1000);
        when(orderRepository.save(any(Order.class))).thenReturn(order);

        // when
        Order result = orderService.createOrder("A001", 1000);

        // then
        assertEquals("A001", result.getOrderId());
    }
}
```

`when(...).thenReturn(...)`은 "이 의존성이 이렇게 동작한다고 가정하고, 내 로직만
검증하겠다"는 선언이다. 여기서 검증 대상은 `OrderRepository`가 실제로 잘 동작하는지가
아니라 `OrderService.createOrder`의 로직 자체다.

## Controller 계층: `@WebMvcTest`로 웹 계층만 떼어낸다

Service 테스트를 통과했다고 API가 제대로 동작한다는 보장은 없다 — 요청 파라미터
바인딩, 응답 JSON 구조, HTTP 상태 코드는 Controller 계층의 책임이고, Service
단위 테스트는 이 계층을 전혀 검증하지 않는다.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Test
    void 주문_조회_API_테스트() throws Exception {
        OrderResponse response = new OrderResponse("A001", 1000);
        when(orderService.getOrder("A001")).thenReturn(response);

        mockMvc.perform(get("/orders/A001"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.orderId").value("A001"))
                .andExpect(jsonPath("$.amount").value(1000));
    }
}
```

`@WebMvcTest`는 전체 애플리케이션 컨텍스트를 띄우는 `@SpringBootTest`와 달리
**Controller와 관련된 빈(필터, `@ControllerAdvice`, JSON 컨버터 등)만 로드**한다.
Service는 `@MockBean`으로 대체되므로 실제 비즈니스 로직이나 DB는 전혀 개입하지
않는다. 이게 중요한 이유는 속도만이 아니다 — Controller 테스트에서 실패가 나면
"요청/응답 매핑 문제"라고 바로 좁혀서 판단할 수 있다는 게 진짜 이점이다. 전체
컨텍스트를 띄우는 테스트에서 같은 실패가 나면 원인이 Service 로직인지 Controller
매핑인지부터 구분해야 한다.

`jsonPath`로 응답 JSON의 특정 필드를 직접 검증하는 것도 포인트다 — 응답 객체
전체를 문자열로 비교하면 필드 순서나 포맷 변화에 테스트가 쉽게 깨지는데, `jsonPath`는
필드 단위로 검증하므로 이런 노이즈에 덜 민감하다.

## 외부 API 연동: 진짜 네트워크 없이 테스트하기

외부 API를 호출하는 코드가 가장 테스트하기 까다로운 이유는, 테스트 중에 실제
네트워크 호출이 나가면 느려지고 결과도 예측 불가능해지기 때문이다(외부 서비스
장애, 네트워크 지연이 테스트 실패로 나타난다). `MockWebServer`는 로컬에 가짜
HTTP 서버를 띄워서 이 문제를 해결한다.

```java
class ExternalApiServiceTest {

    private MockWebServer mockWebServer;
    private ExternalApiService apiService;

    @BeforeEach
    void setup() throws IOException {
        mockWebServer = new MockWebServer();
        mockWebServer.start();
        String baseUrl = mockWebServer.url("/").toString();
        apiService = new ExternalApiService(baseUrl);
    }

    @AfterEach
    void tearDown() throws IOException {
        mockWebServer.shutdown();
    }

    @Test
    void 외부_API_응답_테스트() throws Exception {
        String mockResponse = "{\"status\":\"OK\",\"data\":\"success\"}";
        mockWebServer.enqueue(new MockResponse()
                .setBody(mockResponse)
                .addHeader("Content-Type", "application/json"));

        ApiResponse result = apiService.callExternalApi();

        RecordedRequest request = mockWebServer.takeRequest();
        assertEquals("/external/api", request.getPath());
        assertEquals("success", result.getData());
    }
}
```

일반 Mockito로 HTTP 클라이언트 자체를 Mock하는 방식과 다른 점은, `MockWebServer`가
**실제 HTTP 레이어를 그대로 통과**한다는 것이다 — 요청이 실제로 직렬화되어
소켓을 타고 나가고, 응답도 실제로 역직렬화된다. 그래서 `RecordedRequest`로
"내 코드가 실제로 어떤 경로에, 어떤 헤더로 요청을 보냈는지"까지 검증할 수 있다.
Mockito로 클라이언트 자체를 Mock했다면 이 부분(요청이 올바르게 조립됐는가)은
검증 범위 밖으로 빠진다 — 요청 조립 로직에 버그가 있어도 테스트는 통과한다.
이 차이가 두 방식을 가르는 핵심이다.

## TDD 리듬: 계층이 달라도 사이클은 같다

1. **가정 세우기** — "이 입력이면 이렇게 동작해야 한다."
2. **테스트 작성** — 그 가정을 코드로 명확히 표현한다.
3. **구현하기** — 테스트를 통과할 최소한의 코드만 작성한다.
4. **리팩터링** — 테스트가 깨지지 않는 선에서 개선한다.

```java
// Given
when(paymentService.pay(any())).thenReturn(true);

// When
boolean result = orderService.processPayment("A001");

// Then
assertTrue(result);
verify(paymentService).pay(any());
```

`verify(paymentService).pay(any())`가 `assertTrue(result)`와 함께 있다는 걸
눈여겨볼 만하다 — 반환값만 확인하는 게 아니라 **"의존성이 실제로 호출됐는가"**까지
검증한다. 결과값이 우연히 맞아떨어졌을 뿐 실제로는 결제 로직이 호출되지 않은
경우(예: 로직 분기가 잘못돼서 다른 경로로 샜는데 우연히 같은 값을 반환)를
`verify` 없이는 잡아낼 수 없다.

## 정리

Service는 Mockito로 의존성을 갈아끼우고, Controller는 `@WebMvcTest`로 웹 계층만
떼어내고, 외부 API 연동은 `MockWebServer`로 진짜 HTTP 레이어를 통과시키되 네트워크는
로컬로 격리한다 — 세 계층 모두 "무엇을 진짜로 두고 무엇을 가짜로 대체할 것인가"를
계층의 책임에 맞춰 다르게 가져가는 게 핵심이다. TDD 사이클(가정→테스트→구현→리팩터링)
자체는 계층에 상관없이 동일하게 적용된다.
