---
title: "Redis Lua 스크립트 쉽게 이해하기 — 일일 결제 한도 예제"
date: 2026-10-03
categories: [Redis]
---

선불 지갑 실습에서 "하루 결제 한도"를 Redis로 만들다가 Lua 스크립트를 쓰게 됐다.
처음엔 "왜 갑자기 Lua?"였는데, 순서대로 정리해 보면 이유가 간단하다.

## 상황: 하루 결제 한도 체크

지갑마다 "오늘 쓴 금액"을 Redis 키 하나에 쌓아 둔다.

```
키:  wallet:daily:{walletId}:{yyyyMMdd}
값:  오늘 결제한 누적 금액
```

결제가 들어오면 이렇게 확인한다.

```java
public void reserve(Long walletId, long amount, LocalDate date) {
	String key = key(walletId, date);
	long used = used(walletId, date);                         // ① GET    : 오늘 얼마 썼지?
	if (used + amount > dailyPayLimit) {                      // ② 비교   : 자바에서 계산
		throw new WalletException(ErrorCode.DAILY_LIMIT_EXCEEDED);
	}
	redisTemplate.opsForValue().increment(key, amount);       // ③ INCRBY : 금액 더하기
}
```

코드만 보면 아무 문제 없어 보인다.

## 문제: 세 번에 나눠서 보낸다

①과 ③은 Redis에 **따로따로** 보내는 명령이다. 그 사이에 다른 요청이 끼어들 수 있다.

한도 100만 원, 이미 98만 원을 쓴 상태에서 2만 원 결제 두 건이 동시에 들어오면:

```
시간 →
요청 A:  GET → 98만   비교 98+2=100 OK           INCRBY → 100만
요청 B:        GET → 98만   비교 98+2=100 OK            INCRBY → 102만  ❌ 한도 초과
```

B가 읽을 때는 A가 아직 더하기 전이라, 둘 다 "여유 있음"으로 판단한다.

> 실습에서는 이 코드를 지갑 락(Redisson) 안에서만 호출해서 문제가 안 생긴다.
> 하지만 그건 **락 덕분**이지 한도 코드가 안전해서가 아니다.
> 락 밖에서 부르거나, 한도 기준이 "사용자별"(지갑 여러 개)로 바뀌면 바로 뚫린다.

## 해결: ①②③을 한 덩어리로 Redis에 보낸다

Redis에는 "비교하고 나서 더하기"를 한 번에 해 주는 명령이 없다. (`INCRBY`는 무조건 더한다.)
그래서 **세 단계를 작은 프로그램으로 묶어 Redis에 통째로 보내는 방법**을 쓴다. 이게 Lua 스크립트다.

```
지금:  자바 ─GET→ Redis,  자바에서 비교,  자바 ─INCRBY→ Redis    (사이에 끼어들 수 있음)
Lua:   자바 ─[GET + 비교 + INCRBY]→ Redis                        (끼어들 수 없음)
```

Redis는 명령을 **한 번에 하나씩** 처리하고, Lua 스크립트 하나도 "명령 하나"로 취급한다.
스크립트가 도는 동안 다른 요청은 기다려야 하니까, 위의 B는 A가 끝난 뒤에 실행된다.

```
요청 A:  [GET → 비교 → INCRBY] → 100만
요청 B:                          [GET → 100만, 비교 102 > 100 → 거절] ✅
```

## Lua 스크립트 기초 (redis-cli로 직접 해 보기)

### EVAL — 스크립트 실행

```
EVAL "<스크립트>"  <키 개수>  <키들...>  <값들...>
```

```
> EVAL "return 'hello'" 0
"hello"
```

마지막 `0`은 "넘기는 키가 0개"라는 뜻이다.

### KEYS와 ARGV — 밖에서 값 넘기기

```
> EVAL "return {KEYS[1], ARGV[1], ARGV[2]}" 1 mykey 20000 1000000
1) "mykey"     ← KEYS[1]
2) "20000"     ← ARGV[1]
3) "1000000"   ← ARGV[2]
```

- 키 개수를 `1`로 줬으니 첫 값은 `KEYS`, 나머지는 `ARGV`로 들어간다.
- **키 이름은 KEYS로, 금액 같은 값은 ARGV로** 넘긴다. (Redis가 어떤 키를 건드리는지 미리 알아야 해서)
- Lua 배열은 **1부터** 시작한다. `KEYS[0]`이 아니다.
- ARGV는 문자열로 들어오므로 계산하려면 `tonumber()`가 필요하다.

### redis.call — 스크립트 안에서 Redis 명령 쓰기

```
> EVAL "redis.call('SET', KEYS[1], ARGV[1]); return redis.call('GET', KEYS[1])" 1 demo:used 980000
"980000"
```

`redis.call('SET', ...)`은 redis-cli에서 `SET ...`을 치는 것과 같다.

## 한도 체크 Lua 스크립트

`src/main/resources/scripts/daily_limit_reserve.lua`

