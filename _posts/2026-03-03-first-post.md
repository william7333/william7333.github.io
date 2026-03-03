---
layout: post
title: "RAG 시스템에서 LangGraph를 도입한 이유"
date: 2026-03-03
---

## 왜 LangGraph를 사용했는가?

기존 RAG는 단순 Query → Retrieve → Generate 구조였다.

하지만 우리 시스템은:

- SQL 검색
- Vector 검색
- Graph 검색

이 3가지를 조합해야 했다.

그래서 LangGraph 기반 Multi-node 설계를 도입했다.
