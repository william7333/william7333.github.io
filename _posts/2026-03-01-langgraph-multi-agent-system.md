---
title: "RAG 시스템에서 LangGraph를 도입한 이유와 설계 전략"
date: 2026-03-01 10:00:00 +0900
categories: [AI, RAG]
tags: [LangGraph, Multi-Agent, LLM]
---

## 초기 RAG의 한계점

초기에 구성한 RAG 아키텍처는 전형적인 `Query → Retrieve → Generate`의 단방향 파이프라인 형태였습니다. 하지만 실무에서 사용자들의 질문은 점점 더 복잡해졌고, 하나의 Vector DB 탐색만으로는 완벽한 답변을 제공하는 데 한계가 있었습니다.

![LangGraph Logo](https://upload.wikimedia.org/wikipedia/commons/thumb/c/c3/Python-logo-notext.svg/110px-Python-logo-notext.svg.png) <!-- placeholder -->

우리 비즈니스 특성상 데이터는 다음과 같이 흩어져 있었습니다:
1. **정형 데이터** (PostgreSQL): 유저 프로필 및 요금제 정보
2. **비정형 문서 데이터** (Elasticsearch / pgvector): 매뉴얼, 가이드 문서
3. **지식 그래프 데이터** (Neo4j): 도메인 엔티티 간의 관계 정보

단방향 체인(Chain) 방식에서는, 어떤 데이터 소스를 쿼리해야 할지 결정하기 위한 동적 라우팅과 검색 실패 시 재검색(Fall-back) 로직을 짜는 것이 너무 복잡했습니다.

## LangGraph 기반 흐름 제어

이 문제를 해결하기 위해 LangGraph를 도입하여 Multi-Agent Orchestration 구조로 개편했습니다. LangGraph 구조 안에서는 각 검색기(Retriever), SQL 에이전트, 검증기(Validator) 등을 노드(Node)로 취급하고, LLM이 각 노드 사이의 상태(State) 전이를 관리합니다.

* **Router Node**: 질문의 의도를 분석해 적합한 DB 노드로 이동 (SQL vs ES vs Neo4j)
* **Retriever Nodes**: 각 툴을 사용해 정보 수집
* **Grader Node**: 수집된 정보가 질문을 해결하기 충분한지 판단. 불충분하면 검색어(Query)를 재생성 후 재탐색 루프

### 장점

1. **상태 관리(State Management)** 가 용이해졌습니다.
2. 복잡한 사이클(순환 참조 및 재수행)이 자연스럽게 구현되어 환각(Hallucination) 빈도가 대폭 감소했습니다.

LangGraph 도입으로 단순 RAG가 아닌 "자율형 AI 검색 시스템"으로 진화할 수 있는 강력한 기틀을 마련했습니다. 다음 포스트에서는 이 중에서도 Elasticsearch와 pgvector를 어떻게 결합했는지를 다뤄보겠습니다.
