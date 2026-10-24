---
title: "X-Forwarded-For와 getRemoteAddr() 차이, ALB append 확인과 Spring native 설정"
date: 2026-10-24 10:00:00 +0900
categories: [인프라]
---

IP별 요청 제한을 걸면서 **"클라이언트 IP를 어디서 읽어야 하나"**를 정리할 일이 생겼다.
`request.getRemoteAddr()`와 `X-Forwarded-For` 헤더가 각각 무엇인지,
ALB 뒤에서는 왜 ALB 설정이 **append**인지 확인해야 하는지,
그리고 Spring Boot에서 `server.forward-headers-strategy: native`로 처리하는 방법까지 적어 둔다.

> 실무에서 적용한 내용을 일반화해서 적었다. IP 주소와 CIDR은 예시 값이다.
> 이 내용을 실제로 쓴 곳은 [Redis 클러스터 + Lua 스크립트로 문자 인증 API IP별 횟수 제한하기](/posts/redis-cluster-lua-sms-ip-limit/)다.

## 한눈에 보는 차이

| | `getRemoteAddr()` | `X-Forwarded-For` |
|---|---|---|
| 어디서 오나 | **TCP 연결의 상대방 IP** (서버가 직접 받은 연결) | **HTTP 헤더** (중간 프록시들이 적어 준 값) |
| 위조 가능? | 사실상 불가능 (TCP 연결이 맺어져야 하므로) | **가능** (클라이언트가 헤더를 직접 넣을 수 있음) |
| ALB 뒤에서 값 | **ALB의 사설 IP** (예: `10.0.1.23`) | `원래 클라이언트, ..., ALB가 본 IP` |

`getRemoteAddr()`는 **"지금 나한테 TCP로 붙어 있는 상대"**다.
클라이언트가 서버에 직접 붙으면 클라이언트 IP지만, 중간에 ALB가 있으면 서버에 붙은 건 ALB이므로 ALB IP가 나온다.

ALB는 HTTP를 끝까지 해석하는 L7 로드밸런서라 **클라이언트와 TCP 연결을 끊고, 서버와는 새 연결을 맺는다.**
그래서 원래 클라이언트 IP는 TCP 정보로는 전달되지 않고, 대신 **`X-Forwarded-For` 헤더**에 적어서 넘겨준다.

```
[클라이언트 203.0.113.7] ──TCP──> [ALB 10.0.1.23] ──TCP──> [서버]

서버에서 보면
  getRemoteAddr()             = 10.0.1.23       ← ALB
  X-Forwarded-For 헤더         = 203.0.113.7     ← ALB가 적어 준 클라이언트 IP
```

ALB 뒤에서 `getRemoteAddr()`를 그대로 쓰면 **모든 사용자가 ALB IP 몇 개로 보인다.**
IP별 제한을 걸었다면 사용자 전체가 같은 카운터를 쓰게 되어 한꺼번에 막힌다.

## X-Forwarded-For는 어떻게 쌓이나

`X-Forwarded-For`는 프록시를 지날 때마다 **"내가 받은 연결의 상대 IP"를 오른쪽 끝에 붙이는** 헤더다.

```
X-Forwarded-For: <클라이언트>, <프록시1>, <프록시2>
                  가장 왼쪽 = 처음        가장 오른쪽 = 가장 최근에 붙인 값
```

여기서 중요한 점은 **클라이언트도 이 헤더를 직접 넣어서 보낼 수 있다**는 것이다.

```bash
curl -H "X-Forwarded-For: 1.1.1.1" https://api.example.com/sms/verification
```

ALB가 append 모드라면 서버가 받는 헤더는 이렇게 된다.

```
X-Forwarded-For: 1.1.1.1, 203.0.113.7
                 ↑ 클라이언트가 지어낸 값  ↑ ALB가 실제 TCP 연결에서 본 IP
```

- **맨 왼쪽 값**을 클라이언트 IP로 쓰면 → `1.1.1.1`. 요청마다 다른 값을 넣으면 IP 제한을 얼마든지 피할 수 있다.
- **맨 오른쪽 값**(우리 ALB가 붙인 값)은 ALB가 TCP 연결에서 직접 본 IP라 위조할 수 없다.

즉 **믿을 수 있는 건 "우리 쪽 프록시가 붙인 값"뿐**이고, 오른쪽부터 거꾸로 읽으면서
우리 프록시가 아닌 첫 번째 IP를 클라이언트 IP로 봐야 한다.
이걸 직접 구현하지 않고 톰캣의 `RemoteIpValve`에 맡기는 게 뒤에서 볼 `native` 설정이다.

## ALB에서 append인지 확인해야 하는 이유

