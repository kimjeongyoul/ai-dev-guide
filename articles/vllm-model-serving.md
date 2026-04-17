# 온프레미스 LLM 서빙 — Ollama vs vLLM, 프로덕션에서 AI 모델을 돌린다는 것

&nbsp;

ChatGPT, Claude는 API를 호출하면 된다.

돈 내고 요청 보내면 답이 온다. 간단하다.

&nbsp;

근데 이런 상황이 있다.

&nbsp;

- "사내 데이터가 외부 서버로 나가면 안 된다"
- "API 비용이 월 500만원이 넘는다"
- "인터넷 없는 환경에서도 돌아가야 한다"

&nbsp;

이때 **직접 AI 모델을 서버에 올려서 돌린다.** 이걸 "모델 서빙"이라고 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 모델 서빙이란

&nbsp;

```
클라우드 API 방식:
  내 서버 → HTTP 요청 → OpenAI/Anthropic 서버 → 응답
  = 남의 GPU를 빌려 쓴다

모델 서빙 방식:
  내 서버 (GPU 장착) → 모델 로드 → 직접 처리 → 응답
  = 내 GPU에서 직접 돌린다
```

&nbsp;

모델 서빙 = **AI 모델을 내 서버에 올려놓고, API처럼 요청을 받아 응답하는 것.**

&nbsp;

&nbsp;

---

&nbsp;

# 2. Ollama vs vLLM

&nbsp;

둘 다 모델을 서빙하는 도구인데, 성격이 완전히 다르다.

&nbsp;

```
Ollama = 혼자 쓰는 로컬 PC용 (개발/테스트)
vLLM   = 여러 사람이 동시에 쓰는 서버용 (프로덕션)
```

&nbsp;

## Ollama — 5분 만에 시작

&nbsp;

```bash
# 설치
curl -fsSL https://ollama.com/install.sh | sh

# 모델 다운로드 + 실행
ollama run llama3

# API 서버로 실행
ollama serve
# → http://localhost:11434
```

&nbsp;

```bash
# API 호출
curl http://localhost:11434/api/generate \
  -d '{"model": "llama3", "prompt": "Hello, how are you?"}'
```

&nbsp;

**장점:** 설치가 5분. 명령어 하나로 끝.

**한계:** 동시 요청 처리 못 함.

&nbsp;

```
Ollama에 3명이 동시에 요청하면:

사용자 A 요청 → 처리 (3초)
사용자 B 요청 → 대기... → 처리 (3초)
사용자 C 요청 → 대기... → 대기... → 처리 (3초)

= 한 번에 하나씩 처리, C는 9초 기다림
```

&nbsp;

개발자 혼자 테스트할 때는 상관없다.

서비스에 붙이면 문제다.

&nbsp;

## vLLM — 프로덕션용 서빙 엔진

&nbsp;

```bash
# 설치
pip install vllm

# 서버 시작
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3-8B-Instruct \
  --port 8000
# → http://localhost:8000 (OpenAI 호환 API)
```

&nbsp;

```bash
# OpenAI SDK로 바로 호출 가능 (호환 API)
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3-8B-Instruct",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

&nbsp;

**같은 3명이 동시에 요청하면:**

```
사용자 A 요청 ─┐
사용자 B 요청 ─┼→ GPU에서 동시 처리 (3.5초)
사용자 C 요청 ─┘

= 3명 다 3.5초 만에 응답
```

&nbsp;

어떻게 이게 가능한가?

&nbsp;

&nbsp;

---

&nbsp;

# 3. vLLM이 빠른 이유

&nbsp;

## Continuous Batching

&nbsp;

```
일반 배칭:
  요청 3개 모임 → 한 번에 처리 → 3개 다 끝나면 다음 배치
  = 빨리 끝난 요청도 느린 요청이 끝날 때까지 대기

Continuous Batching (vLLM):
  요청 A 처리 중... → B 도착 → 바로 합류 → A 끝남 → C 도착 → 바로 합류
  = 끝난 자리에 새 요청이 바로 들어감
```

&nbsp;

비유: 일반 배칭은 "4인석에 3명 앉으면 1자리 빌 때까지 다음 손님 대기." 
Continuous Batching은 "한 명 나가면 바로 다음 손님 착석."

&nbsp;

## PagedAttention

&nbsp;

LLM은 토큰을 생성할 때 **이전 토큰들의 정보(KV Cache)**를 GPU 메모리에 저장한다.

&nbsp;

```
일반 방식:
  요청마다 최대 길이(예: 4096토큰)만큼 메모리 미리 할당
  → 실제로 100토큰만 써도 4096토큰분 메모리 차지
  → GPU 메모리 낭비 60~80%

