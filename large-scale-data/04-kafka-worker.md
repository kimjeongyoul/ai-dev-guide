# 대규모 데이터 처리 시리즈 (4) — Kafka + Worker 패턴, 1억 건을 병렬로 처리하기

&nbsp;

3편에서 Kafka의 기초를 다뤘다. 이번 편에서는 Kafka를 **실전에서 어떻게 쓰는지** 다룬다.

핵심 패턴은 간단하다. **Scheduler가 작업을 분할하고, Kafka에 넣고, Worker Pool이 병렬로 처리한다.** 이 패턴 하나로 1억 건도 처리할 수 있다.

&nbsp;

---

&nbsp;

## 1. 아키텍처

&nbsp;

```
┌──────────────┐
│  Scheduler   │  "오늘 처리할 데이터를 분할해서 Kafka에 넣는다"
│  (Spring)    │
└──────┬───────┘
       │ 작업 메시지 발행
       │ (ID 범위, 날짜 등)
       ▼
┌──────────────────────────────────────────────┐
│                 Kafka                        │
│                                              │
│  Topic: settlement-tasks (partition 12)      │
│  [0~100K] [100K~200K] [200K~300K] ...       │
└──────┬───────┬───────┬───────┬──────────────┘
       │       │       │       │
       ▼       ▼       ▼       ▼
  ┌────────┐┌────────┐┌────────┐┌────────┐
  │Worker 1││Worker 2││Worker 3││Worker 4│
  │        ││        ││        ││        │
  │ Kafka  ││ Kafka  ││ Kafka  ││ Kafka  │
  │Listener││Listener││Listener││Listener│
  │   +    ││   +    ││   +    ││   +    │
  │ JDBC   ││ JDBC   ││ JDBC   ││ JDBC   │
  │ Batch  ││ Batch  ││ Batch  ││ Batch  │
  └────┬───┘└────┬───┘└────┬───┘└────┬───┘
       │         │         │         │
       ▼         ▼         ▼         ▼
  ┌────────────────────────────────────────┐
  │              Database                  │
  └────────────────────────────────────────┘
```

&nbsp;

**왜 이 구조인가?**

- Spring Batch 단일 노드: 1,000만 건까지 → 그 이상은 느림
- Kafka + Worker: 수평 확장 가능 → Worker를 늘리면 처리량도 늘어남
- Worker가 독립적이라 한 Worker가 죽어도 다른 Worker에 영향 없음

&nbsp;

---

&nbsp;

## 2. 작업 분할 전략

&nbsp;

1억 건을 어떻게 나눌 것인가? 전략은 세 가지다.

&nbsp;

### ID 범위로 분할

```java
@Component
@RequiredArgsConstructor
public class SettlementScheduler {

    private final KafkaTemplate<String, TaskMessage> kafkaTemplate;
    private final JdbcTemplate jdbcTemplate;

    @Scheduled(cron = "0 0 2 * * *")  // 매일 새벽 2시
    public void scheduleSettlement() {
        // 1. 처리 대상의 ID 범위를 조회
        Long minId = jdbcTemplate.queryForObject(
            "SELECT MIN(id) FROM orders WHERE order_date = CURRENT_DATE - 1 AND status = 'CONFIRMED'",
            Long.class);
        Long maxId = jdbcTemplate.queryForObject(
            "SELECT MAX(id) FROM orders WHERE order_date = CURRENT_DATE - 1 AND status = 'CONFIRMED'",
            Long.class);

        if (minId == null || maxId == null) {
            log.info("처리 대상 없음");
            return;
        }

        // 2. 10만 건 단위로 분할하여 Kafka에 발행
        long chunkSize = 100_000;
        int taskCount = 0;

        for (long start = minId; start <= maxId; start += chunkSize) {
            long end = Math.min(start + chunkSize - 1, maxId);
            TaskMessage task = TaskMessage.builder()
                    .taskId(UUID.randomUUID().toString())
                    .targetDate(LocalDate.now().minusDays(1))
                    .startId(start)
                    .endId(end)
                    .build();

            kafkaTemplate.send("settlement-tasks", task.getTaskId(), task);
            taskCount++;
        }

        log.info("정산 작업 발행 완료: {} tasks, ID range [{} ~ {}]",
                taskCount, minId, maxId);
    }
}
```

&nbsp;

### 날짜로 분할

