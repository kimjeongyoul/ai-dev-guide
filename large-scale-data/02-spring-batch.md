# 대규모 데이터 처리 시리즈 (2) — Spring Batch 심화, 100만~1,000만 건 처리의 정석

&nbsp;

1편에서 배치와 실시간의 차이를 다뤘다. 이번 편에서는 배치 처리의 대표 프레임워크인 **Spring Batch**를 깊이 파고든다.

"100만 건을 30분 안에 처리해야 합니다." 이런 요구사항을 받았을 때, Spring Batch를 **제대로** 쓰면 해결할 수 있다. 문제는 "제대로"가 무엇인지 아는 것이다.

&nbsp;

---

&nbsp;

## 1. Spring Batch 핵심 개념

&nbsp;

Spring Batch의 구조는 계층적이다.

```
┌─────────────────────────────────────────────────┐
│                    Job                          │
│  ┌─────────────┐  ┌─────────────┐              │
│  │   Step 1    │→ │   Step 2    │→ ...         │
│  │             │  │             │              │
│  │ ┌─────────┐ │  │ ┌─────────┐ │              │
│  │ │ Chunk   │ │  │ │ Tasklet │ │              │
│  │ │         │ │  │ │         │ │              │
│  │ │ Reader  │ │  │ │ 단일    │ │              │
│  │ │ Processor│ │  │ │ 작업    │ │              │
│  │ │ Writer  │ │  │ │ 실행    │ │              │
│  │ └─────────┘ │  │ └─────────┘ │              │
│  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────┘
```

&nbsp;

| 개념 | 역할 | 비유 |
|:---:|:---|:---|
| **Job** | 배치 작업 전체 | "일 매출 정산" 이라는 작업 |
| **Step** | Job 안의 단계 | "1단계: 주문 집계, 2단계: 수수료 계산" |
| **Chunk** | Step의 처리 모델 | "1,000건씩 읽고 → 가공 → 저장" |
| **Tasklet** | Step의 단순 모델 | "테이블 한 번 비우기" |
| **Reader** | 데이터 읽기 | DB, 파일, API에서 데이터 읽기 |
| **Processor** | 데이터 가공 | 비즈니스 로직 적용 |
| **Writer** | 결과 저장 | DB, 파일에 결과 쓰기 |

&nbsp;

가장 중요한 개념은 **Chunk**다. 전체 데이터를 한꺼번에 메모리에 올리지 않고, **N건씩 잘라서** 읽기 → 가공 → 쓰기를 반복한다.

```
Read(1000건) → Process(1000건) → Write(1000건) → 커밋
Read(1000건) → Process(1000건) → Write(1000건) → 커밋
Read(1000건) → Process(1000건) → Write(1000건) → 커밋
...
```

&nbsp;

---

&nbsp;

## 2. Reader / Processor / Writer 패턴

&nbsp;

실제 코드로 살펴보자. "일일 주문 데이터를 읽어서 매출 요약을 생성하는" 배치다.

&nbsp;

### Reader: 데이터 읽기

```java
@Bean
public JpaCursorItemReader<Order> orderReader(EntityManagerFactory emf) {
    return new JpaCursorItemReaderBuilder<Order>()
            .name("orderReader")
            .entityManagerFactory(emf)
            .queryString("""
                SELECT o FROM Order o
                WHERE o.orderDate = :targetDate
                  AND o.status = 'CONFIRMED'
                ORDER BY o.id
                """)
            .parameterValues(Map.of("targetDate", LocalDate.now().minusDays(1)))
            .build();
}
```

&nbsp;

### Processor: 비즈니스 로직

```java
@Bean
public ItemProcessor<Order, DailySales> salesProcessor() {
    return order -> {
        // 취소된 주문은 건너뜀 (null 반환 시 Writer로 전달 안 됨)
        if (order.isCancelled()) {
            return null;
        }

        return DailySales.builder()
                .storeId(order.getStoreId())
                .salesDate(order.getOrderDate())
                .totalAmount(order.calculateNetAmount())
                .itemCount(order.getItems().size())
                .commission(order.calculateCommission())
                .build();
    };
}
```

&nbsp;

### Writer: 결과 저장

```java
@Bean
public JpaItemWriter<DailySales> salesWriter(EntityManagerFactory emf) {
    JpaItemWriter<DailySales> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(emf);
    return writer;
}
```

