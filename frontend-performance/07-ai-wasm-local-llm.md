# [FE 최적화 7편] 브라우저 안의 AI — WebAssembly(Wasm)와 WebGPU를 활용한 로컬 LLM 렌더링 최적화

&nbsp;

모든 AI 연산을 클라우드 서버에서 처리하는 아키텍처는 막대한 API 비용, 네트워크 지연(Latency), 그리고 데이터 프라이버시라는 세 가지 치명적인 한계를 갖는다. 이를 타개하기 위해 등장한 혁신이 바로 **In-Browser AI (로컬 구동 AI)**다. Llama 3 8B나 Mistral 같은 소형 거대 언어 모델(SLM)을 사용자의 크롬(Chrome) 브라우저 메모리에 직접 올리고 추론(Inference)을 수행하는 것이다.

&nbsp;

하지만 기가바이트(GB) 단위의 모델 가중치(Weights)를 로드하고 초당 수억 번의 행렬 곱 연산을 수행하는 딥러닝 엔진을 자바스크립트 기반의 웹 환경에서 돌리는 것은 자살 행위에 가깝다. 메인 스레드는 즉각적으로 뻗고, UI는 완전히 프리징(Freezing)된다. 본 글에서는 브라우저의 한계를 돌파하기 위해 **WebAssembly(Wasm)**와 **WebGPU**를 병합하여 하드웨어 가속 파이프라인을 구축하고, 메인 스레드를 완벽히 보호하는 극한의 프론트엔드 최적화 아키텍처를 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 자바스크립트 엔진의 패배와 WebAssembly의 구원

&nbsp;

자바스크립트(V8 엔진)는 동적 타입 검사와 가비지 컬렉션(GC)을 수행하는 인터프리터(JIT) 기반 언어다. 딥러닝 추론의 핵심인 거대한 텐서(Tensor) 연산을 처리하기에는 태생적으로 연산 비용이 너무 무겁다.

&nbsp;

## 1-1. Wasm (WebAssembly)의 한계 돌파
Wasm은 C, C++, Rust로 작성된 코드를 브라우저에서 네이티브 수준의 속도로 실행할 수 있게 해주는 바이너리 포맷이다. AI 추론 프레임워크(예: ONNX Runtime, ggml)를 Wasm으로 컴파일하여 브라우저에 이식하면, V8 엔진의 파싱이나 GC 오버헤드를 완벽히 우회하여 메모리에 직접 접근(Linear Memory)하고 CPU 코어를 극한까지 활용할 수 있다.

&nbsp;

하지만 CPU 연산만으로는 LLM의 텍스트 생성 속도(Tokens per second)가 초당 1~2 토큰 수준에 머물러 실사용이 불가능하다. 진정한 병렬 처리 가속이 필요하다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. WebGPU: 브라우저가 그래픽 카드를 지배하다

&nbsp;

과거 WebGL이 그래픽 렌더링에 초점이 맞춰져 있었다면, 최신 웹 표준인 **WebGPU**는 그래픽뿐만 아니라 **GPGPU(General-Purpose computing on GPU)** 연산을 브라우저에서 직접 제어할 수 있게 해주는 게임 체인저다.

&nbsp;

## 2-1. Compute Shader를 통한 병렬 연산
LLM의 추론은 거대한 행렬과 행렬의 곱(MatMul)의 연속이다. WebGPU의 Compute Shader를 활용하면, 브라우저는 사용자 기기의 깡통 GPU(애플 M칩의 Neural Engine, 엔비디아 그래픽카드 등) 코어 수천 개에 직접 명령을 내려 이 행렬 곱 연산을 병렬로 처리한다.
`WebLLM`이나 `MLC LLM` 같은 라이브러리는 내부적으로 모델 가중치를 WebGPU 버퍼(Buffer)에 올리고 연산을 위임함으로써, 웹 환경임에도 불구하고 초당 20~30 토큰 이상의 경이로운 추론 속도를 달성한다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 실전 아키텍처: 메인 스레드 셧다운 방어 로직 (Web Worker & SAB)

&nbsp;

연산이 아무리 빨라져도 이 거대한 코드를 UI를 그리는 **메인 스레드(Main Thread)**에서 돌려버리면, 모델이 텍스트를 생성하는 동안 사용자는 화면 스크롤조차 할 수 없는 Total Blocking Time(TBT) 무한대 상태에 빠진다.

