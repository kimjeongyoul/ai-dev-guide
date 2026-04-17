# [AI Agent 실전 가이드 3편] 사무직 Agent — 이메일, 문서, 데이터 자동화

&nbsp;

회사에서 가장 많은 시간을 잡아먹는 일이 뭔가?

&nbsp;

코딩? 회의? 아니다.

**이메일 읽고, 문서 정리하고, 보고서 만드는** 사무 작업이다.

&nbsp;

한 조사에 따르면 사무직 직원은 하루 평균 2.5시간을 이메일에 쓴다. 주간 보고서 작성에 1시간, 회의록 정리에 30분. 이 시간의 **70%는 패턴이 반복되는 단순 작업**이다.

&nbsp;

사무직 Agent는 이 반복 작업을 자동화한다.

&nbsp;

&nbsp;

---

&nbsp;

## 1. 활용 사례 4가지

&nbsp;

### 1-1. 이메일 자동 분류 + 요약 + 답장 초안

&nbsp;

```
[수신함] 새 이메일 50건
   ↓
[Agent] 분류:
  - 계약 관련: 3건 → 법무팀 전달
  - 견적 요청: 5건 → 영업팀 전달 + 답장 초안 생성
  - 내부 공지: 12건 → 요약 후 Slack 전송
  - 스팸/광고: 30건 → 자동 아카이브
   ↓
[결과] 담당자별 요약 + 답장 초안이 30초 만에 완성
```

&nbsp;

### 1-2. 회의록 자동 생성 + 액션 아이템 추출

&nbsp;

```
[Zoom 녹음] 45분 회의 음성 파일
   ↓
[Agent] Whisper API로 음성 → 텍스트 변환
   ↓
[Agent] LLM으로 구조화:
  - 참석자: 김대리, 박과장, 이팀장
  - 주요 논의: API 성능 개선 방안
  - 결정 사항: Redis 캐시 도입
  - 액션 아이템:
    □ 김대리: Redis 벤치마크 (4/18까지)
    □ 박과장: 인프라 비용 산출 (4/20까지)
   ↓
[Agent] Notion에 자동 저장 + Slack에 요약 전송
```

&nbsp;

### 1-3. 엑셀/CSV 데이터 분석 + 리포트 생성

&nbsp;

```
[입력] 월간 매출 CSV (10만 행)
   ↓
[Agent] 데이터 분석:
  - 전월 대비 매출 +12%
  - 최고 실적 제품: A상품 (전월 대비 +35%)
  - 하락 품목: C상품 (-8%, 원인: 재고 부족)
  - 지역별: 서울 +15%, 부산 -3%
   ↓
[Agent] 리포트 생성 (차트 포함) → 이메일로 자동 발송
```

&nbsp;

### 1-4. 사내 규정 Q&A (RAG 기반)

&nbsp;

```
[직원] "연차 사용 시 며칠 전에 신청해야 하나요?"
   ↓
[Agent] RAG 검색 → 취업규칙 제23조 검색
   ↓
[Agent] "취업규칙 제23조에 따르면, 연차는 사용 3일 전까지 
        직속 상관에게 신청해야 합니다. 다만, 긴급한 경우 
        당일 신청이 가능하며 사유서를 제출해야 합니다."
```

&nbsp;

&nbsp;

---

&nbsp;

## 2. 아키텍처

&nbsp;

```
┌──────────────────────────────────────────────────────────────┐
│                      사무직 Agent 아키텍처                     │
│                                                               │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐  │
│  │ 트리거       │    │  Agent 서버   │    │  LLM API        │  │
│  │             │───►│              │───►│ (GPT/Claude)    │  │
│  │ - 이메일 수신│    │  분류/판단    │◄───│                 │  │
│  │ - 스케줄     │    │  도구 호출    │    └─────────────────┘  │
│  │ - 웹훅      │    │  결과 전달    │                         │
│  └─────────────┘    └──────┬───────┘    ┌─────────────────┐  │
│                            │            │  벡터DB (RAG)    │  │
│                            │───────────►│ 사내 규정/정책   │  │
│                            │            └─────────────────┘  │
│                            │                                  │
│                    ┌───────▼──────────────────────┐           │
│                    │        외부 시스템 연동        │           │
│                    ├────────────┬─────────────────┤           │
│                    │ MS365 /    │ Slack /         │           │
│                    │ Google     │ Notion /        │           │
│                    │ Workspace  │ Jira            │           │
│                    └────────────┴─────────────────┘           │
└──────────────────────────────────────────────────────────────┘
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. 구현 방법 3가지

&nbsp;

### 3-1. Power Automate + Azure OpenAI

&nbsp;

Microsoft 생태계를 쓰고 있다면 가장 쉬운 방법.

&nbsp;

```
[Power Automate 플로우]
트리거: 새 이메일 수신 (Outlook)
   ↓
