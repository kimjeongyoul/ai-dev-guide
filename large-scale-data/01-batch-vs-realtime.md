# 대규모 데이터 처리 시리즈 (1) — 배치 vs 실시간, 왜 나누고 언제 합치는가

&nbsp;

데이터가 1만 건일 때는 아무렇게나 처리해도 된다. 하지만 1,000만 건, 1억 건, 10억 건이 되면 이야기가 완전히 달라진다.

이 시리즈는 **대규모 데이터를 실제로 처리하는 기술**을 단계적으로 다룬다. 첫 번째 편에서는 가장 근본적인 질문에서 시작한다. **배치와 실시간, 무엇을 선택해야 하는가?**

&nbsp;

---

&nbsp;

## 1. 배치 처리란

&nbsp;

배치(Batch) 처리는 데이터를 **모아서 한꺼번에, 주기적으로** 처리하는 방식이다.

```
[데이터 축적] ──→ [특정 시점에 일괄 처리] ──→ [결과 저장]
     ↑                                           │
     └───────── 다음 주기까지 대기 ──────────────┘
```

&nbsp;

**대표적인 배치 처리 시나리오:**
- 매일 자정에 일 매출 집계
- 매주 월요일 주간 리포트 생성
- 매월 1일 회원 등급 재산정
- 매 시간 로그 파일 파싱 및 적재

&nbsp;

Spring Batch로 작성한 간단한 배치 예시:

```java
@Configuration
public class DailySalesJobConfig {

    @Bean
    public Job dailySalesJob(JobRepository jobRepository,
                             Step calculateSalesStep) {
        return new JobBuilder("dailySalesJob", jobRepository)
                .start(calculateSalesStep)
                .build();
    }

    @Bean
    public Step calculateSalesStep(JobRepository jobRepository,
                                    PlatformTransactionManager tx,
                                    ItemReader<Order> orderReader,
                                    ItemProcessor<Order, SalesSummary> processor,
                                    ItemWriter<SalesSummary> writer) {
        return new StepBuilder("calculateSalesStep", jobRepository)
                .<Order, SalesSummary>chunk(1000, tx)
                .reader(orderReader)
                .processor(processor)
                .writer(writer)
                .build();
    }
}
```

&nbsp;

배치의 핵심 특징:
- **지연이 허용된다** — 결과가 즉시 필요하지 않다
- **전체 데이터를 대상으로 한다** — 특정 시점까지의 모든 데이터
- **재실행이 가능하다** — 실패 시 다시 돌릴 수 있다
- **자원을 예측할 수 있다** — 야간 등 트래픽이 적은 시간에 실행

&nbsp;

---

&nbsp;

## 2. 실시간 처리란

&nbsp;

실시간(Real-time) 처리는 데이터가 **들어오는 즉시, 지연을 최소화하여** 처리하는 방식이다.

```
[이벤트 발생] ──→ [즉시 처리] ──→ [즉시 반영]
     ↑                               │
     └─── 다음 이벤트 ────────────────┘
```

&nbsp;

**대표적인 실시간 처리 시나리오:**
- 결제 승인 즉시 재고 차감
- 사용자 클릭 즉시 추천 업데이트
- 이상 거래 탐지 (FDS)
- 실시간 대시보드 갱신

&nbsp;

Kafka Consumer를 이용한 실시간 처리 예시:

```java
@Service
public class OrderEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "inventory-group")
    public void handleOrderEvent(OrderEvent event) {
        switch (event.getType()) {
            case CREATED:
                inventoryService.decrease(event.getProductId(), event.getQuantity());
                break;
            case CANCELLED:
                inventoryService.increase(event.getProductId(), event.getQuantity());
                break;
        }
        log.info("재고 반영 완료: productId={}, lag={}ms",
                event.getProductId(),
                System.currentTimeMillis() - event.getTimestamp());
    }
}
```

&nbsp;

실시간의 핵심 특징:
- **지연이 허용되지 않는다** — 밀리초~초 단위 응답
- **건건이 처리한다** — 이벤트 하나하나가 대상
- **장애에 민감하다** — 멈추면 데이터가 밀린다
- **자원이 항상 필요하다** — 24시간 대기

