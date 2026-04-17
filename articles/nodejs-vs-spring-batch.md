# Node.js vs Spring Batch — 배치 처리, 뭘로 할까

&nbsp;

배치 처리를 해야 한다. 매일 정산을 돌리든, 로그를 집계하든, 리포트를 생성하든. 문제는 도구 선택이다.

Java 진영에는 Spring Batch라는 검증된 프레임워크가 있다. Node.js 진영에는... 딱히 없다. 그래서 직접 만든다. 이게 장점이 될 수도, 지옥이 될 수도 있다.

이 글에서는 **같은 배치 문제를 양쪽 기술로 풀어보고**, 규모별로 어떤 선택이 합리적인지 정리한다.

&nbsp;

---

&nbsp;

## 1. 배치 처리가 필요한 순간

&nbsp;

배치 처리는 어디에나 있다.

- 매일 자정: 일 매출 정산
- 매시간: 로그 파일 파싱 → 통계 테이블 적재
- 매주 월요일: 주간 리포트 생성 및 이메일 발송
- 분기 1회: 대규모 데이터 마이그레이션
- 수시: CSV 파일 업로드 → DB 벌크 INSERT

&nbsp;

"그거 cron에 스크립트 하나 걸어두면 되지 않나?"

10만 건까지는 맞다. 하지만 규모가 커지면 이런 문제가 터진다.

```
1. 메모리 부족 — 전체 데이터를 한 번에 읽으려다 OOM
2. 중간 실패 — 70만 건 처리 후 에러, 처음부터 다시?
3. 속도 부족 — 단일 프로세스로 100만 건 = 몇 시간
4. 모니터링 없음 — 지금 어디까지 처리됐는지 모름
5. 중복 처리 — 재실행 시 이미 처리된 건 또 처리
```

&nbsp;

이 문제들을 해결하는 것이 "배치 프레임워크"의 역할이다. Spring Batch는 이걸 프레임워크 레벨에서 제공한다. Node.js는 직접 만들어야 한다.

&nbsp;

---

&nbsp;

## 2. Node.js로 배치 처리하기

&nbsp;

Node.js에는 Spring Batch 같은 표준 배치 프레임워크가 없다. 대신 여러 라이브러리를 조합해서 만든다.

&nbsp;

### 2-1. node-cron으로 스케줄링

```typescript
import cron from 'node-cron';

// 매일 새벽 2시 실행
cron.schedule('0 2 * * *', async () => {
  console.log('[Settlement] 정산 배치 시작:', new Date().toISOString());
  try {
    await runSettlement();
    console.log('[Settlement] 정산 배치 완료');
  } catch (error) {
    console.error('[Settlement] 정산 배치 실패:', error);
    await sendAlertToSlack(error);
  }
});
```

&nbsp;

간단하다. 하지만 이건 "언제 실행할지"만 해결한다. **어떻게 대량 데이터를 처리할지**는 별개의 문제다.

&nbsp;

### 2-2. 스트림 기반 처리

Node.js의 진짜 강점은 스트림이다. 메모리에 전체 데이터를 올리지 않고, 흘려보내면서 처리한다.

```typescript
import { Transform, pipeline } from 'stream';
import { promisify } from 'util';

const pipelineAsync = promisify(pipeline);

// DB 커서 → Transform → 벌크 INSERT
async function processWithStream() {
  const cursor = db.query('SELECT * FROM orders WHERE settled = false')
    .stream(); // 커서 기반 스트림

  const transformer = new Transform({
    objectMode: true,
    transform(order, encoding, callback) {
      const settlement = {
        orderId: order.id,
        amount: order.amount,
        fee: Math.floor(order.amount * 0.03),
        netAmount: order.amount - Math.floor(order.amount * 0.03),
        settledAt: new Date(),
      };
      callback(null, settlement);
    }
  });

  const bulkWriter = new BulkInsertStream({
    tableName: 'settlements',
    batchSize: 1000, // 1,000건씩 벌크 INSERT
  });

  await pipelineAsync(cursor, transformer, bulkWriter);
}
```

&nbsp;

### 2-3. BullMQ — Redis 기반 작업 큐

규모가 커지면 단일 프로세스로는 부족하다. BullMQ를 쓰면 작업을 분할하고 여러 Worker가 병렬로 처리할 수 있다.

