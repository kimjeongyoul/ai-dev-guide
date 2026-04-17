# [AI Agent 실전 가이드 2편] 아키텍처 기초 — LLM + RAG + Tool Use 조합

&nbsp;

1편에서 AI Agent의 개념을 잡았다.

이번 편에서는 **실제로 어떻게 만드는지** 구조를 본다.

&nbsp;

Agent를 구성하는 핵심 3가지:

1. **LLM** — 두뇌 (판단)
2. **RAG** — 기억 (사내 지식)
3. **Tool Use** — 손발 (실행)

&nbsp;

이 3가지를 어떻게 조합하느냐에 따라 Agent의 능력이 결정된다.

&nbsp;

&nbsp;

---

&nbsp;

## 1. LLM API 호출 구조

&nbsp;

### 1-1. 주요 LLM 제공자

&nbsp;

| 제공자 | 대표 모델 | 특징 |
|--------|-----------|------|
| OpenAI | GPT-4o, GPT-4.1 | 가장 넓은 생태계, Function Calling 원조 |
| Anthropic | Claude Sonnet, Opus | 긴 컨텍스트(200K), 코딩 강점 |
| Google | Gemini 2.5 Pro | 멀티모달, 100만 토큰 컨텍스트 |
| Meta | Llama 3 (오픈소스) | 무료, 온프레미스 가능 |
| Mistral | Mistral Large | 유럽 기반, 경량 모델 강점 |

&nbsp;

### 1-2. 기본 API 호출

&nbsp;

```typescript
// OpenAI
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const response = await openai.chat.completions.create({
  model: 'gpt-4o',
  messages: [
    { role: 'system', content: '당신은 고객 상담 Agent입니다.' },
    { role: 'user', content: '주문 상태 확인해주세요.' },
  ],
});
console.log(response.choices[0].message.content);
```

&nbsp;

```typescript
// Anthropic
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  system: '당신은 고객 상담 Agent입니다.',
  messages: [
    { role: 'user', content: '주문 상태 확인해주세요.' },
  ],
});
console.log(response.content[0].text);
```

&nbsp;

```python
# Python (Anthropic)
import anthropic

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system="당신은 고객 상담 Agent입니다.",
    messages=[
        {"role": "user", "content": "주문 상태 확인해주세요."}
    ],
)
print(response.content[0].text)
```

&nbsp;

### 1-3. 로컬 모델 (온프레미스)

&nbsp;

데이터가 외부로 나가면 안 되는 경우, 사내에서 직접 모델을 돌릴 수 있다.

&nbsp;

```bash
# Ollama로 로컬에서 Llama 3 실행
ollama pull llama3
ollama run llama3 "서버 에러 로그를 분석해줘"

# API 서버로도 사용 가능
curl http://localhost:11434/api/chat -d '{
  "model": "llama3",
  "messages": [{"role": "user", "content": "에러 분석해줘"}]
}'
```

&nbsp;

```python
# vLLM으로 프로덕션 수준의 모델 서빙
# pip install vllm
from vllm import LLM, SamplingParams

llm = LLM(model="meta-llama/Llama-3-70B-Instruct")
params = SamplingParams(temperature=0.1, max_tokens=512)
outputs = llm.generate(["에러 로그를 분석해줘"], params)
print(outputs[0].outputs[0].text)
```

&nbsp;

&nbsp;

---

&nbsp;

## 2. RAG — 사내 문서를 AI가 알게 하려면

&nbsp;

### 2-1. 문제: LLM은 사내 정보를 모른다

&nbsp;

```
[사용자] "우리 회사 환불 정책 알려줘"
[LLM]   "일반적으로 환불은 구매 후 7일 이내..." ← 우리 회사 정책이 아님!
```

&nbsp;

LLM은 인터넷의 일반 지식으로 학습됐다. 사내 정책, 제품 정보, 고객 데이터는 모른다.

&nbsp;

**방법 1:** 프롬프트에 전부 넣는다 → 컨텍스트 한계 + 비용 폭발