&nbsp;

## 3-1. Web Worker로의 물리적 격리
AI 엔진 로드, 텐서 가공, WebGPU 추론 명령은 모두 `Web Worker`라는 별도의 백그라운드 스레드에서 실행되어야 한다. 메인 스레드는 오직 사용자의 입력을 Worker로 넘겨주고(`postMessage`), Worker가 생성해 낸 토큰 스트림을 받아 화면(DOM)에 페인팅하는 가벼운 역할만 수행한다.

&nbsp;

```javascript
// 메인 스레드 (app.js)
const aiWorker = new Worker('llm-worker.js');

// 텍스트 스트리밍 수신 및 UI 업데이트
aiWorker.onmessage = (e) => {
  const { type, textToken } = e.data;
  if (type === 'chunk') {
    requestAnimationFrame(() => {
      document.getElementById('chat').textContent += textToken;
    });
  }
};

// 질문 전송
aiWorker.postMessage({ prompt: "양자역학을 설명해줘" });
```

&nbsp;

## 3-2. SharedArrayBuffer (SAB)의 메모리 공유
Worker와 메인 스레드 간에 거대한 데이터를 `postMessage`로 주고받으면, 직렬화/역직렬화(Structured Clone Algorithm) 과정에서 엄청난 메모리 복사 오버헤드가 발생한다. 이를 방지하기 위해 **SharedArrayBuffer**를 도입하여, 두 스레드가 메모리 내의 동일한 바이트 배열을 복사 없이 '공유'하도록 설계해야 한다. 단, SAB를 사용하기 위해서는 서버 응답 시 COOP/COEP 보안 헤더 설정이 필수적이다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 메모리 관리와 모델 양자화 (Quantization)

&nbsp;

브라우저는 하나의 탭이 차지할 수 있는 메모리 상한선(통상 2~4GB 내외)이 엄격하게 정해져 있다. 7B 파라미터 모델을 16비트 실수(FP16)로 올리면 14GB가 넘어가므로 브라우저는 OOM(Out of Memory)으로 즉시 크래시된다.

&nbsp;

- **INT4 양자화**: 모델 가중치를 4비트 정수(INT4)로 압축(Quantization)해야 한다. 7B 모델의 용량이 4GB 이하로 줄어들어 브라우저 메모리에 안전하게 안착할 수 있다.
- **가비지 컬렉션 회피**: Wasm 내에서 메모리를 할당할 때는 자바스크립트의 GC를 피하기 위해 수동으로 메모리를 `malloc` 하고 `free` 하는 철저한 C/C++ 수준의 메모리 관리가 요구된다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 웹은 진화하는 운영체제다

&nbsp;

브라우저에서 LLM을 구동한다는 것은 프론트엔드 엔지니어링의 한계선이 폭발적으로 확장되었음을 의미한다. 우리는 더 이상 DOM 트리를 조작하는 스크립터가 아니다. WebGPU로 하드웨어를 직접 제어하고, Web Worker와 SAB로 멀티스레드 병렬 처리를 구현하며, Wasm을 통해 메모리를 바이트 단위로 다루는 시스템 프로그래머의 영역에 진입했다.

&nbsp;

서버 비용 0원, 절대적인 개인정보 보호(Zero-Data-Retention), 완전한 오프라인 작동. 이 세 가지 무기를 가진 In-Browser AI 아키텍처는 다가올 웹 생태계의 패권을 쥐게 될 것이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[FE 최적화 8편] A/B 테스트의 종말 — 강화학습(RL) 기반 MAB(Multi-Armed Bandit) 알고리즘과 동적 렌더링**

&nbsp;

단순히 트래픽을 5:5로 쪼개어 몇 주를 기다렸다가 수동으로 승자를 고르는 멍청한 A/B 테스트는 끝났다. 브라우저 단에서 사용자의 클릭 보상(Reward)을 실시간으로 학습하고, 전환율이 높은 컴포넌트로 렌더링 가중치를 자동으로 이동시키는 강화학습 기반의 Multi-Armed Bandit(MAB) 아키텍처와 이를 지원하는 Edge 렌더링 최적화 기법을 하드코어하게 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

WebGPU, WebAssembly, Wasm, 로컬LLM, WebWorker, 메인스레드해방, 프론트엔드최적화, 양자화