액션 1: Azure OpenAI에 이메일 본문 전달
   ↓
액션 2: 분류 결과에 따라 분기
   ├── "계약" → 법무팀 Teams 채널에 전달
   ├── "견적" → 답장 초안 생성 → 임시보관함에 저장
   └── "광고" → 자동 아카이브
```

&nbsp;

장점: 코드 없이 구축 가능 (Low-code)

단점: 복잡한 로직에 한계, Microsoft 종속

비용: Power Automate Premium $15/월 + Azure OpenAI 사용량

&nbsp;

### 3-2. n8n + Anthropic API (추천)

&nbsp;

오픈소스 워크플로우 자동화 도구 + LLM API.

셀프 호스팅 가능하고, 유연성이 높다.

&nbsp;

```json
// n8n 워크플로우 JSON (이메일 요약 Agent)
{
  "name": "Email Summary Agent",
  "nodes": [
    {
      "name": "Gmail Trigger",
      "type": "n8n-nodes-base.gmailTrigger",
      "parameters": {
        "pollTimes": { "item": [{ "mode": "everyMinute" }] },
        "filters": { "labelIds": ["INBOX"] }
      }
    },
    {
      "name": "Classify Email",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "method": "POST",
        "url": "https://api.anthropic.com/v1/messages",
        "headers": {
          "x-api-key": "={{ $env.ANTHROPIC_API_KEY }}",
          "anthropic-version": "2023-06-01"
        },
        "body": {
          "model": "claude-sonnet-4-20250514",
          "max_tokens": 256,
          "messages": [{
            "role": "user",
            "content": "다음 이메일을 분류하세요. 카테고리: 계약/견적/내부공지/광고\n\n제목: {{ $json.subject }}\n본문: {{ $json.body }}"
          }]
        }
      }
    },
    {
      "name": "Route by Category",
      "type": "n8n-nodes-base.switch",
      "parameters": {
        "rules": [
          { "value": "계약", "output": 0 },
          { "value": "견적", "output": 1 },
          { "value": "내부공지", "output": 2 }
        ]
      }
    },
    {
      "name": "Send to Slack",
      "type": "n8n-nodes-base.slack",
      "parameters": {
        "channel": "#notifications",
        "text": "📧 새 이메일 ({{ $json.category }})\n제목: {{ $json.subject }}\n요약: {{ $json.summary }}"
      }
    }
  ]
}
```

&nbsp;

장점: 오픈소스, 셀프 호스팅, 200+ 연동

단점: 초기 설정 필요, 서버 운영

비용: 셀프 호스팅 무료 + LLM API 사용량

&nbsp;

### 3-3. Google Apps Script + Gemini

&nbsp;

Google Workspace를 쓰고 있다면 가장 빠른 방법.

&nbsp;

```javascript
// Google Apps Script: 이메일 요약 + 스프레드시트 기록
function processNewEmails() {
  const threads = GmailApp.getInboxThreads(0, 10);
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('이메일로그');
  
  for (const thread of threads) {
    const message = thread.getMessages()[0];
    const subject = message.getSubject();
    const body = message.getPlainBody().substring(0, 2000);
    
    // Gemini API로 분류 + 요약
    const result = callGemini(`
      다음 이메일을 분류하고 요약하세요.
      
      카테고리: 계약/견적/내부공지/기타
      
      제목: ${subject}
      본문: ${body}
      
      JSON으로 응답: {"category": "...", "summary": "...", "priority": "high/medium/low"}
    `);
    
    const parsed = JSON.parse(result);
    
    // 스프레드시트에 기록
    sheet.appendRow([
      new Date(),
      subject,
      parsed.category,
      parsed.summary,
      parsed.priority,
    ]);
    
    // 우선순위 높으면 Slack 알림
    if (parsed.priority === 'high') {
      sendSlackNotification(subject, parsed.summary);
    }
  }
}

function callGemini(prompt) {
  const url = 'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent';
  const response = UrlFetchApp.fetch(`${url}?key=${GEMINI_API_KEY}`, {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify({
      contents: [{ parts: [{ text: prompt }] }],
    }),
  });
  const data = JSON.parse(response.getContentText());
  return data.candidates[0].content.parts[0].text;
}

// 5분마다 자동 실행 (트리거 설정)
```

&nbsp;

장점: 무료 (Google Workspace + Gemini 무료 티어), 빠른 구축

단점: Google 종속, 실행 시간 제한 (6분)

&nbsp;

&nbsp;

---

&nbsp;

## 4. 실전 구현: Node.js로 이메일 요약 Agent

&nbsp;

프레임워크 없이 직접 만드는 방법.

&nbsp;

```typescript
// email-agent.ts
import Anthropic from '@anthropic-ai/sdk';
import Imap from 'imap';
import { simpleParser } from 'mailparser';

