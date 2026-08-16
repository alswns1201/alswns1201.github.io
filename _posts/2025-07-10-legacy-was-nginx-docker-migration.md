---
title: "레거시 WAS & Nginx를 Docker로 현대화하기: 마이그레이션 A to Z 가이드"
date: 2025-07-10
categories: [아키텍쳐 설계 관련 글, 서버,인프라]
tags: [docker, devops, architecture]
---

서버에 Nginx와 Tomcat을 직접 설치해 운영하는 방식은 눈에 보이는 문제가 없다가도,
서버를 옮기거나 복제해야 하는 순간 비용이 드러난다. "왜 굳이 Docker로 바꾸는가"에
대한 답은 결국 **의존성 지옥, 환경 불일치, 이관의 어려움** 세 가지로 요약된다 — OS/
라이브러리 버전에 따라 애플리케이션이 오작동하고, 개발/테스트/운영 환경이 미묘하게
달라 예상 못 한 버그가 생기고, 서버를 옮길 때마다 설정을 손으로 다시 해야 한다.
Docker는 애플리케이션과 실행 환경을 컨테이너라는 단위로 묶어서 이 문제들을 구조적으로
없앤다.

## 1단계: 이관 전에 AS-IS를 정확히 파악한다

마이그레이션에서 가장 실수하기 쉬운 지점은 "그냥 최신 버전으로 새로 깔지"라는 유혹이다.
버전이 다르면 설정 문법이나 기본 동작이 달라질 수 있고, 그 차이가 이관 후 원인 불명의
장애로 나타난다. 그래서 첫 단계는 항상 기존 환경을 있는 그대로 확인하는 것이어야 한다.

```bash
# Nginx 버전, 실행 포트, 설정 파일 경로
nginx -v
sudo netstat -nlpt | grep nginx
sudo nginx -t   # 문법 검사 + 설정 파일 경로를 함께 알려주는 가장 확실한 방법

# Tomcat 버전
cd /usr/local/tomcat/bin && ./version.sh
```

Nginx 설정에서 특히 눈여겨봐야 할 건 `proxy_pass` 부분이다 — 이게 WAS로 요청을
넘기는 지점이라, Docker 환경으로 옮기면서 반드시 값이 바뀌어야 하는 곳이기 때문이다
(뒤에서 다시 다룬다). HTTPS를 쓰고 있다면 `ssl_certificate`/`ssl_certificate_key`
경로에 있는 인증서 파일도 함께 챙겨야 한다 — 이걸 놓치면 새 서버에서 HTTPS가 통째로
안 뜬다.

## 2단계: 마이그레이션 프로젝트 구조 만들기

```
/home/ubuntu/my-app/
├── docker-compose.yml
├── was/
│   ├── Dockerfile
│   └── ROOT.war
├── nginx/
│   └── nginx.conf
├── ssl/
│   ├── fullchain.pem
│   └── privkey.pem
└── logs/
    └── nginx/
```

설정 파일을 이렇게 미리 구조화해두는 이유는, `docker-compose.yml`에서 이 경로들을
볼륨으로 그대로 마운트해서 쓰기 때문이다. 구조가 잡혀 있으면 이후 설정 변경이 컨테이너
재빌드 없이 파일 수정만으로 끝난다.

## 3단계: Docker Compose로 WAS + Nginx 함께 띄우기

**WAS Dockerfile** — 기존 서버와 동일한 Tomcat 버전을 베이스 이미지로 맞추는 게
중요하다. 버전이 다르면 WAR가 배포는 되어도 미묘한 동작 차이가 날 수 있다.

```dockerfile
FROM tomcat:9.0-jdk11-openjdk
COPY ./ROOT.war /usr/local/tomcat/webapps/
```

**Nginx 설정에서 실제로 바뀌는 건 딱 한 줄이다.** 기존:

```
proxy_pass http://127.0.0.1:8080;
```

Docker 환경에서는 WAS가 더 이상 같은 호스트의 `127.0.0.1`이 아니라 별도 컨테이너다.
Docker Compose가 만들어주는 내부 네트워크에서는 **서비스 이름 자체가 DNS 이름**이
되므로:

```
proxy_pass http://was:8080;
```

이 한 줄을 놓치는 게 이런 마이그레이션에서 가장 흔한 실수다 — 로컬에서는 멀쩡히
동작하던 설정을 그대로 복사해오면, Nginx 컨테이너 입장에서 `127.0.0.1`은 "Nginx
컨테이너 자기 자신"을 가리키게 되어 WAS로 요청이 아예 안 간다.

**docker-compose.yml**로 두 서비스를 묶는다.

```yaml
services:
  was:
    build:
      context: ./was
    restart: always

  nginx:
    image: nginx:1.18.0
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
      - ./logs/nginx:/var/log/nginx
      - /etc/localtime:/etc/localtime:ro
    depends_on:
      - was
```

몇 가지 설계 판단이 여기 숨어 있다.

- `nginx.conf`를 이미지에 굽지 않고 **볼륨으로 마운트**한 이유: 설정만 바꿀 때 이미지를
  다시 빌드할 필요가 없어진다. 운영 중 자잘한 Nginx 설정 변경이 잦다면 이 차이가 꽤 크다.
- `/etc/localtime:ro`를 마운트하는 이유: 컨테이너는 기본적으로 UTC를 쓰는 경우가 많아서,
  이걸 안 맞추면 Nginx 로그의 타임스탬프가 실제 서버 시간과 어긋난다 — 장애 시각을
  로그로 추적할 때 이게 꼬이면 매우 성가시다.
- `depends_on: was`는 **시작 순서만** 보장하지, WAS 애플리케이션이 실제로 요청을 받을
  준비가 됐는지(헬스체크)는 보장하지 않는다. Tomcat 컨테이너가 "떠 있지만 아직 앱
  배포 중"인 짧은 구간에 Nginx가 먼저 요청을 넘기면 502가 날 수 있다 — 트래픽이 있는
  환경으로 갈수록 `depends_on`만으로는 부족하고 healthcheck를 별도로 구성해야 한다.

## 4단계: DNS 전환

마지막은 도메인의 A 레코드를 새 서버의 공인 IP로 바꾸는 것이다. 여기서 놓치기 쉬운
건 **DNS 전파(propagation) 시간**이다 — 몇 분에서 최대 48시간까지 걸릴 수 있어서,
전환 직후 일부 사용자는 여전히 예전 서버로 요청이 갈 수 있다. 이 기간 동안은 예전
서버를 바로 내리지 말고 최소한 읽기 전용으로라도 살려두는 편이 안전하다 — 그렇지
않으면 전파가 덜 된 사용자들이 그 창구에서 서비스 장애를 겪는다.

## 정리

이 마이그레이션에서 진짜 중요한 결정 지점은 세 곳이다: **Nginx의 `proxy_pass` 대상을
컨테이너 이름으로 바꾸는 것**(안 바꾸면 통신 자체가 안 됨), **설정 파일을 이미지가
아니라 볼륨으로 분리하는 것**(운영 편의성), 그리고 **`depends_on`이 헬스체크가
아니라는 것을 인지하고 트래픽이 있는 환경에서는 별도 대비를 하는 것**(초기 요청
유실 방지). 나머지는 대부분 기계적인 작업이다.
