# [AI 하네스 2편] 프로젝트 규칙 파일 작성법 — AI가 읽는 사내 코딩 가이드

&nbsp;

AI 에이전트에게 가장 먼저 알려줘야 하는 것은 **"우리 프로젝트의 규칙"**이다.

&nbsp;

어떻게 빌드하는지, 어떤 스타일로 코드를 쓰는지, 뭘 하면 안 되는지.

&nbsp;

이걸 담는 것이 **프로젝트 규칙 파일**이다.

&nbsp;

이 파일 하나가 AI의 결과물 품질을 결정한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 규칙 파일에 뭘 넣어야 하나

&nbsp;

## 1-1. 빌드 / 실행 명령

&nbsp;

AI가 가장 먼저 알아야 하는 것. "이 프로젝트를 어떻게 돌리는가."

&nbsp;

```markdown
## 빌드 & 실행

### 개발
- `npm run dev` — 개발 서버 (http://localhost:3000)
- `npm run dev:worker` — 백그라운드 워커

### 빌드
- `npm run build:staging` — 스테이징 빌드
- `npm run build:prod` — 프로덕션 빌드
- 수동으로 `npm run build` 후 직접 배포하지 마라. 반드시 환경별 스크립트 사용.

### 테스트
- `npm run test` — 전체 테스트
- `npm run test:unit` — 유닛 테스트만
- `npm run test:e2e` — E2E 테스트 (Playwright)
```

&nbsp;

> **핵심: "하지 마라"를 명시하라.** AI는 일반적인 방법을 시도하므로, 금지 사항을 알려줘야 한다.

&nbsp;

## 1-2. 코딩 컨벤션

&nbsp;

```markdown
## 코딩 컨벤션

### 네이밍
- 변수/함수: camelCase (`getUserData`, `isValid`)
- 컴포넌트/타입/인터페이스: PascalCase (`UserProfile`, `ApiResponse`)
- 상수: UPPER_SNAKE_CASE (`MAX_RETRY_COUNT`, `API_BASE_URL`)
- 파일명: kebab-case (`user-profile.tsx`, `api-client.ts`)
- 테스트 파일: `*.test.ts` 또는 `*.spec.ts`

### 함수 규칙
- 한 함수는 30줄 이내
- 파라미터는 4개 이내 (초과 시 객체로 묶기)
- `any` 타입 사용 금지
- 매직 넘버 금지 (상수로 추출)

### Import 순서
1. React / Next.js 내장
2. 외부 라이브러리
3. 내부 패키지 (@packages/*)
4. 로컬 모듈 (./*)
5. 타입 (type imports)
```

&nbsp;

## 1-3. 아키텍처 규칙

&nbsp;

```markdown
## 아키텍처

### 레이어 분리
- `app/` — 라우트 핸들러 (비즈니스 로직 금지)
- `services/` — 비즈니스 로직
- `repositories/` — 데이터 접근
- `types/` — 타입 정의
- 라우트 핸들러에서 DB 직접 접근 금지. 반드시 서비스 레이어를 거친다.

### 금지 패턴
- 순환 참조 (A → B → A) 금지
- 하위 레이어에서 상위 레이어 import 금지 (repository가 service를 import하면 안 됨)
- 전역 상태 남용 금지 (Redux에 UI 상태 넣지 마라)
```

&nbsp;

## 1-4. 커밋 컨벤션

&nbsp;

```markdown
## 커밋 컨벤션

Conventional Commits 형식:
- `feat(scope): 설명` — 새 기능
- `fix(scope): 설명` — 버그 수정
- `refactor(scope): 설명` — 리팩토링
- `chore: 설명` — 빌드/설정/잡일
- `docs: 설명` — 문서

### 규칙
- 영문 소문자로 시작, 마침표 없음
- 명령형 (add, fix, update, remove)
- 50자 이내
- Jira 티켓이 있으면 Footer에 `Refs: PROJ-123`
```

&nbsp;

## 1-5. 배포 규칙

&nbsp;

```markdown
## 배포

### 프로세스
1. feature 브랜치에서 개발
2. PR 생성 → 리뷰 → 머지
3. staging 자동 배포 (CI/CD)
4. QA 확인 후 production 배포

### 금지 사항
- main 브랜치에 직접 push 금지
- force push 금지
- 수동 빌드 후 서버 직접 복사 금지
```

