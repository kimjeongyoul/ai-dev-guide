# [AI Agent 실전 가이드 6편] 인프라 & 스펙 — 뭘 깔아야 하나

&nbsp;

"AI Agent를 만들고 싶은데, 서버 뭐 사야 해요? GPU 필요해요?"

&nbsp;

결론부터 말하면 — **대부분의 경우 GPU가 필요 없다.**

&nbsp;

클라우드 API(OpenAI, Anthropic)를 쓰면 일반 서버에서 Agent를 돌릴 수 있다. GPU가 필요한 건 온프레미스에서 직접 LLM을 돌릴 때뿐이다.

&nbsp;

이 편에서는 "우리 상황에 맞는 인프라"를 판단하는 기준을 정리한다.

&nbsp;

&nbsp;

---

&nbsp;

## 1. GPU가 필요한가?

&nbsp;

### 판단 기준

&nbsp;

```
데이터가 외부로 나가도 되는가?
├── YES → 클라우드 API (GPU 불필요)
│         OpenAI, Anthropic, Google 등
│         서버: 일반 VM (2vCPU/4GB)
│
└── NO  → 온프레미스 LLM (GPU 필요)
          Llama, Mistral 등 오픈소스 모델
          서버: GPU 서버 (NVIDIA A100/H100)
```

&nbsp;

### 현실적인 선택

&nbsp;

| 상황 | 선택 | 이유 |
|------|------|------|
| 스타트업, 빠른 PoC | 클라우드 API | 초기 투자 0, 바로 시작 |
| 일반 기업 | 클라우드 API | 비용 효율, 운영 부담 없음 |
| 금융/의료/공공 | 온프레미스 | 규제 요구사항 |
| 월 API 비용 $3,000+ | 하이브리드 검토 | 손익분기점 |
| 대기업 (자체 인프라 보유) | 온프레미스 | 기존 GPU 활용 |

&nbsp;

&nbsp;

---

&nbsp;

## 2. 클라우드 API 기반 인프라

&nbsp;

대부분의 Agent는 이 구성으로 충분하다.

&nbsp;

```
┌──────────────────────────────────────────────────┐
│              클라우드 API 기반 구성                 │
│                                                   │
│  ┌────────────┐    ┌──────────────────────────┐  │
│  │  Agent 서버  │    │  외부 서비스               │  │
│  │             │───►│                          │  │
│  │  Node.js    │    │  OpenAI / Anthropic API  │  │
│  │  or Python  │◄───│  (LLM 처리)              │  │
│  │             │    └──────────────────────────┘  │
│  │  2vCPU/4GB  │                                  │
│  │             │    ┌──────────────────────────┐  │
│  │             │───►│  벡터DB                   │  │
│  │             │    │  Pinecone (관리형)        │  │
│  │             │    │  or pgvector (자체)       │  │
│  │             │    └──────────────────────────┘  │
│  │             │                                  │
│  │             │    ┌──────────────────────────┐  │
│  │             │───►│  Redis                    │  │
│  │             │    │  대화 컨텍스트 캐시         │  │
│  │             │    └──────────────────────────┘  │
│  └────────────┘                                   │
└──────────────────────────────────────────────────┘
```

&nbsp;

### 서버 스펙

&nbsp;

```yaml
# Agent 서버 (동시 사용자 100명 이하)
server:
  cpu: 2 vCPU
  memory: 4 GB
  storage: 50 GB SSD
  os: Ubuntu 22.04
  runtime: Node.js 20 / Python 3.11
  cost: ~$20/월 (약 28,600원)  # AWS t3.medium 기준

# Agent 서버 (동시 사용자 500명)
server:
  cpu: 4 vCPU
  memory: 8 GB
  storage: 100 GB SSD
  cost: ~$60/월 (약 85,800원)  # AWS t3.xlarge 기준
```

&nbsp;

Agent 서버 자체는 가벼운 편이다. CPU 부하의 대부분은 LLM API로 넘어가기 때문에, Agent 서버는 "요청을 받아서 API를 호출하고 결과를 전달하는" 중계 역할만 한다.

&nbsp;

### 벡터DB 선택

&nbsp;

```yaml
# 옵션 1: Pinecone (관리형 — 추천)
pinecone:
  plan: Starter (무료) / Standard ($70/월, 약 100,100원)
  vectors: 100만 개까지 (Starter)
  장점: 설정 불필요, 자동 스케일링
  단점: 데이터가 외부에 저장됨

# 옵션 2: pgvector (PostgreSQL 확장)
pgvector:
  서버: 기존 PostgreSQL에 확장만 추가
  cost: 추가 비용 없음 (기존 DB 활용)
  장점: 데이터 자체 관리, 기존 인프라 활용
  단점: 대규모에서 성능 한계, 직접 튜닝 필요

# 옵션 3: Chroma (오픈소스)
chroma:
  서버: 별도 서버 또는 Agent 서버에 임베디드
  cost: 서버비만
  장점: 가볍고 간편, 빠른 PoC
  단점: 프로덕션 안정성 검증 부족
```

