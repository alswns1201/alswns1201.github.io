---
title: "Spring Security 필터 기반 인증 직접 구현하기 — 그리고 이 방식의 함정들"
date: 2026-01-21
categories: [Spring Boot]
tags: [spring-security, jwt, spring-boot]
---

Spring Security로 로그인을 구현하는 방법은 크게 두 갈래다. `UserDetailsService` +
`AuthenticationProvider`를 표준 흐름에 태우는 방식, 또는 `UsernamePasswordAuthenticationFilter`를
상속해서 필터 자체를 새로 쓰는 방식. 여기서는 후자를 다룬다 — 인증 요청을 가로채는
지점부터 토큰 발급까지 직접 제어하고 싶을 때 쓰는 패턴이다.

## 필터 체인 구성

```java
protected SecurityFilterChain configure(HttpSecurity http) throws Exception {
    AuthenticationManagerBuilder authenticationManagerBuilder =
            http.getSharedObject(AuthenticationManagerBuilder.class);
    authenticationManagerBuilder.userDetailsService(userService).passwordEncoder(bCryptPasswordEncoder);
    AuthenticationManager authenticationManager = authenticationManagerBuilder.build();

    http.csrf((csrf) -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
                .requestMatchers("/h2-console/**").permitAll()
                .requestMatchers("/**").access(
                        new WebExpressionAuthorizationManager("hasIpAddress('127.0.0.1') or hasIpAddress('::1'))")
                )
                .anyRequest().authenticated()
            )
            .authenticationManager(authenticationManager)
        .httpBasic(Customizer.withDefaults())
        .headers((headers) -> headers
            .frameOptions((frameOptions) -> frameOptions.sameOrigin()));

    return http.build();
}
```

`UserDetailsService`를 직접 구현한다:

```java
public interface UserService extends UserDetailsService {
    UserDto getUserDetailsByEmail(String username);
}

public class UserServiceImpl implements UserService {
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserEntity userEntity = userRepository.findByEmail(username);
        if (userEntity == null)
            throw new UsernameNotFoundException(username + ": not found");

        return new User(userEntity.getEmail(), userEntity.getEncryptedPwd(),
                true, true, true, true, new ArrayList<>());
    }
}
```

여기서 로그인 아이디로 email을 쓴다는 게 포인트다 — `loadUserByUsername`의 파라미터명은
관습적으로 username이지만, 실제로 어떤 값을 유저 식별자로 쓸지는 구현이 정한다.

## 인증 필터

```java
public class AuthenticationFilter extends UsernamePasswordAuthenticationFilter {
    private UserService userService;
    private Environment env;

    @Override
    public Authentication attemptAuthentication(HttpServletRequest req, HttpServletResponse res)
            throws AuthenticationException {
        try {
            RequestLogin creds = new ObjectMapper().readValue(req.getInputStream(), RequestLogin.class);
            return getAuthenticationManager().authenticate(
                    new UsernamePasswordAuthenticationToken(creds.getEmail(), creds.getPassword(), new ArrayList<>())
            );
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest req, HttpServletResponse res,
                                            FilterChain chain, Authentication authResult)
            throws IOException, ServletException {
        String userEmail = ((User) authResult.getPrincipal()).getUsername();
        UserDto userDetails = userService.getUserDetailsByEmail(userEmail);

        byte[] secretKeyBytes = env.getProperty("token.secret").getBytes(StandardCharsets.UTF_8);
        SecretKey secretKey = Keys.hmacShaKeyFor(secretKeyBytes);
        Instant now = Instant.now();

        String token = Jwts.builder()
                .subject(userDetails.getUserId())
                .expiration(Date.from(now.plusMillis(Long.parseLong(env.getProperty("token.expiration-time")))))
                .issuedAt(Date.from(now))
                .signWith(secretKey)
                .compact();

        res.addHeader("token", token);
        res.addHeader("userId", userDetails.getUserId());
    }
}
```

## 이 구조에서 실무적으로 짚어야 할 지점들

**1. 토큰을 응답 헤더로 노출하면 클라이언트 저장 방식이 곧 보안 모델을 결정한다.**
`res.addHeader("token", token)`으로 넘긴 토큰을 프론트에서 어디에 저장하느냐가 관건이다.
JS로 접근 가능한 곳(localStorage, 커스텀 헤더로 재사용)에 두면 XSS 한 방에 탈취된다.
이 문제 때문에 실무에서는 토큰을 `HttpOnly` 쿠키로 내려주는 방식을 더 선호하는데, 그건
서버가 `Set-Cookie` 헤더로 직접 내려야 한다는 뜻이라 이 필터 구조와는 응답 처리 방식이
달라진다. "토큰을 어떻게 만드느냐"와 "토큰을 어떻게 전달하느냐"는 별개 문제이고, 후자를
안 챙기면 아무리 서명을 잘 해도 소용없다.

**2. IP 기반 접근 제어는 프록시 뒤에서 무력화된다.** `hasIpAddress('127.0.0.1')`는 클라이언트가
서버에 직접 붙는 환경에서만 의미가 있다. 앞단에 로드밸런서나 리버스 프록시(Nginx, ALB 등)가
있으면 Spring이 보는 remote address는 프록시의 IP이지 실제 클라이언트 IP가 아니다.
`X-Forwarded-For`를 신뢰하도록 별도 설정(`ForwardedHeaderFilter` 등)을 하지 않으면 이 규칙은
있으나 마나 하거나, 반대로 헤더를 스푸핑하는 요청에 뚫릴 수 있다.

**3. HMAC 키 길이는 알고리즘이 요구하는 최소 엔트로피를 만족해야 한다.** `Keys.hmacShaKeyFor`는
내부적으로 키 바이트 길이가 알고리즘(예: HS256은 256비트=32바이트) 최소 요구치에 못 미치면
예외를 던진다. `token.secret` 프로퍼티 값을 사람이 읽기 좋은 짧은 문자열로 넣으면 애플리케이션이
기동 시점이 아니라 첫 로그인 요청이 올 때 런타임 예외로 터진다 — 배포 후에야 발견되는
전형적인 케이스다.

**4. 인증 실패 흐름이 이 코드엔 없다.** `attemptAuthentication`에서 인증이 실패하면 Spring
Security 기본 `AuthenticationFailureHandler`가 401을 내려주긴 하지만, 클라이언트가 구분 가능한
에러 바디(아이디 없음/비번 틀림/잠긴 계정)를 원한다면 `unsuccessfulAuthentication`을 오버라이드해서
직접 응답을 만들어야 한다. 안 하면 프론트는 "로그인 실패"라는 뭉뚱그려진 상태만 받는다.

이 패턴 자체는 낡지 않았다 — 다만 필터를 직접 짠다는 건 Spring이 기본으로 챙겨주는 예외
처리, 헤더 전달 관례, 프록시 인지를 전부 직접 챙겨야 한다는 뜻이기도 하다. "직접 구현"의
비용은 여기서 나온다.