&nbsp;

---

&nbsp;

## 3. 규모별 판단 기준

&nbsp;

"우리 서비스는 배치로 충분한가, 실시간이 필요한가?"

이 질문에 답하려면 **데이터 규모**와 **비즈니스 요구사항**을 함께 봐야 한다.

&nbsp;

| 일일 데이터량 | 처리 방식 | 기술 스택 |
|:---:|:---:|:---|
| ~100만 건 | 배치로 충분 | Spring Batch + RDBMS |
| 100만~1,000만 건 | 배치 + 일부 실시간 | Spring Batch + Kafka + RDBMS |
| 1,000만~1억 건 | 실시간 주도 + 배치 보조 | Kafka + Worker Pool + 읽기 분리 DB |
| 1억~10억 건 | 실시간 + 분산 배치 | Kafka + Spark + 데이터 레이크 |
| 10억 건~ | 전용 인프라 | Flink + Spark + HDFS/S3 + OLAP DB |

&nbsp;

중요한 건 **규모만 보면 안 된다**는 것이다.

```
일일 1만 건이라도 "1초 안에 이상 거래를 잡아야 한다"면 → 실시간
일일 1억 건이라도 "다음 날 아침 리포트만 있으면 된다"면 → 배치
```

&nbsp;

**판단 체크리스트:**

1. 결과가 즉시 필요한가? → 실시간
2. 전체 데이터를 한번에 봐야 하는가? → 배치
3. 데이터 유실이 발생하면 복구할 수 있는가? → 배치 (재실행 가능)
4. 장애 시 몇 분까지 허용되는가? → 1분 미만이면 실시간
5. 인프라 비용을 감당할 수 있는가? → 실시간은 24시간 자원 소비

&nbsp;

---

&nbsp;

## 4. Lambda 아키텍처

&nbsp;

Lambda 아키텍처는 Nathan Marz가 제안한 구조로, **배치 레이어와 스피드 레이어를 동시에 운영**한다.

&nbsp;

```
                    ┌─────────────────────────────────┐
                    │         데이터 소스              │
                    │   (로그, 이벤트, 트랜잭션)       │
                    └──────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │    메시지 큐         │
                    │    (Kafka)          │
                    └───┬────────────┬────┘
                        │            │
           ┌────────────▼──┐   ┌────▼────────────┐
           │  배치 레이어   │   │  스피드 레이어   │
           │  (Spark)      │   │  (Flink/Storm)  │
           │               │   │                 │
           │ 전체 데이터를  │   │ 최신 데이터만   │
           │ 주기적으로     │   │ 즉시 처리       │
           │ 재계산         │   │                 │
           └───────┬───────┘   └───────┬─────────┘
                   │                   │
           ┌───────▼───────┐   ┌───────▼─────────┐
           │  배치 뷰       │   │  실시간 뷰      │
           │  (정확한 결과) │   │  (근사 결과)    │
           └───────┬───────┘   └───────┬─────────┘
                   │                   │
                   └─────────┬─────────┘
                             │
                    ┌────────▼────────┐
                    │   서빙 레이어    │
                    │   (API 서버)    │
                    │                 │
                    │  배치 뷰와      │
                    │  실시간 뷰를    │
                    │  병합하여 응답   │
                    └─────────────────┘
```

&nbsp;

**각 레이어의 역할:**

| 레이어 | 역할 | 특징 |
|:---:|:---|:---|
| 배치 레이어 | 전체 데이터를 주기적으로 재계산 | 정확하지만 느림 |
| 스피드 레이어 | 마지막 배치 이후의 데이터만 실시간 처리 | 빠르지만 근사값 |
| 서빙 레이어 | 두 결과를 병합하여 최종 응답 | 정확성 + 실시간성 |

&nbsp;

**장점:**
- 정확한 결과와 실시간 결과를 모두 제공
- 배치가 재계산하면 스피드 레이어의 오차가 보정됨

**단점:**
- 같은 로직을 배치와 실시간 두 곳에 구현해야 함 (코드 중복)
- 운영 복잡도가 높음 (두 시스템 관리)