```java
// 월간 정산: 30일을 1일씩 분할
for (int day = 1; day <= 30; day++) {
    LocalDate targetDate = YearMonth.now().minusMonths(1).atDay(day);
    TaskMessage task = TaskMessage.builder()
            .taskId(UUID.randomUUID().toString())
            .targetDate(targetDate)
            .build();
    kafkaTemplate.send("monthly-settlement", task.getTaskId(), task);
}
```

&nbsp;

### 해시로 분할

```java
// 가맹점 ID를 해시로 분할 (균등 분배)
for (String storeId : allStoreIds) {
    // key를 storeId로 하면 Kafka가 해시로 파티션 분배
    kafkaTemplate.send("store-settlement", storeId, task);
}
```

&nbsp;

**분할 전략 비교:**

| 전략 | 장점 | 단점 | 적합한 경우 |
|:---:|:---|:---|:---|
| ID 범위 | 구현 간단, 균등 | 삭제된 ID가 있으면 불균등 | 연속 ID 데이터 |
| 날짜 | 의미 있는 단위 | 날짜별 데이터량이 다름 | 시계열 데이터 |
| 해시 | 자동 균등 분배 | 범위 조회 불가 | 키 기반 데이터 |

&nbsp;

---

&nbsp;

## 3. Worker 코드 예시

&nbsp;

### Kafka Consumer + JDBC 배치

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class SettlementWorker {

    private final JdbcTemplate jdbcTemplate;

    @KafkaListener(
        topics = "settlement-tasks",
        groupId = "settlement-worker",
        concurrency = "12"          // 12개 파티션에 맞춤
    )
    public void processTask(TaskMessage task, Acknowledgment ack) {
        long startTime = System.currentTimeMillis();
        log.info("작업 시작: taskId={}, range=[{} ~ {}]",
                task.getTaskId(), task.getStartId(), task.getEndId());

        try {
            // 1. 대상 데이터 조회
            List<Order> orders = fetchOrders(task.getStartId(), task.getEndId());

            // 2. 정산 계산
            List<Settlement> settlements = calculateSettlements(orders);

            // 3. JDBC 배치로 저장
            batchInsert(settlements);

            // 4. 성공 → offset 커밋
            ack.acknowledge();

            log.info("작업 완료: taskId={}, 건수={}, 소요={}ms",
                    task.getTaskId(), settlements.size(),
                    System.currentTimeMillis() - startTime);

        } catch (Exception e) {
            log.error("작업 실패: taskId={}", task.getTaskId(), e);
            // ack 안 하면 재처리됨
        }
    }

    private List<Order> fetchOrders(long startId, long endId) {
        return jdbcTemplate.query(
            """
            SELECT id, store_id, order_date, amount, commission_rate, status
            FROM orders
            WHERE id BETWEEN ? AND ?
              AND status = 'CONFIRMED'
            ORDER BY id
            """,
            new BeanPropertyRowMapper<>(Order.class),
            startId, endId);
    }

    private List<Settlement> calculateSettlements(List<Order> orders) {
        return orders.stream()
                .collect(Collectors.groupingBy(Order::getStoreId))
                .entrySet().stream()
                .map(entry -> {
                    String storeId = entry.getKey();
                    List<Order> storeOrders = entry.getValue();

                    long totalAmount = storeOrders.stream()
                            .mapToLong(Order::getAmount).sum();
                    long commission = storeOrders.stream()
                            .mapToLong(o -> (long)(o.getAmount() * o.getCommissionRate())).sum();

                    return Settlement.builder()
                            .storeId(storeId)
                            .settlementDate(LocalDate.now())
                            .totalAmount(totalAmount)
                            .commission(commission)
                            .netAmount(totalAmount - commission)
                            .orderCount(storeOrders.size())
                            .build();
                })
                .toList();
    }
}
```

&nbsp;

---

&nbsp;

## 4. JPA vs JDBC: 1억 건에서 JPA를 쓰면 안 되는 이유

&nbsp;

JPA는 훌륭한 기술이지만 **대량 처리에는 적합하지 않다.**

&nbsp;

### JPA로 1만 건 INSERT

```java
// JPA: 엔티티 하나씩 persist → 1만 건에 약 8초
for (Settlement settlement : settlements) {
    entityManager.persist(settlement);
}
entityManager.flush();
entityManager.clear();
```

&nbsp;

### JDBC batchUpdate로 1만 건 INSERT

```java
// JDBC batchUpdate: 1만 건에 약 0.3초 (27배 빠름)
jdbcTemplate.batchUpdate(
    """
    INSERT INTO settlement (store_id, settlement_date, total_amount, commission, net_amount, order_count)
    VALUES (?, ?, ?, ?, ?, ?)
    """,
    new BatchPreparedStatementSetter() {
        @Override
        public void setValues(PreparedStatement ps, int i) throws SQLException {
            Settlement s = settlements.get(i);
            ps.setString(1, s.getStoreId());
            ps.setDate(2, Date.valueOf(s.getSettlementDate()));
            ps.setLong(3, s.getTotalAmount());
            ps.setLong(4, s.getCommission());
            ps.setLong(5, s.getNetAmount());
            ps.setInt(6, s.getOrderCount());
        }

        @Override
        public int getBatchSize() {
            return settlements.size();
        }
    }
);
```

&nbsp;

### 왜 JPA가 느린가?

```
JPA가 하는 일 (건당):
1. 엔티티 상태 확인 (NEW? MANAGED? DETACHED?)
2. 영속성 컨텍스트에 등록
3. 변경 감지 (Dirty Checking)
4. SQL 생성
5. 1차 캐시에 저장

