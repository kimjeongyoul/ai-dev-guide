# [Doc-to-Code 3편] 안전한 코드 생성을 위한 샌드박스 검증 — CI/CD 기반의 환각(Hallucination) 방어

&nbsp;

AI 에이전트가 코드를 기가 막히게 잘 짰다고 가정해 보자. 디렉토리 구조도 완벽하고 로직도 그럴싸하다. 하지만 백엔드 엔지니어라면 이 코드를 곧바로 프로덕션 브랜치에 머지(Merge)하는 짓은 절대 하지 않는다. LLM은 문법적으로 완벽해 보이는 코드를 생성할지라도, 치명적인 런타임 에러(Runtime Error)나 비즈니스 로직 결함을 포함하는 환각(Hallucination)을 필연적으로 동반하기 때문이다.

&nbsp;

진정한 Doc-to-Code 파이프라인의 핵심은 '생성'이 아니라 **'검증(Validation)'**에 있다. 에이전트가 생성한 코드를 인간이 리뷰하기 전에, 기계가 먼저 격리된 환경(Sandbox)에서 컴파일하고 테스트 코드를 돌려 스스로 결함을 고치는(Self-Correction) 무결점 검증 아키텍처를 심층 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. Ephemeral Sandbox: 격리된 실행 환경 구축

&nbsp;

AI가 짜준 코드를 호스트 서버(에이전트가 도는 서버)에서 직접 `node index.js`로 실행하는 것은 보안상, 리소스상 완벽한 자살 행위다. 악의적인 프롬프트 인젝션이나 무한 루프 코드가 호스트를 파괴할 수 있다.

&nbsp;

우리는 매 코드 생성 사이클마다 1회용(Ephemeral) Docker 컨테이너를 띄워 코드를 실행하고 즉시 파기하는 샌드박스를 구축해야 한다.

&nbsp;

## 1-1. Docker API 연동 엔진
`dockerode` 라이브러리를 활용하여, Node.js 또는 Python 실행 환경이 세팅된 컨테이너를 백그라운드에서 실행하고, 에이전트가 짠 코드를 볼륨 마운트(Volume Mount)로 주입한다.

&nbsp;

```javascript
import Docker from 'dockerode';
const docker = new Docker();

export async function runInSandbox(codeDir, testCommand) {
  // 경량화된 Alpine 기반 컨테이너 생성
  const container = await docker.createContainer({
    Image: 'node:18-alpine',
    Cmd: ['sh', '-c', testCommand],
    HostConfig: {
      Binds: [`${codeDir}:/app`], // 생성된 코드가 담긴 폴더 마운트
      Memory: 512 * 1024 * 1024,  // 512MB 메모리 하드 리미트
      NetworkMode: 'none'         // 외부 네트워크 차단 (NPM Install은 사전에 완료된 베이스 이미지 사용)
    },
    WorkingDir: '/app'
  });

  await container.start();
  const stream = await container.logs({ stdout: true, stderr: true, follow: true });
  const result = await container.wait(); // Exit Code 획득
  await container.remove();

  return { exitCode: result.StatusCode, logs: stream.toString() };
}
```

&nbsp;

이 샌드박스 환경은 네트워크가 철저히 단절(Air-gapped)되어 있으며 메모리 제한이 걸려 있으므로, 어떤 악성 코드나 무한 루프가 돌더라도 5초 후 타임아웃 킬(Kill)을 통해 호스트를 안전하게 보호한다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. Acceptance Criteria 기반의 테스트 코드 자동 생성 (TDD)

&nbsp;

코드를 실행할 환경은 마련되었다. 이제 무엇을 실행할 것인가? 
1편에서 블로그 마크다운을 파싱할 때 우리가 추출해 둔 **Acceptance Criteria (테스트 통과 기준)** 객체를 기억하는가? 에이전트는 애플리케이션 코드를 짜기 전, 이 기준을 바탕으로 Jest 나 Vitest 기반의 테스트 코드를 먼저(TDD) 작성해야 한다.

&nbsp;

## 2-1. Test Generation Prompt
블로그 글의 체크리스트가 기계가 이해할 수 있는 BDD(Behavior-Driven Development) 테스트 코드로 번역된다.

&nbsp;