```typescript
import { Queue, Worker } from 'bullmq';
import Redis from 'ioredis';

const connection = new Redis({ host: '127.0.0.1', port: 6379 });

// 큐 생성
const settlementQueue = new Queue('settlement', { connection });

// 작업 분할: 10만 건씩 나눠서 큐에 등록
async function enqueueSettlementJobs() {
  const totalCount = await db.query(
    'SELECT COUNT(*) as cnt FROM orders WHERE settled = false'
  );

  const chunkSize = 100_000;
  for (let offset = 0; offset < totalCount; offset += chunkSize) {
    await settlementQueue.add('process-chunk', {
      offset,
      limit: chunkSize,
      date: new Date().toISOString(),
    });
  }

  console.log(`총 ${Math.ceil(totalCount / chunkSize)}개 작업 등록`);
}

// Worker: 각 청크를 처리
const worker = new Worker('settlement', async (job) => {
  const { offset, limit } = job.data;

  const orders = await db.query(
    'SELECT * FROM orders WHERE settled = false ORDER BY id OFFSET ? LIMIT ?',
    [offset, limit]
  );

  const settlements = orders.map(order => ({
    orderId: order.id,
    amount: order.amount,
    fee: Math.floor(order.amount * 0.03),
    netAmount: order.amount - Math.floor(order.amount * 0.03),
  }));

  await db.bulkInsert('settlements', settlements);
  await db.query(
    'UPDATE orders SET settled = true WHERE id IN (?)',
    [orders.map(o => o.id)]
  );

  return { processed: orders.length };
}, {
  connection,
  concurrency: 3, // Worker 당 3개 작업 동시 처리
});
```

&nbsp;

### 2-4. Worker Threads — CPU 작업 분리

Node.js는 싱글 스레드다. CPU 연산이 무거우면 Worker Threads로 분리해야 한다.

```typescript
// main.ts
import { Worker as ThreadWorker } from 'worker_threads';

function runInThread(data: any): Promise<any> {
  return new Promise((resolve, reject) => {
    const worker = new ThreadWorker('./settlement-worker.js', {
      workerData: data,
    });
    worker.on('message', resolve);
    worker.on('error', reject);
  });
}

// 4개 스레드로 병렬 처리
const ranges = [
  { start: 0, end: 250_000 },
  { start: 250_000, end: 500_000 },
  { start: 500_000, end: 750_000 },
  { start: 750_000, end: 1_000_000 },
];

const results = await Promise.all(
  ranges.map(range => runInThread(range))
);
console.log('전체 처리 완료:', results);
```

```typescript
// settlement-worker.js
import { parentPort, workerData } from 'worker_threads';

async function process() {
  const { start, end } = workerData;
  // 범위별 데이터 처리
  const orders = await db.query(
    'SELECT * FROM orders WHERE id >= ? AND id < ? AND settled = false',
    [start, end]
  );

  let processed = 0;
  for (let i = 0; i < orders.length; i += 1000) {
    const chunk = orders.slice(i, i + 1000);
    await processChunk(chunk);
    processed += chunk.length;
  }

  parentPort?.postMessage({ processed });
}

process();
```

&nbsp;

### 2-5. 재시작 구현

Spring Batch는 재시작이 자동이다. Node.js에서는 직접 만들어야 한다. 핵심은 **마지막 처리 ID를 저장**하는 것이다.

