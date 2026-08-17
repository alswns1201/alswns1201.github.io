---
title: "대용량 페이지네이션 API를 안전하게 순회하며 적재하기"
date: 2026-07-28
categories: [아키텍처]
tags: [java, api-design, architecture]
---

`nextUrl` 기반으로 페이지네이션된 API를 순회하면서 결과를 DB에 적재하는 로직을 짤
때는 두 가지 축을 구분해서 접근하는 게 좋다. 하나는 **한 번의 응답을 어떻게 메모리
효율적으로 처리할 것인가**이고, 다른 하나는 **수많은 페이지 반복을 어떻게 끝까지
안정적으로 처리할 것인가**다. 이 둘은 서로 다른 문제라서 해법도 다르다.

## 메모리: 통째로 읽지 않는다

가장 흔한 함정은 응답 전체를 문자열이나 트리로 통째로 읽는 방식이다.

```java
byte[] allBytes = inputStream.readAllBytes();
JsonNode root = objectMapper.readTree(allBytes);
```

이 방식은 두 단계에서 메모리를 소비한다. `readAllBytes()`가 스트림 전체를 하나의
byte 배열로 만드는 시점에 응답 크기만큼의 연속된 메모리가 필요하고, `readTree()`가
그걸 파싱해 `JsonNode` 객체 그래프로 만드는 시점엔 자바 객체 오버헤드가 더해져
원본보다 몇 배 큰 메모리가 필요해진다. 필요한 최대 메모리가 응답 크기에 비례해서
계속 커진다는 게 핵심 문제다.

대안은 HTTP 계층과 파싱 계층 **모두**에서 스트리밍 방식을 쓰는 것이다.

```java
public class StreamingJsonFetcher {

    private final HttpClient httpClient = HttpClient.newHttpClient();
    private final JsonFactory jsonFactory = new JsonFactory();
    private final ObjectMapper objectMapper = new ObjectMapper();
    private static final int CHUNK_SIZE = 500;

    public void fetchAll(String startUrl) throws IOException, InterruptedException {
        String url = startUrl;
        while (url != null) {
            url = fetchOnePage(url);
        }
    }

    private String fetchOnePage(String url) throws IOException, InterruptedException {
        HttpRequest request = HttpRequest.newBuilder(URI.create(url)).GET().build();

        // 응답 바디를 InputStream으로 받는다 - byte[]나 String으로 먼저 통째로 만들지 않는다
        HttpResponse<InputStream> response =
                httpClient.send(request, HttpResponse.BodyHandlers.ofInputStream());

        String nextUrl = null;
        List<ResultDto> chunk = new ArrayList<>(CHUNK_SIZE);

        try (InputStream body = response.body();
             JsonParser parser = jsonFactory.createParser(body)) {

            while (parser.nextToken() != null) {
                String fieldName = parser.getCurrentName();

                if ("result".equals(fieldName)) {
                    parser.nextToken(); // START_ARRAY로 이동
                    while (parser.nextToken() != JsonToken.END_ARRAY) {
                        // 배열 원소 하나만 트리로 변환 - 원소 단위로만 메모리 사용
                        ResultDto dto = objectMapper.readValue(parser, ResultDto.class);
                        chunk.add(dto);

                        if (chunk.size() >= CHUNK_SIZE) {
                            batchInsert(chunk);
                            chunk.clear();
                        }
                    }
                } else if ("nextUrl".equals(fieldName)) {
                    parser.nextToken();
                    nextUrl = parser.getValueAsString();
                }
            }
        }

        if (!chunk.isEmpty()) {
            batchInsert(chunk);
        }
        return nextUrl;
    }

    private void batchInsert(List<ResultDto> chunk) {
        jdbcTemplate.batchUpdate(
            "INSERT INTO result_table (id, value) VALUES (?, ?)",
            chunk, chunk.size(),
            (ps, dto) -> {
                ps.setLong(1, dto.getId());
                ps.setString(2, dto.getValue());
            }
        );
    }
}
```

### `InputStream`으로 받는다는 것의 의미

`BodyHandlers.ofString()`을 쓰면 `HttpClient` 내부에서 소켓 응답을 끝까지 읽어
byte 배열로 모은 뒤 문자열로 디코딩한다 — 이 시점에 이미 응답 크기만큼의 메모리가
소비된 상태다. 반면 `ofInputStream()`은 응답을 미리 다 읽지 않고, 소켓 버퍼에서
필요한 만큼씩만 읽어 넘겨주는 스트림을 반환한다. `read`를 호출하는 시점에 실제로
네트워크에서 데이터를 가져온다.

