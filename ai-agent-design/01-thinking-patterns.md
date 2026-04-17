# [AI 에이전트 1편] AI 에이전트의 사고 방식 — ReAct, CoT, Tool Use 패턴 비교

&nbsp;

AI에게 "이 버그 고쳐줘"라고 하면, AI는 어떻게 생각할까?

&nbsp;

그냥 답을 내놓는 게 아니다.

내부적으로 **추론하고, 도구를 쓰고, 결과를 관찰하고, 다시 생각한다.**

&nbsp;

이 "생각하는 방식"에 세 가지 패턴이 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. Chain-of-Thought (CoT) — 단계별 추론

&nbsp;

"생각을 말하면서 풀어라."

&nbsp;

```
질문: "서버 응답이 5초 걸리는 이유는?"

CoT:
1. 먼저 API 엔드포인트를 확인한다
2. DB 쿼리가 느린지 본다
3. N+1 문제가 있는지 확인한다
4. 외부 API 호출이 있는지 본다
→ 결론: getOrders에서 N+1 쿼리가 100번 실행되고 있다
```

&nbsp;

**특징:**
- AI가 중간 과정을 보여준다
- 외부 도구 없이 추론만으로
- "생각해보자..." 프롬프트만 추가하면 정확도가 올라간다

&nbsp;

```typescript
// CoT 프롬프트
const prompt = `
다음 문제를 단계별로 생각해서 풀어줘.
각 단계마다 왜 그렇게 판단했는지 이유를 설명해.

문제: ${question}
`;
```

&nbsp;

**적합한 경우:** 논리적 추론, 수학, 코드 분석 (도구 없이 생각만으로 풀 수 있는 것)

&nbsp;

&nbsp;

---

&nbsp;

# 2. ReAct — 추론 + 행동 반복

&nbsp;

"생각하고, 행동하고, 관찰하고, 다시 생각하라."

&nbsp;

```
질문: "users 테이블에 데이터가 몇 건이야?"

Thought 1: DB를 조회해야 한다
Action 1: sql_query("SELECT COUNT(*) FROM users")
Observation 1: 42,531

Thought 2: 결과를 확인했다
Answer: users 테이블에 42,531건이 있습니다
```

&nbsp;

CoT와의 차이: **중간에 도구를 사용한다.**

&nbsp;

```typescript
// ReAct 루프
async function reactLoop(question: string, tools: Tool[]) {
  let context = question;
  
  for (let i = 0; i < 10; i++) {
    // 1. 생각
    const thought = await ai.think(context);
    
    // 2. 행동 결정
    if (thought.action === 'FINISH') {
      return thought.answer;
    }
    
    // 3. 도구 실행
    const result = await executeTool(thought.action, thought.input);
    
    // 4. 관찰 결과를 컨텍스트에 추가
    context += `\nAction: ${thought.action}\nObservation: ${result}`;
  }
}
```

&nbsp;

**적합한 경우:** 외부 데이터가 필요한 작업, 멀티스텝 작업, 검색 기반 질의응답

&nbsp;

&nbsp;

---

&nbsp;

# 3. Tool Use — 도구 목록을 주고 AI가 선택

&nbsp;

"이 도구들이 있다. 필요한 걸 골라 써라."

&nbsp;

```json
{
  "tools": [
    {
      "name": "read_file",
      "description": "파일 내용을 읽는다",
      "parameters": { "path": { "type": "string" } }
    },
    {
      "name": "search_code",
      "description": "코드베이스에서 키워드를 검색한다",
      "parameters": { "query": { "type": "string" } }
    },
    {
      "name": "run_test",
      "description": "테스트를 실행한다",
      "parameters": { "command": { "type": "string" } }
    }
  ]
}
```

&nbsp;

```
사용자: "login 함수에 버그가 있는 것 같아"

AI 내부:
→ search_code("login") 호출
→ read_file("src/auth/login.ts") 호출
→ 코드 분석
→ 수정 제안
→ run_test("npm test") 호출
→ 결과 확인
```

&nbsp;

ReAct와의 차이: **프롬프트가 아니라 API 레벨에서 도구를 정의한다.**

AI가 JSON Schema 기반으로 정확한 파라미터로 도구를 호출한다.

&nbsp;

```typescript
// Anthropic Claude Tool Use API
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  messages: [{ role: 'user', content: question }],
  tools: [
    {
      name: 'search_code',
      description: '코드베이스에서 키워드를 검색한다',
      input_schema: {
        type: 'object',
        properties: {
          query: { type: 'string', description: '검색할 키워드' }
        },
        required: ['query']
      }
    }
  ]
});

// AI가 도구 호출을 결정하면
if (response.stop_reason === 'tool_use') {
  const toolCall = response.content.find(c => c.type === 'tool_use');
  const result = await executeTool(toolCall.name, toolCall.input);
  // 결과를 다시 AI에 전달 → 다음 행동 결정
}
```

&nbsp;

**적합한 경우:** AI 코딩 도구, 복잡한 에이전트, 정확한 파라미터가 필요한 도구 호출

&nbsp;

&nbsp;

---

&nbsp;

# 4. 비교

&nbsp;

| | CoT | ReAct | Tool Use |
|:---|:---|:---|:---|
| **동작** | 생각만 | 생각 + 행동 | 도구 선택 + 호출 |
| **외부 도구** | 없음 | 프롬프트로 유도 | API로 정의 |
| **정확도** | 추론 의존 | 관찰로 보정 | 스키마로 강제 |
| **구현** | 프롬프트만 | 프롬프트 + 루프 | API + 스키마 |
| **제어** | 낮음 | 중간 | 높음 |
| **속도** | 빠름 (1회) | 느림 (여러 턴) | 중간 |

&nbsp;

&nbsp;

---

&nbsp;

# 5. 실전에서는 조합한다

&nbsp;

```
사용자: "이 프로젝트의 성능 병목을 찾아서 수정해줘"

AI 내부:
1. CoT: "먼저 느린 API를 찾아야 한다. 로그를 보자."
2. Tool Use: search_code("performance") → read_file("src/api/orders.ts")
3. CoT: "N+1 쿼리가 보인다. fetch join으로 바꿔야 한다."
4. Tool Use: edit_file("src/api/orders.ts", ...)
5. Tool Use: run_test("npm test")
6. CoT: "테스트 통과. 수정 완료."
```

&nbsp;

**CoT로 방향을 정하고, Tool Use로 실행하고, 결과를 관찰해서 다시 CoT.**

이게 현대 AI 코딩 도구가 동작하는 방식이다.

&nbsp;

&nbsp;

---

&nbsp;

# 6. 내 프로젝트에 적용하려면

&nbsp;

| 상황 | 패턴 |
|:---|:---|
| AI에게 코드 분석만 시키기 | CoT |
| AI가 DB 조회하고 코드 수정하게 | Tool Use |
| AI가 여러 단계를 스스로 판단하게 | ReAct + Tool Use |
| 챗봇에 외부 API 연동 | Tool Use |
| 복잡한 워크플로우 자동화 | ReAct + Tool Use + MCP |

&nbsp;

다음 편에서는 **MCP로 실전 자동화 파이프라인을 만드는 방법**을 다룬다.

&nbsp;

&nbsp;

---

AI에이전트, ReAct, Chain-of-Thought, Tool Use, 프롬프트엔지니어링, Claude, MCP, 자동화, AI코딩, 바이브코딩, LLM, 함수호출, 에이전트설계
