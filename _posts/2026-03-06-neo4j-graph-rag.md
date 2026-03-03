---
layout: post
title: "Neo4j Graph RAG: 지식 그래프로 LLM이 복잡한 관계형 질문을 해결하는 법"
date: 2026-03-06 15:00:00 +0900
tags: [Neo4j, Graph RAG, Knowledge Graph, LangChain, LLM]
---

전통적인 Vector RAG는 문서를 청크로 분리해 저장한다. 그러나 이 방식의 치명적 한계는 **"관계"를 표현하지 못한다**는 것이다. "A 시스템이 장애나면 B, C, D에 영향을 주고, 그 담당자는 누구이며, 최근 3개월 유사 장애 이력은?"이라는 질문에 벡터 검색으로는 답하기 어렵다.

지식 그래프(Knowledge Graph)와 Neo4j는 이 문제의 해답이다.

## 왜 Graph DB인가?

```
# Vector RAG가 저장하는 방식 (청크 단위):
"API 서버는 주문 서비스와 결제 서비스에 의존한다..."    ← text chunk 1
"결제 서비스 담당자는 김철수이며..."                    ← text chunk 2

# Graph RAG가 저장하는 방식 (구조적 관계):
(API 서버) -[DEPENDS_ON]-> (주문 서비스)
(API 서버) -[DEPENDS_ON]-> (결제 서비스)
(결제 서비스) -[MANAGED_BY]-> (김철수)
(결제 서비스) -[HAS_INCIDENT]-> (2025-12-15: 30분 장애)
```

3홉(hop) 관계를 단일 Cypher query로 즉시 탐색할 수 있다. 벡터 검색이라면 청크가 분리돼 있어 여러 쿼리와 LLM 호출이 필요하다.

## 지식 그래프 구축 파이프라인

### Step 1: Entity & Relation 추출

LLM을 활용해 비정형 문서에서 (`주어`, `관계`, `목적어`) 트리플렛을 추출한다.

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
import json

extraction_prompt = ChatPromptTemplate.from_template("""
다음 문서에서 엔티티와 관계를 JSON 형식으로 추출하세요.

문서: {text}

출력 형식:
{{
  "entities": [
    {{"id": "E1", "type": "System", "name": "주문 API"}}
  ],
  "relations": [
    {{"source": "E1", "relation": "DEPENDS_ON", "target": "E2"}}
  ]
}}
""")

def extract_graph(text: str) -> dict:
    chain = extraction_prompt | ChatOpenAI(model="gpt-4o", temperature=0)
    result = chain.invoke({"text": text})
    return json.loads(result.content)
```

### Step 2: Neo4j 적재

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

def load_to_neo4j(graph_data: dict):
    with driver.session() as session:
        # 엔티티 생성 (MERGE: 중복 방지)
        for entity in graph_data["entities"]:
            session.run(
                f"MERGE (n:{entity['type']} {{id: $id}}) "
                "SET n.name = $name",
                id=entity["id"], name=entity["name"]
            )
        
        # 관계 생성
        for rel in graph_data["relations"]:
            session.run(
                f"MATCH (a {{id: $src}}), (b {{id: $tgt}}) "
                f"MERGE (a)-[:{rel['relation']}]->(b)",
                src=rel["source"], tgt=rel["target"]
            )
```

### Step 3: Text-to-Cypher Agent

사용자의 자연어 질문을 Cypher 쿼리로 자동 변환한다.

```python
cypher_prompt = ChatPromptTemplate.from_template("""
당신은 Neo4j 전문가입니다. 사용자 질문을 Cypher 쿼리로 변환하세요.

스키마:
- (System)-[DEPENDS_ON]->(System): 시스템 의존성
- (System)-[MANAGED_BY]->(Person): 시스템 담당자
- (System)-[HAS_INCIDENT]->(Incident): 장애 이력

질문: {question}

Cypher 쿼리 (반드시 RETURN 절 포함):
""")

def text_to_cypher_query(question: str) -> str:
    chain = cypher_prompt | ChatOpenAI(model="gpt-4o", temperature=0)
    return chain.invoke({"question": question}).content.strip()

# 예시
question = "결제 서비스에 장애가 나면 영향받는 모든 서비스와 담당자는?"
cypher = text_to_cypher_query(question)
# 생성된 쿼리:
# MATCH (ps:System {name: "결제 서비스"})
# MATCH (s:System)-[*1..3]->(ps)
# MATCH (ps)-[:MANAGED_BY]->(p:Person)
# RETURN DISTINCT s.name AS affected_system, p.name AS owner
```

## Vector + Graph 하이브리드 RAG

벡터 검색과 그래프 검색을 LangGraph로 라우팅하는 완성형 아키텍처:

```python
def search_router(state: RAGState) -> str:
    """질문 유형에 따라 검색 경로 결정"""
    relational_keywords = ["관계", "의존", "영향", "담당", "연결", "소속"]
    
    if any(kw in state["query"] for kw in relational_keywords):
        return "graph_search"
    return "vector_search"
```

## 실전 성능 비교

| 질문 유형 | Vector RAG | Graph RAG |
|----------|-----------|-----------|
| "A서비스 장애 시 영향 범위는?" | ❌ 관계 파악 불가 | ✅ 즉시 3홉 탐색 |
| "B 기능 담당자 연락처" | △ 운이 좋으면 | ✅ 직접 엔티티 조회 |
| "최근 유사 장애 이력" | ✅ 텍스트 유사도 | △ 구조화 정도에 따라 |
| "C 개념을 설명해줘" | ✅ 최적 | △ 비구조적 정보 약함 |

**결론**: Vector와 Graph는 경쟁 관계가 아니다. 두 가지를 라우팅해 사용하는 하이브리드 아키텍처가 최고의 정확도를 달성한다.
