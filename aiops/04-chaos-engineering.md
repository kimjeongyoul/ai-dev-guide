# [실전 AIOps 4편] 카오스 엔지니어링(Chaos Engineering)의 AI 튜닝 — 스스로 시스템의 약점을 공격하는 봇

&nbsp;

장애가 터진 후에야 "아, 이 서버가 죽으면 저 서버도 같이 죽는구나"를 깨닫는 것은 삼류 엔지니어링이다. 진정한 분산 시스템의 신뢰성(Reliability)은 장애가 터지기 전, 의도적으로 시스템의 일부를 망가뜨려 서킷 브레이커(Circuit Breaker)나 Fallback 로직이 정상 작동하는지 확인하는 **카오스 엔지니어링(Chaos Engineering)**을 통해 증명된다.

&nbsp;

하지만 MSA 환경에서 "어떤 마이크로서비스를, 언제, 어떻게 끊어봐야 가장 치명적인 결함을 찾을 수 있을까?"를 사람이 고민하고 스크립트를 짜는 것은 매우 소모적이다. 본 글에서는 LLM이 시스템 아키텍처(K8s Manifest, Terraform)를 딥리딩(Deep Reading)하여 스스로 '가장 아플 만한 취약점'을 계산하고, Chaos Mesh나 Gremlin API를 찔러 네트워크 지연(Latency)과 패킷 드롭(Packet Drop) 공격을 수행하는 자율형 카오스 에이전트(Autonomous Chaos Agent)의 아키텍처를 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 아키텍처 분석 엔진: AI가 인프라의 급소를 찾는 법

&nbsp;

카오스 봇(Bot)에게 무작정 "아무 파드(Pod)나 랜덤으로 죽여봐"라고 시키는 것은 의미가 없다. AI는 먼저 시스템의 의존성(Dependency Graph)을 수학적으로 이해해야 한다.

&nbsp;

## 1-1. K8s Manifest와 Zipkin 데이터의 컨텍스트 주입
에이전트는 매주 정해진 시간(Game Day)에 실행되며, 가장 먼저 프로덕션 환경의 인프라 메타데이터를 파싱한다.
- `kubectl get deployments -o yaml`을 통해 각 서비스의 레플리카 개수, 리소스 Limits, Liveness Probe 설정을 읽어 들인다.
- 분산 트레이싱(Zipkin/Jaeger) API를 호출하여 최근 1주일간 트래픽 양이 가장 많았고, 호출 깊이(Depth)가 깊은 서비스 간의 연관 관계(Topology) JSON을 추출한다.

&nbsp;

## 1-2. Threat Modeling Prompt
수집된 데이터를 바탕으로 LLM에게 취약점 시나리오를 설계하게 만든다.

&nbsp;

```python
prompt = f"""
당신은 극악무도한 카오스 엔지니어링 마스터입니다. 
제공된 K8s 토폴로지와 Zipkin 트래픽 데이터를 분석하여, 시스템의 가용성을 무너뜨릴 수 있는 '단일 장애점(SPOF)'을 3개 도출하고 공격 시나리오를 작성하십시오.

[인프라 컨텍스트]:
{k8s_topology_json}
{zipkin_dependency_graph}

[공격 조건]
1. 단순 파드 삭제(Pod Kill) 외에, 특정 서비스 간의 네트워크 지연(Network Latency Injection 3000ms) 공격을 반드시 포함하십시오.
2. 공격 후 타겟 서비스가 뱉어내야 할 정상적인 에러 코드(Fallback 처리 결과)를 예측하여 기술하십시오.

[출력 포맷 (Chaos Mesh YAML)]
```yaml
# 실제 K8s 클러스터에 배포 가능한 Chaos Mesh 형식의 YAML을 생성할 것
```
"""
```

&nbsp;

이 프롬프트를 통과하면, AI는 "결제 서비스(Payment)가 타임아웃을 뱉을 때 주문 서비스(Order)의 쓰레드 풀이 고갈되는지 보자"며, `Payment`와 `Order` 사이의 네트워크에만 3초의 지연을 먹이는 섬뜩하고 정교한 `NetworkChaos` YAML 파일을 생성해 낸다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 실행 파이프라인: 안전망(Blast Radius)과 자동 중단(Abort)

&nbsp;