&nbsp;

### 조합: Step과 Job

```java
@Bean
public Step dailySalesStep(JobRepository jobRepository,
                           PlatformTransactionManager tx) {
    return new StepBuilder("dailySalesStep", jobRepository)
            .<Order, DailySales>chunk(1000, tx)
            .reader(orderReader(null))
            .processor(salesProcessor())
            .writer(salesWriter(null))
            .faultTolerant()
            .skipLimit(100)
            .skip(DataIntegrityViolationException.class)
            .build();
}

@Bean
public Job dailySalesJob(JobRepository jobRepository,
                         Step dailySalesStep) {
    return new JobBuilder("dailySalesJob", jobRepository)
            .start(dailySalesStep)
            .build();
}
```

&nbsp;

---

&nbsp;

## 3. JpaCursorItemReader vs JpaPagingItemReader

&nbsp;

Reader 선택은 성능에 큰 영향을 미친다. 두 가지 주요 Reader의 차이를 이해해야 한다.

&nbsp;

### JpaPagingItemReader

```java
@Bean
public JpaPagingItemReader<Order> pagingReader(EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<Order>()
            .name("pagingReader")
            .entityManagerFactory(emf)
            .queryString("SELECT o FROM Order o WHERE o.status = 'CONFIRMED' ORDER BY o.id")
            .pageSize(1000)   // 1000건씩 페이지 단위로 쿼리
            .build();
}
```

**동작 방식:** `LIMIT 1000 OFFSET 0`, `LIMIT 1000 OFFSET 1000`, `LIMIT 1000 OFFSET 2000` ...

**문제점:** OFFSET이 커지면 느려진다. 100만 번째부터 읽으려면 앞의 100만 건을 스캔해야 한다.

&nbsp;

### JpaCursorItemReader

```java
@Bean
public JpaCursorItemReader<Order> cursorReader(EntityManagerFactory emf) {
    return new JpaCursorItemReaderBuilder<Order>()
            .name("cursorReader")
            .entityManagerFactory(emf)
            .queryString("SELECT o FROM Order o WHERE o.status = 'CONFIRMED' ORDER BY o.id")
            .build();
}
```

**동작 방식:** DB 커서를 열고, 한 건씩 스트리밍으로 읽음. OFFSET 없음.

&nbsp;

### 비교

| 항목 | PagingReader | CursorReader |
|:---:|:---:|:---:|
| 쿼리 방식 | 페이지마다 새 쿼리 | 커서 1개 유지 |
| OFFSET 문제 | 있음 (뒤로 갈수록 느림) | 없음 |
| DB 커넥션 | 쿼리마다 반납 | 처리 끝날 때까지 점유 |
| 스레드 안전성 | 안전 | 안전하지 않음 |
| 메모리 | 페이지 단위 | 1건씩 (낮음) |
| 추천 | 데이터 변동 가능 + 멀티스레드 | 대량 데이터 순차 처리 |

&nbsp;

**결론:**
- 100만 건 이하: PagingReader로 충분
- 100만 건 이상 순차 처리: CursorReader 추천
- 멀티스레드 필요: PagingReader (또는 Partitioning)

&nbsp;

---

&nbsp;

## 4. Chunk 사이즈 최적화

&nbsp;

chunk 사이즈는 **한 번의 트랜잭션에서 처리할 건수**다. 이 값이 성능을 크게 좌우한다.

&nbsp;

```java
// chunk(100)  → 100건 읽기 → 100건 처리 → 100건 쓰기 → 커밋
// chunk(1000) → 1000건 읽기 → 1000건 처리 → 1000건 쓰기 → 커밋
// chunk(5000) → 5000건 읽기 → 5000건 처리 → 5000건 쓰기 → 커밋
```

&nbsp;

| chunk 사이즈 | 장점 | 단점 |
|:---:|:---|:---|
| 100 | 메모리 적음, 실패 시 손실 적음 | 커밋 빈번 → 느림 |
| 1,000 | 균형 잡힌 선택 | - |
| 5,000 | 커밋 횟수 적음 → 빠름 | 메모리 많이 사용, 실패 시 5,000건 롤백 |
| 10,000+ | 이론상 최고 속도 | OOM 위험, 롤백 부담 |

&nbsp;

**실측 예시 (500만 건 처리):**

