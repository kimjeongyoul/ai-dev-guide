# [Doc-to-Code 5편] 실시간 피드백 루프 — PR 코멘트를 통한 에이전트와의 페어 프로그래밍

&nbsp;

AI 에이전트가 블로그 글을 파싱하여 코드를 짜고(1~2편), 샌드박스에서 에러를 잡아내며(3편), GitHub에 완벽한 명세가 담긴 PR(Pull Request)을 자동으로 올리는 데(4편)까지 성공했다. 하지만 소프트웨어 개발은 '원샷(One-shot)'으로 끝나는 법이 없다. 시니어 엔지니어의 아키텍처 리뷰, 기획 변경에 따른 로직 수정 등 필연적으로 인간의 개입과 수정 요구가 뒤따른다.

&nbsp;

에이전트가 올린 PR에 인간이 코드를 직접 수정해서 푸시한다면, 그것은 반쪽짜리 자동화다. 진정한 Agentic Workflow는 인간이 PR 코멘트(Review Comment)를 남기면, 에이전트가 그 피드백을 실시간으로 수용하여 스스로 코드를 고치고 추가 커밋을 밀어 넣는 **쌍방향 피드백 루프(Bidirectional Feedback Loop)**에 있다. 본 글에서는 GitHub Webhook과 LangGraph 상태 관리 엔진을 결합하여, 인간과 AI가 GitHub PR을 무대로 페어 프로그래밍(Pair Programming)을 수행하는 최상위 아키텍처를 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 아키텍처 설계: Webhook 기반의 Event-Driven 트리거

&nbsp;

에이전트가 GitHub의 변화를 실시간으로 감지하려면 Polling(계속 찔러보기) 방식이 아닌 Webhook 이벤트 수신 아키텍처를 채택해야 한다.

&nbsp;

## 1-1. Webhook Payload 파싱
GitHub 레포지토리에 Webhook을 설정하여 `issue_comment` 및 `pull_request_review_comment` 이벤트를 우리 백엔드 서버(또는 Lambda)로 라우팅한다. 
리뷰어가 코드 특정 라인에 "이 부분은 Redis 캐시를 타게 수정해 줘"라고 코멘트를 남기는 순간, 다음과 같은 방대한 컨텍스트가 페이로드로 날아온다.

&nbsp;

```json
{
  "action": "created",
  "comment": {
    "body": "이 부분은 Redis 캐시를 타게 수정해 줘",
    "path": "src/payment/payment.service.ts",
    "line": 42,
    "diff_hunk": "@@ -38,7 +38,7 @@\n async getPoint(id) {\n-   return await this.repo.findById(id);\n+   // TODO: add cache logic"
  },
  "pull_request": {
    "number": 105,
    "head": { "ref": "feat/bot-add-point-system" }
  }
}
```

&nbsp;

이 페이로드를 통해 에이전트는 **1) 어떤 파일의, 2) 몇 번째 라인의, 3) 어떤 코드 컨텍스트(diff_hunk)에 대해, 4) 사용자가 무엇을 요구했는지**를 정확히 타겟팅할 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 에이전트 상태(State)의 복원과 컨텍스트 병합

&nbsp;

코멘트를 받은 LLM이 뜬금없는 코드를 짜지 않게 하려면, 해당 PR을 처음 생성했을 때의 상태(AST 명세서, 파일 트리 등)를 복원해야 한다.

&nbsp;

## 2-1. Thread Context 복구 (LangGraph Checkpointer)
1~3편에서 구축했던 LangGraph의 `Checkpointer(MemorySaver)`를 활용한다. 에이전트가 PR을 열 때 상태 객체(State)를 `thread_id = PR_NUMBER_105`로 Redis에 저장해 두었다고 가정하자. 
Webhook이 들어오면 서버는 `thread_id: 105`의 상태를 로드하여, 에이전트의 뇌(메모리)를 "내가 이 코드를 짤 때의 맥락"으로 순식간에 동기화(Hydration)시킨다.

&nbsp;