&nbsp;

### Redis

&nbsp;

```yaml
# 대화 컨텍스트 캐시 + 세션 관리
redis:
  용도:
    - 대화 히스토리 캐시 (TTL: 30분)
    - Agent 상태 관리
    - Rate Limiting
    - 작업 큐 (Bull/BullMQ)
  
  스펙:
    memory: 1~2 GB (사용자 1,000명 기준)
    cost: ~$15/월 (약 21,450원)  # AWS ElastiCache t3.micro
    
  # 또는 Docker로 직접 운영
  # docker run -d --name redis -p 6379:6379 redis:7-alpine
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. 온프레미스 LLM 인프라

&nbsp;

데이터가 사내에서만 처리되어야 하는 경우.

&nbsp;

### GPU 선택 가이드

&nbsp;

```
모델 크기별 필요 VRAM:

┌─────────────┬────────────┬──────────────┬──────────────────┐
│ 모델 크기    │ 필요 VRAM   │ 추천 GPU      │ GPU 가격          │
├─────────────┼────────────┼──────────────┼──────────────────┤
│ 7B (소형)   │ 8 GB       │ RTX 4060 Ti  │ $400 (57만원)     │
│ 13B (중형)  │ 16 GB      │ RTX 4090     │ $1,600 (229만원)  │
│ 34B (대형)  │ 40 GB      │ A100 40GB    │ $10,000 (1,430만원)│
│ 70B (초대형)│ 80 GB      │ A100 80GB ×2 │ $30,000 (4,290만원)│
│ 405B       │ 320 GB     │ H100 ×4      │ $120,000+ (1.7억원)│
└─────────────┴────────────┴──────────────┴──────────────────┘

* 양자화(4-bit)를 적용하면 필요 VRAM이 약 1/4로 줄어듦
  예: 70B 모델 4-bit → ~20GB → A100 40GB 1장으로 가능
```

&nbsp;

### 모델 서빙: Ollama vs vLLM

&nbsp;

```bash
# Ollama: 간편한 로컬 실행 (개발/소규모)
ollama pull llama3:70b
ollama serve
# → http://localhost:11434 에서 API 제공

# API 호출
curl http://localhost:11434/api/chat -d '{
  "model": "llama3:70b",
  "messages": [{"role": "user", "content": "서버 에러를 분석해줘"}],
  "stream": false
}'
```

&nbsp;

```bash
# vLLM: 프로덕션 수준 서빙 (고성능, 대규모)
pip install vllm

# 서버 시작
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3-70B-Instruct \
  --tensor-parallel-size 2 \
  --gpu-memory-utilization 0.9 \
  --port 8000

# OpenAI 호환 API로 호출 가능
curl http://localhost:8000/v1/chat/completions -d '{
  "model": "meta-llama/Llama-3-70B-Instruct",
  "messages": [{"role": "user", "content": "서버 에러를 분석해줘"}]
}'
```

&nbsp;

### Ollama vs vLLM 비교

&nbsp;

| 항목 | Ollama | vLLM |
|------|--------|------|
| 설치 | 1분 | 10분 |
| 설정 | 거의 없음 | 튜닝 필요 |
| 성능 | 보통 | 높음 (2~3배 빠름) |
| 동시 처리 | 제한적 | 뛰어남 (Continuous Batching) |
| GPU 활용률 | 보통 | 최적화됨 |
| 적합한 용도 | 개발, PoC, 소규모 | 프로덕션, 대규모 |

&nbsp;

&nbsp;

---

&nbsp;

## 4. 하이브리드 구성

&nbsp;

민감 데이터는 온프레미스, 나머지는 클라우드.

가장 현실적인 기업 구성.

&nbsp;

```
┌──────────────────────────────────────────────────────────────┐
│                    하이브리드 아키텍처                          │
│                                                               │
│  ┌─────────────────────────┐   ┌────────────────────────────┐ │
│  │      사내 (온프레미스)     │   │     클라우드                 │ │
│  │                          │   │                            │ │
│  │  ┌────────────────────┐  │   │  ┌──────────────────────┐  │ │
│  │  │  보안 Agent         │  │   │  │  일반 Agent            │  │ │
│  │  │  (Ollama + Llama)  │  │   │  │  (Claude API)         │  │ │
│  │  │                    │  │   │  │                        │  │ │
│  │  │  처리 대상:         │  │   │  │  처리 대상:             │  │ │
│  │  │  - 개인정보 포함     │  │   │  │  - 일반 고객 상담       │  │ │
│  │  │  - 재무 데이터      │  │   │  │  - 문서 요약            │  │ │
│  │  │  - 의료 기록        │  │   │  │  - 코드 리뷰            │  │ │
│  │  │  - 법적 문서        │  │   │  │  - 공개 데이터 분석      │  │ │
│  │  └────────────────────┘  │   │  └──────────────────────┘  │ │
│  │                          │   │                            │ │
│  │  ┌────────────────────┐  │   │  ┌──────────────────────┐  │ │
│  │  │  벡터DB (pgvector)  │  │   │  │  벡터DB (Pinecone)    │  │ │
│  │  │  사내 민감 문서      │  │   │  │  공개 FAQ, 매뉴얼      │  │ │
│  │  └────────────────────┘  │   │  └──────────────────────┘  │ │
│  └─────────────────────────┘   └────────────────────────────┘ │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    라우터 (Agent Router)                   │ │
│  │  요청 분석 → 민감 데이터 포함? → 온프레미스 / 클라우드 분기  │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

