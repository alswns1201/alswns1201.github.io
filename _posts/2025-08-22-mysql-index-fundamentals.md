---
title: "데이터베이스 인덱스, 제대로 알고 썼더니 빛이 났다 (Feat. MySQL users 테이블)"
date: 2025-08-22
categories: ["아키텍쳐 설계 관련 글"]
tags: [database, mysql, index]
---

"인덱스를 걸면 쿼리가 빨라진다"는 말은 절반만 맞다. 어떤 컬럼에, 어떤 순서로, 언제
인덱스를 걸어야 하는지를 실제 `users` 테이블로 살펴본다.

## 인덱스란 무엇인가

인덱스는 책의 "찾아보기"와 같다. 데이터베이스는 특정 컬럼 값을 미리 정렬해둔
별도 구조를 만들어서, `WHERE 컬럼 = '값'` 쿼리가 들어오면 테이블 전체를 스캔하지
않고 이 구조를 참조해 필요한 데이터만 빠르게 찾는다.

## 실제 테이블로 보기

```sql
CREATE TABLE `users` (
  `user_id` CHAR(36) NOT NULL,
  `login_id` VARCHAR(255) NOT NULL,
  `status` VARCHAR(20) NOT NULL,
  `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `last_login_at` TIMESTAMP NULL,
  PRIMARY KEY (`user_id`),
  UNIQUE KEY `uq_users_login_id` (`login_id`),
  KEY `idx_users_status` (`status`),
  KEY `idx_users_last_login` (`last_login_at`)
) ENGINE=InnoDB;
```

- **PRIMARY KEY (user_id)**: 자동으로 인덱스가 생성되며, 단일 사용자 조회에 가장
  빠르다.
- **UNIQUE KEY (login_id)**: 로그인 시 가장 먼저 실행되는 쿼리라 중요도가 높다.

이 두 개는 데이터베이스 설계에서 거의 필수다. 문제는 그다음, 카디널리티가 낮은
컬럼에 건 일반 인덱스다.

## 인덱스가 독이 되는 지점: 카디널리티

**카디널리티(Cardinality)**는 컬럼에 저장된 고유한 값의 개수다. `user_id`처럼
거의 모든 값이 고유하면 높은 카디널리티, `status`처럼 `active`/`locked`/`withdrawn`
몇 종류밖에 없으면 낮은 카디널리티다.

100만 명 중 95만 명이 `active`, 3만 명이 `locked`, 2만 명이 `withdrawn`이라고
하자.

- `WHERE status = 'active'` → 전체의 95%를 찾아야 한다. 옵티마이저는 "인덱스로
  95만 개를 찾아 95만 번 데이터 파일에 접근"하는 것보다 "테이블 전체를 순차
  스캔"하는 게 더 빠르다고 판단할 수 있다. 인덱스 스캔은 **랜덤 I/O**, 풀 스캔은
  **순차 I/O**이기 때문이다.
- `WHERE status = 'locked'` → 전체의 3%. 이 경우엔 인덱스가 정확히 제 역할을 한다.

즉 "낮은 카디널리티 = 무조건 나쁜 인덱스"가 아니라, **그 인덱스로 걸러지는 비율이
얼마나 되는지**가 기준이다. 참고로 이 판단은 MySQL 옵티마이저가 **테이블 통계
정보**를 기반으로 내리는데, 이 통계는 실시간이 아니라 주기적으로(또는
`ANALYZE TABLE`로 수동) 갱신된다. 그래서 대량 삭제나 급격한 데이터 분포 변화
직후에는 통계가 실제 분포와 어긋나 있어서, 분명 적절한 인덱스가 있는데도
옵티마이저가 풀 스캔을 선택하는 경우가 생긴다 — 이럴 때 `EXPLAIN`으로 확인해도
"왜 인덱스를 안 타지?"가 바로 안 풀리는 이유가 여기에 있다.

## 복합 인덱스: 컬럼 순서가 곧 인덱스다

"활동 중인 사용자 중 최근 가입한 10명"을 뽑는 쿼리:

```sql
SELECT user_id, login_id, created_at
FROM users
WHERE status = 'active'
ORDER BY created_at DESC
LIMIT 10;
```

이 쿼리는 `status`와 `created_at` 두 컬럼을 쓴다. 복합 인덱스를 걸면:

```sql
KEY idx_users_status_created_at (status, created_at)
```

동작 순서: (1) `status = 'active'`인 레코드 그룹을 인덱스로 빠르게 찾고, (2) 그
그룹 내에서 `created_at`이 이미 정렬돼 있으므로 별도 정렬 없이 상위 10개를 바로
가져온다.

**컬럼 순서가 다르면 다른 인덱스다.** `(status, created_at)`과
`(created_at, status)`는 완전히 별개로 취급된다. 이걸 **최좌측 접두사 규칙
(leftmost prefix rule)**이라고 부른다 — 복합 인덱스는 왼쪽부터 순서대로 매칭되는
조건에만 효율적으로 쓰인다. `WHERE created_at = '...'`만 있는 쿼리는
`(status, created_at)` 인덱스를 제대로 활용하지 못한다(`status` 조건이 없으므로
인덱스의 첫 컬럼부터 시작할 수 없다). 일반적으로 `=` 조건으로 쓰이거나 범위가
좁은(카디널리티 높은) 컬럼을 앞에 두는 게 정석이지만, 실제 쿼리 패턴을 봐야
정확히 판단할 수 있다.

## 인덱스는 공짜가 아니다

- **쓰기 성능 저하**: INSERT/UPDATE/DELETE마다 인덱스도 함께 갱신해야 한다.
  인덱스가 많을수록 쓰기 비용이 누적된다.
- **저장 공간**: 인덱스도 물리적 공간을 차지한다.
- **옵티마이저 부담**: 인덱스 후보가 많을수록 실행 계획을 고르는 데 드는 비용도
  늘어난다.

## 정리 체크리스트

1. PK, UNIQUE는 사실상 필수.
2. 조회(SELECT) 패턴을 먼저 분석한다 — WHERE, ORDER BY, JOIN에 자주 쓰이는 컬럼.
3. 카디널리티를 보되, "낮으면 무조건 나쁨"이 아니라 실제 필터링 비율로 판단한다.
4. 복합 인덱스는 컬럼 순서(최좌측 접두사)에 따라 완전히 다른 인덱스가 된다는 걸
   전제로 설계한다.
5. `EXPLAIN`으로 실제 실행 계획을 확인하고, 통계가 오래됐다면 `ANALYZE TABLE`도
   의심해본다.
6. 필요한 곳에만 건다 — 과도한 인덱스는 쓰기 성능과 저장 공간을 갉아먹는다.
