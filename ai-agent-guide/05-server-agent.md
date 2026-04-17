# [AI Agent 실전 가이드 5편] 서버 Agent — 모니터링, 장애 감지, 자동 복구

&nbsp;

새벽 3시. Sentry 알림이 울린다.

&nbsp;

"TypeError: Cannot read properties of undefined (reading 'id')"

에러율 5% 초과. 사용자 불만 접수 시작.

&nbsp;

**Before (사람):**

1. 알림 확인 (폰 울려서 깸) — 5분
2. VPN 연결, 로그 확인 — 10분
3. 원인 파악 — 20분
4. 코드 수정 — 15분
5. 테스트, 배포 — 15분
6. 총 65분 + 수면 부족

&nbsp;

**After (서버 Agent):**

1. 에러 감지 — 즉시
2. 스택트레이스 분석 — 10초
3. 롤백 실행 — 30초
4. Slack 보고 — 즉시
5. 총 1분. 사람은 아침에 보고만 확인.

&nbsp;

&nbsp;

---

&nbsp;

## 1. 활용 사례

&nbsp;

### 1-1. 에러 로그 분석 → 자동 수정 PR

&nbsp;

```
[Sentry] "TypeError: Cannot read properties of undefined (reading 'id')"
   ↓ 웹훅
[Agent] 스택트레이스 분석
   ↓
[Agent] 원인: src/api/user.ts:42 — user 객체 null 체크 누락
   ↓
[Agent] 수정 코드 생성 + 테스트 작성
   ↓
[Agent] GitHub PR #342 생성: "fix: add null check for user object in getUser"
   ↓
[Slack] "에러 수정 PR을 생성했습니다. 리뷰해주세요."
```

&nbsp;

### 1-2. 서버 부하 감지 → 오토스케일링

&nbsp;

```
[CloudWatch] CPU 사용률 90% 초과
   ↓
[Agent] 최근 30분 메트릭 분석
   ↓
[Agent] 판단: 트래픽 급증 (정상 패턴, 보안 이슈 아님)
   ↓
[Agent] ECS 서비스 인스턴스 3 → 6으로 스케일업
   ↓
[Agent] 10분 후 CPU 40%로 안정화 확인
   ↓
[Slack] "트래픽 급증으로 스케일업했습니다. 현재 안정. 30분 후 스케일다운 예정."
```

&nbsp;

### 1-3. 보안 이상 감지 → 자동 차단

&nbsp;

```
[WAF 로그] 동일 IP에서 1분간 500회 요청 (평소 10회)
   ↓
[Agent] 패턴 분석: SQL Injection 시도 포함
   ↓
[Agent] IP 차단 + Rate Limiting 강화
   ↓
[Agent] 보안팀 긴급 알림 + 상세 보고서 생성
   ↓
[보고서] 공격 시각, 소스 IP, 시도 내역, 차단 조치 요약
```

&nbsp;

### 1-4. 배포 모니터링 → 롤백 판단

&nbsp;

```
[배포] v2.5.1 프로덕션 배포 완료
   ↓ (5분 대기)
[Agent] 배포 후 메트릭 비교:
  - 에러율: 0.1% → 3.2% (3x 증가)
  - 응답시간: 120ms → 450ms (4x 증가)
  - 정상 트래픽: 유지
   ↓
[Agent] 판단: 배포로 인한 성능 저하 → 자동 롤백
   ↓
[Agent] v2.5.0으로 롤백 실행
   ↓
[Slack] "v2.5.1 자동 롤백. 에러율 3.2%, 응답시간 450ms 감지. v2.5.0 복원 완료."
```

&nbsp;

&nbsp;

---

&nbsp;

## 2. 아키텍처

&nbsp;

