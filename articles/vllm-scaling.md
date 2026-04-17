# LLM 서버에 사람이 몰리면 — 대기열, 스케일 아웃, 오토스케일링

&nbsp;

vLLM으로 AI 모델을 서빙하고 있다.

동시 50명까지 처리 가능하다.

&nbsp;

근데 출근 시간에 200명이 몰린다.

&nbsp;

어떻게 하지?

&nbsp;

&nbsp;

---

&nbsp;

# 1. 아무것도 안 하면

&nbsp;

```
동시 50명:  응답 3초 (정상)
동시 100명: 응답 6초 (느려짐)
동시 200명: 응답 12초 (사용자 이탈)
동시 500명: 타임아웃 (서비스 장애)
```

&nbsp;

vLLM이 내부적으로 대기열은 있다.

50명 넘으면 큐에 넣고 순서대로 처리한다.

&nbsp;

문제는 **대기 시간**이다.

챗봇이 12초 동안 아무 말 안 하면 사용자는 나간다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 방법 1 — 서버 여러 대 (스케일 아웃)

&nbsp;

가장 직관적인 방법. 서버를 늘린다.

&nbsp;

```
vLLM 1대: 동시 50명
vLLM 3대: 동시 150명

Load Balancer (nginx)
  ├→ vLLM 서버 A (GPU 1장)
  ├→ vLLM 서버 B (GPU 1장)
  └→ vLLM 서버 C (GPU 1장)
```

&nbsp;

## nginx 로드밸런싱

&nbsp;

```nginx
upstream vllm_cluster {
    server vllm-1:8000;
    server vllm-2:8000;
    server vllm-3:8000;
}

server {
    listen 8000;

    location / {
        proxy_pass http://vllm_cluster;
        proxy_read_timeout 120s;    # LLM 응답은 오래 걸릴 수 있음
    }
}
```

&nbsp;

## Docker Compose로 구성

&nbsp;

```yaml
services:
  vllm-1:
    image: vllm/vllm-openai:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0']
              capabilities: [gpu]
    command: >
      python -m vllm.entrypoints.openai.api_server
      --model meta-llama/Llama-3-8B-Instruct
      --host 0.0.0.0 --port 8000

  vllm-2:
    image: vllm/vllm-openai:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['1']
              capabilities: [gpu]
    command: >
      python -m vllm.entrypoints.openai.api_server
      --model meta-llama/Llama-3-8B-Instruct
      --host 0.0.0.0 --port 8000

  nginx:
    image: nginx
    ports:
      - "8000:8000"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - vllm-1
      - vllm-2
```

&nbsp;

**장점:** 단순하다. nginx 설정 하나로 끝.

**단점:** 서버가 항상 떠 있어야 한다. 새벽에도 GPU 비용 나간다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 방법 2 — 오토스케일링

&nbsp;

트래픽에 따라 서버가 **자동으로 늘었다 줄었다** 한다.

&nbsp;

```
              요청 수
피크 ───────►  ████
              ████████
              ████████████
평소 ───────►  ██
새벽 ───────►  
              ──────────────► 시간
              06   12   18   24

서버 수:
              1대  4대  2대  0대
```

&nbsp;

## 쿠버네티스 HPA

&nbsp;

```yaml
# deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm
  template:
    metadata:
      labels:
        app: vllm
    spec:
      containers:
        - name: vllm
          image: vllm/vllm-openai:latest
          args:
            - "python"
            - "-m"
            - "vllm.entrypoints.openai.api_server"
            - "--model"
            - "meta-llama/Llama-3-8B-Instruct"
            - "--host"
            - "0.0.0.0"
            - "--port"
            - "8000"
          ports:
            - containerPort: 8000
          resources:
            limits:
              nvidia.com/gpu: 1
```

&nbsp;

```yaml
# hpa.yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm
  minReplicas: 1          # 최소 1대
  maxReplicas: 8          # 최대 8대
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # CPU 70% 넘으면 서버 추가
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # 1분 관찰 후 증설
    scaleDown:
      stabilizationWindowSeconds: 300   # 5분 관찰 후 축소
```

&nbsp;

```
동작 흐름:
1. CPU 사용률 70% 초과 감지
2. 1분 동안 유지되는지 확인
3. Pod 1개 추가 (GPU 서버 1대 증설)
4. 트래픽 줄면 5분 관찰 후 축소
```

&nbsp;

## 스케줄 기반 스케일링

&nbsp;

피크 시간이 예측 가능하면 미리 늘려놓는다.

&nbsp;

```yaml
# 출근 시간 전에 미리 4대로 증설
apiVersion: autoscaling/v1
kind: CronHorizontalPodAutoscaler
metadata:
  name: vllm-schedule
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm
  jobs:
    - name: scale-up-morning
      schedule: "0 8 * * 1-5"     # 평일 오전 8시
      targetSize: 4
    - name: scale-down-evening
      schedule: "0 20 * * *"       # 매일 오후 8시
      targetSize: 1
    - name: scale-down-night
      schedule: "0 0 * * *"        # 자정
      targetSize: 0                # 완전 종료 (비용 0)
```

&nbsp;

**오토스케일링이 반응형이라면, 스케줄은 예방형이다.**

출근 시간에 몰리는 게 확실하면 8시에 미리 늘려놓는 게 낫다.

오토스케일링은 "몰린 다음에" 반응하니까 초반 몇 분은 느릴 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 방법 3 — 대기열 + 우선순위

&nbsp;

서버를 늘려도 한계는 있다. 그때는 **대기열을 직접 관리**한다.

