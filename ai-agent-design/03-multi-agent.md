# [AI 에이전트 3편] AI 에이전트 설계 패턴 — 단일 에이전트 vs 멀티 에이전트 오케스트레이션

&nbsp;

AI 에이전트 하나가 코드를 짜고, 리뷰하고, 테스트하고, 배포한다.

&nbsp;

편하다. 하지만 문제가 있다.

**자기가 짠 코드를 자기가 리뷰하면 편향이 생긴다.**

&nbsp;

"이거 맞지?" → "네, 제가 짰으니까 맞습니다."

&nbsp;

이걸 해결하는 게 **멀티 에이전트 패턴**이다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 단일 에이전트의 한계

&nbsp;

## 컨텍스트 오염

&nbsp;

```
나: "이 파일 100개 분석해줘"
AI: (파일 읽기 × 100)
AI: (컨텍스트 가득 참)
나: "이제 수정해줘"
AI: (앞에서 읽은 내용 일부 잊어버림)
```

&nbsp;

하나의 세션에서 너무 많은 작업을 하면 컨텍스트 윈도우가 포화된다.

&nbsp;

## 역할 편향

&nbsp;

```
AI가 코드 작성 → AI가 코드 리뷰
"제가 방금 짠 코드니까 괜찮습니다"
→ race condition 놓침
→ 에러 처리 누락 놓침
```

&nbsp;

같은 세션에서 작성과 리뷰를 하면 자기 코드에 관대해진다.

&nbsp;

## 전문성 부족

&nbsp;

한 에이전트에게 "보안 + 성능 + UX + 접근성" 모두 확인하라고 하면

다 얕게 본다. 전문가 한 명이 깊게 보는 것보다 못하다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 멀티 에이전트 패턴

&nbsp;

## 패턴 1: 역할 분리 (Role-based)

&nbsp;

```
PM (사용자)
  ↓ "이 기능 만들어줘"
코더 (에이전트 A)
  ↓ 코드 작성 완료
리뷰어 (에이전트 B) — 새 세션, 코더의 맥락 모름
  ↓ "race condition 있어요"
코더 (에이전트 A)
  ↓ 수정
테스터 (에이전트 C)
  ↓ 테스트 작성 + 실행
완료
```

&nbsp;

**핵심: 리뷰어는 코더의 컨텍스트를 모른다.** 그래서 편향 없이 본다.

&nbsp;

```typescript
// 오케스트레이터
async function developFeature(requirement: string) {
  // 1. 코더: 코드 작성
  const code = await runAgent('coder', {
    prompt: `다음 요구사항을 구현해줘: ${requirement}`,
    tools: ['read', 'write', 'search', 'bash']
  });

  // 2. 리뷰어: 독립 세션에서 리뷰 (코더의 맥락 없음)
  const review = await runAgent('reviewer', {
    prompt: `다음 코드를 리뷰해줘. race condition, 에러 처리, 보안 관점으로.\n${code}`,
    tools: ['read', 'search']  // 읽기만 가능
  });

  // 3. 리뷰 결과에 문제가 있으면 코더에게 수정 요청
  if (review.hasIssues) {
    const fixedCode = await runAgent('coder', {
      prompt: `리뷰 피드백을 반영해서 수정해줘:\n${review.feedback}`,
      tools: ['read', 'write', 'search', 'bash']
    });
  }

  // 4. 테스터: 테스트 작성 + 실행
  await runAgent('tester', {
    prompt: `다음 코드에 대한 테스트를 작성하고 실행해줘`,
    tools: ['read', 'write', 'bash']
  });
}
```

&nbsp;

## 패턴 2: 전문가 패널 (Expert Panel)

&nbsp;

```
같은 코드를 3명의 전문가가 각자 리뷰:

보안 전문가: "SQL injection 가능성 있음"
성능 전문가: "N+1 쿼리 발생"
UX 전문가: "에러 메시지가 사용자 친화적이지 않음"

→ 3개의 독립된 피드백 취합
```

