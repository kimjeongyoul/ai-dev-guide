# [AI 서빙 2편] 분산 추론(Distributed Inference) — Tensor vs Pipeline Parallelism 아키텍처 비교

&nbsp;

70B, 405B... 거대 모델의 시대다. 최신 LLM의 파라미터 수와 활성화 함수가 사용하는 메모리 양은 이제 단일 GPU(예: NVIDIA H100 80GB)의 물리적 한계를 가볍게 뛰어넘는다. 결국 우리는 모델을 조각내어 여러 장의 GPU, 혹은 여러 대의 서버 노드에 분산 배치해야 한다.

&nbsp;

하지만 모델을 어떻게 쪼개느냐에 따라 추론 속도와 통신 비용은 천차만별로 달라진다. 본 글에서는 모델 분산의 두 축인 **Tensor Parallelism (TP)**과 **Pipeline Parallelism (PP)**의 기술적 차이점을 하드웨어 통신 프로토콜(NVLink, InfiniBand) 관점에서 분석하고, 실무에서 어떤 상황에 어떤 전략을 택해야 하는지 가이드를 제시한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. Tensor Parallelism (TP): 연산의 수평적 분할

&nbsp;

TP는 하나의 트랜스포머 레이어 내부의 행렬 연산 자체를 쪼개어 여러 GPU가 동시에 수행하게 만드는 방식이다.

&nbsp;

- **작동 원리**: 예를 들어 `Y = XW`라는 거대 행렬 곱이 있다면, 가중치 행렬 `W`를 열(Column) 단위로 쪼개어 GPU 1과 GPU 2에 나눠준다. 각 GPU는 자신의 몫만큼 부분 연산을 수행하고, 마지막에 **All-Reduce** 통신을 통해 결과를 합친다.
- **장점**: 모델 전체의 지연 시간(Latency)을 낮추는 데 가장 효과적이다. 모든 GPU가 동시에 연산에 참여하기 때문이다.
- **단점**: 매 레이어마다 GPU 간의 데이터 동기화(All-Reduce)가 발생한다. 따라서 GPU들이 **NVLink**와 같은 초고속 내부 대역폭으로 묶여 있지 않으면, 통신 시간이 연산 시간보다 길어지는 배보다 배꼽이 더 큰 상황이 벌어진다. 

&nbsp;

&nbsp;

---

&nbsp;

# 2. Pipeline Parallelism (PP): 레이어의 수직적 배치

&nbsp;

PP는 모델의 레이어 뭉치를 여러 GPU에 순차적으로 할당하는 방식이다. 1~20번 레이어는 GPU 1에, 21~40번 레이어는 GPU 2에 두는 식이다.

&nbsp;

- **작동 원리**: 첫 번째 GPU가 자신의 레이어 연산을 끝내면 그 결과(Activation)를 다음 GPU로 넘겨준다. 공장의 조립 라인과 흡사하다.
- **장점**: GPU 간 통신량이 TP에 비해 압도적으로 적다. 레이어 뭉치의 결과값만 넘기면 되기 때문이다. 따라서 NVLink가 없는 서로 다른 서버 노드 간 분산(Multi-node)에 유리하다.
- **단점**: '파이프라인 버블(Bubble)' 문제가 발생한다. 뒤쪽 GPU는 앞쪽 GPU가 작업을 끝내서 결과물을 넘겨줄 때까지 놀게 된다. 이를 해결하기 위해 데이터를 작은 단위(Micro-batch)로 쪼개어 밀어 넣는 스케줄링 기법이 동반되어야 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. TP vs PP: 엔지니어의 선택 기준

&nbsp;

어떤 병렬화 전략을 쓸 것인가는 보유한 인프라의 **네트워크 토폴로지**에 달려 있다.

&nbsp;

1. **단일 서버 내 8장 GPU (NVLink 탑재)**: **Tensor Parallelism (TP)**이 최우선이다. 지연 시간이 가장 짧고 구현이 직관적이다.
2. **여러 대의 서버 노드 (InfiniBand 탑재)**: 노드 내에서는 TP를 쓰고, 노드 간에는 **Pipeline Parallelism (PP)**을 섞어 쓰는 **Hybrid Parallelism**을 구축해야 한다.
3. **가성비 구성 (일반 Ethernet 환경)**: TP는 거의 불가능하다. 데이터 병렬화(Data Parallelism)나 PP 위주로 설계하고, 최대한 마이크로 배치를 최적화하여 버블을 줄여야 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 실전 도구: Megatron-LM과 DeepSpeed

&nbsp;

이러한 복잡한 분산 연산을 밑바닥부터 짤 필요는 없다. 
- **Megatron-LM**: NVIDIA가 개발한 프레임워크로, 트랜스포머 구조에 최적화된 TP 구현을 제공한다.
- **DeepSpeed**: Microsoft가 개발한 라이브러리로, ZeRO(Zero Redundancy Optimizer) 기술을 통해 메모리 중복을 제거하며 효율적인 분산 추론을 가능하게 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 병렬화의 본질은 통신 오버헤드와의 싸움

&nbsp;

분산 추론은 단순히 GPU를 많이 붙인다고 비례해서 빨라지는 마법이 아니다. 오히려 "어떻게 하면 GPU들이 서로 대화하느라 낭비하는 시간을 연산 시간 뒤로 숨길 수 있을까?"에 대한 치열한 설계의 산물이다. 

&nbsp;

자신의 인프라가 가진 네트워크 대역폭(Bandwidth)과 지연 시간(Latency) 수치를 정확히 이해하고, 그에 맞는 분산 전략을 선택하는 것. 그것이 수천억 파라미터 모델을 0.1초 만에 응답하게 만드는 AI 인프라 엔지니어의 핵심 역량이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI 서빙 3편] KV Cache의 비밀 — LLM 추론 속도를 10배 높이는 캐시 압축 및 재사용 전략**

&nbsp;

문장이 길어질수록 추론 속도가 느려지는 이유는 무엇일까? 매 토큰 생성 시 반복되는 Attention 연산을 획기적으로 줄여주는 KV Cache의 내부 구조를 분석한다. 또한, 컨텍스트가 겹치는 여러 요청에서 캐시를 재사용하는 **Prefix Caching**과, 중요도가 낮은 정보를 삭제하여 메모리를 확보하는 **KV Cache Eviction** 기술의 실체를 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

분산추론, TensorParallelism, PipelineParallelism, 모델병렬화, NVLink, MegatronLM, AI인프라, GPU네트워킹
