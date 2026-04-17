# [AI 하네스 3편] 메모리와 Skills — AI가 같은 실수를 반복하지 않게

&nbsp;

규칙 파일은 "이렇게 해라"를 알려준다.

&nbsp;

하지만 규칙 파일만으로는 부족한 순간이 온다.

&nbsp;

- "저번에 말했잖아, 배포할 때 staging 스크립트 써라고"
- "또 그 방식으로 짰네? 지난번에 고쳤잖아"
- "커밋할 때마다 왜 이렇게 설명을 길게 해야 하지"

&nbsp;

**메모리**는 피드백을 축적해서 같은 실수를 반복하지 않게 하고,

**Skills**는 반복 작업을 자동화해서 설명 자체를 없앤다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 메모리 시스템

&nbsp;

AI 에이전트의 메모리는 대화가 끝나도 유지되는 정보다.

&nbsp;

다음 세션, 다음 주, 다음 달에도 AI는 이 정보를 기억한다.

&nbsp;

## 1-1. 메모리의 4가지 유형

&nbsp;

```
┌─────────────────────────────────────────────┐
│                  메모리 구조                   │
├──────────────┬──────────────────────────────┤
│ 피드백 메모리  │ "이렇게 하지 마" 축적           │
│ 프로젝트 메모리 │ 진행 상황, 미해결 이슈          │
│ 사용자 메모리  │ 개인 선호, 역할                │
│ 참조 메모리    │ 외부 리소스 위치, 가이드 링크    │
└──────────────┴──────────────────────────────┘
```

&nbsp;

### 피드백 메모리

&nbsp;

개발 중 AI에게 교정한 내용이 자동으로 저장된다.

&nbsp;

```markdown
# 피드백 메모리

## 배포 규칙
- 배포 시 반드시 `npm run deploy:staging` 사용 (수동 빌드 금지)
- deploy 실패 시 수동 build + pack 절대 금지 (.env 사고 위험)

## 코딩 규칙
- API 응답은 항상 `{ success: boolean, data: T, error?: string }` 형식
- 에러 핸들링에서 catch 블록 비우지 말 것

## DB 규칙
- 마이그레이션 파일 수동 수정 금지
- seed 데이터 변경 시 반드시 팀 공유
```

&nbsp;

**시나리오:**

```
세션 1:
  개발자: "배포해줘"
  AI: npm run build 후 서버에 복사합니다.
  개발자: "아니! deploy 스크립트 써야지!"
  → 메모리 저장: "배포 시 반드시 deploy 스크립트 사용"

세션 2:
  개발자: "배포해줘"
  AI: npm run deploy:staging을 실행합니다.  ← 기억하고 있음
```

&nbsp;

### 프로젝트 메모리

&nbsp;

진행 중인 작업, 미해결 이슈를 추적한다.

&nbsp;

```markdown
# 프로젝트 메모리

## 진행 중
- 결제 API v2 리팩토링 (payment/ 디렉토리)
- 다국어 번역 48키 외부 업체 요청 중 (4월 말 완료 예정)

## 미해결 이슈
- 외부 API 서버에서 간헐적 500 에러 (담당자 확인 중)
- 모바일 웹뷰에서 키보드 올라올 때 레이아웃 깨짐

## 최근 결정 사항
- ORM은 Prisma → TypeORM으로 전환 (2024-04 결정)
- 상태관리: Zustand → Redux Toolkit으로 통일
```

&nbsp;

**시나리오:**

```
개발자: "결제 API 작업 이어서 해줘"
AI: (프로젝트 메모리를 읽고)
    "결제 API v2 리팩토링 중이었습니다. payment/ 디렉토리에서
     이전에 createOrder와 processPayment를 분리하는 작업을 하고 있었습니다."
```

&nbsp;

별도의 설명 없이 바로 이어서 작업할 수 있다.

&nbsp;

### 사용자 메모리

&nbsp;

개인별 선호와 역할 정보다.

&nbsp;

```markdown
# 사용자 메모리

## 프로필
- 이름: 김개발
- 역할: 풀스택 개발자
- 이메일: dev@company.co.kr

## 선호
- 응답 언어: 한국어
- 코드 설명: 주석 대신 대화로
- 커밋 메시지: 한국어 설명 선호
```

&nbsp;

### 참조 메모리

&nbsp;

외부 리소스의 위치를 기억한다.

&nbsp;

```markdown
# 참조 메모리

## 문서
- API 명세서: Notion > 기술팀 > API Docs
- 디자인 시스템: Figma > Design System v2
- 인프라 구성도: Confluence > Infra Architecture

## 가이드
- Node.js 스레드/프로세스 가이드: docs/node-thread-guide.md
- 배포 매뉴얼: docs/deploy-manual.md
```

&nbsp;

&nbsp;

---

&nbsp;

