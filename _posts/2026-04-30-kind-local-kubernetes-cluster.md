---
title: "kind로 로컬 쿠버네티스 클러스터 띄우고 Spring Boot 앱 배포하기"
date: 2026-04-30
categories: [MSA, 쿠버네티스]
tags: [kubernetes, docker, devops]
---

쿠버네티스 개념([Pod, Deployment, Service의 역할 분리](/posts/kubernetes-basics/))을
글로 읽는 것과, 실제로 로컬에서 클러스터를 띄워 애플리케이션을 배포해보는 건 다른
문제다. 클라우드 클러스터를 매번 띄우기엔 비용과 시간이 아깝고, 그렇다고 개념만
알고 넘어가면 YAML 필드 하나하나가 실제로 뭘 하는지 감이 안 잡힌다. **kind**(Kubernetes
IN Docker)는 이 간극을 메워준다 — 가상 머신 없이 도커 컨테이너를 쿠버네티스 노드로
써서, 로컬 환경에서 가볍고 빠르게 진짜 클러스터를 띄울 수 있다.

```bash
choco install kind      # (macOS는 brew install kind)
kind create cluster
```

클러스터는 컨트롤 플레인(전체 상태를 결정·관리하는 지휘소)과 노드(컨테이너가 실제로
배치되는 작업 공간)로 나뉜다. kind 환경에서는 도커 컨테이너 하나가 이 역할을 전부
떠맡는다.

## JAR를 이미지로 만든다

쿠버네티스는 JAR 파일을 직접 다루지 않는다 — 컨테이너 단위로만 관리한다. 그래서
빌드된 JAR를 실행할 수 있는 자바 환경이 포함된 이미지로 감싸야 한다. 이때 컴파일된
클래스 파일의 버전과 실행 환경의 자바 버전이 맞아야 한다는 걸 놓치기 쉽다.

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
COPY math-0.0.1-SNAPSHOT.jar math-0.0.1-SNAPSHOT.jar
ENTRYPOINT ["java", "-jar", "/math-0.0.1-SNAPSHOT.jar"]
```

## Deployment와 Service: "원하는 상태"를 선언한다

쿠버네티스는 명령형이 아니라 선언형이다 — "이렇게 해라"가 아니라 "이 상태를
유지해라"를 YAML로 적어두면, 컨트롤 플레인이 현재 상태를 계속 그 목표에 맞춘다.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app-container
          image: my-app:v1
          imagePullPolicy: Never
          ports:
            - containerPort: 8080
```

`replicas: 2`가 셀프 힐링의 핵심이다. 포드 하나가 에러로 죽거나 노드에서 사라지면,
컨트롤 플레인은 "설정된 개수(2)보다 현재 포드 수가 부족하다"는 걸 감지하고 즉시
새 포드를 만들어 개수를 다시 맞춘다. 애플리케이션 코드는 이 복구 로직을 전혀 몰라도
된다 — 인프라 레벨에서 알아서 처리된다.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30001
```

Service가 필요한 이유는 포드의 내부 주소가 재생성될 때마다 바뀌기 때문이다. Deployment가
포드를 새로 띄울 때마다 IP가 달라지면, 그걸 직접 호출하던 쪽은 매번 주소를 다시
알아내야 한다. Service는 `selector`로 대상 포드들을 묶어서, 포드가 몇 번을 다시
생성되든 변하지 않는 진입점 하나를 제공한다.

## `imagePullPolicy: Never`가 로컬 실습에서 중요한 이유

이미지를 어디서 가져올지 결정하는 옵션인데, 로컬 kind 환경에서는 기본값 그대로 두면
바로 함정에 빠진다.

- **Never**: 외부 저장소에서 이미지를 절대 받아오지 않는다. 로컬에서 빌드한 이미지를
  `kind load docker-image` 명령으로 클러스터 노드 내부에 직접 넣어준 것만 쓰도록
  강제한다.
- **IfNotPresent**: 노드에 이미지가 있으면 그걸 쓰고, 없으면 외부 저장소에서 받아온다.

로컬 클러스터는 도커 레지스트리에 push한 적 없는, 방금 빌드한 이미지를 쓰는 게
일반적이다. 이때 `imagePullPolicy`가 기본값(`Always` 또는 태그에 따라 `IfNotPresent`)이면
쿠버네티스가 존재하지도 않는 원격 저장소에서 이미지를 당겨오려다 실패한다 — 로컬에
이미지가 멀쩡히 있는데도 `ImagePullBackOff`가 뜨는 흔한 원인이 이거다. `Never`로
명시하면 "이 이미지는 로컬에 있는 것만 쓴다"는 의도가 분명해지고, 실수로 레지스트리를
호출하는 일 자체가 없어진다.

## Kustomization: 여러 YAML을 한 번에

Deployment와 Service처럼 항상 같이 적용해야 하는 매니페스트가 여러 개면, 매번
`kubectl apply -f`를 반복하는 대신 Kustomization으로 묶는다.

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  project: my-first-k8s
```

```
project/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml
```

```bash
kubectl apply -k .
```

`commonLabels`처럼 여러 리소스에 공통으로 적용할 설정을 한 곳에 모아둘 수 있다는
것도 장점이다 — 리소스가 늘어날수록 각 YAML에 같은 라벨을 반복해서 넣는 대신
Kustomization 한 곳만 고치면 된다.

## 정리

- kind는 VM 없이 도커 컨테이너를 노드로 써서 로컬에 진짜 쿠버네티스 클러스터를
  띄워준다 — 클라우드 클러스터 없이 개념을 손으로 확인해보기에 적합하다.
- JAR는 쿠버네티스가 직접 다루는 단위가 아니다. 자바 버전이 맞는 베이스 이미지로
  감싸서 컨테이너 이미지를 만들어야 한다.
- Deployment는 "이 상태를 유지해라"는 선언이고, `replicas`가 셀프 힐링의 근거다.
  Service는 포드의 흔들리는 주소 대신 고정된 진입점을 제공한다.
- 로컬에서 빌드한 이미지를 쓸 땐 `imagePullPolicy: Never`를 명시해야 한다 —
  아니면 존재하지 않는 원격 저장소를 찾다가 `ImagePullBackOff`로 막힌다.
- 매니페스트가 여러 개로 늘어나면 Kustomization으로 묶어서 `kubectl apply -k .`
  한 번에 적용한다.
