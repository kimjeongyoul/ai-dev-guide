# [AI 하네스 1편] AI 하네스란 — AI에게 회사 규칙을 가르치는 법

&nbsp;

AI 코딩 에이전트를 도입했다.

&nbsp;

개발자 A가 쓰면 깔끔한 코드가 나온다.

개발자 B가 쓰면 `console.log`가 100개 남는다.

개발자 C가 쓰면 `.env`에 비밀번호를 하드코딩한다.

&nbsp;

**같은 AI인데 결과물이 왜 이렇게 다를까?**

&nbsp;

AI에게 "회사 규칙"을 알려주지 않았기 때문이다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. AI를 그냥 쓰면 생기는 문제

&nbsp;

## 1-1. 매번 다른 코딩 스타일

&nbsp;

```typescript
// 어제 AI가 쓴 코드
const getUserData = async (id: string) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
};

// 오늘 AI가 쓴 코드
async function fetchUser(userId: number) {
  const response = await axios.get('/api/users/' + userId);
  return response.data;
}
```

&nbsp;

같은 AI인데 네이밍, 타입, HTTP 라이브러리, 함수 선언 방식이 전부 다르다.

&nbsp;

팀 컨벤션? AI는 모른다. **알려주지 않았으니까.**

&nbsp;

## 1-2. 보안 정책 무시

&nbsp;

```typescript
// AI가 자연스럽게 작성하는 코드
const DB_PASSWORD = "admin1234";  // 하드코딩
console.log("user data:", userData);  // 민감정보 로그 출력
const query = `SELECT * FROM users WHERE id = '${userId}'`;  // SQL Injection
```

&nbsp;

AI는 "동작하는 코드"를 만든다.

"안전한 코드"를 만드는 건 규칙을 알려줬을 때만 가능하다.

&nbsp;

## 1-3. 배포 규칙을 모른다

&nbsp;

```bash
# AI가 하는 것
npm run build
scp -r ./dist server:/app/

# 실제 규칙
npm run build:staging   # 환경별 빌드 스크립트 사용
npm run deploy:staging  # 배포 스크립트로만 배포
# 수동 빌드 + 수동 복사 = 환경변수 사고
```

&nbsp;

빌드 명령이 `build`인지 `build:prod`인지, 배포가 스크립트인지 CI/CD인지 AI는 모른다.

개발자가 매번 설명해야 한다. **매번.**

&nbsp;

## 1-4. 팀 컨벤션 무시

&nbsp;

```bash
# AI의 커밋 메시지
git commit -m "fix bug"
git commit -m "update code"
git commit -m "added new feature for user management"

# 팀 컨벤션
git commit -m "fix(auth): resolve token refresh race condition"
git commit -m "feat(user): add email verification flow"
```

&nbsp;

Conventional Commits? Jira 티켓 연동? AI는 알 리가 없다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 하네스(Harness)란 무엇인가

&nbsp;

**하네스 = AI 모델을 감싸는 실행 환경.**

&nbsp;

```
┌──────────────────────────────────────────┐
│              AI 하네스                     │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │ 규칙 파일  │  │  메모리   │  │ Skills │ │
│  └──────────┘  └──────────┘  └────────┘ │
│                                          │
│  ┌──────────┐  ┌──────────┐             │
│  │  Hooks   │  │ 권한 관리  │             │
│  └──────────┘  └──────────┘             │
│                                          │
│            ┌──────────┐                  │
│            │  AI 모델  │                  │
│            └──────────┘                  │
└──────────────────────────────────────────┘
```

&nbsp;

AI가 아무렇게나 동작하지 않고, **기업의 코딩 컨벤션, 보안 정책, 배포 규칙, 워크플로우를 따르게 만드는 구조**다.

&nbsp;

비유하면:

- AI 모델 = 신입 개발자 (실력은 있지만 회사 규칙을 모른다)
- 하네스 = 온보딩 가이드 + 사내 위키 + CI/CD + 코드 리뷰 룰

&nbsp;

&nbsp;

---

&nbsp;

# 3. 하네스의 구성 요소 5가지

&nbsp;

## 3-1. 프로젝트 규칙 파일

