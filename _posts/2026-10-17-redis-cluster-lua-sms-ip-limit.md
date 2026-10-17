---
title: "Redis 클러스터 + Lua 스크립트로 문자 인증 API IP별 횟수 제한하기"
date: 2026-10-17
categories: [Redis]
---

문자(SMS) 인증번호 발송 API에 **IP별 요청 횟수 제한**을 걸었다.
서버가 여러 대이고 Redis는 **클러스터** 환경이라, 카운터는 Redis에 두고 **Lua 스크립트**로 "증가 + 만료 설정 + 한도 비교"를 한 번에 처리했다.
Spring Boot에 적용한 방법과 클러스터라서 신경 써야 했던 점을 정리해 둔다.

> 실무에서 적용한 내용을 일반화해서 적었다. 키 이름, 제한 횟수, 클래스 이름은 예시다.
> Lua 스크립트 자체가 처음이라면 [Redis Lua 스크립트 쉽게 이해하기](/posts/redis-lua-script-daily-limit/)를 먼저 보면 좋다.
>
> 실습 코드: [github.com/alswns1201/redis-cluster-rate-limit](https://github.com/alswns1201/redis-cluster-rate-limit) — 커밋 하나가 한 단계라 `git log --reverse`로 따라가면 된다.

## 왜 문자 인증 API에 횟수 제한이 필요한가

문자 인증 API는 **로그인 전에 누구나 호출할 수 있는 API**다. 그런데 호출될 때마다

- **문자 발송 비용**이 나간다. 누군가 스크립트로 계속 호출하면 그대로 비용이 된다.
- 다른 사람 번호를 넣고 반복 호출하면 그 사람에게 **문자 폭탄**이 간다.

그래서 **같은 IP에서 일정 시간 안에 보낼 수 있는 횟수**를 제한했다.

```
같은 IP에서 하루 N회까지 인증번호 요청 가능
넘으면 429 Too Many Requests
```

## 왜 Redis인가

서버가 여러 대라 **서버 메모리에 카운터를 두면 서버마다 따로 센다.**
로드밸런서가 요청을 나눠 주면 서버 3대 기준으로 한도의 3배까지 뚫린다.
모든 서버가 같은 카운터를 봐야 하니 **공용 저장소인 Redis**에 둔다.

```
키:  sms:limit:{ip}
값:  이 IP의 요청 횟수
TTL: 제한 기간 (예: 하루)
```

## 그냥 INCR + EXPIRE로 하면 생기는 문제

처음 떠올리는 코드는 이렇다.

```java
Long count = redisTemplate.opsForValue().increment(key);   // ① INCR
if (count == 1) {
	redisTemplate.expire(key, Duration.ofDays(1));          // ② 첫 요청이면 만료 시간 설정
}
if (count > limit) {
	throw new TooManyRequestsException();
}
```

`INCR` 자체는 원자적이라 **횟수가 빠지는 문제는 없다.** 문제는 ①과 ②가 **따로 가는 명령**이라는 점이다.

- ①이 성공하고 ②를 보내기 전에 **서버가 죽거나 네트워크 오류**가 나면, 키가 **만료 시간 없이** 남는다.
- 그 IP는 카운터가 영원히 줄지 않아서 **영구 차단**된다. 실제 사용자가 같은 IP(회사, 카페 공유기)를 쓰고 있었다면 그 사람들 전부 인증을 못 한다.

```
요청 A:  INCR → 1   (서버 다운 💥)   EXPIRE 안 감
→ sms:limit:{1.2.3.4} = 1, TTL 없음 → 하루가 지나도 안 지워짐
```

만료가 꼭 붙어 있어야 하는 카운터라서 **"증가 + 만료 설정"이 반드시 같이 실행**되어야 한다.
Redis에는 이 둘을 한 번에 해 주는 명령이 없어서 Lua 스크립트로 묶었다.

## Lua 스크립트

`src/main/resources/scripts/sms_ip_limit.lua`

```lua
-- KEYS[1] : sms:limit:{ip}
-- ARGV[1] : 제한 횟수,  ARGV[2] : 제한 기간(초)
-- return  : { 허용 여부(1/0), 현재 횟수, 남은 TTL(초) }
local count = redis.call('INCR', KEYS[1])
if count == 1 then
	redis.call('EXPIRE', KEYS[1], tonumber(ARGV[2]))   -- 첫 요청에만 만료 설정
end

local ttl = redis.call('TTL', KEYS[1])
if ttl < 0 then                                         -- 혹시 TTL 없는 키가 남아 있으면 복구
	redis.call('EXPIRE', KEYS[1], tonumber(ARGV[2]))
	ttl = tonumber(ARGV[2])
end

if count > tonumber(ARGV[1]) then
	return { 0, count, ttl }
end
return { 1, count, ttl }
```

- **INCR와 EXPIRE가 한 번에 실행**된다. 스크립트 중간에 다른 요청이 끼어들 수 없고, 서버가 스크립트를 보낸 뒤 죽어도 Redis 안에서는 끝까지 실행된다.
- 만료는 **첫 요청(`count == 1`)에만** 건다. 매 요청마다 EXPIRE를 다시 걸면 계속 요청하는 한 기간이 끝없이 늘어난다.
- `TTL < 0` 확인은 예전 코드로 만들어진, 만료가 없는 키가 혹시 남아 있을 때를 위한 안전장치다.
- 남은 TTL을 같이 돌려줘서 응답에 **"몇 초 뒤에 다시 시도하세요"(`Retry-After`)**를 넣을 수 있다.

> 한도를 넘은 요청도 카운트는 올라간다. 막힌 상태에서 계속 두드려도 결과는 똑같이 "거절"이고,
> 만료 시간은 첫 요청 기준이라 늘어나지 않으니 문제없다.

## Spring Boot에서 호출하기

```java
@Component
@RequiredArgsConstructor
public class SmsIpRateLimiter {

	private static final RedisScript<List> SCRIPT =
			RedisScript.of(new ClassPathResource("scripts/sms_ip_limit.lua"), List.class);

	private final StringRedisTemplate redisTemplate;

	@Value("${sms.limit.per-ip}")
	private int limit;

	@Value("${sms.limit.window}")
	private Duration window;

	public void check(String ip) {
		List<Long> result = redisTemplate.execute(SCRIPT,
				List.of("sms:limit:{" + ip + "}"),         // KEYS[1]
				String.valueOf(limit),                    // ARGV[1]
				String.valueOf(window.toSeconds()));      // ARGV[2]

		if (result.get(0) == 0L) {
			throw new SmsRateLimitException(result.get(2));   // 남은 초 → Retry-After
		}
	}
}
```

```java
@PostMapping("/sms/verification")
public ResponseEntity<Void> sendCode(@RequestBody SmsRequest request, HttpServletRequest http) {
	smsIpRateLimiter.check(clientIp(http));   // 문자 발송 전에 먼저 확인
	smsService.sendVerificationCode(request.phone());
	return ResponseEntity.ok().build();
}
```

```java
@ExceptionHandler(SmsRateLimitException.class)
public ResponseEntity<ProblemDetail> handle(SmsRateLimitException e) {
	ProblemDetail body = ProblemDetail.forStatusAndDetail(HttpStatus.TOO_MANY_REQUESTS,
			"인증번호 요청 횟수를 초과했습니다. 잠시 후 다시 시도해 주세요.");
	return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
			.header(HttpHeaders.RETRY_AFTER, String.valueOf(e.getRetryAfterSeconds()))
			.body(body);
}
```

- 스크립트는 `RedisScript`로 한 번만 만들어 재사용한다. `execute`가 `EVALSHA`로 보내고, Redis에 스크립트가 없으면(`NOSCRIPT`) 알아서 `EVAL`로 다시 보낸다.
- Lua의 숫자 배열은 자바에서 `List<Long>`으로 받는다.
- 횟수 제한은 **문자 발송 전에** 확인한다. 발송 후에 세면 이미 비용이 나간 뒤다.

## 클러스터 환경에서 신경 쓴 점

### 1) 스크립트가 쓰는 키는 같은 슬롯에 있어야 한다

Redis 클러스터는 키를 **16,384개의 해시 슬롯**으로 나누고, 슬롯마다 담당 노드가 다르다.
Lua 스크립트는 **한 노드에서 실행**되기 때문에, 스크립트가 건드리는 키들이 다른 슬롯에 있으면 실행 자체가 거절된다.

```
(error) CROSSSLOT Keys in request don't hash to the same slot
```

이번 스크립트는 키를 **하나만** 쓰니 이 문제가 없다.
그래도 나중에 "IP별 분당 제한 + 일별 제한"처럼 키를 늘릴 수 있어서, 키 이름에 **해시 태그 `{}`**를 넣어 두었다.

```
sms:limit:{1.2.3.4}          ← 중괄호 안 "1.2.3.4"로만 슬롯을 계산
sms:limit:{1.2.3.4}:minute   ← 같은 슬롯
```

클러스터는 키에 `{...}`가 있으면 **중괄호 안의 문자열로만** 슬롯을 정한다.
같은 IP에 대한 키는 항상 같은 슬롯·같은 노드에 모이니, 키를 여러 개 써도 한 스크립트에서 처리할 수 있다.

### 2) 키는 반드시 KEYS로 넘긴다

스크립트 안에서 키 이름을 문자열로 만들어 쓰면(`redis.call('INCR', 'sms:limit:' .. ARGV[1])`) 단일 Redis에서는 돌아가지만,
클라이언트가 **어느 노드로 보낼지 모른다.** 클라이언트(Lettuce)는 `KEYS`로 넘어온 키를 보고 슬롯을 계산해 해당 노드로 보낸다.
그래서 키는 **항상 `KEYS`로 넘기고**, 스크립트 안에서 키 이름을 조합하지 않는다.

### 3) Spring 설정

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
            adaptive: true       # 페일오버로 마스터가 바뀌면 토폴로지 갱신
            period: 30s
```

- 노드 주소를 클러스터 설정으로 주면 `RedisTemplate`이 **클러스터 모드**로 동작한다. 코드는 단일 Redis와 똑같다.
- 토폴로지 갱신을 켜 두지 않으면, 마스터 노드가 바뀌었을 때 예전 노드로 계속 요청을 보내다가 실패할 수 있다.
- 스크립트 캐시는 **노드별**이다. 노드가 바뀌거나 재시작돼서 캐시가 비어도 위에서 말한 `NOSCRIPT → EVAL` 재전송으로 처리된다.

## 클라이언트 IP를 제대로 가져오기

IP로 제한하는데 **IP를 잘못 읽으면** 의미가 없다.

- 서버가 로드밸런서(ALB 등) 뒤에 있으면 `request.getRemoteAddr()`는 **로드밸런서의 IP**다. 모든 사용자가 같은 IP로 잡혀서 전원이 같이 막힌다.
- 실제 IP는 `X-Forwarded-For` 헤더에 들어온다. 다만 이 헤더는 **클라이언트가 마음대로 넣을 수 있어서**, 헤더 맨 앞 값을 그대로 믿으면 매번 다른 IP를 넣어 제한을 피할 수 있다.
- **우리 로드밸런서가 붙인 값**만 믿도록, Spring Boot의 `server.forward-headers-strategy`(또는 톰캣 `RemoteIpValve`)에 신뢰할 프록시를 지정해 처리하는 게 안전하다.

```yaml
server:
  forward-headers-strategy: native   # 신뢰하는 프록시가 붙인 X-Forwarded-For 기준으로 remoteAddr 설정
```

## 주의할 점

- **IP 제한만으로는 부족하다.** 같은 IP를 여러 사람이 쓰기도 하고(회사, 학교), 공격자는 IP를 바꿀 수 있다.
  **전화번호별 제한**을 같이 두면 특정 번호로의 문자 폭탄을 막을 수 있다(키가 다르니 별도 스크립트 호출로).
- **한도와 기간은 설정값으로** 뺀다. 운영하면서 조정할 일이 생긴다.
- 지금 방식은 **고정 윈도우**(첫 요청부터 기간을 셈)다. 기간 경계에서 짧은 시간에 한도의 두 배까지 몰릴 수 있는데,
  문자 인증처럼 "하루 몇 회" 정도의 제한에서는 충분했다. 더 정밀해야 하면 슬라이딩 윈도우(Sorted Set)나 토큰 버킷을 고려한다.
- **Redis가 죽었을 때** 어떻게 할지 정해 둔다. 제한 체크가 실패했다고 인증 자체를 막을지(fail-closed), 통과시킬지(fail-open)는 서비스 성격에 따라 다르다.

## 한 줄 정리

> 문자 인증 API의 IP별 횟수 제한은 여러 서버가 함께 보도록 Redis 카운터로 두고,
> `INCR`과 `EXPIRE`를 Lua 스크립트로 묶어 만료 없는 키(영구 차단)가 생기지 않게 했다.
> 클러스터에서는 스크립트의 키를 `KEYS`로 넘기고, 해시 태그 `{ip}`로 같은 IP의 키를 같은 슬롯에 모은다.