```
┌────────────────────────────────────────────────────────────────┐
│                    서버 Agent 아키텍처                          │
│                                                                 │
│  ┌────────────────────────────────────────┐                     │
│  │           이벤트 소스                    │                     │
│  │                                         │                     │
│  │  Sentry  CloudWatch  GitHub  WAF  Kafka │                     │
│  └────┬────────┬─────────┬───────┬────┬───┘                     │
│       │        │         │       │    │                          │
│       ▼        ▼         ▼       ▼    ▼                          │
│  ┌─────────────────────────────────────────┐                     │
│  │         이벤트 큐 (Kafka / SQS)          │                     │
│  └──────────────────┬──────────────────────┘                     │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────┐                    │
│  │              Agent 서버                   │                    │
│  │                                           │                    │
│  │  ┌──────────────┐  ┌──────────────────┐  │                    │
│  │  │ 이벤트 분류기  │  │  LLM API         │  │                    │
│  │  │ (심각도 판단)  │──│  (원인 분석)      │  │                    │
│  │  └──────────────┘  └────────┬─────────┘  │                    │
│  │                             │             │                    │
│  │  ┌──────────────────────────▼──────────┐  │                    │
│  │  │           액션 실행기                 │  │                    │
│  │  │                                      │  │                    │
│  │  │  자율 실행     │  승인 후 실행         │  │                    │
│  │  │  - 스케일링    │  - 롤백              │  │                    │
│  │  │  - IP 차단    │  - 코드 수정          │  │                    │
│  │  │  - 알림 전송  │  - 인프라 변경         │  │                    │
│  │  └──────────────────────────────────────┘  │                    │
│  └──────────────────────────────────────────┘                    │
│                     │                                            │
│            ┌────────┼────────┐                                   │
│            ▼        ▼        ▼                                   │
│       ┌────────┐┌───────┐┌──────┐                                │
│       │ Slack  ││GitHub ││ AWS  │                                │
│       │ 알림   ││ PR    ││ API  │                                │
│       └────────┘└───────┘└──────┘                                │
└────────────────────────────────────────────────────────────────┘
```

&nbsp;

### 자율성 레벨에 따른 액션 분류

&nbsp;

```
┌──────────────────────────────────────────────────────┐
│                  자율 실행 (L4)                        │
│                                                       │
│  위험도 낮음:                                          │
│  ✅ Slack/이메일 알림 전송                              │
│  ✅ 로그 수집 + 분석 보고서 생성                         │
│  ✅ 오토스케일링 (사전 정의 범위 내)                      │
│  ✅ IP 차단 (보안 이벤트)                               │
│  ✅ 캐시 초기화                                        │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                승인 후 실행 (L3)                       │
│                                                       │
│  위험도 중간:                                          │
│  ⚠️ 프로덕션 롤백                                      │
│  ⚠️ 코드 수정 PR 생성                                  │
│  ⚠️ DB 마이그레이션                                    │
│  ⚠️ 인프라 설정 변경                                    │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│               사람만 실행 (L1)                         │
│                                                       │
│  위험도 높음:                                          │
│  🚫 데이터 삭제/수정                                    │
│  🚫 보안 정책 변경                                      │
│  🚫 결제/과금 관련 변경                                  │
│  🚫 인프라 삭제 (서버, DB)                               │
└──────────────────────────────────────────────────────┘
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. 구현 예시: Sentry 웹훅 → AI 분석 → GitHub PR

&nbsp;

```typescript
// sentry-agent.ts
import Anthropic from '@anthropic-ai/sdk';
import { Octokit } from '@octokit/rest';
import express from 'express';

const anthropic = new Anthropic();
const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });
const app = express();

const REPO_OWNER = 'your-org';
const REPO_NAME = 'your-repo';

// 1. Sentry 웹훅 수신
app.post('/webhook/sentry', express.json(), async (req, res) => {
  const event = req.body;
  res.sendStatus(200);  // 즉시 응답 (비동기 처리)

  const { title, culprit, metadata, contexts } = event.data.event;
  const stacktrace = extractStacktrace(event.data.event);

  console.log(`[Sentry] 에러 수신: ${title}`);

  try {
    // 2. 관련 소스 코드 가져오기
    const filePath = extractFilePath(culprit);
    const sourceCode = await getFileContent(filePath);

    // 3. AI로 원인 분석 + 수정 코드 생성
    const analysis = await analyzeError(title, stacktrace, sourceCode, filePath);

    // 4. 수정이 가능한 경우 PR 생성
    if (analysis.canFix) {
      await createFixPR(analysis, filePath);
      await sendSlackNotification('fix_pr_created', analysis);
    } else {
      await sendSlackNotification('needs_human', analysis);
    }
  } catch (error) {
    console.error('Agent 처리 실패:', error);
    await sendSlackNotification('agent_error', { error });
  }
});