JDBC가 하는 일:
1. SQL 실행

JPA는 "편리함"을 위해 많은 일을 한다.
대량 처리에서는 그 "편리함"이 곧 "오버헤드"가 된다.
```

&nbsp;

### 성능 비교

| 방식 | 1만 건 | 10만 건 | 100만 건 |
|:---:|:---:|:---:|:---:|
| JPA (건별) | 8s | 90s | OOM |
| JPA (flush/clear) | 5s | 50s | 500s |
| JDBC batchUpdate | 0.3s | 2s | 18s |
| JDBC + rewriteBatchedStatements | 0.2s | 1.2s | 10s |

&nbsp;

```yaml
# MySQL에서 JDBC 배치 성능을 극대화하는 설정
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?rewriteBatchedStatements=true
    # rewriteBatchedStatements=true:
    # INSERT INTO t VALUES (1), (2), (3) 형태로 묶어서 전송
```

&nbsp;

### 5,000건씩 묶어서 처리

```java
private void batchInsert(List<Settlement> settlements) {
    int batchSize = 5000;

    for (int i = 0; i < settlements.size(); i += batchSize) {
        List<Settlement> batch = settlements.subList(
            i, Math.min(i + batchSize, settlements.size()));

        jdbcTemplate.batchUpdate(
            "INSERT INTO settlement (store_id, settlement_date, total_amount, commission, net_amount, order_count) " +
            "VALUES (?, ?, ?, ?, ?, ?)",
            new BatchPreparedStatementSetter() {
                @Override
                public void setValues(PreparedStatement ps, int idx) throws SQLException {
                    Settlement s = batch.get(idx);
                    ps.setString(1, s.getStoreId());
                    ps.setDate(2, Date.valueOf(s.getSettlementDate()));
                    ps.setLong(3, s.getTotalAmount());
                    ps.setLong(4, s.getCommission());
                    ps.setLong(5, s.getNetAmount());
                    ps.setInt(6, s.getOrderCount());
                }

                @Override
                public int getBatchSize() {
                    return batch.size();
                }
            }
        );
        log.debug("배치 INSERT 완료: {}/{}", Math.min(i + batchSize, settlements.size()), settlements.size());
    }
}
```

&nbsp;

---

&nbsp;

## 5. 멱등성 (Idempotency)

&nbsp;

Kafka에서 같은 메시지가 **2번 전달**될 수 있다. 네트워크 문제, Consumer 재시작, 리밸런싱 등 다양한 이유로.

> **멱등성이란:** 같은 작업을 2번, 3번 수행해도 결과가 동일한 성질.

&nbsp;

### 멱등하지 않은 코드 (위험)

```java
// 같은 메시지가 2번 오면 금액이 2배로 잡힘!
@KafkaListener(topics = "settlement-tasks")
public void process(TaskMessage task) {
    jdbcTemplate.update(
        "UPDATE store_balance SET balance = balance + ? WHERE store_id = ?",
        task.getAmount(), task.getStoreId());
}
```

&nbsp;

### 멱등한 코드 (안전)

```java
// 방법 1: UPSERT (INSERT ... ON DUPLICATE KEY UPDATE)
@KafkaListener(topics = "settlement-tasks")
public void process(TaskMessage task) {
    jdbcTemplate.update("""
        INSERT INTO settlement (task_id, store_id, amount, settlement_date)
        VALUES (?, ?, ?, ?)
        ON DUPLICATE KEY UPDATE
            amount = VALUES(amount),
            updated_at = NOW()
        """,
        task.getTaskId(), task.getStoreId(), task.getAmount(), task.getTargetDate());
}