**방법 2:** 모델을 사내 데이터로 파인튜닝 → 비용 + 시간 + 업데이트 어려움

**방법 3:** RAG — 필요할 때 관련 문서만 찾아서 프롬프트에 넣는다 ← **정답**

&nbsp;

### 2-2. RAG 파이프라인

&nbsp;

```
┌─────────────────── 인덱싱 (1회) ─────────────────────┐
│                                                        │
│  [사내 문서] → [텍스트 분할] → [임베딩 변환] → [벡터DB 저장] │
│   PDF, 위키,     chunk       OpenAI         Pinecone    │
│   Notion 등      (512토큰)   text-embedding  pgvector   │
│                                                        │
└────────────────────────────────────────────────────────┘

┌─────────────────── 검색 (매 질문) ───────────────────┐
│                                                       │
│  [사용자 질문] → [임베딩 변환] → [벡터DB 검색] → [상위 N개 문서] │
│  "환불 정책?"    질문 벡터화      코사인 유사도    환불 관련 문서  │
│                                                       │
│  → [LLM에 컨텍스트로 전달] → [사내 정보 기반 답변]            │
│                                                       │
└───────────────────────────────────────────────────────┘
```

&nbsp;

### 2-3. 임베딩이란

&nbsp;

텍스트를 숫자 벡터(배열)로 변환하는 것.

의미가 비슷한 텍스트는 벡터도 비슷해진다.

&nbsp;

```typescript
import OpenAI from 'openai';

const openai = new OpenAI();

// 텍스트 → 벡터 변환
const response = await openai.embeddings.create({
  model: 'text-embedding-3-small',
  input: '환불은 구매 후 14일 이내에 가능합니다.',
});

const vector = response.data[0].embedding;
// → [0.0023, -0.0142, 0.0087, ...] (1536차원 벡터)
```

&nbsp;

### 2-4. 벡터DB 비교

&nbsp;

| 벡터DB | 유형 | 가격 | 장점 | 단점 |
|--------|------|------|------|------|
| Pinecone | 관리형 SaaS | 무료~$70/월 | 설정 불필요, 빠름 | 데이터가 외부에 |
| Weaviate | 오픈소스/클라우드 | 무료~ | 하이브리드 검색 | 운영 복잡도 |
| Chroma | 오픈소스 | 무료 | 가볍고 간편 | 대규모에 약함 |
| pgvector | PostgreSQL 확장 | 무료 | 기존 DB 활용 | 전용 DB 대비 느림 |
| Qdrant | 오픈소스/클라우드 | 무료~ | 빠르고 기능 풍부 | 생태계 작음 |

&nbsp;

### 2-5. RAG 구현 예시

&nbsp;

```typescript
import { ChromaClient } from 'chromadb';
import OpenAI from 'openai';

const chroma = new ChromaClient();
const openai = new OpenAI();

// 1. 컬렉션 생성 + 문서 인덱싱
const collection = await chroma.createCollection({ name: 'company-docs' });

const documents = [
  '환불 정책: 구매 후 14일 이내 전액 환불 가능. 개봉 제품은 7일 이내.',
  '교환 정책: 동일 상품 교환은 30일 이내. 다른 상품 교환은 14일 이내.',
  '배송 정책: 주문 후 1~3 영업일 배송. 도서산간 추가 2일.',
];

await collection.add({
  ids: ['doc1', 'doc2', 'doc3'],
  documents,
});

// 2. 질문이 들어오면 관련 문서 검색
const results = await collection.query({
  queryTexts: ['환불하고 싶은데 언제까지 가능한가요?'],
  nResults: 2,
});

// 3. 검색된 문서를 LLM에 컨텍스트로 전달
const context = results.documents[0].join('\n');

const response = await openai.chat.completions.create({
  model: 'gpt-4o',
  messages: [
    {
      role: 'system',
      content: `아래 사내 문서를 참고해서 답변하세요.\n\n${context}`,
    },
    {
      role: 'user',
      content: '환불하고 싶은데 언제까지 가능한가요?',
    },
  ],
});

// → "구매 후 14일 이내에 전액 환불이 가능합니다. 
//    개봉된 제품의 경우 7일 이내에 환불하셔야 합니다."
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. Tool Use — AI가 실제로 "행동"하게 하려면

&nbsp;

### 3-1. Tool Use란

&nbsp;

LLM에게 "이런 도구들을 쓸 수 있어"라고 알려주면, LLM이 상황에 맞는 도구를 골라서 호출한다.

&nbsp;

```
[사용자] "주문번호 ORD-1234 상태 알려줘"
   ↓
