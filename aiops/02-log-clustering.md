# [실전 AIOps 2편] 로그의 의미론적 클러스터링 — ELK 스택과 Vector DB를 활용한 에러 압축 파이프라인

&nbsp;

장애가 발생했을 때 가장 먼저 열어보는 것은 로그(Logs)다. 그러나 대규모 트래픽이 몰리는 분산 시스템(MSA)에서 500 에러가 터지기 시작하면, Kibana나 Datadog 대시보드에는 1초에 수백, 수천 줄의 Stack Trace가 폭포수처럼 쏟아진다. 

&nbsp;

엔지니어는 이 무자비한 텍스트의 바다 속에서 스크롤을 내리며 "아, 이거 아까 본 그 에러랑 같은 거네", "어? 이건 디비 타임아웃 에러가 섞여 있네?"라며 눈으로 패턴을 분류해야 한다. 이 인지적 과부하 상태에서 LLM에게 "이 10만 줄의 로그를 보고 원인을 요약해 줘"라고 던지면 컨텍스트 초과로 터져버릴 뿐이다. 본 글에서는 무의미하게 반복되는 수만 줄의 로그를 기계적으로 압축(Deduping)하고, Vector DB의 유사도 검색을 통해 의미론적으로 클러스터링(Clustering)하여 단 한 장의 '에러 요약 리포트'로 뽑아내는 AIOps 파이프라인을 심층 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 1차 압축: 변수 마스킹과 Log Signature 추출

&nbsp;

에러 로그가 무수히 많아 보이는 이유는 그 안에 포함된 '변수'들 때문이다. 

&nbsp;

```text
// 로그 1: NullPointerException at UserService.java:45 (user_id=8912)
// 로그 2: NullPointerException at UserService.java:45 (user_id=1024)
// 로그 3: NullPointerException at UserService.java:45 (user_id=7731)
```

&nbsp;

이 3개의 로그는 인간의 눈에는 본질적으로 완전히 동일한 에러(Root Cause)다. 하지만 기계의 텍스트 비교나 단순 `GROUP BY`로는 서로 다른 문자열로 취급되어 파편화된다. 

&nbsp;

## 1-1. 정규식 기반의 Drain 알고리즘 적용
로그 전처리 레이어(Logstash 또는 파이썬 스크립트)에서 가장 먼저 해야 할 일은 동적인 변수들을 정적 템플릿(Signature)으로 치환하는 것이다. 대표적인 로그 파싱 알고리즘인 **Drain**이나 커스텀 정규식(Regex)을 사용하여 숫자, UUID, IP 주소, 타임스탬프를 `<VAR>` 토큰으로 마스킹한다.

&nbsp;

```text
// 정제 후의 Log Signature
NullPointerException at UserService.java:<VAR> (user_id=<VAR>)
```

&nbsp;

이 단순한 마스킹 작업만 거친 후 해시(Hash) 값을 기준으로 `GROUP BY count`를 때려도, 10만 줄의 로그가 10~20개의 고유한 '에러 패턴'으로 99% 압축된다. 이제 LLM에게 넘길 페이로드가 수십 줄 단위로 가벼워진 것이다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 2차 압축: Vector 임베딩을 통한 의미론적 클러스터링

&nbsp;

1차 압축을 통해 고유한 패턴 20개를 추려냈다 하더라도, 그중에는 "데이터베이스 커넥션 타임아웃", "Hikari 풀 고갈", "Redis 읽기 실패"처럼 서로 다른 서비스에서 발생했지만 근본적으로 '네트워크/인프라 병목'이라는 동일한 원인을 공유하는 에러들이 섞여 있다. 

&nbsp;

## 2-1. 임베딩(Embedding)과 코사인 유사도 연산
압축된 20개의 Log Signature를 `text-embedding-3-small` 같은 경량 임베딩 모델에 태워 고차원 벡터로 변환한다. 이를 메모리 기반 Vector DB(Milvus, Pinecone 등)나 FAISS 라이브러리를 통해 로드한 뒤, 밀도 기반 클러스터링 알고리즘(DBSCAN 또는 K-Means)을 돌린다.

&nbsp;

