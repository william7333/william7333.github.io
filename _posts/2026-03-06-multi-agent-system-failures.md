---
title: "Multi-Agent 시스템 비즈니스 도입이 실패하는 이유 패턴 분석"
date: 2026-03-06 13:15:00 +0900
categories: [AI, Architecture]
tags: [Multi-Agent, LLM, System Design, LangChain]
---

## "우리도 오토GPT 같은 거 만듭시다!"

최근 많은 기업들이 복잡한 문제를 해결하기 위해 **Agentic Workflow** 혹은 **Multi-Agent** 아키텍처 도입을 시도하고 있습니다. 
단일 프롬프트로는 해결할 수 없는 복합적인 테스크(예: 주식 분석 후 리포트 작성 및 자동 메일 발송)를 여러 특화된 에이전트(Agent)가 분업하여 처리하게 만드는 매력적인 방식입니다.

하지만 화려한 데모(Demo)와 달리, 프로덕션 레벨에서는 예상치 못한 장애물이 많습니다. 제가 경험한 성공과 실패 패턴을 공유합니다.

### 실패 패턴 1: 무한 루프(Infinite Loop)와 토큰 폭탄
에이전트 A와 B가 서로 대화를 주고받게 설계하면, 종종 예기치 않게 "네가 해," "아니 네가 해" 형태의 탁구 랠리가 벌어집니다.
이는 명확한 탈출 조건(Exit Condition)이나 최대 반복 횟수(Max Iterations) 제한 스크립트가 누락되었을 때 발생하며, 하룻밤 사이 수백만 개의 토큰 비용 청구서로 돌아옵니다.

### 실패 패턴 2: 도구(Tools)의 불명확한 정의
에이전트가 "인터넷 검색", "사내 DB 조회", "계산기" 같은 Tool을 선택해야 할 때, **도구의 Description(설명)**이 모호하면 에이전트가 어떤 상황에서 어떤 Tool을 써야 할지 혼동합니다.
* *나쁜 예*: `SearchDB: 데이터베이스를 검색합니다.`
* *좋은 예*: `Search_User_Payment_DB: 고객 ID를 기반으로 최근 1개월 간 결제 내역(금액, 날짜, 상태)을 조회합니다. 그 외 정보는 조회할 수 없습니다.`

### 성공을 위한 디자인 패턴: Router와 Supervisor
자율성(Autonomy)을 완전히 주는 대신, 초반에는 **Supervisor(관리자)** 역할을 하는 최고 노드(LLM)가 하위 에이전트들의 작업 순서와 결과를 엄격히 통제하는 **Hierarchical(계층적)** 구조나 제한된 **State Machine (LangGraph)** 기반의 워크플로우를 먼저 도입해야 합니다.

현업에서는 완전한 자율 에이전트(Autonomous Agent)보다, 잘 짜여진 "절차적 지능망(Procedural Intelligence)"이 훨씬 안정적입니다.
