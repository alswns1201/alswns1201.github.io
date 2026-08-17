---
title: "Spring Cloud Eureka로 보는 서비스 디스커버리"
date: 2025-12-26
categories: [아키텍처]
tags: [msa, spring-boot, service-discovery]
---

MSA에서 서비스가 늘어나면 가장 먼저 마주치는 질문이 있다 — "서비스가 서로의 주소를
어떻게 찾을 것인가?" IP/포트를 코드나 설정에 하드코딩하면 인스턴스가 늘거나 포트가
바뀔 때마다 여기저기 고쳐야 하고, 장애가 나도 유연하게 대응할 수 없다. 이 문제를 푸는
게 **Service Discovery**고, Spring 진영의 대표 구현체가 **Eureka**다.

## 구조: 서버는 주소록, 클라이언트는 등록자이자 조회자

- **Eureka Server**: 서비스 정보를 저장하는 중앙 레지스트리
- **Eureka Client**: 자신을 등록하고, 다른 서비스를 조회하는 주체

Eureka를 쓰면 서비스는 IP/포트가 아니라 **이름만으로 통신**하고, 실제 주소는 Eureka가
관리한다. 인스턴스가 늘거나 줄어도 자동으로 반영된다.

### 클라이언트 등록 흐름

1. 애플리케이션 기동
2. Eureka Server 주소 확인
3. 자신의 IP/Port/상태 정보 전송
4. Eureka Server에 인스턴스 등록
5. 30초 주기로 heartbeat 전송
6. 서버는 상태를 UP으로 유지

```yaml
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8761/eureka/  # 반드시 /eureka 경로
```

관리 UI(`http://localhost:8761/`)와 실제 클라이언트 통신 엔드포인트(`/eureka`)는
다르다 — Client의 `defaultZone`을 대시보드 주소로 잘못 넣는 실수가 은근히 흔하다.

## Eureka의 진짜 약점: 자기보호모드(Self-Preservation)

Eureka를 실무에서 쓸 때 가장 중요한 건 기능 사용법이 아니라 **장애 상황에서 어떻게
행동하는가**다. Eureka는 CAP 이론 기준으로 **AP(가용성 우선)** 시스템으로 설계됐다 —
네트워크 파티션이 발생해서 일부 인스턴스로부터 heartbeat가 끊겨도, Eureka Server는
"진짜 죽은 건지 네트워크 문제인지" 확신할 수 없으면 **레지스트리에서 즉시 제거하지 않고
그대로 유지**한다. 이게 자기보호모드다.

이 설계의 트레이드오프는 명확하다: 실제로 죽은 인스턴스로 트래픽이 계속 라우팅될 수
있다(가용성을 위해 정확성을 희생). 반대로 즉시 제거하는 CP 방향으로 설계했다면, 네트워크
문제로 잠깐 heartbeat가 끊긴 멀쩡한 인스턴스까지 레지스트리에서 빠져나가서 불필요한
장애 전파가 일어날 수 있다. Eureka는 후자의 위험을 더 크게 보고 전자를 감수하는
쪽을 택한 것이다 — 이 사실을 모르고 "왜 죽은 인스턴스로 요청이 계속 가지?"라고
당황하면 안 된다.

## 언제 적합하고 언제 부적합한가

**적합한 경우**: Spring 기반 MSA 학습, Gateway + Service Discovery 구조를 이해하려는
경우, 이미 Eureka로 구축된 레거시 MSA 유지보수.

**부적합한 경우**: Kubernetes 환경. K8s는 자체 DNS 기반 서비스 디스커버리(Service
리소스 + kube-dns/CoreDNS)를 이미 갖고 있어서, 그 위에 Eureka를 얹으면 디스커버리
메커니즘이 두 겹으로 겹친다 — 오히려 장애 지점과 운영 복잡도만 늘어난다. 최신
클라우드 네이티브 아키텍처에서 Eureka를 새로 도입할 이유는 거의 없다.

## 참고: Spring Cloud 버전에 따른 활성화 방식 차이

Spring Cloud 2023.x까지는 `eureka-client` 의존성만 추가하면 Discovery Client가 자동
활성화됐지만, 최신 버전에서는 `@EnableDiscoveryClient`를 명시적으로 붙여야 하는 경우가
있다 — 의존성만 넣으면 알아서 될 거라 생각하고 넘어갔다가 "서비스가 대시보드에
안 보이는" 문제로 이어지는 흔한 원인 중 하나다.

### 자주 겪는 문제

- **대시보드에 서비스가 안 보임**: Client 의존성 누락, Spring Cloud BOM 미적용, 애플리케이션
  기동 실패, Spring Boot/Cloud 버전 호환 문제 중 하나일 가능성이 높다.
- **종료 시 `Connection refused` WARN**: 애플리케이션 종료 시 등록 해제 요청을 보내는데
  그 시점에 Eureka Server가 이미 꺼져 있으면 발생한다. 정상적인 종료 시나리오이므로
  무시해도 된다.

## 정리

- Eureka는 이름 기반 통신으로 서비스 주소 관리 문제를 해결한다.
- 진짜 이해해야 할 부분은 등록 흐름이 아니라 **자기보호모드가 만드는 가용성/정확성
  트레이드오프**다.
- Kubernetes 환경에서는 굳이 도입하지 않는다 — 이미 있는 디스커버리 메커니즘과 중복된다.
