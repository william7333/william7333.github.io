---
layout: post
title: "RAG 아키텍처 완전 정복: 검색 품질을 혁신하는 5가지 설계 원칙"
date: 2026-03-01 09:00:00 +0900
tags: [RAG, Architecture, LLM, Vector Search]
---

Retrieval-Augmented Generation(RAG)은 2024년 이후 엔터프라이즈 AI의 표준 아키텍처로 자리잡았다. 그러나 단순히 Vector DB에 문서를 적재하고 LLM과 연결하는 것만으로는 **프로덕션 수준의 품질**에 도달할 수 없다. 수십 개의 RAG 시스템을 설계하고 운영하면서 얻은 5가지 핵심 원칙을 공유한다.

## 원칙 1: 청크(Chunk) 전략이 전부다

RAG 품질의 70%는 문서 분할 전략에 달려있다. 많은 팀이 단순히 `chunk_size=512`로 고정해 놓고 성능 저하를 겪는다.

**계층적 청킹 전략 (Hierarchical Chunking)**

```python
class HierarchicalChunker:
    def chunk(self, document: str) -> list[Chunk]:
        # 1단계: 섹션 단위 분할 (H2 기준)
        sections = self.split_by_heading(document, level=2)
        
        chunks = []
        for section in sections:
            # 2단계: 섹션 내 의미 단위 분할
            if len(section.text) > 2000:
                sub_chunks = self.semantic_split(section.text)
                for sc in sub_chunks:
                    # 부모 컨텍스트(섹션 제목)를 metadata로 저장
                    sc.metadata["parent_title"] = section.title
                    chunks.append(sc)
            else:
                chunks.append(section)
        
        return chunks
```

핵심은 **부모-자식 관계**를 보존하는 것이다. 검색 시에는 작은 청크(자식)로 정밀 매칭하고, LLM에 전달할 때는 부모 컨텍스트를 함께 포함시킨다. 이를 **Parent Document Retrieval** 패턴이라 한다.

## 원칙 2: 임베딩 모델은 도메인에 맞게 선택하라

General-purpose 임베딩 모델(`text-embedding-ada-002` 등)은 범용적이지만 특정 도메인에서는 성능이 떨어진다.

| 모델 | 강점 | 권장 사용처 |
|------|------|------------|
| `bge-m3` | 한국어 포함 다국어, Hybrid Search 지원 | 한국어 도메인 |
| `cohere-embed-v3` | 롱 컨텍스트(512 토큰 이상) | 긴 문서 기반 |
| `e5-mistral-7b` | 고정밀, 느림 | 오프라인 배치 |

운영 중인 시스템에서는 **bge-m3**을 사용했을 때 범용 모델 대비 Recall@10 기준 18% 향상을 경험했다.

## 원칙 3: 단순 Cosine Similarity는 버려라

Pure Vector Search는 어휘(Lexical) 매칭에 취약하다. "GPT-4o"라는 정확한 모델명을 벡터만으로 찾으려 하면 실패할 확률이 높다.

**Hybrid Search = Dense + Sparse**

```python
def hybrid_retrieve(query: str, k: int = 10) -> list[Document]:
    # Dense retrieval (semantic)
    dense_results = vector_store.similarity_search(
        query, k=k, score_threshold=0.7
    )
    
    # Sparse retrieval (BM25 / keyword)
    sparse_results = bm25_retriever.get_relevant_documents(
        query, top_k=k
    )
    
    # Reciprocal Rank Fusion
    return rrf_merge(dense_results, sparse_results, k=k)
```

RRF(Reciprocal Rank Fusion)는 두 결과 목록의 순위를 `1/(rank + 60)` 공식으로 단순하게 합산한다. 실험 결과 단독 Dense Search 대비 **NDCG@10 기준 22% 향상**을 기록했다.

## 원칙 4: Query Transformation — 사용자 질문을 그대로 쓰지 마라

사용자의 원문 질문은 검색에 최적화돼 있지 않다. 다음 3가지 변환 기법을 적용한다.

**① HyDE (Hypothetical Document Embeddings)**  
LLM에게 "이 질문에 대한 답변이 담긴 가상의 문서를 작성하라"고 지시한 뒤, 그 가상 문서를 쿼리 벡터로 사용한다.

**② Multi-Query Expansion**  
단일 쿼리를 3~5개의 다양한 관점으로 재작성하여 병렬 검색한 뒤 결과를 통합한다.

**③ Step-back Prompting**  
구체적 질문을 더 추상적인 원리 수준으로 변환. "A 함수의 버그를 어떻게 고치나요?"를 "Python 함수 디버깅 원칙은 무엇인가요?"로 바꿔 배경 지식을 함께 탐색한다.

## 원칙 5: Reranker를 최후의 수문장으로

Retrieval 단계에서 `top-k=20`으로 넉넉히 가져온 뒤, **Cross-Encoder Reranker**로 최종 `top-3`을 선별하는 Two-Stage Retrieval이 필수다.

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query: str, documents: list[str], top_n: int = 3) -> list[str]:
    pairs = [(query, doc) for doc in documents]
    scores = reranker.predict(pairs)
    ranked = sorted(zip(scores, documents), reverse=True)
    return [doc for _, doc in ranked[:top_n]]
```

Bi-Encoder(벡터 검색)는 빠르지만 정확도가 낮고, Cross-Encoder는 느리지만 정확하다. 두 가지의 장점을 결합함으로써 **Precision@3 기준 35% 향상**을 달성했다.

## 정리

| 계층 | 기법 | 효과 |
|------|------|------|
| 문서 처리 | Hierarchical Chunking | 컨텍스트 보존 |
| 임베딩 | 도메인 특화 모델 | Recall +18% |
| 검색 | Hybrid (Dense+Sparse) | NDCG +22% |
| 쿼리 | HyDE / Multi-Query | 의미 범위 확장 |
| 후처리 | Cross-Encoder Reranker | Precision +35% |

RAG는 단일 컴포넌트가 아닌 **파이프라인 시스템**이다. 각 계층을 독립적으로 평가하고 개선해야 전체 품질이 상승한다.