```typescript
// 배치 진행 상태 저장
interface BatchCheckpoint {
  jobName: string;
  lastProcessedId: number;
  processedCount: number;
  status: 'RUNNING' | 'COMPLETED' | 'FAILED';
  startedAt: Date;
  updatedAt: Date;
}

async function runSettlementWithCheckpoint() {
  // 마지막 체크포인트 조회
  let checkpoint = await db.findOne('batch_checkpoints', {
    jobName: 'daily-settlement',
    status: 'FAILED', // 실패한 이전 실행이 있으면 이어서
  });

  const lastId = checkpoint?.lastProcessedId ?? 0;

  if (checkpoint) {
    console.log(`이전 실패 지점부터 재개: ID > ${lastId}`);
  } else {
    checkpoint = await db.insert('batch_checkpoints', {
      jobName: 'daily-settlement',
      lastProcessedId: 0,
      processedCount: 0,
      status: 'RUNNING',
      startedAt: new Date(),
    });
  }

  try {
    let currentId = lastId;
    while (true) {
      const orders = await db.query(
        'SELECT * FROM orders WHERE id > ? AND settled = false ORDER BY id LIMIT 1000',
        [currentId]
      );

      if (orders.length === 0) break;

      await processAndSettle(orders);

      currentId = orders[orders.length - 1].id;
      await db.update('batch_checkpoints', checkpoint.id, {
        lastProcessedId: currentId,
        processedCount: checkpoint.processedCount + orders.length,
        updatedAt: new Date(),
      });
    }

    await db.update('batch_checkpoints', checkpoint.id, { status: 'COMPLETED' });
  } catch (error) {
    await db.update('batch_checkpoints', checkpoint.id, { status: 'FAILED' });
    throw error;
  }
}
```

&nbsp;

### 2-6. Node.js 배치의 장점과 한계

**장점:**
- 가볍다. 별도의 JVM, 프레임워크 셋업 불필요
- 풀스택 JS/TS 팀이면 백엔드와 같은 언어로 배치까지
- 스트림 처리가 자연스럽다
- npm 생태계에서 필요한 라이브러리를 조합할 수 있다

**한계:**
- 프레임워크가 없다. 재시작, 모니터링, 병렬 처리를 전부 직접 구현
- CPU 바운드 작업에 약하다
- 대규모 트랜잭션 관리가 번거롭다
- "잘 만든 배치"를 위한 코드량이 Spring Batch 대비 많다

&nbsp;

---

&nbsp;

## 3. Spring Batch로 배치 처리하기

&nbsp;

Spring Batch는 배치 처리의 **모든 문제를 이미 풀어놓은** 프레임워크다. Job, Step, Chunk라는 3계층 구조로 배치를 설계한다.

&nbsp;

### 3-1. Job / Step / Chunk 구조

```
┌───────────────────────────────────────────┐
│                  Job                      │
│  ┌─────────────┐    ┌─────────────┐       │
│  │   Step 1    │ →  │   Step 2    │       │
│  │             │    │             │       │
│  │ Reader      │    │ Tasklet     │       │
│  │ Processor   │    │ (후처리)     │       │
│  │ Writer      │    │             │       │
│  │ chunk=1000  │    │             │       │
│  └─────────────┘    └─────────────┘       │
└───────────────────────────────────────────┘
```

&nbsp;

### 3-2. Reader → Processor → Writer

같은 100만 건 정산을 Spring Batch로 구현하면 이렇다.

```java
@Configuration
@RequiredArgsConstructor
public class SettlementJobConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final EntityManagerFactory entityManagerFactory;

    @Bean
    public Job settlementJob(Step settlementStep) {
        return new JobBuilder("settlementJob", jobRepository)
                .start(settlementStep)
                .build();
    }

    @Bean
    public Step settlementStep(
            ItemReader<Order> reader,
            ItemProcessor<Order, Settlement> processor,
            ItemWriter<Settlement> writer) {
        return new StepBuilder("settlementStep", jobRepository)
                .<Order, Settlement>chunk(1000, transactionManager)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .faultTolerant()
                .retryLimit(3)
                .retry(DeadlockLoserDataAccessException.class)
                .build();
    }
}
```

&nbsp;

**Reader** — 커서 기반으로 메모리 절약:

```java
@Bean
public JpaCursorItemReader<Order> orderReader() {
    return new JpaCursorItemReaderBuilder<Order>()
            .name("orderReader")
            .entityManagerFactory(entityManagerFactory)
            .queryString("SELECT o FROM Order o WHERE o.settled = false ORDER BY o.id")
            .build();
}
```

&nbsp;

**Processor** — 비즈니스 로직:

```java
@Bean
public ItemProcessor<Order, Settlement> settlementProcessor() {
    return order -> {
        long fee = Math.round(order.getAmount() * 0.03);
        return Settlement.builder()
                .orderId(order.getId())
                .amount(order.getAmount())
                .fee(fee)
                .netAmount(order.getAmount() - fee)
                .settledAt(LocalDateTime.now())
                .build();
    };
}
```

&nbsp;

**Writer** — 벌크 저장:

