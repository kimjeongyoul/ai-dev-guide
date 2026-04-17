# [AI Agent 실전 가이드 1편] AI Agent란 — 챗봇과 뭐가 다른가

&nbsp;

"AI 도입했습니다."

&nbsp;

보통 이 말은 "챗봇 하나 붙였습니다"라는 뜻이다.

사용자가 질문하면 AI가 답변하고, 끝.

&nbsp;

그런데 최근 1~2년 사이에 **AI Agent**라는 단어가 급격히 퍼지고 있다.

Agent는 챗봇과 뭐가 다른가? 마케팅 용어인가, 실체가 있는 건가?

&nbsp;

결론부터 말하면 — **실체가 있다.** 그리고 그 차이는 "자율성"에 있다.

&nbsp;

&nbsp;

---

&nbsp;

## 1. 챗봇 vs AI Agent — 핵심 차이

&nbsp;

### 1-1. 챗봇: 질문 → 답변 (수동, 1회성)

&nbsp;

```
[사용자] "오늘 날씨 알려줘"
     ↓
[챗봇] "서울 맑음, 최고 24도입니다."
     ↓
  (끝. 다음 질문을 기다림)
```

&nbsp;

챗봇의 특징:

- 사용자가 먼저 말해야 동작한다
- 1번 질문 → 1번 답변으로 끝난다
- 외부 시스템을 건드리지 않는다
- "정보 제공"이 목적이다

&nbsp;

### 1-2. AI Agent: 트리거 → 판단 → 행동 → 결과 보고 (자율, 연속)

&nbsp;

```
[트리거] 서버 에러율 5% 초과 감지
     ↓
[판단] 에러 로그 분석 → 최근 배포가 원인으로 추정
     ↓
[행동 1] 이전 버전으로 롤백 실행
[행동 2] Slack에 장애 보고 전송
[행동 3] 관련 개발자에게 멘션
     ↓
[결과 보고] "롤백 완료. 에러율 0.3%로 정상화. 원인: 결제 API 타임아웃"
```

&nbsp;

Agent의 특징:

- **트리거**가 동작을 시작한다 (사용자 입력이 아닐 수도 있다)
- **여러 단계**를 스스로 판단하고 실행한다
- **외부 시스템을 실제로 조작**한다 (DB, API, 파일, 배포)
- "업무 수행"이 목적이다

&nbsp;

### 1-3. 한눈에 비교

&nbsp;

| 항목 | 챗봇 | AI Agent |
|------|------|----------|
| 시작 조건 | 사용자 질문 | 트리거 (이벤트, 스케줄, 명령) |
| 응답 방식 | 텍스트 답변 | 판단 + 행동 + 보고 |
| 외부 시스템 | 읽기만 (있다면) | 읽기 + 쓰기 + 실행 |
| 대화 횟수 | 1회성 | 연속 (멀티스텝) |
| 자율성 | 낮음 | 높음 |
| 실패 시 | "죄송합니다, 다시 질문해주세요" | 재시도, 대안 경로, 에스컬레이션 |

&nbsp;

&nbsp;

---

&nbsp;

## 2. 자율성 레벨 — L1부터 L5까지

&nbsp;

자율주행에 레벨이 있듯, AI Agent에도 자율성 레벨이 있다.

&nbsp;

```
L1 ──── L2 ──── L3 ──── L4 ──── L5
수동     제안     실행     자율     완전
실행     승인     감독     예외     자율
```

&nbsp;

### L1: 사람이 명령, AI가 실행

&nbsp;

```typescript
// 개발자가 "이 함수 리팩토링해줘" → AI가 코드 수정
// GitHub Copilot, ChatGPT 코드 생성이 여기에 해당

// 사람: "getUserById를 에러 핸들링 추가해서 리팩토링해줘"
// AI가 생성:
async function getUserById(id: string): Promise<User> {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    logger.error('Failed to fetch user', { id, error });
    throw error;
  }
}
// 사람이 코드 확인 → 직접 커밋
```

&nbsp;

**핵심:** AI는 도구일 뿐, 모든 판단과 실행은 사람이 한다.

&nbsp;

### L2: AI가 제안, 사람이 승인

&nbsp;

```typescript
// Claude Code, Cursor가 여기에 해당
// AI: "이 PR에서 3개 파일을 수정해야 합니다. 변경 내역:"
// AI: "1. src/api/user.ts - 에러 핸들링 추가"
// AI: "2. src/types/user.ts - 타입 수정"  
// AI: "3. src/tests/user.test.ts - 테스트 추가"
// AI: "실행할까요? (Y/n)"

// 사람: "Y"
// → AI가 3개 파일 수정 + 커밋 + PR 생성
```

