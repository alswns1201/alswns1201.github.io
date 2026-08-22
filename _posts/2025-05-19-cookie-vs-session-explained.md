---
title: "쿠키 vs 세션: HTTP가 상태를 기억 못 해서 생긴 개념들"
date: 2025-05-19
categories: [Java/Spring]
---

HTTP는 **비연결지향(connectionless)**이고 **상태 정보를 유지하지 않는다(stateless)**.
요청이 끝나면 연결이 끊기고, 서버는 방금 대화한 클라이언트를 기억하지 못한다. 로그인
상태를 유지하거나 장바구니를 유지하려면 이 한계를 어딘가에서 메워야 하는데, 그 자리를
채우는 게 쿠키와 세션이다. 둘 다 "상태를 기억한다"는 목적은 같지만, **어디에** 기억하느냐가
완전히 다르고 이 차이가 곧 트레이드오프다.

## 쿠키: 클라이언트가 기억한다

쿠키는 서버가 응답에 실어 보낸, 클라이언트(브라우저)에 저장되는 작은 텍스트 파일이다.
`Key=Value; Domain=...; Path=...; Expires=...` 형태로 저장되고, 이후 요청마다 브라우저가
자동으로 실어 보낸다.

```java
private void setInitSessionCookie(HttpServletResponse response, String value, int maxAge) {
    Cookie ck = new Cookie(session.getId(), value);
    ck.setMaxAge(maxAge);
    ck.setPath("/");
    response.addCookie(ck);
}
```

핵심 성질은 **서버 자원을 쓰지 않는다**는 것이다. 상태를 클라이언트가 들고 다니기
때문에 서버는 그 값을 그대로 신뢰하거나(위조 가능성을 열어두거나), 서명/암호화해서
위조를 막아야 한다. 이 위조 가능성 때문에 쿠키에 민감한 정보(비밀번호, 권한 정보 등)를
평문으로 담는 건 위험하다 — 담더라도 서명된 토큰(JWT 등) 형태여야 한다.

## 세션: 서버가 기억한다

세션은 상태 자체를 서버에 저장하고, 클라이언트에는 그 상태를 가리키는 **식별자(세션 ID)**만
쿠키(`JSESSIONID`)로 내려준다.

```java
@PostMapping("/login")
public String login(MemberLoginDto dto, HttpSession session, Model model) {
    Member loginMember = loginService.login(dto.getUserId(), dto.getPassword());
    session.setAttribute("loginMember", loginMember);
    model.addAttribute("member", loginMember);
    return "/member/memberInfo";
}
```

동작 흐름: 로그인 시 서버가 세션을 생성하고 추정 불가능한 랜덤 세션 ID를 발급 →
브라우저가 이후 요청마다 이 ID를 쿠키로 실어 보냄 → 서버는 ID를 키로 삼아 서버 측
저장소에서 실제 상태를 찾아 요청을 처리한다.

세션 ID 자체는 정보를 담고 있지 않다는 점이 중요하다. 탈취되더라도 그 자체로는 아무
의미가 없고, 서버 저장소에 매핑된 값이 있어야만 유효하다 — 그래서 로그아웃(`session.invalidate()`)
한 번으로 즉시 무효화할 수 있다. 이게 쿠키 단독 방식과의 근본적인 차이다: **세션은 서버가
"이 세션은 이제 끝났다"고 선언하는 순간 즉시 죽지만, 클라이언트가 들고 있는 서명된 토큰(JWT
등)은 서버가 강제로 회수할 방법이 없다** (만료 시간이 될 때까지는 여전히 유효한 토큰으로 남는다).

## 트레이드오프: 왜 요즘은 세션 대신 JWT로 가는가

세션의 대가는 **서버 자원**이다. 세션이 생길 때마다 서버 메모리(또는 별도 저장소)에
공간을 차지하고, 사용자가 늘어날수록 이 저장소도 함께 커진다. 더 큰 문제는 **수평
확장(scale-out)**이다. 서버가 여러 대로 늘어나면, 로그인한 서버와 다음 요청을 받는
서버가 다를 수 있는데 세션은 기본적으로 그 서버의 로컬 메모리에 있다. 해법은 두 가지뿐이다:

- **Sticky session**: 로드밸런서가 같은 사용자를 항상 같은 서버로 보낸다 — 서버 하나가
  죽으면 그 서버의 세션이 전부 날아간다.
- **세션 저장소 외부화**: Redis 같은 공유 저장소에 세션을 두고 모든 서버가 함께 참조한다 —
  별도 인프라와 네트워크 왕복 비용이 생긴다.

JWT 기반 인증이 뜬 이유가 여기 있다. 토큰 자체에 필요한 정보(사용자 ID, 권한 등)를
서명해서 담아두면, 서버는 서명만 검증하면 되고 별도 저장소 조회가 필요 없다 — 확장성
문제가 사라진다. 대신 앞서 말한 "즉시 무효화가 안 된다"는 대가를 치른다. 그래서 실무에서는
JWT의 만료 시간을 짧게 잡고, Refresh Token으로 갱신하는 구조를 함께 쓰는 경우가 많다.

## 여러 세션을 중앙에서 관리하기: HttpSessionListener

특정 조건의 세션을 찾아 강제 종료하거나, 접속 중인 모든 사용자에게 공지를 보내려면
개별 `HttpSession` 접근만으로는 부족하다 — 현재 살아있는 세션 전체를 한곳에서 알고
있어야 한다. `HttpSessionListener`가 이 역할을 한다.

```java
@WebListener
public class SessionConfig implements HttpSessionListener, HttpSessionAttributeListener {

    private static final Map<String, HttpSession> sessions = new ConcurrentHashMap<>();

    @Override
    public void sessionCreated(HttpSessionEvent event) {
        sessions.put(event.getSession().getId(), event.getSession());
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent event) {
        sessions.remove(event.getSession().getId());
    }
}
```

`ConcurrentHashMap`을 쓴 이유는 명확하다 — 세션 생성/소멸은 여러 요청 스레드에서
동시에 일어나는 이벤트라서, 동기화되지 않은 `HashMap`을 쓰면 레이스 컨디션으로 맵
내부 구조가 깨질 수 있다. `sessionDestroyed`는 `invalidate()` 호출뿐 아니라
`setMaxInactiveInterval`로 지정한 타임아웃이 지나 자동 만료될 때도 똑같이 호출되므로,
이 한 곳에서 정리 로직을 관리하면 명시적 로그아웃과 타임아웃 만료를 따로 처리할 필요가
없다.

과거엔 `web.xml`에 `<listener>`를 등록하는 방식이었지만, `@WebListener` 애노테이션으로
대체된 지 오래다 — XML 설정 방식은 스프링 부트 생태계에서 점점 쓰이지 않는 흐름과
맞물려 있다.

## 정리

- 쿠키: 클라이언트 저장, 서버 자원 안 씀, 위조 가능성 있음 → 서명 없이는 신뢰할 수 없음.
- 세션: 서버 저장, 즉시 무효화 가능, 확장 시 sticky session 또는 외부 저장소 필요.
- 여러 세션을 다루는 기능(강제 로그아웃, 공지 등)이 필요하면 `HttpSessionListener` +
  스레드 세이프한 컬렉션으로 중앙 관리한다.
