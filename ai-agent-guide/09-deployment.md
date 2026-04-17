# [AI Agent 실전 가이드 9편] 배포 & 운영 — Docker, 모니터링, 장애 대응

&nbsp;

Agent를 만들었다. 로컬에서 잘 돌아간다.

&nbsp;

그런데 **서비스로 운영**하는 건 전혀 다른 문제다.

&nbsp;

- 서버가 재시작되면? Agent도 자동으로 올라와야 한다.
- 트래픽이 몰리면? 자동으로 확장되어야 한다.
- AI가 환각을 일으키면? 감지하고 알림이 와야 한다.
- 장애가 나면? 대응 절차가 있어야 한다.

&nbsp;

이 편에서는 Agent를 **프로덕션에 올리고 안정적으로 운영하는** 전 과정을 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

## 1. Docker 기반 배포

&nbsp;

### 1-1. Dockerfile: Node.js Agent

&nbsp;

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

# 의존성 설치 (캐시 활용)
COPY package*.json ./
RUN npm ci

# 소스 복사 및 빌드
COPY . .
RUN npm run build

# 프로덕션 이미지
FROM node:20-alpine AS runner

WORKDIR /app

# 보안: non-root 사용자
RUN addgroup -g 1001 -S agent && \
    adduser -S agent -u 1001 -G agent

# 필요한 파일만 복사
COPY --from=builder --chown=agent:agent /app/dist ./dist
COPY --from=builder --chown=agent:agent /app/node_modules ./node_modules
COPY --from=builder --chown=agent:agent /app/package.json ./

USER agent

EXPOSE 3000

# 헬스체크
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

&nbsp;

### 1-2. Dockerfile: Python Agent

&nbsp;

```dockerfile
# Dockerfile.python
FROM python:3.11-slim AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim AS runner

WORKDIR /app

RUN useradd -m -u 1001 agent

COPY --from=builder /root/.local /home/agent/.local
COPY --chown=agent:agent . .

USER agent
ENV PATH=/home/agent/.local/bin:$PATH

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

&nbsp;

### 1-3. docker-compose.yml: 전체 스택

&nbsp;

```yaml
# docker-compose.yml
version: '3.8'

services:
  # AI Agent 서버
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
      - DATABASE_URL=postgresql://agent:${DB_PASSWORD}@postgres:5432/agent_db
      - CHROMA_URL=http://chromadb:8000
      - LOG_LEVEL=info
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: >
      redis-server
      --appendonly yes
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped

  # PostgreSQL + pgvector
  postgres:
    image: pgvector/pgvector:pg16
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=agent
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=agent_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U agent"]
      interval: 10s
      timeout: 5s
      retries: 3
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

  # Prometheus (메트릭 수집)
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    restart: unless-stopped

  # Grafana (대시보드)
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/dashboards:/etc/grafana/provisioning/dashboards
    depends_on:
      - prometheus
    restart: unless-stopped

volumes:
  redis_data:
  postgres_data:
  chroma_data:
  prometheus_data:
  grafana_data:
```

&nbsp;

&nbsp;

---

&nbsp;

## 2. Kubernetes 배포 (대규모)

&nbsp;

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ai-agent

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: agent-secrets
  namespace: ai-agent
type: Opaque
stringData:
  anthropic-api-key: "sk-ant-xxx"
  db-password: "your-password"

---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-agent
  namespace: ai-agent
  labels:
    app: ai-agent
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: ai-agent
  template:
    metadata:
      labels:
        app: ai-agent
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: agent
          image: your-registry/ai-agent:v1.2.0
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: ANTHROPIC_API_KEY
              valueFrom:
                secretKeyRef:
                  name: agent-secrets
                  key: anthropic-api-key
            - name: REDIS_URL
              value: "redis://redis-master.ai-agent:6379"
            - name: NODE_ENV
              value: "production"
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
            failureThreshold: 3

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-agent-hpa
  namespace: ai-agent
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. CI/CD 파이프라인

&nbsp;

```yaml
# .github/workflows/deploy.yml
name: Deploy AI Agent

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'Dockerfile'
      - 'package*.json'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/ai-agent

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run test
      # Agent 전용 테스트: 프롬프트 검증
      - run: npm run test:prompts

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to Staging
        run: |
          kubectl set image deployment/ai-agent \
            agent=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n ai-agent-staging
          kubectl rollout status deployment/ai-agent \
            -n ai-agent-staging --timeout=300s

      - name: Smoke Test
        run: |
          # Agent 헬스체크
          curl -f https://staging-agent.your-domain.com/health
          # 간단한 기능 테스트
          curl -f -X POST https://staging-agent.your-domain.com/api/test \
            -H "Content-Type: application/json" \
            -d '{"message": "테스트 질문입니다"}'

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # 수동 승인 필요
    steps:
      - name: Deploy to Production
        run: |
          kubectl set image deployment/ai-agent \
            agent=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n ai-agent
          kubectl rollout status deployment/ai-agent \
            -n ai-agent --timeout=300s

      - name: Verify Deployment
        run: |
          # 배포 후 5분간 에러율 모니터링
          sleep 300
          ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=agent_error_rate_5m" \
            | jq '.data.result[0].value[1]')
          if (( $(echo "$ERROR_RATE > 5" | bc -l) )); then
            echo "에러율 높음: ${ERROR_RATE}%. 롤백 실행."
            kubectl rollout undo deployment/ai-agent -n ai-agent
            exit 1
          fi
```

&nbsp;

&nbsp;

---

&nbsp;

## 4. Agent 전용 모니터링

&nbsp;

일반 서버 메트릭 외에 Agent만의 메트릭이 필요하다.

&nbsp;

### 4-1. Agent 메트릭

&nbsp;

```typescript
// metrics.ts
import { Counter, Histogram, Gauge, Registry } from 'prom-client';

const register = new Registry();

// 1. LLM 호출 메트릭
const llmCallsTotal = new Counter({
  name: 'agent_llm_calls_total',
  help: 'Total LLM API calls',
  labelNames: ['model', 'status'],
  registers: [register],
});

const llmLatency = new Histogram({
  name: 'agent_llm_latency_seconds',
  help: 'LLM API response time',
  labelNames: ['model'],
  buckets: [0.5, 1, 2, 5, 10, 30],
  registers: [register],
});

// 2. 토큰 사용량
const tokensUsed = new Counter({
  name: 'agent_tokens_total',
  help: 'Total tokens used',
  labelNames: ['model', 'type'],  // type: input | output
  registers: [register],
});

// 3. 도구 사용 메트릭
const toolCallsTotal = new Counter({
  name: 'agent_tool_calls_total',
  help: 'Total tool calls',
  labelNames: ['tool', 'status'],
  registers: [register],
});

// 4. 에러율
const agentErrors = new Counter({
  name: 'agent_errors_total',
  help: 'Total agent errors',
  labelNames: ['type'],  // llm_error, tool_error, timeout, validation
  registers: [register],
});

// 5. 환각 감지 (중요!)
const hallucinationDetected = new Counter({
  name: 'agent_hallucination_detected_total',
  help: 'Detected hallucination count',
  labelNames: ['severity'],  // low, medium, high
  registers: [register],
});

// 6. 비용 추적
const costAccumulated = new Counter({
  name: 'agent_cost_dollars_total',
  help: 'Accumulated API cost in dollars',
  labelNames: ['model'],
  registers: [register],
});

// 7. 활성 대화 수
const activeConversations = new Gauge({
  name: 'agent_active_conversations',
  help: 'Number of active conversations',
  registers: [register],
});