ALB에는 `X-Forwarded-For` 헤더를 어떻게 다룰지 정하는 속성이 있다.
속성 키는 `routing.http.xff_header_processing.mode`이고 값은 세 가지다.

| 모드 | 동작 | 서버에서 보이는 결과 |
|---|---|---|
| **append** (기본값) | 들어온 헤더 끝에 **ALB가 본 클라이언트 IP를 붙인다** | `1.1.1.1, 203.0.113.7` → 오른쪽 값을 믿으면 됨 |
| preserve | 들어온 헤더를 **그대로 넘기고 아무것도 안 붙인다** | `1.1.1.1` → 클라이언트가 지어낸 값만 남음 |
| remove | 헤더를 **지운다** | 헤더 없음 → 서버는 ALB IP밖에 모름 |

- **preserve**면 가장 위험하다. 서버가 오른쪽부터 읽어도 맨 오른쪽 값이 이미 **클라이언트가 넣은 값**이라 그대로 속는다.
  헤더 없이 온 요청은 ALB IP로 잡혀서 IP 제한이 전원에게 같이 걸리는 문제도 생긴다.
- **remove**면 위조는 막히지만 실제 클라이언트 IP를 알 방법이 없다.
- 그래서 서버에서 "오른쪽부터 신뢰" 방식이 성립하려면 **ALB가 반드시 append**여야 한다.

기본값이 append라서 보통은 문제없지만, 누군가 콘솔이나 IaC에서 바꿔 놨을 수 있으니 **직접 확인**한다.

### 콘솔에서 확인

EC2 → 로드 밸런서 → 대상 ALB 선택 → **속성(Attributes)** 탭 → 트래픽 구성의
**X-Forwarded-For 헤더** 항목이 `추가(Append)`인지 본다.

### CLI로 확인

```bash
aws elbv2 describe-load-balancer-attributes \
  --load-balancer-arn <ALB ARN> \
  --query "Attributes[?starts_with(Key, 'routing.http.xff')]"
```

```json
[
  { "Key": "routing.http.xff_client_port.enabled", "Value": "false" },
  { "Key": "routing.http.xff_header_processing.mode", "Value": "append" }
]
```

- `xff_header_processing.mode`가 `append`인지 확인한다.
- `xff_client_port.enabled`가 `true`면 IP 뒤에 포트까지 붙는다(`203.0.113.7:51234`).
  서버 쪽에서 IP만 기대하고 파싱하는 코드나 설정이 있다면 깨질 수 있으니 **특별한 이유가 없으면 `false`**로 둔다.

바꿔야 한다면 이렇게 한다. (Terraform이라면 `aws_lb`의 `xff_header_processing_mode = "append"`)

```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn <ALB ARN> \
  --attributes Key=routing.http.xff_header_processing.mode,Value=append
```

## Spring Boot에서 native 설정

ALB가 append로 붙여 준 값을 서버에서 올바르게 읽는 방법이 `server.forward-headers-strategy`다.

| 값 | 누가 처리하나 |
|---|---|
| `none` | 처리 안 함. `getRemoteAddr()`는 그대로 ALB IP |
| **`native`** | **서블릿 컨테이너**(톰캣이면 `RemoteIpValve`)가 처리 |
| `framework` | Spring의 `ForwardedHeaderFilter`가 처리 |

> 설정하지 않으면 기본은 처리하지 않는다. (Spring Boot가 Kubernetes 같은 일부 클라우드 플랫폼을 감지했을 때는 `native`처럼 동작한다.)

`native`를 고른 이유는 **톰캣의 `RemoteIpValve`가 "신뢰하는 프록시"를 기준으로 오른쪽부터 읽기 때문**이다.
반면 `framework`의 `ForwardedHeaderFilter`는 `X-Forwarded-For`의 **맨 왼쪽 값**을 클라이언트 주소로 쓰기 때문에,
ALB append 환경에서는 클라이언트가 지어낸 값을 그대로 쓰게 된다. (그래서 `framework`는 앞단 프록시가 헤더를 정리해 주는 환경에서 쓰는 게 맞다.)

### 설정

```yaml
server:
  forward-headers-strategy: native
  tomcat:
    remoteip:
      # ALB가 있는 VPC 대역 (예: 10.0.0.0/16). 정규식이다.
      internal-proxies: '10\.0\.\d{1,3}\.\d{1,3}'
      remote-ip-header: x-forwarded-for     # 기본값
      protocol-header: x-forwarded-proto    # 기본값
```

- `native`로 두면 Spring Boot가 톰캣에 `RemoteIpValve`를 등록한다.
- `internal-proxies`는 **"이 IP에서 온 연결이면 X-Forwarded-For를 믿겠다"**는 프록시 목록이다(정규식).
  톰캣 기본값에도 `10.x`, `172.16~31.x`, `192.168.x` 같은 사설 대역이 들어 있어서 따로 안 적어도 ALB가 대부분 잡히지만,
  **우리 VPC 대역으로 좁혀서 명시**해 두는 편이 의도가 분명하다.
