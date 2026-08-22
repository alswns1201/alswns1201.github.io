---
title: "벡터 DB는 왜 필요한가 — ChromaDB로 보는 임베딩 검색의 실체"
date: 2026-05-04
categories: [LLM]
---

*(RAG의 전체 흐름과 검색 품질 문제는 [RAG란 무엇인가]({% post_url 2026-04-26-rag-basics %})에서
다뤘다. 여기서는 그 파이프라인의 한 조각인 **벡터 DB**가 정확히 뭘 하는 물건인지를 판다.)*

## "그냥 DB에 벡터 컬럼 하나 추가하면 안 되나?"

이 질문에 답하려면 벡터 검색이 실제로 어떤 연산인지 봐야 한다. 임베딩 모델은 텍스트를
고차원 숫자 벡터로 바꾸고, 의미가 비슷한 텍스트는 벡터 공간에서 가까운 위치에 놓인다.
질문이 들어오면 그 질문도 벡터로 바꾼 뒤, 저장된 모든 벡터와의 거리(코사인 유사도
등)를 계산해서 가장 가까운 것을 찾는다.

문제는 "모든 벡터와 거리를 계산"하는 부분이다. 문서가 수천 개면 이 브루트포스
계산도 감당할 만하지만, 수백만 개가 되면 질문 하나마다 수백만 번의 거리 계산이
필요해진다 — 일반 RDB에 벡터를 blob이나 배열 컬럼으로 넣고 애플리케이션 레벨에서
전수 비교하면 이 비용을 고스란히 떠안는다. 벡터 DB가 실제로 하는 일은 이 전수
비교를 피하기 위한 **근사 최근접 이웃(ANN, Approximate Nearest Neighbor) 인덱스**를
구축하고 관리하는 것이다 — HNSW(계층적 그래프 탐색) 같은 자료구조로, 정확도를 아주
약간 희생하는 대신 검색 속도를 몇 자릿수 끌어올린다. "벡터를 저장하는 DB"가 아니라
"벡터를 빠르게 찾기 위한 인덱스를 관리하는 DB"라고 이해하는 게 더 정확하다.

## ChromaDB가 하는 일

ChromaDB는 이 역할을 하는 오픈소스 경량 벡터 DB다. 컬렉션(collection) 단위로
데이터를 격리해서 관리하고, 각 항목에 문서(document), 임베딩(embedding),
메타데이터(metadata)를 함께 저장한다.

```
질문 → 임베딩 변환 → 벡터 DB에서 유사 문서 검색 → LLM에 context로 전달 → 답변 생성
```

로컬에서도 가볍게 띄울 수 있고 임베딩 자동 생성 기능도 있어서, RAG 프로토타입
단계에서 특히 많이 쓰인다.

## Java에서 호출하기 (v2 REST API)

ChromaDB는 공식 Java SDK보다 REST API 기반 호출이 일반적이다.

```java
public class ChromaCollectionManager {
    private final String baseUrl = "http://localhost:8000/api/v2";
    private final RestTemplate restTemplate = new RestTemplate();

    public void createCollection(String collectionName) {
        Map<String, String> request = new HashMap<>();
        request.put("name", collectionName);
        request.put("get_or_create", "true");
        restTemplate.postForEntity(baseUrl + "/collections", request, String.class);
    }

    public void addDocument(String collectionId, String id, List<Double> embedding, String document) {
        Map<String, Object> request = new HashMap<>();
        request.put("ids", Collections.singletonList(id));
        request.put("embeddings", Collections.singletonList(embedding));
        request.put("documents", Collections.singletonList(document));
        restTemplate.postForEntity(baseUrl + "/collections/" + collectionId + "/add", request, String.class);
    }

    public String queryVector(String collectionId, List<Double> queryEmbedding, int nResults) {
        Map<String, Object> request = new HashMap<>();
        request.put("query_embeddings", Collections.singletonList(queryEmbedding));
        request.put("n_results", nResults);
        return restTemplate.postForObject(baseUrl + "/collections/" + collectionId + "/query", request, String.class);
    }
}
```

데이터 흐름은 단순하다: 컬렉션 생성 → 임베딩 모델로 벡터 생성 → ChromaDB에 POST →
쿼리 시 결과에서 문서와 거리(distance) 값을 추출. v2 API는 배치(batch) 처리에서
성능 이점이 있다.

## 메타데이터 필터링에서 흔히 놓치는 함정

ChromaDB는 벡터 유사도뿐 아니라 메타데이터로도 필터링할 수 있다(예: `category: "법률"`인
문서만 검색). 여기서 필터를 **언제** 적용하느냐가 결과 품질을 크게 좌우한다.

- **사후 필터링(post-filter)**: 일단 유사도 top-k를 뽑은 뒤 메타데이터 조건에
  안 맞는 것을 걸러낸다. 문제는, top-k 안에 조건에 맞는 문서가 하나도 없으면
  필터링 후 결과가 텅 비거나 k보다 훨씬 적어질 수 있다는 것이다.
- **사전 필터링(pre-filter)**: 메타데이터 조건을 먼저 적용한 부분집합 안에서만
  유사도 검색을 한다. 결과 개수는 보장되지만, 인덱스 구조에 따라 이게 항상
  효율적으로 지원되는 건 아니다.

"검색 결과가 이상하게 적게 나온다"는 문제를 겪으면, 사후 필터링 때문에 top-k에서
필터에 걸러진 게 아닌지부터 의심해봐야 한다 — 벡터 DB를 프로덕션에 붙일 때 실제로
가장 자주 만나는 버그 패턴 중 하나다.

## 언제 벡터 DB가 과한 선택인가

문서가 수백~수천 건 수준이고 검색어가 대체로 정확한 키워드 매칭으로 충분하다면,
벡터 DB와 임베딩 파이프라인을 새로 구축하는 비용이 오히려 배보다 배꼽이 클 수
있다. 전문 검색(full-text search) 엔진(Elasticsearch, PostgreSQL의 `tsvector` 등)의
기본 기능만으로 충분한 경우가 많다. 벡터 DB가 진짜 값을 하는 지점은, 정확한
키워드가 아니라 **의미**로 찾아야 하고 코퍼스가 커서 브루트포스 비교가 부담스러운
경우다.

## 프로토타입과 프로덕션의 간극

ChromaDB(특히 임베디드/로컬 모드)는 설치 없이 바로 실험하기 좋지만, 다중 인스턴스
클러스터링이나 복제 같은 프로덕션급 가용성 기능은 Pinecone, Qdrant, Weaviate 같은
관리형/분산 벡터 DB에 비해 약하다. 로컬 프로토타입에서 검증한 로직을 그대로
프로덕션 트래픽에 노출하기 전에, 가용성과 확장성 요구사항에 맞는 벡터 DB로
옮겨야 하는지를 별도로 검토해야 한다.

## 정리

- 벡터 DB의 본질은 "벡터 저장소"가 아니라 **ANN 인덱스로 유사도 검색을 빠르게
  만드는 인프라**다.
- ChromaDB는 컬렉션 단위로 document+embedding+metadata를 함께 관리하는 경량
  구현체다.
- 메타데이터 필터를 언제(사전/사후) 적용하는지가 결과 개수와 품질에 직접 영향을
  준다.
- 코퍼스가 작고 키워드 매칭으로 충분하면 벡터 DB는 과한 선택일 수 있다.