# 2. 메모리 활용 실전 예시

&nbsp;

## 2-1. 이커머스 팀

&nbsp;

```markdown
# 피드백 메모리

## 주문 처리
- 주문 상태 변경 시 반드시 이벤트 발행 (OrderStatusChanged)
- 재고 차감은 트랜잭션 내에서 (optimistic lock 사용 금지, pessimistic lock 사용)
- 할인 금액은 정수로 계산 (Math.floor), 반올림 사용 금지

## 배송
- 배송 상태 API는 외부 업체 API 직접 호출하지 말고 캐시 사용 (5분 TTL)
- 배송비 무료 조건: 50000원 이상 (하드코딩 금지, 설정값으로)
```

&nbsp;

이 메모리 덕분에 AI는:

- 주문 상태 변경 코드를 작성할 때 **자동으로 이벤트 발행 코드를 포함**한다
- 할인 계산에서 **Math.round 대신 Math.floor를 사용**한다
- 배송비 조건을 **코드에 직접 넣지 않고 설정값을 참조**한다

&nbsp;

## 2-2. 핀테크 팀

&nbsp;

```markdown
# 피드백 메모리

## 규정 준수
- 로그인 실패 5회 시 계정 잠금 (5가 아닌 다른 값 사용 금지, 규정)
- 비밀번호 변경 후 기존 세션 전체 무효화 필수
- 거래 내역 조회 시 본인 것만 반환 (tenantId가 아닌 userId로 필터)

## API 규칙
- 금액 필드명: amount (price, cost 금지)
- 통화 필드명: currency (항상 명시, 기본값 없음)
- 거래 ID: UUID v4 (auto-increment 금지)
```

&nbsp;

## 2-3. 메모리가 쌓이는 과정

&nbsp;

```
1주차: 메모리 5줄
  - "any 타입 쓰지 마"
  - "deploy 스크립트 써"

2주차: 메모리 15줄
  + "API 응답 형식 통일해"
  + "에러 코드는 상수로"
  + "DB 쿼리는 서비스 레이어에서만"

1개월: 메모리 40줄
  → 이 시점에서 AI는 "팀의 규칙을 이해하는 주니어 개발자" 수준

3개월: 메모리 80줄
  → AI는 "사내 위키를 다 읽은 중니어 개발자" 수준
```

&nbsp;

**메모리는 시간이 갈수록 가치가 올라간다.**

&nbsp;

&nbsp;

---

&nbsp;

# 3. Skills (커스텀 슬래시 커맨드)

&nbsp;

Skills는 반복되는 워크플로우를 하나의 명령어로 묶는 것이다.

&nbsp;

## 3-1. 기본 Skills

&nbsp;

### /commit — 커밋 자동화

&nbsp;

```markdown
# /commit Skill 정의

## 실행 단계
1. `git status`와 `git diff`로 변경 사항 확인
2. 변경 내용을 분석하여 커밋 메시지 생성
   - 형식: `type(scope): subject`
   - type은 변경 성격에 맞게 (feat/fix/refactor/chore)
   - scope는 변경된 주요 디렉토리
   - subject는 변경의 "왜"를 설명
3. 시크릿 파일(.env, credentials) 포함 여부 확인 — 포함 시 경고
4. 커밋 실행

## 규칙
- 커밋 메시지는 영문, 소문자 시작, 50자 이내
- body에 상세 설명 (선택)
- footer에 Jira 티켓 (있으면)
```

&nbsp;

**사용 전:**

```
개발자: "지금 변경사항 커밋해줘. 형식은 conventional commits로,
        scope는 auth이고, 변경 내용을 잘 설명해줘.
        아 그리고 .env 파일은 빼고."

→ 매번 이걸 설명해야 함
```

&nbsp;

**사용 후:**

```
개발자: /commit
AI: (자동으로 diff 확인 → 메시지 생성 → 커밋)
    "feat(auth): add JWT refresh token rotation"
```

&nbsp;

### /deploy — 배포 자동화

&nbsp;

```markdown
# /deploy Skill 정의

## 실행 단계
1. 변경된 앱 감지 (git diff로 어떤 앱이 변경되었는지)
2. 변경된 앱만 버전 bump (package.json)
3. 버전 bump 커밋
4. 빌드 실행 (`npm run build:{env}`)
5. 패키징 (`npm run deploy:{app}:{env}`)
6. 결과물 검증 (파일 크기, 포함 파일 확인)

## 규칙
- 변경되지 않은 앱은 빌드하지 않음
- 빌드 실패 시 수동 재시도 금지, 원인 먼저 파악
- 환경(staging/prod)은 반드시 명시
```

&nbsp;

### /review-pr — PR 리뷰 자동화

&nbsp;

