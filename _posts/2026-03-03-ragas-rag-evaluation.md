---
layout: post
title: "Hallucination과의 전쟁: RAGAS로 RAG 파이프라인 품질 정량화하기"
date: 2026-03-03 11:00:00 +0900
tags: [RAGAS, Evaluation, RAG, LLM, Metrics]
---

RAG 시스템을 만들고 나서 "잘 작동하는가?"라는 질문에 **데이터로 답하지 못하는** 팀이 너무 많다. 몇 개의 예시 쿼리를 손으로 테스트하는 "눈대중 평가"는 재현성이 없고, 모델/프롬프트 변경 시 회귀(Regression)를 잡아낼 수 없다. **RAGAS**는 이 문제를 해결하는 RAG 전용 자동 평가 프레임워크다.

## RAGAS의 4가지 핵심 지표

RAGAS는 RAG의 두 축인 **검색(Retrieval)**과 **생성(Generation)**을 독립적으로 측정한다.

```
RAG Pipeline
   │
   ├── Retrieval Quality
   │     ├── Context Precision    (검색된 청크의 정밀도)
   │     └── Context Recall       (정답 도출에 필요한 정보 포함률)
   │
   └── Generation Quality
         ├── Faithfulness         (환각 방지: 컨텍스트 기반 답변 여부)
         └── Answer Relevancy     (질문과 답변의 관련성)
```

### 1. Context Precision
검색된 청크들 중 실제로 정답 생성에 **유용한 청크의 비율**이다. 불필요한 노이즈를 얼마나 걸러냈는지 측정한다.

$$\text{Context Precision} = \frac{|\text{유용한 청크}|}{|\text{전체 검색 청크}|}$$

점수가 낮으면 → 청크 품질 문제, Reranker 미흡

### 2. Context Recall
정답을 생성하는 데 필요한 정보 중 실제로 검색된 비율이다. Ground Truth Answer(정답)를 기반으로 LLM이 채점한다.

점수가 낮으면 → top-k 부족, Embedding 모델 약점, 청크 분할 문제

### 3. Faithfulness (가장 중요)
생성된 답변이 **오직 검색된 컨텍스트에 기반**하는지 측정한다. 이것이 환각(Hallucination) 측정 지표다.

```python
# RAGAS 내부 동작 (개념 코드)
def measure_faithfulness(answer: str, contexts: list[str]) -> float:
    # 1. 답변을 개별 주장(claim)으로 분해
    claims = extract_claims(answer)
    
    # 2. 각 주장이 컨텍스트에서 추론 가능한지 검증
    verified = 0
    for claim in claims:
        if can_be_inferred_from(claim, contexts):
            verified += 1
    
    return verified / len(claims)
```

### 4. Answer Relevancy
답변이 원래 질문에 얼마나 잘 대응하는지 측정한다. 답변이 풍부하더라도 질문과 어긋나면 낮은 점수를 받는다.

## 실제 평가 파이프라인 구축

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset

# 평가 데이터셋 준비 (사내 정답셋)
eval_data = {
    "question": ["RAG에서 Reranker의 역할은?", ...],
    "answer": [rag_pipeline.invoke(q) for q in questions],
    "contexts": [retrieve(q) for q in questions],
    "ground_truth": ["Reranker는 ...", ...],  # 전문가가 작성한 정답
}
dataset = Dataset.from_dict(eval_data)

# 평가 실행
results = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
    llm=ChatOpenAI(model="gpt-4o"),  # Judge LLM
)

print(results)
# Output:
# faithfulness        : 0.82
# answer_relevancy    : 0.91
# context_precision   : 0.75
# context_recall      : 0.88
```

## CI/CD에 평가 파이프라인 통합

GitHub Actions를 활용해 프롬프트 변경 PR마다 자동으로 RAGAS 점수를 측정한다.

```yaml
# .github/workflows/eval.yml
name: RAG Quality Gate

on:
  pull_request:
    paths:
      - 'prompts/**'
      - 'retrievers/**'

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run RAGAS evaluation
        run: python scripts/eval_ragas.py
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      
      - name: Quality Gate
        run: |
          python -c "
          import json
          with open('eval_results.json') as f:
              results = json.load(f)
          assert results['faithfulness'] >= 0.80, 'Faithfulness 기준 미달!'
          assert results['context_recall'] >= 0.85, 'Recall 기준 미달!'
          print('✅ 품질 기준 통과')
          "
```

## 6개월 운영 결과

| 지표 | 도입 전 | 3개월 후 | 6개월 후 |
|------|---------|---------|---------|
| Faithfulness | 0.58 | 0.79 | 0.89 |
| Answer Relevancy | 0.72 | 0.84 | 0.93 |
| Context Recall | 0.64 | 0.81 | 0.88 |

**Faithfulness**가 가장 극적으로 개선됐다. "컨텍스트에 없으면 답하지 말 것"이라는 System Prompt 강화와 Reranker 도입이 결정적이었다. RAGAS 없이는 이 인과관계를 파악할 수 없었을 것이다.