// 방법 2: 처리 여부 확인
@KafkaListener(topics = "settlement-tasks")
public void process(TaskMessage task) {
    // 이미 처리된 task인지 확인
    boolean alreadyProcessed = jdbcTemplate.queryForObject(
        "SELECT COUNT(*) FROM processed_tasks WHERE task_id = ?",
        Integer.class, task.getTaskId()) > 0;

    if (alreadyProcessed) {
        log.info("이미 처리된 작업: taskId={}", task.getTaskId());
        return;
    }

    // 처리 + 처리 기록을 하나의 트랜잭션으로
    transactionTemplate.execute(status -> {
        jdbcTemplate.update("INSERT INTO settlement ...");
        jdbcTemplate.update("INSERT INTO processed_tasks (task_id) VALUES (?)", task.getTaskId());
        return null;
    });
}
```

&nbsp;

---

&nbsp;

## 6. Dead Letter Queue (DLQ)

&nbsp;

처리에 실패한 메시지를 무한 재시도하면 안 된다. **Dead Letter Queue**에 격리해야 한다.

&nbsp;

```
정상 흐름:
[settlement-tasks] → Worker → DB  ✅

실패 흐름:
[settlement-tasks] → Worker → 3회 실패 → [settlement-tasks.DLT] (격리)
                                          ↑
                                  나중에 원인 파악 후 재처리
```

&nbsp;

### Spring Kafka DLQ 설정

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        // DLT(Dead Letter Topic)로 보내기
        DeadLetterPublishingRecoverer recoverer =
                new DeadLetterPublishingRecoverer(kafkaTemplate);

        // 3회 재시도 후 DLT로 이동
        DefaultErrorHandler handler = new DefaultErrorHandler(
                recoverer,
                new FixedBackOff(1000L, 3)  // 1초 간격, 3회 재시도
        );

        // 재시도하면 안 되는 예외는 바로 DLT로
        handler.addNotRetryableExceptions(
                IllegalArgumentException.class,
                DataIntegrityViolationException.class
        );

        return handler;
    }
}
```

&nbsp;

### DLT 메시지 모니터링

```java
@Service
@Slf4j
public class DltConsumer {

    @KafkaListener(topics = "settlement-tasks.DLT", groupId = "dlt-monitor")
    public void handleDlt(ConsumerRecord<String, TaskMessage> record) {
        log.error("DLT 메시지 수신: key={}, value={}, originalTopic={}, exception={}",
                record.key(),
                record.value(),
                record.headers().lastHeader("kafka_dlt-original-topic"),
                record.headers().lastHeader("kafka_dlt-exception-message"));

        // Slack 알림, 관리자 통보 등
        alertService.sendDltAlert(record);
    }
}
```

&nbsp;

---

&nbsp;

## 7. 모니터링

&nbsp;

### Consumer Lag

Consumer Lag은 **아직 처리하지 못한 메시지 수**다. 이게 계속 쌓이면 처리 속도가 느리다는 의미.

```bash
# Consumer Lag 확인
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group settlement-worker \
  --describe

# 출력 예시:
# TOPIC              PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# settlement-tasks   0          45230           45235           5
# settlement-tasks   1          44890           52100           7210  ← 문제!
```

&nbsp;

### Prometheus + Grafana 연동

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus, health
  metrics:
    tags:
      application: settlement-worker

# Kafka Consumer 메트릭 자동 수집:
# kafka_consumer_records_consumed_total
# kafka_consumer_records_lag
# kafka_consumer_fetch_manager_records_per_request_avg
```

&nbsp;

### 처리 속도 커스텀 메트릭

```java
@Service
@RequiredArgsConstructor
public class SettlementWorker {

    private final MeterRegistry meterRegistry;