```java
@Bean
public JpaItemWriter<Settlement> settlementWriter() {
    JpaItemWriter<Settlement> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(entityManagerFactory);
    return writer;
}
```

&nbsp;

### 3-3. 재시작 — 자동

Spring Batch는 `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` 테이블에 진행 상태를 자동 저장한다. 실패 후 같은 JobParameters로 재실행하면 **마지막 성공 청크 이후부터 자동으로 이어서 처리**한다.

```java
// 실패한 Job 재실행 — 코드 변경 없음
@Scheduled(cron = "0 30 2 * * *") // 2시에 실패했으면 2시 30분에 재시도
public void retryFailedJobs() {
    JobExecution lastExecution = jobExplorer
            .getLastJobExecution("settlementJob", jobParameters);

    if (lastExecution != null &&
        lastExecution.getStatus() == BatchStatus.FAILED) {
        jobLauncher.run(settlementJob, jobParameters);
        // → Step의 마지막 커밋 포인트부터 자동 재개
    }
}
```

&nbsp;

Node.js에서 30줄 넘게 작성했던 체크포인트 로직이, Spring Batch에서는 **프레임워크가 알아서 한다.**

&nbsp;

### 3-4. Partitioning — 범위별 병렬 처리

100만 건을 4개로 나눠서 병렬 처리:

```java
@Bean
public Step partitionedStep(Step workerStep) {
    return new StepBuilder("partitionedStep", jobRepository)
            .partitioner("workerStep", rangePartitioner())
            .step(workerStep)
            .gridSize(4) // 4개 파티션
            .taskExecutor(taskExecutor())
            .build();
}

@Bean
public Partitioner rangePartitioner() {
    return gridSize -> {
        long min = orderRepository.findMinId();
        long max = orderRepository.findMaxId();
        long range = (max - min) / gridSize + 1;

        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", min + (range * i));
            ctx.putLong("maxId", min + (range * (i + 1)) - 1);
            partitions.put("partition" + i, ctx);
        }
        return partitions;
    };
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(4);
    executor.setThreadNamePrefix("batch-");
    return executor;
}
```

&nbsp;

### 3-5. Spring Batch의 장점과 한계

**장점:**
- 재시작, 스킵, 재시도가 모두 내장
- Job/Step 단위 모니터링 (Spring Batch Admin, Spring Cloud Data Flow)
- Partitioning으로 병렬 처리가 선언적
- 트랜잭션 관리가 Chunk 단위로 자동
- 10년 넘게 검증된 안정성

**한계:**
- Java/Spring 생태계가 필요하다 (JVM, Gradle/Maven, Spring Boot)
- 러닝커브가 있다. Job, Step, Chunk, ExecutionContext, JobRepository...
- 간단한 배치도 보일러플레이트가 많다
- JS/TS 팀에게는 기술 스택 분리가 부담

&nbsp;

---

&nbsp;

## 4. 같은 문제를 양쪽으로 풀어보기

&nbsp;

**문제:** 매일 새벽 2시, 전날 주문 100만 건 정산. 수수료 3% 차감 후 정산 테이블에 저장. 중간 실패 시 이어서 처리.

&nbsp;

### 4-1. Node.js 버전 — 전체 코드