&nbsp;

```typescript
async function expertReview(code: string) {
  // 병렬로 3명의 전문가 실행
  const [security, performance, ux] = await Promise.all([
    runAgent('security-expert', {
      system: '당신은 시니어 보안 엔지니어입니다. OWASP Top 10 기준으로 리뷰하세요.',
      prompt: `보안 관점에서 리뷰:\n${code}`
    }),
    runAgent('performance-expert', {
      system: '당신은 성능 최적화 전문가입니다. DB 쿼리, 메모리, 응답 시간을 확인하세요.',
      prompt: `성능 관점에서 리뷰:\n${code}`
    }),
    runAgent('ux-expert', {
      system: '당신은 UX 엔지니어입니다. 에러 메시지, 응답 형식, 일관성을 확인하세요.',
      prompt: `사용자 경험 관점에서 리뷰:\n${code}`
    })
  ]);

  return { security, performance, ux };
}
```

&nbsp;

## 패턴 3: 계층적 오케스트레이션 (Hierarchical)

&nbsp;

```
오케스트레이터 (작업 분배 + 결과 취합)
  ├→ 프론트엔드 에이전트: React 컴포넌트 구현
  ├→ 백엔드 에이전트: API 엔드포인트 구현
  └→ DB 에이전트: 스키마 설계 + 마이그레이션
```

&nbsp;

```typescript
async function buildFeature(spec: string) {
  // 오케스트레이터가 작업 분할
  const plan = await runAgent('orchestrator', {
    system: '당신은 테크 리드입니다. 작업을 프론트/백엔드/DB로 나눠주세요.',
    prompt: `다음 기능을 구현해야 합니다:\n${spec}`
  });

  // 각 에이전트가 독립 작업 (worktree로 충돌 방지)
  const results = await Promise.all([
    runAgent('frontend', { prompt: plan.frontendTask, worktree: true }),
    runAgent('backend', { prompt: plan.backendTask, worktree: true }),
    runAgent('database', { prompt: plan.dbTask, worktree: true })
  ]);

  // 오케스트레이터가 결과 취합 + 통합 테스트
  await runAgent('orchestrator', {
    prompt: `3개 팀의 결과를 통합하고 테스트해줘:\n${JSON.stringify(results)}`
  });
}
```

&nbsp;

## 패턴 4: 토론/합의 (Debate)

&nbsp;

```
설계 결정이 필요할 때:

에이전트 A: "MongoDB가 좋겠습니다. 스키마 유연성이 필요합니다."
에이전트 B: "PostgreSQL이 낫습니다. 트랜잭션 안정성이 중요합니다."
에이전트 A: "하지만 스키마가 자주 바뀌는데..."
에이전트 B: "JSONB 컬럼으로 유연성도 확보할 수 있습니다."
심판: "PostgreSQL + JSONB로 결정합니다."
```

&nbsp;

```typescript
async function debate(question: string, rounds: number = 3) {
  let history = '';

  for (let i = 0; i < rounds; i++) {
    const proArg = await runAgent('advocate-a', {
      prompt: `${question}\n이전 논의:\n${history}\n찬성 입장에서 주장해줘.`
    });
    history += `\n찬성: ${proArg}`;

    const conArg = await runAgent('advocate-b', {
      prompt: `${question}\n이전 논의:\n${history}\n반대 입장에서 반박해줘.`
    });
    history += `\n반대: ${conArg}`;
  }

  // 심판이 최종 결정
  const decision = await runAgent('judge', {
    prompt: `다음 논의를 보고 최종 결정을 내려줘:\n${history}`
  });

  return decision;
}
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. 패턴 비교

&nbsp;

| 패턴 | 에이전트 수 | 적합한 경우 | 복잡도 |
|:---|:---:|:---|:---:|
| **역할 분리** | 2~4 | 코드 작성 + 리뷰 | 낮음 |
| **전문가 패널** | 3~5 | 코드 리뷰, 설계 검토 | 중간 |
| **계층적** | 3~7 | 대규모 기능 개발 | 높음 |
| **토론** | 2~3 | 설계 결정, 기술 선택 | 중간 |

&nbsp;

&nbsp;

---

&nbsp;

# 4. 실전 적용: 코드 리뷰 자동화

&nbsp;

가장 실용적인 멀티 에이전트 활용.

&nbsp;

```
코더: 코드 작성 (세션 A)
  ↓ 코드 완성