const anthropic = new Anthropic();

interface EmailSummary {
  subject: string;
  from: string;
  category: 'contract' | 'quote' | 'internal' | 'spam';
  summary: string;
  priority: 'high' | 'medium' | 'low';
  suggestedAction: string;
}

// 1. 이메일 수신
async function fetchNewEmails(): Promise<any[]> {
  return new Promise((resolve, reject) => {
    const imap = new Imap({
      user: process.env.EMAIL_USER!,
      password: process.env.EMAIL_PASSWORD!,
      host: 'imap.gmail.com',
      port: 993,
      tls: true,
    });

    imap.once('ready', () => {
      imap.openBox('INBOX', false, (err, box) => {
        // 오늘 수신한 읽지 않은 이메일
        imap.search(['UNSEEN', ['SINCE', new Date()]], (err, results) => {
          if (!results.length) { resolve([]); return; }
          
          const emails: any[] = [];
          const fetch = imap.fetch(results, { bodies: '' });
          
          fetch.on('message', (msg) => {
            msg.on('body', async (stream) => {
              const parsed = await simpleParser(stream);
              emails.push({
                subject: parsed.subject,
                from: parsed.from?.text,
                body: parsed.text?.substring(0, 3000),
                date: parsed.date,
              });
            });
          });
          
          fetch.once('end', () => { imap.end(); resolve(emails); });
        });
      });
    });

    imap.connect();
  });
}

// 2. AI로 분류 + 요약
async function classifyEmail(email: any): Promise<EmailSummary> {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 512,
    messages: [{
      role: 'user',
      content: `다음 이메일을 분석하세요.

제목: ${email.subject}
발신자: ${email.from}
본문:
${email.body}

아래 JSON 형식으로 응답하세요:
{
  "category": "contract | quote | internal | spam",
  "summary": "2줄 이내 요약",
  "priority": "high | medium | low",
  "suggestedAction": "권장 조치 (예: 법무팀 전달, 견적 회신 필요, 참고)"
}

JSON만 응답하세요.`,
    }],
  });

  const text = response.content[0].type === 'text' ? response.content[0].text : '';
  return { ...JSON.parse(text), subject: email.subject, from: email.from };
}

// 3. Slack 알림
async function notifySlack(summaries: EmailSummary[]) {
  const highPriority = summaries.filter(s => s.priority === 'high');
  const grouped = summaries.reduce((acc, s) => {
    acc[s.category] = (acc[s.category] || 0) + 1;
    return acc;
  }, {} as Record<string, number>);

  const message = [
    `*이메일 요약 (${summaries.length}건)*`,
    `계약: ${grouped.contract || 0}건 | 견적: ${grouped.quote || 0}건 | 내부: ${grouped.internal || 0}건 | 스팸: ${grouped.spam || 0}건`,
    '',
    highPriority.length > 0 ? '*긴급:*' : '',
    ...highPriority.map(s => `- [${s.category}] ${s.subject}\n  ${s.summary}\n  조치: ${s.suggestedAction}`),
  ].join('\n');

  await fetch(process.env.SLACK_WEBHOOK_URL!, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ text: message }),
  });
}

// 4. 메인 루프
async function run() {
  console.log('이메일 Agent 시작...');
  
  const emails = await fetchNewEmails();
  console.log(`새 이메일 ${emails.length}건`);
  
  if (emails.length === 0) return;
  
  const summaries = await Promise.all(emails.map(classifyEmail));
  await notifySlack(summaries);
  
  console.log('완료. Slack에 요약 전송됨.');
}

// 5분마다 실행
setInterval(run, 5 * 60 * 1000);
run();
```

&nbsp;

&nbsp;

---

&nbsp;

## 5. 보안 고려사항

&nbsp;

사무직 Agent는 이메일, 문서 등 **민감한 데이터**를 다룬다.

반드시 아래 항목을 검토해야 한다.

&nbsp;

### 5-1. 데이터 외부 전송 차단

&nbsp;

```typescript
// BAD: 이메일 전문을 외부 API로 전송
await llm.chat({ message: email.fullBody });  // 고객 개인정보 포함 가능

// GOOD: 개인정보 마스킹 후 전송
function maskPII(text: string): string {
  return text
    .replace(/\d{3}-\d{4}-\d{4}/g, '[PHONE]')        // 전화번호
    .replace(/\d{6}-\d{7}/g, '[RRN]')                 // 주민등록번호
    .replace(/[\w.-]+@[\w.-]+\.\w+/g, '[EMAIL]')      // 이메일
    .replace(/\d{4}-\d{4}-\d{4}-\d{4}/g, '[CARD]');   // 카드번호
}

