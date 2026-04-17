# [AI 하네스 4편] Hooks와 보안 — AI의 행동을 강제하고 위험을 차단하기

&nbsp;

규칙 파일에 "비밀번호 하드코딩 금지"라고 썼다.

&nbsp;

AI가 지킬까?

&nbsp;

**대부분은 지킨다. 하지만 "대부분"은 "항상"이 아니다.**

&nbsp;

복잡한 맥락에서, 긴 대화에서, AI는 규칙을 놓칠 수 있다.

"실수로" `.env` 파일에 프로덕션 비밀번호를 남기면?

"실수로" `git push --force`를 main 브랜치에 실행하면?

&nbsp;

**규칙 파일 = "이렇게 해주세요" (권고)**

**Hooks = "이렇게 안 하면 실행이 안 됩니다" (강제)**

&nbsp;

보안은 권고가 아니라 강제여야 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. Hooks란 무엇인가

&nbsp;

Hooks는 AI 에이전트의 특정 행동 **전후에 자동으로 실행되는 스크립트**다.

&nbsp;

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  AI가 도구를  │────▶│   Hook 실행   │────▶│  도구 실행    │
│  사용하려 함  │     │  (검증/차단)   │     │  (허용된 경우) │
└─────────────┘     └──────────────┘     └──────────────┘
                           │
                           │ 차단 시
                           ▼
                    ┌──────────────┐
                    │  실행 거부    │
                    │  + 사유 출력  │
                    └──────────────┘
```

&nbsp;

Git의 pre-commit hook, CI/CD의 파이프라인 게이트와 같은 개념이다.

AI가 "규칙을 모르거나 잊어도" Hook이 물리적으로 막는다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. Hook 종류와 실전 예시

&nbsp;

## 2-1. PreToolUse — 도구 실행 전

&nbsp;

AI가 도구(bash, 파일 수정 등)를 사용하기 **직전에** 실행된다.

위험한 명령을 **사전 차단**하는 용도다.

&nbsp;

### 위험 명령 차단

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "bash",
        "command": "bash -c 'if echo \"$TOOL_INPUT\" | grep -qE \"rm\\s+-rf|rm\\s+-r\\s+/\"; then echo \"BLOCK: rm -rf 명령은 차단됩니다\"; exit 1; fi'"
      }
    ]
  }
}
```

&nbsp;

```
AI: rm -rf /tmp/build 를 실행하겠습니다.
Hook: BLOCK: rm -rf 명령은 차단됩니다.
AI: 삭제 대신 다른 방법을 찾겠습니다.
```

&nbsp;

### .env 파일 수정 차단

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$TOOL_INPUT\" | grep -qE \"\\.env\"; then echo \"BLOCK: .env 파일은 직접 수정할 수 없습니다. 환경 변수는 관리자에게 요청하세요.\"; exit 1; fi'"
      }
    ]
  }
}
```

&nbsp;

### main 브랜치 직접 push 차단

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "bash",
        "command": "bash -c 'if echo \"$TOOL_INPUT\" | grep -qE \"git\\s+push.*(main|master)\"; then echo \"BLOCK: main 브랜치에 직접 push할 수 없습니다. PR을 생성하세요.\"; exit 1; fi'"
      }
    ]
  }
}
```

&nbsp;

### force push 전면 차단

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "bash",
        "command": "bash -c 'if echo \"$TOOL_INPUT\" | grep -qE \"git\\s+push\\s+--force|git\\s+push\\s+-f\"; then echo \"BLOCK: force push는 차단됩니다. 히스토리를 보존하세요.\"; exit 1; fi'"
      }
    ]
  }
}
```

&nbsp;

### 외부 패키지 설치 알림

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "bash",
        "command": "bash -c 'if echo \"$TOOL_INPUT\" | grep -qE \"npm\\s+install|yarn\\s+add|pnpm\\s+add\"; then echo \"WARN: 새 패키지를 설치하려고 합니다. 팀 승인이 필요할 수 있습니다.\"; fi'"
      }
    ]
  }
}
```