```typescript
import cron from 'node-cron';
import knex from 'knex';

const db = knex({
  client: 'mssql',
  connection: { host: 'localhost', user: 'sa', password: '****', database: 'shop' },
});

// 체크포인트 관리
async function getCheckpoint(jobName: string) {
  return db('batch_checkpoints')
    .where({ jobName, status: 'FAILED' })
    .first();
}

async function saveCheckpoint(id: number, data: Partial<BatchCheckpoint>) {
  return db('batch_checkpoints').where({ id }).update({ ...data, updatedAt: new Date() });
}

async function createCheckpoint(jobName: string) {
  const [row] = await db('batch_checkpoints').insert({
    jobName,
    lastProcessedId: 0,
    processedCount: 0,
    status: 'RUNNING',
    startedAt: new Date(),
    updatedAt: new Date(),
  }).returning('*');
  return row;
}

// 정산 처리 코어
async function processChunk(orders: any[]) {
  const settlements = orders.map(order => ({
    orderId: order.id,
    amount: order.amount,
    fee: Math.floor(order.amount * 0.03),
    netAmount: order.amount - Math.floor(order.amount * 0.03),
    settledAt: new Date(),
  }));

  await db.transaction(async (trx) => {
    // 벌크 INSERT (1,000건씩)
    for (let i = 0; i < settlements.length; i += 1000) {
      await trx('settlements').insert(settlements.slice(i, i + 1000));
    }
    // 원본 주문 settled 플래그 갱신
    const orderIds = orders.map(o => o.id);
    await trx('orders').whereIn('id', orderIds).update({ settled: true });
  });
}

// 메인 배치 실행
async function runSettlement() {
  let checkpoint = await getCheckpoint('daily-settlement');
  const lastId = checkpoint?.lastProcessedId ?? 0;

  if (checkpoint) {
    console.log(`[재개] ID > ${lastId} 부터`);
    await saveCheckpoint(checkpoint.id, { status: 'RUNNING' });
  } else {
    checkpoint = await createCheckpoint('daily-settlement');
  }

  let currentId = lastId;
  let totalProcessed = checkpoint.processedCount || 0;

  try {
    while (true) {
      const orders = await db('orders')
        .where('id', '>', currentId)
        .andWhere('settled', false)
        .orderBy('id')
        .limit(1000);

      if (orders.length === 0) break;

      await processChunk(orders);

      currentId = orders[orders.length - 1].id;
      totalProcessed += orders.length;

      await saveCheckpoint(checkpoint.id, {
        lastProcessedId: currentId,
        processedCount: totalProcessed,
      });

      console.log(`[진행] ${totalProcessed}건 처리됨, 마지막 ID: ${currentId}`);
    }

    await saveCheckpoint(checkpoint.id, { status: 'COMPLETED' });
    console.log(`[완료] 총 ${totalProcessed}건 정산`);
  } catch (error) {
    await saveCheckpoint(checkpoint.id, { status: 'FAILED' });
    console.error(`[실패] ${totalProcessed}건 처리 후 에러:`, error);
    throw error;
  }
}

// 스케줄 등록
cron.schedule('0 2 * * *', () => {
  runSettlement().catch(console.error);
});
```

&nbsp;

**약 90줄.** 체크포인트, 청크 처리, 에러 핸들링, 트랜잭션을 전부 직접 구현했다.

&nbsp;

### 4-2. Spring Batch 버전 — 전체 코드

```java
@Configuration
@RequiredArgsConstructor
public class DailySettlementJobConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final EntityManagerFactory entityManagerFactory;

    @Bean
    public Job dailySettlementJob(Step settlementStep) {
        return new JobBuilder("dailySettlementJob", jobRepository)
                .start(settlementStep)
                .incrementer(new RunIdIncrementer())
                .listener(new JobExecutionListenerSupport() {
                    @Override
                    public void afterJob(JobExecution execution) {
                        log.info("정산 완료: status={}, 처리건수={}",
                            execution.getStatus(),
                            execution.getStepExecutions().stream()
                                .mapToLong(StepExecution::getWriteCount)
                                .sum());
                    }
                })
                .build();
    }

    @Bean
    public Step settlementStep() {
        return new StepBuilder("settlementStep", jobRepository)
                .<Order, Settlement>chunk(1000, transactionManager)
                .reader(orderReader())
                .processor(settlementProcessor())
                .writer(settlementWriter())
                .faultTolerant()
                .retryLimit(3)
                .retry(DeadlockLoserDataAccessException.class)
                .skipLimit(100)
                .skip(DataIntegrityViolationException.class)
                .listener(new StepExecutionListener() {
                    @Override
                    public void afterStep(StepExecution stepExecution) {
                        log.info("Step 완료: read={}, write={}, skip={}",
                            stepExecution.getReadCount(),
                            stepExecution.getWriteCount(),
                            stepExecution.getSkipCount());
                    }
                })
                .build();
    }

    @Bean
    public JpaCursorItemReader<Order> orderReader() {
        return new JpaCursorItemReaderBuilder<Order>()
                .name("orderReader")
                .entityManagerFactory(entityManagerFactory)
                .queryString(
                    "SELECT o FROM Order o WHERE o.settled = false ORDER BY o.id")
                .build();
    }

    @Bean
    public ItemProcessor<Order, Settlement> settlementProcessor() {
        return order -> Settlement.builder()
                .orderId(order.getId())
                .amount(order.getAmount())
                .fee(Math.round(order.getAmount() * 0.03))
                .netAmount(order.getAmount() - Math.round(order.getAmount() * 0.03))
                .settledAt(LocalDateTime.now())
                .build();
    }

    @Bean
    public JpaItemWriter<Settlement> settlementWriter() {
        JpaItemWriter<Settlement> writer = new JpaItemWriter<>();
        writer.setEntityManagerFactory(entityManagerFactory);
        return writer;
    }
}
```

