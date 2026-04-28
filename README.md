# AI & Backend Dev Guide

AI 에이전트, 대규모 데이터 처리, 그리고 현대적인 풀스택 아키텍처까지 — 실무 지향적 개발 가이드 모음입니다.

---

## 🚀 심화 기술 아티클 시리즈 (Deep Dive)

아키텍처 설계와 성능 최적화의 핵심 원리를 심층적으로 다룹니다.

### 백엔드 심화 (Backend Deep Dive)
- [01. Kafka Zero-copy와 고성능 메시징의 원리](backend-deep-dive/01-kafka-zero-copy.md)
- [02. 분산 락(Distributed Lock) 구현과 동시성 제어](backend-deep-dive/02-distributed-lock.md)
- [03. OAuth 2.0과 OIDC 보안 아키텍처](backend-deep-dive/03-oauth-oidc-security.md)
- [04. gRPC와 Protobuf의 내부 동작 및 성능 분석](backend-deep-dive/04-grpc-protobuf-internal.md)
- [05. Saga 패턴을 활용한 분산 트랜잭션 구현](backend-deep-dive/05-saga-pattern-implementation.md)
- [06. 분산 환경의 데이터 일관성 전략: Saga 패턴 설계](backend-deep-dive/06-distributed-transaction-saga-pattern.md)
- [07. 분산 시스템의 신뢰성 기초: 합의 알고리즘과 쿼럼 설계](backend-deep-dive/07-distributed-consensus-raft-paxos.md)
- [08. Java 동시성 모델의 변화: Virtual Thread 도입 가이드](backend-deep-dive/08-java-virtual-thread-performance.md)
- [09. 인프라 부하를 유발하는 Cache Stampede 현상과 방어 전략](backend-deep-dive/09-cache-stampede-prevention-strategy.md)
- [10. 대규모 데이터 처리를 위한 DB 샤딩과 리샤딩 전략 분석](backend-deep-dive/10-database-sharding-and-resharding.md)
- [11. CAP를 넘어 PACELC로: 분산 시스템 설계 가이드](backend-deep-dive/11-pacelc-theory.md)
- [12. 데이터 유실 없는 고성능의 비밀: Write-Ahead Logging(WAL)](backend-deep-dive/12-write-ahead-logging.md)
- [13. 무중단 배포의 정석: Blue-Green vs Canary 선택 가이드](backend-deep-dive/13-blue-green-vs-canary.md)
- [14. AI 시대의 백엔드: RAG 아키텍처와 Vector DB의 역할](backend-deep-dive/14-rag-vector-db-architecture.md)

### 프런트엔드 심화 (Frontend Deep Dive)
- [01. Micro-Frontends 아키텍처 도입과 통합 전략 분석](frontend-deep-dive/01-micro-frontends-architecture.md)
- [02. 브라우저 CRP 최적화를 통한 LCP 개선 전략](frontend-deep-dive/02-critical-rendering-path-optimization.md)
- [03. React Server Components(RSC) 메커니즘과 페칭 전략 변화](frontend-deep-dive/03-react-server-components-mechanics.md)
- [04. 현대적 프런트엔드 배포 파이프라인과 트래픽 제어 전략](frontend-deep-dive/04-frontend-deployment-strategies.md)
- [05. 기술 부채와 비즈니스 가치의 균형을 위한 의사결정 전략](frontend-deep-dive/05-technical-debt-management-strategy.md)
- [06. Micro-Frontends 런타임 통합: Module Federation과 배포](frontend-deep-dive/06-micro-frontends-runtime-integration.md)
- [07. 웹 성능의 새로운 기준: INP(Interaction to Next Paint) 개선](frontend-deep-dive/07-inp-performance-optimization.md)
- [08. RSC 시대의 상태 관리: 서버와 클라이언트 상태의 재정의](frontend-deep-dive/08-rsc-state-management.md)
- [09. 테스트 자동화 전략: Testing Pyramid vs Testing Trophy](frontend-deep-dive/09-testing-strategy-trophy.md)
- [10. 엔지니어링 리더십: 건강한 코드 리뷰 문화와 갈등 중재](frontend-deep-dive/10-engineering-leadership-culture.md)