PagedAttention (vLLM):
  OS의 가상 메모리처럼 "페이지" 단위로 필요한 만큼만 할당
  → 100토큰이면 100토큰분만 차지
  → GPU 메모리 낭비 거의 0%
  → 같은 GPU에 더 많은 요청을 동시 처리
```

&nbsp;

```
같은 GPU (A100 80GB) 기준:

일반 서빙: 동시 4~8명 처리
vLLM:      동시 20~50명 처리

→ 같은 하드웨어에서 3~10배 처리량
```

&nbsp;

## Tensor Parallelism

&nbsp;

모델이 GPU 1장에 안 들어갈 때:

&nbsp;

```
Llama 70B 모델:
  필요 메모리: ~140GB
  A100 80GB 1장: 안 들어감

Tensor Parallelism:
  GPU 0: 모델의 앞쪽 절반
  GPU 1: 모델의 뒤쪽 절반
  → 2장으로 나눠서 병렬 처리
```

&nbsp;

```bash
# vLLM에서 GPU 2장 사용
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3-70B-Instruct \
  --tensor-parallel-size 2 \
  --port 8000
```

&nbsp;

&nbsp;

---

&nbsp;

# 4. 모델 크기별 필요 스펙

&nbsp;

| 모델 | 파라미터 | 필요 VRAM | 권장 GPU | 동시 처리 (vLLM) |
|:---|:---|:---|:---|:---|
| Llama 3 8B | 80억 | 16GB | RTX 4090 1장 | 20~30명 |
| Mistral 7B | 70억 | 14GB | RTX 4090 1장 | 25~35명 |
| Llama 3 70B | 700억 | 140GB | A100 80GB × 2장 | 10~20명 |
| Mixtral 8x7B | 470억 | 90GB | A100 80GB × 2장 | 15~25명 |
| Llama 3 405B | 4050억 | 810GB | H100 80GB × 8장 | 5~15명 |

&nbsp;

**양자화(Quantization)하면 VRAM을 절반으로 줄일 수 있다:**

```
Llama 3 70B:
  FP16 (원본):  140GB → A100 × 2장
  INT8 (양자화): 70GB  → A100 × 1장
  INT4 (양자화): 35GB  → A6000 × 1장

성능은 5~10% 하락하지만, 비용은 절반.
```

&nbsp;

```bash
# vLLM에서 양자화 모델 사용
python -m vllm.entrypoints.openai.api_server \
  --model TheBloke/Llama-3-70B-GPTQ \
  --quantization gptq \
  --port 8000
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. 실전 구성

&nbsp;

## Docker로 vLLM 배포

&nbsp;

```dockerfile
# Dockerfile
FROM vllm/vllm-openai:latest

ENV MODEL_NAME=meta-llama/Llama-3-8B-Instruct

CMD ["python", "-m", "vllm.entrypoints.openai.api_server", \
     "--model", "${MODEL_NAME}", \
     "--host", "0.0.0.0", \
     "--port", "8000"]
```

&nbsp;

```yaml
# docker-compose.yml
services:
  vllm:
    image: vllm/vllm-openai:latest
    ports:
      - "8000:8000"
    volumes:
      - ./models:/root/.cache/huggingface  # 모델 캐시
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    command: >
      python -m vllm.entrypoints.openai.api_server
      --model meta-llama/Llama-3-8B-Instruct
      --host 0.0.0.0
      --port 8000
      --max-model-len 4096
```

&nbsp;

```bash
docker compose up -d
# → http://localhost:8000/v1/chat/completions
```

&nbsp;

## 앱에서 호출

&nbsp;

vLLM은 OpenAI 호환 API를 제공하니까, **기존 OpenAI SDK 코드를 그대로 쓸 수 있다.**

&nbsp;

```typescript
// baseURL만 바꾸면 됨
import OpenAI from 'openai';

const ai = new OpenAI({
  baseURL: 'http://localhost:8000/v1',  // vLLM 서버
  apiKey: 'not-needed',                  // 로컬이라 키 불필요
});

const response = await ai.chat.completions.create({
  model: 'meta-llama/Llama-3-8B-Instruct',
  messages: [{ role: 'user', content: '안녕하세요' }],
});
```

&nbsp;

```python
# Python도 동일
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed",
)

response = client.chat.completions.create(
    model="meta-llama/Llama-3-8B-Instruct",
    messages=[{"role": "user", "content": "안녕하세요"}],
)
```

