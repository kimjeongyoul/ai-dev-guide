# [AI 에이전트 2편] MCP로 AI 에이전트 만들기 — 실전 자동화 파이프라인

&nbsp;

개발자의 하루:

1. Jira 이슈 확인
2. GitHub에서 코드 확인
3. 수정
4. PR 생성
5. Slack에 알림
6. 다음 이슈로

&nbsp;

이 중 **2~5번을 AI가 대신할 수 있다면?**

&nbsp;

MCP로 외부 도구를 연결하면 가능하다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 개발자가 매일 하는 반복 작업

&nbsp;

```
아침: 이슈 확인 (Jira/GitHub)
오전: 코드 수정 + 테스트
오후: PR 생성 + 리뷰 요청
저녁: 배포 + 모니터링 확인

반복되는 패턴:
- 이슈 읽기 → 관련 코드 찾기 → 수정 → 테스트 → PR
- 에러 로그 확인 → 원인 추적 → 수정
- DB 스키마 보고 → API 만들기
```

&nbsp;

이 패턴들은 **도구만 연결되면 AI가 자동으로 수행**할 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. MCP로 연결할 수 있는 도구들

&nbsp;

| 도구 | 용도 | AI가 하는 일 |
|:---|:---|:---|
| **GitHub** | 코드, 이슈, PR | 이슈 읽기, PR 생성, 코드 검색 |
| **Jira** | 태스크 관리 | 이슈 등록, 상태 업데이트 |
| **Slack** | 커뮤니케이션 | 알림 전송, 채널 모니터링 |
| **PostgreSQL** | 데이터베이스 | 스키마 조회, 쿼리 실행 |
| **Sentry** | 에러 모니터링 | 에러 조회, 스택트레이스 분석 |
| **Figma** | 디자인 | 컴포넌트 스펙 조회 |

&nbsp;

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "ghp_..." }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "postgresql://..." }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": { "SLACK_TOKEN": "xoxb-..." }
    }
  }
}
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. 파이프라인 1 — 이슈 기반 개발

&nbsp;

```
"GitHub 이슈 #42 수정해서 PR 올려줘"
```

&nbsp;

AI 내부:

```
1. GitHub MCP → 이슈 #42 읽기
   "로그인 시 비밀번호 검증이 대소문자를 구분하지 않음"

2. 코드 검색 → src/auth/login.ts 찾기

3. 코드 분석 → password.toLowerCase() 발견

4. 수정 → toLowerCase() 제거

5. 테스트 실행 → 통과

6. GitHub MCP → PR 생성
   제목: "fix: 비밀번호 검증 대소문자 구분 추가"
   본문: 이슈 #42에서 보고된 문제 수정

7. Slack MCP → #dev 채널에 알림
   "이슈 #42 수정 PR이 생성됐습니다. 리뷰 부탁드립니다."
```

&nbsp;

**사람이 할 일: 최종 리뷰 + 머지.**

&nbsp;

&nbsp;

---

&nbsp;

# 4. 파이프라인 2 — 에러 자동 수정

&nbsp;

```
"Sentry에서 가장 많이 발생하는 에러 수정해줘"
```

&nbsp;

```
1. Sentry MCP → 최근 24시간 에러 조회
   "TypeError: Cannot read property 'name' of null" (532건)

2. 스택트레이스 분석 → src/api/users.ts:42

3. 코드 읽기 → user.name에서 user가 null일 수 있음

4. 수정 → optional chaining 적용 (user?.name)

5. 테스트 작성 + 실행 → 통과

6. PR 생성 + Jira 태스크 등록

7. Slack 알림
```

&nbsp;

```typescript
// 수정 전
const displayName = user.name;  // user가 null이면 에러

// 수정 후
const displayName = user?.name || 'Unknown';
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. 파이프라인 3 — DB 기반 API 생성

&nbsp;

```
"users 테이블 기준으로 CRUD API 만들어줘"
```

&nbsp;

```
1. DB MCP → users 테이블 스키마 조회
   id SERIAL, name VARCHAR(100), email VARCHAR(200), 
   created_at TIMESTAMP, active BOOLEAN

2. 엔티티 생성 → src/entities/User.ts

