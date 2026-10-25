---
title: "쿠버네티스 기본기: Pod, Deployment, Service의 역할 분리"
date: 2025-05-24
categories: [인프라]
---

쿠버네티스를 처음 배울 때 헷갈리는 지점은 대개 "왜 개념이 이렇게 많이 나뉘어 있는가"다.
컨테이너 하나만 잘 띄우면 될 것 같은데 Pod, Deployment, Service가 따로 존재한다.
각각이 어떤 문제를 풀기 위해 분리됐는지를 알면 훨씬 덜 헷갈린다.

## 쿠버네티스가 푸는 문제

컨테이너 기술(도커) 자체는 "애플리케이션 하나를 격리해서 실행하는 방법"만 제공한다.
컨테이너가 수백~수천 개가 되면 "어떤 컨테이너를 어떤 서버(노드)에 배치할지",
"트래픽이 몰리면 몇 개를 늘릴지", "죽으면 어떻게 되살릴지"는 별도의 문제가 된다.
쿠버네티스는 이 배치·확장(오토 스케일링)·자가 치유를 자동화하는 오케스트레이션 시스템이다.

## Pod: 왜 컨테이너를 직접 다루지 않는가

Pod는 쿠버네티스가 관리하는 최소 배포 단위다. 컨테이너를 직접 실행하는 게 아니라
컨테이너들이 실행될 논리적 환경(네트워크, 스토리지)을 제공한다.

Docker Compose와 헷갈리기 쉬운데, 목적이 다르다.

| 특징 | Docker Compose | Kubernetes Pod |
|---|---|---|
| 주요 목적 | 로컬 개발, 단일 호스트 다중 컨테이너 실행 | 클러스터 내 배포의 최소 단위 |
| 범위 | 보통 애플리케이션 전체(웹서버+DB+캐시) | 애플리케이션의 한 부분/인스턴스 |
| 네트워킹 | 서비스별 별도 IP, 서비스 이름으로 통신 | Pod 내 모든 컨테이너가 같은 IP/포트 공유 |
| 컨테이너 관계 | 독립 프로세스로도 실행 가능 | 반드시 함께 시작·종료, 서로 강하게 의존 |

**Pod가 여러 컨테이너를 묶을 수 있다는 게 핵심이다.** 그런데 실무에서는 Pod에
컨테이너 하나만 넣는 경우가 대부분이다 — 여러 개를 넣는 건 메인 컨테이너를 보조하는
사이드카(로그 수집기, 프록시 등)가 반드시 생명주기를 같이해야 할 때뿐이다. "Pod = 컨테이너
여러 개를 넣는 곳"으로 이해하면 안 되고, "생명주기를 반드시 공유해야 하는 것들의 단위"로
이해해야 한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: spring-pod
spec:
  containers:
    - name: spring-container
      image: spring-server
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 8080
```

`imagePullPolicy`는 로컬 실습에서 자주 발목을 잡는 설정이다.

| 값 | 동작 |
|---|---|
| `Always` | 항상 원격 레지스트리에서 새로 받는다. 로컬 캐시 무시. |
| `IfNotPresent` | 로컬에 있으면 그걸 쓰고, 없으면 원격에서 받는다. |
| `Never` | 로컬 이미지만 쓴다. 없으면 Pod 생성 실패. |
| 미지정 | 태그가 `latest`면 `Always`, 아니면 `IfNotPresent`로 취급된다. |

`ImagePullBackOff` 에러를 만났다면 대개 로컬에서만 빌드한 이미지를 원격 레지스트리에서
찾으려 한 경우다 — 정책과 이미지 출처가 어긋난 것이다.

## Deployment: Pod 하나로는 왜 부족한가

Pod를 여러 개 복사해서 쓰고 싶을 때 YAML을 계속 복사-붙여넣기 하는 건 비효율적이다.
그보다 근본적인 문제는, Pod 하나만 떠 있으면 그게 죽었을 때 아무도 되살려주지 않는다는
것이다. Deployment는 이 두 문제를 함께 해결한다 — "몇 개를 유지할지"를 선언하면,
쿠버네티스가 그 개수를 계속 맞춰준다(ReplicaSet을 통해).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-app
  template:
    metadata:
      labels:
        app: backend-app
    spec:
      containers:
        - name: spring-container
          image: spring-server
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

`replicas: 3`의 의미는 단순히 "부하 분산을 위해 3개를 띄운다"가 아니다. 더 중요한 건
**가용성**이다 — Pod 하나가 노드 장애나 컨테이너 내부 오류로 죽어도, 컨트롤 플레인이
"설정된 개수(3)보다 부족하다"는 걸 감지하고 즉시 새 Pod를 만들어 채운다. 확장(scale)과
자가 치유(self-healing)가 사실 같은 메커니즘에서 나온다는 걸 이해하면, replicas 값을
1로 두는 게 왜 위험한지도 명확해진다 — 그 하나가 죽는 순간 서비스가 완전히 끊긴다.

## Service: Pod의 IP가 계속 바뀌는 문제

Pod는 재생성될 때마다 내부 IP가 바뀐다. Deployment가 Pod를 죽이고 새로 만드는 걸
반복하는 이상, "이 IP로 접속하면 된다"는 방식은 성립하지 않는다. Service는 여러 Pod
앞에 변하지 않는 진입점을 제공해서 이 문제를 해결한다.

| 타입 | 특징 | 외부 접근 | 주 사용처 |
|---|---|---|---|
| ClusterIP (기본) | 클러스터 내부 전용 IP | 불가 | 내부 서비스 간 통신 (프론트→백엔드 API) |
| NodePort | 각 노드의 특정 포트를 개방 | 가능 | 개발/테스트, 외부 로드밸런서가 아직 없을 때 |
| LoadBalancer | 클라우드의 외부 로드밸런서를 자동 프로비저닝 | 가능 | 프로덕션에서 외부에 안정적으로 노출 |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-service
spec:
  type: NodePort
  selector:
    app: backend-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30000
```

Service의 `selector`가 Deployment의 `labels`와 일치해야 트래픽이 연결된다는 점이
실무에서 흔한 실수 지점이다 — 리소스 종류가 다르다 보니 라벨을 복사하면서 오타를
내거나 누락해도 쿠버네티스는 별다른 에러 없이 그냥 "연결된 Pod가 0개인 Service"를
만들어버린다.

## 정리

- Pod는 컨테이너가 아니라 "생명주기를 공유하는 단위"다.
- Deployment의 `replicas`는 확장과 가용성 둘 다를 위한 설정이다 — 1로 두면 사실상
  단일 장애점(SPOF)이 된다.
- Service는 Pod IP의 휘발성을 감추기 위한 안정적 진입점이며, `selector`/`labels`
  일치가 전제 조건이다.

세 개념을 하나씩 손으로 만져보고 싶다면, [로컬 kind 클러스터로 실습해보는 글]({{ '/posts/kind-kubernetes-local-cluster/' | relative_url }})에서 이어서 다룬다.
