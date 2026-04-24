# [Doc-to-Code 2편] LLM 에이전트의 코드 번역기 — Context-Aware 프롬프팅과 다중 파일 생성

&nbsp;

이전 편에서 우리는 자연어로 작성된 블로그 포스트를 기계가 100% 신뢰할 수 있는 구조화된 객체(AST 명세서)로 변환했다. 이제 남은 과제는 이 명세서를 실제 동작하는 소스 코드로 번역해 내는 것이다. 

&nbsp;

문제는 최신 프레임워크(Next.js App Router나 Spring Boot)의 코드는 단일 파일로 동작하지 않는다는 점이다. 기능 하나를 추가하려면 라우팅 파일, 서비스 로직, DB 리포지토리, 그리고 DTO 모델 등 최소 3~4개의 파일이 유기적으로 생성되고 연결되어야 한다. 단순히 "이 기능 짜줘"라고 던지는 제로샷(Zero-shot) 프롬프트로는 절대 불가능한 작업이다. 본 글에서는 에이전트가 기존 코드베이스의 아키텍처를 이해(Context-Aware)하고, 엄격한 패턴에 맞추어 **다중 파일(Multi-file)을 일관성 있게 생성해 내는 Function Calling 기반의 번역 파이프라인**을 설계한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 컨텍스트 붕괴 방어: RAG를 통한 기존 아키텍처 주입

&nbsp;

LLM이 아무리 똑똑해도 우리 회사의 사내 프로젝트 컨벤션을 알 수는 없다. 새로운 `PaymentService`를 짤 때 기존의 `DatabaseManager`나 `AuthGuard`를 상속받지 않고 독자적인 코드를 짜버리면, 그 코드는 컴파일조차 되지 않는 쓰레기다.

&nbsp;

## 1-1. Skeleton Tree 추출
코드 생성 전, 에이전트는 타겟 레포지토리의 전체 코드를 읽지 않는다(토큰 낭비 및 노이즈 발생). 대신 로컬 환경에서 AST 정적 분석기를 돌려 핵심 인터페이스와 클래스의 시그니처(Signature)만 추출한 '스켈레톤 트리(Skeleton Tree)'를 생성한다.

&nbsp;

```typescript
// 에이전트에게 주입되는 압축된 컨텍스트 예시 (TypeScript)
/* 
[Existing Context]
interface BaseRepository<T> {
  save(entity: T): Promise<T>;
  findById(id: string): Promise<T | null>;
}
class AuthMiddleware implements NestMiddleware { ... }
*/
```

&nbsp;

## 1-2. RAG (Retrieval-Augmented Generation) 결합
블로그 명세서에서 `포인트 결제`라는 키워드가 등장하면, 에이전트의 내부 RAG 엔진이 기존 코드베이스에서 `Point`, `Payment`, `Transaction`과 관련된 인터페이스 선언부만 Vector DB에서 검색하여 프롬프트 상단에 주입한다. 이 과정을 통해 에이전트는 "새로 생성할 코드는 반드시 기존의 `BaseRepository`를 상속받아야 한다"는 강제적인 문맥(Context)을 획득하게 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 다중 파일 생성을 위한 Function Calling (Tool Use) 아키텍처

&nbsp;

마크다운 코드 블록으로 여러 개의 코드를 내뱉게 하면, 파일 경로 파싱이 꼬이고 에러가 속출한다. 에이전트에게 코드를 '텍스트'로 뱉게 하지 말고, 시스템이 제공하는 **`create_file` 함수를 호출(Function Calling)**하게 만들어야 한다.

&nbsp;

## 2-1. Tool Schema 정의
LLM에게 오직 아래의 JSON 스키마 규격으로만 응답하도록 도구를 강제한다.

&nbsp;

```json
{
  "name": "write_multiple_files",
  "description": "생성된 코드 조각들을 적절한 디렉토리와 파일명으로 저장합니다.",
  "parameters": {
    "type": "object",
    "properties": {
      "files": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "path": {
              "type": "string",
              "description": "예: src/modules/payment/payment.service.ts"
            },
            "content": {
              "type": "string",
              "description": "생성된 소스 코드 전문"
            },
            "layer": {
              "type": "string",
              "enum": ["controller", "service", "repository", "dto"]
            }
          },
          "required": ["path", "content", "layer"]
        }
      }
    },
    "required": ["files"]
  }
}
```