&nbsp;

## 1-6. 보안 규칙

&nbsp;

```markdown
## 보안

### 필수
- 환경 변수는 `.env` 파일 사용 (코드에 하드코딩 금지)
- API 키, 비밀번호는 절대 코드에 포함하지 마라
- SQL raw query 금지, ORM 사용
- 사용자 입력은 반드시 검증 (zod, joi 등)

### 금지
- `console.log`에 사용자 정보/토큰 출력 금지
- `eval()` 사용 금지
- `dangerouslySetInnerHTML` 사용 시 반드시 DOMPurify 적용
```

&nbsp;

&nbsp;

---

&nbsp;

# 2. 업종별 규칙 파일 실전 예시

&nbsp;

## 2-1. 이커머스

&nbsp;

```markdown
## 이커머스 필수 규칙

### 결제
- 가격 계산은 반드시 정수(원 단위)로 처리. 부동소수점 금지.
- 결제 관련 코드(`payment/`, `checkout/`) 수정 시 반드시 시니어 리뷰.
- 할인 계산 후 반드시 최종 금액 >= 0 검증.

### 재고
- 재고 차감은 트랜잭션 내에서 처리.
- 동시성 이슈 방지를 위해 비관적 락 사용.

### 주문
- 주문 상태 변경은 상태 머신 패턴 사용.
- 허용되지 않는 상태 전이 시 에러 throw.
```

&nbsp;

```typescript
// 잘못된 예: 부동소수점 가격 계산
const total = price * 0.9;  // 10% 할인 → 부동소수점 오류 가능

// 올바른 예: 정수 계산
const discountAmount = Math.floor(price * 10 / 100);  // 원 단위 절삭
const total = price - discountAmount;
```

&nbsp;

## 2-2. 핀테크

&nbsp;

```markdown
## 핀테크 필수 규칙

### 감사 로그
- 모든 금융 API 호출은 감사 로그 기록 필수.
- 로그에 포함: 호출자, 타임스탬프, 요청/응답 (민감정보 마스킹).

### 개인정보
- 카드번호: 앞 6자리, 뒤 4자리만 표시 (`123456******7890`)
- 전화번호: 중간 4자리 마스킹 (`010-****-5678`)
- 주민번호: 뒷자리 전체 마스킹 (`880101-*******`)

### 금액 처리
- BigNumber 또는 Decimal 라이브러리 사용 필수.
- 통화 코드(KRW, USD) 항상 명시.
- 환율 계산 시 소수점 이하 처리 규칙 문서화.
```

&nbsp;

```typescript
// 감사 로그 래퍼 예시
async function withAuditLog<T>(
  action: string,
  fn: () => Promise<T>
): Promise<T> {
  const startTime = Date.now();
  try {
    const result = await fn();
    await auditLog.record({
      action,
      status: 'success',
      duration: Date.now() - startTime,
    });
    return result;
  } catch (error) {
    await auditLog.record({
      action,
      status: 'failure',
      error: error.message,
      duration: Date.now() - startTime,
    });
    throw error;
  }
}
```

&nbsp;

## 2-3. SaaS (멀티테넌트)

&nbsp;

```markdown
## SaaS 필수 규칙

### 테넌트 격리
- 모든 DB 쿼리에 tenantId 조건 필수.
- 테넌트 간 데이터 접근 절대 불가.
- API 응답에 다른 테넌트 데이터 포함 여부 반드시 검증.

### API 버전 관리
- 엔드포인트: `/api/v1/`, `/api/v2/`
- Breaking change 시 반드시 새 버전 생성.
- 이전 버전 최소 6개월 유지.

### Rate Limiting
- 테넌트별 Rate Limit 적용 (무료: 100req/min, 유료: 1000req/min).
- Rate Limit 초과 시 429 응답 + Retry-After 헤더.
```

&nbsp;

```typescript
// 잘못된 예: tenantId 누락
const users = await db.query('SELECT * FROM users WHERE role = $1', [role]);

// 올바른 예: tenantId 필수
const users = await db.query(
  'SELECT * FROM users WHERE tenant_id = $1 AND role = $2',
  [tenantId, role]
);
```

&nbsp;

## 2-4. 게임

&nbsp;