리뷰어: 코드 리뷰 (세션 B — 독립)
  ↓ 피드백
보안: 보안 검사 (세션 C — 독립)
  ↓ 취약점 보고
테스터: 테스트 (세션 D — 독립)
  ↓ 테스트 결과
→ 취합해서 PR에 코멘트
```

&nbsp;

실제로 이렇게 하려면 터미널 4개를 열 필요는 없다.

AI 코딩 도구에서:

```
1. 코드 작성 (기본 세션)
2. /clear
3. "방금 수정된 파일들 리뷰해줘" (새 컨텍스트 = 독립 리뷰)
```

&nbsp;

**/clear 한 번이면 "새로운 리뷰어"가 된다.** 가장 가성비 좋은 멀티 에이전트.

&nbsp;

&nbsp;

---

&nbsp;

# 5. 에이전트 간 통신

&nbsp;

```typescript
// 간단한 메시지 전달 구조
interface AgentMessage {
  from: string;      // 'coder' | 'reviewer' | 'tester'
  to: string;
  type: 'request' | 'response' | 'feedback';
  content: string;
  artifacts?: {      // 코드, 테스트 결과 등
    files?: string[];
    testResults?: any;
  };
}

// 오케스트레이터가 메시지를 라우팅
class Orchestrator {
  private agents = new Map<string, Agent>();

  async send(message: AgentMessage): Promise<AgentMessage> {
    const agent = this.agents.get(message.to);
    if (!agent) throw new Error(`Agent ${message.to} not found`);

    const response = await agent.process(message);
    return response;
  }

  async pipeline(steps: AgentMessage[]): Promise<AgentMessage[]> {
    const results: AgentMessage[] = [];
    for (const step of steps) {
      const result = await this.send(step);
      results.push(result);
      // 이전 결과를 다음 에이전트 입력에 포함
      if (steps.indexOf(step) < steps.length - 1) {
        steps[steps.indexOf(step) + 1].content += `\n이전 결과: ${result.content}`;
      }
    }
    return results;
  }
}
```

&nbsp;

&nbsp;

---

&nbsp;

# 6. 실패 처리

&nbsp;

한 에이전트가 실패하면?

&nbsp;

```typescript
async function safeAgentCall(agent: string, task: string, retries = 2) {
  for (let i = 0; i <= retries; i++) {
    try {
      const result = await runAgent(agent, { prompt: task });

      // 결과 검증
      if (!result || result.error) {
        throw new Error(result?.error || 'Empty result');
      }

      return result;
    } catch (e) {
      if (i === retries) {
        // 최종 실패 → 사람에게 에스컬레이션
        await notify('slack', `에이전트 ${agent} 실패: ${e.message}`);
        return { success: false, error: e.message, needsHuman: true };
      }
      // 재시도
      await sleep(2000);
    }
  }
}
```

&nbsp;

| 실패 유형 | 대응 |
|:---|:---|
| 에이전트 응답 없음 | 재시도 (2회) |
| 잘못된 결과 | 다른 에이전트에게 재할당 |
| 모든 재시도 실패 | 사람에게 알림 |
| 에이전트 간 충돌 | 오케스트레이터가 중재 |

&nbsp;

&nbsp;

---

&nbsp;

# 7. 현실적 조언

&nbsp;

```
어떤 규모에 어떤 패턴?

1인 개발:
  → 단일 에이전트 + /clear 후 리뷰
  → 이것만으로도 코드 품질 30% 향상

2~3인 팀:
  → 코더 + 리뷰어 2개면 충분
  → 사람 리뷰 전에 AI 리뷰로 기본 필터링