&nbsp;

**OpenAI에서 vLLM으로 전환할 때 바꿀 것: baseURL 1줄.**

&nbsp;

&nbsp;

---

&nbsp;

# 6. 비용 비교

&nbsp;

월 10만 건 요청 기준 (평균 1,000토큰/건):

&nbsp;

| 방식 | 월 비용 | 비고 |
|:---|:---|:---|
| **OpenAI GPT-4o** | $350 (약 50만원) | 입력+출력 토큰 비용 |
| **Anthropic Claude Sonnet** | $450 (약 64만원) | 입력+출력 토큰 비용 |
| **vLLM + RTX 4090** | $150 (약 21만원) | 전기세 + 서버 유지비 (GPU 보유 시) |
| **vLLM + A100 (클라우드)** | $800 (약 114만원) | AWS p4d.xlarge 온디맨드 |
| **Ollama + RTX 4090** | $150 (약 21만원) | 동시 1명만 가능 |

&nbsp;

## 손익분기점

&nbsp;

```
클라우드 API: 사용량에 비례 (쓸수록 비쌈)
온프레미스:   고정 비용 (GPU 구매/임대)

                비용
                 ↑
  클라우드 API   /
                /
               /
              /        ────── 온프레미스 (고정)
             /
            /
           ──────────────────→ 사용량

교차점 ≈ 월 $500~$1,000 (약 70~140만원)
```

&nbsp;

- 월 $500 이하: 클라우드 API가 저렴
- 월 $500~$1,000: 비슷 (관리 비용 고려)
- 월 $1,000 이상: 온프레미스가 저렴

&nbsp;

&nbsp;

---

&nbsp;

# 7. Ollama vs vLLM 최종 비교

&nbsp;

| | Ollama | vLLM |
|:---|:---|:---|
| **설치** | 1분 | 10분 |
| **난이도** | 쉬움 | 중간 |
| **동시 처리** | 1명 | 20~50명 |
| **처리량** | 낮음 | 높음 (3~10배) |
| **메모리 효율** | 보통 | 높음 (PagedAttention) |
| **멀티 GPU** | 제한적 | 네이티브 지원 |
| **API 호환** | 자체 API | OpenAI 호환 |
| **적합한 용도** | 개발, 테스트, 개인용 | 서비스, 팀용, 프로덕션 |

&nbsp;

```
판단 기준:

나 혼자 쓴다 → Ollama
팀이 쓴다 → vLLM
서비스에 붙인다 → vLLM
개발/테스트만 → Ollama
비용 최적화 필요 → vLLM
빨리 시작해야 → Ollama → 나중에 vLLM으로 전환
```

&nbsp;

&nbsp;

---

&nbsp;

# 8. 다른 서빙 도구들

&nbsp;

| 도구 | 특징 |
|:---|:---|
| **vLLM** | 가장 빠른 처리량, PagedAttention |
| **Ollama** | 가장 쉬운 설치, 개인용 |
| **TGI (Text Generation Inference)** | HuggingFace 공식, 안정적 |
| **TensorRT-LLM** | NVIDIA 공식, GPU 최적화 극대 |
| **llama.cpp** | CPU에서도 돌림, 경량 |
| **LocalAI** | OpenAI 호환 드롭인 대체, 다양한 모델 |

&nbsp;

시작은 Ollama, 프로덕션은 vLLM이 현재 가장 많이 쓰이는 조합이다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론

&nbsp;

"프로덕션 수준의 모델 서빙" = **여러 사용자의 요청을 동시에, 빠르게, 안정적으로 처리하는 것.**

&nbsp;

Ollama는 식당에 셰프 1명이 주문 하나씩 요리하는 것이고,

vLLM은 같은 셰프가 주문 10개를 동시에 프라이팬에 올리는 것이다.

&nbsp;

같은 GPU인데 **처리 방식이 달라서** 처리량이 3~10배 차이난다.

&nbsp;

- 데이터 보안 때문에 → 온프레미스
- 비용 때문에 → 월 $1,000 넘으면 온프레미스 고려
- 성능 때문에 → vLLM + 적절한 GPU
- 빠른 시작 → Ollama로 검증 → vLLM으로 전환

&nbsp;

&nbsp;

---

LLM, vLLM, Ollama, 모델서빙, 온프레미스, GPU, PagedAttention, 양자화, Docker, AI인프라, Llama, 프로덕션, 비용최적화