### 인프라 심화 (Infrastructure Deep Dive)
- [01. Multi-Region Active-Active 아키텍처 설계와 데이터 정합성 관리](infrastructure-deep-dive/01-multi-region-active-active-design.md)
- [02. FinOps 관점의 클라우드 인프라 비용 최적화 로드맵](infrastructure-deep-dive/02-finops-cost-optimization-roadmap.md)
- [03. Kubernetes 대규모 트래픽 대응을 위한 고도화된 스케일링 전략](infrastructure-deep-dive/03-kubernetes-advanced-scaling-strategy.md)
- [04. Zero Trust 모델 기반의 인프라 보안 설계 전략](infrastructure-deep-dive/04-zero-trust-infrastructure-security.md)
- [05. AI/LLM 서비스를 위한 인프라 설계와 추론 최적화 전략](infrastructure-deep-dive/05-ai-llm-serving-infrastructure.md)
- [06. Sidecarless 아키텍처: Istio Ambient Mesh의 등장 배경](infrastructure-deep-dive/06-sidecarless-mesh-ambient.md)
- [07. 플랫폼 공학의 정수: IDP 설계와 거버넌스 유지 전략](infrastructure-deep-dive/07-platform-engineering-idp.md)
- [08. 멀티 클라우드 전략: 리전 간 지연 시간과 데이터 동기화](infrastructure-deep-dive/08-multi-cloud-data-networking.md)
- [09. AI 시대를 대비하는 GPU 인프라: 스케줄링과 비용 최적화](infrastructure-deep-dive/09-gpu-resource-optimization.md)
- [10. Zero Trust 보안 모델: 서비스 메시와 mTLS 방어 전략](infrastructure-deep-dive/10-zero-trust-mtls-security.md)
- [11. 카오스 엔지니어링: 시스템 복원력(Resilience) 검증](infrastructure-deep-dive/13-chaos-engineering-resilience.md)
- [12. 리전 장애와 연쇄적 붕괴 방지: 클라우드 복원력 설계](infrastructure-deep-dive/14-regional-resilience-and-cascading-failure.md)

---

## 🤖 AI 에이전트 및 실전 자동화

### AI Agent 실전 가이드
개념부터 아키텍처, 비용, 보안, 배포까지 AI 에이전트 도입의 전 과정을 다룹니다.
- [개념 및 아키텍처](ai-agent-guide/01-concept.md) / [사무·고객·서버 에이전트 활용](ai-agent-guide/03-office-agent.md) / [비용 및 보안](ai-agent-guide/07-cost.md) / [배포 및 운영](ai-agent-guide/09-deployment.md)

### AI 에이전트 설계 패턴
- [사고 방식 패턴 (ReAct, CoT)](ai-agent-design/01-thinking-patterns.md) / [자동화 파이프라인](ai-agent-design/02-automation-pipeline.md) / [멀티 에이전트 오케스트레이션](ai-agent-design/03-multi-agent.md)

### AI 하네스 (AI Harness)
AI에게 사내 규칙을 가르치고 행동을 제어하는 프레임워크 가이드입니다.
- [하네스의 개념](ai-harness/01-what-is-harness.md) / [프로젝트 규칙 작성](ai-harness/02-project-rules.md) / [메모리와 Skills](ai-harness/03-memory-skills.md) / [보안 및 도입 전략](ai-harness/04-hooks-security.md)

---

## 📊 대규모 데이터 및 실무 기술 가이드

### 대규모 데이터 처리
- [배치 vs 실시간 전략](large-scale-data/01-batch-vs-realtime.md) / [Spring Batch 심화](large-scale-data/02-spring-batch.md) / [Kafka 기반 병렬 처리](large-scale-data/04-kafka-worker.md) / [10억 건 아키텍처](large-scale-data/07-billion-architecture.md)

### DB 트래픽 및 트러블슈팅 (DB Troubleshooting)
- [인덱스 최적화](db-troubleshooting/01-perf-index.md) / [슬로우 쿼리 분석](db-troubleshooting/02-perf-slow-query.md) / [데드락 방지](db-troubleshooting/06-sapjil-deadlock-prevention.md) / [커넥션 풀 누수 관리](db-troubleshooting/07-sapjil-connection-pool-leak.md)

### Java & Node.js 실무 패턴 (Java-Node Practice)
- [JPA N+1 문제 해결](java-node-practice/01-java-mistake-n-plus-1.md) / [트랜잭션 관리 실수 사례](java-node-practice/02-java-mistake-transaction.md) / [Java vs Node.js 레이어 설계 비교](java-node-practice/06-java-node-structure.md)

---

## 📝 단독 기술 아티클

- [AI 사고 방식 패턴 완전 정리](articles/ai-patterns-guide.md)
- [CI/CD와 AI 결합 가이드](articles/cicd-ai-pipeline.md)
- [클라우드 비용 비교 및 실전 견적](articles/cloud-cost-comparison.md)
- [API 보안 및 백오피스 설계 체크리스트](articles/api-security-admin-checklist.md)
- [효율적인 패키지 네이밍 가이드](articles/package-naming.md)

---

## About

현업 개발자가 실무에서 경험한 기술적 도전과 해결책을 정리한 저장소입니다.

- AI 에이전트 설계 및 도입
- 고성능 백엔드 아키텍처 및 분산 시스템
- 인프라 보안 및 비용 최적화 (FinOps)
- 프런트엔드 성능 및 마이크로 아키텍처

---

**개발자 가이드:** 다른 PC에서 작업할 때 비공개 규칙 파일을 로드하려면 아래 명령어를 실행하세요:
```bash
git submodule update --init
```
