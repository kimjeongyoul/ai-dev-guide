# [실전 AIOps 1편] 장애 탐지의 진화 — 시계열 이상 탐지(Anomaly Detection)와 LLM의 결합

&nbsp;

새벽 3시, PagerDuty 알람이 울린다. "CPU 사용률 90% 초과". 잠결에 랩탑을 열고 Datadog 대시보드와 Kibana 로그를 번갈아 보며 1시간 동안 헤맨 끝에 내린 결론은, 전날 퇴근 직전 배포된 특정 API의 정규식(Regex) 무한 루프 버그였다. 이 고통스러운 On-call 과정은 지난 10년간 모든 백엔드/SRE 엔지니어의 일상이었다.

&nbsp;

단순히 임계치(Threshold)를 넘었다고 알람을 쏘는 멍청한 모니터링의 시대는 끝났다. 
현대의 AIOps는 통계적 시계열(Time-Series) 이상 탐지 알고리즘으로 스파이크를 잡아내고, 그 즉시 LLM이 최근 시스템 변경 사항(Git PR, CI/CD 이벤트)을 교차 분석하여 "어제 배포된 PR #104의 정규식 변경이 CPU 폭증의 원인입니다. 롤백하시겠습니까?"라고 원인 추론(Root Cause Analysis)까지 완료해 내야 한다. 본 글에서는 Prometheus 지표와 LLM 추론 엔진을 엮어낸 장애 탐지 파이프라인의 실전 아키텍처를 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 기존 임계치(Static Threshold) 모니터링의 한계

&nbsp;

"CPU 80% 이상 5분 유지 시 알람"과 같은 고정 임계치 룰은 끔찍한 오탐(False Positive)과 미탐(False Negative)을 낳는다.
- **False Positive**: 트래픽이 몰리는 점심시간에 자연스럽게 치솟는 CPU 사용량을 장애로 오인하여 엔지니어를 양치기 소년으로 만든다 (Alert Fatigue).
- **False Negative**: 평소 CPU를 10%만 쓰던 새벽 시간에, 특정 스레드의 데드락으로 인해 40%까지 치솟았다면 이는 명백한 장애의 전조 증상이지만 80%를 넘지 않았으므로 시스템은 침묵한다.

&nbsp;

## 1-1. 머신러닝 기반의 동적 베이스라인 (Dynamic Baseline)
단순 임계치를 버리고, 시스템의 과거 주기적 패턴(Seasonality)을 학습하여 현재 시점의 '정상 범주'를 동적으로 예측하는 Z-Score 기반의 이상 탐지나 Isolation Forest 알고리즘을 Prometheus / Datadog 앞단에 배치해야 한다. 
"지금 이 시간대라면 CPU는 15~25%여야 하는데, 40%를 찍었으니 비정상(Anomaly)이다"라고 수학적으로 판단하는 것이 1차 탐지 레이어의 핵심이다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 아키텍처: Anomaly Data의 추출과 컨텍스트 어셈블리

&nbsp;

이상 탐지 엔진이 튀는 지표를 발견하면, 알람을 쏘기 전에 **컨텍스트 어셈블리(Context Assembly) 파이프라인**이 가동된다. LLM이 장애 원인을 분석하려면 당시 시스템의 주변 상황을 완벽하게 주입(Injection)해야 하기 때문이다.

&nbsp;

## 2-1. Data Gatherer (Python Worker)
Webhook을 통해 이상 탐지 이벤트를 수신한 백엔드 워커는 다음 3가지 데이터를 10초 이내에 긁어모은다.

&nbsp;

1. **Metrics Snapshot**: Prometheus API를 호출하여 장애 발생 시점 전후 15분의 CPU, Memory, DB Active Connections, HTTP 5xx Error Rate 지표를 JSON 매트릭스로 추출.
2. **Recent Deployments**: GitHub API와 ArgoCD(또는 Jenkins) API를 찔러, 최근 6시간 이내에 해당 마이크로서비스에 배포된 Pull Request의 제목, 커밋 메시지, 변경된 파일 목록(Diff)을 추출.
3. **Recent Logs**: Elasticsearch에 쿼리를 날려 해당 시간대에 급증한 에러 로그(Exception Stack Trace)의 샘플 3개를 확보.

&nbsp;

&nbsp;

---

&nbsp;

# 3. LLM 추론 엔진: Root Cause Analysis 프롬프팅

&nbsp;

수집된 방대한 메타데이터를 LLM(GPT-4o 등)에게 분석하도록 지시한다. 단순히 데이터를 나열하는 것이 아니라, SRE 엔지니어의 사고 과정(Chain of Thought)을 프롬프트에 하드코딩하여 분석의 깊이를 강제한다.

