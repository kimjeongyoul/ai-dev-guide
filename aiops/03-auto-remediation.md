# [실전 AIOps 3편] AI 런북(Runbook) 자동 실행 — K8s Pod 재시작과 DB Failover를 수행하는 Action Agent

&nbsp;

장애 탐지(1편)와 로그 분석(2편)을 통해 장애의 원인을 1분 만에 파악했다면, 다음 단계는 당연히 복구(Remediation)다. 

&nbsp;

"CPU가 폭주해서 `payment-api` 파드가 응답하지 않습니다"라는 완벽한 AI 진단 리포트를 슬랙으로 받아보았자, 엔지니어가 노트북을 켜고 VPN에 접속해 터미널을 열고 `kubectl delete pod` 명령어를 수동으로 치기 전까지 서비스는 멈춰 있다. 인지 시간은 1분으로 줄였어도, 실제 조치 시간(Time to Resolve)은 여전히 인간의 물리적 한계에 묶여 있는 셈이다.

&nbsp;

왜 AI가 원인도 찾았는데 버튼까지 대신 눌러주지 않는가? 본 글에서는 단순한 Read-only 어드바이저를 넘어, 시스템 권한을 쥐고 인프라를 직접 조작하는 **Action Agent (자동 복구 에이전트)**의 구현 아키텍처와, 자율 시스템이 시스템을 더 망가뜨리지 않게 제어하는 보안 가드레일(Guardrails)을 심도 있게 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 자율 복구 아키텍처: 런북(Runbook)을 코드로 번역하다

&nbsp;

경험 많은 SRE 팀은 장애 유형별로 어떤 명령어를 쳐서 복구해야 하는지 정리한 문서인 런북(Runbook)을 가지고 있다. Action Agent의 역할은 자연어로 된 런북을 파싱하여, 상황에 맞는 API(또는 셸 스크립트)를 실행하는 것이다.

&nbsp;

## 1-1. Tool Use (Function Calling) 인터페이스
에이전트에게 인프라 제어 권한을 주기 위해, 시스템 내부에 철저히 격리된 실행 가능한 툴(Tool) 세트를 정의한다.

&nbsp;

```json
// K8s 제어 툴 규격
{
  "name": "restart_k8s_deployment",
  "description": "비정상 상태에 빠진 특정 Deployment의 파드들을 순차적으로 롤링 리스타트합니다.",
  "parameters": {
    "type": "object",
    "properties": {
      "namespace": { "type": "string" },
      "deployment_name": { "type": "string" },
      "take_heap_dump_before_restart": { "type": "boolean", "description": "재시작 전 분석용 덤프 생성 여부" }
    },
    "required": ["namespace", "deployment_name"]
  }
}
```

&nbsp;

에이전트는 앞선 로그 분석에서 `OutOfMemory` 에러를 확정 지은 후, 스스로 위 툴을 호출하여 "이 파드는 죽이기 전에 덤프(Heap Dump)를 떠야 사후 분석이 가능하다"는 SRE의 철학까지 로직에 반영(`take_heap_dump_before_restart: true`)하여 조치를 실행한다.

&nbsp;

## 1-2. 워크플로우(Workflow) 자동화 엔진
에이전트는 단일 툴만 호출하지 않는다. 
"DB 커넥션 풀 고갈" 알람이 뜨면, 에이전트는 
1. `scale_up_replicas` 툴을 호출해 파드 개수를 늘려 트래픽을 분산시키고, 
2. RDS API를 찔러 `describe_db_connections`로 범인 IP를 찾아낸 뒤, 
3. `kill_idle_connections` 툴로 좀비 커넥션을 날려버리는 다단계 조치(Multi-step Remediation)를 자율적으로 수행한다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 치명적 함정 방어: 가드레일과 Human-in-the-Loop

&nbsp;

권한을 쥔 AI는 무섭다. "트래픽이 많으니 파드를 늘려"라고 지시했는데, 로직의 무한 루프 오류로 K8s 노드(Node)의 자원이 고갈될 때까지 수백 개의 파드를 생성해 버려 클러스터 전체를 다운시킬 수 있다.

&nbsp;

