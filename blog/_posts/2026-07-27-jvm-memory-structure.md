---
title: "JVM 메모리 구조: StackOverflowError와 OutOfMemoryError를 이해하는 법"
date: 2026-07-27
categories: [Java/Spring]
---

JVM 메모리 구조를 외우는 건 큰 의미가 없다. 실제로 필요한 건 "이 에러가 왜, 어느
영역에서 났는가"를 구조로부터 추론하는 능력이다. 그래서 이 글은 영역 나열보다
**각 영역이 어떤 에러의 원인이 되는지**를 중심으로 정리한다.

## 공유 영역과 스레드별 영역

JVM 메모리는 크게 모든 스레드가 공유하는 영역과, 스레드마다 별도로 생성되는
영역으로 나뉜다.

**공유 영역**
- **Method Area (Metaspace)**: 클래스 메타데이터(필드, 메서드 정보, 런타임 상수
  풀)와 static 변수를 저장한다. Java 8부터 PermGen 대신 네이티브 메모리 영역인
  Metaspace가 이 역할을 담당한다.
- **Heap**: `new`로 생성된 객체 인스턴스가 저장되는 공간. GC의 주요 대상이며,
  내부적으로 Young Generation(Eden, Survivor 0/1)과 Old Generation으로 나뉜다.
  새 객체는 Eden에 할당되고, 여러 번의 GC에서 살아남은 객체만 Old Generation으로
  이동한다.

**스레드별 영역**
- **Stack**: 메서드가 호출될 때마다 Stack Frame이 push되고, 메서드가 끝나면
  pop된다. 각 프레임에는 지역 변수, 매개변수, 연산 중간값(오퍼랜드 스택)이
  담긴다. 스레드가 종료되면 이 영역도 함께 사라진다.
- **PC Register**: 현재 스레드가 실행 중인 JVM 명령의 주소를 가리킨다. 멀티스레드
  환경에서 컨텍스트 스위칭이 일어날 때 각 스레드가 자신이 어디까지 실행했는지
  기억해야 하므로 스레드마다 별도로 필요하다.
- **Native Method Stack**: Java 코드가 아닌 네이티브 코드(JNI 등)를 호출할 때
  쓰는 스택.

## 왜 PermGen이 Metaspace로 바뀌었는가

Java 7까지는 클래스 메타데이터가 Heap의 일부인 PermGen에 있었고, 이 영역은
`-XX:MaxPermSize`로 크기가 고정돼 있었다. 동적으로 클래스를 많이 생성하는
프레임워크(리플렉션 기반 프록시, 스크립트 엔진 등)를 쓰면 PermGen이 가득 차서
`OutOfMemoryError: PermGen space`가 자주 발생했고, 이걸 피하려면 애플리케이션을
재시작하거나 `-XX:MaxPermSize`를 계속 늘려야 했다. Metaspace는 네이티브 메모리를
쓰면서 기본적으로 OS가 허용하는 한도까지 자동으로 늘어나도록 바뀌어 이 문제를
구조적으로 완화했다 — 다만 무한정 늘어나는 건 아니고, 여전히 클래스로더 누수가
있으면 `OutOfMemoryError: Metaspace`로 형태만 바뀌어 재발한다.

## StackOverflowError: 언제, 왜 나는가

Stack 영역에서 발생한다. 스레드마다 할당된 Stack 크기(기본 512KB~1MB, `-Xss`로
조정 가능)를 초과해 Stack Frame이 쌓일 때 발생한다.

```
Exception in thread "main" java.lang.StackOverflowError
    at com.example.Recursive.call(Recursive.java:5)
    at com.example.Recursive.call(Recursive.java:5)
    at com.example.Recursive.call(Recursive.java:5)
    ...
```

가장 흔한 원인은 종료 조건이 없거나 잘못된 재귀 호출이다. 스택 트레이스에 동일한
메서드 호출이 반복되는 패턴이 보이면 재귀 종료 조건을 먼저 의심하면 된다.

`-Xss` 값을 늘리면 임시로 완화되지만, 이건 증상을 늦추는 것이지 원인을 없애는 게
아니다 — 스택이 커진 만큼 스레드 하나가 차지하는 메모리도 커지고, 스레드를 많이
띄우는 서버라면 오히려 전체 메모리 압박이 커질 수 있다. 근본 해법은 재귀 깊이를
줄이거나 반복문(또는 꼬리 재귀 최적화가 되는 언어라면 그 형태)으로 바꾸는 것이다.

## OutOfMemoryError: 발생 위치로 원인을 좁히기

Heap이나 Method Area 같은 공유 영역에서 더 이상 메모리를 할당할 공간이 없을 때
발생한다. 메시지가 발생 위치에 따라 다르게 나온다는 점이 디버깅에 실질적으로
유용하다.

- **`java.lang.OutOfMemoryError: Java heap space`**: Heap이 가득 찼는데 GC를
  수행해도 공간을 확보하지 못한 경우. 객체를 계속 참조하고 있어 GC 대상이 되지
  못하는 **메모리 누수**이거나, 단순히 힙 크기(`-Xmx`)가 애플리케이션이 필요로
  하는 양보다 작을 때 나타난다. 이 둘을 구분하려면 힙 덤프를 떠서 어떤 객체가
  비정상적으로 많이 살아있는지 확인하는 게 먼저다 — `-Xmx`를 무작정 늘리는 건
  누수가 원인일 때는 문제를 더 늦게 터지게 할 뿐이다.
- **`java.lang.OutOfMemoryError: Metaspace`**: 클래스 메타데이터 저장 공간이
  가득 찬 경우. 동적으로 클래스를 계속 생성하는 프레임워크를 잘못 쓰거나
  클래스로더 누수가 있을 때 자주 나타난다.
- **`java.lang.OutOfMemoryError: GC overhead limit exceeded`**: GC가 너무 자주
  실행되는데도 회수되는 메모리가 거의 없을 때(기준: GC에 98% 이상의 시간을
  쓰는데 회수되는 힙이 2% 미만) 발생한다. Heap space 에러의 전조 증상에 가깝다 —
  이 에러가 보이면 곧 완전한 Heap 고갈이 온다는 신호로 봐야 한다.

## 정리: 에러 메시지가 곧 진단 지점이다

JVM 메모리 구조를 알아야 하는 실질적인 이유는, 에러 메시지가 알려주는 영역
이름(Heap, Metaspace, Stack)만 보고도 원인을 좁힐 수 있기 때문이다. `StackOverflowError`는
거의 항상 재귀 문제이고, `OutOfMemoryError`는 뒤에 붙은 문구(heap space /
Metaspace / GC overhead limit)가 조사 방향을 곧바로 알려준다 — 이 매핑을 알고
있으면 에러 로그 한 줄만 보고도 다음에 뭘 확인해야 할지 바로 판단할 수 있다.