&nbsp;

> WARN은 경고만 하고 실행을 허용한다. BLOCK(exit 1)은 실행을 차단한다.

&nbsp;

## 2-2. PostToolUse — 도구 실행 후

&nbsp;

AI가 도구를 사용한 **직후에** 실행된다.

자동 포맷팅, 린트, 검증 등에 사용한다.

&nbsp;

### 파일 수정 후 자동 포맷팅

&nbsp;

```json
{
  "hooks": {
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "npx prettier --write $FILE_PATH"
      }
    ]
  }
}
```

&nbsp;

AI가 파일을 수정할 때마다 자동으로 Prettier가 실행된다.

코드 스타일이 항상 일관된다.

&nbsp;

### 파일 수정 후 자동 린트

&nbsp;

```json
{
  "hooks": {
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "npx eslint $FILE_PATH --fix --quiet"
      }
    ]
  }
}
```

&nbsp;

### TypeScript 타입 체크

&nbsp;

```json
{
  "hooks": {
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if [[ \"$FILE_PATH\" == *.ts || \"$FILE_PATH\" == *.tsx ]]; then npx tsc --noEmit --pretty 2>&1 | head -20; fi'"
      }
    ]
  }
}
```

&nbsp;

파일 수정 후 타입 에러가 있으면 즉시 알려준다.

&nbsp;

## 2-3. PreFileEdit — 파일 수정 전 (특정 파일 보호)

&nbsp;

특정 파일이나 디렉토리를 수정하려 할 때 경고하거나 차단한다.

&nbsp;

### 결제 로직 수정 시 경고

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$FILE_PATH\" | grep -qE \"(payment|checkout|billing)\"; then echo \"WARN: 결제 관련 코드를 수정합니다. 시니어 리뷰가 필요합니다.\"; fi'"
      }
    ]
  }
}
```

&nbsp;

### 인증 코드 수정 시 경고

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$FILE_PATH\" | grep -qE \"(auth|session|token|jwt)\"; then echo \"WARN: 인증 관련 코드를 수정합니다. 보안 리뷰가 필요합니다.\"; fi'"
      }
    ]
  }
}
```

&nbsp;

### DB 마이그레이션 파일 수정 차단

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$FILE_PATH\" | grep -qE \"migrations/\"; then echo \"BLOCK: 마이그레이션 파일은 직접 수정하지 마세요. 새 마이그레이션을 생성하세요.\"; exit 1; fi'"
      }
    ]
  }
}
```

&nbsp;

## 2-4. SessionStart — 세션 시작 시

&nbsp;

AI 에이전트 세션이 시작될 때 자동으로 실행된다.

&nbsp;

```json
{
  "hooks": {
    "sessionStart": [
      {
        "command": "bash -c 'echo \"=== 오늘의 컨텍스트 ===\"; echo \"브랜치: $(git branch --show-current)\"; echo \"미커밋 변경: $(git status --short | wc -l)건\"; echo \"최근 커밋: $(git log --oneline -3)\"'"
      }
    ]
  }
}
```

&nbsp;

세션 시작 시 자동으로:
- 현재 브랜치
- 미커밋 변경 사항
- 최근 커밋 히스토리

를 AI에게 알려준다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 보안 하네스 구축

&nbsp;

## 3-1. 비밀번호/토큰 하드코딩 감지

&nbsp;

```json
{
  "hooks": {
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if grep -nE \"(password|secret|token|api_key|apiKey)\\s*[=:]\\s*[\\x27\\\"][^\\x27\\\"]{8,}\" \"$FILE_PATH\" 2>/dev/null; then echo \"BLOCK: 하드코딩된 시크릿이 감지되었습니다. 환경 변수를 사용하세요.\"; exit 1; fi'"
      }
    ]
  }
}
```

&nbsp;

```typescript
// 감지됨 → 차단
const API_KEY = "sk-1234567890abcdef";  // BLOCK!