// AI 분석 함수
async function analyzeError(
  errorTitle: string,
  stacktrace: string,
  sourceCode: string,
  filePath: string
): Promise<ErrorAnalysis> {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 2048,
    messages: [{
      role: 'user',
      content: `프로덕션 에러를 분석하고 수정해주세요.

## 에러
${errorTitle}

## 스택트레이스
${stacktrace}

## 소스 코드 (${filePath})
\`\`\`typescript
${sourceCode}
\`\`\`

## 요청 사항
1. 에러 원인을 한 문장으로 설명하세요.
2. 수정 가능 여부를 판단하세요 (단순 null 체크, 타입 오류 등은 가능).
3. 수정 가능하면 수정된 전체 파일 코드를 작성하세요.
4. 수정이 위험하거나 복잡하면 canFix: false로 응답하세요.

JSON으로 응답:
{
  "rootCause": "원인 설명",
  "severity": "critical | high | medium | low",
  "canFix": true | false,
  "fixDescription": "수정 내용 설명",
  "fixedCode": "수정된 전체 코드 (canFix가 true일 때만)",
  "testCode": "추가할 테스트 코드"
}`,
    }],
  });

  const text = response.content[0].type === 'text' ? response.content[0].text : '';
  return JSON.parse(text);
}

// GitHub PR 생성
async function createFixPR(analysis: ErrorAnalysis, filePath: string) {
  const baseBranch = 'main';
  const fixBranch = `fix/auto-${Date.now()}`;

  // 1. 새 브랜치 생성
  const baseRef = await octokit.git.getRef({
    owner: REPO_OWNER, repo: REPO_NAME, ref: `heads/${baseBranch}`,
  });

  await octokit.git.createRef({
    owner: REPO_OWNER, repo: REPO_NAME,
    ref: `refs/heads/${fixBranch}`,
    sha: baseRef.data.object.sha,
  });

  // 2. 파일 수정
  const currentFile = await octokit.repos.getContent({
    owner: REPO_OWNER, repo: REPO_NAME, path: filePath, ref: fixBranch,
  });

  await octokit.repos.createOrUpdateFileContents({
    owner: REPO_OWNER, repo: REPO_NAME, path: filePath,
    message: `fix: ${analysis.fixDescription}`,
    content: Buffer.from(analysis.fixedCode).toString('base64'),
    sha: (currentFile.data as any).sha,
    branch: fixBranch,
  });

  // 3. PR 생성
  const pr = await octokit.pulls.create({
    owner: REPO_OWNER, repo: REPO_NAME,
    title: `fix: ${analysis.fixDescription}`,
    head: fixBranch,
    base: baseBranch,
    body: `## AI Agent 자동 수정

**에러:** ${analysis.rootCause}
**심각도:** ${analysis.severity}
**수정 내용:** ${analysis.fixDescription}

> 이 PR은 서버 Agent가 자동으로 생성했습니다. 반드시 리뷰 후 머지해주세요.`,
  });

  console.log(`PR 생성 완료: ${pr.data.html_url}`);
}

app.listen(3100, () => console.log('Server Agent listening on 3100'));
```

&nbsp;

&nbsp;

---

&nbsp;

## 4. 구현 예시: Python으로 서버 모니터링 Agent

&nbsp;

```python
# server-monitoring-agent.py
import anthropic
import boto3
import requests
import json
import time
from datetime import datetime, timedelta
from dataclasses import dataclass

client = anthropic.Anthropic()
cloudwatch = boto3.client('cloudwatch')
ecs = boto3.client('ecs')

SLACK_WEBHOOK = "https://hooks.slack.com/services/xxx"
CLUSTER_NAME = "production"
SERVICE_NAME = "api-server"

@dataclass
class MetricSnapshot:
    cpu_percent: float
    memory_percent: float
    error_rate: float
    request_count: int
    response_time_ms: float
    timestamp: str