```
chunk=100   → 47분 (커밋 50,000회)
chunk=500   → 28분 (커밋 10,000회)
chunk=1,000 → 22분 (커밋 5,000회)
chunk=5,000 → 18분 (커밋 1,000회)
chunk=10,000→ 17분 (커밋 500회, OOM 간헐 발생)
```

&nbsp;

**실무 권장값:**
- 일반적인 경우: **1,000~5,000**
- 메모리가 넉넉하고 실패 허용: **5,000~10,000**
- 처리 로직이 무거운 경우 (외부 API 호출 등): **100~500**

&nbsp;

---

&nbsp;

## 5. 재시작 메커니즘

&nbsp;

Spring Batch의 가장 강력한 기능 중 하나가 **실패 지점부터 재시작**이다.

&nbsp;

```
[1차 실행]
chunk 1 (1~1000건)      ✅ 커밋
chunk 2 (1001~2000건)   ✅ 커밋
chunk 3 (2001~3000건)   ✅ 커밋
chunk 4 (3001~4000건)   ❌ 실패 (DB 커넥션 끊김)
→ JobExecution: FAILED, StepExecution: FAILED (readCount=3000)

[2차 실행 — 재시작]
chunk 4 (3001~4000건)   ✅ 커밋  ← 3001건부터 재시작!
chunk 5 (4001~5000건)   ✅ 커밋
...
→ JobExecution: COMPLETED
```

&nbsp;

이게 가능한 이유는 Spring Batch가 **메타데이터 테이블**에 실행 상태를 기록하기 때문이다.

```sql
-- Spring Batch가 자동 생성하는 테이블들
BATCH_JOB_INSTANCE        -- Job 정의
BATCH_JOB_EXECUTION       -- Job 실행 이력
BATCH_JOB_EXECUTION_PARAMS -- Job 파라미터
BATCH_STEP_EXECUTION      -- Step 실행 이력 (readCount, writeCount 등)
BATCH_STEP_EXECUTION_CONTEXT -- Step 컨텍스트 (재시작 정보)
```

&nbsp;

재시작이 동작하려면 주의할 점:

```java
@Bean
public Step restartableStep(JobRepository jobRepository,
                            PlatformTransactionManager tx) {
    return new StepBuilder("restartableStep", jobRepository)
            .<Order, DailySales>chunk(1000, tx)
            .reader(orderReader(null))
            .processor(salesProcessor())
            .writer(salesWriter(null))
            .allowStartIfComplete(false)  // 완료된 Step은 재실행 안 함
            .build();
}

// Job 파라미터에 날짜를 넣어야 같은 Job을 다른 날 실행 가능
@Bean
public Job dailySalesJob(JobRepository jobRepository, Step step) {
    return new JobBuilder("dailySalesJob", jobRepository)
            .start(step)
            .incrementer(new RunIdIncrementer())  // 자동 ID 증가
            .build();
}
```

&nbsp;

---

&nbsp;

## 6. 병렬 Step: 멀티스레드 처리

&nbsp;

단일 스레드로 부족하면 **TaskExecutor**를 연결해서 멀티스레드로 전환할 수 있다.

&nbsp;

```java
@Bean
public Step multiThreadStep(JobRepository jobRepository,
                            PlatformTransactionManager tx) {
    return new StepBuilder("multiThreadStep", jobRepository)
            .<Order, DailySales>chunk(1000, tx)
            .reader(pagingReader(null))        // 주의: CursorReader는 스레드 안전하지 않음!
            .processor(salesProcessor())
            .writer(salesWriter(null))
            .taskExecutor(batchTaskExecutor())  // 멀티스레드 적용
            .throttleLimit(4)                   // 동시 스레드 수 제한
            .build();
}

@Bean
public TaskExecutor batchTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(25);
    executor.setThreadNamePrefix("batch-");
    executor.initialize();
    return executor;
}
```

&nbsp;

**주의사항:**
- `JpaCursorItemReader`는 스레드 안전하지 않다 → `JpaPagingItemReader` 사용
- `SynchronizedItemStreamReader`로 감싸면 CursorReader도 멀티스레드 가능 (단, 성능 이점 감소)
- Writer에서 동시 INSERT 충돌 주의 → unique 제약조건 확인

&nbsp;

```java
// CursorReader를 멀티스레드에서 쓰고 싶다면
@Bean
public SynchronizedItemStreamReader<Order> synchronizedReader() {
    SynchronizedItemStreamReader<Order> reader = new SynchronizedItemStreamReader<>();
    reader.setDelegate(cursorReader(null));
    return reader;
}
```