AI가 짠 공격 스크립트를 프로덕션에 그대로 실행하는 것은 자살 행위다. 실험이 실제 고객의 결제를 심각하게 방해한다면 즉각 멈춰야 한다(Abort Condition).

&nbsp;

## 2-1. Blast Radius (폭발 반경) 통제
카오스 봇은 실험을 시작할 때, 전체 트래픽의 1%만 통과하는 카나리(Canary) 노드나 특정 테스트 네임스페이스에만 Chaos Mesh 객체를 적용(Apply)하도록 강제된다. 이는 에이전트의 툴(Tool) 내부에 하드코딩된 논리적 방어막이다.

&nbsp;

## 2-2. Prometheus 기반의 실시간 모니터링 (Halt Logic)
봇은 공격을 시작함과 동시에 모니터링 워커를 별도 스레드로 띄운다.
- 공격 대상이 아닌 '메인 게이트웨이'의 HTTP 500 에러 비율이 평소보다 5% 이상 상승하거나, 결제 성공률 지표가 하락하면,
- 워커는 즉시 메인 에이전트에게 인터럽트(Interrupt)를 보내고, K8s API를 찔러 `kubectl delete -f chaos.yaml`을 실행하여 1초 만에 인프라를 원상 복구시킨다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 실험 결과의 검증과 자동 리포팅

&nbsp;

공격이 무사히(혹은 중단되며) 끝났다면, 에이전트는 결과 보고서를 작성한다. 단순히 "실험 끝"이 아니라, 서킷 브레이커(Resilience4j 등)가 의도한 대로 작동했는지 코드를 교차 검증한다.

&nbsp;

### 📊 AI 카오스 실험 리포트 (예시)
- **공격 시나리오**: `payment-service` 네트워크 지연 3000ms 주입
- **예상 결과**: `order-service`에서 1초 만에 Read Timeout을 뱉고 Fallback 로직(임시 저장)을 타야 함.
- **실제 결과 (FAILED)**: `order-service`의 커넥션 풀이 3초씩 물고 대기하다가 풀이 고갈되며 연쇄 장애 발생 (500 Error 200건 감지되어 자동 중단됨).
- **원인 분석**: `order-service`의 FeignClient `readTimeout` 설정이 5000ms로 너무 길게 잡혀 있음.
- **Action Item**: `application.yml`의 `feign.client.config.payment.readTimeout`을 1000ms로 수정하고 재배포 요망.

&nbsp;

이 보고서를 받은 개발팀은 실제로 코드의 설정값을 수정하게 되며, 시스템의 내결함성(Fault Tolerance)은 사람의 감이 아닌 데이터를 기반으로 진화하게 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 약점을 사랑하는 시스템만이 살아남는다

&nbsp;

클라우드 환경에서 장애는 '예외'가 아니라 '상수'다. 언제든 랙(Rack) 전원이 나가거나 네트워크 스위치가 고장 날 수 있다. 

&nbsp;

카오스 엔지니어링을 AI 에이전트에게 맡긴다는 것은, 해커보다 더 집요하게 우리 시스템의 약점을 찾아내어 공격하는 화이트해커 봇을 내부 인프라에 상주시키는 것과 같다. 매주 금요일 오후, 봇이 스스로 생성한 공격 시나리오를 견뎌내는 시스템. 그것이 SRE가 도달할 수 있는 신뢰성(Reliability)의 극한이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 AIOps 5편] 포스트모템(Post-mortem) 자동화 — 슬랙 대화와 메트릭을 취합하는 장애 보고서 에이전트**

&nbsp;

장애 복구가 끝난 뒤 찾아오는 또 다른 고통, 바로 '포스트모템 작성'이다. 언제 누가 알람을 확인했고, 무슨 명령어를 쳤으며, 근본 원인이 무엇이었는지 슬랙 로그와 Datadog 차트를 뒤적이며 3시간 동안 문서를 쓰는 노가다. AI 에이전트가 슬랙 타임라인, K8s 이벤트 로그, Git 커밋 내역을 긁어모아 마크다운 포맷의 완벽한 'Blameless Post-mortem' 문서를 1분 만에 자동 생성해 내는 NLP 요약 파이프라인 아키텍처를 공개한다.

&nbsp;

&nbsp;

---

&nbsp;

카오스엔지니어링, ChaosMesh, SRE, LLM에이전트, 내결함성, AIOps, 서킷브레이커, 인프라테스트