[LLM] "주문 조회 도구를 써야겠다" → lookupOrder("ORD-1234") 호출
   ↓
[도구 실행 결과] { status: "배송중", eta: "2025-04-17" }
   ↓
[LLM] "주문번호 ORD-1234는 현재 배송 중이며, 4월 17일 도착 예정입니다."
```

&nbsp;

### 3-2. Function Calling (OpenAI)

&nbsp;

```typescript
const response = await openai.chat.completions.create({
  model: 'gpt-4o',
  messages: [
    { role: 'user', content: '주문번호 ORD-1234 상태 알려줘' },
  ],
  tools: [
    {
      type: 'function',
      function: {
        name: 'lookupOrder',
        description: '주문번호로 주문 상태를 조회합니다',
        parameters: {
          type: 'object',
          properties: {
            orderId: { type: 'string', description: '주문번호' },
          },
          required: ['orderId'],
        },
      },
    },
    {
      type: 'function',
      function: {
        name: 'processRefund',
        description: '주문에 대한 환불을 처리합니다',
        parameters: {
          type: 'object',
          properties: {
            orderId: { type: 'string' },
            reason: { type: 'string' },
          },
          required: ['orderId', 'reason'],
        },
      },
    },
  ],
});

// LLM이 도구 호출을 결정하면:
const toolCall = response.choices[0].message.tool_calls[0];
// → { function: { name: 'lookupOrder', arguments: '{"orderId":"ORD-1234"}' } }

// 실제 도구 실행
const result = await lookupOrder(JSON.parse(toolCall.function.arguments));
// → { status: '배송중', eta: '2025-04-17' }
```

&nbsp;

### 3-3. Tool Use (Anthropic)

&nbsp;

```typescript
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  tools: [
    {
      name: 'lookupOrder',
      description: '주문번호로 주문 상태를 조회합니다',
      input_schema: {
        type: 'object',
        properties: {
          orderId: { type: 'string', description: '주문번호' },
        },
        required: ['orderId'],
      },
    },
  ],
  messages: [
    { role: 'user', content: '주문번호 ORD-1234 상태 알려줘' },
  ],
});

// tool_use 블록에서 도구 호출 확인
const toolUse = response.content.find(block => block.type === 'tool_use');
// → { type: 'tool_use', name: 'lookupOrder', input: { orderId: 'ORD-1234' } }
```

&nbsp;

### 3-4. MCP (Model Context Protocol)

&nbsp;

Anthropic이 제안한 **표준화된 도구 연동 프로토콜.**

각 서비스마다 다른 연동 방식 대신, 하나의 표준으로 통일한다.

&nbsp;

```
기존 방식:
[AI Agent] ──── 각각 다른 방식으로 ────── [GitHub API]
           ──── 각각 다른 방식으로 ────── [Slack API]
           ──── 각각 다른 방식으로 ────── [DB]
           ──── 각각 다른 방식으로 ────── [Jira API]

MCP 방식:
[AI Agent] ──── MCP 프로토콜 ────── [GitHub MCP 서버]
           ──── MCP 프로토콜 ────── [Slack MCP 서버]
           ──── MCP 프로토콜 ────── [DB MCP 서버]
           ──── MCP 프로토콜 ────── [Jira MCP 서버]
```

&nbsp;

```json
// MCP 서버 설정 예시
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "ghp_xxx" }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": { "SLACK_TOKEN": "xoxb-xxx" }
    },
    "database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "postgresql://..." }
    }
  }
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 4. 프롬프트 체인 — 복잡한 작업을 여러 단계로