&nbsp;

AI가 세션 시작 시 자동으로 읽는 설정 파일이다.

빌드 명령, 코딩 스타일, 아키텍처 규칙, 커밋 컨벤션 등을 정의한다.

&nbsp;

```markdown
# 프로젝트 규칙

## 빌드
- `npm run build:prod` 명령만 사용
- 수동 빌드 후 직접 복사 금지

## 코딩 컨벤션
- 함수는 30줄 이내
- any 타입 사용 금지
- 네이밍: camelCase (변수/함수), PascalCase (컴포넌트/타입)

## 보안
- 비밀번호/토큰 하드코딩 금지
- console.log에 사용자 정보 출력 금지
- SQL raw query 사용 금지, ORM 사용 필수
```

&nbsp;

이것 하나만 있어도 AI의 결과물 품질이 **극적으로 달라진다.**

&nbsp;

## 3-2. 메모리

&nbsp;

AI가 대화 세션이 끝나도 기억하는 정보다.

&nbsp;

```markdown
# 메모리

## 피드백
- 배포 시 반드시 deploy 스크립트 사용 (수동 빌드 금지)
- 테스트 없이 커밋하지 말 것

## 진행 상황
- 결제 API v2 리팩토링 진행 중
- i18n 번역 48키 외부 업체 요청 중
```

&nbsp;

"이전에 말했잖아"를 반복하지 않아도 된다.

&nbsp;

## 3-3. Skills (커스텀 슬래시 커맨드)

&nbsp;

반복되는 작업을 자동화하는 명령어다.

&nbsp;

```bash
/commit      # diff 확인 → 컨벤션에 맞는 커밋 메시지 생성 → 커밋
/deploy      # 버전 올리기 → 빌드 → 패키징 → 배포
/review-pr   # PR 코드 리뷰 (보안, 성능, 컨벤션 체크)
/create-api  # 엔티티 → 서비스 → 라우트 자동 생성
```

&nbsp;

개발자가 매번 "커밋 메시지는 이런 형식으로..."를 설명하지 않아도 된다.

&nbsp;

## 3-4. Hooks (강제 실행 규칙)

&nbsp;

규칙 파일이 "권고"라면, Hooks는 **"강제"**다.

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "bash",
        "command": "if echo '$INPUT' | grep -q 'rm -rf'; then echo 'BLOCK: 삭제 명령 차단'; exit 1; fi"
      }
    ],
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "npx prettier --write $FILE"
      }
    ]
  }
}
```

&nbsp;

- `rm -rf` 실행 시도 → **차단**
- 파일 수정 후 → **자동으로 포맷팅**
- `.env` 수정 시도 → **경고 후 차단**

&nbsp;

AI가 규칙을 "몰라서" 어기는 것과, Hooks가 "물리적으로" 막는 것은 다르다.

&nbsp;

## 3-5. 권한 관리

&nbsp;

AI가 할 수 있는 것과 없는 것을 명확히 구분한다.

&nbsp;

```json
{
  "permissions": {
    "allow": [
      "read:**",
      "search:**",
      "bash:npm run lint",
      "bash:npm run test"
    ],
    "deny": [
      "bash:rm -rf *",
      "bash:git push --force",
      "edit:.env*",
      "edit:**/credentials*"
    ]
  }
}
```

&nbsp;

- 읽기, 검색: **자동 허용**
- 파일 수정, 명령 실행: **확인 후 허용**
- 삭제, force push, 시크릿 파일 수정: **차단**

&nbsp;

&nbsp;

---

&nbsp;

# 4. 하네스가 없는 팀 vs 있는 팀

&nbsp;

## Before: 하네스 없음

&nbsp;

| 상황 | 결과 |
|------|------|
| 개발자 A가 AI로 API 작성 | fetch 사용, camelCase |
| 개발자 B가 AI로 API 작성 | axios 사용, snake_case |
| 개발자 C가 AI로 배포 | `npm run build` 후 수동 복사 |
| 코드 리뷰 | "이거 왜 이렇게 짰어?" 반복 |
| 보안 점검 | `.env` 하드코딩 3건 발견 |

&nbsp;

> "AI 쓰니까 더 느려졌다"는 말이 나온다.

&nbsp;

## After: 하네스 있음

&nbsp;

| 상황 | 결과 |
|------|------|
| 개발자 A가 AI로 API 작성 | 규칙대로 fetch + camelCase + 에러 핸들링 |
| 개발자 B가 AI로 API 작성 | **동일한 결과** |
| 개발자 C가 AI로 배포 | `/deploy` 명령으로 자동 처리 |
| 코드 리뷰 | 컨벤션 이슈 0건, 로직만 리뷰 |
| 보안 점검 | Hooks가 하드코딩 원천 차단 |

&nbsp;

> 누가 AI를 써도 **같은 규칙, 같은 품질.**

&nbsp;

&nbsp;

---

&nbsp;

# 5. 기업 규모별 하네스 복잡도

&nbsp;

## 5-1. 1인 개발자 / 사이드 프로젝트

&nbsp;

```
필요한 것: 규칙 파일 1개 (50줄)
```

&nbsp;

```markdown
# 프로젝트 규칙

