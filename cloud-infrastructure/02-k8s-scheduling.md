# [플랫폼 2편] K8s 스케줄링 메커니즘 — Resource 오버커밋(Overcommit)과 Bin-packing 최적화 전략

&nbsp;

쿠버네티스(Kubernetes) 클러스터에서 파드(Pod)를 배포하면, `kube-scheduler`는 수십, 수백 개의 노드 중 가장 적합한 하나를 선택하여 파드를 할당한다. 이 과정은 단순히 빈자리를 찾는 행위가 아니라, 시스템 가용성과 자원 효율성(Efficiency) 사이의 치열한 최적화 알고리즘의 결과다. 

&nbsp;

우리는 흔히 `Request`와 `Limit` 설정을 대충 넘기지만, 이 숫자 하나가 스케줄러의 의사결정 트리를 뒤흔든다. 본 글에서는 쿠버네티스 스케줄러의 내부 동작 3단계(Filtering, Scoring, Binding)를 분석하고, 클러스터 자원 점유율을 극대화하면서도 안정성을 해치지 않는 **Bin-packing** 전략과 **Resource Overcommit**의 기술적 실체를 파헤친다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 스케줄러의 의사결정 파이프라인

&nbsp;

파드가 생성되면 스케줄러는 해당 파드를 대기열(Scheduling Queue)에 넣고, 다음의 3단계 공정을 거쳐 노드를 확정한다.

&nbsp;

## 1-1. Filtering (Predicates) — "누가 자격이 있는가?"
가장 먼저 파드가 요구하는 제약 조건을 만족하지 못하는 노드들을 탈락시킨다.
- **Resource Check**: 노드의 가용 자원(Allocatable)이 파드의 `Request` 보다 큰가?
- **Node Selector / Affinity**: 파드가 특정 레이블(Label)이 있는 노드를 원하는가?
- **Taints and Tolerations**: 노드가 파드의 진입을 거부하고 있지는 않은가?
- **Port Check**: 파드가 요구하는 호스트 포트가 이미 점유되어 있는가?

&nbsp;

## 1-2. Scoring (Priorities) — "누가 가장 훌륭한가?"
필터링을 통과한 후보 노드들을 대상으로 점수를 매긴다. 여기서 플랫폼 엔지니어의 정책 방향에 따라 스케줄러의 성격이 결정된다.
- **LeastRequestedPriority**: 자원이 가장 많이 남은 노드에 점수를 더 준다 (자원 분산, 안전 우선).
- **MostRequestedPriority**: 자원이 이미 많이 사용 중인 노드에 점수를 더 준다 (**Bin-packing**, 비용 최적화 우선).
- **ImageLocalityPriority**: 파드가 사용할 이미지가 이미 캐싱된 노드에 가산점을 준다 (기동 속도 우선).

&nbsp;

## 1-3. Binding — "계약 확정"
가장 높은 점수를 받은 노드가 결정되면, 스케줄러는 API 서버에 `Binding` 객체를 전송하여 노드 할당을 확정한다. 이제 해당 노드의 `kubelet`이 파드 실행을 준비하게 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. Bin-packing 전략: 비용 절감의 핵심 아키텍처

&nbsp;

클라우드 비용을 아끼기 위해 엔지니어는 노드 개수를 최소화해야 한다. 이를 위해 파드들을 특정 노드에 꽉꽉 채워 넣는 기법이 **Bin-packing**이다.

&nbsp;

## 2-1. 왜 Bin-packing인가?
모든 노드에 파드를 골고루 분산하면(Spread), 트래픽이 적을 때 모든 노드가 20%씩의 부하만 갖게 된다. 이 경우 노드를 하나도 끌(Scale-in) 수 없다. 반면 Bin-packing을 통해 특정 노드들을 80~90%까지 채우면, 나머지 빈 노드들을 완전히 종료시켜 막대한 인프라 비용을 절감할 수 있다.

&nbsp;

## 2-2. 기술적 위험: 자원 경합(Contention)
Bin-packing의 부작용은 '여유 공간의 부재'다. 특정 파드의 트래픽이 갑자기 튀어 `Limit`까지 자원을 소모하려 할 때, 옆에 있는 파드 역시 확장을 시도한다면 노드의 CPU/메모리 압박(Pressure)이 심해져 연쇄적인 응답 지연이 발생할 수 있다. 

&nbsp;

&nbsp;

---

&nbsp;

# 3. Resource Overcommit의 명암

&nbsp;

쿠버네티스는 물리적인 실제 자원보다 더 많은 `Request` 합계를 허용할 수 있다. 이를 **Overcommit**이라 한다.

&nbsp;

- **원리**: 모든 파드가 동시에 100%의 자원을 사용하지 않는다는 통계적 가설에 기반한다. 
- **설계**: `Request`는 스케줄링의 기준이 되고, `Limit`은 물리적 제어의 기준이 된다. 
  - `Request`의 합계가 노드 용량을 초과하지 않게 설계하되, `Limit`의 합계는 노드 용량의 1.5~2배까지 설정하여 유연성을 확보한다.
- **대참사 방어 (Eviction)**: 만약 실제 메모리 사용량이 노드의 한계를 넘어서면, Kubelet은 `BestEffort` -> `Burstable` -> `Guaranteed` 순으로 파드를 강제 종료(Eviction)하여 노드를 보호한다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 실전 최적화: 커스텀 스케줄러와 프로파일

&nbsp;

기본 스케줄러의 Scoring 가중치가 마음에 들지 않는다면, **Scheduling Profiles**를 통해 정책을 수정할 수 있다.

&nbsp;

```yaml
# kube-scheduler configuration 예시
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: bin-packing-scheduler
    plugins:
      score:
        enabled:
          - name: NodeResourcesFit # MostRequested 전략 활성화
            weight: 100
```

&nbsp;

이 설정을 통해 엔지니어는 특정 서비스(배치 작업 등)에는 자원을 꽉 채워 넣는 스케줄러를 적용하고, 핵심 API 서비스에는 자원을 넓게 퍼뜨리는 스케줄러를 적용하는 식의 이원화 전략을 취할 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 스케줄링은 통계와의 싸움이다

&nbsp;

쿠버네티스 스케줄링의 본질은 "하드웨어 자원의 활용률을 극대화하면서도 장애 발생 확률을 용인 가능한 수준으로 통제하는 것"이다. 단순히 파드가 배포되는 것에 만족하지 마라. 

&nbsp;

현재 우리 클러스터의 노드들이 얼마나 파편화(Fragmentation)되어 있는지, 특정 파드의 `Request` 설정이 너무 과하여 '유령 자원'을 점유하고 있지는 않은지 상시 모니터링해야 한다. 스케줄러의 머릿속을 이해하는 자만이 1원의 낭비도 없는 견고한 플랫폼을 구축할 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[플랫폼 3편] eBPF와 관측성(Observability) — 커널 레벨에서 애플리케이션 병목을 훔쳐보는 기술**

&nbsp;

애플리케이션 코드를 단 한 줄도 수정하지 않고, 어떻게 0.1ms 단위의 함수 호출 지연이나 네트워크 패킷 유실을 잡아낼 수 있을까? 최근 플랫폼 엔지니어링의 대세로 떠오른 eBPF의 작동 원리를 파헤친다. 커널 샌드박스에서 돌아가는 코드의 안전성을 보장하는 Verifier 메커니즘과, 이를 활용한 Cilium/Hubble 아키텍처를 심층 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

K8s스케줄러, BinPacking, 자원최적화, Overcommit, 리소스제한, 쿠버네티스인터널, 플랫폼엔지니어링, 인프라비용절감
