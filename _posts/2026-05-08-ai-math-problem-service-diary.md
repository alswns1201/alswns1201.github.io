---
title: "AI로 수학 기출문제 변형 서비스 만들기 — 1~3주차 개발일지"
date: 2026-05-08
categories: [AI]
tags: [ai, rag, kafka]
---

RAG와 LLM을 실제로 손에 익히려고 사이드 프로젝트를 하나 시작했다. 주제는 수학
기출문제 — PDF로 된 기출문제를 넣으면 벡터로 인덱싱해서 분석하고, 그 문제를
변형한 유사 문제를 만들어주는 서비스다. 문제를 관리자가 직접 PDF로 올리는
구조로 잡은 이유도 있다 — LLM이 문제를 스스로 창작하게 하면 법적/정책적으로
애매해지지만, 선생님이나 관리자가 만든 문제를 PDF로 올리는 형태면 그 문제가
없다.

## 아키텍처

```
BackEnd   - Spring Boot 3.x
MQ        - Apache Kafka (KRaft 모드, Zookeeper 없이)
Main DB   - MariaDB
Vector DB - Chroma
LLM       - Claude API (claude-haiku-4-5)
Front     - React
인프라     - Docker Compose
```

핵심 파이프라인은 네 단계다.

1. **PDF 처리**: pymupdf로 텍스트/수식을 추출하고 문제 단위로 청킹한다. 수학
   기출은 수식이 이미지로 박혀 있는 경우가 많아서, LaTeX로 변환할지 이미지
   그대로 LLM에 넘길지가 첫 번째 갈림길이었다.
2. **RAG**: 문제를 임베딩해서 Chroma(로컬)/Pinecone(배포용)에 저장한다.
   단원·난이도·연도를 메타데이터로 같이 저장해두면 "미적분 3점짜리만" 같은
   필터링이 가능해진다.
3. **변형 문제 생성**: 원본 문제 + 유사 문제들을 컨텍스트로 Claude에 넘기고
   "숫자만 바꾸지 말고 풀이 구조를 바꿔라"고 지시한다.
4. **학습 데이터 축적**: 정답률·소요 시간·오답 패턴을 쌓아서 "이 학생은
   삼각함수 치환이 약하다" 같은 분석과 맞춤 추천으로 이어간다.

PDF 파싱, 변형 문제 생성, 채점처럼 시간이 걸리는 작업들은 요청-응답으로 묶지
않고 Kafka로 비동기 처리하기로 했다. 토픽은 세 개로 나눴다 — `pdf-uploaded`
(PDF 업로드 → 파싱+임베딩), `problem-generated`(변형 문제 생성 완료 → 알림/저장),
`student-answered`(학생 제출 → 채점+피드백). 각 단계가 오래 걸려도 앞 단계가
막히지 않게 하려는 목적이다.

## 1주차 — 인프라 구성

CLAUDE.md에 넣을 기술 스택/인프라를 먼저 정리하고, Docker Compose로 MariaDB(3306),
Kafka(9092), Chroma(8000)를 띄웠다. Kafka는 Zookeeper 없이 KRaft 모드로 구성했다.

## 2주차 — 파이프라인 뼈대와 비용 추정

개발 순서를 체크리스트로 잡았다.

```
1. docker-compose.yml (MariaDB + Kafka + Chroma)
2. Spring Boot 프로젝트 세팅 (Kafka Producer/Consumer 기본 구조)
3. PDF 업로드 → 이미지 변환 → Claude Vision OCR 파이프라인
4. Chroma 벡터 저장 + 유사 문제 검색
5. 변형 문제 생성 (Claude API)
6. 학생 답변 제출 → 채점/피드백 비동기 처리
7. React UI (관리자 / 학생 페이지)
```

3번까지 진행하면서, PDF만이 아니라 학생/강사가 캡처한 이미지로도 문제를 넣을
수 있어야겠다 싶어서 파일 타입을 분기했다.

```java
public enum FileType {
    PDF, IMAGE
}

List<String> pages = event.getFileType() == FileType.PDF
        ? pdfPageExtractor.extractBase64Pages(event.getStoredPath())
        : imageFileExtractor.extractBase64Images(event.getStoredPath());
```