```java
// 스케줄러
@Component
@RequiredArgsConstructor
public class SettlementScheduler {

    private final JobLauncher jobLauncher;
    private final Job dailySettlementJob;

    @Scheduled(cron = "0 0 2 * * *") // 매일 새벽 2시
    public void run() throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addLocalDateTime("runDate", LocalDateTime.now())
                .toJobParameters();
        jobLauncher.run(dailySettlementJob, params);
    }
}
```

&nbsp;

**약 80줄.** 비슷한 코드 양이지만, 재시작/스킵/재시도/모니터링이 **선언적으로 내장**되어 있다. 체크포인트 테이블을 직접 만들 필요 없다.

&nbsp;

### 4-3. 비교

| 항목 | Node.js | Spring Batch |
|:---|:---|:---|
| 코드 양 | ~90줄 | ~80줄 |
| 재시작 | 직접 구현 (체크포인트 테이블) | 자동 (JobExecution) |
| 재시도 | 직접 구현 (try-catch + 루프) | `.retryLimit(3).retry(Exception.class)` |
| 스킵 | 직접 구현 | `.skipLimit(100).skip(Exception.class)` |
| 모니터링 | 직접 로깅 | StepExecution에 read/write/skip 카운트 자동 |
| 트랜잭션 | 직접 관리 (knex transaction) | Chunk 단위 자동 커밋 |
| 병렬 처리 | Worker Threads 직접 구현 | Partitioner 선언 |
| 셋업 비용 | npm install 몇 개 | Spring Boot + 의존성 + DB 스키마 |

&nbsp;

핵심 차이: Node.js는 **"다 된다, 직접 만들면"**. Spring Batch는 **"다 된다, 이미 만들어져 있으니까"**.

&nbsp;

---

&nbsp;

## 5. 규모별 선택 기준

&nbsp;

| 규모 | Node.js | Spring Batch |
|:---|:---|:---|
| **10만 건 이하** | 충분. cron + 단순 스크립트 | 과하다. 프레임워크 오버헤드만 늘어남 |
| **100만 건** | 가능. 스트림 + 청크 필수 | 편하다. Chunk 모델이 딱 맞음 |
| **1,000만 건** | 가능하지만 직접 구현이 많아짐 | 권장. Partitioning으로 병렬 처리 |
| **1억 건 이상** | 한계. Kafka + Worker 조합 필요 | Partitioning + Remote Chunking으로 가능 |

&nbsp;

좀 더 구체적으로 풀어보면:

&nbsp;

**10만 건 이하** — Node.js가 유리하다.

```typescript
// 이 정도면 충분하다
const orders = await db('orders').where('settled', false);
for (const order of orders) {
  await settle(order);
}
```

Spring Batch의 Job/Step/Reader/Processor/Writer를 세팅하는 것 자체가 오버엔지니어링이다.

&nbsp;

**100만 건** — 양쪽 다 가능하지만 성격이 다르다.

Node.js는 스트림 + 청크를 직접 구현해야 하고, Spring Batch는 Chunk 사이즈만 정하면 된다. 처음 만들 때는 Node.js가 빠르고, 유지보수할 때는 Spring Batch가 편하다.

&nbsp;

**1,000만 건** — Spring Batch가 유리해지는 지점이다.

이 규모에서는 병렬 처리, 재시작, 모니터링이 "있으면 좋은 것"이 아니라 "없으면 안 되는 것"이 된다. Node.js로 해도 되지만, 결국 Spring Batch가 제공하는 것과 비슷한 코드를 직접 작성하게 된다.

&nbsp;

**1억 건 이상** — 둘 다 단독으로는 부족하다.