- YAML에서 정규식은 **작은따옴표**로 감싼다. 큰따옴표면 `\`를 두 번 써야 한다.
- `x-forwarded-proto`도 같이 처리되어서 ALB에서 HTTPS를 끝내도 `request.isSecure()`, `getScheme()`이 `https`로 나온다.
  리다이렉트 URL이 `http://`로 만들어지는 문제도 이걸로 해결된다.

### RemoteIpValve가 하는 일

```
요청: getRemoteAddr() = 10.0.1.23
      X-Forwarded-For = 1.1.1.1, 203.0.113.7

1) 연결 상대 10.0.1.23 이 internal-proxies에 해당하나?  → 예 (아니면 아무것도 안 하고 끝)
2) X-Forwarded-For를 오른쪽부터 읽는다
   - 203.0.113.7 : 내부 프록시인가? → 아니오 → 이게 클라이언트 IP
3) getRemoteAddr()를 203.0.113.7 로 바꾼다
   (왼쪽에 남은 1.1.1.1 은 X-Forwarded-For 헤더에 그대로 남는다)
```

- **연결 상대가 신뢰하는 프록시일 때만** 헤더를 본다. 누가 서버에 직접 붙어서 헤더를 보내도 무시된다.
- 오른쪽부터 읽다가 **신뢰하지 않는 첫 IP**에서 멈추니, 왼쪽에 아무 값을 넣어도 영향이 없다.

이렇게 바뀐 값은 **필터, 인터셉터, 컨트롤러 어디서든 `request.getRemoteAddr()`로** 읽으면 된다.
`RemoteIpValve`는 톰캣 단계에서 동작해서 Spring의 필터보다 먼저 실행되기 때문이다.

```java
// 헤더를 직접 파싱하지 않는다. native 설정이 이미 바꿔 놨다.
String clientIp = request.getRemoteAddr();
```

> 코드에서 `request.getHeader("X-Forwarded-For").split(",")[0]`처럼 **맨 앞 값을 꺼내는 유틸**이 있다면
> 위에서 말한 위조에 그대로 뚫린다. native 설정을 했다면 이런 유틸은 지우고 `getRemoteAddr()`만 쓴다.

## ALB 앞에 프록시가 더 있다면

CloudFront처럼 ALB 앞에 프록시가 하나 더 있으면 헤더는 이렇게 된다.

```
X-Forwarded-For: 203.0.113.7, 198.51.100.10
                 ↑ 클라이언트   ↑ CloudFront 엣지 IP (ALB가 append)
```

CloudFront IP는 사설 대역이 아니라 `internal-proxies`에 걸리지 않는다.
이 상태로는 **CloudFront 엣지 IP가 클라이언트 IP로** 잡히니, CloudFront 대역을 `server.tomcat.remoteip.trusted-proxies`에 추가해야 한다.
**앞단 구성이 바뀌면 신뢰 프록시 목록도 같이 바꿔야 한다**는 점을 기억해 둔다.

## 적용 후 확인

1. 로그에 `getRemoteAddr()`를 남기고, **헤더를 위조해서** 요청해 본다.

   ```bash
   curl -H "X-Forwarded-For: 1.1.1.1" https://api.example.com/health
   ```

   로그에 `1.1.1.1`이 아니라 **내 실제 공인 IP**가 찍히면 정상이다.

2. 동작이 이상하면 밸브 로그를 켜서 어떤 값으로 판단했는지 본다.

   ```yaml
   logging:
     level:
       org.apache.catalina.valves.RemoteIpValve: debug
   ```

3. 로컬(ALB 없음)에서는 연결 상대가 `127.0.0.1` 또는 `0:0:0:0:0:0:0:1`(IPv6 루프백)이다.
   로컬에서 IPv6 주소가 찍히는 이유는 [IPv4와 IPv6 차이](/posts/ipv4-vs-ipv6/)에 정리했다.

## 한 줄 정리

> `getRemoteAddr()`는 TCP 연결 상대라 ALB 뒤에서는 ALB IP가 나오고, `X-Forwarded-For`는 클라이언트가 위조할 수 있는 헤더다.
> ALB를 **append**로 두면 맨 오른쪽 값은 ALB가 붙인 믿을 수 있는 값이 되고,
> Spring Boot는 **`forward-headers-strategy: native`**로 톰캣 `RemoteIpValve`가 신뢰 프록시 기준으로 오른쪽부터 읽어
> `getRemoteAddr()`를 실제 클라이언트 IP로 바꿔 준다.
