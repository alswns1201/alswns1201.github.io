---
title: "MariaDB와 Elasticsearch를 다른 서버로 옮기기: 이관 절차와 그 이유"
date: 2025-07-05
categories: [DB]
---

서버를 옮길 때 "덤프 뜨고 복원하면 되지"로 끝나지 않는 이유는, 무엇을 몇 개로 나눠서
어떤 순서로 옮기느냐가 복원 성공률과 다운타임을 결정하기 때문이다. MariaDB와
Elasticsearch를 A 서버에서 B 서버로 옮긴 절차를 정리하면서, 각 단계가 왜 그렇게
설계됐는지를 함께 짚는다.

## MariaDB: 왜 덤프를 3개로 쪼개는가

```bash
# 구조(DDL)만
mysqldump -u root -pP@ssw0rd --no-data --databases my_service > schema.sql
# 데이터(INSERT)만
mysqldump -u root -pP@ssw0rd --no-create-info --databases my_service > data.sql
# 프로시저 + 트리거만
mysqldump -u root -pP@ssw0rd --no-data --routines --triggers --databases my_service > routine.sql
```

한 방에 통짜 덤프를 뜰 수도 있는데 굳이 나누는 이유는 **복원 시 실패 지점을 좁히기 위해서**다.
통짜 덤프가 중간에 실패하면 스키마 문제인지 데이터 문제인지 트리거 문제인지 로그를 다시
헤집어야 한다. 나눠두면 복원도 단계별로 되고, 실패해도 어느 단계인지 바로 안다.

복원은 반드시 **스키마 → 데이터 → 프로시저** 순서다. 데이터를 스키마보다 먼저 넣으려 하면
테이블이 없어서 실패하고, 프로시저가 특정 테이블/컬럼을 참조한다면 데이터까지 다 들어온
뒤에 만드는 게 안전하다.

```bash
docker exec -i mariadb mysql -u root -pP@ssw0rd < ./schema.sql
docker exec -i mariadb mysql -u root -pP@ssw0rd < ./data.sql
docker exec -i mariadb mysql -u root -pP@ssw0rd < ./routine.sql
```

`docker exec -i`로 파이프 처리하는 이유는, 컨테이너 내부로 파일을 굳이 복사해 넣지 않고
호스트의 스트림을 그대로 컨테이너 프로세스의 표준입력으로 흘려보낼 수 있어서다 — 중간
파일 복사 단계가 없으니 그만큼 실패 지점도 줄어든다.

## Elasticsearch: 전체가 아니라 특정 인덱스만 스냅샷

```
PUT _snapshot/my_backup
{ "type": "fs", "settings": { "location": "/usr/share/elasticsearch/backup" } }

PUT _snapshot/my_backup/snapshot_202507
{ "indices": "my_index_202507", "ignore_unavailable": true, "include_global_state": false }
```

`include_global_state: false`가 핵심이다 — 이걸 켜두면 클러스터 설정 자체(다른 인덱스,
템플릿, 보안 설정 등)까지 스냅샷에 묶여서, 복원할 때 B 서버의 기존 클러스터 설정을
덮어써버릴 위험이 생긴다. 특정 인덱스만 옮기는 상황에서는 반드시 꺼야 한다.

복원 절차는 리포지토리를 B 서버에 다시 등록하고, `_restore`를 호출하는 순서다.

```
POST _snapshot/my_backup/snapshot_202507/_restore
```

**여기서 실무적으로 자주 놓치는 지점**: `_restore`는 인덱스의 **데이터와 매핑**은 복원하지만,
인덱스 생성 시점에만 지정 가능한 설정(예: `number_of_shards`)은 원본 스냅샷 시점 값 그대로
복원된다 — 즉 B 서버의 클러스터 규모에 맞게 샤드 수를 바꾸고 싶다면 복원 후가 아니라
스냅샷을 다시 만들 때(reindex 등으로) 조정해야 한다. 그냥 복원만 하고 끝내면 예전 서버
기준의 샤드 설계를 그대로 물려받는다.

## 볼륨 마운트: 컨테이너를 지워도 데이터가 남는 이유

| 컨테이너 | 호스트 디렉토리 | 컨테이너 내부 | 역할 |
|---|---|---|---|
| MariaDB | `./db_data` | `/var/lib/mysql` | 실제 DB 파일 |
| Elasticsearch | `./es_backup` | `/usr/share/elasticsearch/backup` | 스냅샷 저장 위치 |

볼륨을 호스트에 마운트해두는 이유는 단순하다 — 컨테이너는 언제든 재생성될 수 있어야 하는
휘발성 자원이고, 데이터는 그래선 안 되는 영속 자원이기 때문이다. 이 둘의 생명주기를
분리해두지 않으면 컨테이너를 재시작하거나 이미지 버전을 올릴 때마다 데이터 유실 위험을
감수해야 한다.

## 이 방식의 한계와 대안

이 절차는 **다운타임을 전제로 한 일괄 이관**이다. 덤프를 뜨는 순간부터 복원이 끝날 때까지
A 서버의 데이터 변경분은 반영되지 않는다. 트래픽이 있는 서비스를 무중단으로 옮겨야 한다면
이 방식만으로는 부족하고, CDC(Change Data Capture, 예: Debezium + Kafka)로 실시간 변경분을
따로 흘려보내면서 dual-write 기간을 두는 전략이 필요하다. 이 글의 절차는 "정지 시간을
감수할 수 있는 규모"에서의 실용적인 접근이라는 전제를 깔고 있다는 걸 명확히 해두는 게 좋다.

## 정리

| 항목 | A 서버 | B 서버 |
|---|---|---|
| MariaDB | 구조/데이터/프로시저 3분할 덤프 | 컨테이너 기동 후 파이프 방식 순차 복원 |
| Elasticsearch | 특정 인덱스만 스냅샷 생성 | 스냅샷 디렉토리 복사 후 리스토어 |
| 공통 | `scp`, Docker Compose, 볼륨 마운트로 데이터 영속성 확보 | |

실무 팁: Elasticsearch는 스냅샷을 만든 버전과 복원할 버전 간 호환성을 반드시 먼저 확인해야
한다 — 메이저 버전이 다르면 스냅샷 자체가 호환되지 않을 수 있다.