&nbsp;

하나의 프롬프트로 모든 걸 처리하려면 실패한다.

복잡한 작업은 **여러 단계의 프롬프트**로 나눈다.

&nbsp;

```typescript
// 단일 프롬프트 (실패하기 쉬움)
const response = await llm.chat({
  message: '이 에러 로그를 분석하고, 원인을 찾고, 수정 코드를 작성하고, PR을 만들어줘',
});

// 프롬프트 체인 (안정적)
// Step 1: 에러 분석
const analysis = await llm.chat({
  message: `이 에러 로그를 분석하세요: ${errorLog}`,
});

// Step 2: 원인 파악 (Step 1 결과 사용)
const rootCause = await llm.chat({
  message: `분석 결과를 바탕으로 근본 원인을 파악하세요: ${analysis}`,
});

// Step 3: 수정 코드 생성 (Step 2 결과 사용)
const fix = await llm.chat({
  message: `이 원인에 대한 수정 코드를 작성하세요: ${rootCause}`,
  context: sourceCode,
});

// Step 4: PR 생성 (Step 3 결과 사용)
await github.createPR({
  title: `fix: ${rootCause.summary}`,
  body: analysis.full,
  changes: fix.code,
});
```

&nbsp;

&nbsp;

---

&nbsp;

## 5. 메모리 — 단기 vs 장기

&nbsp;

### 5-1. 단기 메모리 (대화 컨텍스트)

&nbsp;

현재 대화의 흐름을 기억하는 것. messages 배열에 이전 대화를 담아서 전달한다.

&nbsp;

```typescript
// 대화 기록을 유지하면서 API 호출
const conversationHistory: Message[] = [];

async function chat(userMessage: string): Promise<string> {
  conversationHistory.push({ role: 'user', content: userMessage });

  const response = await llm.chat({
    messages: conversationHistory,  // 이전 대화 전부 포함
  });

  conversationHistory.push({ role: 'assistant', content: response });
  return response;
}

// 대화가 길어지면 → 토큰 초과 → 오래된 대화 요약/삭제 필요
```

&nbsp;

### 5-2. 장기 메모리 (벡터DB)

&nbsp;

과거 대화, 사용자 선호도, 학습된 패턴 등을 영구 저장.

&nbsp;

```typescript
// 대화 후 핵심 정보를 벡터DB에 저장
await vectorDB.add({
  id: `memory-${Date.now()}`,
  text: '이 사용자는 환불 시 계좌이체를 선호함',
  metadata: { userId: 'U001', type: 'preference' },
});

// 다음 대화에서 관련 기억 검색
const memories = await vectorDB.query({
  text: '환불 처리',
  filter: { userId: 'U001' },
});
// → ['이 사용자는 환불 시 계좌이체를 선호함']
```

&nbsp;

```
┌──────────────────────────────────────────────┐
│              Agent 메모리 구조                 │
│                                               │
│  ┌─────────────┐    ┌──────────────────────┐ │
│  │  단기 메모리  │    │     장기 메모리        │ │
│  │  (Context)   │    │    (벡터DB)           │ │
│  │             │    │                       │ │
│  │ 현재 대화    │    │ 과거 대화 요약         │ │
│  │ 이번 세션    │    │ 사용자 선호도          │ │
│  │ 작업 컨텍스트│    │ 학습된 패턴            │ │
│  │             │    │ 사내 문서 (RAG)        │ │
│  │ 휘발성      │    │ 영구 저장              │ │
│  └─────────────┘    └──────────────────────┘ │
└──────────────────────────────────────────────┘
```

&nbsp;

&nbsp;

---

&nbsp;

## 6. 아키텍처 다이어그램 3가지

&nbsp;

### 6-1. 단순 구조 (LLM만)

&nbsp;

```
┌──────────┐       ┌──────────────┐
│  사용자   │ ────► │   LLM API    │
│          │ ◄──── │ (GPT/Claude) │
└──────────┘       └──────────────┘

비용: $50~200/월 (약 7만~29만원)
용도: 간단한 Q&A, 텍스트 생성
한계: 사내 정보 모름, 외부 시스템 접근 불가
```