&nbsp;

---

&nbsp;

## 5. Kappa 아키텍처

&nbsp;

Kappa 아키텍처는 Jay Kreps(Kafka 창시자)가 제안한 구조로, **스트리밍 하나로 배치까지 대체**한다.

&nbsp;

```
                    ┌─────────────────────────────────┐
                    │         데이터 소스              │
                    │   (로그, 이벤트, 트랜잭션)       │
                    └──────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   불변 이벤트 로그   │
                    │   (Kafka — 장기 보관)│
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   스트리밍 처리      │
                    │   (Kafka Streams /  │
                    │    Flink)           │
                    │                    │
                    │  로직 변경 시       │
                    │  처음부터 재처리    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   서빙 레이어        │
                    │   (API 서버)        │
                    └─────────────────────┘
```

&nbsp;

핵심 아이디어는 간단하다:

> "Kafka에 모든 이벤트를 보관한다. 로직이 바뀌면 offset을 처음으로 되감아서 전체를 재처리한다."

&nbsp;

```yaml
# Kafka 토픽 설정 — 이벤트를 오래 보관
# server.properties
log.retention.hours=8760        # 1년 보관
log.retention.bytes=-1          # 용량 제한 없음
log.segment.bytes=1073741824    # 1GB 세그먼트
```

&nbsp;

**Lambda vs Kappa 비교:**

| 항목 | Lambda | Kappa |
|:---:|:---:|:---:|
| 처리 엔진 | 배치 + 스트리밍 | 스트리밍만 |
| 코드 중복 | 있음 (두 벌) | 없음 (한 벌) |
| 운영 복잡도 | 높음 | 낮음 |
| 재처리 방식 | 배치가 전체 재계산 | offset 되감기 |
| 정확도 보장 | 배치가 보정 | 스트리밍 자체가 정확해야 함 |
| 적합한 경우 | 복잡한 집계, ML | 이벤트 중심 시스템 |

&nbsp;

---

&nbsp;

## 6. 현실: 배치 + 실시간 조합

&nbsp;

아키텍처 이론은 깔끔하지만, 현실은 대부분 **"실시간 처리 + 야간 배치 정산"** 조합이다.

&nbsp;

```
[낮 — 실시간]
사용자 주문 → Kafka → Consumer → DB 즉시 반영
                                  └→ 실시간 대시보드

[밤 — 배치]
23:00 정산 시작 → Spring Batch → 일 매출 집계
                              → 수수료 계산
                              → 정산 리포트 생성
                              → 다음 날 아침 확인
```

&nbsp;

**왜 이렇게 하는가?**

1. 실시간으로 처리하되, 정확한 정산은 배치로 검증
2. 실시간 처리에서 놓친 건이 있으면 배치가 잡아냄
3. 배치가 "진실의 원천(Source of Truth)"이 됨

&nbsp;

```java
// 실시간: 주문 생성 즉시 → 임시 매출 집계
@KafkaListener(topics = "orders")
public void onOrder(OrderEvent event) {
    realtimeSalesCache.increment(event.getStoreId(), event.getAmount());
    // Redis에 실시간 매출 반영 (빠르지만 근사값)
}

// 배치: 자정에 → 확정 매출 집계
@Bean
public Step dailySettlementStep() {
    return new StepBuilder("dailySettlement", jobRepository)
            .<Order, Settlement>chunk(5000, tx)
            .reader(confirmedOrderReader())   // DB에서 확정된 주문만
            .processor(settlementProcessor()) // 수수료, 부가세 계산
            .writer(settlementWriter())       // 정산 테이블에 저장
            .build();
}
```

&nbsp;

---

&nbsp;

## 7. 업계별 사례

&nbsp;

### 카드사 / 금융

```
실시간: 승인/취소 즉시 처리, FDS(이상거래탐지)
배치:   익일 매입 처리, 월말 청구서 생성, 연체 관리
규모:   일 수천만 건 트랜잭션
핵심:   정합성 > 속도 (1원이라도 안 맞으면 안 됨)
```

