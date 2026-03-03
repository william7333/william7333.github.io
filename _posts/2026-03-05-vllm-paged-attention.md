---
layout: post
title: "vLLM PagedAttention 완전 분석: LLM 서빙 처리량을 20배 높이는 핵심 기술"
date: 2026-03-05 14:00:00 +0900
tags: [vLLM, Inference, LLMOps, GPU, Performance]
---

LLM을 프로덕션 서버에 올리는 순간, 개발 환경에서는 경험하지 못했던 벽에 부딪힌다. 동시 사용자가 증가하면서 GPU 메모리는 고갈되고, 응답 지연은 급격히 증가하며, 서버는 OOM으로 죽는다. vLLM의 **PagedAttention**은 이 문제를 근본적으로 해결하는 혁신적인 기술이다.

## 문제 진단: KV Cache Memory Fragmentation

LLM 추론 중 각 토큰을 생성할 때마다 이전 토큰들의 Key-Value 쌍을 메모리에 보존해야 한다. 이를 **KV Cache**라 한다.

전통적인 서빙 프레임워크의 문제는 KV Cache를 **연속된(contiguous) 메모리 블록**에 사전 할당한다는 것이다.

```
전통적 방식:
┌──────────────────────────────────────────────┐
│ Request A [################          ]       │
│ Request B [########                  ]       │ ← 빈 공간 낭비
│ Request C [#######################   ]       │
│           (할당됐지만 사용 안 되는 공간 40-80%) │
└──────────────────────────────────────────────┘
```

응답 길이는 예측 불가능하다. 어떤 응답은 10 토큰, 어떤 응답은 2000 토큰이 될 수 있는데, 최대 길이를 가정해 사전 할당하면 메모리의 **60-80%가 낭비**된다.

## PagedAttention: OS의 가상 메모리에서 배운 해법

vLLM 팀은 운영체제의 **페이지(Page)** 메모리 관리 개념을 KV Cache에 적용했다.

```
PagedAttention 방식:
물리 메모리:     [Block 0] [Block 1] [Block 2] [Block 3] [Block 4]
                  Request A  Request C  Request B  Request A  Request C

Page Table:
  Request A → [Block 0, Block 3]   (실제로 사용하는 블록만 매핑)
  Request B → [Block 2]
  Request C → [Block 1, Block 4]
```

KV Cache를 고정 크기의 **블록(page)**으로 쪼개고, 논리적 주소와 물리적 주소를 **Page Table**로 분리한다. 필요한 블록만 할당하므로 낭비가 거의 없다(< 4%).

## 성능 비교: HuggingFace vs vLLM

```
테스트 환경: Llama-3-8B, A100 80GB, 동시 요청 50개

┌──────────────────┬──────────────────┬──────────────────┐
│ 방식              │ HuggingFace TGI  │ vLLM             │
├──────────────────┼──────────────────┼──────────────────┤
│ Throughput       │ 285 tokens/s     │ 5,840 tokens/s   │
│ P50 Latency      │ 2.1s             │ 0.18s            │
│ P99 Latency      │ 12.4s            │ 0.91s            │
│ Max Concurrency  │ ~8               │ ~200+            │
│ GPU Utilization  │ 45%              │ 91%              │
└──────────────────┴──────────────────┴──────────────────┘
```

**처리량 20배, P99 지연 13배 감소**. 단순 수치가 아니라 동일한 GPU 1개로 처리할 수 있는 동시 사용자 수가 8명에서 200명으로 늘어나는 것을 의미한다.

## 실제 배포 가이드

### Docker로 서빙 시작

```bash
docker run --runtime nvidia --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  vllm/vllm-openai:latest \
  --model meta-llama/Meta-Llama-3-8B-Instruct \
  --max-model-len 8192 \
  --max-num-seqs 256 \
  --tensor-parallel-size 1 \
  --gpu-memory-utilization 0.90
```

### OpenAI 호환 API

vLLM은 OpenAI API를 완벽히 호환하므로 코드 변경 없이 교체 가능하다.

```python
from openai import OpenAI

# 기존 OpenAI 코드와 동일한 인터페이스
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="EMPTY",  # 로컬 서버는 인증 불필요
)

response = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3-8B-Instruct",
    messages=[{"role": "user", "content": "RAG와 Fine-tuning의 차이는?"}],
    stream=True,  # Streaming 지원
)
```

## Continuous Batching: 처리량의 또 다른 비밀

기존 배치 처리는 배치 내 **가장 긴 응답이 완료될 때까지** 모든 요청이 대기해야 한다.

```
기존 방식:
Req A: [■■■■■■■■■■  ]  (완료, 대기 중...)
Req B: [■■■■■■■■■■■■■■■■■■■■]  (진행 중)
Req C:                [대기]  (B가 끝날 때까지 시작 못 함)

Continuous Batching:
Req A: [■■■■■■] → 완료 즉시
Req C:           [■■■■■■■■■]  ← 즉시 시작
Req B: [■■■■■■■■■■■■■■■■]
```

vLLM의 Continuous Batching은 응답이 완료된 슬롯에 **즉시 새 요청을 삽입**한다. GPU 활용률이 45% → 91%로 상승하는 핵심 메커니즘이다.

## 프로덕션 운영 시 체크리스트

- [ ] `--gpu-memory-utilization 0.90` (0.95 이상은 OOM 위험)
- [ ] `--max-model-len`을 실제 최대 컨텍스트 길이로 제한
- [ ] `--tensor-parallel-size`로 다중 GPU 활용
- [ ] 모니터링: Prometheus + Grafana로 `vllm:num_requests_running` 추적
- [ ] LoRA 어댑터: `--enable-lora --lora-modules` 로 런타임 스왑 가능
