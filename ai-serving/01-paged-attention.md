# [AI 서빙 1편] vLLM과 PagedAttention — 메모리 단편화를 해결하는 운영체제(OS)급 혁신

&nbsp;

LLM(거대 언어 모델) 서빙의 가장 큰 적은 '메모리 부족'과 '낮은 처리량(Throughput)'이다. 특히 추론 과정에서 이전 토큰들의 정보를 저장해두는 **KV Cache**는 GPU 메모리의 막대한 양을 잡아먹는다. 기존의 서빙 프레임워크들은 이 KV Cache를 위해 미리 고정된 크기의 연속적인 메모리를 할당했는데, 이는 필연적으로 심각한 메모리 단편화(Fragmentation)와 낭비를 초래했다. 

&nbsp;

이 문제를 운영체제의 '가상 메모리(Virtual Memory)' 개념을 빌려와 혁신적으로 해결한 프로젝트가 바로 **vLLM**이다. 본 글에서는 vLLM의 핵심 알고리즘인 **PagedAttention**의 작동 원리와, 이를 통해 어떻게 기존 대비 2~4배 이상의 서빙 성능을 달성했는지 기술적으로 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. KV Cache: 추론 성능의 핵심이자 병목

&nbsp;

LLM은 Auto-regressive하게 토큰을 하나씩 생성한다. 이때 이전 단계에서 계산했던 Attention 값들을 다시 계산하지 않기 위해 메모리에 저장해두는데, 이를 KV(Key-Value) Cache라고 한다.

&nbsp;

- **문제점**: 생성되는 문장의 길이는 미리 알 수 없다. 따라서 기존 방식은 최대 문장 길이(Max Sequence Length)에 맞춰 메모리를 미리 '연속적으로' 할당한다. 
- **결과**: 사용자가 짧은 문장만 입력하더라도 이미 수 기가바이트의 메모리가 예약되어 버리며, 실제 사용되지 않는 '내부 단편화'가 발생한다. 이는 동시에 처리할 수 있는 사용자 수(Batch Size)를 극도로 제한한다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. PagedAttention: 가상 메모리 기법의 이식

&nbsp;

vLLM은 이 문제를 해결하기 위해 메모리를 '페이지(Page)' 단위로 쪼개어 관리하는 **PagedAttention**을 도입했다.

&nbsp;

## 2-1. 물리적 비연속성 (Non-contiguous Physical Memory)
PagedAttention은 문장 전체를 위한 연속적인 공간을 찾지 않는다. 대신, KV Cache 데이터를 일정한 크기의 블록(Block)으로 나눈다.
- **물리적 블록**: GPU 메모리 상의 실제 저장 공간.
- **논리적 블록**: 특정 요청(Request)이 인식하는 가상 공간.

&nbsp;

커널의 페이징 기법처럼, 논리적으로는 이어져 있는 데이터가 물리적으로는 GPU 메모리 여기저기에 흩어져 저장된다. 이를 관리하기 위해 vLLM은 내부적으로 **Block Table**을 유지하며 논리 주소를 물리 주소로 실시간 매핑한다.

&nbsp;

## 2-2. 낭비 제로: On-demand Allocation
메모리는 토큰이 생성될 때 필요한 만큼만 '블록 단위'로 동적 할당된다. 마지막 블록에서만 미세한 낭비가 발생할 뿐, 전체적인 메모리 활용률은 96% 이상으로 치솟는다. 남은 여유 공간에는 더 많은 사용자의 요청을 수용할 수 있게 되어 처리량이 수직 상승한다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. Copy-on-Write와 효율적인 병렬 샘플링

&nbsp;

PagedAttention의 또 다른 백미는 **메모리 공유(Memory Sharing)** 기능이다. 

&nbsp;

- **시나리오**: 하나의 질문에 대해 AI가 5개의 서로 다른 답변을 동시에 생성하는 경우 (Parallel Sampling).
- **최적화**: 5개의 답변은 질문 부분의 KV Cache를 공통으로 가진다. PagedAttention은 물리 블록의 참조 횟수(Reference Count)를 관리하여, 공통 부분은 메모리에 단 하나만 올리고 5개의 요청이 이를 공유하게 한다.
- **작동**: 특정 요청이 공통 부분을 벗어나 자신만의 토큰을 쓰기 시작할 때만 해당 블록을 복사하는 **Copy-on-Write** 방식을 사용하여 메모리 효율을 극한으로 끌어올린다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 실전 배포 시 고려사항: Block Size 튜닝

&nbsp;

vLLM 운영 시 가장 중요한 파라미터는 `block_size`다.
- **작은 블록 (예: 8)**: 단편화는 최소화되지만, Block Table 관리에 따른 오버헤드가 증가하고 GPU 가속기(Kernel)의 연산 효율이 떨어질 수 있다.
- **큰 블록 (예: 32)**: 연산 효율은 좋아지지만, 다시 단편화 문제가 고개를 든다.

&nbsp;

보통 16 혹은 32가 권장되며, 사용자의 평균 문장 길이에 따라 벤치마크를 통해 최적값을 찾아야 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 소프트웨어가 하드웨어 한계를 넘다

&nbsp;

vLLM의 성공은 모델의 파라미터를 줄이거나 가중치를 압축하는 방식이 아닌, **인프라 레이어에서의 메모리 관리 기법 혁신**만으로도 성능을 몇 배나 올릴 수 있음을 증명했다. 

&nbsp;

PagedAttention은 이제 최신 LLM 서빙 프레임워크의 표준이 되었으며, 이를 이해하는 것은 더 이상 인프라 엔지니어에게 선택이 아닌 필수다. 하드웨어(GPU)의 물리적 한계를 운영체제론적 관점에서 해결해 나가는 과정, 그것이 바로 현대 AI 인프라 엔지니어링의 정수다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI 서빙 2편] 분산 추론(Distributed Inference) — Tensor vs Pipeline Parallelism 아키텍처 비교**

&nbsp;

모델의 크기가 단일 GPU의 메모리를 넘어서면 어떻게 해야 할까? 모델의 레이어를 쪼개어 여러 GPU에 순차적으로 배치하는 **Pipeline Parallelism**과, 하나의 레이어 연산 자체를 여러 GPU에 분산하여 동시에 처리하는 **Tensor Parallelism**의 내부 연산 구조를 분석한다. 각 방식이 유발하는 통신 병목(Communication Overhead)과 최적의 하이브리드 전략을 파헤친다.

&nbsp;

&nbsp;

---

&nbsp;

vLLM, PagedAttention, KVCache, GPU메모리최적화, 단편화해결, LLM서빙, AI인프라, 가상메모리