&nbsp;

```typescript
import Bull from 'bull';

// Redis 기반 대기열
const llmQueue = new Bull('llm-requests', {
  redis: { host: 'localhost', port: 6379 },
  limiter: {
    max: 50,        // 동시 처리 50건
    duration: 1000,  // 1초당
  }
});

// 요청 추가
app.post('/api/chat', async (req, res) => {
  const job = await llmQueue.add({
    message: req.body.message,
    userId: req.user.id,
    priority: req.user.plan === 'premium' ? 1 : 10,  // 프리미엄 우선
  });

  // 결과 대기
  const result = await job.finished();
  res.json(result);
});

// 워커: 대기열에서 꺼내서 처리
llmQueue.process(50, async (job) => {
  const response = await fetch('http://vllm:8000/v1/chat/completions', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model: 'meta-llama/Llama-3-8B-Instruct',
      messages: [{ role: 'user', content: job.data.message }],
    }),
  });

  return await response.json();
});
```

&nbsp;

```
일반 유저:    대기열 우선순위 10 (뒤쪽)
프리미엄 유저: 대기열 우선순위 1  (앞쪽)

200명 몰려도:
  프리미엄 → 3초 응답
  일반     → 10초 응답 (대기)
```

&nbsp;

## 스트리밍으로 체감 속도 개선

&nbsp;

대기 시간이 길어도 **글자가 하나씩 나오면** 사용자는 기다린다.

&nbsp;

```typescript
// 스트리밍 응답
app.post('/api/chat/stream', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');

  const response = await fetch('http://vllm:8000/v1/chat/completions', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model: 'meta-llama/Llama-3-8B-Instruct',
      messages: [{ role: 'user', content: req.body.message }],
      stream: true,  // 스트리밍 활성화
    }),
  });

  // 토큰 하나씩 클라이언트로 전달
  for await (const chunk of response.body) {
    res.write(`data: ${chunk}\n\n`);
  }
  res.end();
});
```

&nbsp;

```
스트리밍 없이: [        3초 대기        ] 전체 응답 한 번에 표시
스트리밍:     [안][녕][하][세][요] 글자 하나씩 바로 표시

실제 시간은 같지만, 체감은 완전히 다르다.
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. 방법 비교

&nbsp;

| 방법 | 복잡도 | 비용 | 적합한 경우 |
|:---|:---:|:---:|:---|
| **서버 고정 증설** | 낮음 | 높음 | 트래픽이 일정할 때 |
| **오토스케일링** | 중간 | 중간 | 피크가 있을 때 |
| **스케줄 스케일링** | 낮음 | 낮음 | 피크 시간이 예측 가능할 때 |
| **대기열 + 우선순위** | 중간 | 낮음 | 무료/유료 사용자 분리 |
| **스트리밍** | 낮음 | 없음 | 모든 경우 (항상 적용 권장) |

&nbsp;

&nbsp;

---

&nbsp;

# 6. 비용 비교

&nbsp;

클라우드 GPU (A100 80GB) 기준, 월 비용:

&nbsp;

| 전략 | 평균 서버 수 | 월 비용 |
|:---|:---:|:---|
| 고정 1대 (24시간) | 1.0 | $2,400 (약 343만원) |
| 고정 4대 (24시간) | 4.0 | $9,600 (약 1,373만원) |
| 오토스케일링 | 2.0 | $4,800 (약 686만원) |
| 오토스케일링 + 새벽 축소 | 1.3 | $3,200 (약 458만원) |
| 스케줄 + 오토스케일링 | 1.5 | $3,600 (약 515만원) |

&nbsp;

```
고정 4대 vs 오토스케일링:
  $9,600 → $3,200 = 67% 절감

연간:
  $115,200 → $38,400 = $76,800 절감 (약 1억 983만원)
```

&nbsp;

**같은 성능, 67% 비용 절감.** 오토스케일링을 안 쓸 이유가 없다.

&nbsp;

&nbsp;

---

&nbsp;

# 7. 실전 조합 추천

&nbsp;

```
소규모 (동시 50명 이하):
  vLLM 1대 + 대기열
  = 단순하고 저렴

중규모 (동시 50~200명):
  vLLM 2~4대 + nginx 로드밸런싱 + 스케줄 스케일링
  = 피크 시간에 늘렸다 줄이기

대규모 (동시 200명+):
  K8s + 오토스케일링 + 대기열 + 우선순위 + 스트리밍
  = 풀 구성

모든 규모:
  스트리밍은 무조건 켜라 (비용 0, 체감 속도 향상)
```

&nbsp;

&nbsp;

---

&nbsp;

# 결론

&nbsp;

LLM 서버에 사람이 몰릴 때:

&nbsp;

1. **스트리밍** — 비용 0, 체감 속도 향상 (항상 켜라)
2. **대기열** — 초과 요청을 순서대로 처리
3. **스케일 아웃** — 서버를 늘린다
4. **오토스케일링** — 트래픽에 따라 자동으로 늘었다 줄었다
5. **우선순위** — 중요한 요청을 먼저 처리

&nbsp;

가장 흔한 실수: "피크 기준으로 서버를 고정"

→ 새벽에도 4대가 돌아감 → 연간 1억원 낭비.

&nbsp;

**트래픽은 변하고, 서버도 따라 변해야 한다.**

&nbsp;

&nbsp;

---

LLM, vLLM, 스케일링, 오토스케일링, 로드밸런싱, 대기열, 쿠버네티스, Docker, GPU, nginx, 스트리밍, AI인프라, 비용최적화
