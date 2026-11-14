---
title: "서버가 여러 대일 때 로그인 세션 다루기 — JWT에 Redis 세션을 붙이는 이유와 Redis Cluster에서 조심할 점"
date: 2026-11-14 11:00:00 +0900
categories: [Redis]
---
{% raw %}

API 서버를 여러 대 띄우면 "로그인 상태를 어디에 두느냐"를 정해야 한다. 서버 메모리의 `HttpSession`은 서버마다 따로라서 쓸 수 없고, 흔히 고르는 답은 둘이다.

1. **JWT만 쓴다.** 서버는 아무것도 저장하지 않고 서명만 검증한다.
2. **JWT(또는 세션 ID) + Redis 세션 저장소.** 토큰은 "어느 세션인지"만 담고, 세션이 살아 있는지는 Redis에 물어본다.

이 글은 2번을 고르는 이유와, Redis가 **클러스터**일 때 세션 설계에서 실제로 문제가 되는 지점을 정리한다.

> 글에 나오는 키 이름·클레임 이름·만료 시간은 모두 예시다. 코드는 구조를 보여 주기 위한 예시 코드다.

## JWT만으로는 "끊을" 수가 없다

JWT는 서명이 맞고 만료 전이면 유효하다. 서버가 저장하는 게 없으니 **발급한 토큰을 나중에 무효로 만들 방법이 없다.**
그래서 다음 요구가 하나라도 있으면 JWT만으로는 부족하다.

| 요구 | JWT만 쓸 때 |
|---|---|
| 로그아웃하면 그 토큰은 바로 못 쓰게 | 만료될 때까지 계속 쓸 수 있다 |
| 비밀번호를 바꾸면 다른 기기 로그인도 끊기게 | 끊을 수 없다 |
| 권한이 바뀌면 바로 반영 | 토큰 안의 권한이 만료까지 유지된다 |
| 계정당 한 곳에서만 로그인 (새 로그인이 이전 세션을 끊음) | 이전 토큰을 끊을 수 없다 |
| 관리자가 특정 사용자를 강제 로그아웃 | 불가 |

토큰 수명을 아주 짧게(몇 분) 잡아 피해를 줄이는 방법도 있지만, 결국 "지금 당장 끊기"는 안 된다. 결제·관리자 화면처럼 **토큰 유출이 바로 사고로 이어지는 곳**에서는 서버가 세션을 쥐고 있어야 한다.

## 구조: 토큰은 세션을 가리키고, 살아 있는지는 Redis가 정한다

```
로그인 성공
  → 세션 ID(sid) 생성, Redis에 저장 (TTL)
  → JWT 발급: { sub: userId, sid, typ: "access", exp }

매 요청
  → JWT 서명·만료 검증 (Redis 없이 가능)
  → typ 검사 (access 자리에 refresh 토큰이 오면 거부)
  → Redis에 sid가 있는지 확인 → 없으면 401
```

- 서명 검증은 그대로 하니 위조 토큰은 Redis까지 가지도 않는다.
- **세션을 끊으려면 Redis에서 지우면 된다.** 로그아웃, 비밀번호 변경, 강제 로그아웃이 모두 "키 삭제"로 끝난다.
- 대가는 **요청마다 Redis 조회 1번**이다. 사실상 상태를 가진(stateful) 구조로 돌아온 것이고, Redis가 인증 경로의 필수 의존성이 된다. 이 트레이드오프를 받아들일지가 1번과 2번을 가르는 기준이다.

### 흔한 실수: 토큰 종류를 검사하지 않는다

access 토큰과 refresh 토큰을 같은 키로 서명하고 같은 필터로 검증하면, **수명이 긴 refresh 토큰으로 일반 API를 호출할 수 있다.** 클레임에 `typ`을 넣고 필터에서 반드시 확인한다. refresh 토큰은 갱신 엔드포인트에서만 받는다.

```java
Claims c = jwtParser.parseSignedClaims(token).getPayload();
if (!"access".equals(c.get("typ", String.class))) {
    throw new BadCredentialsException("invalid token type");
}
```

또 하나: **로그아웃 API가 인증 없이 열린 경로(permitAll)에 들어가 있으면** 필터가 토큰을 읽지 않아 "누구의 세션을 지울지" 모른다. 로그아웃은 인증이 필요한 경로에 두어야 실제로 세션을 지울 수 있다.

## 키 설계: 클러스터에서는 "같이 바꿀 키를 같은 슬롯에"

Redis Cluster는 키를 16,384개 슬롯에 나눠 여러 노드에 둔다. **여러 키를 한 번에 다루는 명령(MULTI, Lua 스크립트, MGET 등)은 모든 키가 같은 슬롯에 있어야 한다.** 다르면 `CROSSSLOT` 에러가 난다. 자세한 건 [Redis Cluster 이해하기](/posts/redis-cluster-hash-slot-lua/)에 정리했다.

