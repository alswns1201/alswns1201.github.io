---
title: "하네스 엔지니어링이란? (개념부터 Claude Code 실전 구성까지)"
date: 2026-06-28
categories: [AI]
---

*(100번 "하네스 엔지니어링이란?"과 104번 "하네스 엔지니어링" 병합본. 날짜는 나중 글(104번, 2026-06-28) 기준)*

AI 에이전트 활용이 확산되면서 생긴 용어인 **하네스(Harness)**에 대해 정리해본다.

## 하네스란 무엇인가

하네스는 AI 에이전트를 안전하게 운용하기 위한 설계 사고방식이다. 배경은 AI 에이전트의 강력함에서 나온다 — 이 강력함이 예측 불가능한 방향으로 가지 않도록 잡아주는 역할을 한다. 여러 참고 자료에서는 이렇게 정의한다.

> 하네스는 AI 에이전트를 억누르는 것이 아니라, 안전한 방식으로 작동하도록 설계된 제어 구조 전체를 말한다.

핵심 기능은 세 가지로 요약된다.

- **제어**: 허용된 범위 내에서만 행동하게 한다
- **감시**: 동작의 상태와 출력을 실시간으로 기록한다
- **개선**: 오류나 이상 행동을 감지하고 피드백한다

AI는 데이터 유출, 품질 불균형, 책임 소재 불명확 같은 문제를 안고 있다. 하네스 엔지니어링은 이런 위험을 사전에 억제하는 접근 방식으로 자리잡고 있고, 이를 위해 등장하는 개념이 **가드레일**이다 — 기술적으로 제어하여 설계된 목적 범위 밖의 동작을 통제하는 것. 가드레일이 없으면 할루시네이션이 발생해 보안 사고로 직결될 수 있다.

하네스 엔지니어링은 사실 특별한 프레임워크가 아니라, AI와 협업하면서 자연스럽게 쌓이는 것에 가깝다. (하시코프 창업자) 미첼 해시모토도 처음부터 설계한 게 아니라 에이전트가 실수할 때마다 하나씩 규칙을 추가했다고 밝힌 바 있다.

