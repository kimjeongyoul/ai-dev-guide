# [AI Agent 실전 가이드 4편] 고객 응대 Agent — 챗봇, 키오스크, 콜센터

&nbsp;

기존 챗봇을 써본 사람이라면 이 경험이 있을 것이다.

&nbsp;

"환불하고 싶은데요"

→ "환불에 대해 안내해드리겠습니다. 1번: 환불 규정 2번: 환불 신청 3번: 상담원 연결"

→ "2번"

→ "환불 신청은 마이페이지 > 주문내역 > 환불신청에서 가능합니다."

→ "아니 그냥 지금 바로 해줘"

→ "죄송합니다. 상담원을 연결해드리겠습니다. 현재 대기 인원: 23명"

&nbsp;

**AI Agent 챗봇은 이렇게 한다:**

&nbsp;

"환불하고 싶은데요"

→ "어떤 주문을 환불하시겠어요? 최근 주문 내역을 확인해보겠습니다."

→ (주문 DB 조회) "4월 12일에 주문하신 A상품(29,900원)이 있네요. 이 주문을 환불하시겠어요?"

→ "네"

→ (환불 API 호출) "환불 처리되었습니다. 29,900원이 3~5영업일 내 원래 결제 수단으로 환급됩니다."

&nbsp;

**이것이 차이다.** 안내가 아니라 **처리**.

&nbsp;

&nbsp;

---

&nbsp;

## 1. 기존 챗봇 vs AI Agent 챗봇

&nbsp;

### 1-1. 규칙 기반 챗봇의 한계

&nbsp;

```typescript
// 규칙 기반 챗봇: if-else의 지옥
function handleMessage(message: string): string {
  if (message.includes('영업시간')) {
    return '평일 09:00~18:00입니다.';
  }
  if (message.includes('환불') && message.includes('기간')) {
    return '구매 후 14일 이내 환불 가능합니다.';
  }
  if (message.includes('환불') && message.includes('방법')) {
    return '마이페이지 > 주문내역에서 신청하세요.';
  }
  if (message.includes('배송') && message.includes('언제')) {
    return '주문 후 1~3 영업일 내 배송됩니다.';
  }
  // ... 시나리오가 늘어날수록 if-else도 늘어남
  
  return '죄송합니다. 상담원을 연결해드리겠습니다.';
}
```

&nbsp;

**문제점:**
- 시나리오 100개 → if-else 100개. 유지보수 지옥
- 조금만 다르게 말하면 매칭 실패 ("반품" vs "환불" vs "돌려받기")
- 새로운 질문 유형 → 개발자가 코드 수정 → 배포
- 멀티 턴 대화 불가 ("아까 그 주문 말이에요" → 이해 못함)

&nbsp;

### 1-2. AI Agent 챗봇의 구조

&nbsp;

```typescript
// AI Agent 챗봇: LLM이 판단, 도구가 실행
async function handleMessage(
  userId: string,
  message: string,
  history: Message[]
): Promise<string> {
  // 1. 사용자 컨텍스트 로드
  const userInfo = await getUserInfo(userId);
  
  // 2. RAG로 관련 정책 검색
  const policies = await vectorDB.search(message);
  
  // 3. LLM에 판단 요청 (도구 포함)
  const response = await llm.chat({
    system: `당신은 고객 상담 Agent입니다.
고객 정보: ${JSON.stringify(userInfo)}
관련 정책: ${policies.join('\n')}

고객의 요청을 직접 처리하세요. 도구를 사용해서 실제 업무를 수행합니다.
처리할 수 없는 경우에만 상담원을 연결하세요.`,
    messages: history.concat({ role: 'user', content: message }),
    tools: [
      { name: 'lookupOrder', fn: orderAPI.lookup },
      { name: 'processRefund', fn: paymentAPI.refund },
      { name: 'trackDelivery', fn: deliveryAPI.track },
      { name: 'issueCoupon', fn: couponAPI.issue },
      { name: 'updateReservation', fn: reservationAPI.update },
      { name: 'escalateToHuman', fn: supportAPI.escalate },
    ],
  });

  return response.text;
}
```

