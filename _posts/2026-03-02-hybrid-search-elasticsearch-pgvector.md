---
title: "성공적인 Hybrid Search 구현 전략 (Elasticsearch + pgvector)"
date: 2026-03-02 12:30:00 +0900
categories: [AI, Search]
tags: [Hybrid Search, Elasticsearch, pgvector, Vector Search]
---

## Hybrid Search란 무엇인가?

RAG 환경에서 순수한 **Vector Search**(Dense Retrieval)는 의미론적인 유사성(Semantic Similarity)을 잘 잡아냅니다. 하지만, '2023 매출액' 같은 특정 키워드 매칭이나 정확한 ID 기반의 필터링에는 취약할 때가 많습니다. 이를 보완하기 위해 키워드 기반의 **Lexical Search**(Sparse Retrieval, BM25 등)와 결합하는 방식이 **Hybrid Search**입니다.

## 우리 팀의 아키텍처 (Elasticsearch + PostgreSQL)

최신 Elasticsearch는 Dense Vector 검색 기능이 강력해졌지만, 우리는 운영 정책상 다음과 같이 분리했습니다.
* **Elasticsearch**: 형태소 분석 및 전문 검색(Full-text Search), BM25
* **PostgreSQL + pgvector**: 정형 메타데이터 필터링 및 Dense Vector 기반 KNN 리트리버

### 왜 나누었을까?

1. **Relation 제어 효율성**:
   Vector Search를 할 때, 특정 유저의 권한이나 카테고리로 필터링을 거는 작업(Pre-filtering)이 중요합니다. 이 필터링 조건이 복잡할 경우, Elasticsearch의 필터보다 기존 RDBMS(PostgreSQL)의 Join과 인덱스를 활용하는 것이 개발 생산성 및 확장에 유리했습니다.
   
2. **PostgreSQL 확장성**:
   `pgvector` 모듈 덕분에 임베딩 데이터를 일반 RDBMS 테이블과 조인(Join)하여 빠르고 안전하게 HNSW 인덱싱할 수 있었습니다. 특히, 메타데이터 업데이트가 잦은 환경에서 ES 동기화 지연(Latency) 이슈 없이 원본 DB에서 직접 벡터 유사도 검사를 실행할 수 있다는 장점이 컸습니다.

## Reciprocal Rank Fusion (RRF) 조합

Elasticsearch에서 넘어온 결과(BM25)와 PostgreSQL에서 넘어온 결과(Vector)는 점수 분포를 그대로 합치기(Score 조합) 어렵습니다.
따라서, 우리는 Langchain과 통합된 사용자 정의 모듈을 바탕으로 **RRF 방식**의 순위 통합을 사용합니다. 

단순하지만 효과적이며, 의미론적 검색과 키워드 매칭 양쪽의 장점을 모두 살리는 추천 파이프라인으로 구성하였습니다.