&nbsp;

### 6-2. RAG 구조 (LLM + 벡터DB)

&nbsp;

```
┌──────────┐       ┌──────────────┐       ┌──────────────┐
│  사용자   │ ────► │  Agent 서버   │ ────► │   LLM API    │
│          │ ◄──── │              │ ◄──── │              │
└──────────┘       │   ┌──────┐  │       └──────────────┘
                   │   │ RAG  │  │
                   │   │ 검색 │──┼──────► ┌──────────────┐
                   │   └──────┘  │       │   벡터DB      │
                   └──────────────┘       │ (Pinecone 등) │
                                          └──────────────┘

비용: $100~500/월 (약 14만~72만원)
용도: 사내 문서 기반 Q&A, 고객 상담
한계: 외부 시스템 "조작"은 불가
```

&nbsp;

### 6-3. 멀티 에이전트 구조 (풀 스택)

&nbsp;

```
                    ┌───────────────────────────────────┐
                    │          오케스트레이터             │
                    │   (작업 분배 + 결과 취합)           │
                    └──────┬──────────┬──────────┬──────┘
                           │          │          │
                    ┌──────▼───┐ ┌────▼─────┐ ┌──▼──────────┐
                    │ 분석 Agent│ │ 실행 Agent│ │ 보고 Agent   │
                    │          │ │          │ │             │
                    │ RAG 검색 │ │ Tool Use │ │ 요약 + 전송  │
                    │ 데이터분석│ │ API 호출  │ │ Slack/이메일 │
                    └──────────┘ └──────────┘ └─────────────┘
                         │            │              │
                    ┌────▼───┐  ┌─────▼────┐  ┌──────▼──────┐
                    │ 벡터DB │  │ 외부 API  │  │  알림 시스템  │
                    │ LLM API│  │ DB, CRM   │  │             │
                    └────────┘  └──────────┘  └─────────────┘

비용: $500~3,000/월 (약 72만~429만원)
용도: 복잡한 업무 자동화, 장애 대응, 고객 응대
구성: 역할별 Agent가 협업
```

&nbsp;

&nbsp;

---

&nbsp;

## 7. 프레임워크 비교

&nbsp;

직접 API를 호출해서 만들 수도 있지만, 프레임워크를 쓰면 빠르게 만들 수 있다.

&nbsp;

### 7-1. LangChain

&nbsp;

가장 넓은 생태계. "Agent 만들기의 React" 같은 존재.

&nbsp;

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool

@tool
def lookup_order(order_id: str) -> dict:
    """주문번호로 주문 상태를 조회합니다."""
    return {"order_id": order_id, "status": "배송중", "eta": "2025-04-17"}

@tool
def process_refund(order_id: str, reason: str) -> dict:
    """주문에 대한 환불을 처리합니다."""
    return {"success": True, "refund_amount": 29900}

llm = ChatAnthropic(model="claude-sonnet-4-20250514")
tools = [lookup_order, process_refund]
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = executor.invoke({"input": "ORD-1234 환불해주세요"})
# Agent가 자동으로:
# 1. lookup_order("ORD-1234") 호출
# 2. 주문 상태 확인
# 3. process_refund("ORD-1234", "고객 요청") 호출
# 4. 결과를 자연어로 응답
```

&nbsp;

### 7-2. LlamaIndex

&nbsp;

RAG에 특화. 문서 인덱싱과 검색이 핵심.

&nbsp;

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# 폴더의 모든 문서를 자동으로 인덱싱
documents = SimpleDirectoryReader("./company-docs").load_data()
index = VectorStoreIndex.from_documents(documents)

# 쿼리 엔진으로 질문
query_engine = index.as_query_engine()
response = query_engine.query("환불 정책이 어떻게 되나요?")
print(response)
# → "구매 후 14일 이내 전액 환불 가능합니다. 개봉 제품은 7일 이내..."
```

&nbsp;

### 7-3. CrewAI

&nbsp;

