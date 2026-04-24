# [실전 삽질 6편] 슬로우 쿼리 분석의 자동화 — Vector DB와 LLM 기반 옵티마이저 튜닝

&nbsp;

매일 아침 출근하면 가장 먼저 하는 일은 데이터베이스의 슬로우 쿼리(Slow Query) 로그를 뒤적이는 것이었다. DBA가 없는 조직에서 백엔드 엔지니어는 `EXPLAIN` 결과를 눈으로 파싱하고, 인덱스 카디널리티를 계산하며, 최적의 쿼리를 유추하는 고통스러운 반복 작업에 시달린다. 

&nbsp;

단순히 "이 쿼리가 느리다"고 알려주는 APM(Datadog, New Relic)을 넘어, **"이 쿼리가 왜 느리고, 어떤 복합 인덱스를 추가해야 하며, 그로 인한 사이드 이펙트는 무엇인지"**를 AI가 스스로 분석하고 리포팅하는 **'LLM 기반 자동화 파이프라인'**을 구축한 실전 아키텍처를 공유한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 아키텍처: Slow Query to LLM 파이프라인

&nbsp;

LLM에게 무작정 쿼리만 던지면 "인덱스를 추가하세요"라는 교과서적인 헛소리만 돌아온다. AI가 DBA 수준의 답변을 내놓으려면 DB의 **메타데이터(스키마, 인덱스 상태, 테이블 크기)**와 **실행 계획(Execution Plan)**이라는 컨텍스트를 완벽하게 주입(Context Injection)해야 한다.

&nbsp;

## 1-1. 데이터 수집 파이프라인 설계
우리는 MySQL의 `long_query_time`을 2초로 설정하고, Fluentd를 통해 슬로우 쿼리 로그를 스트리밍했다.
1. **Log Parser**: Fluentd 정규식을 통해 `Time`, `Rows_examined`, `Query` 본문을 파싱.
2. **Context Gatherer (Python Worker)**: 파싱된 쿼리를 가로채어, 실시간으로 DB에 붙어 `EXPLAIN FORMAT=JSON`을 실행하고 쿼리 실행 계획을 JSON으로 추출.
3. **Schema Fetcher**: `information_schema`에서 해당 테이블의 DDL(Create Table)과 현재 걸려있는 인덱스 통계를 추출.

&nbsp;

## 1-2. Vector DB를 활용한 히스토리 관리
과거에 이미 분석했던 쿼리 패턴을 또 분석하며 비싼 토큰을 낭비할 필요는 없다. Pinecone(Vector DB)에 `Query Embedding` 값을 저장하여, 코사인 유사도 0.95 이상인 쿼리가 들어오면 과거의 튜닝 리포트를 즉시 반환하는 시맨틱 캐싱(Semantic Caching) 레이어를 앞단에 배치했다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 핵심 구현: LLM Prompting과 JSON 파싱

&nbsp;

가장 중요한 것은 LLM에게 던지는 프롬프트의 구조화다. 우리는 LangChain을 활용하여 다음과 같은 구조화된 체인(Chain)을 구성했다.

&nbsp;

```python
from langchain.prompts import PromptTemplate
from langchain.chat_models import ChatOpenAI

template = """
당신은 15년 차 MySQL DBA입니다. 아래 제공된 슬로우 쿼리와 실행 계획(EXPLAIN JSON), 그리고 테이블 스키마를 분석하여 최적화 리포트를 작성하십시오.

[Query]:
{query}

[Table Schema & Index Stats]:
{schema_info}

[EXPLAIN FORMAT=JSON]:
{explain_json}

분석 시 다음 사항을 반드시 포함하여 JSON 형태로 반환하십시오:
1. bottleneck_reason: type이 ALL(Full Scan)인지, filesort가 발생했는지 등 정확한 병목 원인.
2. suggested_index: 추가해야 할 DDL (CREATE INDEX 구문).
3. trade_off: 해당 인덱스 추가 시 INSERT/UPDATE 성능 저하 우려도.
"""

prompt = PromptTemplate(
    input_variables=["query", "schema_info", "explain_json"],
    template=template
)
# GPT-4o 급 모델을 사용하여 고도의 추론 강제
chain = prompt | ChatOpenAI(model_name="gpt-4o", temperature=0)
```