&nbsp;

**핵심:** AI가 계획을 세우고 제안하지만, 사람이 "Go"를 눌러야 실행된다.

&nbsp;

### L3: AI가 실행, 사람이 감독

&nbsp;

```yaml
# CI/CD 파이프라인에 AI Agent가 포함된 경우
# PR이 올라오면 자동으로:
# 1. 코드 리뷰 (AI)
# 2. 테스트 실행 (CI)
# 3. 스테이징 배포 (자동)
# 4. 프로덕션 배포는 사람이 승인

pipeline:
  on: pull_request
  steps:
    - ai-code-review     # AI가 자동 리뷰
    - run-tests           # 테스트 자동 실행
    - deploy-staging      # 스테이징 자동 배포
    - human-approval      # 사람이 확인
    - deploy-production   # 승인 후 프로덕션 배포
```

&nbsp;

**핵심:** 대부분의 단계를 AI가 알아서 하고, 사람은 결과만 확인한다. 중요한 지점에서만 개입.

&nbsp;

### L4: AI가 자율 실행, 예외만 보고

&nbsp;

```python
# 모니터링 Agent 예시
# 24시간 서버를 감시하고, 이상 감지 시 자동 대응

class MonitoringAgent:
    def on_high_cpu(self, server, cpu_percent):
        if cpu_percent > 90:
            self.scale_up(server)           # 자동 스케일업
            self.notify("스케일업 완료")      # Slack 보고만
        
    def on_error_spike(self, service, error_rate):
        if error_rate > 5:
            self.rollback(service)           # 자동 롤백
            self.notify("롤백 실행")
        if error_rate > 20:
            self.page_oncall()               # 여기서만 사람 호출
```

&nbsp;

**핵심:** AI가 대부분 혼자 처리한다. 사람은 보고만 받고, 정말 심각할 때만 개입.

&nbsp;

### L5: 완전 자율

&nbsp;

아직 현실에 없다.

"비즈니스 목표를 주면 AI가 알아서 서비스를 만들고 운영하는" 수준.

현재 기술로는 불가능하고, 가능해지더라도 윤리적/법적 문제가 먼저 해결되어야 한다.

&nbsp;

### 현실적인 현재 위치

&nbsp;

```
2024년 대부분의 기업:  L1 ~ L2
2025년 선도 기업:      L2 ~ L3
가까운 미래 목표:       L3 ~ L4
```

&nbsp;

L2에서 L3으로 가는 것만으로도 엄청난 생산성 차이가 난다.

&nbsp;

&nbsp;

---

&nbsp;

## 3. AI Agent의 4가지 유형

&nbsp;

Agent를 도입한다고 할 때, 보통 4가지 유형 중 하나다.

&nbsp;

### 3-1. 사무직 Agent (Office Agent)

&nbsp;

```
[이메일 수신] → [분류: 계약서] → [핵심 조건 추출] → [CRM에 기록] → [담당자에게 알림]
```

&nbsp;

**하는 일:**
- 이메일 자동 분류 + 요약
- 회의록 작성 + 액션 아이템 추출
- 보고서 초안 작성
- 사내 규정/정책 Q&A

&nbsp;

**적합한 조직:** 사무 직원이 많고, 반복적인 문서 작업이 많은 기업

&nbsp;

### 3-2. 고객 응대 Agent (Customer Agent)

&nbsp;

```
[고객] "주문한 지 3일인데 아직 안 왔어요"
   ↓
[Agent] 주문DB 조회 → 배송 추적 → 지연 원인 확인
   ↓
[Agent] "물류센터에서 금일 출발 예정입니다. 지연 사과로 쿠폰 발급해드릴까요?"
   ↓
[고객] "네"
   ↓
[Agent] 쿠폰 발급 API 호출 → 완료 메시지 전송
```

&nbsp;

**하는 일:**
- 자연어로 고객 문의 처리
- 주문 조회, 환불, 예약 변경 등 실제 업무 수행
- 상담원 연결이 필요한 경우 자동 에스컬레이션

&nbsp;

**적합한 조직:** 고객 문의가 많고, 반복 질문 비율이 높은 기업

&nbsp;

### 3-3. 서버 Agent (Server Agent)

