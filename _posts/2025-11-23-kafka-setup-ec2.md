---
title: "AWS EC2 프리티어에 Kafka 설치하기, 그리고 이 구성이 왜 개발용일 수밖에 없는가"
date: 2025-11-23
categories: [Kafka]
---

Kafka를 처음 접할 때 가장 좋은 방법은 로컬이 아니라 실제 서버에 직접 설치해보는
것이다. 다만 AWS 프리티어(t3.micro, 메모리 1GB) 위에 올리다 보면 튜토리얼에는 잘
안 나오는 제약들과 바로 부딪힌다 — 그리고 그 제약들이 오히려 Kafka가 프로덕션에서
왜 그렇게 무겁게 취급되는지를 이해하는 데 도움이 된다.

## 설치 자체는 단순하다

```bash
sudo apt update
sudo apt install openjdk-17-jdk   # Kafka 4.x는 JDK 17 이상 요구

wget https://dlcdn.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz
tar -xzf kafka_2.13-4.0.0.tgz
```

## 문제 1: 메모리 1GB로는 기본 힙 설정조차 버겁다

Kafka는 기본 힙 크기를 넉넉하게 잡도록 설계되어 있다 — 프로덕션에서는 보통 6GB
이상을 권장한다. t3.micro의 1GB 메모리로는 어림도 없어서, 강제로 낮춰야 한다.

```bash
export KAFKA_HEAP_OPTS="-Xmx400m -Xms400m"
```

이렇게 줄여도 OS와 JVM 자체 오버헤드까지 감안하면 여유가 거의 없다. 그래서 스왑을
추가로 설정해 디스크를 메모리처럼 쓰게 만든다.

```bash
sudo dd if=/dev/zero of=/swapfile bs=128M count=16
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

**여기서 중요한 건 이게 정상적인 해법이 아니라 "학습용 임시방편"이라는 점을 아는
것이다.** 스왑은 디스크 I/O이므로 메모리 접근보다 자릿수가 다르게 느리다. Kafka처럼
지연시간에 민감한 로그 기반 시스템에서 스왑이 활성화된다는 건 사실상 브로커가
성능 저하 상태로 들어간다는 뜻이다. 실습 환경에서는 "일단 켜지게" 만드는 용도로
쓰지만, 프로덕션에서 스왑에 의존한다는 건 인스턴스 스펙 자체가 부족하다는 신호로
봐야 한다.

## 문제 2: advertised.listeners — 클라우드에서 가장 많이 걸려 넘어지는 설정

```bash
vi config/server.properties
```

기본값의 `advertised.listeners`는 `localhost`를 가리킨다. 이대로 두면 브로커
자신은 잘 뜨지만, **외부에서 접속한 클라이언트가 브로커로부터 돌려받는 메타데이터에
"나에게 연결하려면 localhost로 오라"는 잘못된 주소가 담긴다.** Kafka 클라이언트는
먼저 아무 브로커에 접속해서 "이 토픽 담당은 어느 브로커냐"는 메타데이터를 받은 뒤,
그 응답에 적힌 주소로 실제 연결을 다시 맺는 방식으로 동작하기 때문이다. 그래서
`advertised.listeners`를 EC2의 퍼블릭 IP로 바꿔주지 않으면, 최초 연결은 되는데
그 다음 실제 프로듀스/컨슘 요청이 알 수 없는 이유로 계속 실패하는 상황을 겪게 된다 —
클라우드에서 Kafka를 처음 붙일 때 가장 흔한 삽질 포인트다.

## 문제 3: 이 구성은 근본적으로 이중화가 안 된다

```bash
export KAFKA_CLUSTER_ID=$(bin/kafka-storage.sh random-uuid)
bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/server.properties
bin/kafka-server-start.sh -daemon config/server.properties
```

`--standalone` 플래그가 이 실습의 본질을 보여준다 — 브로커가 딱 하나다. 브로커가
하나뿐이면 토픽의 복제 계수(replication factor)는 물리적으로 1일 수밖에 없고,
이 인스턴스가 죽으면 데이터는 그대로 사라진다. 프로덕션 Kafka 클러스터가 최소
3개 브로커로 구성되고 복제 계수 3을 기본값처럼 쓰는 이유가 바로 이 지점이다 —
브로커 한 대가 죽어도 나머지 두 대가 데이터를 들고 있어야 서비스가 안 끊긴다.
프리티어 한 대짜리 구성은 Kafka의 API와 운영 감각을 익히기엔 좋지만, "왜 Kafka
클러스터를 최소 3대로 구성하라고 하는지"는 이 구성만으로는 체감하기 어렵다 —
그 답은 정확히 이 실습에서 생략된 부분(다중 브로커 + 복제)에 있다.

## 정리

- 힙을 줄이고 스왑을 켜는 건 프리티어에서 Kafka를 "돌아가게" 만드는 임시방편이지,
  본받을 설정이 아니다.
- `advertised.listeners`를 실제 접근 가능한 주소로 맞추지 않으면 최초 연결 이후
  실제 데이터 송수신이 막힌다 — 클라우드 환경에서 Kafka 연결 문제의 상당수가
  여기서 시작된다.
- 단일 브로커 구성은 복제 계수 1을 강제한다는 뜻이고, 이는 곧 무장애 요구사항과
  양립할 수 없다 — 이 한계를 인지하고 학습용으로만 쓰는 게 중요하다.
