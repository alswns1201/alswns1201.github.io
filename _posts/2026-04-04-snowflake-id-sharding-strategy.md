---
title: "샤딩 환경의 PK 전략: Shard Key 선택과 Snowflake ID"
date: 2026-04-04
categories: [대용량 고민]
---

대규모 서비스에서는 단순 `AUTO_INCREMENT` PK 전략이 한계에 부딪히는 지점이 온다.
특히 데이터가 여러 DB로 분산되는 샤딩 환경에서는 **ID 생성 전략**과 **Shard Key
선택**이 서로 얽힌 문제라서, 둘을 따로 생각하면 안 된다.

## 예제 테이블

```sql
create table article (
    article_id bigint not null primary key,
    title varchar(100) not null,
    content varchar(3000) not null,
    board_id bigint not null,
    writer_id bigint not null,
    created_at datetime not null,
    modified_at datetime not null
);
```

여기서 중요한 컬럼은 `article_id`(PK)와 `board_id`(Shard Key 후보) 둘이다.

## Shard Key를 article_id가 아니라 board_id로 선택하는 이유

Shard Key 선택은 **데이터 접근 패턴**을 먼저 봐야 한다. 게시판 서비스의 대부분
조회는 이런 형태다.

```sql
select * from article
where board_id = ?
order by article_id desc
limit 20;
```

트래픽 대부분이 `board_id` 기준으로 들어온다는 뜻이다.

**article_id로 샤딩하면** 데이터는 이렇게 흩어진다.

```
Shard1: article_id 1~1000
Shard2: article_id 1001~2000
Shard3: article_id 2001~3000
```

특정 게시판의 글 목록을 조회하려면 어느 shard에 흩어져 있는지 알 수 없으니 **모든
shard를 조회하고 결과를 merge**해야 한다 — 샤딩을 한 의미가 없어질 정도로 비효율적이다.

**board_id로 샤딩하면** 다음과 같이 분산된다.

```
Shard1: board_id 1~100
Shard2: board_id 101~200
Shard3: board_id 201~300
```

`board_id = 10`으로 조회하면 Shard1 하나만 보면 된다 — 조회 패턴과 샤딩 기준이
일치해야 샤딩의 이득이 실제로 발생한다는 원칙이 여기서 드러난다.

**놓치기 쉬운 반례**: 이 선택에도 대가가 있다. 인기 게시판 하나에 트래픽이 몰리면
그 board_id가 속한 shard만 핫스팟이 된다 — board_id 기준 샤딩은 "조회는 빠르지만
샤드 간 부하가 균등하지 않을 수 있다"는 트레이드오프를 함께 가져온다. 게시판별
글 수/트래픽 편차가 크다면 이 불균형을 어떻게 완화할지(핫 샤드 분리, 추가 리밸런싱)
는 별도로 설계해야 한다.

## PK 후보 세 가지: AUTO_INCREMENT, UUID, Snowflake

**AUTO_INCREMENT**는 단일 DB에서는 편리하지만, 분산 환경에서 각 DB가 독립적으로
증가시키면 `DB1 → id=1`, `DB2 → id=1`처럼 충돌이 발생한다. 이걸 막으려면 중앙
ID 생성 서버가 필요하고, 그 서버가 곧 병목이자 단일 장애점이 된다.

**UUID**는 충돌 걱정이 없지만 DB 인덱스에는 비용을 지불한다. MySQL InnoDB는
B-Tree 구조라 PK를 기준으로 데이터가 물리적으로 정렬된다. UUID는 무작위 값이라
`AUTO_INCREMENT`처럼 항상 끝에 삽입되지 않고 중간 위치에 삽입되며, 이게 반복되면
**Page Split**(가득 찬 B-Tree 페이지가 둘로 쪼개지는 현상)이 빈번해져 디스크 I/O와
인덱스 단편화가 늘어난다.

**Snowflake**는 이 둘의 장점을 절충한다. 구조는 `timestamp + nodeId + sequence`
조합이다.

```
현재 시간 확인
  ↓
같은 ms면 sequence 증가
  ↓
sequence overflow → 다음 ms까지 대기
  ↓
timestamp + nodeId + sequence 합치기
```

값이 시간 순으로 대체로 증가하기 때문에 `AUTO_INCREMENT`와 마찬가지로 항상 뒤쪽에
삽입되어 Page Split이 거의 없다. 동시에 `nodeId`가 서버별로 다르게 부여되므로
분산 환경에서 충돌 없이 각자 ID를 발급할 수 있다 — 중앙 조율 서버 없이도 전역
유일성을 확보하는 방식이다.

## Snowflake를 실무에 쓸 때 챙겨야 할 것

- **nodeId 할당**: nodeId가 겹치면 서로 다른 노드가 같은 ID를 만들 수 있다. 그래서
  라이브러리를 그대로 쓰기보다, 배포 환경(pod ID, 인스턴스 번호 등)에서 nodeId를
  안전하게 할당하는 로직을 자체적으로 구현하는 경우가 많다.
- **클럭 스큐(clock skew)**: Snowflake는 시스템 시간에 의존한다. 서버 시간이
  거꾸로 흐르는 상황(NTP 보정 등)이 생기면 같은 ms에 같은 sequence 값이 재사용되어
  ID가 충돌할 수 있다. 이를 막으려면 "마지막으로 생성한 timestamp보다 현재 시간이
  과거라면 예외를 던지거나 대기한다"는 방어 로직이 필요하다.
- **sequence overflow**: 같은 ms 안에서 발급 가능한 sequence 수는 비트 수로
  제한된다(보통 12비트 = 4096개/ms). 이를 넘으면 다음 ms까지 대기하는데, 극단적으로
  트래픽이 몰리는 구간에서는 이 대기가 지연으로 체감될 수 있다.

## UUID vs Sequential ID 논의와의 연결

이 선택은 [UUID vs Sequential ID]({{ "/posts/uuid-vs-sequential-id/" | relative_url }})
에서 다룬 "무작위 분산과 정렬 가능성 사이의 스펙트럼" 논의와 정확히 같은 축 위에
있다. Snowflake는 그 글에서 언급한 UUIDv7/ULID와 같은 계열의 해법이다 — 전역
유일성(분산 환경 요구사항)과 삽입 성능(정렬 가능성)을 동시에 만족시키려는 절충안.
차이는 UUIDv7이 범용 UUID 포맷을 유지하는 반면, Snowflake는 필요에 따라 timestamp/
nodeId/sequence의 비트 폭을 직접 설계할 수 있다는 점이다 — 그만큼 유연하지만,
그 유연성만큼 nodeId 할당과 클럭 스큐 방어를 직접 구현해야 하는 책임도 따라온다.