&nbsp;

```typescript
// 라우터: 요청 내용에 따라 모델 분기
async function routeRequest(message: string, metadata: RequestMetadata): Promise<LLMResponse> {
  const containsPII = detectPII(message);
  const isFinancial = metadata.department === 'finance';
  const isConfidential = metadata.classification === 'confidential';

  if (containsPII || isFinancial || isConfidential) {
    // 온프레미스 모델 사용
    return await callOllama({
      model: 'llama3:70b',
      messages: [{ role: 'user', content: message }],
    });
  } else {
    // 클라우드 API 사용
    return await callAnthropic({
      model: 'claude-sonnet-4-20250514',
      messages: [{ role: 'user', content: message }],
    });
  }
}

function detectPII(text: string): boolean {
  const patterns = [
    /\d{6}-\d{7}/,           // 주민등록번호
    /\d{3}-\d{4}-\d{4}/,     // 전화번호
    /\d{4}-\d{4}-\d{4}-\d{4}/, // 카드번호
  ];
  return patterns.some(p => p.test(text));
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 5. Docker Compose 구성

&nbsp;

### 기본 구성: Agent + 벡터DB + Redis

&nbsp;

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Agent 서버
  agent:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - REDIS_URL=redis://redis:6379
      - CHROMA_URL=http://chromadb:8000
      - POSTGRES_URL=postgresql://agent:password@postgres:5432/agent_db
    depends_on:
      - redis
      - chromadb
      - postgres
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G

  # Redis (대화 캐시 + 작업 큐)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --maxmemory 512mb
    restart: unless-stopped

  # ChromaDB (벡터DB)
  chromadb:
    image: chromadb/chroma:latest
    ports:
      - "8000:8000"
    volumes:
      - chroma_data:/chroma/chroma
    environment:
      - ANONYMIZED_TELEMETRY=false
    restart: unless-stopped

  # PostgreSQL + pgvector (대안 벡터DB + 일반 DB)
  postgres:
    image: pgvector/pgvector:pg16
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=agent
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=agent_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped

volumes:
  redis_data:
  chroma_data:
  postgres_data:
```

&nbsp;

### Agent Dockerfile

&nbsp;

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine AS runner

WORKDIR /app
RUN addgroup -g 1001 -S agent && adduser -S agent -u 1001
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER agent
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

&nbsp;

### 온프레미스 LLM 포함 구성

&nbsp;

```yaml
# docker-compose.gpu.yml — GPU 서버용
version: '3.8'

services:
  agent:
    # ... (위와 동일)
    environment:
      - LLM_URL=http://ollama:11434  # 로컬 LLM 사용

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped

  # Ollama 초기 모델 다운로드
  ollama-setup:
    image: curlimages/curl:latest
    depends_on:
      - ollama
    entrypoint: >
      sh -c "sleep 10 && 
             curl -X POST http://ollama:11434/api/pull -d '{\"name\": \"llama3:70b\"}'"

volumes:
  ollama_data:
```

&nbsp;

&nbsp;

---

&nbsp;

## 6. Kubernetes 구성 (대규모)

&nbsp;

동시 사용자 1,000명 이상, 여러 Agent가 동시에 돌아가는 경우.

