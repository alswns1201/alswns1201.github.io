---
title: "Spring Boot에서 Bucket4j로 API 요청 제한(Rate Limiting) 구현하기"
date: 2025-10-26
categories: [Java/Spring]
---

특정 클라이언트가 과도하게 요청을 보내면 서버 자원을 잠식하고 다른 사용자의 응답
속도까지 떨어뜨린다. 이걸 막는 게 **Rate Limiting**이고, Java 진영에서 간단하고
강력한 선택지 중 하나가 **Bucket4j**다.

## 토큰 버킷 알고리즘

Bucket4j는 **토큰 버킷(Token Bucket)** 알고리즘을 구현한다. 동작은 단순하다.

1. 버킷에 일정량의 토큰이 들어 있다.
2. 요청이 올 때마다 토큰을 하나 꺼낸다.
3. 버킷이 비면 요청은 거부된다(HTTP 429).
4. 일정 주기마다 토큰이 다시 채워진다(refill).

이 알고리즘이 왜 "요청 개수 세서 막기"보다 나은지가 핵심이다. 단순 카운터 방식(예:
"1분에 60개")은 시간 창의 경계에서 버스트를 허용해버리는 문제가 있다 — 59초에 60개,
1초 뒤(다음 창의 시작)에 또 60개를 보내면 2초 만에 120개가 통과한다. 토큰 버킷은
토큰이 연속적으로 리필되기 때문에 이런 경계 버스트 문제가 상대적으로 덜하다.

## 설정

```java
@Configuration
public class Bucket4jConfig {

    @Value("${bucket.capacity}")
    private int capacity;

    @Value("${bucket.refill.tokens}")
    private int refillTokens;

    @Value("${bucket.refill.duration.seconds}")
    private int refillDuration;

    @Bean
    public Bucket bucket() {
        Refill refill = Refill.intervally(refillTokens, Duration.ofSeconds(refillDuration));
        Bandwidth limit = Bandwidth.classic(capacity, refill);
        return Bucket.builder().addLimit(limit).build();
    }
}
```

```yaml
bucket:
  capacity: 10           # 버킷 최대 토큰 수 (순간 허용 버스트)
  refill:
    tokens: 10            # 매 주기마다 채워지는 토큰 수
    duration:
      seconds: 1           # 리필 주기
```

이 설정은 "초당 10회 허용"을 의미하는데, `capacity`와 `refillTokens`를 분리해서
이해하는 게 중요하다. `capacity`는 순간적으로 허용할 수 있는 버스트 크기이고,
`refillTokens`/`refillDuration`은 정상 상태에서의 평균 처리율이다. 만약
`capacity=50`, `refillTokens=10`(초당)으로 설정하면, 평소엔 초당 10개로 제한되지만
버킷이 가득 차 있었다면 순간적으로 50개까지 몰아서 받아줄 수 있다 — 클라이언트가
잠깐 쉬었다가 몰아서 요청하는 정상적인 사용 패턴이 있다면 이 여유가 유용하고, 반대로
버스트 자체를 절대 허용하면 안 되는 경우엔 `capacity`를 `refillTokens`와 같게
맞춘다.

## 실제 적용

```java
@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class ApiController {

    private final Bucket bucket;

    @GetMapping("/data")
    public ResponseEntity<String> getData() {
        if (bucket.tryConsume(1)) {
            return ResponseEntity.ok("정상 요청");
        }
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .body("요청 한도를 초과했습니다. 잠시 후 다시 시도해주세요.");
    }
}
```

## 놓치기 쉬운 지점: 이 버킷은 누구의 것인가

위 예시처럼 `Bucket`을 싱글턴 빈으로 등록하면, **모든 클라이언트가 하나의 버킷을
공유**한다 — 즉 전체 API에 대한 글로벌 rate limit이다. 실무에서 필요한 건 보통
"클라이언트별", "API 키별", "IP별" 제한이다. 이걸 하려면 버킷을 클라이언트 식별자
기준으로 캐시에 담아 관리해야 한다(예: `Map<String, Bucket>` 또는 Caffeine 캐시).
단일 빈 버킷을 그대로 쓰면 한 클라이언트가 한도를 다 쓰는 순간 다른 모든 사용자까지
막혀버리는 문제가 생긴다 — 이건 설계 실수를 코드 리뷰에서 놓치기 쉬운 지점이다.

## 분산 환경에서는 다르게 접근해야 한다

이 구현은 **단일 인스턴스 기준**이다. 서버가 여러 대로 스케일 아웃되면, 인스턴스마다
자기만의 버킷을 갖게 되어 실제 허용량이 인스턴스 수만큼 배가된다 — 인스턴스 3대에
각각 "초당 10개" 제한을 걸면 전체로는 초당 30개까지 통과한다. 이게 의도한 게 아니라면
Redis 같은 공유 저장소에 버킷 상태를 두고 모든 인스턴스가 같은 카운터를 보게 해야
한다. Bucket4j는 Redis 연동을 지원하므로, 분산 환경에서 정확한 전역 제한이 필요하면
이쪽으로 넘어가야 한다.

## 정리

- Bucket4j는 토큰 버킷 알고리즘으로 요청을 제어하는 라이브러리다.
- `capacity`(버스트 허용량)와 `refillTokens`/`refillDuration`(평균 처리율)을 구분해서
  이해해야 정책을 의도대로 설계할 수 있다.
- 버킷을 싱글턴으로 두면 전역 제한이 된다 — 클라이언트별 제한이 필요하면 식별자 기준
  버킷 관리가 필요하다.
- 다중 인스턴스 환경에서는 로컬 버킷만으로는 전역 제한이 보장되지 않는다 — Redis 백엔드가
  필요하다.