&nbsp;

## 2-2. 구조적 렌더링 (Structural Rendering)
블로그 명세서가 위 파이프라인을 통과하면, 에이전트는 다음과 같은 사고 과정(Chain of Thought)을 거쳐 함수를 호출한다.
1. **Thought**: 요구사항을 보니 결제 API다. NestJS 컨벤션에 따라 3개의 레이어로 나눠야 한다.
2. **Thought**: 기존에 존재하는 `JwtAuthGuard`를 Controller에 적용해야겠다.
3. **Action**: `write_multiple_files` 호출
   - 파일 1: `payment.controller.ts`
   - 파일 2: `payment.service.ts`
   - 파일 3: `create-payment.dto.ts`

&nbsp;

에이전트는 텍스트를 중구난방으로 내뱉는 대신, 완벽하게 분리된 파일 경로와 내용을 구조화된 JSON 배열로 반환한다. 호스트 시스템은 이 JSON을 파싱하여 실제 디렉토리에 물리적 파일로 Write(I/O)하기만 하면 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 종속성 충돌(Dependency Hell) 방어 로직

&nbsp;

에이전트가 완벽한 로직을 짰더라도, A 파일이 B 파일을 Import하는 경로가 틀리면 컴파일은 실패한다. 특히 새로 생성된 3개의 파일이 서로를 순환 참조(Circular Dependency)하는 끔찍한 사태를 막아야 한다.

&nbsp;

- **위상 정렬 (Topological Sort) 생성**: 에이전트의 프롬프트에 "코드를 생성할 때 의존성이 없는 DTO부터 생성하고, 그다음 Repository, Service, Controller 순서로 작성하라"는 절차적 지시문을 하드코딩한다.
- **Import Checker**: 에이전트가 뱉어낸 `content` 내부의 `import` 구문을 정규식으로 스캔하여, 실제로 존재하는 파일 경로인지 시스템 단에서 검증한 뒤 파일 I/O를 수행한다. 경로가 틀렸다면 즉각 에이전트에게 "경로 오류: ../common/types 가 존재하지 않음. 수정 요망"이라는 에러 메시지와 함께 재귀(Recursive) 호출을 태운다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 결론: 에이전트는 타이피스트가 아니라 아키텍트다

&nbsp;

자연어 블로그 포스트를 코드로 바꾸는 과정의 핵심은 '단순 번역'이 아니다. 기존에 짜여진 거대한 성(코드베이스)의 규격을 파악하고, 그 성벽의 어느 위치에 어떤 모양의 벽돌을 끼워 넣어야 시스템이 무너지지 않을지 수학적으로 계산하는 아키텍처링의 영역이다.

&nbsp;

RAG 기반의 컨텍스트 주입과 Function Calling을 통한 엄격한 파일 제어 파이프라인은, LLM이 가진 환각의 폭주를 막고 프로덕션 레벨의 정교한 폴더 스트럭처를 빚어내는 완벽한 구속복(Straitjacket)이자 도구가 된다. 

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[Doc-to-Code 3편] 안전한 코드 생성을 위한 샌드박스 검증 — CI/CD 기반의 환각(Hallucination) 방어**

&nbsp;

에이전트가 멋진 파일들을 만들어 냈지만, 이 코드가 진짜로 버그 없이 돌아가는지 우리는 절대 믿을 수 없다. AI가 작성한 코드를 라이브 환경으로 올리기 전, 일회용(Ephemeral) Docker 컨테이너 샌드박스를 띄우고 그 안에서 Jest/Vitest 테스트 코드를 자동 생성 및 실행하는 과정. 그리고 테스트 실패 시 에러 로그를 읽고 에이전트가 코드를 스스로 고치는(Self-Correction) 소름 돋는 무결점 검증 파이프라인을 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

DocToCode, AI개발, FunctionCalling, RAG아키텍처, 코드자동생성, LLMOps, 마이크로서비스설계, 컨텍스트최적화