await llm.chat({ message: maskPII(email.fullBody) });
```

&nbsp;

### 5-2. 권한 최소화

&nbsp;

```yaml
# Agent 권한 설정 원칙
email:
  read: true        # 읽기: 허용
  send: false       # 발송: 차단 (초안만 생성)
  delete: false     # 삭제: 차단

calendar:
  read: true
  create: false     # 일정 생성: 차단

file_system:
  read: true
  write: false      # 파일 수정: 차단
  
database:
  read: true
  write: false      # DB 수정: 차단
```

&nbsp;

### 5-3. 감사 로그

&nbsp;

```typescript
// 모든 AI 호출을 기록
async function callLLMWithAudit(params: LLMParams): Promise<string> {
  const startTime = Date.now();
  const response = await llm.chat(params);
  
  await auditLog.create({
    timestamp: new Date(),
    userId: params.requestedBy,
    action: 'llm_call',
    model: params.model,
    inputTokens: response.usage.input_tokens,
    outputTokens: response.usage.output_tokens,
    cost: calculateCost(response.usage),
    inputPreview: params.message.substring(0, 200),  // 전문 저장 금지
    duration: Date.now() - startTime,
  });

  return response;
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 6. 비용 예시: 직원 100명 기준

&nbsp;

```
가정:
- 직원 100명
- 직원당 하루 평균 50건 이메일 처리
- 이메일 평균 500자 (한글 ≈ 1,000~1,500 토큰)
- 분류 + 요약 응답 ≈ 200 토큰

일간 토큰 사용량:
- 입력: 100명 × 50건 × 1,500토큰 = 7,500,000 토큰
- 출력: 100명 × 50건 × 200토큰 = 1,000,000 토큰

월간 비용 (22영업일):
┌──────────────────┬──────────────┬──────────────────┐
│ 모델              │ 월 비용 (USD) │ 월 비용 (KRW)     │
├──────────────────┼──────────────┼──────────────────┤
│ GPT-4o           │ $632         │ 약 90만원          │
│ Claude Sonnet    │ $825         │ 약 118만원         │
│ Gemini 2.5 Pro   │ $316         │ 약 45만원          │
│ GPT-4o mini      │ $72          │ 약 10만원          │
│ Claude Haiku     │ $110         │ 약 16만원          │
└──────────────────┴──────────────┴──────────────────┘

vs 사무 보조 직원 1명 인건비: 약 300~400만원/월

→ GPT-4o mini로 100명 이메일 처리: 월 10만원
→ 사무 보조 1명이 하루 종일 해도 100명분은 불가능
```

&nbsp;

비용 대비 효과가 가장 확실한 유형이다.

&nbsp;

&nbsp;

---

&nbsp;

## 7. 사무직 Agent 도입 순서

&nbsp;

```
[1주차] PoC
├── 이메일 분류 + 요약 Agent 구축
├── 본인 이메일로 테스트
└── 정확도 측정 (목표: 90% 이상)

[2주차] 파일럿
├── 팀 5명에게 적용
├── 피드백 수집
├── 프롬프트 튜닝

[3~4주차] 확장
├── 회의록 Agent 추가
├── 사내 규정 Q&A 추가
├── 전 팀 확대 적용

[5~8주차] 고도화
├── 데이터 분석 리포트 Agent
├── 다른 팀/부서 확대
└── ROI 측정 + 보고
```

&nbsp;

&nbsp;

---

&nbsp;

## 8. 정리

&nbsp;

| 활용 사례 | 구현 난이도 | 효과 |
|-----------|------------|------|
| 이메일 분류/요약 | 낮음 | 높음 (하루 2시간 절약) |
| 회의록 생성 | 중간 | 높음 (회의당 30분 절약) |
| 데이터 분석 | 중간 | 중간 (보고서 1시간 절약) |
| 사내 Q&A | 높음 (RAG 필요) | 중간 (질문 대응 자동화) |

&nbsp;

사무직 Agent는 AI Agent 중에서 **가장 안전하고, 가장 ROI가 명확하다.**

실수해도 이메일 초안이 이상한 정도이고, 사람이 최종 확인한다.

&nbsp;

"AI Agent를 처음 도입한다면, 사무직 Agent부터 시작하라."

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI Agent 실전 가이드 4편] 고객 응대 Agent — 챗봇, 키오스크, 콜센터**

&nbsp;

고객과 직접 대면하는 Agent를 만든다. 기존 규칙 기반 챗봇과의 차이, WebSocket 실시간 통신, 멀티 턴 대화 관리, 상담원 에스컬레이션, 그리고 비용 대비 효과를 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

AI에이전트, 사무자동화, 이메일자동화, 회의록, RAG, n8n, Power Automate, Google Apps Script, 데이터분석, 사내Q&A, 업무자동화
