# [Doc-to-Code 1편] 블로그 글이 명세서가 되다 — Markdown 기반의 Executable Spec 아키텍처

&nbsp;

개발 조직의 영원한 숙제는 '기획서와 실제 코드의 동기화'다. 기획자가 작성한 Jira 티켓이나 Confluence 문서는 코드가 수정됨과 동시에 레거시가 되며, 반대로 개발자가 작성한 기술 블로그나 마크다운 문서는 실행 불가능한 정적 텍스트에 머무른다. 

&nbsp;

만약 우리가 쓴 기술 블로그 포스트 자체가 시스템을 구동시키는 '실행 가능한 명세서(Executable Specification)'가 된다면 어떨까? 글을 쓰는 행위가 곧 기획이자 코딩의 시작점이 되고, AI 에이전트가 이 글을 파싱하여 GitHub에 실제 동작하는 코드로 번역해 내는 **Doc-to-Code** 아키텍처. 본 글에서는 자연어로 작성된 마크다운(Markdown) 문서를 기계가 오해 없이 100% 이해할 수 있는 엄격한 데이터 구조(AST)로 변환하는 첫 번째 파이프라인을 심층 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. Markdown의 구조적 한계와 AST(Abstract Syntax Tree) 파싱

&nbsp;

LLM에게 블로그 글 전체를 통째로 던지며 "이대로 코드 짜줘"라고 명령하는 것은, 주니어 개발자에게 수백 페이지짜리 기획서를 던지고 알아서 풀스택을 만들라는 것과 같은 폭력이다. 에이전트는 환각(Hallucination)에 빠지고 파일 간의 의존성은 박살 난다.

&nbsp;

텍스트를 의미론적 덩어리로 쪼개기 위해, 마크다운 텍스트를 기계가 탐색 가능한 **AST(추상 구문 트리)**로 파싱해야 한다. Node.js 생태계의 `remark`와 `unist` 인터페이스가 이 작업의 핵심 엔진이 된다.

&nbsp;

## 1-1. Remark 기반의 커스텀 파서 구현
블로그 글에는 서론, 비유, 결론 같은 비즈니스 로직과 무관한 텍스트가 섞여 있다. 에이전트에게 필요한 것은 오직 API 명세, 데이터 스키마, 사용자 스토리(User Story)뿐이다.

&nbsp;

```javascript
import { unified } from 'unified';
import remarkParse from 'remark-parse';
import { visit } from 'unist-util-visit';

// 마크다운 문서를 AST로 변환 후 핵심 노드만 추출하는 엔진
export function extractExecutableSpecs(markdownText) {
  const tree = unified().use(remarkParse).parse(markdownText);
  const specs = {
    schemas: [],
    endpoints: [],
    acceptanceCriteria: []
  };

  visit(tree, 'code', (node) => {
    // 1. Prisma나 SQL 스키마 블록 추출
    if (node.lang === 'prisma' || node.lang === 'sql') {
      specs.schemas.push(node.value);
    }
  });

  visit(tree, 'list', (node) => {
    // 2. 체크리스트(Acceptance Criteria) 추출
    node.children.forEach(listItem => {
      if (typeof listItem.checked === 'boolean') {
        const text = listItem.children.map(c => c.value).join('');
        specs.acceptanceCriteria.push({ text, checked: listItem.checked });
      }
    });
  });

  return specs;
}
```

&nbsp;

이 정적 분석 단계를 거치면, 감성적인 블로그 글은 철저하게 분해되어 에이전트가 소비할 수 있는 정형화된 JSON 객체(Schema, Criteria)로 압축된다. LLM의 컨텍스트 윈도우를 아끼면서도, 명령의 모호성을 수학적으로 제거하는 첫 번째 관문이다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. Executable Spec의 작성 규칙 (Schema Validation)

&nbsp;

AI 에이전트가 코드를 완벽하게 제너레이션하기 위해서는, 사람이 글을 쓸 때 지켜야 할 최소한의 '프로토콜'이 필요하다. 우리는 프론트마터(Frontmatter)와 특정 헤딩(Heading) 컨벤션을 통해 이 프로토콜을 강제한다.

&nbsp;

## 2-1. Frontmatter를 통한 의도 명시
문서 최상단에 YAML 포맷의 프론트마터를 삽입하여 에이전트에게 이 문서가 타겟팅하는 프레임워크와 마이크로서비스 경로를 명시한다.

&nbsp;

```yaml
---
type: feature_spec
target_repo: paradisecity-backend
framework: nestjs, typeorm
epic: "사용자 포인트 적립 시스템"
dependencies:
  - "@nestjs/common"
  - "typeorm"
---
```