```python
prompt = f"""
당신은 QA 자동화 엔지니어입니다. 다음 요구사항(Acceptance Criteria)을 완벽하게 검증하는 Vitest 단위 테스트 코드를 작성하십시오.

[요구사항]:
{acceptance_criteria_json}

[타겟 코드 시그니처]:
{target_interface_signatures}

[제약 사항]
1. DB 연결은 모두 Mocking 하십시오 (jest.mock 사용).
2. Happy Path와 Edge Case를 모두 커버하는 describe 블록을 구성하십시오.
"""
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. Self-Correction: 실패를 학습하는 제어 루프

&nbsp;

생성된 테스트 코드와 애플리케이션 코드를 샌드박스에 밀어 넣고 테스트를 돌리면, 첫 시도에서는 90% 이상의 확률로 에러(Red)가 발생한다. 여기서 끝난다면 장난감에 불과하다. 이 에러 로그를 읽고 코드를 스스로 고치는(Refactoring) 피드백 루프가 파이프라인의 핵심이다.

&nbsp;

## 3-1. Traceback Parsing과 에러 주입
샌드박스에서 뱉어낸 에러 로그에는 쓸데없는 노드 내부 모듈의 스택 트레이스가 가득하다. 이를 정규식으로 파싱하여 **정확히 어느 파일의 몇 번째 줄에서 실패했는지, Expected 값과 Received 값이 어떻게 다른지**만 발췌한다.

&nbsp;

## 3-2. LangGraph 기반 수정 루프 (Correction Edge)
추출된 에러는 다시 코더(Coder) 에이전트에게 쥐여진다.

&nbsp;

```python
def self_correction_node(state: AgentState):
    # 샌드박스의 실행 실패 로그
    failure_log = state["sandbox_log"]
    
    correction_prompt = f"""
    당신이 이전에 작성한 코드는 샌드박스 검증을 통과하지 못했습니다.
    [실패 로그]:
    {failure_log}
    
    테스트 코드가 요구하는 스펙과 실제 구현 로직 사이의 불일치를 분석하고,
    버그가 수정된 새로운 전체 코드를 반환하십시오.
    """
    fixed_code = llm.generate_fix(correction_prompt, state["source_code"])
    
    return {
        "source_code": fixed_code,
        "retry_count": state["retry_count"] + 1
    }
```

&nbsp;

이 사이클을 돌며 에이전트는 자신이 짠 코드를 수정하고 샌드박스에 다시 던지는 작업을 반복한다. 테스트가 'Green(통과)'으로 바뀌거나, 지정된 최대 재시도 횟수(Max Retries = 5)에 도달할 때까지 이 무자비한 기계적 TDD 사이클은 멈추지 않는다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 신뢰는 텍스트가 아닌 '실행'에서 온다

&nbsp;

"AI가 코드를 짰습니다"라는 말은 프로덕션 환경에서 아무런 가치가 없다. "AI가 코드를 짰고, 독립된 샌드박스 환경에서 15개의 엣지 케이스 테스트 코드를 통과했음을 증명했습니다"라는 데이터만이 인간 엔지니어의 신뢰를 얻을 수 있다.

&nbsp;

블로그 명세서를 바탕으로 로직을 짜고, 그 명세서의 체크리스트를 바탕으로 테스트 코드를 짜서 스스로 맞붙게 만드는 쌍방향 검증(Adversarial Validation) 파이프라인. 이것이 환각을 완전히 제어하고 기계가 짠 코드를 프로덕션 레벨로 격상시키는 가장 견고한 엔지니어링 전략이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[Doc-to-Code 4편] GitHub API와 자동화된 PR 생성 — Agentic Git Workflow 구현**

&nbsp;

검증을 통과한 무결점의 코드를 개발자 PC의 로컬 폴더에 묵혀둘 수는 없다. 봇(Bot)이 직접 타겟 레포지토리의 최신 `main` 브랜치를 기반으로 새로운 브랜치를 따고, 변경된 다중 파일을 단일 커밋 트랜잭션으로 묶어 푸시(Push)하며, 블로그 글의 내용을 기반으로 풍부한 마크다운 PR(Pull Request) 본문을 작성하여 리뷰어에게 할당하는 100% 자동화된 Agentic Git Workflow의 Octokit 구현 코드를 깊이 있게 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

DocToCode, AI개발, 샌드박스, TDD자동화, SelfCorrection, 환각방어, CI파이프라인, 에이전트검증