메시지 큐(Kafka, RabbitMQ)와 결합하거나, 분산 배치 시스템이 필요하다. Spring Batch는 Remote Chunking/Partitioning으로 확장할 수 있고, Node.js는 Kafka + BullMQ + Worker 조합으로 확장한다.

&nbsp;

---

&nbsp;

## 6. 하이브리드: Node.js + 메시지 큐

&nbsp;

Node.js가 1,000만 건 이상에서 살아남는 방법은 **메시지 큐와의 조합**이다.

&nbsp;

### 6-1. 아키텍처

```
┌───────────────┐     ┌──────────────┐     ┌───────────────┐
│  Scheduler    │────→│  BullMQ /    │────→│  Worker 1     │
│  (node-cron)  │     │  Kafka       │     │  Worker 2     │
│               │     │  (작업 큐)    │     │  Worker 3     │
│  작업 분할 →   │     │              │     │  Worker 4     │
│  큐에 등록     │     │              │     │  (병렬 처리)   │
└───────────────┘     └──────────────┘     └───────────────┘
                                                   │
                                                   ▼
                                           ┌───────────────┐
                                           │  Database     │
                                           └───────────────┘
```

&nbsp;

### 6-2. BullMQ로 작업 분할 → Worker에서 처리

```typescript
// scheduler.ts — 작업 분할 (매일 새벽 2시)
import { Queue } from 'bullmq';
import cron from 'node-cron';

const queue = new Queue('settlement', {
  connection: { host: '127.0.0.1', port: 6379 },
});

cron.schedule('0 2 * * *', async () => {
  // 전체 범위 파악
  const [{ minId, maxId }] = await db.raw(
    'SELECT MIN(id) as minId, MAX(id) as maxId FROM orders WHERE settled = false'
  );

  if (!minId) {
    console.log('정산 대상 없음');
    return;
  }

  // 10만 건 단위로 작업 분할
  const chunkSize = 100_000;
  const jobs = [];

  for (let start = minId; start <= maxId; start += chunkSize) {
    jobs.push({
      name: 'process-range',
      data: {
        startId: start,
        endId: Math.min(start + chunkSize - 1, maxId),
        date: new Date().toISOString(),
      },
    });
  }

  await queue.addBulk(jobs);
  console.log(`${jobs.length}개 작업 등록 완료 (ID 범위: ${minId} ~ ${maxId})`);
});
```

```typescript
// worker.ts — 각 범위 처리 (여러 인스턴스 실행 가능)
import { Worker } from 'bullmq';

const worker = new Worker('settlement', async (job) => {
  const { startId, endId } = job.data;
  let currentId = startId;
  let processed = 0;

  while (currentId <= endId) {
    const orders = await db('orders')
      .whereBetween('id', [currentId, endId])
      .andWhere('settled', false)
      .orderBy('id')
      .limit(1000);

    if (orders.length === 0) break;

    await db.transaction(async (trx) => {
      const settlements = orders.map(o => ({
        orderId: o.id,
        amount: o.amount,
        fee: Math.floor(o.amount * 0.03),
        netAmount: o.amount - Math.floor(o.amount * 0.03),
        settledAt: new Date(),
      }));

      await trx.batchInsert('settlements', settlements, 500);
      await trx('orders')
        .whereIn('id', orders.map(o => o.id))
        .update({ settled: true });
    });

    currentId = orders[orders.length - 1].id + 1;
    processed += orders.length;

    // 진행률 보고
    await job.updateProgress(
      Math.round(((currentId - startId) / (endId - startId)) * 100)
    );
  }

  return { processed, range: `${startId}-${endId}` };
}, {
  connection: { host: '127.0.0.1', port: 6379 },
  concurrency: 2,
});

// 에러 핸들링
worker.on('failed', (job, error) => {
  console.error(`작업 실패 [${job?.data.startId}-${job?.data.endId}]:`, error.message);
  // BullMQ가 자동 재시도 (설정된 횟수만큼)
});

worker.on('completed', (job, result) => {
  console.log(`작업 완료: ${result.range}, ${result.processed}건 처리`);
});
```

&nbsp;

### 6-3. Kafka + Worker 패턴 (대규모)

1,000만 건 이상에서 BullMQ(Redis)도 부담이 되면, Kafka를 도입한다.