## 2-2. Patch Generation Prompting
복구된 맥락과 리뷰어의 코멘트를 결합하여 LLM에게 수정된 코드를 요구한다. 전체 코드를 다시 내뱉게 하는 것은 토큰 낭비이자 포맷 파괴의 주범이므로, Diff Patch 형태나 함수 단위의 교체만 요구해야 한다.

&nbsp;

```python
prompt = f"""
당신은 방금 생성한 PR 코드에 대해 시니어 리뷰어로부터 피드백을 받았습니다.

[수정 대상 파일]: {comment_path}
[리뷰어가 지적한 코드 부분]:
{diff_hunk}

[리뷰어의 피드백]: "{comment_body}"

이전의 아키텍처 규칙을 유지하면서, 리뷰어의 요구사항을 반영하여 해당 파일의 전체/수정된 코드를 작성하십시오.
반드시 기존 코드베이스의 RedisModule을 의존성 주입(Inject)하여 처리하십시오.
"""
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. 변경 사항의 재검증과 자동 Commit

&nbsp;

LLM이 코드를 수정했다고 해서 바로 GitHub에 푸시하는 것은 위험하다. 수정 과정에서 기존에 통과했던 다른 테스트가 깨질 수 있다(Regression).

&nbsp;

## 3-1. 샌드박스 재회귀 (Regression Test)
에이전트는 3편의 샌드박스 로직으로 돌아간다. 수정된 파일만 임시 컨텍스트에 덮어씌운 뒤, 전체 Acceptance Criteria 테스트 슈트(Test Suite)를 다시 실행한다.
- **Pass**: 리뷰어의 코멘트가 완벽히 적용되었고 기존 로직도 부서지지 않았음.
- **Fail**: 에이전트가 코드를 잘못 고쳐서 에러 발생. 에이전트는 다시 Self-Correction 루프를 돌며 스스로 문제를 해결한 뒤에야 다음 단계로 넘어간다.

&nbsp;

## 3-2. Octokit을 통한 추가 커밋과 코멘트 반영
테스트가 완벽하게 통과하면, 에이전트는 4편에서 구축한 Octokit 다중 커밋 트랜잭션을 활용하여 기존 PR 브랜치(`feat/bot-add-point-system`)에 새로운 커밋을 추가로 얹는다.
그리고 리뷰어의 원래 코멘트에 봇이 답글(Reply)을 단다.

&nbsp;

> 🤖 **@reviewer** 요청하신 Redis 캐시 로직을 반영하여 코드를 수정하고, 기존 테스트 케이스를 모두 통과시킨 후 커밋( `a1b2c3d` )을 추가했습니다. 확인 부탁드립니다!

&nbsp;

&nbsp;

---

&nbsp;

# 4. 결론: Doc-to-Code 파이프라인의 완성

&nbsp;

블로그 포스트라는 정적인 자연어 텍스트에서 출발하여, 
1. 기계가 이해하는 AST 명세서로 파싱하고,
2. 사내 코드베이스 컨텍스트(RAG)를 섞어 다중 파일을 생성해 내며,
3. 샌드박스에서 TDD 방식으로 테스트하고 스스로 버그를 고치며,
4. GitHub에 맥락이 담긴 PR을 올리고,
5. 인간 엔지니어의 코멘트를 받아 코드를 수정하는 데까지 이르렀다.

&nbsp;

이 5단계의 장대한 파이프라인은 단순히 "AI로 코딩 속도를 높인다"는 차원의 문제가 아니다. 기획(블로그 글), 구현, 테스트, 배포에 이르는 소프트웨어 생명주기(SDLC) 전체를 인간과 기계가 하나의 호흡으로 춤추게 만드는 **'인지적 자동화(Cognitive Automation)'의 정점**이다.

&nbsp;

당신이 쓴 그 블로그 글 한 편이, 내일 아침 프로덕션 서버를 돌리고 수만 명의 고객을 맞이하는 살아 숨 쉬는 시스템이 될 것이다. AI Driven Development의 진짜 혁명은 바로 지금, 이 글에서부터 시작된다.

&nbsp;

&nbsp;

---

&nbsp;

DocToCode, AI개발, 페어프로그래밍, GithubWebhook, PR자동화, LangGraph, LLMOps, 백엔드아키텍처
