---
layout: post
title: "Dense vs Sparse: Elastic + pgvector Hybrid Search 실전 구현 가이드"
date: 2026-03-04 13:00:00 +0900
tags: [Hybrid Search, Elasticsearch, pgvector, RAG, Vector Search]
---

Semantic search만으로 부족하고, keyword search만으로는 더욱 부족하다. 두 가지를 결합하는 **Hybrid Search**는 이제 엔터프라이즈 RAG의 기본기가 됐다. 그러나 "Elasticsearch에 벡터도 추가하면 되는 것 아닌가?"라는 단순한 접근 방식은 운영 환경에서 여러 함정을 드러낸다.

## 왜 두 가지 엔진이 필요한가?

| | BM25 (Sparse) | Vector (Dense) |
|--|--|--|
| 강점 | 정확한 키워드 매칭 | 의미론적 유사도 |
| 약점 | 동의어, 문맥 이해 불가 | 정확한 ID/코드/숫자 검색 취약 |
| 속도 | 매우 빠름 (역색인) | 근사 탐색 필요 (HNSW) |
| 예시 쿼리 | "GPT-4o API 요금" | "비용이 저렴한 AI 모델 추천" |

어떤 단일 방식도 두 유형의 쿼리를 모두 잘 처리할 수 없다. **Hybrid = 두 방식의 Union**.

## 우리 팀의 아키텍처 결정

Elasticsearch 8.x는 자체적으로 kNN Vector Search를 지원한다. 그러나 우리는 **역할 분리** 전략을 택했다.

```
Elasticsearch → Full-text search (BM25 + 형태소 분석)
PostgreSQL + pgvector → Dense vector KNN + 메타데이터 필터링
```

**이유:**

1. **권한/필터링 복잡도**: 유저 권한, 카테고리, 날짜 범위 등 복잡한 SQL JOIN 기반 필터링이 기존 PostgreSQL 인프라와 심리스하게 통합된다.
2. **Write 패턴 차이**: Elasticsearch는 full-text 인덱싱이 느린 반면, pgvector는 HNSW 인덱스 업데이트가 실시간 가능하다.
3. **운영 비용**: ES 클러스터를 벡터 전용으로 크게 확장하는 것보다 PostgreSQL 확장이 비용 효율적이다.

## pgvector HNSW 인덱스 설정

```sql
-- pgvector 확장 활성화
CREATE EXTENSION IF NOT EXISTS vector;

-- 문서 테이블 생성
CREATE TABLE documents (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content     TEXT NOT NULL,
    embedding   vector(1536),  -- OpenAI ada-002 차원
    metadata    JSONB,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- HNSW 인덱스 생성 (IVFFlat 대비 빠른 검색)
CREATE INDEX ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

## Reciprocal Rank Fusion (RRF) 구현

두 검색 결과를 합칠 때 단순 score 합산은 스케일이 달라 부정확하다. RRF는 **순위(rank)만** 사용해 score-agnostic 방식으로 통합한다.

```python
def reciprocal_rank_fusion(
    dense_results: list[tuple[str, float]],
    sparse_results: list[tuple[str, float]],
    k: int = 60,
    alpha: float = 0.7,
) -> list[str]:
    """
    k: RRF smoothing parameter (보통 60)
    alpha: dense 가중치 (1-alpha = sparse 가중치)
    """
    scores: dict[str, float] = {}
    
    for rank, (doc_id, _) in enumerate(dense_results):
        scores[doc_id] = scores.get(doc_id, 0) + alpha / (k + rank + 1)
    
    for rank, (doc_id, _) in enumerate(sparse_results):
        scores[doc_id] = scores.get(doc_id, 0) + (1 - alpha) / (k + rank + 1)
    
    return sorted(scores.keys(), key=lambda x: scores[x], reverse=True)

# 사용 예시
async def hybrid_search(query: str, top_k: int = 10) -> list[Document]:
    # 병렬 실행 (성능 중요!)
    dense_task = asyncio.create_task(pgvector_search(query, k=top_k * 2))
    sparse_task = asyncio.create_task(elasticsearch_search(query, k=top_k * 2))
    
    dense_results, sparse_results = await asyncio.gather(dense_task, sparse_task)
    
    merged_ids = reciprocal_rank_fusion(dense_results, sparse_results)
    return fetch_documents(merged_ids[:top_k])
```

## 운영 최적화 팁

**1. 쿼리 분류기로 alpha 동적 조절**

모든 쿼리에 동일한 alpha를 쓰는 것은 비효율적이다. 쿼리 유형에 따라 동적으로 조절한다.

```python
def adaptive_alpha(query: str) -> float:
    """keyword-heavy 쿼리면 sparse 가중치 ↑, 의미론적이면 dense ↑"""
    keyword_signals = len(re.findall(r'\b[A-Z0-9\-]{3,}\b', query))
    return max(0.3, 0.8 - keyword_signals * 0.1)
```

**2. Elasticsearch 한국어 분석기 설정**

```json
{
  "analysis": {
    "analyzer": {
      "korean_analyzer": {
        "type": "custom",
        "tokenizer": "nori_tokenizer",
        "filter": ["nori_posfilter", "lowercase"]
      }
    }
  }
}
```

## 도입 결과

| 검색 방식 | MRR@10 | Recall@10 | 지연(p99) |
|----------|--------|-----------|-----------|
| Dense Only | 0.61 | 0.74 | 45ms |
| Sparse Only | 0.58 | 0.69 | 12ms |
| **Hybrid (RRF)** | **0.79** | **0.88** | **52ms** |

Hybrid 방식이 개별 방식 대비 MRR 기준 약 **29% 향상**을 기록했다. 52ms 지연은 병렬 비동기 실행으로 허용 범위 내에 유지됐다.