// 통과
const API_KEY = process.env.API_KEY;    // OK
```

&nbsp;

## 3-2. console.log 민감정보 출력 감지

&nbsp;

```json
{
  "hooks": {
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if grep -nE \"console\\.log.*\\b(password|token|secret|credential|ssn|cardNumber)\\b\" \"$FILE_PATH\" 2>/dev/null; then echo \"WARN: console.log에 민감한 변수명이 포함되어 있습니다. 제거하거나 마스킹하세요.\"; fi'"
      }
    ]
  }
}
```

&nbsp;

```typescript
// 경고 발생
console.log("user token:", token);       // WARN!
console.log("password:", password);      // WARN!

// 안전
console.log("user logged in:", userId);  // OK
```

&nbsp;

## 3-3. SQL Raw Query 작성 시 경고

&nbsp;

```json
{
  "hooks": {
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if grep -nE \"(query|execute)\\s*\\(\\s*[\\x27\\\"`]\\s*(SELECT|INSERT|UPDATE|DELETE)\" \"$FILE_PATH\" 2>/dev/null; then echo \"WARN: Raw SQL 쿼리가 감지되었습니다. ORM/QueryBuilder 사용을 권장합니다.\"; fi'"
      }
    ]
  }
}
```

&nbsp;

```typescript
// 경고 발생
const users = await db.query(`SELECT * FROM users WHERE id = '${id}'`);  // WARN!

// 안전
const users = await userRepository.findOne({ where: { id } });           // OK
```

&nbsp;

## 3-4. 종합 보안 Hook 설정

&nbsp;

실무에서 사용할 수 있는 종합 설정이다.

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "bash",
        "command": "bash -c 'INPUT=\"$TOOL_INPUT\"; if echo \"$INPUT\" | grep -qE \"rm\\s+-rf\"; then echo \"BLOCK: rm -rf 차단\"; exit 1; fi; if echo \"$INPUT\" | grep -qE \"git\\s+push\\s+--force\"; then echo \"BLOCK: force push 차단\"; exit 1; fi; if echo \"$INPUT\" | grep -qE \"git\\s+push.*(main|master)\"; then echo \"BLOCK: main 직접 push 차단\"; exit 1; fi; if echo \"$INPUT\" | grep -qE \"DROP\\s+TABLE|DROP\\s+DATABASE\"; then echo \"BLOCK: DROP 명령 차단\"; exit 1; fi'"
      },
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$FILE_PATH\" | grep -qE \"\\.env|\\.pem|\\.key|credentials\"; then echo \"BLOCK: 시크릿 파일 수정 차단\"; exit 1; fi'"
      }
    ],
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'npx prettier --write \"$FILE_PATH\" 2>/dev/null; if grep -nE \"(password|secret|token|api_key)\\s*[=:]\\s*[\\x27\\\"][^\\x27\\\"]{8,}\" \"$FILE_PATH\" 2>/dev/null; then echo \"WARN: 하드코딩된 시크릿 감지\"; fi'"
      }
    ]
  }
}
```

&nbsp;

&nbsp;

---

&nbsp;

# 4. 권한 모드

&nbsp;

Hooks가 "행동 단위"로 제어한다면, 권한 모드는 **"카테고리 단위"**로 제어한다.

&nbsp;

## 4-1. 3단계 권한

&nbsp;

```
┌───────────────────────────────────────────────┐
│               권한 모드                         │
├───────────────┬───────────────────────────────┤
│  자동 허용     │ 읽기, 검색, 코드 분석           │
│  (Auto Allow) │ → 확인 없이 바로 실행            │
├───────────────┼───────────────────────────────┤
│  확인 필요     │ 파일 수정, bash 명령 실행        │
│  (Confirm)    │ → 사용자 승인 후 실행            │
├───────────────┼───────────────────────────────┤
│  차단          │ 삭제, force push, 시크릿 접근    │
│  (Block)      │ → 어떤 경우에도 실행 불가         │
└───────────────┴───────────────────────────────┘
```

