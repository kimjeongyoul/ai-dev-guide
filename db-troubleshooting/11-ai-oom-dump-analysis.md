# [실전 삽질 7편] OOM(Out of Memory)의 늪 — 힙 덤프(Heap Dump)와 AI 기반 메모리 누수 패턴 추적

&nbsp;

Java 백엔드 엔지니어에게 가장 공포스러운 예외(Exception)는 단연 `java.lang.OutOfMemoryError`다. OOM이 발생하면 시스템은 비정상 상태에 빠져 모든 연산을 멈추고, 톰캣(Tomcat)은 죽어버린다. 더 끔찍한 것은 이 에러가 '원인'을 가리키지 않는다는 점이다. 메모리가 가득 찬 순간 실행되던 억울한 스레드가 에러를 뿜을 뿐, 진짜 메모리를 서서히 갉아먹은 범인은 수백만 개의 객체 인스턴스 숲 속에 숨어 있다.

&nbsp;

과거에는 수 기가바이트(GB)짜리 힙 덤프(`.hprof`)를 떠서 Eclipse MAT(Memory Analyzer Tool)로 수시간 동안 객체 그래프를 쫓아가며 가비지 컬렉터(GC) 루트를 추적해야 했다. 본 글에서는 이 지루하고 휴먼 에러가 잦은 과정을 **OQL(Object Query Language) 스크립트 기반 데이터 추출과 LLM의 패턴 인식 능력을 결합하여 10분 만에 누수 원인을 도출하는 자동화 파이프라인**으로 혁신한 실전 사례를 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 힙 덤프(Heap Dump) 분석의 한계

&nbsp;

운영 서버에 `-XX:+HeapDumpOnOutOfMemoryError` 옵션을 켜두면, OOM 발생 직전의 메모리 상태가 바이너리 파일로 떨어진다.

&nbsp;

## 1-1. 컨텍스트의 부재
MAT의 Dominator Tree를 열어보면 특정 `HashMap`이나 `ArrayList`가 힙의 80%를 점유하고 있다고 알려준다. 하지만 그 컬렉션이 스프링(Spring)의 내부 빈(Bean)인지, 우리가 짠 `LocalCacheManager`인지, 아니면 서드파티 라이브러리의 `ThreadLocal` 인지 매핑하기 위해서는 패키지 구조와 비즈니스 로직을 완벽히 꿰고 있어야 한다.

&nbsp;

## 1-2. AI 도입의 딜레마: 용량
이 거대한 `.hprof` 파일(보통 2~8GB)을 통째로 ChatGPT에 올릴 수는 없다. LLM에게 필요한 것은 바이너리 덤프가 아니라, **의미론적으로 압축된 객체 참조 통계 데이터**다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 아키텍처: Jhat과 OQL을 활용한 텍스트 데이터 추출

&nbsp;

AI가 분석할 수 있도록 바이너리 덤프를 정형화된 텍스트 리포트로 변환하는 파이프라인을 구축했다.

&nbsp;

### Step 1. CLI 기반 덤프 분석 스크립트
JVM 내장 도구인 `jhat` 또는 오픈소스 `jmap`을 활용하여 객체 크기별 히스토그램을 추출한다.
```bash
jmap -histo:live <pid> | head -n 50 > heap_histogram.txt
```
이 텍스트 파일에는 OOM 당시 메모리를 가장 많이 점유하고 있는 클래스명(패키지 경로 포함)과 인스턴스 개수, 바이트 크기가 담긴다.

&nbsp;