```markdown
# /review-pr Skill 정의

## 체크 항목
1. **코드 품질**: 함수 길이, 타입 안전성, 중복 코드
2. **보안**: 하드코딩, SQL Injection, XSS 가능성
3. **성능**: N+1 쿼리, 불필요한 렌더링, 메모리 누수
4. **컨벤션**: 네이밍, import 순서, 커밋 메시지
5. **테스트**: 변경 사항에 대한 테스트 존재 여부

## 출력 형식
- Critical: 반드시 수정
- Warning: 권장 수정
- Info: 참고 사항
```

&nbsp;

### /create-api — API 엔드포인트 자동 생성

&nbsp;

```markdown
# /create-api Skill 정의

## 입력
- 엔티티 이름 (예: Product)
- 필드 목록 (예: name:string, price:number, category:string)

## 생성 파일
1. `types/product.ts` — 타입 정의
2. `services/product-service.ts` — CRUD 비즈니스 로직
3. `app/api/products/route.ts` — GET (목록), POST (생성)
4. `app/api/products/[id]/route.ts` — GET (단건), PUT, DELETE
5. `tests/product.test.ts` — 기본 테스트

## 규칙
- 아키텍처 레이어 분리 준수
- 에러 핸들링 포함
- 페이지네이션 기본 적용 (목록 API)
```

&nbsp;

## 3-2. 기업 전용 Skills

&nbsp;

### /onboard — 신규 팀원 온보딩

&nbsp;

```markdown
# /onboard Skill 정의

## 실행 내용
1. 프로젝트 개요 설명 (스택, 아키텍처)
2. 디렉토리 구조 안내
3. 개발 환경 셋업 가이드
4. 주요 파일 위치 안내
5. 자주 사용하는 명령어 목록
6. 팀 컨벤션 요약
```

&nbsp;

```bash
개발자: /onboard
AI: "이 프로젝트는 Next.js 14 + TypeScript + Redux Toolkit 기반의
     이커머스 플랫폼입니다.

     주요 디렉토리:
     - apps/web — 고객 웹 (port 3000)
     - apps/admin — 관리자 (port 3001)
     - packages/shared — 공유 유틸리티

     개발 시작:
     1. npm install
     2. cp .env.example .env.local
     3. npm run dev

     반드시 읽어야 할 파일:
     - docs/architecture.md
     - docs/api-guide.md
     ..."
```

&nbsp;

### /incident — 장애 대응 체크리스트

&nbsp;

```markdown
# /incident Skill 정의

## 실행 단계
1. 현재 에러 로그 확인 (`npm run logs:tail`)
2. 최근 배포 이력 확인 (`git log --oneline -10`)
3. 서버 상태 확인 (health check)
4. 체크리스트 출력:
   - [ ] 에러 발생 시점 확인
   - [ ] 최근 배포와 관련 있는가?
   - [ ] 외부 API 장애인가?
   - [ ] DB 연결 문제인가?
   - [ ] 롤백이 필요한가?
5. 장애 보고서 초안 생성
```

&nbsp;

### /release-note — 릴리즈 노트 자동 생성

&nbsp;

```markdown
# /release-note Skill 정의

## 실행 단계
1. 마지막 릴리즈 태그 이후 커밋 수집
2. 커밋을 카테고리별 분류 (feat/fix/refactor/chore)
3. 릴리즈 노트 마크다운 생성

## 출력 형식
### v1.2.0 (2026-04-16)

#### 새로운 기능
- 결제 수단에 네이버페이 추가 (#123)
- 주문 내역 CSV 다운로드 기능 (#125)

#### 버그 수정
- 장바구니 수량 0으로 변경 가능한 버그 수정 (#130)

#### 개선
- 주문 목록 조회 쿼리 성능 개선 (3초 → 0.3초)
```

&nbsp;

### /security-check — 보안 점검

&nbsp;

```markdown
# /security-check Skill 정의

## 체크 항목 (OWASP 기반)
1. **인젝션**: SQL Injection, XSS, Command Injection 가능성
2. **인증**: 하드코딩된 비밀번호, 약한 해시 알고리즘
3. **노출**: 에러 메시지에 민감 정보, console.log에 토큰
4. **접근 제어**: 권한 체크 누락, IDOR 가능성
5. **의존성**: 알려진 취약점이 있는 패키지 (npm audit)

## 출력
- HIGH: 즉시 수정 필요
- MEDIUM: 이번 스프린트 내 수정
- LOW: 백로그에 추가
```

&nbsp;

&nbsp;

---

&nbsp;

# 4. Skills 정의 방법

&nbsp;

Skills는 마크다운 파일로 정의한다. 복잡한 설정 파일이 아니다.

&nbsp;

## 4-1. 파일 구조

&nbsp;

```bash
.ai/
├── skills/
│   ├── commit.md        # /commit
│   ├── deploy.md        # /deploy
│   ├── review-pr.md     # /review-pr
│   ├── create-api.md    # /create-api
│   ├── onboard.md       # /onboard
│   └── security.md      # /security-check
```

