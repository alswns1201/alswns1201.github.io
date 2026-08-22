---
title: "RestTemplate로 짜던 Elasticsearch 쿼리를 elasticsearch-java로 옮기며 만난 함정 두 가지"
date: 2026-04-23
categories: [Java/Spring]
---

기존 프로젝트는 Elasticsearch에 쿼리를 던질 때 `RestTemplate`으로 REST API를 직접
호출하고 있었다. 쿼리 바디는 `Map`을 중첩해서 손으로 조립했다.

```java
Map<String, Object> body = Map.of(
    "query", Map.of(
        "bool", Map.of(
            "filter", List.of(
                Map.of("term", Map.of("user_id", userId))
            )
        )
    )
);
ResponseEntity<Map> response = restTemplate.exchange(url, HttpMethod.POST, new HttpEntity<>(body, headers), Map.class);
```

쿼리가 조금만 복잡해져도 `Map` 중첩이 몇 겹씩 쌓이고, 필드명을 오타내거나
구조를 잘못 짜도 컴파일 타임에는 아무것도 걸리지 않는다 — 런타임에 Elasticsearch가
에러를 뱉어야 비로소 알게 된다. 공식 `elasticsearch-java` 클라이언트로 옮기면
빌더 기반의 타입세이프한 쿼리를 쓸 수 있어서 전환을 진행했다.

```gradle
implementation 'co.elastic.clients:elasticsearch-java:8.13.0'
implementation 'com.fasterxml.jackson.core:jackson-databind:2.17.0'
```

`elasticsearch-java`는 내부적으로 `elasticsearch-rest-client`를 쓰고, JSON
직렬화는 Jackson이 처리한다. 도입 과정에서 두 가지 함정을 만났다.

## 함정 1 — SSL 설정에서 httpclient4/5가 섞인다

`ElasticsearchConfig`를 처음 작성할 때는 자연스럽게 httpclient5 패키지로 SSL을
설정했다.

```java
import org.apache.hc.client5.http.auth.AuthScope;                     // httpclient5
import org.apache.hc.client5.http.impl.auth.BasicCredentialsProvider; // httpclient5
import org.apache.hc.core5.http.HttpHost;                             // httpclient5

RestClient restClient = RestClient.builder(new HttpHost(scheme, ip, port)) // 타입 오류
    .setHttpClientConfigCallback(httpClientBuilder ->
        httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider) // 타입 불일치
    ).build();
```

`RestClient.builder()`에서 바로 타입 오류가 난다. 원인은 `elasticsearch-java`
8.x의 `RestClient`가 **공개 API는 httpclient4 기반이고, 내부 구현에서만
httpclient5를 쓰는 이중 구조**라는 데 있다. 즉 `RestClient.builder()`에 넘기는
`HttpHost`와 콜백 안의 `CredentialsProvider`는 전부 httpclient4 패키지여야 한다.
같은 클래스 이름(`HttpHost`, `BasicCredentialsProvider` 등)이 두 패키지에 다
존재해서, IDE 자동 임포트를 무심코 받아들이면 딱 이 실수를 하게 된다.

```java
import org.apache.http.HttpHost;                             // httpclient4
import org.apache.http.auth.AuthScope;                       // httpclient4
import org.apache.http.auth.UsernamePasswordCredentials;     // httpclient4
import org.apache.http.conn.ssl.NoopHostnameVerifier;        // httpclient4
import org.apache.http.impl.client.BasicCredentialsProvider; // httpclient4
import org.apache.http.ssl.SSLContextBuilder;                // httpclient4

@Bean
public ElasticsearchClient elasticsearchClient() throws Exception {
    SSLContext sslContext = SSLContextBuilder.create()
            .loadTrustMaterial(null, (chain, authType) -> true) // 자체 서명 인증서 신뢰
            .build();

    BasicCredentialsProvider credentialsProvider = new BasicCredentialsProvider();
    credentialsProvider.setCredentials(
            new AuthScope(ip, port),
            new UsernamePasswordCredentials(user, pass)
    );

    RestClient restClient = RestClient.builder(new HttpHost(ip, port, scheme))
            .setHttpClientConfigCallback(httpClientBuilder ->
                    httpClientBuilder
                            .setSSLContext(sslContext)
                            .setSSLHostnameVerifier(NoopHostnameVerifier.INSTANCE)
                            .setDefaultCredentialsProvider(credentialsProvider)
            )
            .build();

    ElasticsearchTransport transport = new RestClientTransport(restClient, new JacksonJsonpMapper());
    return new ElasticsearchClient(transport);
}
```

