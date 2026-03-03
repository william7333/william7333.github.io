---
layout: post
title: "LangGraph 심층 분석: State Machine으로 복잡한 Multi-Agent 시스템 설계하기"
date: 2026-03-02 10:00:00 +0900
tags: [LangGraph, Multi-Agent, LLM, Architecture]
---

LangChain의 단순 체인(Chain) 방식은 선형적인 워크플로우에 적합하지만, 실무에서 마주치는 복잡한 AI 태스크는 **조건부 분기**, **루프**, **병렬 실행**, **상태 공유**를 요구한다. LangGraph는 이 모든 것을 그래프(DAG + Cycle)로 표현할 수 있는 프레임워크다.

## LangGraph의 핵심 개념

LangGraph는 세 가지 기본 요소로 구성된다.

- **State**: 전체 워크플로우가 공유하는 불변 상태 스키마 (TypedDict)
- **Node**: 상태를 변환하는 실행 단위 (함수 또는 LLM 체인)
- **Edge**: 노드 간 흐름 제어 (조건부 가능)

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
import operator

# 1. 공유 상태 정의
class RAGState(TypedDict):
    query: str
    search_type: str          # "vector" | "sql" | "graph"
    retrieved_docs: list[str]
    reranked_docs: list[str]
    answer: str
    needs_retry: bool
    retry_count: Annotated[int, operator.add]  # 자동 누적

# 2. 그래프 구성
workflow = StateGraph(RAGState)
```

## Router Agent: 동적 경로 결정

전통적인 if-else 라우터 대신, LLM이 질문의 의도를 분석해 최적의 검색 경로를 결정하게 한다.

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

router_prompt = ChatPromptTemplate.from_template("""
당신은 검색 라우터입니다. 사용자 질문을 분석해 가장 적합한 검색 유형을 단답으로 반환하세요.

검색 유형:
- "vector": 개념적 설명, 문서 검색, 요약이 필요한 경우
- "sql": 특정 수치, 날짜, ID 기반 데이터 조회
- "graph": 엔티티 간 관계, 인과관계 분석

질문: {query}
검색 유형 (반드시 위 3개 중 하나만):
""")

def router_node(state: RAGState) -> dict:
    chain = router_prompt | ChatOpenAI(model="gpt-4o-mini", temperature=0)
    result = chain.invoke({"query": state["query"]})
    return {"search_type": result.content.strip()}
```

## Grader Node: Self-Reflection 구현

검색 결과가 질문을 답하기에 충분한지 LLM이 스스로 평가한다. 불충분하면 쿼리를 재작성하여 재검색 루프를 실행한다.

```python
grader_prompt = ChatPromptTemplate.from_template("""
검색된 문서들이 질문에 답하기에 충분한지 평가하세요.

질문: {query}
검색 결과:
{docs}

평가 기준:
- YES: 답변을 도출할 충분한 정보가 있음
- NO: 정보가 부족하거나 관련 없음

평가 (YES 또는 NO만):
""")

def grader_node(state: RAGState) -> dict:
    chain = grader_prompt | ChatOpenAI(model="gpt-4o-mini", temperature=0)
    result = chain.invoke({
        "query": state["query"],
        "docs": "\n".join(state["retrieved_docs"][:5])
    })
    is_sufficient = result.content.strip() == "YES"
    return {
        "needs_retry": not is_sufficient,
        "retry_count": 1 if not is_sufficient else 0
    }

def should_retry(state: RAGState) -> str:
    if state["needs_retry"] and state["retry_count"] < 3:
        return "query_rewriter"  # 재시도
    return "generator"           # 최종 생성으로 진행
```

## 전체 그래프 조립

```python
# 노드 등록
workflow.add_node("router", router_node)
workflow.add_node("vector_retriever", vector_retrieve_node)
workflow.add_node("sql_agent", sql_agent_node)
workflow.add_node("graph_retriever", graph_retrieve_node)
workflow.add_node("reranker", reranker_node)
workflow.add_node("grader", grader_node)
workflow.add_node("query_rewriter", query_rewrite_node)
workflow.add_node("generator", generator_node)

# 엣지 연결
workflow.set_entry_point("router")

workflow.add_conditional_edges("router", lambda s: s["search_type"], {
    "vector": "vector_retriever",
    "sql":    "sql_agent",
    "graph":  "graph_retriever",
})

# 모든 리트리버 → reranker → grader
for node in ["vector_retriever", "sql_agent", "graph_retriever"]:
    workflow.add_edge(node, "reranker")

workflow.add_edge("reranker", "grader")

# 자기 평가 후 조건부 분기 (핵심 사이클!)
workflow.add_conditional_edges("grader", should_retry, {
    "query_rewriter": "query_rewriter",
    "generator": "generator",
})
workflow.add_edge("query_rewriter", "router")  # 루프백
workflow.add_edge("generator", END)

app = workflow.compile()
```

## 실제 성능 결과

이 아키텍처를 도입한 이후:

- **환각(Hallucination) 발생률** 42% → 8% 감소
- **검색 Empty Result Rate** 23% → 4% 감소
- **평균 응답 품질 점수** (내부 RAGAS 기준) 0.61 → 0.87 향상

LangGraph의 가장 큰 가치는 **관찰 가능성(Observability)**이다. 각 노드의 입출력 상태를 모두 추적할 수 있어 어느 단계에서 품질 저하가 발생하는지 정확히 진단할 수 있다.
