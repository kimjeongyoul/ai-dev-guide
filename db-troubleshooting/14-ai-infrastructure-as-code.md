# [실전 삽질 10편] 인프라도 코딩이다 — AI 기반 Terraform 상태(State) 충돌 방어와 롤백 자동화

&nbsp;

서버 인프라를 AWS 콘솔에서 마우스 클릭으로 만들던 시대는 지났다. Terraform, Pulumi 같은 IaC(Infrastructure as Code) 도구를 도입하면서, 우리는 인프라 변경 사항을 Git으로 형상 관리하고 코드로 리뷰할 수 있는 축복을 얻었다. 

&nbsp;

하지만 코드는 곧 버그를 내포한다. 애플리케이션 버그는 롤백(Rollback)하면 그만이지만, 데이터베이스 인스턴스를 날려버리는 인프라 버그는 회사의 명운을 가른다. "단지 보안 그룹 룰 하나 수정했을 뿐인데, 왜 RDS가 삭제 후 재성성(Force Replacement)된다고 뜨지?" 식은땀이 흐르는 이 끔찍한 상황. 본 글에서는 무심코 누른 `terraform apply`가 초래할 대참사를 사전에 차단하기 위해, LLM을 인프라 검열관(Guardrail)으로 세워 파괴적 플랜(Plan)을 해석하고 State 충돌을 방어하는 자동화 파이프라인 아키텍처를 심층 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 인프라 파괴의 전조: 멍청한 리뷰와 `Force Replacement`

&nbsp;

Terraform은 선언적(Declarative) 도구다. 현재 인프라 상태(State)와 개발자가 작성한 코드(Desired)를 비교하여 변경 사항(Plan)을 제시한다.

&nbsp;

## 1-1. Plan의 인지적 과부하
변경 사항이 많을 때 `terraform plan`의 출력 로그는 수백 줄의 난해한 텍스트 기호(`+`, `-`, `~`)로 도배된다.
엔지니어는 "VPC 태그만 바뀐 거겠지"라고 짐작하고 스크롤을 휙 넘긴 뒤 `yes`를 타이핑한다. 하지만 AWS 제공자(Provider)의 특성상, 특정 속성(예: RDS의 특정 파라미터나 EBS 볼륨 타입 일부)은 단순 수정(In-place update)이 불가능하여 리소스를 아예 파괴하고 새로 만들어버린다(Destroy and then Create). 라이브 서비스 중인 DB가 수십 분 동안 증발해 버리는 것이다.

&nbsp;

## 1-2. 휴먼 에러를 막을 수 없는 한계
CI/CD 파이프라인(GitHub Actions)에 `terraform plan` 결과를 PR(Pull Request) 코멘트로 달아주는 자동화는 이미 널리 쓰인다. 하지만 이 역시 결국 '사람의 눈'으로 긴 로그를 꼼꼼히 확인해야 한다는 한계를 벗어나지 못한다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 아키텍처: LLM 인프라 가드레일 (Guardrail) 파이프라인

&nbsp;

이 참사를 시스템적으로 막기 위해, Terraform 플랜 결과를 LLM(GPT-4o)이 분석하여 위험도를 평가하고 배포 권한을 통제하는 CI/CD 가드레일 아키텍처를 구축했다.

&nbsp;

## 2-1. Plan JSON 추출과 파싱
Terraform의 출력은 기본적으로 텍스트지만, 머신러닝 처리를 위해 구조화된 JSON으로 변환할 수 있다.
```bash
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
```
이 JSON 파일에는 어떤 리소스가 어떤 액션(`create`, `update`, `delete`, `replace`)을 수행하는지 완벽하게 파싱되어 있다. 우리는 파이썬 스크립트를 통해 `delete`나 `replace` 액션이 포함된 리소스 정보만 발췌하여 컨텍스트 용량을 최적화한다.

&nbsp;

## 2-2. Threat Analysis Prompt 설계
추출된 변경 내역을 바탕으로 LLM에게 인프라 보안/운영 책임자로서의 페르소나를 부여하여 엄격한 심사를 강제한다.

&nbsp;