최근엔 GitHub를 통해 공식화된 구조도 많다. 대표적인 게 [harness-100](https://github.com/revfactory/harness-100/tree/main/ko) — 100가지 하네스 구성 예시를 모아둔 저장소다.

## 하네스의 기본 구성 요소

프로젝트 루트 기준으로 아래 세 가지가 기본 골격이다.

- `AGENTS.md` (또는 `CLAUDE.md`) — AI가 지켜야 할 내용
- `tests/` — 결과가 맞는지 즉시 확인할 피드백 장치
- `docs/` — 행위했던 내용에 대한 기록

**1. 지시 문서 (AGENTS.md / CLAUDE.md)**

좋은 지시 문서의 조건: 목차형으로 구성, 구체적인 규칙 명시, 금지사항 명시, 짧고 명확한 내용. 개인적으로는 프로젝트 전체를 하나로 관리하기보다 폴더·기능 단위로 `.md`를 쪼개는 걸 선호한다.

**2. `tests/` — AI가 스스로 검증할 수 있는 센서**

테스트 케이스를 넣어두고, AI가 직접 테스트를 통과하는지 확인하면서 성공할 때까지 수정하도록 돕는다.

**3. 행위 기록**

보통 결정 기록(ADR)이라 부른다. 에이전트가 잘못된 방향을 제안하거나 오류를 냈을 때 여기에 기록하고 `AGENTS.md`에 연결해두면, 같은 실수가 반복되는 걸 막을 수 있다.

## 실전: Claude Code 프로젝트에 하네스 구성하기

요즘 Claude Code나 Cursor 같은 AI 에이전트를 효과적으로 쓰려면, `.md` 파일로 AI에게 컨텍스트를 주입하고 역할을 분리하는 방식이 쓰인다. AI가 프로젝트 구조와 규칙을 이해하고 일관된 코드를 생성하도록 가이드 문서를 체계적으로 구성하는 것.

### 패키지 구성 예시

```
my-project/
│
├── .claude/                          # Claude Code 전용 설정
│   ├── CLAUDE.md                     # 프로젝트 전체 컨텍스트 (메인 진입점)
│   └── commands/                     # 커스텀 슬래시 커맨드
│       ├── review.md                 # /review 명령어 → 코드 리뷰 수행
│       ├── test.md                   # /test 명령어 → 테스트 코드 생성
│       └── refactor.md               # /refactor 명령어 → 리팩토링 수행
│
├── docs/
│   └── ai/                           # AI 에이전트용 문서
│       ├── architecture.md           # 아키텍처 설명
│       ├── coding-convention.md      # 코딩 컨벤션
│       ├── domain-glossary.md        # 도메인 용어 정의
│       └── agents/                   # 에이전트 역할 분리
│           ├── code-reviewer.md      # 코드 리뷰 에이전트
│           ├── test-generator.md     # 테스트 생성 에이전트
│           └── query-builder.md      # QueryDSL 쿼리 생성 에이전트
│
├── src/
│   └── main/java/com/example/
│       ├── domain/                   # 도메인 레이어
│       │   ├── order/
│       │   │   ├── Order.java
│       │   │   ├── OrderRepository.java
│       │   │   └── OrderService.java
│       │   └── member/
│       │       ├── Member.java
│       │       ├── MemberRepository.java
│       │       └── MemberService.java
│       ├── api/                      # 컨트롤러 레이어
│       │   ├── order/
│       │   │   └── OrderController.java
│       │   └── member/
│       │       └── MemberController.java
│       ├── infra/                    # 인프라 레이어
│       │   ├── redis/
│       │   ├── kafka/
│       │   └── elasticsearch/
│       └── global/                   # 공통
│           ├── config/
│           ├── exception/
│           └── filter/
│
└── prompts/                          # 재사용 프롬프트 모음
    ├── code-review.txt
    ├── test-generation.txt
    └── query-generation.txt
```

### CLAUDE.md 구성 예시

```markdown
# 프로젝트 컨텍스트

## 기술 스택
- Java 17, Spring Boot 3.x
- JPA / QueryDSL
- Redis, Kafka, Elasticsearch
- MariaDB

## 아키텍처 원칙
- 레이어 구조: api → domain → infra
- 도메인 레이어는 인프라에 의존하지 않는다
- 엔티티에 @Setter 사용 금지, 비즈니스 메서드로만 상태 변경

## 코딩 컨벤션
- 메서드명은 동사로 시작 (find, create, update, delete)
- Repository는 인터페이스로 정의
- 예외는 GlobalExceptionHandler에서 처리

## 참고 문서
- 아키텍처: docs/ai/architecture.md
- 코딩 컨벤션: docs/ai/coding-convention.md
- 도메인 용어: docs/ai/domain-glossary.md
```

### 에이전트 역할 분리

**code-reviewer.md** (코드 리뷰 전용 에이전트)

```markdown
# 코드 리뷰 에이전트

## 역할
코드 리뷰 시 아래 항목을 순서대로 체크한다

## 체크리스트
1. N+1 문제 발생 여부
2. 트랜잭션 범위가 적절한지
3. 예외 처리가 누락된 곳은 없는지
4. 엔티티에 @Setter가 사용됐는지
5. 인덱스 활용 여부

## 출력 형식
- [위험] 즉시 수정 필요
- [경고] 개선 권장
- [제안] 선택적 개선
```

**query-builder.md** (QueryDSL 쿼리 생성 전용)

```markdown
# QueryDSL 쿼리 생성 에이전트

## 역할
QueryDSL 기반 동적 쿼리를 생성한다

## 규칙
- BooleanBuilder 사용
- Fetch Join 시 distinct 적용
- Pagination은 커버링 인덱스 서브쿼리 방식 사용
- 조건이 null이면 where절에서 제외

## 출력 형식
Repository 인터페이스 + Impl 구현체 세트로 생성
```

### 실제 구성하고 사용하는 순서

**1. 클로드 코드 설치 및 실행** — `claude init` 같은 명령어는 없다. 그냥 `claude`를 실행하면 바로 대화가 시작된다.

```bash
# 설치
npm install -g @anthropic-ai/claude-code

# 프로젝트 루트에서 실행
cd my-project
claude
```

**2. CLAUDE.md는 직접 생성해서 작성** (`init`으로 자동 생성되는 `.md`는 지양)

```bash
touch CLAUDE.md
```

**3. commands 폴더도 직접 생성** — `.claude/commands/` 안에 `.md` 파일을 만들면 자동으로 슬래시 커맨드로 등록된다.

```bash
mkdir -p .claude/commands
touch .claude/commands/review.md
touch .claude/commands/test.md
```

사용 예시:

```
.claude/commands/review.md  →  /review 로 사용 가능
.claude/commands/test.md    →  /test 로 사용 가능
```

**4. 실제 사용 예시**

`CLAUDE.md`:

```markdown
# 프로젝트 컨텍스트

## 기술 스택
- Java 17, Spring Boot 3.x
- JPA, QueryDSL, MariaDB
- Redis, Kafka

## 아키텍처
- 레이어: api → domain → infra
- 엔티티 @Setter 금지
- 예외는 GlobalExceptionHandler 에서 처리

## 네이밍
- 조회: find
- 생성: create
- 수정: update
- 삭제: delete
```

`.claude/commands/review.md`:

```markdown
# 코드 리뷰

$ARGUMENTS 로 전달된 파일 또는 현재 컨텍스트를 아래 기준으로 리뷰해줘

## 체크 항목
1. N+1 문제 여부
2. 트랜잭션 범위 적절한지
3. @Setter 사용 여부
4. 예외 처리 누락 여부
5. 인덱스 활용 여부

## 출력 형식
- [위험] 즉시 수정
- [경고] 개선 권장
- [제안] 선택적 개선
```

`.claude/commands/test.md`:

```markdown
# 테스트 코드 생성

$ARGUMENTS 로 전달된 클래스의 단위 테스트를 생성해줘

## 규칙
- JUnit5, Mockito 사용
- given/when/then 형식
- 성공 케이스 + 실패 케이스 모두 작성
- Mock은 @ExtendWith(MockitoExtension.class) 사용
```

**5. 실제 사용**

```bash
# Claude Code 실행
claude

# 슬래시 커맨드 사용
/review OrderService.java
/test MemberService.java

# $ARGUMENTS 에 파일명이 들어가는 구조
```

## 요약

- `claude` 설치: `npm install -g @anthropic-ai/claude-code`
- 프로젝트 루트에 `CLAUDE.md` 직접 생성 → 기술 스택, 아키텍처 원칙, 코딩 컨벤션 작성
- `.claude/commands/` 폴더 직접 생성 → `review.md`, `test.md` 등 원하는 커맨드 파일 작성
- `claude` 실행 → `CLAUDE.md` 자동 인식, `/review` `/test` 슬래시 커맨드 자동 등록
- 대화하면서 점진적으로 `CLAUDE.md` 내용 보완

실제로 하네스를 구성하고 `build a harness for this project`를 실행해보면, 절반 이상은 하네스를 구성했다고 볼 수 있다.
