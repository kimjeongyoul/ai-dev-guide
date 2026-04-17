# AI & Backend Dev Guide

AI 에이전트, 하네스, 대규모 데이터 처리까지 — 실무에서 바로 쓸 수 있는 개발 가이드 모음입니다.

---

## AI Agent 실전 가이드 (10편)

AI Agent를 회사에 도입하기 위한 A to Z. 개념부터 비용 산정, 보안, 배포까지.

| # | 제목 |
|---|------|
| 1 | [AI Agent란 — 챗봇과 뭐가 다른가](ai-agent-guide/01-concept.md) |
| 2 | [아키텍처 기초 — LLM + RAG + Tool Use 조합](ai-agent-guide/02-architecture.md) |
| 3 | [사무직 Agent — 이메일, 문서, 데이터 자동화](ai-agent-guide/03-office-agent.md) |
| 4 | [고객 응대 Agent — 챗봇, 키오스크, 콜센터](ai-agent-guide/04-customer-agent.md) |
| 5 | [서버 Agent — 모니터링, 장애 감지, 자동 복구](ai-agent-guide/05-server-agent.md) |
| 6 | [인프라 & 스펙 — 뭘 깔아야 하나](ai-agent-guide/06-infrastructure.md) |
| 7 | [비용 산정 — API 호출부터 서버까지](ai-agent-guide/07-cost.md) |
| 8 | [보안 — 사내 데이터가 외부로 나가면 안 될 때](ai-agent-guide/08-security.md) |
| 9 | [배포 & 운영 — Docker, 모니터링, 장애 대응](ai-agent-guide/09-deployment.md) |
| 10 | [우리 회사에 도입하기 — ROI부터 확산까지](ai-agent-guide/10-adoption.md) |

---

## AI 에이전트 설계 패턴 (3편)

CoT, ReAct, MCP 등 에이전트의 사고 방식과 설계 패턴.

| # | 제목 |
|---|------|
| 1 | [AI 에이전트의 사고 방식 — ReAct, CoT, Tool Use 패턴 비교](ai-agent-design/01-thinking-patterns.md) |
| 2 | [MCP로 AI 에이전트 만들기 — 실전 자동화 파이프라인](ai-agent-design/02-automation-pipeline.md) |
| 3 | [설계 패턴 — 단일 에이전트 vs 멀티 에이전트 오케스트레이션](ai-agent-design/03-multi-agent.md) |

---

## AI 하네스 (6편)

AI에게 회사 규칙을 가르치고, 행동을 제어하고, 팀에 도입하기.

| # | 제목 |
|---|------|
| 1 | [AI 하네스란 — AI에게 회사 규칙을 가르치는 법](ai-harness/01-what-is-harness.md) |
| 2 | [프로젝트 규칙 파일 작성법 — AI가 읽는 사내 코딩 가이드](ai-harness/02-project-rules.md) |
| 3 | [메모리와 Skills — AI가 같은 실수를 반복하지 않게](ai-harness/03-memory-skills.md) |
| 4 | [Hooks와 보안 — AI의 행동을 강제하고 위험을 차단하기](ai-harness/04-hooks-security.md) |
| 5 | [팀에 하네스 도입하기 — 설계 체크리스트와 단계별 가이드](ai-harness/05-team-rollout.md) |
| 6 | [실전 Q&A — 하네스 도입할 때 진짜 궁금한 것들](ai-harness/06-practical-qa.md) |

---

## 대규모 데이터 처리 (7편)

배치, Kafka, DB 최적화, 스트리밍까지 — 데이터가 커지면 알아야 할 것들.

| # | 제목 |
|---|------|
| 1 | [배치 vs 실시간 — 왜 나누고 언제 합치는가](large-scale-data/01-batch-vs-realtime.md) |
| 2 | [Spring Batch 심화 — 100만~1,000만 건 처리의 정석](large-scale-data/02-spring-batch.md) |
| 3 | [Kafka 기초 — 메시지 큐가 왜 대규모 처리의 핵심인가](large-scale-data/03-kafka-basics.md) |
| 4 | [Kafka + Worker 패턴 — 1억 건을 병렬로 처리하기](large-scale-data/04-kafka-worker.md) |
| 5 | [DB 최적화 — 대량 데이터에서 살아남는 SQL](large-scale-data/05-db-optimization.md) |
| 6 | [실시간 스트리밍 — Kafka Streams와 Flink](large-scale-data/06-streaming.md) |
| 7 | [10억 건 아키텍처 — Spark, 데이터 레이크, 그리고 현실](large-scale-data/07-billion-architecture.md) |

---

## 단독 Articles

| 제목 |
|------|
| [AI는 어떻게 생각하고 행동하는가? — CoT, Tool Use, ReAct, MCP 완전 정리](articles/ai-patterns-guide.md) |
| [CI/CD에 AI를 끼얹으면 — PR부터 프로덕션 배포까지 자동화하기](articles/cicd-ai-pipeline.md) |
| [클라우드 비용 비교 — AWS, NCP, GCP, Azure 트래픽 규모별 실전 견적](articles/cloud-cost-comparison.md) |
| [Node.js vs Spring Batch — 배치 처리, 뭘로 할까](articles/nodejs-vs-spring-batch.md) |
| [프롬프트 체인 실전 — 이커머스 고객 문의 자동 처리 시스템 만들기](articles/prompt-chain-ecommerce.md) |
| [온프레미스 LLM 서빙 — Ollama vs vLLM, 프로덕션에서 AI 모델을 돌린다는 것](articles/vllm-model-serving.md) |
| [LLM 서버에 사람이 몰리면 — 대기열, 스케일 아웃, 오토스케일링](articles/vllm-scaling.md) |

---

## About

현업 개발자가 실무에서 겪은 경험을 바탕으로 정리한 글 모음입니다.

- Blog: https://marshmello.tistory.com
- AI, Backend, DevOps 관련 주제를 다룹니다