본격적으로 돌리기 전에 비용부터 가늠해봤다. 수능 수학 1회분(약 30문제, 15페이지)
기준으로 단계별 토큰을 계산하면:

- **Vision OCR**: 페이지당 이미지 입력 ~1,600 토큰 + 시스템 프롬프트 ~500 토큰
  + LaTeX/메타데이터 출력 ~2,000 토큰 = 페이지당 ~4,100 토큰 × 15페이지 = **약 61,500 토큰**
- **임베딩**: 문제당 ~300 토큰 × 30문제 = **약 9,000 토큰**
- **변형 문제 생성**: 문제당 입력 500 + 출력 1,500 = 2,000 토큰 × 30문제 = **약 60,000 토큰**

합치면 약 13만 토큰, Sonnet 기준(Input $3/MTok, Output $15/MTok)으로 환산하면
OCR 약 $0.15 + 임베딩 약 $0.02 + 변형 생성 약 $0.75로 **총 $0.92 정도**다. 다만
그래프/도형이 많은 페이지는 출력 토큰이 2배까지 뛸 수 있고, Kafka 재시도/에러
처리가 붙으면 전체적으로 10~20%는 더 잡아야 한다. 정확한 측정은 결국 전체를
한 번에 돌리기보다 1페이지씩 먼저 실행해서 `response.usage` 로그로 검증하는
게 제일 현실적이었다.

## 3주차 — OCR 테스트와 임베딩에서 만난 문제들

40페이지 중 3페이지만 임시로 돌려서 10문제 가공까지는 성공했는데, Chroma에
저장하는 단계에서 걸렸다.

```
{"error":"Unimplemented","message":"The v1 API is deprecated. Please use /v2 apis"}
```

최신 버전 Chroma는 v1 API가 없어졌고, v2부터는 컬렉션 경로에 `tenant`/`database`를
(기본값이라도) 반드시 넣어야 했다.

```java
String tenant = "default";
String database = "default";

ObjectNode body = objectMapper.createObjectNode();
body.put("name", collectionName);
body.put("get_or_create", true);

String response = webClient.post()
        .uri("/api/v2/tenants/{tenant}/databases/{database}/collections", tenant, database)
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(body)
        .retrieve()
        .bodyToMono(String.class)
        .block();
```

이 문제를 풀고 나서는 유형별 저장까지는 됐는데, 임베딩 모델 선택에서 두 번째
벽을 만났다. 처음엔 태그·유형·문제 내용 세 가지를 다 임베딩에 넣었는데, 쓰고
있던 `paraphrase-multilingual-MiniLM-L12-v2`가 일반 자연어 임베딩 모델이라
수식이 섞인 "문제 내용"은 애초에 의미를 이해하지 못했다 — 그래서 문제 내용은
빼고 태그/유형만 임베딩하는 쪽으로 바꿨다.

그래프·도형을 표현하던 SVG 콘텐츠가 누락되는 문제도 있었다. 처음엔 Claude가
직접 그래프를 그리게 했는데 제대로 안 그려지는 경우가 많았다. 더 나은 방향은
Claude는 함수식(`y = x²-2x+1`) 같은 수식만 뽑아내고, 실제 렌더링은 프론트엔드의
그래핑 라이브러리(JSXGraph)가 맡는 구조였다 — 다만 JSXGraph는 프롬프트에 수식
설명을 더 자세히 써줘야 한다는 트레이드오프가 있었다.

## 3주차 끝 시점 정리

- Docker Compose로 MariaDB + Kafka + Chroma 구성 완료
- Kafka Producer/Consumer 적용 완료 (topic: `pdf-upload`)
- PDF → PNG 변환(그레이톤 처리로 이미지 토큰 절약) → OCR → 유형별 DB 적재
- Chroma 벡터 저장 완료, 임베딩은 `paraphrase-multilingual-MiniLM-L12-v2`를 별도
  FastAPI 서버로 분리해서 처리 (Java 앱 → Python FastAPI → Java 앱 → ChromaDB)
- 임베딩을 붙이고 나니 유사 문제 검색이 실제로 눈에 띄게 정확해졌다
- 그래프 렌더링은 Claude가 직접 그리는 대신 함수식만 추출해 프론트에서 그리는
  구조로 방향 전환
