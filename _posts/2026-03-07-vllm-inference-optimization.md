---
title: "대규모 트래픽 LLM 추론, vLLM을 선택해야 하는 고찰"
date: 2026-03-07 10:45:00 +0900
categories: [LLMOps, Infra]
tags: [vLLM, Inference, PagedAttention, GPU]
---

## HuggingFace `generate()` 함수만으론 부족하다

로컬 환경이나 연구실(Research) 레벨에서는 HuggingFace `transformers` 라이브러리만으로도 훌륭히 작동합니다.
하지만 프로덕션 환경에서 초당 수십~수백 건의 동시 요청(Concurrent Requests)이 밀려오기 시작하면, 여지없이 GPU OOM(Out of Memory) 에러와 느려터진 토큰 생성 속도를 경험하게 됩니다.

### 병목의 원인: KV Cache Memory Fragmentation

LLM이 다음 단어를 예측할 때는 이전 토큰들의 연산 상태인 **Key, Value Cache(KV Cache)**를 메모리에 보관해야 합니다.
기존 서빙 시스템은 이 KV Cache를 연속적인(Contiguous) 메모리 블록에 미리 할당했습니다. 하지만 실제 사용자의 응답(Response) 토큰 길이는 예측할 수 없기 때문에, 할당된 메모리 공간 중간중간에 비어있는 "조각(Fragmentation)"이 생깁니다. 많게는 60~80%의 GPU VRAM이 이렇게 버려집니다.

## vLLM과 PagedAttention의 마법

이를 획기적으로 해결한 프레임워크가 바로 **vLLM**과 핵심 기술인 **PagedAttention**입니다. 
운영체제(OS)의 가상 메모리 페이징(Paging) 개념을 차용하여, KV Cache를 연속적인 공간 대신 고정된 크기의 블록(Block)으로 쪼개어(Page) 메모리에 적재합니다.

### 도입 결과 분석

1. **메모리 파편화 해결**: 버려지는 GPU VRAM을 5% 미만으로 줄일 수 있었습니다.
2. **배치 사이징(Batch Sizing) 향상**: 남게 된 엄청난 양의 VRAM을 활용해, 동시에 처리할 수 있는 Batch Size를 획기적으로 늘렸습니다.
3. **Throughput 폭발적 상승**: 실제 우리 서버 기준으로 14B 모델의 토큰 생성 처리율(Throughput)이 기존 대비 약 **14배 증가**했습니다. API 레이턴시 역시 절반 이하로 감소했습니다.

### 팁: Continuous Batching
기존에는 배치 작업(Batch) 중 가장 긴 문장 출력이 끝날 때까지 짧은 문장이 먼저 끝나도 기다려야 했습니다. vLLM은 **Continuous Batching(연속 배치 처리)**를 통해, 문장 생성이 완료된 요청은 내보내고 그 즉시 새로운 요청을 배치 대열에 끼워 넣습니다. 대규모 트래픽 처리의 핵심입니다.