&nbsp;

```json
{
  "permissions": {
    "autoAllow": [
      "read:**",
      "search:**",
      "bash:git status",
      "bash:git log *",
      "bash:git diff *",
      "bash:npm run lint",
      "bash:npm run test"
    ],
    "requireConfirm": [
      "edit:**",
      "bash:npm install *",
      "bash:git commit *",
      "bash:git push *"
    ],
    "block": [
      "bash:rm -rf *",
      "bash:git push --force *",
      "bash:git reset --hard *",
      "edit:.env*",
      "edit:**/credentials*",
      "edit:**/*.pem",
      "edit:**/*.key"
    ]
  }
}
```

&nbsp;

## 4-2. 역할별 권한 차등

&nbsp;

```json
{
  "roles": {
    "junior": {
      "autoAllow": ["read:**", "search:**"],
      "requireConfirm": ["edit:src/**", "bash:npm run *"],
      "block": ["edit:config/**", "bash:git push *", "bash:npm install *"]
    },
    "senior": {
      "autoAllow": ["read:**", "search:**", "edit:src/**"],
      "requireConfirm": ["bash:git push *", "edit:config/**"],
      "block": ["bash:git push --force *", "edit:.env.prod*"]
    },
    "lead": {
      "autoAllow": ["read:**", "search:**", "edit:**"],
      "requireConfirm": ["bash:git push --force *"],
      "block": ["edit:.env.prod*"]
    }
  }
}
```

&nbsp;

같은 AI 에이전트를 써도, 주니어는 설정 파일을 수정할 수 없고, 시니어는 프로덕션 환경 파일을 수정할 수 없다.

&nbsp;

&nbsp;

---

&nbsp;

# 5. 기업별 보안 정책 예시

&nbsp;

## 5-1. 금융 / 핀테크

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$FILE_PATH\" | grep -qE \"(payment|transaction|transfer|settlement)\"; then echo \"AUDIT: 금융 코드 수정 시작 - $(date +%Y-%m-%dT%H:%M:%S) - $FILE_PATH\" >> /var/log/ai-audit.log; echo \"WARN: 금융 관련 코드 수정. 감사 로그에 기록됩니다.\"; fi'"
      }
    ],
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'echo \"AUDIT: 파일 수정 완료 - $(date +%Y-%m-%dT%H:%M:%S) - $FILE_PATH\" >> /var/log/ai-audit.log'"
      }
    ]
  },
  "permissions": {
    "block": [
      "edit:**/transaction*",
      "edit:**/settlement*",
      "bash:*prod*database*"
    ]
  }
}
```

&nbsp;

핵심: **모든 코드 변경이 감사 로그에 기록된다.**

&nbsp;

## 5-2. 의료 / 헬스케어

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$FILE_PATH\" | grep -qE \"(patient|medical|health|diagnosis|prescription)\"; then echo \"BLOCK: 환자 데이터 관련 코드는 AI로 수정할 수 없습니다. 2인 승인 후 수동 수정하세요.\"; exit 1; fi'"
      }
    ]
  }
}
```

&nbsp;

핵심: **환자 데이터 관련 코드는 AI 수정 자체를 차단.** HIPAA 규정 준수.

&nbsp;

## 5-3. 게임

&nbsp;

```json
{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$FILE_PATH\" | grep -qE \"(gacha|probability|lootbox|iap|in.app.purchase)\"; then echo \"WARN: 확률/과금 관련 코드 수정. QA 팀 리뷰 필수.\"; fi'"
      }
    ],
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$FILE_PATH\" | grep -qE \"\\.(png|jpg|webp|ogg|mp3)$\"; then SIZE=$(stat -f%z \"$FILE_PATH\" 2>/dev/null || stat -c%s \"$FILE_PATH\" 2>/dev/null); if [ \"$SIZE\" -gt 524288 ]; then echo \"WARN: 에셋 파일이 512KB를 초과합니다 (${SIZE}bytes). 최적화하세요.\"; fi; fi'"
      }
    ]
  }
}
```