세션에는 "같이 바꿔야 하는 키"가 꼭 생긴다.

- **계정당 채널별 세션 1개**: 새로 로그인하면 이전 세션을 지우고 새 세션을 등록한다.
- **비밀번호 변경 시 전체 폐기**: 그 사용자의 모든 세션을 찾아서 지운다.

그래서 키에 **해시 태그로 사용자 ID**를 넣어 한 사용자의 키를 한 슬롯에 모은다. `{}` 안의 문자열만 슬롯 계산에 쓰인다.

```
sess:{u:42}:9f1c...        → HASH  { channel: "admin", createdAt: ... }   TTL 30m
sess-index:{u:42}          → SET   { "9f1c...", "a7e2..." }               사용자의 세션 목록
sess-current:{u:42}:admin  → STRING "9f1c..."                            채널별 현재 세션
```

토큰에는 `sub`(사용자 ID)과 `sid`를 함께 넣어 두면 요청마다 키를 바로 만들 수 있다.

### 세션 교체를 원자적으로

"이전 세션 조회 → 삭제 → 새 세션 등록"을 명령 여러 개로 나눠 보내면, 같은 계정으로 동시에 두 번 로그인할 때 세션이 둘 다 남을 수 있다. 한 슬롯에 모았으니 Lua 하나로 묶는다.

```lua
-- KEYS[1] = sess-current:{u:42}:admin
-- KEYS[2] = sess-index:{u:42}
-- KEYS[3] = sess:{u:42}:<newSid>
-- ARGV[1] = newSid, ARGV[2] = ttlSeconds, ARGV[3] = channel, ARGV[4] = key prefix "sess:{u:42}:"
local old = redis.call('GET', KEYS[1])
if old then
  redis.call('DEL', ARGV[4] .. old)
  redis.call('SREM', KEYS[2], old)
end
redis.call('HSET', KEYS[3], 'channel', ARGV[3])
redis.call('EXPIRE', KEYS[3], ARGV[2])
redis.call('SET', KEYS[1], ARGV[1], 'EX', ARGV[2])
redis.call('SADD', KEYS[2], ARGV[1])
return old
```

> 스크립트 안에서 `ARGV`로 키 이름을 조립하는 건 클러스터에서 원칙적으로 권장되지 않는다(키는 `KEYS`로 넘기는 게 규칙). 여기서는 같은 해시 태그를 쓰므로 같은 슬롯이 보장되지만, 엄격하게 하려면 지울 이전 세션 키를 먼저 조회한 뒤 `KEYS`로 넘기는 두 단계로 나누고, 그 사이 경합은 `WATCH`나 버전 비교로 막는다.

### 전체 폐기는 SCAN이 아니라 인덱스로

"이 사용자의 세션 전부"를 `SCAN sess:{u:42}:*`로 찾으면 안 된다. 클러스터에서 `SCAN`은 **노드 하나만** 훑고, 전체 노드를 돌면 느리다. 사용자별 세션 목록(`sess-index`)을 따로 두고, 그 목록으로 지운다. 이 목록에도 TTL을 걸거나, 세션 키가 만료됐는데 목록에 남은 sid는 읽을 때 정리한다.

## 만료: 절대 만료와 활동 기반 연장

| 방식 | 동작 | 문제 |
|---|---|---|
| 절대 만료 | 로그인 시점부터 N분 | 작업 중에도 끊겨 입력하던 내용을 잃는다 |
| 활동 기반 연장(sliding) | 요청이 올 때마다 TTL을 다시 N분으로 | 요청마다 쓰기 1번, 영원히 안 끊길 수 있다 |

관리 화면처럼 오래 작업하는 곳은 "쓰는 동안은 유지, 손 놓으면 만료"가 필요하다. 연장할 때 조심할 점:

- **매 요청마다 `EXPIRE`를 보내지 않는다.** 남은 TTL이 절반 이하일 때만 연장하면 쓰기가 크게 준다.
- **최대 수명을 둔다.** 세션 생성 시각을 저장해 두고, 연장하더라도 "로그인 후 최대 N시간"은 넘기지 않는다. 그래야 탈취된 토큰이 계속 연장되며 살아남지 않는다.
- 토큰 자체의 `exp`가 짧다면, 세션 TTL만 늘려서는 소용없다. **refresh 토큰으로 access 토큰을 다시 발급**하고, 갱신할 때 세션도 함께 연장하는 식으로 맞춘다. refresh 토큰도 갱신 때마다 새로 발급(rotation)하고 이전 것은 무효로 만든다.