```python
prompt = f"""
당신은 최고 수준의 AWS/Terraform 인프라 운영 책임자입니다.
아래 제공된 Terraform Plan JSON 요약본을 분석하여, 현재 프로덕션 환경에 미칠 치명적 영향을 평가하십시오.

[분석 대상 리소스 변경 내역]:
{pruned_plan_json}

[평가 지침]
1. `replace`(삭제 후 재생성) 액션이 포함된 리소스 중, 상태 저장성(Stateful) 리소스(RDS, ElastiCache, S3, EFS 등)가 단 하나라도 존재합니까?
2. 보안 그룹(Security Group)이나 IAM Role의 권한이 지나치게 넓게 열리는(예: 0.0.0.0/0 허용) 변경 사항이 있습니까?
3. 위 항목 중 하나라도 해당한다면 `risk_level`을 "CRITICAL"로 판정하고, 어떤 속성(Attribute) 변경이 이 파괴적 액션을 유발했는지 기술하십시오.

[출력 포맷 JSON]
{{
  "risk_level": "SAFE" | "WARNING" | "CRITICAL",
  "critical_resources": ["aws_db_instance.main_db", ...],
  "human_readable_summary": "엔지니어가 즉각 위험을 인지할 수 있는 3줄 요약 요약"
}}
"""
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. 실전 동작 로직: State 충돌과 배포 차단 (Block)

&nbsp;

## 3-1. PR Auto-Blocking 및 알람
GitHub Actions 파이프라인은 이 LLM의 분석 결과(`risk_level`)를 반환받는다.
- **SAFE**: 기존처럼 `apply` 준비 상태로 넘어간다.
- **CRITICAL**: 파이프라인이 즉시 `exit 1`을 뱉으며 배포(Apply) 단계를 강제 블록(Block)한다. 동시에 슬랙 알람으로 "🚨 **[CRITICAL RISK]** `aws_db_instance.main_db` 리소스가 `allocated_storage` 변경으로 인해 삭제 후 재성성될 위험이 감지되었습니다. 즉시 코드 수정을 요망합니다"라는 구체적인 경고를 발송한다.

&nbsp;

## 3-2. Terraform State Lock 분쟁 해소 가이드
종종 두 엔지니어가 동시에 인프라를 수정하다가 S3 백엔드의 State Lock이 꼬이는(Lock Error) 상황이 발생한다.
우리의 에이전트는 에러 로그에서 `Lock ID`를 파싱한 뒤, 단순히 에러를 뱉고 죽는 것이 아니라 "현재 State가 User B의 작업으로 잠겨 있습니다. 강제로 락을 해제하려면 `terraform force-unlock <LOCK_ID>` 명령어를 사용하시되, User B의 작업이 완전히 종료되었는지 확인하십시오"라는 트러블슈팅 가이드라인 코멘트까지 자동으로 PR에 남겨준다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 결론: 인프라 보호의 마지막 보루는 자동화된 의심이다

&nbsp;

Infrastructure as Code의 철학은 인프라를 애플리케이션 코드처럼 유연하게 다루는 것이지만, 그 실행 결과의 무게는 결코 가볍지 않다. 코드 한 줄의 오타가 전체 데이터베이스를 날려버릴 수 있는 이 살얼음판 위에서, 인간의 주의력만을 믿는 것은 엔지니어링의 직무 유기다.

&nbsp;

Terraform Plan의 결과를 기계적으로 파싱하고, LLM의 추론 능력을 빌려 '파괴적 행위'의 문맥을 이해하여 사전에 멱살을 잡고 멈춰 세우는 가드레일(Guardrail) 시스템. 이것은 AI가 우리의 일자리를 뺏는 도구가 아니라, 우리가 돌이킬 수 없는 실수를 저지르는 것을 막아주는 가장 완벽한 인프라 방패가 될 수 있음을 증명하는 완벽한 유즈케이스다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고 (FE 최적화 시리즈로 이어집니다)

&nbsp;

> **[FE 최적화 6편] 예측형 프리페칭(Predictive Prefetching) — AI 모델 기반의 제로 레이턴시 라우팅**

&nbsp;

모든 링크를 무식하게 백그라운드에서 미리 받아와서 사용자의 대역폭과 배터리를 고갈시키는 기존의 프리페칭 방식은 잊어라. 사용자의 과거 이동 흐름(Google Analytics) 데이터를 바탕으로 마르코프 체인(Markov Chain) 확률 행렬을 빌드 타임에 주입하여, "다음 3초 안에 사용자가 클릭할 확률이 80% 이상인 단 1개의 경로"만을 선별적으로 미리 렌더링해 두는 극한의 프론트엔드 예측형 라우팅 아키텍처를 공개한다.

&nbsp;

&nbsp;

---

&nbsp;

Terraform, IaC, 인프라자동화, LLMOps, 백엔드아키텍처, 인프라가드레일, State관리, DevOps