```lua
-- KEYS[1] : wallet:daily:{walletId}:{yyyyMMdd}
-- ARGV[1] : 결제 금액,  ARGV[2] : 일일 한도,  ARGV[3] : TTL(초)
-- return  : 더한 뒤 누적 금액, 한도 초과면 -1
local used   = tonumber(redis.call('GET', KEYS[1]) or '0')   -- ① GET (없으면 0)
local amount = tonumber(ARGV[1])
local limit  = tonumber(ARGV[2])

if used + amount > limit then                                -- ② 비교
	return -1
end

local total = redis.call('INCRBY', KEYS[1], amount)          -- ③ INCRBY
redis.call('EXPIRE', KEYS[1], tonumber(ARGV[3]))
return total
```

자바 코드와 **하는 일은 똑같고, 실행되는 장소만 Redis 안으로 옮겨졌다.**

- `local` : 지역 변수 (자바의 `var` 느낌)
- `or '0'` : 키가 없으면 GET 결과가 nil이라 기본값 `'0'` 사용 (`value == null ? 0 : ...`)

98만 원 쓴 상태에서 실행해 보면:

```
> 2만 원 결제    → 1000000   (통과)
> 또 2만 원 결제 → -1        (거절)
```

## Spring에서 호출하기

```java
private static final RedisScript<Long> RESERVE =
		RedisScript.of(new ClassPathResource("scripts/daily_limit_reserve.lua"), Long.class);

public void reserve(Long walletId, long amount, LocalDate date) {
	Long result = redisTemplate.execute(RESERVE,
			List.of(key(walletId, date)),             // KEYS
			String.valueOf(amount),                   // ARGV[1]
			String.valueOf(dailyPayLimit),            // ARGV[2]
			String.valueOf(KEY_TTL.toSeconds()));     // ARGV[3]
	if (result == null || result == -1) {
		throw new WalletException(ErrorCode.DAILY_LIMIT_EXCEEDED);
	}
}
```

- 호출하는 쪽 코드는 그대로고 `reserve()` 안쪽만 바뀐다.
- 스크립트를 매번 통째로 보내면 낭비라서 Redis에는 `EVALSHA`(해시값으로 실행)가 있는데,
  Spring의 `execute`가 이걸 **알아서** 처리해 준다. 신경 쓸 필요 없다.

## 헷갈렸던 점: 여기서 "원자성"은 롤백이 아니다

"원자성"이라고 하면 보통 "하나가 실패하면 전부 취소"(DB 트랜잭션)를 떠올린다. 그런데 여기서 말하는 원자성은 뜻이 다르다.

| 문맥 | 원자성의 뜻 |
|---|---|
| DB 트랜잭션 (`@Transactional`) | 전부 성공하거나 전부 취소 (롤백) |
| 동시성 (Lua 스크립트) | **실행 도중에 다른 요청이 끼어들 수 없다** |

실제로 **Lua는 중간에 에러가 나도 롤백하지 않는다.**

```lua
redis.call('INCRBY', KEYS[1], 20000)   -- 성공
redis.call('INCRBY', KEYS[1], 'abc')   -- 에러!
```

```
> EVAL ...       → ERR value is not an integer ...
> GET demo:used  → "20000"    ← 앞에서 더한 값은 그대로 남아 있다
```

그래서 스크립트는 **확인(비교)을 먼저 하고, 값을 바꾸는 명령은 마지막에** 두는 게 안전하다.
위 한도 스크립트도 한도를 넘으면 아무것도 쓰지 않고 `-1`만 돌려주고 끝난다.

## 그럼 지갑 락은 이제 빼도 될까? → 아니다

Lua가 지켜 주는 건 **Redis에 있는 한도 카운터뿐**이다.

| | 지키는 것 | 위치 |
|---|---|---|
| 지갑 락 | 잔액, 거래 내역 | DB |
| Lua 스크립트 | 오늘 쓴 금액 카운터 | Redis |

Lua는 Redis 안에서만 돌기 때문에 DB 잔액은 건드릴 수 없다. 락을 빼면 잔액 쪽에서 동시 업데이트 문제가 다시 생긴다.
그래서 **락은 그대로 두고, 한도 코드만 Lua로 바꿔서 락이 없어도 안전하게 만드는 것**이 목표다.

## 주의할 점

- **스크립트는 짧게.** 스크립트가 도는 동안 Redis 전체가 다른 요청을 못 받는다.
- **Redis 클러스터**에서는 스크립트가 건드리는 키들이 같은 노드(슬롯)에 있어야 한다. 키 하나만 쓰면 상관없다.
- 값을 바꾸는 명령은 **검사를 다 끝낸 뒤 마지막에.** (실패해도 롤백이 안 되니까)

## 한 줄 정리

> Lua 스크립트는 "읽고 → 비교하고 → 바꾸기"처럼 여러 Redis 명령을 **아무도 끼어들지 못하게 한 번에** 실행하는 방법이다.
> 실패 시 롤백(DB 트랜잭션)과는 다른 이야기다.
