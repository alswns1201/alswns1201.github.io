---
title: "Redis Cluster 이해하기 — 해시 슬롯, 해시 태그, 멀티 키 명령과 Lua 스크립트 규칙"
date: 2026-10-17
categories: [Redis]
redirect_from:
  - /posts/redis-cluster-lua-sms-ip-limit/
---

단일 Redis에서 잘 돌던 코드가 **Redis Cluster**로 옮기면 갑자기 `CROSSSLOT`, `MOVED` 같은 에러를 낸다.
클러스터가 데이터를 어떻게 나누는지 알면 왜 그런지, 키를 어떻게 지어야 하는지가 보인다.

이 글은 Redis Cluster의 핵심 개념(해시 슬롯, 리다이렉션, 복제와 페일오버)과, 그 위에서 **여러 키를 다루는 명령과 Lua 스크립트를 쓸 때의 규칙**을 정리한다.
마지막에 IP별 요청 횟수 제한을 예시로 붙였다.

> Lua 스크립트 자체(`EVAL`, `KEYS`/`ARGV`, `redis.call`)가 처음이라면 [Redis Lua 스크립트 쉽게 이해하기](/posts/redis-lua-script-daily-limit/)를 먼저 보면 좋다.
> 키 이름·노드 주소·설정값은 모두 예시다.
>
> 실습 코드: [github.com/alswns1201/redis-cluster-rate-limit](https://github.com/alswns1201/redis-cluster-rate-limit) — 6노드 클러스터, 해시 태그와 CROSSSLOT 재현, 동시 요청 테스트

## Redis Cluster가 푸는 문제

단일 Redis(+ 복제본)는 **메모리와 쓰기 처리량이 노드 한 대에 묶인다.** 데이터가 그보다 커지거나 쓰기가 몰리면 여러 노드에 **나눠 담아야(샤딩)** 한다.

Redis Cluster는 별도 프록시 없이 Redis 노드들끼리 데이터를 나눠 갖는 방식이다.

- 데이터를 여러 **마스터**에 나눠 담는다(샤딩).
- 마스터마다 **복제본**을 두고, 마스터가 죽으면 복제본이 자동으로 승격된다(페일오버).
- 클라이언트는 "어떤 키가 어느 노드에 있는지"를 알고 **해당 노드로 바로** 요청한다.

## 해시 슬롯: 키가 노드를 정하는 방법

클러스터는 키 공간을 **16,384개의 해시 슬롯**으로 나누고, 각 마스터가 슬롯 범위를 나눠 맡는다.

```
slot = CRC16(key) mod 16384

마스터 A : 0     ~ 5460
마스터 B : 5461  ~ 10922
마스터 C : 10923 ~ 16383
```

- 키 → 슬롯은 **항상 같은 계산**이라 어느 클라이언트든 같은 결과가 나온다.
- 노드를 추가하면 키를 하나하나 옮기는 게 아니라 **슬롯 단위로** 옮긴다(리샤딩).
- `CLUSTER KEYSLOT <key>`로 키의 슬롯을 직접 확인할 수 있다.

```
127.0.0.1:7000> CLUSTER KEYSLOT user:42
(integer) 9234
127.0.0.1:7000> CLUSTER KEYSLOT user:43
(integer) 13299      ← 이웃한 키라도 슬롯은 제각각이다
```

## 리다이렉션: MOVED와 ASK

클라이언트가 키를 엉뚱한 노드로 보내면, 노드는 데이터를 대신 가져다주지 않고 **"그 키는 저기 있다"** 고 알려 준다.

```
GET user:42
-MOVED 9234 10.0.0.2:6379
```

| 응답 | 의미 | 클라이언트 동작 |
|---|---|---|
| `MOVED` | 이 슬롯의 담당 노드가 바뀌었다(영구) | 슬롯 표를 갱신하고 새 노드로 다시 보냄 |
| `ASK` | 이 슬롯을 옮기는 중이다(일시) | 이번 요청만 `ASKING` 후 지정 노드로 보냄, 표는 안 바꿈 |

그래서 클러스터용 클라이언트(Lettuce, Jedis 등)는 **슬롯 → 노드 표**를 들고 다니면서 처음부터 맞는 노드로 보낸다. 페일오버나 리샤딩으로 표가 바뀌면 `MOVED`를 받으며 갱신한다.

## 멀티 키 명령의 제약: 같은 슬롯만

하나의 명령이 **노드 여러 대에 걸쳐 원자적으로 실행되지는 않는다.** 그래서 여러 키를 한 번에 다루는 명령은 모든 키가 **같은 슬롯**에 있어야 한다.

- `MGET`, `MSET`, `DEL k1 k2`, `SUNION`, `RENAME` 등
- `MULTI`/`EXEC` 트랜잭션
- **Lua 스크립트**

다르면 실행 자체가 거절된다.

```
127.0.0.1:7000> MGET user:42 user:43
(error) CROSSSLOT Keys in request don't hash to the same slot
```

> 일부 클라이언트는 `MGET`을 슬롯별로 쪼개 여러 노드에 보낸 뒤 합쳐 주기도 한다. 하지만 그건 **여러 개의 명령**이라 원자성이 없다. 트랜잭션과 Lua는 쪼갤 수 없으니 반드시 같은 슬롯이어야 한다.

## 해시 태그: 같은 슬롯에 모으는 방법

키에 `{...}`가 있으면 클러스터는 **중괄호 안의 문자열로만** 슬롯을 계산한다.

```
cart:{user:42}:items     ─┐
cart:{user:42}:total     ─┼─ 모두 "user:42"로 계산 → 같은 슬롯
order:{user:42}:latest   ─┘
```

- 첫 번째 `{`와 그 뒤 첫 번째 `}` 사이만 쓴다. 중괄호 안이 비어 있으면(`{}`) 키 전체로 계산한다.
- **함께 바꿔야 하는 키**(한 사용자의 장바구니와 합계, 한 IP의 분당·일별 카운터 등)를 같은 해시 태그로 묶는다.

### 해시 태그의 대가: 쏠림

같은 태그를 가진 키는 전부 한 노드로 간다. 태그를 너무 크게 잡으면(예: 모든 키에 `{global}`) 클러스터를 쓰는 의미가 사라지고 한 노드에 부하가 몰린다.
**"원자적으로 같이 다뤄야 하는 범위"만큼만** 묶는다. 보통 사용자 ID, 주문 ID, IP처럼 값이 충분히 많은 단위가 적당하다.

## Lua 스크립트의 클러스터 규칙

Lua 스크립트는 **한 노드에서 통째로 실행**된다. 그래서 두 가지 규칙이 생긴다.

### 1) 스크립트가 쓰는 키는 모두 같은 슬롯

멀티 키 명령과 같은 이유다. 스크립트 안에서 다른 슬롯의 키를 건드리면 에러가 난다. 키가 둘 이상이면 해시 태그로 묶는다.

### 2) 키는 반드시 `KEYS`로 넘긴다

```lua
-- ❌ 스크립트 안에서 키 이름을 만든다
redis.call('INCR', 'rate:' .. ARGV[1])

-- ✅ 키는 밖에서 KEYS로 넘긴다
redis.call('INCR', KEYS[1])
```

클라이언트는 **`KEYS`로 넘어온 키를 보고** 슬롯을 계산해 노드를 고른다. 스크립트 안에서 키를 조합하면 클라이언트는 어느 노드로 보내야 할지 모르고, 엉뚱한 노드에서 실행되면 그 노드에 없는 키를 만들거나 에러가 난다.
단일 Redis에서는 둘 다 돌아가서 실수를 눈치채기 어렵다. 처음부터 `KEYS` 규칙을 지켜 두면 클러스터로 옮길 때 코드를 바꿀 필요가 없다.

### 3) 스크립트 캐시는 노드별

`EVALSHA`로 보낼 때 그 노드에 스크립트가 없으면 `NOSCRIPT` 에러가 난다. 노드가 재시작되거나, 페일오버로 승격된 복제본이거나, 처음 가는 노드면 캐시가 비어 있다.
Spring Data Redis의 `RedisScript`는 `NOSCRIPT`를 받으면 자동으로 `EVAL`로 다시 보낸다. 직접 구현할 때는 이 재시도를 넣어야 한다.

### 4) 오래 걸리는 스크립트는 노드를 멈춘다

스크립트가 도는 동안 그 노드는 다른 명령을 처리하지 않는다. 클러스터에서도 마찬가지라 **그 노드가 맡은 슬롯 전체**가 멈춘다. 반복문으로 많은 키를 훑는 스크립트는 피한다.

## 복제와 페일오버에서 알아 둘 것

- 복제는 **비동기**다. 마스터가 쓰기에 응답한 뒤 복제본에 전달하기 전에 죽으면, 승격된 복제본에는 그 쓰기가 없다. 카운터 몇 개가 사라지는 정도는 괜찮지만, "절대 잃으면 안 되는 상태"를 Redis에만 두면 안 되는 이유다.
- 페일오버로 마스터가 바뀌면 클라이언트의 슬롯 표를 갱신해야 한다. 갱신이 늦으면 죽은 노드로 요청을 보내 실패하거나 리다이렉트가 이어지므로, **토폴로지 자동 갱신**을 켜 두는 게 좋다.

```yaml
spring:
  data:
    redis:
      cluster:
        nodes:
          - redis-node-1.example.internal:6379
          - redis-node-2.example.internal:6379
          - redis-node-3.example.internal:6379
      lettuce:
        cluster:
          refresh:
            adaptive: true    # MOVED·연결 끊김 등 신호가 오면 바로 갱신
            period: 30s       # 주기적 갱신
```

노드 주소를 클러스터 설정으로 주면 `RedisTemplate`이 클러스터 모드로 동작하고, 애플리케이션 코드는 단일 Redis와 같다.

- `KEYS *`나 `SCAN`은 **접속한 노드 하나만** 훑는다. 전체를 보려면 노드마다 돌아야 하니, "어떤 사용자의 키 전부" 같은 조회는 별도 인덱스 키(Set 등)로 설계한다.
- 클러스터는 **DB 번호 0만** 쓴다. `SELECT 1` 같은 분리는 안 된다.

## 예시: IP별 요청 횟수 제한

로그인 전에 누구나 호출하는 API(예: 인증 문자 발송)는 반복 호출되면 비용과 피해가 생긴다. 서버가 여러 대면 서버 메모리 카운터는 서버마다 따로 세니, 공용 Redis에 카운터를 둔다.

`INCR` 뒤에 `EXPIRE`를 따로 보내면, 그 사이에 서버가 죽었을 때 **만료 없는 키**가 남아 그 IP가 영구 차단된다. 그래서 둘을 Lua로 묶고, 앞의 규칙대로 키는 `KEYS`로 넘기고 해시 태그로 IP 단위를 묶는다.

```lua
-- KEYS[1] = rate:{1.2.3.4}   ARGV[1] = 한도, ARGV[2] = 기간(초)
local count = redis.call('INCR', KEYS[1])
if count == 1 then
  redis.call('EXPIRE', KEYS[1], tonumber(ARGV[2]))   -- 첫 요청에만 만료 설정
end
local ttl = redis.call('TTL', KEYS[1])
if ttl < 0 then                                        -- 만료 없는 키가 남아 있으면 복구
  redis.call('EXPIRE', KEYS[1], tonumber(ARGV[2]))
  ttl = tonumber(ARGV[2])
end
return { count <= tonumber(ARGV[1]) and 1 or 0, count, ttl }
```

```java
List<Long> r = redisTemplate.execute(SCRIPT,
        List.of("rate:{" + ip + "}"),                    // KEYS[1] — 해시 태그로 IP 단위 고정
        String.valueOf(limit), String.valueOf(window.toSeconds()));
if (r.get(0) == 0L) throw new TooManyRequestsException(r.get(2));   // 남은 초 → Retry-After
```

- 지금은 키가 하나라 슬롯 문제가 없지만, 나중에 `rate:{1.2.3.4}:minute`처럼 키를 늘려도 같은 슬롯이라 한 스크립트에서 처리할 수 있다.
- 실무에 적용할 때는 **IP를 제대로 읽는 것**도 같이 챙겨야 했다. 로드밸런서 뒤에서는 `X-Forwarded-For`를 믿는 기준을 정해야 한다. 이건 [X-Forwarded-For와 getRemoteAddr() 차이](/posts/x-forwarded-for-vs-remote-addr/)에 따로 정리했다.

## 정리

> Redis Cluster는 키를 **16,384개 해시 슬롯**으로 나눠 여러 마스터에 담고, 클라이언트는 슬롯 표를 들고 맞는 노드로 바로 요청한다(`MOVED`/`ASK`로 갱신).
> 한 명령은 여러 노드에 걸쳐 원자적으로 실행될 수 없으니 **멀티 키 명령·트랜잭션·Lua 스크립트는 같은 슬롯의 키만** 다룬다. 같이 다룰 키는 **해시 태그**로 묶되 쏠림이 생기지 않을 만큼만 묶는다.
> Lua 스크립트는 키를 반드시 `KEYS`로 넘기고, 스크립트 캐시가 노드별이라 `NOSCRIPT` 재시도가 필요하며, 오래 걸리면 그 노드의 슬롯 전체가 멈춘다는 점을 기억한다.

## 참고

- [Redis — Scale with Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Redis Cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Redis — Scripting with Lua](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/)
- [Lettuce — Redis Cluster](https://redis.github.io/lettuce/ha-sharding/#redis-cluster)