&nbsp;

### 1-3. 비교표

&nbsp;

| 항목 | 규칙 기반 챗봇 | AI Agent 챗봇 |
|------|---------------|---------------|
| 자연어 이해 | 키워드 매칭 | 문맥 이해 |
| 새 질문 대응 | 코드 수정 필요 | 프롬프트 수정으로 가능 |
| 실제 업무 처리 | 불가 (안내만) | 가능 (API 호출) |
| 멀티 턴 대화 | 제한적 | 자연스러움 |
| 유지보수 | if-else 추가 | 프롬프트 + 도구 |
| 비용 | 서버비만 | 서버비 + LLM API |
| 응답 일관성 | 완벽 (규칙이니까) | 높지만 100%는 아님 |

&nbsp;

&nbsp;

---

&nbsp;

## 2. 아키텍처

&nbsp;

```
┌──────────────────────────────────────────────────────────────────────┐
│                    고객 응대 Agent 아키텍처                            │
│                                                                       │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────────────────┐   │
│  │  고객     │    │  Gateway     │    │      Agent 서버            │   │
│  │          │◄──►│              │◄──►│                           │   │
│  │ 웹/앱/   │ ws │ WebSocket    │    │  ┌─────────────────────┐  │   │
│  │ 키오스크  │    │ 서버         │    │  │  대화 관리자          │  │   │
│  │ 전화     │    │              │    │  │  (Context Manager)   │  │   │
│  └──────────┘    └──────────────┘    │  └─────────┬───────────┘  │   │
│                                       │            │              │   │
│                                       │  ┌─────────▼───────────┐  │   │
│                                       │  │     LLM API         │  │   │
│                                       │  │  (판단 + 도구 호출)   │  │   │
│                                       │  └─────────┬───────────┘  │   │
│                                       │            │              │   │
│                                       │  ┌─────────▼───────────┐  │   │
│                                       │  │    Tool Router       │  │   │
│                                       │  └──┬────┬────┬────┬──┘  │   │
│                                       └─────┼────┼────┼────┼─────┘   │
│                                             │    │    │    │          │
│                                       ┌─────▼┐ ┌─▼──┐│  ┌─▼────────┐│
│                                       │주문DB│ │결제││  │상담원연결 ││
│                                       │      │ │API ││  │(Fallback)││
│                                       └──────┘ └────┘│  └──────────┘│
│                                              ┌───────▼──┐           │
│                                              │ 벡터DB   │           │
│                                              │(FAQ/정책)│           │
│                                              └──────────┘           │
└──────────────────────────────────────────────────────────────────────┘
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. 구현 요소

&nbsp;

### 3-1. RAG: FAQ, 상품 정보, 정책 문서 인덱싱

&nbsp;

```typescript
// 사내 문서를 벡터DB에 인덱싱
import { ChromaClient } from 'chromadb';

const chroma = new ChromaClient();
const collection = await chroma.createCollection({ name: 'customer-support' });

// FAQ 인덱싱
const faqs = [
  { id: 'faq-1', text: '환불은 구매 후 14일 이내 가능합니다. 개봉 제품은 7일 이내.', category: 'refund' },
  { id: 'faq-2', text: '배송은 주문 후 1~3영업일 소요됩니다. 도서산간 추가 2일.', category: 'delivery' },
  { id: 'faq-3', text: '회원 등급은 최근 6개월 구매금액 기준으로 매월 1일 갱신됩니다.', category: 'membership' },
  // ... 수백 개 FAQ
];

await collection.add({
  ids: faqs.map(f => f.id),
  documents: faqs.map(f => f.text),
  metadatas: faqs.map(f => ({ category: f.category })),
});