// 메트릭 수집 래퍼
async function callLLMWithMetrics(params: LLMParams): Promise<LLMResponse> {
  const timer = llmLatency.startTimer({ model: params.model });

  try {
    const response = await callLLM(params);

    llmCallsTotal.inc({ model: params.model, status: 'success' });
    tokensUsed.inc({ model: params.model, type: 'input' }, response.usage.inputTokens);
    tokensUsed.inc({ model: params.model, type: 'output' }, response.usage.outputTokens);
    costAccumulated.inc({ model: params.model }, calculateCost(response.usage));

    // 환각 감지
    const hallucination = detectHallucination(response.content, params.context);
    if (hallucination) {
      hallucinationDetected.inc({ severity: hallucination.severity });
    }

    timer();
    return response;
  } catch (error) {
    llmCallsTotal.inc({ model: params.model, status: 'error' });
    agentErrors.inc({ type: 'llm_error' });
    timer();
    throw error;
  }
}

// Prometheus 엔드포인트
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

&nbsp;

### 4-2. 환각 감지

&nbsp;

```typescript
// hallucination-detector.ts

interface HallucinationResult {
  detected: boolean;
  severity: 'low' | 'medium' | 'high';
  reason: string;
}

function detectHallucination(
  response: string,
  context: { tools: ToolResult[]; rag: string[] }
): HallucinationResult | null {
  // 1. 숫자/금액 검증: 응답의 숫자가 도구 결과에 있는지
  const numbersInResponse = response.match(/\d+[\d,]*/g) || [];
  const numbersInContext = JSON.stringify(context.tools).match(/\d+[\d,]*/g) || [];

  for (const num of numbersInResponse) {
    if (parseInt(num.replace(/,/g, '')) > 100) {
      if (!numbersInContext.includes(num)) {
        return {
          detected: true,
          severity: 'high',
          reason: `응답에 근거 없는 숫자 "${num}" 포함`,
        };
      }
    }
  }

  // 2. URL 검증: 응답에 URL이 있으면 실제 존재하는지
  const urls = response.match(/https?:\/\/\S+/g) || [];
  for (const url of urls) {
    if (!context.rag.some(doc => doc.includes(url))) {
      return {
        detected: true,
        severity: 'medium',
        reason: `근거 없는 URL "${url}" 포함`,
      };
    }
  }

  // 3. 고유명사 검증: 이름, 제품명 등이 컨텍스트에 있는지
  // (고급: 별도 NER 모델 사용)

  return null;
}
```

&nbsp;

### 4-3. Prometheus + Grafana 설정

&nbsp;

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ai-agent'
    static_configs:
      - targets: ['agent:3000']
    metrics_path: '/metrics'

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

&nbsp;