### 이커머스

```
실시간: 재고 차감, 추천 엔진, 검색 인덱싱
배치:   판매자 정산, 쿠폰 만료, 통계 리포트
규모:   일 수백만~수천만 건 주문
핵심:   재고 정합성 + 빠른 검색
```

### 게임

```
실시간: 매칭, 랭킹, 채팅, 인앱 결제
배치:   일간/주간 통계, 어뷰징 탐지, 보상 지급
규모:   일 수억 건 게임 로그
핵심:   낮은 지연 + 대량 로그 처리
```

### 광고

```
실시간: 입찰(RTB), 클릭 추적, 전환 추적
배치:   광고비 정산, 리포트, ML 모델 학습
규모:   일 수십억 건 노출/클릭 로그
핵심:   100ms 안에 입찰 완료 + 대량 로그 집계
```

&nbsp;

---

&nbsp;

## 8. 전환 포인트 — 언제 기술을 도입하는가

&nbsp;

서비스가 성장하면서 기술 도입 시점이 온다. 아래는 전형적인 전환 포인트다.

&nbsp;

```
[Phase 1] 일 10만 건 이하
├── 단일 DB (PostgreSQL/MySQL)
├── 크론탭 + 쉘 스크립트
├── Spring @Scheduled
└── "이게 배치인가?" 수준

[Phase 2] 일 100만 건
├── Spring Batch 도입
├── DB 인덱스 튜닝 시작
├── 읽기 전용 Replica 추가
└── Redis 캐시 도입

[Phase 3] 일 1,000만 건
├── Kafka 도입 (이벤트 기반 전환)
├── Consumer Group으로 병렬 처리
├── DB 파티셔닝
└── 배치/실시간 분리 시작

[Phase 4] 일 1억 건
├── Kafka + Worker Pool (다음 편에서 다룸)
├── JDBC 배치 직접 작성 (JPA 한계)
├── Master/Slave + 샤딩 검토
└── 모니터링 체계 구축 (Grafana, Prometheus)

[Phase 5] 일 10억 건
├── Spark / Flink 도입
├── 데이터 레이크 (S3 + Parquet)
├── OLAP DB (ClickHouse, BigQuery)
└── "팀이 운영할 수 있는가?" 고민 시작
```

&nbsp;

가장 중요한 원칙이 하나 있다:

> **"현재 규모에 맞는 기술을 쓰되, 다음 단계를 준비하라."**

일 10만 건인데 Kafka를 도입하면 오버 엔지니어링이다. 하지만 일 100만 건이 되기 전에 이벤트 기반으로 전환할 준비는 해놔야 한다.

&nbsp;

---

&nbsp;

## 정리

&nbsp;

| 항목 | 배치 | 실시간 |
|:---:|:---:|:---:|
| 처리 시점 | 주기적 | 즉시 |
| 대상 | 전체 데이터 | 개별 이벤트 |
| 지연 | 분~시간 | 밀리초~초 |
| 정확도 | 높음 | 상대적으로 낮을 수 있음 |
| 장애 복구 | 재실행 | 복잡함 |
| 자원 | 실행 시에만 | 항상 |
| 복잡도 | 낮음 | 높음 |

&nbsp;

대부분의 시스템은 배치와 실시간을 **함께 사용**한다. 중요한 것은 각각의 특성을 이해하고, **비즈니스 요구사항과 데이터 규모에 맞는 조합**을 찾는 것이다.

이론적으로 완벽한 아키텍처보다, **팀이 운영할 수 있는 아키텍처**가 좋은 아키텍처다.

&nbsp;

---

&nbsp;

**다음 편 예고:**

2편에서는 배치 처리의 대표 주자 **Spring Batch**를 심화 학습한다. 100만 건에서 1,000만 건까지, Chunk 모델의 최적화 기법과 병렬 처리 전략을 실제 코드와 함께 다룬다.

&nbsp;

---

`#대규모데이터` `#배치처리` `#실시간처리` `#Lambda아키텍처` `#Kappa아키텍처` `#Kafka` `#SpringBatch` `#데이터엔지니어링` `#시스템설계`
