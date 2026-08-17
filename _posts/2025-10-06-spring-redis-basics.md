---
title: "Spring Boot와 Redis: 기본 개념과 실무에서 흔히 놓치는 것들"
date: 2025-10-06
categories: [아키텍처]
tags: [redis, spring-boot, caching]
---

Redis는 캐시, 큐, 세션 관리 등으로 두루 쓰이는 메모리 기반 Key-Value 저장소다.
Spring Boot에서는 대개 `RedisTemplate`을 통해 다루는데, 기본 개념만 알고 쓰면
운영에서 뒤늦게 발견되는 함정이 몇 가지 있다.

## Redis의 기본 개념

Key는 항상 문자열이지만, Value는 다양한 자료구조를 지원한다.

| 자료구조 | 특징 | 대표 명령 |
|---|---|---|
| String | 단일 값 | SET, GET |
| List | 순서 있는 값들의 리스트 | LPUSH, RPUSH, LRANGE |
| Set | 중복 없는 집합 | SADD, SMEMBERS |
| Hash | 필드-값 맵 | HSET, HGETALL |
| Sorted Set (ZSet) | 점수 기반 정렬 집합 | ZADD, ZRANGE |
| Stream | 메시지 큐 형식 | XADD, XREAD |

메모리 기반이라 조회/쓰기가 빠르고, TTL을 걸면 일정 시간 뒤 자동 삭제된다. 이 특징
때문에 캐시·랭킹·세션처럼 "빠르게 읽고 쓰되, 없어져도 치명적이지 않은" 데이터에
적합하다.

## Spring Boot 연동

```java
@Configuration
public class RedisConfiguration {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("localhost");
        config.setPort(6379);
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        return redisTemplate;
    }
}
```

여기서 바로 실무 함정이 하나 시작된다: **`RedisTemplate`을 직렬화 설정 없이 그대로
쓰면 기본값은 JDK 직렬화다.** 그 결과 `redis-cli`로 값을 들여다보면 사람이 읽을 수
없는 바이트 덩어리가 저장되어 있다 — 디버깅이 사실상 불가능해진다. `StringRedisSerializer`나
`Jackson2JsonRedisSerializer`를 key/value에 명시적으로 지정해야 실무에서 쓸 만한
형태가 된다. 이 설정을 빼먹은 채로 배포하고 나서야 "왜 캐시 값이 깨져 보이지"라고
묻는 경우가 흔하다.

## 값 캐싱: ValueOperations

```java
@Service
public class RedisValueCache {
    private final ValueOperations<String, Object> valueOps;

    public RedisValueCache(RedisTemplate<String, Object> redisTemplate) {
        valueOps = redisTemplate.opsForValue();
    }

    public void cache(String key, Object data) {
        valueOps.set(key, data, 40000, TimeUnit.MILLISECONDS); // TTL 명시
    }

    public Object getCachedValue(String key) {
        return valueOps.get(key);
    }
}
```

TTL을 명시하지 않고 `set(key, data)`만 호출하면 해당 키는 **영구히** 남는다. 캐시
용도로 Redis를 쓰면서 TTL을 빼먹으면, 시간이 지날수록 오래된/만료됐어야 할 데이터가
쌓여 메모리를 잠식한다 — Redis는 기본적으로 알아서 청소해주지 않는다.

## 리스트 캐싱: 자료구조 선택이 곧 설계 결정이다

```java
public void cachePersons(String key, List<PersonDTO> persons) {
    for (PersonDTO person : persons) {
        listOps.leftPush(key, person);
    }
}

public List<PersonDTO> getPersonsInRange(String key, RangeDTO range) {
    List<Object> objects = listOps.range(key, range.getFrom(), range.getTo());
    return objects.stream().map(x -> (PersonDTO) x).collect(Collectors.toList());
}
```

Java의 `List`나 일반 컬렉션 문법만으로는 Redis에 저장할 수 없다 — `ListOperations`
같은 Redis 전용 연산을 써야 한다. 이건 단순한 API 차이가 아니라 설계 결정이다.
"최근 N개의 로그", "큐", "순서가 있는 캐시 데이터" 같은 요구사항이면 List, 랭킹처럼
정렬이 필요하면 Sorted Set, 단순 존재 여부 확인이면 Set — **자료구조를 잘못 고르면
나중에 요구사항이 조금만 바뀌어도(예: "최근 N개"에서 "점수 상위 N개"로) 데이터
구조 자체를 다시 설계해야 한다.**

## Redis를 쓸 때 실제로 던져야 할 질문

Redis를 도입하기 전에 확인해야 할 것은 "빠른가"가 아니라 "이 데이터가 사라져도
괜찮은가"다. Redis는 기본적으로 인메모리이므로, 영속성 설정(RDB 스냅샷, AOF)을
따로 하지 않으면 프로세스 재시작만으로 데이터가 날아간다. 영속성을 켜더라도
RDBMS 수준의 트랜잭션 원자성을 기대할 수는 없다. 그래서 원칙은 명확하다 — **빠른
읽기/쓰기와 휘발성이 허용되는 데이터만 Redis에 올리고, 영구 보존이 필요한 핵심
데이터는 RDB(MySQL, PostgreSQL)에 둔다.** Redis를 "사실상의 주 데이터 저장소"로
쓰기 시작하는 순간, 원래 얻으려던 단순함과 속도라는 이점을 잃고 별도의 정합성
문제를 떠안게 된다.

## 정리

- 직렬화 설정을 빼먹으면 캐시가 사람이 읽을 수 없는 바이트로 저장된다 — 반드시
  String/JSON 직렬화를 명시할 것.
- TTL을 명시하지 않은 캐시는 캐시가 아니라 그냥 무기한 저장소가 된다.
- 자료구조 선택은 성능 문제가 아니라 설계 문제다 — 나중에 요구사항이 바뀌면
  자료구조부터 다시 골라야 할 수 있다.
- Redis에 올릴 데이터인지 판단하는 기준은 "빠른가"가 아니라 "사라져도 되는가"다.