```
Grafana 대시보드 구성:

┌─────────────────────────────────────────────────────┐
│                 AI Agent Dashboard                    │
│                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐│
│  │ 호출 수   │ │ 응답시간  │ │ 에러율   │ │  비용    ││
│  │ 1,234/h  │ │ 1.8s avg │ │ 0.3%    │ │ $12/day ││
│  └──────────┘ └──────────┘ └──────────┘ └─────────┘│
│                                                      │
│  ┌──────────────────────┐ ┌────────────────────────┐│
│  │ LLM 응답 시간 추이     │ │ 모델별 토큰 사용량       ││
│  │ ▁▂▃▄▅▆▇ (그래프)       │ │ Sonnet: 60%            ││
│  │                       │ │ Haiku: 35%             ││
│  │                       │ │ GPT-4o: 5%             ││
│  └──────────────────────┘ └────────────────────────┘│
│                                                      │
│  ┌──────────────────────┐ ┌────────────────────────┐│
│  │ 도구 호출 성공률       │ │ 환각 감지 현황           ││
│  │ lookupOrder: 99.2%   │ │ 이번 주: 3건            ││
│  │ processRefund: 98.5% │ │ 지난 주: 7건            ││
│  │ escalate: 100%       │ │ 추이: ▇▅▃ (감소 중)     ││
│  └──────────────────────┘ └────────────────────────┘│
│                                                      │
│  ┌──────────────────────────────────────────────────┐│
│  │ 알림 (Alerts)                                     ││
│  │ ⚠️ 에러율 > 5%                                    ││
│  │ ⚠️ 환각 감지 > 10건/일                              ││
│  │ ⚠️ LLM 응답시간 > 10초                              ││
│  │ ⚠️ 일 비용 > $100                                  ││
│  └──────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

&nbsp;

### 4-4. 알림 규칙

&nbsp;

```yaml
# monitoring/alert-rules.yml
groups:
  - name: ai-agent-alerts
    rules:
      # 에러율 높음
      - alert: AgentHighErrorRate
        expr: rate(agent_errors_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Agent 에러율 5% 초과"

      # LLM 응답 지연
      - alert: AgentHighLatency
        expr: histogram_quantile(0.95, rate(agent_llm_latency_seconds_bucket[5m])) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Agent LLM 응답시간 P95 > 10초"

      # 환각 감지 급증
      - alert: AgentHallucinationSpike
        expr: increase(agent_hallucination_detected_total[1h]) > 10
        labels:
          severity: warning
        annotations:
          summary: "1시간 내 환각 10건 이상 감지"

      # 비용 임계치
      - alert: AgentCostExceeded
        expr: increase(agent_cost_dollars_total[1d]) > 100
        labels:
          severity: warning
        annotations:
          summary: "일 비용 $100 초과"

      # LLM API 연결 실패
      - alert: AgentLLMConnectionFailed
        expr: rate(agent_llm_calls_total{status="error"}[5m]) > 0.5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "LLM API 연결 실패율 50% 초과"
```

&nbsp;

&nbsp;

---

&nbsp;

## 5. 헬스체크

&nbsp;

```typescript
// health.ts — Agent 전용 헬스체크

interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  checks: {
    llm: { status: string; latency?: number };
    vectorDB: { status: string; documentCount?: number };
    redis: { status: string; memoryUsage?: string };
    database: { status: string };
  };
  uptime: number;
  version: string;
}

app.get('/health', async (req, res) => {
  const checks = await Promise.allSettled([
    checkLLM(),
    checkVectorDB(),
    checkRedis(),
    checkDatabase(),
  ]);

  const health: HealthStatus = {
    status: 'healthy',
    checks: {
      llm: resolveCheck(checks[0]),
      vectorDB: resolveCheck(checks[1]),
      redis: resolveCheck(checks[2]),
      database: resolveCheck(checks[3]),
    },
    uptime: process.uptime(),
    version: process.env.npm_package_version || 'unknown',
  };

  // 하나라도 실패하면 degraded
  const statuses = Object.values(health.checks).map(c => c.status);
  if (statuses.some(s => s === 'unhealthy')) {
    health.status = statuses.every(s => s === 'unhealthy') ? 'unhealthy' : 'degraded';
  }

  const statusCode = health.status === 'unhealthy' ? 503 : 200;
  res.status(statusCode).json(health);
});

async function checkLLM(): Promise<any> {
  const start = Date.now();
  try {
    // 간단한 호출로 연결 확인 (비용 최소화)
    await anthropic.messages.create({
      model: 'claude-haiku-3-5-20241022',  // 가장 저렴한 모델
      max_tokens: 10,
      messages: [{ role: 'user', content: 'ping' }],
    });
    return { status: 'healthy', latency: Date.now() - start };
  } catch (error) {
    return { status: 'unhealthy', error: error.message };
  }
}