### Step 2. OQL을 통한 의심 객체 참조 추적
상위 점유 객체가 식별되면, 스크립트가 자동으로 OQL을 실행하여 해당 객체들을 누가 붙잡고(Hold) 있어서 GC가 수거하지 못했는지(Incoming References)를 JSON 형태로 뽑아낸다.
```sql
-- MAT 내장 OQL 문법 (예시)
SELECT objects v FROM java.util.HashMap v WHERE v.size > 100000
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. LLM을 통한 누수 패턴(Leak Pattern) 추론

&nbsp;

추출된 `heap_histogram.txt`와 OQL 결과 JSON을 LLM의 컨텍스트로 주입한다. 이때 프롬프트에는 OOM의 전형적인 안티 패턴 지식 베이스를 명시적으로 포함해야 한다.

&nbsp;

```python
prompt = f"""
당신은 JVM 메모리 튜닝 전문가입니다. 제공된 힙 히스토그램과 OQL 참조 데이터를 분석하여 OOM의 원인을 추론하세요.

[OOM Anti-Patterns Reference]
1. ThreadLocal Leak: 스레드 풀 환경에서 ThreadLocal.remove()를 호출하지 않아 객체가 쌓임.
2. Unbounded Cache: ConcurrentHashMap 등을 캐시로 쓰면서 TTL이나 MaxSize 제한을 두지 않음.
3. Unclosed Resources: DB ResultSet, InputStream 등을 명시적으로 close() 하지 않음.

[Data]
Histogram: {histogram_text}
OQL References: {oql_json}

[Output Format JSON]
{{
  "suspected_class": "패키지명이 포함된 클래스",
  "leak_pattern": "위 Reference 중 하나",
  "reasoning": "왜 그렇게 판단했는지 기술적인 근거",
  "action_item": "코드 레벨의 해결책"
}}
"""
```

&nbsp;

&nbsp;

---

&nbsp;

# 4. 실전 사례: ThreadLocal과 Tomcat 스레드 풀의 끔찍한 동거

&nbsp;

이 파이프라인이 잡아낸 실제 프로덕션 장애 사례다.

&nbsp;

### AI 분석 결과 리포트
- **Suspected Class**: `com.example.security.UserSessionContext`
- **Leak Pattern**: ThreadLocal Leak
- **Reasoning**: 히스토그램 상 `UserSessionContext` 객체가 10,000개 이상 누적되어 힙의 60%를 점유함. OQL 분석 결과 이 객체들은 `java.lang.ThreadLocal$ThreadLocalMap`에 의해 참조되고 있으며, 톰캣의 `http-nio-8080-exec-*` 스레드들이 GC Root로 작용하여 수거가 불가능한 상태임.
- **Action Item**: `HandlerInterceptor` 또는 `Filter`의 `afterCompletion` 단계에서 반드시 `UserSessionContext.clear()`를 호출하여 스레드가 풀로 반환되기 전에 ThreadLocal을 초기화해야 함.

&nbsp;

엔지니어가 이 결론에 도달하려면 반나절 동안 덤프를 파헤치고 톰캣의 스레드 풀 동작 방식을 구글링해야 했을 것이다. 그러나 AI 파이프라인은 단 30초 만에 정확히 코드 라인 수정을 지시하는 해결책을 내놓았다.

&nbsp;

&nbsp;

---

&nbsp;

# 5. 결론: 디버깅의 패러다임 전환

&nbsp;

버그를 고치는 시간보다 버그의 원인을 찾는 시간이 훨씬 길다. OOM 덤프 파일이나 쓰레드 덤프(Thread Dump)처럼 인간의 눈으로 파싱하기 힘든 거대한 메타데이터 텍스트 덩어리는 LLM의 추론 능력이 가장 빛을 발하는 영역이다.

&nbsp;

데이터를 사람이 읽기 좋은 GUI 도구(MAT)로 보는 시대를 넘어, 원시 데이터를 AI가 이해할 수 있는 텍스트 포맷으로 직렬화(Serialize)하여 프롬프트로 쏘아 올리는 'Data to Prompt' 아키텍처 파이프라인 구축 능력이 향후 백엔드 엔지니어의 생산성을 좌우할 것이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 삽질 8편] 마이크로서비스 미아 찾기 — LLM을 활용한 분산 트레이싱(Zipkin) 로그 상관관계 분석**

&nbsp;

A 서비스가 B 서비스를 부르고, B가 C와 D를 비동기로 부르는 MSA 환경. 갑자기 D 서비스에서 타임아웃이 발생했을 때, 도대체 최초에 이 요청을 보낸 A 서비스의 트랜잭션이 무엇인지 추적하는 것은 악몽과 같다. Zipkin/Jaeger에서 쏟아지는 수만 개의 Span 데이터를 Vector화하여, 하나의 Trace ID로 묶인 연쇄 장애의 근본 원인(Root Cause)을 AI가 수학적으로 짚어내는 로그 상관관계 분석 아키텍처를 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

OOM분석, HeapDump, JVM튜닝, LLM활용, 메모리누수, ThreadLocal, 백엔드디버깅, OQL