&nbsp;

## 4-2. Skill 파일 작성 예시

&nbsp;

```markdown
# /commit

커밋을 생성합니다. 변경 사항을 분석하고 컨벤션에 맞는 메시지를 자동으로 만듭니다.

## 단계

1. `git status`로 변경 파일 확인
2. `git diff`로 변경 내용 분석
3. 변경 성격에 맞는 type 선택 (feat/fix/refactor/chore/docs)
4. 변경된 주요 디렉토리로 scope 결정
5. 변경의 목적을 요약하여 subject 작성
6. .env, credentials 등 시크릿 파일이 포함되어 있으면 경고하고 제외
7. 커밋 실행

## 메시지 형식

```
type(scope): subject

[선택] 상세 설명

[선택] Refs: JIRA-123
Co-Authored-By: AI Agent
```

## 예시

```bash
feat(payment): add Naver Pay integration
fix(cart): prevent quantity from going below 1
refactor(auth): extract token validation to middleware
```
```

&nbsp;

## 4-3. 인자를 받는 Skill

&nbsp;

```markdown
# /create-api

API 엔드포인트를 자동으로 생성합니다.

## 사용법
/create-api Product name:string price:number category:string

## 단계

1. 인자에서 엔티티 이름과 필드를 파싱
2. 아래 파일들을 순서대로 생성:
   - `src/types/{entity}.ts` — 타입 정의
   - `src/services/{entity}-service.ts` — CRUD 서비스
   - `src/app/api/{entity}s/route.ts` — 목록/생성 API
   - `src/app/api/{entity}s/[id]/route.ts` — 단건/수정/삭제 API
3. 각 파일은 프로젝트 규칙의 아키텍처 레이어를 따름
4. 에러 핸들링, 입력 검증, 페이지네이션 기본 포함
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. 메모리와 Skills의 시너지

&nbsp;

메모리와 Skills를 함께 쓰면 효과가 배가 된다.

&nbsp;

```
┌────────────────────────────────────────────┐
│                                            │
│  Skills가 실행될 때 메모리를 참조한다          │
│                                            │
│  /commit 실행 시:                           │
│  ├── Skill 규칙: conventional commits 형식   │
│  ├── 메모리: "커밋 메시지에 Jira 티켓 포함"    │
│  └── 메모리: "scope에 변경된 패키지명 사용"    │
│                                            │
│  /deploy 실행 시:                           │
│  ├── Skill 규칙: 버전 bump → 빌드 → 패키징   │
│  ├── 메모리: "수동 빌드 금지"                 │
│  └── 메모리: "변경된 앱만 빌드"               │
│                                            │
└────────────────────────────────────────────┘
```

&nbsp;

**Skills는 "어떻게 하는가"를 정의하고,**

**메모리는 "주의할 점"을 보완한다.**

&nbsp;

둘이 합쳐지면 AI는 팀의 워크플로우를 완벽히 따르는 자동화 파이프라인이 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 6. 메모리 관리 팁

&nbsp;

## 6-1. 정기적으로 정리하라

&nbsp;

```markdown
# 정리 전 (6개월 분량)
- 2024-01: API 응답 형식 통일
- 2024-01: fetch 대신 axios 사용 → 취소: 2024-03에 fetch로 복귀
- 2024-02: 에러 코드 상수화
- 2024-03: fetch로 복귀 (axios 제거)
- ...
```

&nbsp;

```markdown
# 정리 후 (현재 유효한 것만)
- API 응답 형식: { success, data, error }
- HTTP 클라이언트: fetch 사용 (axios 금지)
- 에러 코드: constants/error-codes.ts에 상수로 정의
```

&nbsp;

## 6-2. 카테고리로 분류하라

&nbsp;

```markdown
# 메모리

## 배포
- ...

## 코딩
- ...

## DB
- ...

## 보안
- ...

## 진행 중
- ...
```

&nbsp;

## 6-3. 구체적으로 작성하라

&nbsp;

```markdown
# 나쁜 메모리
- 배포 조심

# 좋은 메모리
- 배포 시 npm run deploy:{app}:{env} 사용. npm run build 후 수동 복사 금지.
  수동 복사 시 .env 파일이 개발 환경 것으로 덮어씌워지는 사고 발생함.
```

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI 하네스 4편] Hooks와 보안 — AI의 행동을 강제하고 위험을 차단하기**

&nbsp;

규칙 파일은 "권고"이고, 메모리는 "경험"이다.

하지만 보안은 "권고"로는 부족하다. **물리적으로 차단**해야 한다.

Hooks로 AI의 위험한 행동을 원천 차단하는 방법을 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

AI, AI에이전트, AI하네스, 메모리, Skills, 자동화, 개발생산성, 기업AI, 워크플로우