핵심은 관련 클래스를 전부 httpclient4(`org.apache.http.*`) 패키지로 통일하는
것이다. 버전이 섞이면 컴파일 타임 타입 불일치로 바로 걸리긴 하지만, 에러
메시지만 봐서는 "패키지를 잘못 골랐다"는 게 바로 안 와닿아서 처음엔 헤맸다.

## 함정 2 — 제네릭 타입 소거 때문에 `Map<String, Object>.class`가 안 된다

검색 결과를 `Map<String, Object>`로 받고 싶어서 이렇게 썼더니 컴파일이 안 됐다.

```java
Class<Map<String, Object>> clazz = Map.class; // 컴파일 에러
```

Java 제네릭은 컴파일 타임에만 존재하고 런타임에는 지워진다(타입 소거). 그래서
`Map<String, Object>.class`라는 표현 자체가 Java 문법에 없다 — `Map.class`(raw
type)만 존재한다. `esClient.search()`는 결과 타입을 `Class<T>`로 요구하는데,
제네릭이 소거된 런타임 시점에는 `Map<String, Object>`라는 타입 정보를 애초에
전달할 방법이 없는 것이다. 우회 방법은 `Class<?>`를 경유한 강제 캐스팅이다.

```java
@SuppressWarnings("unchecked")
public List<Map<String, Object>> search(String index, String field, String keyword) {
    SearchResponse<Map<String, Object>> response = esClient.search(s -> s
                    .index(index)
                    .query(q -> q
                            .match(m -> m.field(field).query(keyword))
                    ),
            (Class<Map<String, Object>>) (Class<?>) Map.class  // 강제 캐스팅
    );

    return response.hits().hits().stream()
            .map(Hit::source)
            .collect(Collectors.toList());
}
```

`(Class<Map<String, Object>>) (Class<?>) Map.class`는 `Class<?>`를 한 번 거쳐서
캐스팅하는 우회다. 컴파일러는 이 캐스팅의 타입 안전성을 증명할 수 없기 때문에
`unchecked` 경고를 낸다. `@SuppressWarnings("unchecked")`는 "이 캐스팅이
불가피하다는 걸 알고 의도적으로 썼다"는 선언이지, 경고를 그냥 지우는 버튼이
아니다 — 무분별하게 붙이면 진짜 타입 오류까지 같이 숨어버리므로, 원인이
분명한 지점에만 최소 범위로 적용해야 한다.

## 최종 구조와 검증

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class ElasticsearchService {

    private final ElasticsearchClient esClient;

    public List<Map<String, Object>> search(String index, String field, String keyword) { ... }
    public List<Map<String, Object>> findAll(String index, int size) { ... }
    public boolean ping() { ... }
}
```

실제 ES 서버에 붙는 통합 테스트를 작성했는데, `ping()`을 가장 먼저 검증하는 게
핵심이다.

```java
@SpringBootTest(classes = {ElasticsearchConfig.class, ElasticsearchService.class})
class ElasticsearchServiceTest {

    @Autowired
    ElasticsearchService elasticsearchService;

    @Test
    void ping() {
        boolean result = elasticsearchService.ping();
        assertThat(result).isTrue(); // false면 버전 불일치 또는 연결 실패
    }

    @Test
    void findAll() {
        List<Map<String, Object>> result = elasticsearchService.findAll("sensor_data_250101", 10);
        assertThat(result).isNotNull();
    }
}
```

`ping()`이 `false`를 반환하면 검색 로직을 파기 전에 두 가지부터 의심한다:
ES 서버와 클라이언트 버전이 안 맞는지, SSL 인증서 설정이 잘못됐는지. 검색
쿼리 자체의 문제와 연결/인증 문제를 미리 분리해두면 디버깅 범위가 확 줄어든다.

## 정리

- `Map`을 중첩해서 REST API를 직접 호출하는 방식은 컴파일 타임에 아무것도
  못 잡는다. `elasticsearch-java`의 빌더 기반 API로 옮기면 오타나 구조
  실수를 컴파일 타임에 잡을 수 있다.
- `elasticsearch-java` 8.x는 공개 API가 httpclient4, 내부 구현이 httpclient5인
  이중 구조다. SSL 설정에 쓰는 클래스는 반드시 `org.apache.http.*`(httpclient4)로
  통일해야 한다.
- 제네릭 타입 소거 때문에 `Map<String, Object>.class`는 애초에 존재하지 않는
  문법이다. `Class<?>`를 경유한 캐스팅 + 좁은 범위의 `@SuppressWarnings("unchecked")`가
  현실적인 우회다.
- 연결 상태를 확인하는 `ping()`을 검색 로직보다 먼저 테스트해두면, 실패했을 때
  "연결/인증 문제"와 "쿼리 문제"를 곧바로 구분할 수 있다.