&nbsp;

```python
prompt = f"""
당신은 10년 차 수석 SRE(Site Reliability Engineer)입니다.
현재 프로덕션 환경의 `payment-api` 서비스에서 [CPU 사용률 이상 급증(Anomaly)]이 탐지되었습니다. 
제공된 시스템 메트릭, 최근 배포된 PR 정보, 에러 로그를 교차 분석하여 가장 유력한 Root Cause를 추론하십시오.

[제공된 컨텍스트]
1. Metrics: {metrics_json} (DB 커넥션은 정상이나 CPU만 급증함)
2. Recent PRs: {recent_prs_json} (PR #892: 정산 로직의 정규표현식 성능 개선)
3. Error Logs: {error_logs_json} (TimeoutException 다수 발생)

[추론 지침]
- CPU만 급증하고 DB 커넥션이 정상이라면 무한 루프나 비효율적인 메모리 연산을 의심하십시오.
- 최근 배포된 PR의 변경 사항이 위 가설과 일치하는지 논리적으로 연결하십시오.

[출력 포맷 (JSON)]
{{
  "suspected_cause": "장애의 구체적 원인 1줄 요약",
  "confidence_score": "0~100 사이의 확신도",
  "reasoning": "왜 이런 결론에 도달했는지 3단계 논리적 근거",
  "action_item": "즉각적인 조치 제안 (예: PR #892 롤백)"
}}
"""
```

&nbsp;

&nbsp;

---

&nbsp;

# 4. 실전 도출 결과: 슬랙으로 배달되는 완벽한 진단서

&nbsp;

이 파이프라인이 가동되면, 새벽에 알람을 받고 일어난 엔지니어는 대시보드를 켤 필요조차 없다. 슬랙(Slack) 채널에는 이미 AI가 1분 만에 분석을 끝낸 완벽한 진단서가 도착해 있다.

&nbsp;

> 🚨 **[CRITICAL] payment-api CPU Anomaly Detected**
> 
> **🧠 AI Root Cause Analysis (Confidence: 92%)**
> - **원인**: 방금 배포된 PR #892의 ReDoS(정규표현식 서비스 거부) 취약점으로 인한 메인 스레드 CPU 점유율 100% 도달.
> - **근거**: 
>   1. 03:15에 PR #892 배포 직후 CPU 지표가 15%에서 98%로 수직 상승함.
>   2. DB I/O나 메모리 누수 지표는 평온하므로 외부 연동 문제가 아님.
>   3. 로그 상 `RegexMatchTimeoutException`이 1분간 400회 발생함.
> - **Action Item**: 현재 시스템이 응답 불능 상태이므로, 즉각적인 **[PR #892 롤백]** 및 **[Pod 재시작]**을 권고합니다.

&nbsp;

이 한 통의 메시지는 장애 인지(Detection)부터 원인 파악(Diagnosis)까지 걸리는 MTTR(Mean Time To Resolution)을 기존 40분에서 단 2분으로 단축시키는 기적을 만들어낸다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 모니터링에서 '관측과 추론'으로의 진화

&nbsp;

기계적인 수치를 모니터링 대시보드에 예쁘게 띄워주는 시대는 지났다. 파편화된 지표(Metrics), 로그(Logs), 배포 이력(Events)을 인간의 머릿속에서 조립하여 원인을 찾는 것은 인지적 과부하를 초래한다.

&nbsp;

이상 탐지 알고리즘이 '언제(When)' 장애가 났는지를 수학적으로 짚어내면, LLM 추론 엔진이 '왜(Why)' 장애가 났는지를 시스템 컨텍스트와 결합하여 서술해 내는 것. 이것이 AIOps 인프라가 지향해야 할 궁극적인 '가시성(Observability)'의 미래다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 AIOps 2편] 로그의 의미론적 클러스터링 — ELK 스택과 Vector DB를 활용한 수만 줄의 에러 압축**

&nbsp;

장애 상황에서 1초에 수천 줄씩 쏟아지는 Java/Node.js의 Stack Trace 에러 로그. 엔지니어는 똑같은 에러가 반복되는 스크롤 지옥 속에서 진짜 새로운 에러를 놓치고 만다. 에러 텍스트의 변수(ID, IP, 타임스탬프)를 마스킹하여 로그 템플릿(Signature) 단위로 압축하고, 수십만 줄의 에러를 Vector DB를 활용해 5개의 의미론적 클러스터로 요약하여 LLM에게 넘기는 '초고속 에러 압축 파이프라인'을 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

AIOps, 이상탐지, AnomalyDetection, LLM추론, 장애대응, SRE, RootCauseAnalysis, 인프라모니터링