멀티 에이전트 협업에 특화. 역할별 Agent를 정의하고 협업시킨다.

&nbsp;

```python
from crewai import Agent, Task, Crew

# 역할별 Agent 정의
analyst = Agent(
    role="데이터 분석가",
    goal="서버 에러 로그를 분석하여 근본 원인을 찾는다",
    backstory="10년 경력의 시스템 엔지니어",
    tools=[log_search_tool, metric_query_tool],
)

developer = Agent(
    role="개발자",
    goal="분석 결과를 바탕으로 버그를 수정한다",
    backstory="시니어 백엔드 개발자",
    tools=[code_edit_tool, git_tool],
)

reporter = Agent(
    role="보고 담당",
    goal="장애 대응 결과를 팀에 보고한다",
    backstory="기술 커뮤니케이션 전문가",
    tools=[slack_tool, email_tool],
)

# Task 정의
analyze_task = Task(description="에러 로그 분석", agent=analyst)
fix_task = Task(description="버그 수정 PR 생성", agent=developer)
report_task = Task(description="Slack에 결과 보고", agent=reporter)

# Crew 실행 (순차)
crew = Crew(
    agents=[analyst, developer, reporter],
    tasks=[analyze_task, fix_task, report_task],
    verbose=True,
)
result = crew.kickoff()
```

&nbsp;

### 프레임워크 비교표

&nbsp;

| | LangChain | LlamaIndex | CrewAI |
|------|-----------|------------|--------|
| 강점 | 범용, 넓은 생태계 | RAG 특화 | 멀티 에이전트 |
| 학습 곡선 | 높음 | 중간 | 낮음 |
| 유연성 | 매우 높음 | RAG 내에서 높음 | 멀티 에이전트 내에서 높음 |
| 언어 | Python, JS/TS | Python | Python |
| 적합한 경우 | 범용 Agent | 문서 검색 중심 | 역할 분담 자동화 |
| GitHub Stars | 100K+ | 40K+ | 25K+ |

&nbsp;

### 선택 가이드

&nbsp;

```
"사내 문서 Q&A를 만들겠다"         → LlamaIndex
"고객 상담 Agent를 만들겠다"       → LangChain
"여러 Agent가 협업해야 한다"       → CrewAI
"프레임워크 없이 가볍게 만들겠다"   → 직접 API 호출
```

&nbsp;

&nbsp;

---

&nbsp;

## 8. 정리 — Agent 아키텍처 체크리스트

&nbsp;

| 구성 요소 | 역할 | 필수 여부 |
|-----------|------|-----------|
| LLM API | 두뇌 (판단, 생성) | 필수 |
| RAG (벡터DB) | 사내 지식 검색 | 사내 데이터 다루면 필수 |
| Tool Use | 외부 시스템 연동 | Agent라면 필수 |
| 프롬프트 체인 | 복잡한 작업 분할 | 멀티스텝이면 필수 |
| 단기 메모리 | 대화 맥락 유지 | 대화형이면 필수 |
| 장기 메모리 | 과거 기억 | 개인화가 필요하면 |
| 프레임워크 | 빠른 개발 | 선택 (직접 구현도 가능) |

&nbsp;

Agent 아키텍처를 한 문장으로 정리하면:

> **LLM이 판단하고, RAG로 사내 지식을 참조하고, Tool Use로 실행한다.**

&nbsp;

이 3가지 조합만 이해하면, 어떤 Agent든 만들 수 있다.

차이는 "어떤 도구를 연결하느냐"와 "얼마나 자율적으로 실행하느냐"뿐이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI Agent 실전 가이드 3편] 사무직 Agent — 이메일, 문서, 데이터 자동화**

&nbsp;

실제 구현으로 들어간다. 이메일 자동 분류, 회의록 생성, 데이터 분석 리포트, 사내 규정 Q&A를 n8n + LLM API로 만드는 방법을 코드와 함께 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

AI에이전트, AI Agent, RAG, Tool Use, Function Calling, MCP, LangChain, LlamaIndex, CrewAI, 벡터DB, 임베딩, 프롬프트체인, 아키텍처