    @KafkaListener(topics = "settlement-tasks")
    public void processTask(TaskMessage task, Acknowledgment ack) {
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            // 처리 로직...
            ack.acknowledge();

            meterRegistry.counter("settlement.processed",
                    "status", "success").increment();
        } catch (Exception e) {
            meterRegistry.counter("settlement.processed",
                    "status", "failure").increment();
        } finally {
            sample.stop(meterRegistry.timer("settlement.duration"));
        }
    }
}
```

&nbsp;

---

&nbsp;

## 8. 실패 복구 전략

&nbsp;

### 재시도 with 백오프

```java
@Bean
public DefaultErrorHandler errorHandler() {
    // 지수 백오프: 1초 → 2초 → 4초 → 8초 (최대 4회)
    ExponentialBackOff backOff = new ExponentialBackOff(1000L, 2.0);
    backOff.setMaxElapsedTime(30000L);  // 최대 30초

    return new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate),
            backOff
    );
}
```

&nbsp;

### 서킷 브레이커

```java
@Service
public class SettlementWorker {

    // Resilience4j 서킷 브레이커
    @CircuitBreaker(name = "db-write", fallbackMethod = "fallback")
    @KafkaListener(topics = "settlement-tasks")
    public void processTask(TaskMessage task, Acknowledgment ack) {
        batchInsert(task);
        ack.acknowledge();
    }

    public void fallback(TaskMessage task, Acknowledgment ack, Exception e) {
        log.error("서킷 브레이커 OPEN — DB 응답 없음. 메시지를 DLT로 이동");
        // DLT로 수동 전송
        kafkaTemplate.send("settlement-tasks.DLT", task.getTaskId(), task);
        ack.acknowledge();
    }
}
```

&nbsp;

---

&nbsp;

## 9. 실무 예시: 주문 정산

&nbsp;

전체 흐름을 하나로 정리하면 이렇다.

```
[매일 새벽 2시]
Scheduler
  ├── 어제 주문 ID 범위 조회: 1 ~ 3,200,000
  ├── 10만 건씩 분할: 32개 Task 메시지
  └── Kafka "settlement-tasks" 토픽에 발행

[Worker Pool — 12 인스턴스]
Worker x 12
  ├── 각자 Kafka에서 Task를 꺼냄
  ├── 해당 범위의 주문을 JDBC로 조회
  ├── 가맹점별 정산 금액 계산
  ├── JDBC batchUpdate로 정산 테이블에 저장
  ├── 성공 시 offset 커밋
  └── 실패 시 3회 재시도 → DLT 격리

[처리 결과]
  ├── 320만 건 주문 → 32개 Task → 12 Worker → 약 3분 완료
  ├── Consumer Lag: 0 (전부 처리됨)
  └── DLT: 0건 (실패 없음)
```

&nbsp;

**성능 비교:**

| 방식 | 320만 건 처리 시간 |
|:---:|:---:|
| Spring Batch 단일 스레드 | ~45분 |
| Spring Batch Partitioning (8 파티션) | ~8분 |
| Kafka + Worker 12 인스턴스 | ~3분 |
| Kafka + Worker 24 인스턴스 | ~2분 |

&nbsp;

---

&nbsp;

## 정리

&nbsp;

| 개념 | 핵심 |
|:---:|:---|
| 작업 분할 | Scheduler가 대량 데이터를 작은 Task로 분할 |
| 병렬 처리 | Kafka 파티션 수 = Worker 수 = 동시 처리 수 |
| JDBC 배치 | JPA 대신 JDBC batchUpdate로 10배 이상 빠르게 |
| 멱등성 | UPSERT 또는 처리 기록으로 중복 방지 |
| DLQ | 실패 메시지 격리 후 나중에 재처리 |
| 모니터링 | Consumer Lag으로 처리 상태 실시간 확인 |

&nbsp;

이 패턴의 장점은 **수평 확장이 쉽다**는 것이다. 처리가 느리면 Worker를 늘리면 된다. 파티션도 늘리면 된다. 코드를 바꿀 필요가 없다.

&nbsp;

---

&nbsp;

**다음 편 예고:**

5편에서는 대량 데이터의 근본인 **DB 최적화**를 다룬다. EXPLAIN 읽는 법, 인덱스 전략, 파티셔닝, 커서 기반 페이지네이션, N+1 해결까지. Worker가 DB에 접근할 때 알아야 할 모든 것을 정리한다.

&nbsp;

---

`#Kafka` `#Worker패턴` `#대규모데이터` `#JDBC` `#배치처리` `#멱등성` `#DeadLetterQueue` `#분산처리` `#데이터엔지니어링`