&nbsp;

이러한 `Few-Shot` 기반의 구조화된 프롬프트를 통해, LLM은 단순히 "인덱스를 거세요"가 아니라 `"idx_status_created_at 복합 인덱스를 걸되, status의 카디널리티가 낮으므로 created_at을 선행 컬럼으로 두는 것이 유리합니다"`라는 소름 돋는 통찰을 내놓기 시작했다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 실제 적용 사례: Filesort 병목의 해결

&nbsp;

어느 날 파이프라인을 통해 슬랙으로 경고 알람이 도착했다.

&nbsp;

### AI 분석 리포트 요약
- **문제 쿼리**: `SELECT * FROM payments WHERE user_id = 123 ORDER BY amount DESC LIMIT 10;`
- **병목 지점**: `user_id` 인덱스는 타고 있으나, `ORDER BY amount`를 처리하기 위해 `Using filesort` (메모리 밖 임시 파일 정렬)가 발생하여 디스크 I/O 유발.
- **제안 (Suggested DDL)**: `ALTER TABLE payments ADD INDEX idx_user_amount (user_id, amount DESC);`
- **트레이드 오프**: 해당 테이블은 초당 50회의 `INSERT`가 발생하므로, B-Tree 재정렬 오버헤드가 약 3% 증가할 수 있으나, Read 부하 90% 감소 이점이 압도적임.

&nbsp;

놀랍게도 MySQL 8.0부터 지원되는 내림차순 인덱스(Descending Index) 문법까지 정확히 짚어냈다. 개발팀은 이 리포트를 그대로 복사하여 Flyway 마이그레이션 스크립트에 추가했고, 해당 API의 지연 시간(p95)은 3.2초에서 0.05초로 수직 낙하했다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 결론: AI는 코딩을 돕는 게 아니라 인프라를 해석한다

&nbsp;

AI를 단순히 "보일러플레이트 코드 짜주는 봇"으로 쓰는 것은 능력 낭비다. 로그 텍스트, JSON 메타데이터, 스택 트레이스 등 기계가 뱉어내는 파편화된 데이터를 통합하여 의미 있는 **엔지니어링 컨텍스트**로 번역해 내는 것이 LLM의 진정한 가치다.

&nbsp;

이 자동화 파이프라인 구축 후, 우리 팀은 슬로우 쿼리 분석에 쏟던 주당 10시간의 리소스를 0으로 만들었으며, DBA 없이도 안정적인 쿼리 성능을 유지할 수 있는 강력한 무기를 얻었다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 삽질 7편] OOM(Out of Memory)의 늪 — 힙 덤프(Heap Dump)와 AI 기반 메모리 누수 패턴 추적**

&nbsp;

JVM 기반 서버가 주기적으로 뻗으며 `java.lang.OutOfMemoryError: Java heap space`를 뱉는다. 수 기가바이트(GB)에 달하는 힙 덤프(`.hprof`) 파일을 Eclipse MAT로 열어 수동으로 객체 참조 트리(GC Root)를 뒤지던 원시적인 디버깅을 버리고, OQL(Object Query Language) 추출 결과와 LLM을 결합하여 ThreadLocal 누수와 캐시 비대화를 자동으로 추적해 내는 메모리 분석 파이프라인을 공개한다.

&nbsp;

&nbsp;

---

&nbsp;

슬로우쿼리, 데이터베이스최적화, LLM활용, 옵티마이저분석, VectorDB, LangChain, 백엔드자동화, MySQL튜닝