def get_metrics(minutes: int = 5) -> MetricSnapshot:
    """CloudWatch에서 메트릭 수집"""
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(minutes=minutes)

    # CPU 사용률
    cpu = cloudwatch.get_metric_statistics(
        Namespace='AWS/ECS',
        MetricName='CPUUtilization',
        Dimensions=[
            {'Name': 'ClusterName', 'Value': CLUSTER_NAME},
            {'Name': 'ServiceName', 'Value': SERVICE_NAME},
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average'],
    )

    # 에러율 (커스텀 메트릭)
    errors = cloudwatch.get_metric_statistics(
        Namespace='Custom/API',
        MetricName='ErrorRate',
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average'],
    )

    return MetricSnapshot(
        cpu_percent=cpu['Datapoints'][-1]['Average'] if cpu['Datapoints'] else 0,
        memory_percent=0,  # 생략
        error_rate=errors['Datapoints'][-1]['Average'] if errors['Datapoints'] else 0,
        request_count=0,  # 생략
        response_time_ms=0,  # 생략
        timestamp=datetime.utcnow().isoformat(),
    )


def analyze_and_act(metrics: MetricSnapshot):
    """AI가 메트릭을 분석하고 행동을 결정"""
    
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        system="""서버 모니터링 Agent입니다.
메트릭을 분석하고 적절한 조치를 결정하세요.

사용 가능한 액션:
1. scale_up: 인스턴스 수 증가 (현재 3대, 최대 10대)
2. scale_down: 인스턴스 수 감소 (최소 2대)
3. rollback: 최근 배포 롤백
4. block_ip: 의심 IP 차단
5. notify: Slack 알림만 전송
6. none: 조치 불필요

JSON으로 응답:
{
  "analysis": "현재 상태 분석",
  "action": "액션 이름",
  "params": { ... },
  "severity": "critical | warning | info",
  "reason": "조치 사유"
}""",
        messages=[{
            "role": "user",
            "content": f"""현재 서버 메트릭:
- CPU: {metrics.cpu_percent:.1f}%
- 에러율: {metrics.error_rate:.2f}%
- 시각: {metrics.timestamp}

최근 30분 추이: CPU 50% → 70% → {metrics.cpu_percent:.1f}% (상승 중)
최근 배포: 2시간 전 v2.5.1

어떤 조치가 필요한가요?""",
        }],
    )

    text = response.content[0].text
    decision = json.loads(text)
    
    print(f"[분석] {decision['analysis']}")
    print(f"[액션] {decision['action']} - {decision['reason']}")

    # 액션 실행
    execute_action(decision)


def execute_action(decision: dict):
    """결정된 액션 실행"""
    action = decision["action"]
    params = decision.get("params", {})

    if action == "scale_up":
        desired_count = params.get("count", 6)
        ecs.update_service(
            cluster=CLUSTER_NAME,
            service=SERVICE_NAME,
            desiredCount=min(desired_count, 10),  # 최대 10대 제한
        )
        notify_slack(f"스케일업 실행: {desired_count}대\n사유: {decision['reason']}")

    elif action == "rollback":
        # 롤백은 알림만 보내고, 사람이 확인 후 실행
        notify_slack(
            f"⚠️ 롤백 권고\n"
            f"분석: {decision['analysis']}\n"
            f"사유: {decision['reason']}\n"
            f"롤백 명령: `aws ecs update-service --force-new-deployment`",
            urgent=True,
        )

    elif action == "notify":
        notify_slack(f"[{decision['severity']}] {decision['analysis']}")

    elif action == "none":
        print("조치 불필요")


def notify_slack(message: str, urgent: bool = False):
    """Slack 알림 전송"""
    prefix = "🚨" if urgent else "🤖"
    requests.post(SLACK_WEBHOOK, json={
        "text": f"{prefix} *서버 Agent*\n{message}",
    })


# 메인 루프: 1분마다 체크
if __name__ == "__main__":
    print("서버 모니터링 Agent 시작")
    while True:
        try:
            metrics = get_metrics(minutes=5)
            
            # 임계치 초과 시에만 AI 분석 (비용 절약)
            if metrics.cpu_percent > 70 or metrics.error_rate > 1:
                analyze_and_act(metrics)
            else:
                print(f"[정상] CPU: {metrics.cpu_percent:.1f}%, 에러율: {metrics.error_rate:.2f}%")
            
        except Exception as e:
            print(f"Agent 에러: {e}")
            notify_slack(f"Agent 자체 에러: {e}")
        
        time.sleep(60)  # 1분 대기
```

&nbsp;

&nbsp;

---

&nbsp;

## 5. 이벤트 스트리밍: Kafka 연동

&nbsp;

대규모 서비스에서는 이벤트를 Kafka로 스트리밍하고, Agent가 컨슈머로 처리한다.

&nbsp;

```typescript
// kafka-agent-consumer.ts
import { Kafka, EachMessagePayload } from 'kafkajs';
import Anthropic from '@anthropic-ai/sdk';