## 스택
- Next.js 14, TypeScript, TailwindCSS
- DB: Prisma + PostgreSQL

## 빌드
- npm run dev (개발)
- npm run build (빌드)

## 규칙
- any 타입 금지
- 커밋: conventional commits 형식
```

&nbsp;

이것만 있어도 AI가 프로젝트 구조를 이해하고 일관된 코드를 작성한다.

&nbsp;

## 5-2. 5인 팀 / 스타트업

&nbsp;

```
필요한 것: 규칙 파일 + 메모리 + Skills
```

&nbsp;

```markdown
# 규칙 파일 (100줄)
- 코딩 컨벤션, 아키텍처, 배포 규칙

# 메모리
- 팀 피드백 축적
- "API 응답은 항상 { success, data, error } 형식"
- "테스트 커버리지 80% 미만이면 머지 불가"

# Skills
- /commit: 팀 커밋 컨벤션 자동 적용
- /deploy: staging → production 배포 파이프라인
```

&nbsp;

## 5-3. 20인 이상 / 중견 기업

&nbsp;

```
필요한 것: 규칙 파일 + 메모리 + Skills + Hooks + 권한 + MCP
```

&nbsp;

```markdown
# 규칙 파일 (계층 구조)
- 전사 공통 규칙 (보안, 코딩 스타일)
  - 프론트엔드 팀 규칙 (React, 상태관리)
    - 프로젝트 A 규칙 (이 레포 전용)

# 메모리
- 팀별 피드백, 프로젝트별 컨텍스트

# Skills
- /commit, /deploy, /review-pr
- /security-check (OWASP 기준)
- /onboard (신규 팀원 온보딩)

# Hooks
- 파일 수정 시 자동 린트
- 위험 명령 차단
- 특정 디렉토리(결제, 인증) 수정 시 경고

# 권한
- 주니어: 읽기 + 제한된 수정
- 시니어: 전체 수정 + 배포
- 리드: 규칙 파일 수정 권한

# MCP (외부 도구 연동)
- Jira 연동 (이슈 생성/업데이트)
- Slack 알림 (배포 완료 시)
- 모니터링 대시보드 조회
```

&nbsp;

&nbsp;

---

&nbsp;

# 6. 하네스를 한 문장으로 정리하면

&nbsp;

> **하네스 = AI가 "우리 회사 개발자"처럼 행동하게 만드는 실행 환경.**

&nbsp;

규칙 파일 하나로 시작해서, 메모리, Skills, Hooks, 권한 관리를 점진적으로 추가한다.

&nbsp;

처음부터 완벽할 필요 없다.

**규칙 파일 50줄이면 이미 "하네스가 없는 팀"과 차이가 난다.**

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI 하네스 2편] 프로젝트 규칙 파일 작성법 — AI가 읽는 사내 코딩 가이드**

&nbsp;

규칙 파일에 뭘 넣어야 하는지, 업종별 실전 예시, 잘 쓴 규칙과 못 쓴 규칙의 차이를 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

AI, AI에이전트, AI하네스, 코딩에이전트, DevOps, 개발생산성, AI거버넌스, 기업AI