&nbsp;

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-agent
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ai-agent
  template:
    metadata:
      labels:
        app: ai-agent
    spec:
      containers:
        - name: agent
          image: your-registry/ai-agent:latest
          ports:
            - containerPort: 3000
          env:
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: ai-agent-secrets
                  key: anthropic-api-key
            - name: REDIS_URL
              value: redis://redis-master:6379
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 4Gi
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ai-agent
spec:
  selector:
    app: ai-agent
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-agent-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    nginx.ingress.kubernetes.io/websocket-services: "ai-agent"
spec:
  rules:
    - host: agent.your-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ai-agent
                port:
                  number: 80

---
# k8s/hpa.yaml (수평 자동 확장)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-agent-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-agent
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

&nbsp;

&nbsp;

---

&nbsp;

## 7. 스펙 가이드표: 유형별 권장 인프라

&nbsp;

```
┌──────────────────┬──────────────┬────────────┬──────────────┬──────────────┐
│ 유형              │ 서버          │ 벡터DB      │ Redis        │ 월 비용 (USD) │
├──────────────────┼──────────────┼────────────┼──────────────┼──────────────┤
│ PoC / 테스트      │ 1vCPU/2GB    │ Chroma     │ 불필요       │ $10          │
│                  │              │ (임베디드)   │              │ (14,300원)   │
├──────────────────┼──────────────┼────────────┼──────────────┼──────────────┤
│ 소규모            │ 2vCPU/4GB    │ Pinecone   │ 1GB          │ $50~100      │
│ (사용자 ~100)     │              │ Starter    │              │ (7~14만원)   │
├──────────────────┼──────────────┼────────────┼──────────────┼──────────────┤
│ 중규모            │ 4vCPU/8GB    │ Pinecone   │ 2GB          │ $200~400     │
│ (사용자 ~500)     │ ×2 (HA)     │ Standard   │              │ (29~57만원)  │
├──────────────────┼──────────────┼────────────┼──────────────┼──────────────┤
│ 대규모            │ K8s 클러스터  │ pgvector   │ 4GB+         │ $500~2,000   │
│ (사용자 1,000+)   │ 3+ 노드     │ (자체)      │ (클러스터)    │ (72~286만원) │
├──────────────────┼──────────────┼────────────┼──────────────┼──────────────┤
│ 온프레미스 LLM    │ GPU 서버     │ pgvector   │ 2GB          │ GPU 구매비    │
│ (보안 요구)       │ A100 1~2장   │            │              │ + 전기료      │
└──────────────────┴──────────────┴────────────┴──────────────┴──────────────┘

* 위 비용은 인프라만. LLM API 비용은 별도. (7편에서 상세 다룸)
```

&nbsp;

&nbsp;

---

&nbsp;

## 8. 체크리스트: 인프라 결정 전 확인 사항

&nbsp;

```
□ 데이터 보안 요구사항은? (클라우드 API 가능 / 온프레미스 필수)
□ 동시 사용자 수는? (서버 스펙 결정)
□ RAG가 필요한가? (벡터DB 필요 여부)
□ 대화형인가? (Redis 캐시 필요 여부)
□ 고가용성이 필요한가? (이중화 / K8s)
□ 기존 인프라가 있는가? (PostgreSQL → pgvector 활용)
□ GPU가 있는가? (온프레미스 LLM 가능 여부)
□ 운영 인력이 있는가? (관리형 vs 자체 운영)
```

&nbsp;

&nbsp;

---

&nbsp;

## 9. 정리

&nbsp;

| 질문 | 답변 |
|------|------|
| GPU 필요한가? | 클라우드 API 쓰면 불필요 |
| 서버 스펙은? | 2vCPU/4GB로 시작, 필요 시 확장 |
| 벡터DB는? | Pinecone(편함) 또는 pgvector(자체 관리) |
| Redis는? | 대화형 Agent면 필요 |
| Docker? | 필수. 배포 단위 |
| Kubernetes? | 사용자 1,000명 이상이면 검토 |
| 온프레미스 LLM? | 보안 요구사항이 있을 때만 |

&nbsp;

**대부분의 Agent는 "클라우드 API + 일반 서버 + Docker"로 시작하면 된다.**

온프레미스 LLM은 보안 요구사항이 있거나, API 비용이 월 $3,000을 넘을 때 검토하면 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI Agent 실전 가이드 7편] 비용 산정 — API 호출부터 서버까지**

&nbsp;

"Agent 돌리면 한 달에 얼마나 드는가?" 모델별 API 비용 비교, 유형별 월 비용 산정, 온프레미스와 클라우드의 손익분기점, 비용 최적화 팁을 숫자로 정리한다.

&nbsp;

&nbsp;

---

&nbsp;

AI에이전트, 인프라, GPU, 온프레미스, 클라우드API, Docker, Kubernetes, Ollama, vLLM, 벡터DB, Pinecone, pgvector, Redis, 서버스펙