const kafka = new Kafka({
  clientId: 'server-agent',
  brokers: ['kafka1:9092', 'kafka2:9092'],
});

const consumer = kafka.consumer({ groupId: 'server-agent-group' });
const anthropic = new Anthropic();

// 이벤트 버퍼: 단건이 아닌 배치로 AI 분석 (비용 절약)
let eventBuffer: ServerEvent[] = [];
const BUFFER_SIZE = 10;
const BUFFER_TIMEOUT_MS = 30_000;

interface ServerEvent {
  type: 'error' | 'metric' | 'security' | 'deploy';
  source: string;
  data: any;
  timestamp: string;
}

async function run() {
  await consumer.connect();
  await consumer.subscribe({ topics: ['server-events', 'security-events'] });

  await consumer.run({
    eachMessage: async ({ topic, message }: EachMessagePayload) => {
      const event: ServerEvent = JSON.parse(message.value!.toString());
      
      // 긴급 이벤트는 즉시 처리
      if (event.type === 'security' && isUrgent(event)) {
        await handleUrgentEvent(event);
        return;
      }
      
      // 일반 이벤트는 버퍼에 쌓아서 배치 처리
      eventBuffer.push(event);
      if (eventBuffer.length >= BUFFER_SIZE) {
        await processBatch([...eventBuffer]);
        eventBuffer = [];
      }
    },
  });
}

async function processBatch(events: ServerEvent[]) {
  const summary = events.map(e => 
    `[${e.type}] ${e.source}: ${JSON.stringify(e.data)}`
  ).join('\n');

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    messages: [{
      role: 'user',
      content: `최근 ${events.length}건의 서버 이벤트를 분석하세요.\n\n${summary}\n\n패턴이 있으면 보고하고, 조치가 필요하면 알려주세요.`,
    }],
  });

  const analysis = response.content[0].type === 'text' ? response.content[0].text : '';
  
  // 조치가 필요한 경우에만 알림
  if (analysis.includes('조치 필요') || analysis.includes('위험')) {
    await sendSlackNotification(analysis);
  }
}

async function handleUrgentEvent(event: ServerEvent) {
  console.log(`[긴급] ${event.type} 이벤트 즉시 처리`);
  
  if (event.type === 'security') {
    // 보안 이벤트: IP 차단 자동 실행
    const ip = event.data.sourceIp;
    await blockIP(ip);
    await sendSlackNotification(`🚨 보안 이벤트: ${ip} 자동 차단\n상세: ${JSON.stringify(event.data)}`);
  }
}

run().catch(console.error);
```

&nbsp;

&nbsp;

---

&nbsp;

## 6. 위험 관리: 자동 복구의 범위 제한

&nbsp;

서버 Agent는 가장 강력하지만, 가장 위험하기도 하다.

반드시 **안전장치**를 설계해야 한다.

&nbsp;

```typescript
// 안전장치 설정
const SAFETY_CONFIG = {
  // 자동 실행 제한
  maxAutoScalePerHour: 3,       // 시간당 최대 3번 스케일링
  maxAutoRollbackPerDay: 1,     // 하루 최대 1번 자동 롤백
  maxAutoBlockIPsPerHour: 10,   // 시간당 최대 10개 IP 차단
  
  // 금지 액션 (절대 자동 실행 안 함)
  forbiddenActions: [
    'delete_database',
    'delete_server',
    'modify_security_group',
    'change_dns',
    'modify_payment_config',
  ],
  
  // 쿨다운: 같은 액션 반복 방지
  cooldownMinutes: {
    scale_up: 10,
    scale_down: 30,
    rollback: 60,
    restart: 15,
  },
  
  // 업무 시간 외 제한
  offHoursPolicy: {
    // 야간(22:00~08:00)에는 알림만, 실행은 안 함
    autoExecute: false,
    notifyOnly: true,
    exceptUrgent: true,  // 긴급 보안 이벤트는 예외
  },
};