3. 서비스 생성 → src/services/UserService.ts
   findAll, findById, create, update, delete

4. 라우트 생성 → src/routes/users.ts
   GET /users, GET /users/:id, POST /users, 
   PUT /users/:id, DELETE /users/:id

5. 테스트 생성 → tests/users.test.ts

6. 테스트 실행 → 통과
```

&nbsp;

AI가 스키마를 **직접 보고** 만드니까, 필드명이나 타입이 틀릴 일이 없다.

&nbsp;

&nbsp;

---

&nbsp;

# 6. 커스텀 MCP 서버 만들기

&nbsp;

공식 MCP 서버에 없는 사내 도구를 연결하려면 직접 만든다.

&nbsp;

```typescript
// 사내 API를 MCP 서버로 연결
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { z } from 'zod';

const server = new McpServer({ name: 'internal-api', version: '1.0.0' });

// Tool 정의: 회원 조회
server.tool(
  'get_member',
  '회원 정보를 조회합니다',
  { memberId: z.string().describe('회원 ID') },
  async ({ memberId }) => {
    const res = await fetch(`https://internal-api.com/members/${memberId}`);
    const data = await res.json();
    return { content: [{ type: 'text', text: JSON.stringify(data, null, 2) }] };
  }
);

// Tool 정의: 주문 목록
server.tool(
  'list_orders',
  '최근 주문 목록을 조회합니다',
  { 
    status: z.enum(['pending', 'completed', 'cancelled']).optional(),
    limit: z.number().default(10)
  },
  async ({ status, limit }) => {
    const params = new URLSearchParams();
    if (status) params.set('status', status);
    params.set('limit', String(limit));
    
    const res = await fetch(`https://internal-api.com/orders?${params}`);
    const data = await res.json();
    return { content: [{ type: 'text', text: JSON.stringify(data, null, 2) }] };
  }
);

// 서버 시작
const transport = new StdioServerTransport();
await server.connect(transport);
```

&nbsp;

```json
// settings.json에 등록
{
  "mcpServers": {
    "internal": {
      "command": "node",
      "args": ["./mcp-servers/internal-api.js"],
      "env": { "API_TOKEN": "..." }
    }
  }
}
```

&nbsp;

이제 AI에게 "회원 cc00032의 주문 내역 확인해줘"라고 하면 사내 API를 직접 호출한다.

&nbsp;

&nbsp;

---

&nbsp;

# 7. 보안 고려사항

&nbsp;

| 원칙 | 이유 |
|:---|:---|
| **읽기 전용으로 시작** | AI가 실수로 데이터를 삭제하는 걸 방지 |
| **토큰 권한 최소화** | GitHub 토큰은 repo 읽기만, 전체 admin 권한 주지 않기 |
| **위험 작업은 사람 승인** | PR 머지, 배포, DB 쓰기는 AI가 제안하고 사람이 실행 |
| **운영 DB 직접 연결 금지** | 읽기 전용 레플리카만 연결 |
| **토큰을 코드에 넣지 않기** | 환경 변수로 관리 |

&nbsp;

```
안전한 순서:
1단계: 읽기 전용 도구만 연결 (GitHub 이슈 읽기, DB 스키마 조회)
2단계: 로컬 파일 수정 허용 (코드 편집, 테스트 실행)
3단계: 외부 쓰기 허용 (PR 생성, Jira 등록) — 신중하게
```

&nbsp;

&nbsp;

---

&nbsp;

# 결론

&nbsp;

MCP는 AI에게 **눈과 손**을 준다.

&nbsp;

- 눈: GitHub 이슈, DB 스키마, Sentry 에러를 직접 본다
- 손: PR 생성, Jira 등록, Slack 알림을 직접 한다

&nbsp;

사람은 **판단과 승인**에 집중하면 된다.

&nbsp;

다음 편에서는 **단일 에이전트의 한계를 넘어서는 멀티 에이전트 설계 패턴**을 다룬다.

&nbsp;

&nbsp;

---

MCP, AI에이전트, 자동화, GitHub, Jira, Slack, Sentry, DB연동, 파이프라인, 개발생산성, 커스텀MCP, AI코딩, 바이브코딩