// 상품 정보 인덱싱
const products = await db.query('SELECT * FROM products WHERE active = 1');
await collection.add({
  ids: products.map(p => `product-${p.id}`),
  documents: products.map(p => `${p.name}: ${p.description}. 가격 ${p.price}원. ${p.specs}`),
  metadatas: products.map(p => ({ category: 'product', productId: p.id })),
});
```

&nbsp;

### 3-2. Tool Use: 주문 조회, 환불 처리, 예약 변경

&nbsp;

```typescript
// 도구 정의: Agent가 호출할 수 있는 함수들
const tools = [
  {
    name: 'lookupOrder',
    description: '고객의 주문을 조회합니다. 주문번호 또는 고객ID로 검색 가능.',
    parameters: {
      orderId: { type: 'string', optional: true },
      customerId: { type: 'string', optional: true },
      limit: { type: 'number', default: 5 },
    },
    handler: async ({ orderId, customerId, limit }) => {
      if (orderId) {
        return await db.query('SELECT * FROM orders WHERE id = ?', [orderId]);
      }
      return await db.query(
        'SELECT * FROM orders WHERE customer_id = ? ORDER BY created_at DESC LIMIT ?',
        [customerId, limit]
      );
    },
  },
  {
    name: 'processRefund',
    description: '주문에 대한 환불을 처리합니다. 환불 사유가 필요합니다.',
    parameters: {
      orderId: { type: 'string', required: true },
      reason: { type: 'string', required: true },
      amount: { type: 'number', optional: true, description: '부분 환불 금액. 미지정 시 전액' },
    },
    handler: async ({ orderId, reason, amount }) => {
      // 환불 가능 여부 확인
      const order = await db.query('SELECT * FROM orders WHERE id = ?', [orderId]);
      if (!order) return { success: false, error: '주문을 찾을 수 없습니다.' };
      
      const daysSincePurchase = daysBetween(order.createdAt, new Date());
      if (daysSincePurchase > 14) {
        return { success: false, error: '환불 기간(14일)이 지났습니다.' };
      }
      
      const refundAmount = amount || order.totalAmount;
      await paymentAPI.refund(order.paymentId, refundAmount, reason);
      await db.update('orders', { id: orderId }, { status: 'refunded' });
      
      return { success: true, refundAmount, estimatedDate: '3~5영업일' };
    },
  },
  {
    name: 'trackDelivery',
    description: '배송 상태를 추적합니다.',
    parameters: {
      trackingNumber: { type: 'string', required: true },
    },
    handler: async ({ trackingNumber }) => {
      return await deliveryAPI.track(trackingNumber);
    },
  },
  {
    name: 'escalateToHuman',
    description: 'AI가 처리할 수 없는 경우 상담원에게 연결합니다.',
    parameters: {
      reason: { type: 'string', required: true },
      priority: { type: 'string', enum: ['normal', 'urgent'] },
    },
    handler: async ({ reason, priority }) => {
      const ticket = await supportAPI.createTicket({ reason, priority });
      return { ticketId: ticket.id, estimatedWait: ticket.estimatedWait };
    },
  },
];
```

&nbsp;

### 3-3. Fallback: AI가 못 하면 상담원 연결

&nbsp;

```typescript
// Fallback 전략
const ESCALATION_RULES = {
  // 자동 에스컬레이션 조건
  autoEscalate: [
    '결제 오류',      // 금전 관련 → 무조건 상담원
    '법적 분쟁',      // 법률 관련 → 무조건 상담원
    '개인정보 유출',   // 보안 사고 → 무조건 상담원
  ],
  
  // 대화 품질 기반 에스컬레이션
  qualityCheck: {
    maxTurns: 5,          // 5번 대화해도 해결 안 되면
    maxToolFailures: 2,   // 도구 호출이 2번 실패하면
    customerFrustration: true, // 고객 감정이 부정적이면
  },
};