```markdown
## 게임 필수 규칙

### 에셋
- 에셋 경로: `assets/{type}/{category}/{name}` 형식.
- 이미지: WebP 형식, 최대 512KB.
- 사운드: OGG 형식, 최대 1MB.

### 성능
- 렌더링 루프에서 메모리 할당 금지 (오브젝트 풀 사용).
- 프레임당 GC 발생 시 프로파일링 필수.
- API 호출은 로딩 화면에서만 (인게임 중 네트워크 요청 최소화).

### 밸런스
- 확률 테이블은 코드가 아닌 데이터 시트에서 관리.
- 확률 변경 시 반드시 QA 승인.
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. 규칙 파일의 계층 구조

&nbsp;

규칙은 세 단계로 나뉜다.

&nbsp;

```
┌─────────────────────────────────────┐
│        조직 레벨 (전사 공통)           │
│  보안 정책, 커밋 컨벤션, 코딩 스타일     │
├─────────────────────────────────────┤
│        프로젝트 레벨 (이 레포)          │
│  아키텍처, 빌드 명령, 배포 규칙         │
├─────────────────────────────────────┤
│        개인 레벨 (개인 선호)           │
│  에디터 설정, 응답 스타일              │
└─────────────────────────────────────┘
```

&nbsp;

**우선순위: 조직 > 프로젝트 > 개인**

&nbsp;

```bash
# 계층 구조 예시
~/.config/ai-agent/rules.md          # 조직 레벨 (전사 공통)
./project-rules.md                    # 프로젝트 레벨
~/.personal/ai-preferences.md        # 개인 레벨
```

&nbsp;

```markdown
# 조직 레벨 (전사 공통)
- TypeScript strict mode 필수
- ESLint + Prettier 필수
- Conventional Commits 필수
- 보안: 하드코딩 금지, ORM 필수

# 프로젝트 레벨 (이 레포만)
- Next.js App Router 사용
- 상태관리: Redux Toolkit
- 스타일: TailwindCSS (CSS-in-JS 금지)

# 개인 레벨
- 한국어 응답 선호
- 코드 설명은 주석이 아닌 대화로
```

&nbsp;

개인 선호가 "코드에 주석 달지 마"인데 조직 규칙이 "공개 API에는 JSDoc 필수"라면?

**조직 규칙이 이긴다.**

&nbsp;

&nbsp;

---

&nbsp;

# 4. 잘 쓴 규칙 vs 못 쓴 규칙

&nbsp;

## 못 쓴 규칙

&nbsp;

```markdown
# 프로젝트 규칙
- 코드를 깨끗하게 작성하세요
- 성능을 고려하세요
- 보안에 신경 쓰세요
- 테스트를 작성하세요
- 좋은 커밋 메시지를 쓰세요
```

&nbsp;

AI가 읽으면: **"알겠습니다"** (그리고 아무것도 달라지지 않는다)

&nbsp;

"깨끗하게"가 뭔지, "성능을 고려"가 뭔지 AI는 모른다.

&nbsp;

## 잘 쓴 규칙

&nbsp;

```markdown
# 프로젝트 규칙

## 코드 품질
- 함수는 30줄 이내. 초과 시 분리.
- 파라미터 4개 이내. 초과 시 객체로 묶기.
- `any` 타입 사용 금지. 최소 `unknown` 사용.
- 중첩 if문 3단계 이상 금지. Early return 패턴 사용.

## 성능
- 리스트 렌더링 시 반드시 `key` prop 사용.
- 1000개 이상 리스트는 가상 스크롤 사용 (react-virtual).
- useEffect 의존성 배열 누락 금지.
- 이미지: next/image 사용, width/height 필수.

## 보안
- 환경 변수: `process.env.DB_PASSWORD` (코드에 직접 값 금지)
- SQL: TypeORM QueryBuilder 또는 Repository 패턴 사용. raw query 금지.
- XSS: 사용자 입력을 innerHTML에 직접 넣지 마라. DOMPurify 필수.

## 테스트
- 유틸리티 함수: 유닛 테스트 필수 (최소 3개 케이스).
- API 엔드포인트: 통합 테스트 필수 (정상/에러/엣지 케이스).
- 커버리지 80% 미만이면 CI 실패.