## 짧게 쓰는 일회성 세션: 리소스에 묶는다

로그인 세션 말고도 "이 거래 하나를 처리하는 동안만 쓰는 토큰" 같은 짧은 세션이 있다. 팝업이나 외부 화면에 주는 토큰이 그렇다.

```
popup-token:{tx:T123}:<token> → { ownerId: 77 }   TTL 10m
```

- **토큰이 어떤 리소스용인지 저장하고, 요청 경로의 리소스 ID와 반드시 비교한다.** 비교를 "경로에 ID가 있을 때만" 하면, ID가 없는 경로(예: 새 리소스 생성)에서는 비교를 건너뛰어 그 토큰으로 엉뚱한 작업이 통과한다. 토큰을 쓸 수 있는 경로를 정해 두고, 그 외 경로는 거부한다.
- 토큰을 브라우저 쿠키에 담는다면 쿠키 값을 서명하거나, 쿠키에는 랜덤 토큰만 두고 내용은 Redis에서 읽는다. 쿠키에 Base64 JSON을 그대로 두면 사용자가 내용을 바꿀 수 있다.

## 클러스터 운영에서 놓치기 쉬운 것

### 1) 페일오버 때 최근 쓰기가 사라질 수 있다

Redis 복제는 **비동기**다. 마스터가 쓰기를 받고 응답한 뒤 복제본에 전달하기 전에 죽으면, 승격된 복제본에는 그 쓰기가 없다. 세션에서는 이렇게 나타난다.

- 방금 로그인한 세션이 사라짐 → 사용자가 다시 로그인하면 되니 불편 정도
- **방금 한 로그아웃(삭제)이 사라짐 → 끊었던 세션이 되살아남.** 보안상 더 나쁜 쪽이다

`WAIT` 명령으로 복제본 반영을 기다릴 수 있지만 지연이 늘고, 그래도 완전한 보장은 아니다. 로그아웃·비밀번호 변경처럼 **되살아나면 안 되는 삭제**는 토큰 수명을 짧게 유지해 피해 시간을 줄이고, 필요하면 "이 시각 이전에 발급된 토큰은 무효"라는 기준 시각을 DB에 함께 남기는 방법도 있다.

### 2) 인증 조회는 복제본에서 읽지 않는다

읽기 부하를 줄이려고 복제본 읽기(`ReadFrom.REPLICA_PREFERRED`)를 켜면, 방금 지운 세션이 복제본에는 아직 남아 있을 수 있다. **세션 확인은 마스터에서 읽는다.** 복제본 읽기는 캐시처럼 조금 늦어도 되는 데이터에만 쓴다.

### 3) 메모리가 차면 세션이 지워질 수 있다

`maxmemory-policy`가 `allkeys-lru`나 `volatile-lru`면, 메모리가 찰 때 Redis가 **세션 키를 알아서 지운다.** 사용자는 이유 없이 로그아웃된다. 세션을 캐시와 같은 Redis에 둔다면 정책과 메모리 여유를 같이 봐야 하고, 가능하면 세션 전용 Redis를 따로 두거나 `noeviction`으로 운영한다.

### 4) Redis가 죽으면 로그인도 죽는다

요청마다 세션을 확인하니 Redis 장애가 곧 인증 장애다. 이때 "세션 확인 실패면 일단 통과(fail-open)"로 짜면 끊었던 토큰이 다 살아난다. 관리자·결제 경로는 **거부(fail-closed)** 가 맞고, 대신 Redis 상태를 알림으로 빨리 알 수 있게 해 둔다.

## 정리

> 서버가 여러 대일 때 JWT만 쓰면 확장은 쉽지만 **토큰을 끊을 수 없다.** 로그아웃·비밀번호 변경·단일 세션이 필요하면 토큰에 세션 ID를 넣고, 세션의 생사는 Redis가 정하게 한다.
> Redis Cluster에서는 **같이 바꿀 키를 해시 태그로 한 슬롯에 모으고**, 세션 교체는 Lua로 원자적으로, 전체 폐기는 SCAN 대신 인덱스로 처리한다.
> 활동 기반 연장은 최대 수명과 함께 쓰고, 토큰 종류(`typ`)와 리소스 묶음은 반드시 검사한다. 운영에서는 비동기 복제로 인한 삭제 유실, 복제본 읽기, 메모리 축출, Redis 장애 시 fail-closed를 같이 챙긴다.

## 참고

- [Redis — Scale with Redis Cluster (hash tags, multi-key operations)](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Redis — Replication (asynchronous replication, WAIT)](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
- [Redis — Key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [RFC 8725 — JSON Web Token Best Current Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [OWASP — Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
{% endraw %}