&nbsp;

```
[Sentry 알림] "TypeError: Cannot read property 'id' of undefined"
   ↓
[Agent] 스택트레이스 분석 → 원인: user 객체 null 체크 누락
   ↓
[Agent] 코드 수정 → 테스트 작성 → PR 생성
   ↓
[Agent] Slack: "버그 수정 PR #342 생성했습니다. 리뷰해주세요."
```

&nbsp;

**하는 일:**
- 에러 로그 분석 + 자동 수정
- 서버 모니터링 + 오토스케일링
- 보안 이상 감지 + 자동 차단
- 배포 모니터링 + 롤백

&nbsp;

**적합한 조직:** 서비스 안정성이 중요하고, 야간/주말 운영이 필요한 기업

&nbsp;

### 3-4. 개발 보조 Agent (Dev Agent)

&nbsp;

```
[Jira 티켓] "사용자 프로필 페이지에 최근 주문 내역 추가"
   ↓
[Agent] 기존 코드 분석 → 컴포넌트 구조 파악 → 코드 작성
   ↓
[Agent] 테스트 작성 → 린트 통과 → PR 생성
   ↓
[Agent] "PR #155 생성. 변경 파일 3개, 테스트 커버리지 92%"
```

&nbsp;

**하는 일:**
- 코드 작성, 리팩토링, 테스트
- PR 리뷰, 코드 분석
- 문서 자동 생성
- 기술 부채 탐지 + 수정 제안

&nbsp;

**적합한 조직:** 개발팀이 있고, 반복적인 코딩 작업이 많은 기업

&nbsp;

### 4가지 유형 요약

&nbsp;

| 유형 | 자율성 레벨 | 주요 도구 | 위험도 |
|------|------------|-----------|--------|
| 사무직 Agent | L2~L3 | MS365, Google Workspace, CRM | 낮음 |
| 고객 응대 Agent | L2~L3 | 채팅, CRM, 주문DB, 결제API | 중간 |
| 서버 Agent | L3~L4 | 모니터링, CI/CD, 인프라 | 높음 |
| 개발 보조 Agent | L2~L3 | IDE, Git, CI/CD | 중간 |

&nbsp;

&nbsp;

---

&nbsp;

## 4. "우리 회사에 Agent가 필요한가?" 판단 기준

&nbsp;

Agent 도입을 검토할 때, 아래 5가지 질문에 답해보자.

&nbsp;

### 질문 1: 반복 업무가 있는가?

&nbsp;

```
□ 매일 같은 형식의 이메일을 처리한다
□ 같은 유형의 고객 질문이 반복된다
□ 서버 에러가 나면 항상 같은 절차로 대응한다
□ 보고서를 매주 같은 형식으로 작성한다
□ 코드 리뷰에서 같은 피드백을 반복한다
```

&nbsp;

**3개 이상 체크 → Agent가 도움이 된다.**

&nbsp;

### 질문 2: 얼마나 자주 발생하는가?

&nbsp;

```
일 10건 미만     → 사람이 하는 게 낫다 (도입 비용 > 절감 효과)
일 10~100건      → Agent 도입 검토 가치 있음
일 100건 이상    → Agent가 거의 필수
```

&nbsp;

### 질문 3: 실수의 비용이 큰가?

&nbsp;

```
이메일 오타        → 낮음 → Agent 자율 실행 가능
고객 환불 처리     → 중간 → Agent 실행 + 사람 감독
서버 프로덕션 배포 → 높음 → Agent 제안 + 사람 승인
금융 거래 처리     → 매우 높음 → Agent 보조 + 사람 실행
```

&nbsp;

### 질문 4: 데이터가 정리되어 있는가?

&nbsp;

```
정리됨: DB에 구조화, API로 접근 가능 → 바로 도입 가능
반정리: 엑셀, 문서에 산재 → RAG 구축 먼저 필요 (2~4주)
미정리: 담당자 머릿속에만 존재 → 정리부터 시작 (수개월)
```

&nbsp;

### 질문 5: 보안 요구사항은?

&nbsp;

```
공개 데이터만 다룸       → 클라우드 API 사용 가능
사내 데이터 포함         → 데이터 마스킹 필요
개인정보/금융 데이터     → 온프레미스 LLM 또는 엄격한 접근 제어
규제 대상 (금융/의료)    → 법무 검토 + 감사 로그 필수
```

&nbsp;

### 판단 매트릭스

&nbsp;