async function shouldEscalate(
  conversation: Message[],
  lastResponse: AgentResponse
): Promise<boolean> {
  // 1. 규칙 기반 체크
  const lastMessage = conversation[conversation.length - 1].content;
  for (const keyword of ESCALATION_RULES.autoEscalate) {
    if (lastMessage.includes(keyword)) return true;
  }
  
  // 2. 대화 길이 체크
  if (conversation.length > ESCALATION_RULES.qualityCheck.maxTurns * 2) {
    return true;
  }
  
  // 3. 고객 감정 분석
  const sentiment = await analyzeSentiment(lastMessage);
  if (sentiment.frustration > 0.8) return true;
  
  return false;
}
```

&nbsp;

### 3-4. 멀티 언어: 6개 국어 대응

&nbsp;

```typescript
// 멀티 언어 지원: 언어 감지 → 해당 언어로 응답
const SYSTEM_PROMPT = `당신은 다국어 고객 상담 Agent입니다.

지원 언어: 한국어, English, 中文, 日本語, Русский, Монгол

규칙:
1. 고객이 사용하는 언어를 자동 감지하여 해당 언어로 응답하세요.
2. 도구 호출 결과는 고객의 언어로 번역하여 전달하세요.
3. 전문 용어는 원어를 병기하세요. 예: "환불(Refund)"
4. 통화는 고객의 국가에 맞게 표시하세요.`;

// 언어별 프롬프트 캐싱으로 비용 절약
const cachedPrompts = new Map<string, string>();