다만 `InputStream`을 받는 것 자체가 메모리 절약을 보장하지는 않는다. 받은 스트림에
`readAllBytes()`를 호출해버리면 결국 전체를 한 번에 메모리로 끌어오는 것과
동일해진다. 진짜 이점을 보려면 스트림을 소비하는 **파싱 계층도** 끝까지 한 번에
읽지 않고 필요한 만큼만 순차적으로 읽어야 한다. HTTP 계층과 파싱 계층 둘 다
스트리밍이어야 전체 파이프라인이 메모리 효율적이다.

### 토큰 단위로 읽는다는 것의 의미

`JsonParser`는 내부적으로 크지 않은 고정 크기 버퍼만 가진다. `nextToken()`을
호출하면 파서는 버퍼에 아직 해석할 데이터가 남았는지 확인하고, 비어 있으면 그때
`InputStream`으로부터 버퍼 크기만큼만 추가로 읽는다. `{"result": [{"id":1}, {"id":2}], "nextUrl": "..."}`를
읽으면 대략 이런 토큰 시퀀스가 순서대로 나온다.

```
START_OBJECT
FIELD_NAME       ("result")
START_ARRAY
START_OBJECT
FIELD_NAME       ("id")
VALUE_NUMBER_INT (1)
END_OBJECT
...
END_ARRAY
FIELD_NAME       ("nextUrl")
VALUE_STRING     ("...")
END_OBJECT
```

`objectMapper.readValue(parser, ResultDto.class)`처럼 상위 레벨 메서드를 쓰면 그
내부에서 `START_OBJECT`부터 `END_OBJECT`까지 토큰을 자동으로 순회해 객체 하나로
조립해주므로, 실제 코드를 짤 땐 "객체 하나 단위로 읽는다"는 감각으로 다뤄도 된다.

### 통째로 읽는 방식과의 차이

두 방식 모두 결국 네트워크에서 바이트를 읽어온다는 점은 같다. 차이는 그 바이트를
**동시에 얼마나 오래, 얼마나 많이 메모리에 붙잡아두는가**에 있다. 통째로 읽는
방식은 응답을 다 모을 때까지 계속 메모리에 쌓아두므로 최대 메모리 사용량이 응답
크기에 비례해서 커진다. 스트리밍 방식은 지나간 토큰은 바로 버려지고, 조립된 객체도
청크 크기만큼 쌓이면 DB에 저장한 뒤 참조를 끊기 때문에, 어느 순간에도 메모리에
살아있는 데이터양이 파서 버퍼와 청크 크기 정도로 상한이 고정된다. 이 상한은
원본 응답이 얼마나 크든 바뀌지 않는다.

## 안정성: 페이지 수백만 번을 실패 없이 돌기

이건 메모리보다는 시간·안정성·장애 복구의 문제다. 전체를 하나의 긴 프로세스로
처리하면, 중간에 API가 타임아웃되거나 애플리케이션이 재시작될 때 처음부터 다시
시작해야 한다.

**체크포인트**: 마지막으로 성공한 URL을 DB나 Redis에 기록해두고, 재시작 시 그
지점부터 이어서 처리하도록 설계해야 한다.

**멱등성**: 체크포인트 기반 재시작이 안전하려면 DB insert가 반드시 멱등적이어야
한다 — 같은 URL을 두 번 처리해도 중복 저장되지 않도록 유니크 제약과 upsert를
함께 쓴다.

**순차 처리라는 제약**: `nextUrl`이 응답에 포함되어야만 다음 페이지를 알 수 있는
구조라면 본질적으로 순차 처리일 수밖에 없다. 이런 경우엔 처리량을 늘리는 것보다
재시도와 체크포인트를 통한 안정성 확보에 집중하는 편이 맞다. 외부 API 호출이
반복되는 만큼 일시적 네트워크 오류나 rate limit에 대비한 지수 백오프 재시도, 반복
실패 시 회로를 차단하는 처리도 필요하다.

**직접 짜는 대신 프레임워크에 맡기는 선택지**: 이 정도 규모라면 직접 루프를 짜는
것보다 Spring Batch의 Chunk-oriented Step 구조를 검토할 만하다. `ItemReader`가
`nextUrl`을 따라가며 스트리밍으로 읽고 `ItemWriter`가 청크 단위로 배치 삽입하는
구조를 쓰면, 체크포인트·재시작·트랜잭션 경계 관리를 프레임워크가 대신 처리해준다.

## 정리

메모리 사용량은 청크 크기로 고정하고, 전체 진행 상황은 체크포인트로 외부화하는
것이 핵심이다. 이렇게 설계하면 응답이 아무리 크거나 페이지가 아무리 많아도
애플리케이션의 메모리 사용량과 장애 복구 가능성은 일정하게 유지된다.
