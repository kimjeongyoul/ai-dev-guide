# [실전 삽질 8편] 마이크로서비스 미아 찾기 — LLM을 활용한 분산 트레이싱(Zipkin) 로그 상관관계 분석

&nbsp;

모놀리식(Monolithic) 아키텍처에서는 에러가 나면 스택 트레이스(Stack Trace) 하나만 보면 원인을 잡을 수 있었다. 하지만 마이크로서비스 아키텍처(MSA) 환경에서는 이야기가 다르다. 

&nbsp;

사용자가 '주문하기' 버튼을 누르면 API 게이트웨이를 거쳐 주문 서버(A), 재고 서버(B), 결제 서버(C), 포인트 서버(D)가 비동기 메시지 큐(Kafka)를 통해 얽히고설키며 수십 개의 내부 API 호출을 발생시킨다. 만약 결제 서버(C)에서 타임아웃이 발생해 주문이 실패했다면, 엔지니어는 수백 대의 서버 컨테이너에 분산된 로그 더미 속에서 이 거대한 트랜잭션의 연쇄 고리를 수동으로 맞춰봐야 한다.

&nbsp;

우리는 Zipkin과 Jaeger 같은 분산 트레이싱 도구를 도입했지만, 여전히 수만 개의 Span(구간) 데이터를 눈으로 파싱하는 한계에 부딪혔다. 본 글에서는 이 거대한 로그의 바다에 LLM을 투입하여, 하나의 Trace ID로 묶인 연쇄 장애의 **진정한 근본 원인(Root Cause)**을 수학적으로 짚어내는 로그 상관관계 분석 파이프라인 구축기를 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 분산 트레이싱의 구조와 인지적 한계

&nbsp;

Zipkin의 아키텍처는 간단하다. 클라이언트의 최초 요청 시 하나의 고유한 `Trace ID`가 발급되고, 각 마이크로서비스를 거칠 때마다 부모-자식 관계를 나타내는 `Span ID`와 `Parent Span ID`가 발급되어 HTTP 헤더(B3 Propagation)를 통해 전파된다.

&nbsp;

- **문제점**: 장애 발생 시 `Trace ID`로 검색하면, 하나의 트랜잭션에 엮인 50~100개의 Span 로그가 시간순으로 쏟아진다.
- 결제 서버(C)가 타임아웃이 나서 실패 로그를 뱉었지만, 진짜 원인은 결제 서버가 찌른 외부 PG사의 응답 지연일 수도 있고, 결제 서버가 참조하는 Redis 캐시 서버의 일시적인 순단일 수도 있다. 
- 인간의 뇌는 100개의 병렬/순차 Span 트리 구조를 머릿속으로 그리며 "아, C가 늦어진 이유는 3단계 아래에 있는 D의 응답 지연이 전파된(Cascading) 것이구나"라고 단번에 추론하지 못한다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 아키텍처: Span 데이터의 Vectorization과 구조화

&nbsp;

이 100개의 JSON Span 데이터를 LLM에게 쌩으로 던지면 토큰 한도 초과(Context Window Overflow)로 뻗어버리거나 환각(Hallucination)을 일으킨다. 이를 해결하기 위해 에이전트 파이프라인을 구축했다.

&nbsp;

## 2-1. Graph Tree Builder (파이썬 스크립트)
Elasticsearch에 저장된 Zipkin 데이터를 가져와서, `Span ID`와 `Parent Span ID`를 바탕으로 방향성 비순환 그래프(DAG) 형태의 트리를 메모리상에 먼저 구축한다.
이 트리에서 정상적으로 10ms 이내에 끝난 Span 노드들은 모조리 가지치기(Pruning) 해버린다. 오직 에러 코드(5xx)를 뱉었거나 평균 응답 시간(p95)을 초과한 노드, 그리고 그 노드의 직계 부모/자식 노드들만 추출하여 컨텍스트의 크기를 1/10로 압축한다.

&nbsp;

## 2-2. 텍스트 직렬화 (Serialization)
압축된 노드 데이터를 LLM이 가장 이해하기 쉬운 텍스트 포맷(예: YAML 또는 간소화된 텍스트 트리)으로 직렬화한다.

&nbsp;