&nbsp;

---

&nbsp;

## 7. Partitioning: 범위별 병렬 처리

&nbsp;

멀티스레드 Step보다 더 강력한 방법이 **Partitioning**이다. 데이터를 범위별로 나눠서 **독립적인 Step 인스턴스**로 실행한다.

&nbsp;

```
                    ┌─────────────────────┐
                    │   Master Step       │
                    │   (Partitioner)     │
                    │                     │
                    │  ID 1~250,000       │
                    │  ID 250,001~500,000 │
                    │  ID 500,001~750,000 │
                    │  ID 750,001~1,000,000│
                    └──────────┬──────────┘
                               │
              ┌────────┬───────┼───────┬────────┐
              ▼        ▼       ▼       ▼        ▼
         ┌────────┐┌────────┐┌────────┐┌────────┐
         │Worker 1││Worker 2││Worker 3││Worker 4│
         │1~250K  ││250K~   ││500K~   ││750K~   │
         │        ││500K    ││750K    ││1M      │
         └────────┘└────────┘└────────┘└────────┘
```

&nbsp;

### Partitioner 구현

```java
@Component
public class OrderIdPartitioner implements Partitioner {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Long minId = jdbcTemplate.queryForObject(
            "SELECT MIN(id) FROM orders WHERE order_date = CURRENT_DATE - 1", Long.class);
        Long maxId = jdbcTemplate.queryForObject(
            "SELECT MAX(id) FROM orders WHERE order_date = CURRENT_DATE - 1", Long.class);

        long range = (maxId - minId) / gridSize + 1;
        Map<String, ExecutionContext> partitions = new HashMap<>();

        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            long start = minId + (range * i);
            long end = Math.min(start + range - 1, maxId);

            ctx.putLong("minId", start);
            ctx.putLong("maxId", end);
            partitions.put("partition" + i, ctx);
        }
        return partitions;
    }
}
```

&nbsp;

### Partition Step 설정

```java
@Bean
public Step partitionedStep(JobRepository jobRepository,
                            PlatformTransactionManager tx) {
    return new StepBuilder("partitionedStep", jobRepository)
            .partitioner("workerStep", orderIdPartitioner)
            .step(workerStep(jobRepository, tx))
            .gridSize(8)                       // 8개 파티션
            .taskExecutor(batchTaskExecutor())  // 8개 동시 실행
            .build();
}

@Bean
public Step workerStep(JobRepository jobRepository,
                       PlatformTransactionManager tx) {
    return new StepBuilder("workerStep", jobRepository)
            .<Order, DailySales>chunk(1000, tx)
            .reader(partitionedReader(null))   // @StepScope로 범위 주입
            .processor(salesProcessor())
            .writer(salesWriter(null))
            .build();
}

@Bean
@StepScope
public JpaCursorItemReader<Order> partitionedReader(
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId,
        EntityManagerFactory emf) {
    return new JpaCursorItemReaderBuilder<Order>()
            .name("partitionedReader")
            .entityManagerFactory(emf)
            .queryString("""
                SELECT o FROM Order o
                WHERE o.id BETWEEN :minId AND :maxId
                  AND o.status = 'CONFIRMED'
                ORDER BY o.id
                """)
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .build();
}
```

&nbsp;

**Partitioning의 장점:**
- 각 Worker가 독립적이라 스레드 안전 문제 없음
- CursorReader 사용 가능 (각 Worker가 자기 커서를 가짐)
- 파티션 단위로 재시작 가능 (실패한 파티션만 재실행)

&nbsp;

---

&nbsp;

## 8. 실무 팁

&nbsp;

### flush/clear로 메모리 관리

JPA를 쓰면 영속성 컨텍스트에 엔티티가 계속 쌓인다. 명시적으로 비워줘야 한다.

```java
@Bean
public ItemWriter<DailySales> memoryAwareWriter(EntityManagerFactory emf) {
    return items -> {
        EntityManager em = emf.createEntityManager();
        EntityTransaction tx = em.getTransaction();
        tx.begin();

        for (DailySales sale : items) {
            em.persist(sale);
        }

        em.flush();   // DB에 반영
        em.clear();   // 영속성 컨텍스트 비우기 (메모리 해제)
        tx.commit();
        em.close();
    };
}
```

