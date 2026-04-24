# [AI 서빙 4편] Triton Inference Server — 다이나믹 배칭(Dynamic Batching)을 통한 GPU 효율 극대화

&nbsp;

GPU는 대규모 병렬 연산에 특화된 하드웨어다. 하지만 실시간 추론(Inference) 서비스 환경에서는 개별 사용자의 요청이 무작위 시점에 하나씩 들어오게 된다. 만약 GPU가 요청 하나가 들어올 때마다 연산을 수행한다면, 수천 개의 코어(CUDA Cores) 중 극히 일부만 사용되고 나머지는 놀게 되는 심각한 리소스 낭비가 발생한다. 

&nbsp;

이 문제를 해결하기 위해 NVIDIA가 내놓은 정답이 바로 **Triton Inference Server**다. Triton의 핵심 기능인 **다이나믹 배칭(Dynamic Batching)**이 어떻게 서로 다른 시점의 요청을 찰나의 순간에 하나로 묶어 GPU 가동률을 100%에 가깝게 끌어올리는지, 그 내부 스케줄링 메커니즘을 심층 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 배칭(Batching)의 딜레마: 처리량(Throughput) vs 지연 시간(Latency)

&nbsp;

추론 효율을 높이는 가장 간단한 방법은 요청을 모아서 한꺼번에 처리하는 것이다.
- **배치 크기(Batch Size)가 클수록**: 한 번의 연산으로 많은 사용자를 처리하므로 전체 처리량(Throughput)은 비약적으로 상승한다.
- **문제점**: 배치를 채우기 위해 기다리는 시간(Queueing Delay) 동안 첫 번째 사용자의 대기 시간(Latency)이 길어진다. 

&nbsp;

실시간 서비스에서는 1초의 지연도 치명적이다. Triton은 이 상충하는 가치를 **'최대 대기 시간(Max Queue Delay)'**이라는 파라미터로 조율한다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 다이나믹 배칭 스케줄러의 동작 원리

&nbsp;

Triton의 다이나믹 배칭은 요청이 들어오는 즉시 큐(Queue)에 쌓고, 설정된 기준에 따라 배치를 형성한다.

&nbsp;

## 2-1. 스케줄링 파라미터 제어
Triton 설정 파일(`config.pbtxt`)에서 핵심적인 두 설정은 다음과 같다.
- **`max_batch_size`**: GPU로 한 번에 보낼 수 있는 최대 요청 수.
- **`max_queue_delay_microseconds`**: 배치가 가득 차지 않더라도, 최대 몇 마이크로초(us)까지 기다린 후 GPU로 연산을 명령할 것인지에 대한 임계값.

&nbsp;

## 2-2. 적응형 배치 형성
예를 들어 `max_batch_size`가 8이고 `max_queue_delay`가 100ms라고 가정하자.
1. 첫 번째 요청이 들어오면 타이머가 시작된다.
2. 100ms 이내에 8개의 요청이 모두 차면, 즉시 배치를 형성하여 GPU로 전달한다.
3. 만약 100ms가 지났는데 3개의 요청만 들어왔다면, 더 기다리지 않고 현재 3개만 묶어서 GPU로 보낸다. 

&nbsp;

이 매커니즘 덕분에 트래픽이 적을 때는 낮은 지연 시간을 유지하고, 트래픽이 폭주할 때는 자동으로 배치를 꽉 채워 GPU 파워를 풀가동하는 유연한 대응이 가능해진다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 모델 동시성(Model Concurrency)과 인스턴스 그룹

&nbsp;

Triton은 하나의 GPU 메모리 내에 동일한 모델의 복제본(Instance)을 여러 개 띄울 수 있다. 

&nbsp;

- **Instance Group**: GPU 연산 유닛이 여유롭다면, 첫 번째 배치가 GPU에서 연산 되는 동안 두 번째 배치를 준비하여 바로 다음 사이클에 밀어 넣는 파이프라이닝이 가능하다. 
- **효과**: 이를 통해 하드웨어 가속기(Accelerator)의 유휴 시간(Idle time)을 0에 가깝게 수렴시킬 수 있다. 특히 모델의 크기가 작고 연산량이 적은 경우, 인스턴스 그룹 최적화만으로도 단일 서버의 처리량을 수십 배 이상 높일 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 실전 팁: 모델 아키텍처별 배치 최적화

&nbsp;

- **CNN/RNN 계열**: 고정된 입력 크기를 가지므로 다이나믹 배칭의 효과가 매우 명확하다.
- **LLM (Transformer) 계열**: 입력 문장의 길이가 제각각이므로, 짧은 문장이 긴 문장 때문에 기다려야 하는 **Padding** 문제가 발생한다. 이를 해결하기 위해 Triton은 최근 **In-flight Batching**(연산 도중 완료된 요청을 즉시 빼고 새 요청을 끼워 넣는 기술) 기능을 도입하여 최적화 수준을 한 단계 더 끌어올렸다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: GPU 가성비의 마침표, Triton

&nbsp;

H100 한 장의 가격이 수천만 원을 호가하는 시대에 GPU를 놀리는 것은 엔지니어링적 직무 유기다. 

&nbsp;

사용자의 실시간 경험(Latency)을 보장하면서도 인프라 비용(Throughput)을 최소화하는 최적의 교차점. 그 접점을 수학적 스케줄링으로 찾아주는 Triton Inference Server는 AI 모델 서빙 아키텍처의 중추다. 모델의 성능을 측정할 때 단순히 한 번의 추론 속도만 보지 마라. 다이나믹 배칭을 적용했을 때의 **초당 쿼리 처리량(QPS)**이야말로 비즈니스 성패를 가르는 진짜 지표다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI 서빙 5편] 모델 양자화(Quantization) — FP16에서 INT4로, 성능 하락 없이 용량을 줄이는 수학적 원리**

&nbsp;

100GB가 넘는 모델을 어떻게 1/4 크기로 줄여서 서빙할 수 있을까? 가중치의 정밀도를 낮추는 **Post-Training Quantization (PTQ)**과 학습 과정에서 이를 보정하는 **Quantization-Aware Training (QAT)**의 차이를 분석한다. 소수점 데이터를 정수로 변환할 때 발생하는 오차(Quantization Error)를 최소화하는 아웃라이어 관리 전략과 최신 AWQ, GPTQ 알고리즘의 실체를 파헤친다.

&nbsp;

&nbsp;

---

&nbsp;

Triton, InferenceServer, DynamicBatching, GPU최적화, 모델서빙, 처리량개선, 지연시간단축, AI인프라