```yaml
Trace ID: a1b2c3d4
Anomalies Detected:
  - Service: api-gateway
    Span ID: 111
    Duration: 5020ms
    Error: "504 Gateway Timeout"
    Children:
      - Service: order-service
        Span ID: 222
        Duration: 5015ms
        Error: "feign.RetryableException: Read timed out"
        Children:
          - Service: payment-service
            Span ID: 333
            Duration: 5005ms
            Error: "java.net.SocketTimeoutException: connect timed out"
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. LLM 프롬프팅: Root Cause 추론 엔진

&nbsp;

압축된 트레이스 데이터를 LangChain을 통해 GPT-4o 급의 모델에 주입한다. 프롬프트에는 MSA 환경의 전형적인 장애 전파(Cascading Failure) 패턴에 대한 지식 베이스를 명시해야 한다.

&nbsp;

```text
당신은 마이크로서비스 트러블슈팅 전문가입니다. 제공된 분산 트레이싱(Span) 압축 데이터를 분석하여 다음을 수행하세요.

[규칙]
1. 에러의 '결과'를 뱉은 서비스(예: Timeout을 맞은 Gateway)가 아닌, 지연을 '유발한' 가장 밑단(Leaf)의 서비스를 찾아내세요.
2. 각 서비스 간의 Duration(소요 시간) 차이를 분석하여, 네트워크 지연(Network Latency)인지 애플리케이션 처리 지연인지 수학적으로 유추하세요.

[출력 포맷]
- Root Cause Service: (가장 밑단에서 지연/에러를 유발한 서비스명)
- Cascading Path: (장애가 전파된 경로 A -> B -> C)
- Analysis: (원인에 대한 기술적 분석 3줄 요약)
- Action Item: (담당 팀이 확인해야 할 인프라 지표 또는 코드 포인트)
```

&nbsp;

&nbsp;

---

&nbsp;

# 4. 실전 도입 효과: MTTR의 혁명적인 단축

&nbsp;

이 시스템을 슬랙 봇(Slack Bot)에 연동했다. 서버 장애 알람이 울리면, 엔지니어는 슬랙 명령어 `/analyze-trace [TraceID]`만 입력한다. 

&nbsp;

30초 뒤, 봇은 **"장애의 근본 원인은 `payment-service`가 아니라, 그 하위에서 호출된 `pg-proxy-service`의 외부 방화벽 룰 변경에 따른 Socket Timeout 현상이며, 이 지연이 5초간 쌓여 상위의 `order-service` 커넥션 풀을 고갈시켰습니다. PG 연동 모듈의 Read Timeout 설정을 현재 5초에서 1초로 줄이고 Fallback 로직을 타게 하십시오"** 라는 완벽한 사후 분석 리포트를 뱉어낸다.

&nbsp;

기존에 시니어 엔지니어 3명이 모여 키바나(Kibana)와 Zipkin 화면을 띄워놓고 1시간 동안 벌여야 했던 숨바꼭질이 단 30초 만에 끝난 것이다. MTTR(Mean Time To Recovery, 평균 복구 시간)이 혁명적으로 단축되었다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 데이터의 바다에는 나침반이 필요하다

&nbsp;

마이크로서비스 아키텍처는 시스템의 결합도를 낮추는 대신, 운영의 복잡도를 기하급수적으로 높였다. 쏟아지는 로그와 트레이스 데이터를 인간의 눈과 직관으로 분석하는 시대는 이미 지났다.

&nbsp;

가비지 데이터를 쳐내는 정교한 전처리(Pruning) 파이프라인과, 상관관계를 수학적으로 추론하는 LLM 엔진의 결합. 이것이 고도화된 분산 시스템 환경에서 백엔드 엔지니어가 반드시 갖추어야 할 AIOps(AI for IT Operations)의 청사진이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 삽질 9편] 레거시 청산의 공포 — AI 기반 섀도우 트래픽(Shadow Traffic) 검증과 무중단 마이그레이션**

&nbsp;

수백만 줄의 스파게티 코드로 얽힌 10년 된 레거시 API를 새로운 MSA로 걷어내는 과정. 기존 API와 신규 API의 응답값을 완벽히 일치시키기 위한 고통스러운 텍스트 비교 노가다를 끝내기 위해, Envoy 프록시의 섀도우 트래픽 라우팅 기능과 LLM의 '의미론적 JSON Diff 분석기'를 결합하여 무중단/무결점 마이그레이션을 이뤄낸 하드코어 아키텍처를 공개한다.

&nbsp;

&nbsp;

---

&nbsp;

분산트레이싱, 마이크로서비스, Zipkin, Jaeger, LLMOps, 로그분석, AIOps, 백엔드디버깅