&nbsp;

### 배치 전용 DataSource 분리

배치가 메인 DB 커넥션을 다 쓰면 서비스에 영향이 간다.

```yaml
spring:
  # 서비스용 DataSource
  datasource:
    url: jdbc:mysql://main-db:3306/service
    hikari:
      maximum-pool-size: 30

  # 배치용 DataSource (별도 설정)
  batch:
    datasource:
      url: jdbc:mysql://replica-db:3306/service
      hikari:
        maximum-pool-size: 10
```

```java
@Configuration
public class BatchDataSourceConfig {

    @Bean("batchDataSource")
    @ConfigurationProperties("spring.batch.datasource")
    public DataSource batchDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public JpaCursorItemReader<Order> orderReader(
            @Qualifier("batchDataSource") DataSource batchDs) {
        // Replica DB에서 읽기
    }
}
```

&nbsp;

### 실행 스케줄링

```java
@Component
@EnableScheduling
public class BatchScheduler {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job dailySalesJob;

    // 매일 새벽 2시 실행
    @Scheduled(cron = "0 0 2 * * *")
    public void runDailySalesJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addLocalDate("targetDate", LocalDate.now().minusDays(1))
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters();

        JobExecution execution = jobLauncher.run(dailySalesJob, params);
        log.info("Job 완료: status={}, duration={}s",
                execution.getStatus(),
                Duration.between(execution.getStartTime(), execution.getEndTime()).getSeconds());
    }
}
```

&nbsp;

---

&nbsp;

## 9. 모니터링

&nbsp;

Spring Batch는 실행 이력을 메타데이터 테이블에 자동 저장한다.

```sql
-- 최근 Job 실행 이력 확인
SELECT
    ji.JOB_NAME,
    je.STATUS,
    je.START_TIME,
    je.END_TIME,
    DATEDIFF(SECOND, je.START_TIME, je.END_TIME) AS duration_sec
FROM BATCH_JOB_EXECUTION je
JOIN BATCH_JOB_INSTANCE ji ON je.JOB_INSTANCE_ID = ji.JOB_INSTANCE_ID
ORDER BY je.START_TIME DESC;

-- Step별 처리 건수 확인
SELECT
    se.STEP_NAME,
    se.STATUS,
    se.READ_COUNT,
    se.WRITE_COUNT,
    se.SKIP_COUNT,
    se.ROLLBACK_COUNT,
    se.START_TIME,
    se.END_TIME
FROM BATCH_STEP_EXECUTION se
WHERE se.JOB_EXECUTION_ID = ?
ORDER BY se.STEP_EXECUTION_ID;
```

&nbsp;

**Actuator 연동으로 모니터링 자동화:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, batch
  metrics:
    tags:
      application: batch-service
```

```java
// Prometheus + Grafana로 배치 메트릭 수집
// spring_batch_job_duration_seconds
// spring_batch_step_read_count
// spring_batch_step_write_count
```

&nbsp;

---

&nbsp;

## 정리

&nbsp;

| 기법 | 적용 시점 | 성능 향상 |
|:---:|:---|:---:|
| Chunk 최적화 | 항상 | 1.5~3x |
| CursorReader | 100만 건 이상 순차 | 2~5x |
| 멀티스레드 Step | 500만 건 이상 | 3~8x |
| Partitioning | 1,000만 건 이상 | 5~10x |
| JDBC 직접 사용 | 1,000만 건 이상 쓰기 | 10x+ |
| DataSource 분리 | 서비스 영향 우려 시 | - |

&nbsp;

Spring Batch는 1,000만 건까지의 배치 처리를 안정적으로 해결한다. 하지만 그 이상이 되면 한계가 온다. DB 한 대로는 부족하고, 처리 노드도 여러 대가 필요해진다.

&nbsp;

---

&nbsp;

**다음 편 예고:**

3편에서는 대규모 처리의 핵심 인프라인 **Apache Kafka**를 기초부터 다룬다. 메시지 큐가 왜 필요한지, Kafka는 왜 빠른지, Spring Kafka로 Producer/Consumer를 어떻게 구현하는지 코드와 함께 살펴본다.

&nbsp;

---

`#SpringBatch` `#대규모데이터` `#배치처리` `#Chunk` `#Partitioning` `#JPA` `#Java` `#데이터엔지니어링` `#Spring`