## 커밋
- 형식: `type(scope): subject` (예: `feat(auth): add JWT refresh`)
- type: feat, fix, refactor, chore, docs, test
- subject: 영문 소문자, 명령형, 50자 이내, 마침표 없음
```

&nbsp;

차이가 보이는가?

&nbsp;

**구체적인 숫자, 명확한 예시, "하지 마라"가 있는 규칙이 좋은 규칙이다.**

&nbsp;

&nbsp;

---

&nbsp;

# 5. 규칙 파일 크기 관리

&nbsp;

## 200줄 넘으면 안 되는 이유

&nbsp;

규칙 파일은 AI가 **매 세션마다 읽는다.**

&nbsp;

500줄짜리 규칙 파일 = AI의 컨텍스트 윈도우를 낭비하는 것.

&nbsp;

더 심각한 문제: **규칙이 너무 많으면 AI가 중요한 규칙을 놓친다.**

&nbsp;

```
# 안 좋은 예
규칙 파일 1개, 500줄
→ AI가 전부 읽지만 중요도 구분을 못 함
→ 보안 규칙을 놓치고 코딩 스타일만 지킴

# 좋은 예
메인 규칙 파일 100줄 + 참조 파일 3개
→ AI가 핵심만 읽고, 필요시 참조
```

&nbsp;

## @import로 다른 파일 참조하기

&nbsp;

```markdown
# 프로젝트 규칙 (메인 파일)

## 필수 규칙 (항상 적용)
- TypeScript strict mode
- any 타입 금지
- 커밋 컨벤션: conventional commits

## 참조 문서
- 아키텍처 상세: @docs/architecture.md
- API 설계 가이드: @docs/api-guide.md
- DB 스키마: @docs/database.md
- 배포 가이드: @docs/deploy.md
```

&nbsp;

메인 파일은 **"항상 지켜야 하는 핵심 규칙"**만 담고,

상세 내용은 별도 파일로 분리한다.

&nbsp;

AI는 관련 작업을 할 때만 참조 파일을 읽는다.

&nbsp;

&nbsp;

---

&nbsp;

# 6. 규칙 파일 작성 체크리스트

&nbsp;

```markdown
[ ] 빌드/실행 명령이 있는가?
[ ] "하지 마라"가 명시되어 있는가?
[ ] 네이밍 규칙이 구체적인가? (camelCase, PascalCase 등)
[ ] 아키텍처 레이어가 정의되어 있는가?
[ ] 커밋 컨벤션이 예시와 함께 있는가?
[ ] 보안 금지 사항이 있는가?
[ ] 배포 프로세스가 있는가?
[ ] 200줄 이내인가?
[ ] 모호한 표현("깨끗하게", "적절히")을 제거했는가?
[ ] 구체적인 숫자와 예시가 있는가?
```

&nbsp;

&nbsp;

---

&nbsp;

# 7. 실전: 규칙 파일 템플릿

&nbsp;

지금 바로 사용할 수 있는 템플릿이다.

&nbsp;

```markdown
# 프로젝트 규칙

## 프로젝트 개요
[프로젝트 이름]. [한 줄 설명].
스택: [프레임워크], [언어], [DB], [기타]

## 빌드 & 실행
- 개발: `[명령어]`
- 빌드: `[명령어]`
- 테스트: `[명령어]`
- 금지: [수동 빌드 후 직접 복사 등]

## 디렉토리 구조
- `src/app/` — [역할]
- `src/services/` — [역할]
- `src/types/` — [역할]

## 코딩 컨벤션
- 네이밍: [규칙]
- 함수: [줄 수 제한], [파라미터 제한]
- 타입: [any 금지 등]
- Import 순서: [규칙]

## 아키텍처 규칙
- [레이어 분리 규칙]
- [금지 패턴]

## 커밋 컨벤션
- 형식: `type(scope): subject`
- 예시: `feat(auth): add login API`

## 보안
- [하드코딩 금지]
- [SQL Injection 방지]
- [XSS 방지]

## 배포
- [프로세스]
- [금지 사항]
```

&nbsp;

이 템플릿을 프로젝트에 맞게 채우면 된다. 15분이면 충분하다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI 하네스 3편] 메모리와 Skills — AI가 같은 실수를 반복하지 않게**

&nbsp;

규칙 파일이 "현재 규칙"이라면, 메모리는 "축적된 경험"이다.

피드백을 저장하고, 반복 작업을 자동화하는 방법을 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

AI, AI에이전트, AI하네스, 프로젝트규칙, 코딩컨벤션, 개발생산성, 기업AI, 코드품질
