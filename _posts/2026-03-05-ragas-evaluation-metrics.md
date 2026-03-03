---
title: "RAG 시스템 성능은 어떻게 평가할까? (RAGAS 도입기)"
date: 2026-03-05 14:00:00 +0900
categories: [AI, RAG]
tags: [RAGAS, Evaluation, Metrics, LangChain]
---

## RAG의 '잘함의 기준'은 무엇일까?

RAG (Retrieval-Augmented Generation) 시스템을 만들고 나면, 기획자와 개발자 모두가 마주하는 질문이 있습니다. 
*"이거 진짜 답변 잘 하는 거 맞아요?"*

단순히 몇 개의 테스트 쿼리를 날려보는 "눈대중 평가"로는 지속적인 릴리즈 및 프롬프트 개선을 보장할 수 없습니다. 우리는 이를 정량화하기 위해 **RAGAS (RAG Assessment)** 프레임워크를 파이프라인에 통합했습니다.

## RAGAS의 4가지 핵심 지표 (Metrics)

RAGAS는 크게 **검색(Retrieval) 품질**과 **생성(Generation) 품질**을 분리하여 평가합니다.

1. **Context Relevance (문맥 적합성)**
   * 검색된 문서(Chunk)들이 사용자의 질문과 얼마나 직접적으로 관련이 있는가? 
   * 쓸데없는 노이즈를 얼마나 잘 걸러냈는지 평가합니다.
2. **Context Recall (문맥 재현율)**
   * 완벽한 답변을 하기 위해 필요한 정보들이 모두 검색결과에 포함되어 있는가?
3. **Faithfulness (사실성/충실성)**
   * LLM이 생성한 답변이 **오직** 검색된 컨텍스트(문서)에 기반하여 작성되었는가? (환각(Hallucination) 측정)
4. **Answer Relevance (답변 적합성)**
   * 생성된 답변이 사용자의 **질문 의도**에 정확히 부합하는가?

## 평가 자동화 파이프라인 연동

우리는 CI/CD 툴(GitHub Actions)과 연동하여, 프롬프트나 Embedding 모델 체중이 변경될 때마다 자동 평가를 수행합니다.

* **Ground Truth Dataset (정답셋) 구축**: 프로덕션에서 수집된 실제 사용자 질문 100~200개를 메인 테스트셋으로 구성.
* **LLM-as-a-Judge**: RAGAS 내부적으로는 강력한 모델(예: GPT-4)을 평가관(Judge)으로 사용하여, 로컬 RAG 모델이 뱉어낸 결과를 위 4가지 지표(0.0 ~ 1.0 점수)로 채점합니다.

### 깨달은 점
이전에는 Chunk Size를 늘릴지, Hybrid Search 가중치를 조절할지 감으로 때려맞췄다면, 이제는 **"Context Relevance가 떨어지니 Chunk 오버랩을 줄이자"**, **"Faithfulness가 낮으니 프롬프트에 '문서에 없으면 모른다고 답하라'는 제약을 더 강하게 주자"** 등 데이터 옵저버빌리티(Observability) 기반의 의사결정이 가능해졌습니다.