```
                    반복 빈도 높음
                        │
        ┌───────────────┼───────────────┐
        │               │               │
  실수 비용   서버 Agent (L4)   고객 Agent (L3)
   높음       자동화 + 감독       자동화 + Fallback
        │               │               │
        ├───────────────┼───────────────┤
        │               │               │
  실수 비용   개발 Agent (L2)   사무직 Agent (L3)
   낮음       보조 + 승인         자동화
        │               │               │
        └───────────────┼───────────────┘
                        │
                    반복 빈도 낮음
```

&nbsp;

&nbsp;

---

&nbsp;

## 5. 챗봇에서 Agent로의 진화 경로

&nbsp;

이미 챗봇을 운영 중이라면, 단계적으로 Agent로 진화할 수 있다.

&nbsp;

```
[1단계] 챗봇 (현재)
├── 규칙 기반 응답
├── FAQ 매칭
└── "담당자 연결해드리겠습니다"

[2단계] 스마트 챗봇
├── LLM 기반 자연어 이해
├── RAG로 사내 문서 검색
└── 여전히 "답변만" 한다

[3단계] Agent (목표)
├── LLM + RAG + Tool Use
├── 외부 시스템 조작 (주문 조회, 환불 등)
├── 멀티스텝 대화
└── 자동 에스컬레이션
```

&nbsp;

각 단계의 차이를 코드로 보면:

&nbsp;

```typescript
// 1단계: 규칙 기반 챗봇
function handleMessage(message: string): string {
  if (message.includes('영업시간')) return '평일 09:00~18:00입니다.';
  if (message.includes('환불')) return '고객센터(1588-0000)로 연락주세요.';
  return '죄송합니다. 담당자를 연결해드리겠습니다.';
}

// 2단계: LLM + RAG (답변만)
async function handleMessage(message: string): Promise<string> {
  const context = await vectorDB.search(message);  // 관련 문서 검색
  const response = await llm.chat({
    system: '고객 상담원입니다. 아래 문서를 참고해서 답변하세요.',
    context,
    message,
  });
  return response.text;  // 여전히 텍스트만 반환
}

// 3단계: AI Agent (판단 + 행동)
async function handleMessage(message: string): Promise<string> {
  const context = await vectorDB.search(message);
  const response = await llm.chat({
    system: '고객 상담 Agent입니다. 도구를 사용해서 실제로 처리하세요.',
    context,
    message,
    tools: [
      { name: 'lookupOrder', fn: orderAPI.lookup },
      { name: 'processRefund', fn: paymentAPI.refund },
      { name: 'issueCoupon', fn: couponAPI.issue },
      { name: 'escalateToHuman', fn: supportAPI.escalate },
    ],
  });
  // LLM이 도구를 직접 호출하고, 결과를 조합해서 응답
  return response.text;
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 6. 정리

&nbsp;

| 항목 | 설명 |
|------|------|
| 챗봇 | 질문 → 답변. 수동적, 1회성 |
| AI Agent | 트리거 → 판단 → 행동 → 보고. 자율적, 연속적 |
| 자율성 레벨 | L1(수동) ~ L5(완전 자율). 현실은 L2~L3 |
| 4가지 유형 | 사무직, 고객응대, 서버, 개발보조 |
| 도입 판단 | 반복 빈도 + 실수 비용 + 데이터 상태 + 보안 요구 |

&nbsp;

Agent는 "더 똑똑한 챗봇"이 아니다.

**"일을 대신 해주는 디지털 직원"**에 가깝다.

&nbsp;

다만, 모든 업무에 Agent가 필요한 건 아니다.

"사람이 하면 5분인데 Agent 만드는 데 2주" 걸리면 의미가 없다.

**반복 빈도가 높고, 패턴이 명확한 업무**부터 시작하는 게 정답이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI Agent 실전 가이드 2편] 아키텍처 기초 — LLM + RAG + Tool Use 조합**

&nbsp;

Agent를 만들려면 어떤 기술을 조합해야 하는가? LLM API 호출 구조, RAG로 사내 문서를 AI에게 알려주는 방법, Tool Use로 외부 시스템을 연동하는 방법을 다룬다. LangChain, LlamaIndex, CrewAI 프레임워크도 비교한다.

&nbsp;

&nbsp;

---

&nbsp;

AI에이전트, AI Agent, 챗봇, 자율성레벨, LLM, Tool Use, RAG, 사무자동화, 고객응대, 서버모니터링, 개발보조, AI도입