```typescript
// producer.ts — Kafka로 작업 발행
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ brokers: ['kafka:9092'] });
const producer = kafka.producer();

async function publishSettlementTasks() {
  await producer.connect();

  const cursor = db('orders')
    .where('settled', false)
    .orderBy('id')
    .stream();

  let batch: any[] = [];

  for await (const order of cursor) {
    batch.push({
      key: String(order.id),
      value: JSON.stringify({
        orderId: order.id,
        amount: order.amount,
      }),
    });

    if (batch.length >= 1000) {
      await producer.send({
        topic: 'settlement-tasks',
        messages: batch,
      });
      batch = [];
    }
  }

  if (batch.length > 0) {
    await producer.send({ topic: 'settlement-tasks', messages: batch });
  }

  await producer.disconnect();
}
```

```typescript
// consumer.ts — Kafka Consumer (여러 인스턴스 실행)
const consumer = kafka.consumer({ groupId: 'settlement-workers' });

async function startConsumer() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'settlement-tasks', fromBeginning: false });

  let buffer: any[] = [];

  await consumer.run({
    eachMessage: async ({ message }) => {
      const order = JSON.parse(message.value!.toString());
      buffer.push(order);

      // 500건 모이면 벌크 처리
      if (buffer.length >= 500) {
        await flushBuffer(buffer);
        buffer = [];
      }
    },
  });
}

async function flushBuffer(orders: any[]) {
  const settlements = orders.map(o => ({
    orderId: o.orderId,
    amount: o.amount,
    fee: Math.floor(o.amount * 0.03),
    netAmount: o.amount - Math.floor(o.amount * 0.03),
    settledAt: new Date(),
  }));

  await db.transaction(async (trx) => {
    await trx.batchInsert('settlements', settlements, 500);
    await trx('orders')
      .whereIn('id', orders.map(o => o.orderId))
      .update({ settled: true });
  });
}
```

&nbsp;

이 구조의 장점:
- Kafka 파티션 수만큼 Consumer를 늘려서 **수평 확장**
- Consumer가 죽어도 Kafka가 offset을 관리하므로 **자동 재시작**
- Node.js 프로세스 여러 개를 띄우면 병렬 처리

결국 Node.js로도 대규모 배치를 할 수 있다. 다만 **메시지 큐라는 인프라가 추가**된다.

&nbsp;

---

&nbsp;

## 7. 결론

&nbsp;

### 7-1. 선택 기준 정리

```
팀이 JS/TS만 쓰고, 규모가 100만 건 이하
→ Node.js (cron + 스트림 + 청크)

규모가 크거나, 재시작/모니터링/병렬이 중요하거나, Java 팀이 있음
→ Spring Batch

Node.js인데 규모가 큼
→ Node.js + BullMQ/Kafka (하이브리드)
```

&nbsp;

### 7-2. 제일 나쁜 선택

**"나중에 커질 수 있으니까" Spring Batch를 도입하는 것.**

10만 건짜리 배치에 Spring Batch를 도입하면 이런 일이 벌어진다:
- Job/Step/Chunk 세팅에 반나절
- JobRepository용 DB 테이블 9개 생성
- 간단한 로직 수정에도 Reader/Processor/Writer 3개 파일 수정
- 팀에 Java를 아는 사람이 본인뿐

오버엔지니어링은 기술 부채와 다름없다.

&nbsp;

### 7-3. 제일 좋은 선택

**현재 규모에 맞는 도구를 쓰는 것.**

10만 건이면 스크립트 하나. 100만 건이면 스트림 + 청크. 1,000만 건이면 프레임워크. 규모가 바뀌면 그때 도구도 바꾸면 된다.

"미래를 대비한다"는 명목으로 현재에 과도한 복잡도를 끌어들이는 것보다, **지금 딱 맞는 도구로 빠르게 만들고, 규모가 커질 때 점진적으로 발전시키는 것**이 현실적인 엔지니어링이다.

배치 처리도 다른 모든 기술 선택과 같다. 은탄환은 없고, 트레이드오프만 있다.

&nbsp;

---

`#NodeJS` `#SpringBatch` `#배치처리` `#BullMQ` `#Kafka` `#대규모데이터` `#BackendEngineering` `#기술선택`
