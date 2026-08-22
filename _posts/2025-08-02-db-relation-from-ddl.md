---
title: "DDL만 보고 테이블 관계(1:1, 1:N, N:M) 판단하는 법"
date: 2025-08-02
categories: [DB]
---

테이블 A의 기본키(PK)가 테이블 B의 외래키(FK)로 쓰이고 있을 때, DDL만 보고 관계를 판단하는 기준을 정리한다. 핵심은 **그 FK 컬럼에 UNIQUE 제약이 걸려 있는지**다.

## 관계는 "FK가 유일한가"로 결정된다

FK는 참조 무결성만 보장한다 — "이 값은 반드시 상대 테이블에 존재해야 한다"는 것뿐,
"몇 번 참조될 수 있는가"는 말해주지 않는다. 참조 횟수를 제한하는 건 UNIQUE(또는 PK) 제약이다.
그래서 관계의 카디널리티는 FK 자체가 아니라 **FK 위에 얹힌 제약 조건**이 결정한다.

## 1:N — FK에 UNIQUE가 없을 때

```sql
CREATE TABLE jwt_tokens (
  token_id CHAR(36) PRIMARY KEY,
  user_id  CHAR(36) NOT NULL,
  FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

`user_id`에 UNIQUE가 없다 → 같은 `user_id`를 가진 행이 여러 개 존재할 수 있다
→ 한 유저가 토큰을 여러 개 가질 수 있다는 뜻 → `users(1) : jwt_tokens(N)`.

이게 기본값이다. 별다른 제약을 안 걸면 FK는 자동으로 1:N을 의미한다.

## 1:1 — FK에 UNIQUE를 "추가로" 걸었을 때

```sql
CREATE TABLE user_details (
  user_id CHAR(36) NOT NULL,
  bio     TEXT,
  PRIMARY KEY (user_id),
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  UNIQUE (user_id)
);
```

1:1은 1:N에 제약을 하나 더 얹은 특수 케이스다. `user_id`가 PK이면서 FK이므로
자동으로 UNIQUE 성질을 갖고(PK 자체가 UNIQUE + NOT NULL이므로), 결과적으로
"한 유저당 정확히 하나의 행만" 존재할 수 있다.

**여기서 실무에서 자주 놓치는 포인트**: FK 컬럼을 PK로 쓰지 않고 별도 PK를 두면서
FK에 UNIQUE만 추가한 경우도 1:1이다. 즉 "PK=FK"는 1:1을 만드는 여러 방법 중 하나일 뿐,
필요조건이 아니다. 판별 기준은 항상 **UNIQUE 여부**이지 "PK와 FK가 같은 컬럼인가"가 아니다.

또 하나, `user_id`가 NULL을 허용하면 "선택적 1:1"(optional 1:1)이 되고 NOT NULL이면
"필수 1:1"(mandatory 1:1)이 된다. JPA로 옮기면 `@OneToOne` + `optional = false/true`
차이가 여기서 나온다 — DDL의 NULL 허용 여부를 안 보고 매핑하면 실제 제약과 엔티티 설계가
어긋난다.

## N:M — 중간 테이블의 PK가 두 FK의 "조합"일 때

```sql
CREATE TABLE user_roles (
  user_id CHAR(36),
  role_id CHAR(36),
  PRIMARY KEY (user_id, role_id),
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  FOREIGN KEY (role_id) REFERENCES roles(role_id)
);
```

각 FK 단독으로는 UNIQUE가 아니지만(user_id 하나로 여러 role_id 조합 가능), **두 FK를
합친 조합**이 PK(=UNIQUE)다. 즉 "같은 유저-역할 조합은 한 번만 존재"라는 제약만 걸려 있을 뿐,
`user_id` 기준으로도 `role_id` 기준으로도 여러 행이 나올 수 있다 → `users(N) : roles(N)`.

## 헷갈리기 쉬운 케이스: 복합 FK인데 N:M이 아닌 경우

중간 테이블이라고 다 N:M은 아니다. 예를 들어 이력(history) 테이블처럼
`(user_id, version)`을 복합 PK로 쓰는 테이블은 겉보기엔 비슷해 보여도, `version`이
다른 테이블을 참조하는 FK가 아니라면 N:M 관계가 아니라 그냥 **1:N을 시계열로 쪼갠 것**뿐이다.
"복합 PK = N:M"으로 기계적으로 외우면 이런 케이스에서 틀린다 — 복합 PK를 이루는 컬럼들이
**각각 서로 다른 테이블을 참조하는 FK인지**까지 확인해야 한다.

## 판단 순서 (체크리스트)

1. FK 컬럼(들)에 UNIQUE 또는 PK 제약이 있는가?
2. 있다면 → 그 제약이 컬럼 하나에 걸려 있는가, 여러 컬럼 조합에 걸려 있는가?
   - 컬럼 하나 → 1:1
   - 여러 컬럼의 조합이고, 그 컬럼들이 각각 다른 테이블을 참조하는 FK → N:M (중간 테이블)
3. 없다면 → 1:N

| 관계 | DDL 판별 키워드 |
|---|---|
| 1:1 | FK + UNIQUE (단일 컬럼) |
| 1:N | FK만 있고 UNIQUE 없음 |
| N:M | 중간 테이블의 복합 PK, 각 컬럼이 서로 다른 FK |
