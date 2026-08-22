---
title: "gRPC, REST 대신 선택하는 기준 (with Spring Boot)"
date: 2025-05-19
categories: [Java/Spring]
---

REST API가 표준처럼 쓰이는 상황에서 gRPC를 도입하려면, "왜 굳이"에 먼저 답할 수 있어야
한다. 실무에서 Spring Boot 환경에 gRPC를 붙여본 경험을 기준으로, 언제 gRPC가 REST보다
나은 선택인지부터 정리한다.

## 왜 REST 대신 gRPC인가

JSON 기반 REST 통신의 비용은 두 군데서 발생한다.

1. **직렬화 비용**: 객체를 JSON 텍스트로 변환하고 다시 파싱하는 과정이 생각보다
   무겁다. 필드명을 매번 텍스트로 주고받는 것도 페이로드 크기를 키운다.
2. **스키마가 코드로 강제되지 않는다**: REST는 스펙 문서(OpenAPI 등)를 별도로
   관리해야 하고, 문서와 실제 구현이 어긋나는 걸 컴파일 타임에 잡을 방법이 없다.

gRPC는 **Protocol Buffer**로 이 두 문제를 동시에 해결한다. 바이너리 직렬화라 페이로드가
작고 빠르며, `.proto` 파일이 계약(contract) 그 자체라서 클라이언트/서버 코드가 여기서
자동 생성된다 — 스펙과 구현이 어긋날 수가 없는 구조다. 여기에 HTTP/2 기반이라
멀티플렉싱과 스트리밍을 기본으로 지원한다.

## 대가: gRPC를 피해야 하는 경우

- **브라우저에서 직접 호출이 필요할 때**: 브라우저는 HTTP/2의 트레일러 프레임을
  다루지 못해 gRPC를 네이티브로 지원하지 않는다. grpc-web 같은 프록시 계층이
  필요해지고, 그만큼 구조가 복잡해진다. 웹 프론트엔드가 주 클라이언트라면 REST나
  GraphQL이 여전히 더 간단하다.
- **디버깅 난이도**: JSON은 사람이 눈으로 읽고 curl로 바로 찔러볼 수 있지만, 바이너리
  프로토콜인 gRPC는 전용 도구(grpcurl, BloomRPC 등)가 없으면 요청 내용을 확인하기
  번거롭다.
- **외부 파트너/서드파티 연동**: 대부분의 외부 API는 REST/JSON을 기본으로 제공한다.
  범용성이 필요한 인터페이스라면 gRPC보다 REST가 여전히 안전한 선택이다.

즉 gRPC는 **내부 서비스 간 통신**(MSA 내부, 고성능이 필요한 백엔드-백엔드 구간)에서
강점이 크고, 외부에 공개하는 API나 브라우저가 직접 붙는 구간에는 잘 안 맞는다.

## Spring Boot 설정

`net.devh:grpc-spring-boot-starter`가 필요한 의존성(`grpc-netty-shaded`,
`grpc-protobuf`, `grpc-stub`)을 통합 관리해준다.

```groovy
implementation 'net.devh:grpc-spring-boot-starter'
```

## .proto: 계약을 코드로 정의하기

```protobuf
syntax = "proto3";

package helloworld;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```

`.proto`는 크게 **서비스 정의**(어떤 메서드가 있는가)와 **메시지 정의**(요청/응답
타입)로 구성된다. REST의 인터페이스 명세 + DTO를 한 파일에 합쳐놓은 셈이다.

메시지 필드 뒤에 붙는 `= 1`, `= 2` 같은 숫자는 **필드 번호(Field Number)**다.
직렬화 시 필드명이 아니라 이 번호로 데이터를 식별하기 때문에, **한번 배포된 필드
번호는 절대 재사용하면 안 된다** — 재사용하면 구버전 클라이언트가 이 필드를 다른
의미로 잘못 해석하게 된다. 필드를 제거할 때는 번호를 `reserved`로 명시해 재사용을
막는 것이 안전하다. 이게 gRPC의 하위 호환성 관리 방식이다.

`repeated`는 반복 가능한 필드(배열), `optional`은 값이 없으면 타입별 기본값(문자열
`""`, 정수 `0`)이 들어간다는 뜻이다.

빌드 시 `protobuf-gradle-plugin`이 `.proto`를 실제 Java 클래스(서비스 스텁 + 메시지
클래스)로 변환한다.

```groovy
plugins {
  id 'com.google.protobuf' version '0.9.2'
}
protobuf {
    protoc {
        artifact = 'com.google.protobuf:protoc:3.17.3'
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}
```

## 통신 방식 선택: Blocking vs Future

gRPC 클라이언트 스텁은 크게 두 가지 호출 방식을 제공한다.

```java
// 동기 - 응답이 올 때까지 스레드가 블로킹된다
Fromage.SetUserInforResponse response = blockingStub.setUserInfoWith(request);

// 비동기 - 즉시 ListenableFuture를 반환, 호출 스레드는 블로킹되지 않는다
ListenableFuture<Fromage.SetUserInforResponse> futureResponse = futureStub.setUserInfoWith(request);
Fromage.SetUserInforResponse response = futureResponse.get(); // 여기서 기다리면 사실상 동기와 동일해진다
```

**판단 기준**: 호출 스레드가 다른 작업과 병행할 여지가 있는가. 단순히 응답을 받아서
바로 다음 로직에 써야 한다면 `blockingStub`이 코드가 단순해서 낫다. 여러 gRPC 호출을
동시에 날리고 결과를 나중에 합쳐야 하는 상황(예: 병렬로 3개 서비스에 조회 요청)이라면
`futureStub` + `ListenableFuture` 조합이 스레드를 낭비하지 않는다. `.get()`을 호출하는
순간 블로킹으로 되돌아간다는 점은 꼭 기억해야 한다 — future를 받아놓고 바로 `.get()`을
부르면 비동기로 얻는 이득이 없다.

## 정리

gRPC를 고르는 기준은 "빠르다"가 아니라 **"계약이 코드로 강제되어야 하고, 내부 서비스
간 통신이며, 클라이언트가 브라우저가 아닌가"** 세 가지다. 이 조건에 안 맞으면 REST가
여전히 더 단순하고 생산적인 선택이다.