&nbsp;

핵심: **인앱결제/확률 코드 수정 시 강제 리뷰, 에셋 크기 자동 검증.**

&nbsp;

## 5-4. SaaS

&nbsp;

```json
{
  "hooks": {
    "postToolUse": [
      {
        "matcher": "edit",
        "command": "bash -c 'if echo \"$FILE_PATH\" | grep -qE \"\\.(ts|tsx)$\"; then if grep -nE \"(findMany|findAll|query|select)\" \"$FILE_PATH\" | grep -v \"tenantId\" | grep -v \"tenant_id\" | head -5; then echo \"WARN: DB 쿼리에 tenantId 조건이 누락된 것 같습니다. 테넌트 격리를 확인하세요.\"; fi; fi'"
      }
    ]
  }
}
```

&nbsp;

핵심: **DB 쿼리에 tenantId가 빠지면 자동 경고.** 멀티테넌트 데이터 유출 방지.

&nbsp;

&nbsp;

---

&nbsp;

# 6. Hooks vs 규칙 파일 — 언제 뭘 쓸까

&nbsp;

| 상황 | 규칙 파일 (권고) | Hooks (강제) |
|------|-----------------|-------------|
| 코딩 스타일 | O | - |
| 네이밍 컨벤션 | O | - |
| 커밋 메시지 형식 | O | - |
| 포맷팅 (Prettier) | - | O (PostToolUse) |
| 린트 (ESLint) | - | O (PostToolUse) |
| 시크릿 하드코딩 방지 | O (규칙에도 명시) | O (PostToolUse로 감지) |
| 위험 명령 차단 | O (규칙에도 명시) | O (PreToolUse로 차단) |
| 특정 파일 보호 | - | O (PreToolUse로 차단) |
| main 직접 push 방지 | O (규칙에도 명시) | O (PreToolUse로 차단) |

&nbsp;

**원칙: 보안과 관련된 것은 반드시 Hooks로. 스타일과 관련된 것은 규칙 파일로.**

&nbsp;

규칙 파일에 "하지 마라"를 적어도, AI가 긴 대화 끝에 잊을 수 있다.

Hooks는 잊을 수 없다. **물리적으로 실행되니까.**

&nbsp;

&nbsp;

---

&nbsp;

# 7. Hook 설정 시 주의사항

&nbsp;

## 7-1. 너무 많으면 느려진다

&nbsp;

```
Hook 1개: 파일 수정 후 0.1초 추가
Hook 5개: 파일 수정 후 0.5초 추가
Hook 20개: 파일 수정 후 2초 추가 → 체감 가능한 지연
```

&nbsp;

**필수적인 것만 Hook으로, 나머지는 규칙 파일로.**

&nbsp;

## 7-2. 차단(BLOCK)은 신중하게

&nbsp;

```json
// 너무 넓은 차단 (비추)
{ "block": ["bash:*"] }  // 모든 bash 명령 차단 → 아무것도 못 함

// 적절한 차단 (추천)
{ "block": ["bash:rm -rf *", "bash:git push --force *"] }  // 위험한 것만 차단
```

&nbsp;

## 7-3. 경고(WARN)와 차단(BLOCK) 구분

&nbsp;

```
WARN: "주의하세요" → 실행은 됨 (사용자가 판단)
BLOCK: "안 됩니다" → 실행 자체가 불가 (exit 1)

보안 필수 사항 → BLOCK
주의 사항 → WARN
```

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[AI 하네스 5편] 팀에 하네스 도입하기 — 설계 체크리스트와 단계별 가이드**

&nbsp;

규칙 파일, 메모리, Skills, Hooks를 배웠다.

이제 **팀 전체에 어떻게 도입하는가?**

도입 전 체크리스트, 주차별 가이드, 실패하는 도입과 성공하는 도입의 차이를 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

AI, AI에이전트, AI하네스, Hooks, 보안, DevSecOps, AI거버넌스, 기업AI, 코드안전