&nbsp;

이 메타데이터는 이후 에이전트가 GitHub Repository를 클론(Clone)하고 어떤 디렉토리 구조에 파일을 생성해야 할지 결정하는 '라우팅 기준표'가 된다.

&nbsp;

## 2-2. Zod를 활용한 프롬프트 주입 검증
에이전트가 마크다운에서 추출한 요구사항이 실제로 코드를 짜기에 충분한 구체성을 띠고 있는지, 역으로 시스템이 검증해야 한다. 추출된 `specs` 객체를 `zod` 스키마 밸리데이터에 통과시킨다.

&nbsp;

```typescript
import { z } from 'zod';

const SpecSchema = z.object({
  schemas: z.array(z.string()).min(1, "데이터베이스 스키마 정의가 누락되었습니다."),
  endpoints: z.array(z.object({
    method: z.enum(["GET", "POST", "PUT", "DELETE"]),
    path: z.string(),
    reqBody: z.string().optional()
  })),
  acceptanceCriteria: z.array(z.object({
    text: z.string(),
    checked: z.boolean()
  })).min(3, "최소 3개 이상의 테스트 통과 기준(Acceptance Criteria)이 필요합니다.")
});
```

&nbsp;

만약 블로그 글에 "API를 만들어주세요"라고만 적혀 있고 구체적인 엔드포인트나 최소 3개의 체크리스트가 없다면, 파이프라인은 코드를 생성하기도 전에 `ValidationError`를 발생시키며 사용자에게 "명세서의 구체성이 부족합니다"라는 피드백을 던진다. 이 강력한 가드레일이 쓰레기 코드 생성을 원천 차단한다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 맥락의 보존: LLM에 주입하기 전의 정제 과정

&nbsp;

파싱된 데이터는 곧바로 코드 생성 LLM으로 가지 않는다. "포인트 적립 로직을 짜줘"라는 명령을 수행하려면, 시스템에 이미 존재하는 `User` 엔티티나 `DatabaseConnection` 클래스의 맥락(Context)을 알아야 하기 때문이다.

&nbsp;

에이전트는 프론트마터에 명시된 `target_repo`를 바탕으로 GitHub API나 로컬 클론된 디렉토리의 핵심 파일들(예: `src/entities/User.ts`)의 인터페이스만을 AST 파서를 통해 가볍게 추출한다. 
그리고 우리가 블로그 글에서 파싱해 낸 `Executable Specs`와 기존 코드베이스의 `Interface Context`를 하나로 병합하여 최종적인 **'Generation Prompt Payload'**를 조립한다.

&nbsp;

이 과정은 마치 시니어 개발자가 주니어 개발자에게 기획서(블로그 글)를 건네면서, "우리 사내의 기존 User 모델은 이거니까 이걸 상속받아서 구현해"라고 가이드라인(Context)을 함께 건네주는 것과 완벽히 동일한 엔지니어링 프로세스다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 문서는 곧 코드가 되어야 한다

&nbsp;

'문서화'라는 행위는 개발 후반부에 억지로 해치우는 부채(Debt)로 여겨져 왔다. 하지만 Doc-to-Code 아키텍처 환경에서 블로그 포스트와 마크다운 문서는 애플리케이션의 동작을 규정하는 가장 강력한 '루트 컴파일러(Root Compiler)'로 격상된다.

&nbsp;

자연어를 텍스트로 두지 않고 AST로 구조화하여 기계의 언어로 번역하는 이 첫 번째 파이프라인 설계야말로, 인간의 '의도'와 기계의 '실행' 사이의 간극을 0으로 수렴하게 만드는 AI Driven Development의 완벽한 시작점이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[Doc-to-Code 2편] LLM 에이전트의 코드 번역기 — AST 파싱과 Context-Aware 프롬프팅**

&nbsp;

블로그 글에서 완벽하게 추출된 요구사항 명세서(AST)를 받아 든 LLM. 하지만 "코드를 짜줘"라는 단순한 프롬프트로는 하나의 파일에 모든 로직을 쑤셔 넣는 스파게티 코드가 탄생할 뿐이다. 어떻게 에이전트가 Controller, Service, Repository 패턴을 이해하고 여러 개의 파일로 완벽하게 쪼개진 디렉토리 구조를 설계할 수 있을까? RAG(Retrieval-Augmented Generation)를 통한 기존 코드베이스 컨텍스트 인젝션과, 다중 파일 생성을 위한 Function Calling(Tool Use)의 극한 설계 기법을 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

DocToCode, AI개발, ExecutableSpec, AST파싱, Markdown, LLMOps, 기획서자동화, 프론트엔드아키텍처