## 2-1. 하드 리미트(Hard Limit) 통제
에이전트가 사용하는 툴 자체에 방어 코드를 하드코딩해야 한다. 
에이전트가 "파드 100개 추가해"라고 명령하더라도, 툴의 내부 구현 코드에서 `if (target_replicas > MAX_LIMIT) return Error;` 와 같이 절대로 넘을 수 없는 물리적 한계를 인프라 엔지니어가 직접 설정해 두어야 한다.

&nbsp;

## 2-2. 파괴적 액션의 사전 승인 (Human-in-the-Loop)
파드 재시작이나 일시적인 스케일 아웃은 자동화하더라도, **DB Failover (마스터-슬레이브 전환)** 나 **테이블 롤백(Rollback)** 같이 데이터 정합성이 깨질 수 있는 파괴적(Irreversible) 액션은 100% 자동화를 금지한다.

&nbsp;

이때 슬랙(Slack)의 Interactive Block을 활용한다.
1. AI 봇: "🚨 Primary DB의 응답이 30초 이상 지연 중입니다. Failover 스크립트 실행을 승인하시겠습니까? [승인] / [거절]"
2. 이 메시지는 엔지니어의 모바일로 푸시되며, 엔지니어가 침대에서 눈을 비비며 [승인] 버튼을 클릭하는 순간, 멈춰 있던 에이전트의 워크플로우가 재개되며 페일오버 스크립트가 실행된다. 

&nbsp;

이것이 시스템의 자율성과 인프라의 안정성을 모두 지켜내는 완벽한 AIOps 협업 모델이다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 사후 보고: 조치 내역의 자동 문서화

&nbsp;

조치가 끝나면 에이전트는 자신이 무슨 짓을 했는지 명확히 기록해야 한다. "03:15 알람 발생 -> 03:16 OOM 진단 -> 03:17 파드 3개 덤프 후 재시작 -> 03:18 CPU 지표 정상화 확인 완료."

&nbsp;

이 모든 타임라인은 Jira 티켓이나 사내 위키에 자동으로 포스팅되어, 다음 날 아침 출근한 팀원들이 밤사이 시스템이 스스로 감기에 걸렸다가 스스로 약을 먹고 나았음을 스탠드업 미팅에서 담소하듯 확인할 수 있게 만든다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 새벽의 평화는 스크립트가 지킨다

&nbsp;

모니터링은 사람이 보고 고치기 위해 존재하는 것이 아니다. 진정한 SRE(Site Reliability Engineering)의 완성은 반복되는 인간의 개입을 코드로 대체하여 소멸시키는 데 있다.

&nbsp;

로그를 분석해 원인을 추론하는 두뇌(LLM)와, K8s와 AWS 인프라를 직접 조작하는 팔다리(Action Tools)를 결합한 Auto-Remediation 아키텍처. 이것은 단순히 비용 절감을 넘어 엔지니어의 수면권과 정신적 건강을 보호하고, 서비스의 평균 복구 시간(MTTR)을 분 단위에서 초 단위로 끌어내리는 엔터프라이즈 인프라 혁신의 종착지다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 AIOps 4편] 카오스 엔지니어링(Chaos Engineering)의 AI 튜닝 — 스스로 시스템의 약점을 공격하는 봇**

&nbsp;

장애가 터진 후 고치는 것은 하수다. 진정한 고수는 장애가 터질 상황을 미리 만들어서 시스템이 버티는지 테스트한다. 시스템 아키텍처(Terraform, K8s YAML) 문서를 LLM이 딥리딩(Deep Reading)한 뒤, "결제 큐가 다운되었을 때 서킷 브레이커가 정상 동작하는지 테스트해 보자"며 스스로 네트워크 단절 및 지연(Latency) 공격 스크립트를 생성하고 시스템의 약점을 집요하게 파고드는 AI 기반 카오스 엔지니어링 봇의 구현과 실전 테스트 아키텍처를 공개한다.

&nbsp;

&nbsp;

---

&nbsp;

AIOps, 자동복구, AutoRemediation, SRE, 쿠버네티스, LLM에이전트, 인프라자동화, 장애대응