// 액션 실행 전 안전 검사
async function safeExecute(action: string, params: any): Promise<boolean> {
  // 1. 금지 액션 체크
  if (SAFETY_CONFIG.forbiddenActions.includes(action)) {
    console.log(`[차단] 금지된 액션: ${action}`);
    await notifySlack(`⚠️ Agent가 금지된 액션을 시도했습니다: ${action}`);
    return false;
  }

  // 2. 실행 횟수 체크
  const recentCount = await getRecentActionCount(action, 60);
  const maxCount = getMaxCount(action);
  if (recentCount >= maxCount) {
    console.log(`[차단] 실행 횟수 초과: ${action} (${recentCount}/${maxCount})`);
    return false;
  }

  // 3. 쿨다운 체크
  const lastExecution = await getLastExecutionTime(action);
  const cooldown = SAFETY_CONFIG.cooldownMinutes[action] || 10;
  if (lastExecution && minutesSince(lastExecution) < cooldown) {
    console.log(`[차단] 쿨다운 중: ${action} (${cooldown}분)`);
    return false;
  }

  // 4. 업무 시간 체크
  const hour = new Date().getHours();
  if ((hour >= 22 || hour < 8) && !SAFETY_CONFIG.offHoursPolicy.autoExecute) {
    if (action !== 'block_ip') {  // 보안은 예외
      await notifySlack(`[야간] Agent 액션 보류: ${action}\n파라미터: ${JSON.stringify(params)}`);
      return false;
    }
  }

  return true;
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 7. "새벽 3시 장애 대응" 시나리오

&nbsp;

```
03:00  [Sentry] 에러 급증 감지 — "Connection refused: payment-service:8080"
       │
03:00  [Agent] 이벤트 수신. 긴급도 분석.
       │  → 결제 서비스 연결 실패. 심각도: CRITICAL.
       │
03:01  [Agent] 원인 분석 시작
       │  → 결제 서비스 헬스체크: DOWN
       │  → 최근 배포: 02:45 결제 서비스 v3.2.1 배포
       │  → 결론: 배포로 인한 장애 가능성 높음
       │
03:01  [Agent] 자동 조치 실행
       │  ✅ 결제 서비스 v3.2.0 롤백 실행
       │  ✅ Slack #incident 채널에 장애 보고
       │  ✅ 온콜 담당자 PagerDuty 알림
       │
03:03  [Agent] 롤백 완료 확인
       │  → 결제 서비스 헬스체크: UP
       │  → 에러율: 5.2% → 0.1% (정상)
       │
03:03  [Slack 보고]
       │  "🚨 장애 감지 및 자동 복구 완료
       │   - 원인: 결제 서비스 v3.2.1 배포 장애
       │   - 조치: v3.2.0 롤백
       │   - 현재 상태: 정상
       │   - 장애 시간: 약 3분
       │   - 상세 분석 보고서: [링크]"
       │
08:00  [온콜 담당자] 출근 후 보고서 확인
       │  → v3.2.1 변경 내역 검토
       │  → 원인: DB 커넥션 풀 설정 오류
       │  → 수정 후 재배포
```

&nbsp;

이 시나리오에서 사람이 한 일: **아침에 보고서 읽고 근본 원인 수정.**

새벽에 깨지 않아도 된다.

&nbsp;

&nbsp;

---

&nbsp;

## 8. 정리

&nbsp;

| 항목 | 자율 실행 | 승인 후 실행 | 사람만 |
|------|----------|-------------|--------|
| Slack 알림 | O | | |
| 로그 분석 보고 | O | | |
| 오토스케일링 | O (범위 내) | O (범위 초과) | |
| IP 차단 | O | | |
| 롤백 | | O | |
| 코드 수정 PR | | O (리뷰 필수) | |
| DB 변경 | | | O |
| 인프라 삭제 | | | O |

&nbsp;

서버 Agent의 핵심 원칙:

1. **분석은 자율, 실행은 단계적** — 모든 걸 자동화하려고 하지 마라
2. **안전장치 먼저** — 횟수 제한, 쿨다운, 금지 액션
3. **감사 로그 필수** — Agent가 뭘 했는지 전부 기록
4. **점진적 확대** — 알림 → 분석 → 제한적 실행 → 자율 실행

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI Agent 실전 가이드 6편] 인프라 & 스펙 — 뭘 깔아야 하나**

&nbsp;

Agent를 돌리려면 GPU가 필요한가? 서버 스펙은? 클라우드 API만 쓰면 되는가, 온프레미스 모델을 돌려야 하는가? Docker Compose 구성부터 Kubernetes 배포까지, 인프라 관점에서 정리한다.

&nbsp;

&nbsp;

---

&nbsp;

AI에이전트, 서버모니터링, 장애대응, 자동복구, Sentry, CloudWatch, Kafka, 오토스케일링, 롤백, DevOps, SRE, 보안자동화
