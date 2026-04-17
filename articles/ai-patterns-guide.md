# AI는 어떻게 생각하고 행동하는가? — CoT, Tool Use, ReAct, MCP 완전 정리

> "AI가 알아서 해주는 거 아니야?" 라고 생각했다면, 이 글을 읽어보세요.
> AI가 똑똑하게 동작하는 건 마법이 아니라 **설계된 패턴** 덕분입니다.

---

## 목차
1. [들어가며 — AI는 만능이 아니다](#1-들어가며)
2. [CoT — AI에게 "생각"을 시키는 법](#2-cot)
3. [Tool Use — AI에게 "도구"를 쥐어주는 법](#3-tool-use)
4. [ReAct — AI가 "스스로 판단"하게 하는 법](#4-react)
5. [MCP — AI에게 "외부 시스템"을 연결하는 법](#5-mcp)
6. [실전 예제 — 사내 챗봇 만들기](#6-실전-예제)
7. [정리 — 언제 뭘 쓰나?](#7-정리)

---

## 1. 들어가며

ChatGPT나 Claude를 쓰면서 이런 경험 해보셨을 겁니다.

- "우리 DB에서 오늘 매출 알려줘" → **못 합니다** (DB 접근 권한이 없으니까)
- "이 코드 분석해줘" → **잘 합니다** (텍스트 분석은 잘하니까)
- "버그 찾아서 고쳐줘" → **반반입니다** (코드를 읽을 순 있지만, 파일을 수정할 권한이 없을 수도 있으니까)

AI는 기본적으로 **텍스트를 받아서 텍스트를 내뱉는 모델**입니다.
그런데 어떻게 Claude Code 같은 도구는 파일을 읽고, 검색하고, 코드를 수정할까요?

그 답이 바로 **CoT, Tool Use, ReAct, MCP** 입니다.

---

## 2. CoT (Chain of Thought) — AI에게 "생각"을 시키는 법

### 한 줄 요약
> "답부터 말하지 말고, 단계별로 생각해봐"

### 일반 질문 vs CoT 질문

**일반 질문:**
```
Q: 서울에서 부산까지 KTX로 얼마나 걸려?
A: 약 2시간 30분입니다.
```

**CoT 질문:**
```
Q: 서울에서 부산까지 KTX로 얼마나 걸려? 단계별로 생각해봐.
A: 
1. 서울역에서 부산역까지 KTX 노선 거리는 약 417km
2. KTX 최고 속도는 305km/h이지만 평균 속도는 약 170km/h
3. 417km / 170km/h ≈ 2.45시간
4. 정차역 시간 포함하면 약 2시간 30분~2시간 50분
```

### 코드에서의 CoT

```typescript
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  messages: [{
    role: "user",
    content: `다음 코드의 버그를 찾아줘. 
              단계별로 분석해줘:
              1. 코드가 뭘 하려는 건지
              2. 입력값이 어떻게 흘러가는지
              3. 어디서 문제가 생기는지
              4. 수정 방법
              
              ${code}`
  }]
});
```

### 핵심 포인트
- AI에게 "생각의 과정"을 요구하면 정확도가 올라간다
- 복잡한 문제일수록 효과가 큼
- **비용**: 추가 비용 없음 (프롬프트만 바꾸면 됨)

---

## 3. Tool Use — AI에게 "도구"를 쥐어주는 법

### 한 줄 요약
> "AI야, 네가 직접 못하는 건 이 도구를 써"

### AI의 한계

AI는 기본적으로 이런 것들을 **못 합니다**:
- DB 조회
- 파일 읽기/쓰기
- API 호출
- 계산기 (의외로 큰 숫자 계산을 틀림)

그래서 **도구(Tool)를 정의해서 쥐어줍니다**.

### 동작 원리

```
사용자: "오늘 주문 현황 알려줘"

   ↓

AI 판단: "DB를 조회해야 하는데, 내가 직접은 못 하지. 
         getOrders 도구를 쓰자"

   ↓

AI → 서버: { tool: "getOrders", params: { date: "2026-04-16" } }

   ↓

서버: DB 조회 후 결과 반환
      { total: 42, completed: 35, pending: 7 }

   ↓

AI: "오늘 주문 현황입니다. 
     총 42건 중 완료 35건, 대기 7건입니다."
```

**중요한 건**: AI가 도구를 직접 실행하는 게 아닙니다.
AI는 "이 도구를 이 파라미터로 호출해줘"라고 **요청만** 하고, **서버(우리 코드)가 실행**합니다.

### 실제 코드

```typescript
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic();

// 1단계: 도구를 정의한다
const tools = [
  {
    name: "getWeather",
    description: "특정 도시의 현재 날씨를 조회합니다",
    input_schema: {
      type: "object",
      properties: {
        city: {
          type: "string",
          description: "도시명 (예: Seoul, Tokyo)"
        }
      },
      required: ["city"]
    }
  }
];

// 2단계: AI에게 도구와 함께 질문한다
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 1024,
  tools: tools,
  messages: [
    { role: "user", content: "서울 날씨 어때?" }
  ]
});

// 3단계: AI가 도구 사용을 요청하면 실행해준다
for (const block of response.content) {
  if (block.type === "tool_use") {
    console.log(`도구: ${block.name}`);       // "getWeather"
    console.log(`입력: ${JSON.stringify(block.input)}`); // { city: "Seoul" }
    
    // 여기서 실제로 날씨 API를 호출한다
    // const result = await fetchWeather(block.input.city);
  }
}
```

### 핵심 포인트
- AI가 **언제 어떤 도구를 쓸지 스스로 판단**한다
- 개발자는 **도구의 이름, 설명, 파라미터만 정의**하면 됨
- AI는 도구를 직접 실행하지 않음 → **서버가 실행하고 결과를 돌려줌**
- 도구를 잘 설명해줄수록(description) AI가 정확하게 판단함

---

## 4. ReAct — AI가 "스스로 판단"하게 하는 법

### 한 줄 요약
> "생각(Reasoning) → 행동(Action) → 관찰(Observation)을 반복"

### Tool Use와 뭐가 다른가?

**Tool Use**: AI가 도구를 **한 번** 쓰고 끝
**ReAct**: AI가 결과를 보고 **다음에 뭘 할지 스스로 판단**하는 루프

### 예시 — 여행 계획 세우기

```
사용자: "이번 주말 부산 여행 계획 세워줘"

[생각] 먼저 이번 주말 부산 날씨를 확인해야겠다.
[행동] getWeather("Busan") 호출
[관찰] 토요일 맑음 25도, 일요일 비 18도

[생각] 토요일은 야외, 일요일은 실내 위주로 잡자.
       인기 관광지를 검색해보자.
[행동] searchPlaces("부산 관광지") 호출
[관찰] 해운대, 감천문화마을, 자갈치시장, 부산시립미술관...

[생각] 토요일에 해운대+감천문화마을, 
       일요일에 미술관+자갈치시장이 좋겠다.

[결론] 
"토요일 (맑음): 오전 감천문화마을 → 오후 해운대
 일요일 (비): 오전 부산시립미술관 → 오후 자갈치시장"
```

AI가 한번에 답한 게 아니라, **도구 결과를 보고 다음 행동을 판단**하면서 진행합니다.

### 코드로 구현하면

```typescript
async function reactLoop(question: string) {
  let messages = [{ role: "user", content: question }];

  while (true) {
    // AI에게 물어본다
    const response = await anthropic.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 4096,
      tools: tools,
      messages: messages
    });

    // 도구 사용 요청이 있으면
    const toolUse = response.content.find(b => b.type === "tool_use");

    if (toolUse) {
      // 도구 실행
      const result = await executeTool(toolUse.name, toolUse.input);

      // 결과를 대화에 추가하고 다시 AI에게 물어봄
      messages.push({ role: "assistant", content: response.content });
      messages.push({
        role: "user",
        content: [{
          type: "tool_result",
          tool_use_id: toolUse.id,
          content: JSON.stringify(result)
        }]
      });

      // → 다시 while 루프 처음으로 (AI가 다음 행동 판단)
    } else {
      // 도구 사용 없이 텍스트 응답 = 최종 답변
      const textBlock = response.content.find(b => b.type === "text");
      console.log(textBlock.text);
      break;
    }
  }
}
```

### 핵심 포인트
- Tool Use에 **while 루프만 추가**하면 ReAct가 된다
- AI가 "더 조사할 게 있다" 싶으면 도구를 또 호출한다
- "이제 충분하다" 싶으면 텍스트로 최종 답변한다
- **Claude Code가 정확히 이 방식으로 동작합니다**

---

## 5. MCP (Model Context Protocol) — AI에게 "외부 시스템"을 연결하는 법

### 한 줄 요약
> "AI가 쓸 수 있는 도구를 표준화된 방식으로 제공하는 프로토콜"

### Tool Use와 뭐가 다른가?

**Tool Use**: 내 코드 안에서 직접 도구를 정의
**MCP**: 외부 서버가 도구를 제공하고, AI가 연결해서 사용

비유하자면:
- Tool Use = 내 컴퓨터에 프로그램을 설치하는 것
- MCP = USB로 외부 장치를 꽂아서 쓰는 것

### 구조

```
┌──────────┐     ┌─────────────┐     ┌──────────────┐
│  AI 모델  │ ←→ │  MCP Client  │ ←→ │  MCP Server   │
│ (Claude)  │     │ (내 앱)      │     │ (도구 제공자)  │
└──────────┘     └─────────────┘     └──────────────┘
                                           │
                                     ┌─────┴─────┐
                                     │  DB 조회   │
                                     │  파일 읽기  │
                                     │  API 호출   │
                                     └───────────┘
```

### MCP 서버 예시

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "my-tools",
  version: "1.0.0"
});

// 도구 등록
server.tool(
  "getWeather",
  "특정 도시의 현재 날씨를 조회합니다",
  { city: z.string().describe("도시명") },
  async ({ city }) => {
    const weather = await fetchWeatherAPI(city);
    return {
      content: [{ type: "text", text: JSON.stringify(weather) }]
    };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

### MCP가 강력한 이유

1. **재사용**: 한번 만든 MCP 서버를 여러 AI 앱에서 갖다 쓸 수 있음
2. **분리**: AI 로직과 도구 로직을 완전히 분리
3. **생태계**: 이미 만들어진 MCP 서버가 많음 (GitHub, Slack, DB, Google Drive 등)
4. **표준**: 어떤 AI 모델이든 같은 방식으로 연결 가능

---

## 6. 실전 예제 — 사내 챗봇 만들기

"오늘 매출 알려줘" 라고 물으면 DB를 조회해서 답해주는 챗봇을 만든다고 가정합니다.

### 전체 흐름

```
사용자: "오늘 매출은?"
          │
          ▼
   ┌─────────────┐
   │  내 서버      │  (Express 등)
   │  POST /chat  │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │ Claude API   │  AI가 판단:
   │ + Tool Use   │  "getSales 도구를 호출해야겠다"
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │  DB 조회     │  SELECT * FROM sales WHERE ...
   └──────┬──────┘
          │ { total: 1500000, count: 42 }
          ▼
   ┌─────────────┐
   │ Claude API   │  결과를 자연어로 변환
   └──────┬──────┘
          │
          ▼
사용자: "오늘 매출은 150만원, 총 42건입니다."
```

### 전체 코드

```typescript
import Anthropic from "@anthropic-ai/sdk";
import express from "express";

const app = express();
const anthropic = new Anthropic();

app.use(express.json());

// 1. 도구 정의
const tools: Anthropic.Tool[] = [
  {
    name: "getSales",
    description: "매출 현황을 조회합니다",
    input_schema: {
      type: "object" as const,
      properties: {
        date: { type: "string", description: "조회 날짜 (YYYY-MM-DD)" }
      },
      required: ["date"]
    }
  },
  {
    name: "getOrders",
    description: "주문 목록을 조회합니다",
    input_schema: {
      type: "object" as const,
      properties: {
        status: { type: "string", description: "주문 상태 (pending, completed, all)" }
      },
      required: ["status"]
    }
  }
];

// 2. 도구 실행 함수
async function executeTool(name: string, input: any) {
  switch (name) {
    case "getSales":
      // 실제로는 DB 조회
      return { total: 1500000, count: 42 };
    case "getOrders":
      return { orders: [{ id: 1, item: "상품A", amount: 30000 }] };
    default:
      return { error: "unknown tool" };
  }
}

// 3. 챗봇 API (ReAct 루프)
app.post("/chat", async (req, res) => {
  const { message } = req.body;

  let messages: Anthropic.MessageParam[] = [
    { role: "user", content: message }
  ];

  while (true) {
    const response = await anthropic.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 1024,
      system: "당신은 사내 업무를 도와주는 AI 어시스턴트입니다. 한국어로 답변하세요.",
      tools: tools,
      messages: messages
    });

    const toolUseBlock = response.content.find(
      (block): block is Anthropic.ToolUseBlock => block.type === "tool_use"
    );

    if (toolUseBlock) {
      const toolResult = await executeTool(toolUseBlock.name, toolUseBlock.input);

      messages.push({ role: "assistant", content: response.content });
      messages.push({
        role: "user",
        content: [{
          type: "tool_result",
          tool_use_id: toolUseBlock.id,
          content: JSON.stringify(toolResult)
        }]
      });
    } else {
      const textBlock = response.content.find(
        (block): block is Anthropic.TextBlock => block.type === "text"
      );
      res.json({ answer: textBlock?.text || "" });
      break;
    }
  }
});

app.listen(3000, () => console.log("서버 시작: http://localhost:3000"));
```

### 테스트

```bash
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "오늘 매출 알려줘"}'

# 응답
# { "answer": "오늘 매출은 150만원이며, 총 42건의 거래가 있었습니다." }
```

### 이 코드에서 일어나는 일

1. 사용자가 "오늘 매출 알려줘" 전송
2. Claude가 메시지를 보고 `getSales` 도구를 호출하기로 판단
3. 우리 서버가 `executeTool`에서 DB 조회 실행
4. 결과를 Claude에게 돌려줌
5. Claude가 결과를 자연어로 변환하여 응답

**핵심은 `executeTool` 함수입니다.** 여기에 실제 DB 조회, 외부 API 호출 등을 연결하면 됩니다.

---

## 7. 정리 — 언제 뭘 쓰나?

### 패턴 비교

| 패턴 | 하는 일 | 난이도 | 비유 |
|------|--------|--------|------|
| **CoT** | AI에게 생각 과정을 요구 | 쉬움 | "풀이 과정을 써라" |
| **Tool Use** | AI에게 도구를 쥐어줌 | 중간 | "계산기 써도 돼" |
| **ReAct** | AI가 판단→행동→관찰 반복 | 중간 | "알아서 조사해와" |
| **MCP** | 외부 도구를 표준화하여 연결 | 높음 | "USB 꽂아서 쓰기" |

### 내 프로젝트에 적용하려면?

| 하고 싶은 것 | 필요한 패턴 |
|-------------|-----------|
| AI에게 코드 리뷰만 시키기 | CoT만으로 충분 |
| AI가 DB 조회해서 답변하기 | Tool Use |
| AI가 알아서 여러 API 호출하기 | ReAct + Tool Use |
| 챗봇에 Slack/GitHub 등 연동 | MCP |
| 복잡한 워크플로우 자동화 | ReAct + Tool Use + MCP |

### 시작 추천 순서

```
1단계: CoT       — 프롬프트만 잘 써도 효과 큼 (비용 0원)
2단계: Tool Use  — Claude API + 도구 1~2개 (가장 실용적)
3단계: ReAct     — Tool Use에 while 루프만 추가
4단계: MCP       — 여러 시스템을 연결해야 할 때
```

대부분의 프로젝트는 **2단계(Tool Use)만으로도 충분**합니다.
거기에 while 루프 하나 감싸면 3단계(ReAct)가 되고, 그게 Claude Code의 동작 방식입니다.

---

## 참고

- [Anthropic Tool Use 공식 문서](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview)
- [MCP 공식 사이트](https://modelcontextprotocol.io)
- [Claude API 시작하기](https://docs.anthropic.com/en/api/getting-started)