- **Cluster A (DB/Network)**: 커넥션 타임아웃, 소켓 에러, 풀 고갈 등
- **Cluster B (Logic Error)**: NullPointer, Syntax Error, Type Mismatch 등
- **Cluster C (Security)**: 403 Forbidden, 토큰 만료 등

&nbsp;

기계적인 텍스트 비교를 넘어, 단어의 '의미(Semantic)'를 기준으로 얽히고설킨 에러들을 정확히 3~4개의 군집으로 묶어낸다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. LLM 기반의 최종 요약 파이프라인

&nbsp;

이제 완벽하게 압축되고 그룹화된 3~4개의 에러 덩어리를 LLM에게 넘겨, 인간 친화적인 리포트를 작성하도록 프롬프트를 주입한다.

&nbsp;

```python
prompt = f"""
당신은 SRE 로그 분석 전문가입니다. 
장애 발생 기간 동안 수집된 10만 줄의 로그를 의미론적으로 압축한 클러스터 데이터가 주어집니다.

[Error Clusters]:
{clustered_logs_json}

각 클러스터별로 다음 항목을 분석하여 마크다운 포맷으로 출력하십시오.
1. 핵심 요약: 이 에러 그룹이 발생한 근본 원인(Root Cause)의 1줄 요약.
2. 연쇄 반응 추론: Cluster A의 에러가 Cluster B의 에러를 유발한 원인인지 (Cascading Effect) 관계를 분석.
3. 발생 빈도: 전체 에러 중 해당 클러스터가 차지하는 비중(%).
"""
```

&nbsp;

이 프롬프트를 통과한 LLM의 출력 결과는 경이롭다.

&nbsp;

> 📊 **장애 로그 분석 리포트 (총 125,000건 분석 완료)**
> 
> **1. 핵심 장애 원인 (네트워크 타임아웃군 / 85%)**
> - `payment-db`의 부하로 인한 HikariCP Connection Timeout이 전체 장애의 85%를 차지하는 주범입니다.
> 
> **2. 연쇄 반응 에러 (비즈니스 로직 에러군 / 12%)**
> - DB 연결 실패로 인해 반환된 `null` 객체를 `OrderService.java`에서 그대로 참조하려다 발생한 `NullPointerException` 덩어리입니다. DB 장애가 로직 장애로 전파(Cascading)된 결과입니다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 무의미한 데이터는 노이즈일 뿐이다

&nbsp;

로그를 많이 남기는 것만이 능사가 아니다. 아무리 방대하고 디테일한 스택 트레이스라도, 위급한 장애 상황에서 인간의 뇌가 이를 1분 안에 파싱하고 구조화하지 못한다면 그것은 시스템 모니터링을 방해하는 끔찍한 시각적 노이즈(Noise)에 불과하다.

&nbsp;

Drain 알고리즘으로 변수를 쳐내고, Vector 임베딩으로 의미를 묶어내며, LLM으로 인과 관계를 추론해 내는 3단계의 로그 클러스터링 파이프라인. 이것은 쏟아지는 데이터의 홍수 속에서 엔지니어의 정신적 피로(Burnout)를 막고, 진정한 문제 해결에만 두뇌 리소스를 집중하게 만드는 AIOps의 핵심 코어 엔진이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 AIOps 3편] AI 런북(Runbook) 자동 실행 — K8s Pod 재시작과 DB Failover를 수행하는 Action Agent**

&nbsp;

장애 원인을 AI가 찾아냈다면, 복구(Remediation)도 AI가 직접 하면 안 될까? 새벽 4시에 알람을 받고 일어난 엔지니어가 비몽사몽간에 쿠버네티스(K8s) 콘솔을 열어 `kubectl delete pod`을 치거나 DB 커넥션 풀을 플러시하는 단순 반복 작업을 완전히 자동화한다. 에이전트가 쿠버네티스 API 권한을 부여받고, 사전에 정의된 런북(Runbook) 스크립트를 기반으로 OOM 파드를 덤프 뜨고 안전하게 재시작하는 **Auto-Remediation(자동 복구)** 시스템의 아키텍처와 보안 가드레일을 심층적으로 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

로그분석, AIOps, LogClustering, VectorDB, ELK, 스택트레이스, SRE, LLM파이프라인