5인 이상 팀:
  → 역할별 에이전트 + 오케스트레이터
  → CI/CD에 자동 리뷰 파이프라인 통합

대규모 프로젝트:
  → 전문가 패널 + 계층적 오케스트레이션
  → 설계 결정에 토론 패턴
```

&nbsp;

**가장 흔한 실수:**

"에이전트를 많이 쓸수록 좋다" → 아니다.

에이전트 수가 늘면 통신 오버헤드, 결과 취합 비용, 실패 확률이 다 올라간다.

**필요한 만큼만.** 대부분은 "코더 + 리뷰어" 2개면 충분하다.

&nbsp;

&nbsp;

---

&nbsp;

# 8. 분리의 함정 — 나누면 안 되는 걸 나누면

&nbsp;

멀티 에이전트의 가장 큰 리스크는 **"나누면 안 되는 작업을 나눠버리는 것"**이다.

&nbsp;

### 누가 작업을 나누는가?

오케스트레이터(= LLM)가 사용자의 자연어를 분해한다.

```
사용자: "어제 매출이랑 고객 문의 분석해서 보고해줘"
                    │
                    ▼
            오케스트레이터 (LLM)
            "이 요청을 분해하면..."
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   "매출 분석"  "고객문의 분석"  "보고서 작성"
   (독립 가능)  (독립 가능)     (앞 2개 끝나야 가능)
```

&nbsp;

### 잘못 나누면 어떻게 되는가?

&nbsp;

```
사용자: "매출이 떨어진 원인 분석해서 대응 방안 보고해줘"
```

&nbsp;

**잘못된 분리 (병렬):**
```
Agent-1: 매출 데이터 조회      ─┐ 동시에
Agent-2: 대응 방안 작성        ─┘
→ Agent-2가 원인도 모르는데 대응 방안을 쓴다 → 엉터리
```

&nbsp;

**올바른 처리 (순차):**
```
1단계: 매출 데이터 조회 + 원인 분석  → 끝나면
2단계: 원인 기반으로 대응 방안 작성  → 그 다음
```

&nbsp;

### 판단 기준

&nbsp;

| 질문 | 병렬 가능? |
|:---|:---:|
| B가 A의 결과를 모르고도 실행할 수 있는가? | O → 병렬 |
| B가 A의 결과에 따라 달라지는가? | X → 순차 |
| A와 B가 완전히 다른 데이터를 다루는가? | O → 병렬 |
| A의 "판단"이 B의 입력이 되는가? | X → 순차 |

&nbsp;

### 현실적 결론

&nbsp;

대부분의 실무 요청은 **순차적 의존성**이 있다.

```
분석 → 그 결과로 판단 → 그 판단으로 실행 → 보고
```

병렬로 돌릴 수 있는 경우가 생각보다 많지 않다.

&nbsp;

**원칙:**
- 소규모 → **통합 1개로 시작**하는 게 맞다
- 병렬이 명확한 케이스가 생기면 그때 분리
- 잘못 분리하면 틀린 답이 나온다 — **안 나누는 게 나누는 것보다 안전하다**

&nbsp;

&nbsp;

---

&nbsp;

# 결론

&nbsp;

멀티 에이전트의 핵심은 **"자기 코드를 남이 본다"**는 것이다.

&nbsp;

사람을 더 뽑지 않아도 된다.

세션 하나만 더 열면 된다.

&nbsp;

시작은 단순하게:

1. 코드 작성
2. /clear
3. 리뷰

&nbsp;

이것만으로도 혼자 개발할 때의 사각지대를 줄일 수 있다.

복잡한 오케스트레이션은 **그 다음**이다.

&nbsp;

&nbsp;

---

AI에이전트, 멀티에이전트, 오케스트레이션, 코드리뷰, 역할분리, 전문가패널, AI코딩, 바이브코딩, 자동화, 에이전트설계, LLM, 프롬프트엔지니어링
