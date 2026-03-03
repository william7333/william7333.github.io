---
title: "LLM이 연결된 정보를 이해하는 방법: Graph RAG (Neo4j)"
date: 2026-03-03 15:45:00 +0900
categories: [AI, Data Engineering]
tags: [Neo4j, GraphRAG, LangChain, LLM]
---

## Vector Search의 근본적 한계: '관계망' 누락

일반적인 Vector DB 기반 RAG는 청크(Chunk) 단위로 쪼개진 텍스트에서 유사도를 검사합니다. 
문제는 문서 간 복잡한 비즈니스 로직, 즉 **"A 시스템이 B 시스템과 C 프로토콜로 연동된다"** 같은 다대다/인과 관계가 여러 텍스트 청크에 파편화되어 있을 경우 LLM이 전체 구조를 파악하기 힘들다는 것입니다.

이를 극복하기 위해 도입한 것이 지식 그래프(Knowledge Graph)와 Graph Database(**Neo4j**)입니다.

## Graph RAG 파이프라인

1. **Entity & Relationship 추출 파이프라인**
   사내 전체 문서(API 명세서, 가이드라인 등)를 대상으로 LLM을 활용해 `(Entity A) - [Relationship] -> (Entity B)` 형태의 트리플렛(Triplet)을 추출합니다.
   
2. **Neo4j 적재**
   추출된 트리플렛들을 바탕으로 거대한 Graph를 Neo4j에 구성합니다. 이 과정에서 중복된 Entity(예: "유저", "고객")를 모아서 클러스터링하는 정규화 작업이 수반됩니다.

3. **Cypher Query Agent (LangGraph 노드)**
   사용자가 복잡한 관계형 조회를 요구하는 질문(예: "이 API에 장애가 나면 영향을 받는 다른 서비스는 어떤 게 있고 담당자는 누구야?")을 던질 경우, RAG 에이전트는 Vector DB가 아닌 Graph Agent 노드로 넘깁니다.
   에이전트는 텍스트를 파싱하여 Neo4j 조회 언어인 Cypher Query를 스스로 생성하고 그래프 DB에 질의하여 정답을 추출합니다.

### 회고 및 다음 스텝

지식 그래프를 구축하는 것은 데이터 클렌징 비용이 막대하지만, 답변의 "설명 가능성(Explainability)"과 환각을 잡는 데에는 최고의 수단입니다. 다음에는 데이터 추출 시 LLM 토큰 비용을 줄이기 위해 소형 모델(SLM)을 파인튜닝하는 것을 적용해 볼 계획입니다.