function getSystemPrompt(lang: string): string {
  if (!cachedPrompts.has(lang)) {
    const localized = SYSTEM_PROMPT + `\n\n현재 고객 언어: ${lang}`;
    cachedPrompts.set(lang, localized);
  }
  return cachedPrompts.get(lang)!;
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 4. 실전 구현: Node.js + WebSocket + Anthropic API

&nbsp;

```typescript
// customer-agent-server.ts
import express from 'express';
import { WebSocketServer, WebSocket } from 'ws';
import Anthropic from '@anthropic-ai/sdk';
import { createServer } from 'http';

const app = express();
const server = createServer(app);
const wss = new WebSocketServer({ server, path: '/ws' });

const anthropic = new Anthropic();

// 세션별 대화 기록 관리
const sessions = new Map<string, {
  messages: Anthropic.MessageParam[];
  customerId: string;
  language: string;
}>();

// 도구 정의
const tools: Anthropic.Tool[] = [
  {
    name: 'lookupOrder',
    description: '고객의 최근 주문을 조회합니다',
    input_schema: {
      type: 'object' as const,
      properties: {
        customerId: { type: 'string', description: '고객 ID' },
      },
      required: ['customerId'],
    },
  },
  {
    name: 'processRefund',
    description: '주문에 대한 환불을 처리합니다',
    input_schema: {
      type: 'object' as const,
      properties: {
        orderId: { type: 'string', description: '주문번호' },
        reason: { type: 'string', description: '환불 사유' },
      },
      required: ['orderId', 'reason'],
    },
  },
  {
    name: 'escalateToHuman',
    description: 'AI가 해결할 수 없어 상담원에게 연결합니다',
    input_schema: {
      type: 'object' as const,
      properties: {
        reason: { type: 'string', description: '연결 사유' },
      },
      required: ['reason'],
    },
  },
];

// 도구 실행 함수
async function executeTool(name: string, input: any): Promise<string> {
  switch (name) {
    case 'lookupOrder': {
      // 실제로는 DB 조회
      return JSON.stringify({
        orders: [
          { id: 'ORD-1234', product: 'A상품', amount: 29900, status: '배송완료', date: '2025-04-12' },
          { id: 'ORD-1235', product: 'B상품', amount: 15000, status: '배송중', date: '2025-04-15' },
        ],
      });
    }
    case 'processRefund': {
      return JSON.stringify({ success: true, refundAmount: 29900, estimatedDate: '3~5영업일' });
    }
    case 'escalateToHuman': {
      return JSON.stringify({ ticketId: 'TKT-567', estimatedWait: '약 3분' });
    }
    default:
      return JSON.stringify({ error: '알 수 없는 도구' });
  }
}

// WebSocket 연결 처리
wss.on('connection', (ws: WebSocket) => {
  const sessionId = crypto.randomUUID();
  sessions.set(sessionId, {
    messages: [],
    customerId: '',
    language: 'ko',
  });

  ws.send(JSON.stringify({
    type: 'connected',
    sessionId,
    message: '안녕하세요! 무엇을 도와드릴까요?',
  }));

  ws.on('message', async (data) => {
    const { text, customerId } = JSON.parse(data.toString());
    const session = sessions.get(sessionId)!;
    
    if (customerId) session.customerId = customerId;
    session.messages.push({ role: 'user', content: text });

    try {
      // Agent 루프: 도구 호출이 완료될 때까지 반복
      let response = await anthropic.messages.create({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 1024,
        system: `고객 상담 Agent입니다. 고객 ID: ${session.customerId}. 
도구를 사용해서 고객 요청을 직접 처리하세요. 
처리 불가 시에만 escalateToHuman을 호출하세요.`,
        tools,
        messages: session.messages,
      });

      // 도구 호출이 있으면 실행하고 결과를 다시 LLM에 전달
      while (response.stop_reason === 'tool_use') {
        const toolUse = response.content.find(
          (block): block is Anthropic.ToolUseBlock => block.type === 'tool_use'
        );
        
        if (!toolUse) break;

        // 도구 실행 중 상태 전송
        ws.send(JSON.stringify({ type: 'processing', tool: toolUse.name }));

        const result = await executeTool(toolUse.name, toolUse.input);
        
        // Assistant 메시지와 도구 결과를 대화에 추가
        session.messages.push({ role: 'assistant', content: response.content });
        session.messages.push({
          role: 'user',
          content: [{
            type: 'tool_result',
            tool_use_id: toolUse.id,
            content: result,
          }],
        });

        // 다시 LLM 호출
        response = await anthropic.messages.create({
          model: 'claude-sonnet-4-20250514',
          max_tokens: 1024,
          system: `고객 상담 Agent입니다. 고객 ID: ${session.customerId}.`,
          tools,
          messages: session.messages,
        });
      }

      // 최종 텍스트 응답 추출
      const textBlock = response.content.find(
        (block): block is Anthropic.TextBlock => block.type === 'text'
      );
      const reply = textBlock?.text || '죄송합니다, 다시 말씀해주세요.';

      session.messages.push({ role: 'assistant', content: reply });
      
      ws.send(JSON.stringify({ type: 'message', text: reply }));
    } catch (error) {
      ws.send(JSON.stringify({
        type: 'error',
        text: '일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요.',
      }));
    }
  });

  ws.on('close', () => {
    sessions.delete(sessionId);
  });
});

server.listen(3000, () => {
  console.log('Customer Agent Server running on port 3000');
});
```

&nbsp;

**클라이언트 (브라우저):**

&nbsp;

```typescript
// customer-chat.tsx
'use client';

import { useState, useEffect, useRef } from 'react';

interface ChatMessage {
  role: 'user' | 'agent';
  text: string;
  timestamp: Date;
}

export default function CustomerChat() {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [input, setInput] = useState('');
  const [isProcessing, setIsProcessing] = useState(false);
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const ws = new WebSocket('wss://your-server.com/ws');
    wsRef.current = ws;

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      
      switch (data.type) {
        case 'connected':
          setMessages([{
            role: 'agent',
            text: data.message,
            timestamp: new Date(),
          }]);
          break;
        case 'processing':
          setIsProcessing(true);
          break;
        case 'message':
          setIsProcessing(false);
          setMessages(prev => [...prev, {
            role: 'agent',
            text: data.text,
            timestamp: new Date(),
          }]);
          break;
      }
    };

    return () => ws.close();
  }, []);

  const sendMessage = () => {
    if (!input.trim() || !wsRef.current) return;
    
    setMessages(prev => [...prev, {
      role: 'user',
      text: input,
      timestamp: new Date(),
    }]);
    
    wsRef.current.send(JSON.stringify({
      text: input,
      customerId: 'CUST-001',
    }));
    
    setInput('');
  };

  return (
    <div className="flex flex-col h-screen">
      <div className="flex-1 overflow-y-auto p-4">
        {messages.map((msg, i) => (
          <div key={i} className={`mb-4 ${msg.role === 'user' ? 'text-right' : 'text-left'}`}>
            <div className={`inline-block p-3 rounded-lg max-w-[80%] ${
              msg.role === 'user' ? 'bg-blue-500 text-white' : 'bg-gray-100'
            }`}>
              {msg.text}
            </div>
          </div>
        ))}
        {isProcessing && (
          <div className="text-gray-400">처리 중...</div>
        )}
      </div>
      <div className="p-4 border-t flex gap-2">
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          onKeyDown={e => e.key === 'Enter' && sendMessage()}
          className="flex-1 border rounded-lg px-4 py-2"
          placeholder="메시지를 입력하세요"
        />
        <button onClick={sendMessage} className="bg-blue-500 text-white px-6 py-2 rounded-lg">
          전송
        </button>
      </div>
    </div>
  );
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 5. 키오스크/단말기 배포 고려사항

&nbsp;

고객 응대 Agent를 물리적 단말기에 배포할 때의 고려사항:

&nbsp;

```
┌─────────────────────────────────────┐
│           키오스크 단말기             │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  브라우저 (전체 화면)            │  │
│  │                                │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │  Agent 채팅 UI           │  │  │
│  │  │  (React + WebSocket)     │  │  │
│  │  └──────────────────────────┘  │  │
│  │                                │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │  오프라인 Fallback       │  │  │
│  │  │  (기본 FAQ 로컬 저장)     │  │  │
│  │  └──────────────────────────┘  │  │
│  └────────────────────────────────┘  │
│                                      │
│  [네트워크]──→ Agent 서버 ──→ LLM API │
│  [NFC 리더]    [카메라]    [프린터]    │
└─────────────────────────────────────┘
```

&nbsp;

```typescript
// 오프라인 Fallback: 네트워크 끊겼을 때 기본 응답
const OFFLINE_RESPONSES: Record<string, string> = {
  '영업시간': '평일 09:00~18:00, 주말 10:00~17:00입니다.',
  '위치': '1층 로비에서 엘리베이터를 이용해 3층으로 오시면 됩니다.',
  '전화번호': '대표 번호: 02-1234-5678',
};

function getOfflineResponse(message: string): string | null {
  for (const [keyword, response] of Object.entries(OFFLINE_RESPONSES)) {
    if (message.includes(keyword)) return response;
  }
  return '현재 네트워크 연결이 원활하지 않습니다. 잠시 후 다시 시도해주세요.';
}

// WebSocket 연결 상태 감시
function createResilientWebSocket(url: string) {
  let ws: WebSocket;
  let isOnline = false;
  let reconnectAttempts = 0;
  const MAX_RECONNECT = 5;

  function connect() {
    ws = new WebSocket(url);
    ws.onopen = () => { isOnline = true; reconnectAttempts = 0; };
    ws.onclose = () => {
      isOnline = false;
      if (reconnectAttempts < MAX_RECONNECT) {
        reconnectAttempts++;
        setTimeout(connect, 2000 * reconnectAttempts);  // 점진적 재연결
      }
    };
  }

  connect();
  return { ws: () => ws, isOnline: () => isOnline };
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 6. 비용 분석: AI Agent vs 상담원

&nbsp;

```
가정:
- 일 1,000건 상담
- 상담당 평균 5턴 대화
- 턴당 입력 500토큰, 출력 200토큰
- 하루 20% (200건)는 도구 호출 포함

일간 토큰:
- 입력: 1,000건 × 5턴 × 500토큰 = 2,500,000 토큰
- 출력: 1,000건 × 5턴 × 200토큰 = 1,000,000 토큰

월간 AI 비용 (30일):
┌──────────────────┬──────────────┬──────────────────┐
│ 모델              │ 월 비용 (USD) │ 월 비용 (KRW)     │
├──────────────────┼──────────────┼──────────────────┤
│ GPT-4o           │ $750         │ 약 107만원         │
│ Claude Sonnet    │ $975         │ 약 139만원         │
│ GPT-4o mini      │ $84          │ 약 12만원          │
│ Claude Haiku     │ $131         │ 약 19만원          │
└──────────────────┴──────────────┴──────────────────┘

+ 인프라 비용: 약 $100~200/월 (14~29만원)

vs 상담원 인건비:
- 상담원 1명: 약 300만원/월
- 일 1,000건 처리에 필요한 상담원: 약 5~7명
- 총 인건비: 1,500~2,100만원/월

비용 비교:
┌─────────────────┬──────────────┬──────────────────┐
│ 방식             │ 월 비용       │ 절감률            │
├─────────────────┼──────────────┼──────────────────┤
│ 상담원 6명       │ 1,800만원     │ 기준              │
│ AI Agent (GPT-4o)│ 약 120만원    │ 93% 절감          │
│ AI Agent (Haiku) │ 약 35만원     │ 98% 절감          │
│ 하이브리드       │ 약 500만원    │ 72% 절감          │
│ (AI 80% + 상담원1)│              │                   │
└─────────────────┴──────────────┴──────────────────┘

현실적 추천: 하이브리드 (AI 80% + 상담원 1~2명)
- AI가 처리 가능한 80%를 자동화
- 복잡한 20%는 상담원이 처리
- 상담원은 AI가 만든 요약을 보고 바로 대응
```

&nbsp;

&nbsp;

---

&nbsp;

## 7. 주의사항

&nbsp;

### 환각 (Hallucination) 방지

&nbsp;

```typescript
// BAD: Agent가 없는 정보를 만들어낼 수 있음
system: '고객 질문에 최선을 다해 답변하세요.'

// GOOD: 모르면 모른다고 하게 제한
system: `고객 상담 Agent입니다.

규칙:
1. 도구 호출 결과에 기반해서만 답변하세요.
2. 도구 호출 없이 주문 정보, 가격, 재고를 언급하지 마세요.
3. 확실하지 않으면 "확인 후 안내드리겠습니다"라고 하세요.
4. 절대로 정보를 추측하거나 만들어내지 마세요.`
```

&nbsp;

### 금전 관련 처리는 이중 확인

&nbsp;

```typescript
// 환불, 결제 등 금전 관련은 고객 확인을 받고 처리
const confirmationPrompt = `
[환불 확인]
- 주문번호: ${orderId}
- 환불 금액: ${amount}원
- 환급 수단: ${paymentMethod}

위 내용으로 환불을 진행할까요? (예/아니오)
`;
// Agent가 자의적으로 환불하지 않도록 프롬프트에 명시
```

&nbsp;

&nbsp;

---

&nbsp;

## 8. 정리

&nbsp;

| 항목 | 기존 챗봇 | AI Agent 챗봇 |
|------|----------|---------------|
| 질문 이해 | 키워드 매칭 | 자연어 이해 |
| 업무 처리 | 안내만 | 실제 처리 (환불, 조회 등) |
| 유지보수 | 시나리오 추가 = 코드 수정 | 프롬프트 + 도구 추가 |
| 멀티 언어 | 언어별 시나리오 필요 | LLM이 자동 감지 |
| 비용 | 서버비만 | LLM API + 서버비 |
| 정확도 | 규칙 내 100% | 95~99% (환각 주의) |

&nbsp;

고객 응대 Agent의 핵심은 **"안내"에서 "처리"로의 전환**이다.

"고객센터에 문의하세요" 대신 "지금 바로 처리해드렸습니다."

&nbsp;

다만, **금전 처리는 반드시 고객 확인을 받고**, **AI가 모르는 건 상담원에게 넘기는** 설계가 필수다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI Agent 실전 가이드 5편] 서버 Agent — 모니터링, 장애 감지, 자동 복구**

&nbsp;

에러 로그를 분석해서 자동으로 PR을 만들고, 서버 부하를 감지해서 오토스케일링하고, 보안 이상을 감지해서 자동 차단하는 서버 Agent를 다룬다. "새벽 3시에 장애가 났는데 사람 없이 대응하는" 시나리오를 코드와 함께 구현한다.

&nbsp;

&nbsp;

---

&nbsp;

AI에이전트, 고객응대, 챗봇, AI챗봇, WebSocket, Tool Use, RAG, 멀티턴대화, 키오스크, 콜센터, 상담자동화, 환불자동화