async function checkVectorDB(): Promise<any> {
  try {
    const collection = await chroma.getCollection({ name: 'main' });
    const count = await collection.count();
    return { status: 'healthy', documentCount: count };
  } catch (error) {
    return { status: 'unhealthy', error: error.message };
  }
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 6. 프롬프트 버전 관리 + A/B 테스트

&nbsp;

```typescript
// prompt-versioning.ts

interface PromptVersion {
  id: string;
  version: string;
  systemPrompt: string;
  temperature: number;
  model: string;
  createdAt: Date;
  isActive: boolean;
  abTestWeight: number;  // 0~1 (A/B 테스트 비율)
}

class PromptManager {
  private versions: PromptVersion[] = [];

  // A/B 테스트: 요청마다 가중치 기반으로 버전 선택
  selectVersion(): PromptVersion {
    const activeVersions = this.versions.filter(v => v.isActive);
    const random = Math.random();
    let cumulative = 0;

    for (const version of activeVersions) {
      cumulative += version.abTestWeight;
      if (random <= cumulative) return version;
    }

    return activeVersions[0];
  }

  // 버전별 성과 추적
  async getVersionMetrics(versionId: string): Promise<VersionMetrics> {
    return {
      totalCalls: await countCalls(versionId),
      avgLatency: await avgLatency(versionId),
      errorRate: await errorRate(versionId),
      hallucinationRate: await hallucinationRate(versionId),
      customerSatisfaction: await csat(versionId),
    };
  }
}

// 사용 예시
const promptManager = new PromptManager();

// v1: 기존 프롬프트 (80% 트래픽)
promptManager.addVersion({
  id: 'v1',
  version: '1.0.0',
  systemPrompt: '고객 상담 Agent입니다. ...',
  temperature: 0.3,
  model: 'claude-sonnet-4-20250514',
  abTestWeight: 0.8,
});

// v2: 개선된 프롬프트 (20% 트래픽)
promptManager.addVersion({
  id: 'v2',
  version: '1.1.0',
  systemPrompt: '고객 상담 전문 Agent입니다. 간결하게 답변하세요. ...',
  temperature: 0.2,
  model: 'claude-sonnet-4-20250514',
  abTestWeight: 0.2,
});

// 1주 후 결과 비교
// v1: 환각률 2.1%, CSAT 4.2/5
// v2: 환각률 0.8%, CSAT 4.5/5
// → v2를 100%로 전환
```

&nbsp;

&nbsp;

---

&nbsp;

## 7. 장애 대응 플레이북

&nbsp;

```
┌──────────────────────────────────────────────────────────┐
│             AI Agent 장애 대응 플레이북                     │
│                                                           │
│  [장애 유형 1] LLM API 연결 실패                            │
│  ─────────────────────────────                            │
│  증상: Agent 응답 불가, 에러율 100%                         │
│  원인: LLM 제공자 장애, API 키 만료, Rate Limit             │
│  대응:                                                    │
│    1. LLM 제공자 상태 페이지 확인                            │
│       - status.openai.com                                 │
│       - status.anthropic.com                              │
│    2. 백업 모델로 자동 전환 (Fallback)                       │
│       PRIMARY: Claude Sonnet → BACKUP: GPT-4o             │
│    3. Rate Limit인 경우 요청 큐잉 + 지연 처리               │
│    4. API 키 만료 시 즉시 갱신                              │
│                                                           │
│  [장애 유형 2] 환각 급증                                    │
│  ─────────────────────                                    │
│  증상: 사용자 불만 증가, 잘못된 정보 제공                     │
│  원인: 프롬프트 변경, 컨텍스트 오염, 모델 업데이트             │
│  대응:                                                    │
│    1. 최근 프롬프트 변경 확인 → 이전 버전 롤백               │
│    2. RAG 인덱스 확인 → 오염된 문서 제거                     │
│    3. 모델 버전 고정 (자동 업데이트 차단)                     │
│    4. 응답에 "확인 필요" 면책 문구 임시 추가                  │
│                                                           │
│  [장애 유형 3] 비용 폭주                                    │
│  ──────────────────                                       │
│  증상: 일 예산 초과 알림                                    │
│  원인: 무한 루프, 과도한 재시도, 프롬프트 비대화              │
│  대응:                                                    │
│    1. Rate Limiting 강화 (사용자당 분당 5회)                 │
│    2. 무한 루프 원인 파악 (도구 호출 반복 감지)               │
│    3. 프롬프트 길이 검사 (비정상적으로 긴 컨텍스트)            │
│    4. 작은 모델로 임시 전환                                 │
│                                                           │
│  [장애 유형 4] 벡터DB 장애                                  │
│  ──────────────────────                                   │
│  증상: RAG 검색 실패, Agent가 일반적인 답변만 제공            │
│  대응:                                                    │
│    1. 벡터DB 연결 확인 + 재시작                              │
│    2. RAG 없이 동작하는 Graceful Degradation 모드            │
│    3. 캐시된 FAQ 응답 제공 (Redis)                           │
│                                                           │
│  [공통] Fallback 체인                                      │
│  ─────────────────                                        │
│  1차: 기본 Agent (Claude Sonnet + RAG)                     │
│  2차: 백업 Agent (GPT-4o + RAG)                            │
│  3차: 경량 Agent (Claude Haiku, RAG 없음)                   │
│  4차: 정적 응답 (FAQ 데이터베이스)                           │
│  5차: 에러 메시지 + 상담원 연결                              │
└──────────────────────────────────────────────────────────┘
```

&nbsp;

```typescript
// fallback-chain.ts
class FallbackChain {
  private providers = [
    { name: 'primary', model: 'claude-sonnet-4-20250514', useRAG: true },
    { name: 'backup', model: 'gpt-4o', useRAG: true },
    { name: 'lightweight', model: 'claude-haiku-3-5-20241022', useRAG: false },
  ];

  async process(message: string, context: any): Promise<string> {
    for (const provider of this.providers) {
      try {
        return await this.callProvider(provider, message, context);
      } catch (error) {
        console.warn(`${provider.name} 실패, 다음 시도: ${error.message}`);
        continue;
      }
    }

    // 모든 LLM 실패 시 정적 응답
    const staticResponse = await this.searchFAQ(message);
    if (staticResponse) return staticResponse;

    return '현재 서비스에 일시적인 문제가 있습니다. 잠시 후 다시 시도해주세요.';
  }
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 8. 정리

&nbsp;

| 항목 | 소규모 | 중규모 | 대규모 |
|------|--------|--------|--------|
| 배포 | Docker Compose | Docker Compose + CI/CD | Kubernetes |
| 모니터링 | 로그 + 기본 메트릭 | Prometheus + Grafana | 전용 대시보드 + 알림 |
| 헬스체크 | HTTP /health | LLM + DB + 벡터DB 개별 체크 | + 환각 감지 |
| 롤백 | Docker 이미지 태그 | CI/CD 자동 롤백 | K8s 롤링 업데이트 |
| 장애 대응 | 수동 | 자동 알림 + 수동 대응 | 자동 Fallback 체인 |

&nbsp;

Agent 운영의 핵심은 **"일반 서비스 운영 + AI 전용 메트릭"**이다.

일반 서버 메트릭(CPU, 메모리, 에러율)만으로는 부족하다.

**토큰 사용량, 환각률, LLM 응답 시간, 비용 추적**이 추가로 필요하다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI Agent 실전 가이드 10편] 우리 회사에 도입하기 — ROI부터 확산까지**

&nbsp;

마지막 편. 도입 전 5가지 질문, ROI 계산법, 파일럿 프로젝트 설계, 실패/성공하는 도입 패턴, 조직 규모별 도입 사례, 그리고 시리즈 전체를 정리한다.

&nbsp;

&nbsp;

---

&nbsp;

AI에이전트, 배포, Docker, Kubernetes, CI/CD, 모니터링, Prometheus, Grafana, 헬스체크, 환각감지, 장애대응, 프롬프트버전관리
