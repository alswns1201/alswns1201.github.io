---
title: "DDL만 보고 테이블 관계(1:1, 1:N, N:M) 판단하는 법"
date: 2025-08-02
categories: [아키텍쳐 설계 관련 글]
---

*(원래 제목은 "테이블 설계의 기본"이었는데, 실제 내용에 맞춰 제목을 바꿨습니다 — 확인 부탁드려요)*

테이블 A의 기본키(PK)가 테이블 B의 외래키(FK)로 쓰이고 있을 때, DDL만 보고 관계를 판단하는 기준을 정리한다. 핵심은 **그 FK 컬럼에 UNIQUE 제약이 걸려 있는지**다.

## 1:N (One-to-Many) 판단 기준

B 테이블이 A 테이블의 PK를 FK로 가지고 있는데, **UNIQUE 제약이 없다면 1:N**이다.

```sql
CREATE TABLE jwt_tokens (
  token_id CHAR(36) PRIMARY KEY,
  user_id CHAR(36) NOT NULL,
  FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

`user_id`에 UNIQUE 제약이 없으므로 한 명의 유저가 여러 개의 토큰을 가질 수 있다 → `users(1) : jwt_tokens(N)`, 즉 1:N.

## 1:1 판단 기준

A 테이블의 PK가 B 테이블의 FK로 쓰이면서, 그 FK 컬럼에 **UNIQUE 제약까지 걸려 있으면 1:1**이다.

```sql
CREATE TABLE user_details (
  user_id CHAR(36) NOT NULL,
  ...,
  PRIMARY KEY (user_id),
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  UNIQUE (user_id)
);
```

`user_id`가 PK이자 동시에 `users` 테이블을 참조하는 FK이고 UNIQUE이므로, 한 유저당 정확히 하나의 `user_details` 행만 존재할 수 있다 → 1:1.

## N:M (Many-to-Many) 판단 기준

두 테이블의 PK를 조합한 **중간 테이블(조인 테이블)**이 있고, 그 조합이 **복합 PK(또는 UNIQUE)**로 설정되어 있으면 N:M이다.

```sql
CREATE TABLE user_roles (
  user_id CHAR(36),
  role_id CHAR(36),
  PRIMARY KEY (user_id, role_id),
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  FOREIGN KEY (role_id) REFERENCES roles(role_id)
);
```

`user_roles`라는 중간 테이블을 통해 `users(N) : roles(N)`, 즉 N:M 관계가 성립한다.

## 정리

| 관계 | DDL 판별 키워드 |
|---|---|
| 1:1 | FK + UNIQUE (또는 PK) |
| 1:N | FK (UNIQUE 없음) |
| N:M | 중간 테이블 + 복합 PK |
