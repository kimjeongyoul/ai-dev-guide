# 대규모 데이터 처리 시리즈 (6) — 실시간 스트리밍, Kafka Streams와 Flink

&nbsp;

5편에서 DB 최적화를 다뤘다. 이번 편에서는 **실시간 스트리밍**의 세계로 들어간다.

배치는 "어제 일어난 일"을 처리한다. 스트리밍은 **"지금 일어나고 있는 일"**을 처리한다. 실시간 대시보드, 이상 탐지, 실시간 집계 — 이 모든 것의 기반이 스트리밍이다.

&nbsp;

---

&nbsp;

## 1. 배치와 스트리밍의 근본적 차이

&nbsp;

| 항목 | 배치 | 스트리밍 |
|:---:|:---:|:---:|
| 데이터 | Bounded (끝이 있음) | Unbounded (끝이 없음) |
| 처리 시점 | 데이터가 다 모인 후 | 데이터가 도착하는 즉시 |
| 결과 | 최종 결과 1번 | 중간 결과가 계속 갱신 |
| 시간 개념 | 처리 시간 (언제 실행했나) | 이벤트 시간 (언제 발생했나) |

&nbsp;

```
배치:
[어제 주문 데이터 전부] ──→ [집계] ──→ [최종 결과]
        끝이 있음                      1번 완료

스트리밍:
[주문1] → [집계: 1건] → [결과: 10,000원]
[주문2] → [집계: 2건] → [결과: 25,000원]   ← 갱신
[주문3] → [집계: 3건] → [결과: 38,000원]   ← 갱신
   ...     끝이 없음      계속 갱신
```

&nbsp;

스트리밍에서 가장 어려운 문제는 **"언제 결과를 확정하는가?"**다. 데이터가 끝없이 들어오니까.

이 문제를 해결하는 것이 **윈도우(Window)**다.

&nbsp;

---

&nbsp;

## 2. Kafka Streams

&nbsp;

Kafka Streams는 **라이브러리**다. 별도 클러스터 없이, Spring Boot 애플리케이션 안에서 바로 스트리밍 처리를 할 수 있다.

```
┌────────────────────────────────────┐
│   Spring Boot Application         │
│                                    │
│   ┌──────────────────────────┐     │
│   │    Kafka Streams         │     │
│   │    (라이브러리)           │     │
│   │                          │     │
│   │  Input Topic  → 처리 →  │     │
│   │                Output    │     │
│   │                Topic     │     │
│   └──────────────────────────┘     │
│                                    │
│   별도 클러스터 불필요!             │
└────────────────────────────────────┘
```

&nbsp;

### 핵심 개념

| 개념 | 설명 | 비유 |
|:---:|:---|:---|
| **KStream** | 이벤트 스트림 (레코드 하나하나) | 흘러가는 물 |
| **KTable** | 최신 상태 테이블 (같은 key는 덮어씀) | 화이트보드 |
| **GlobalKTable** | 모든 인스턴스에 복제된 KTable | 모든 교실에 있는 공지 |

&nbsp;

```
KStream (이벤트 하나하나):
key=user1, value=주문A  →  기록
key=user1, value=주문B  →  기록
key=user1, value=주문C  →  기록
→ 3건 모두 존재

KTable (최신 상태만):
key=user1, value=주문A  →  저장
key=user1, value=주문B  →  덮어쓰기
key=user1, value=주문C  →  덮어쓰기
→ 최종: user1 = 주문C
```

&nbsp;

### 윈도우 집계

스트리밍에서 "최근 5분간 주문 수"를 구하려면 **윈도우**가 필요하다.

&nbsp;

#### Tumbling Window (고정 윈도우)

```
시간: |  0분  |  5분  | 10분  | 15분  |
      |-------|-------|-------|-------|
      | 윈도우1 | 윈도우2 | 윈도우3 | 윈도우4 |
      | 겹치지 않음, 빈틈 없음                |
```

&nbsp;

#### Hopping Window (슬라이딩 간격)

```
시간: |  0분  |  5분  | 10분  | 15분  |
      |-------|-------|-------|-------|
      |--- 윈도우1 (10분) ---|
            |--- 윈도우2 (10분) ---|
                  |--- 윈도우3 (10분) ---|
      | 5분마다 새 윈도우, 10분 길이, 겹침 있음 |
```

&nbsp;

#### Session Window (유동 윈도우)

```
이벤트:  ●  ●  ●          ●  ●              ●
시간:    0  1  2          8  9              20
         |---------|     |-----|          |---|
         세션 1 (gap=5분)  세션 2           세션 3
         활동 간 5분 이상 공백이면 새 세션
```

&nbsp;

### 코드 예시: 실시간 주문 집계

```java
@Configuration
public class OrderStreamConfig {

    @Bean
    public KafkaStreamsConfiguration kafkaStreamsConfig() {
        Map<String, Object> props = new HashMap<>();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "order-aggregation");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000);
        return new KafkaStreamsConfiguration(props);
    }

    @Bean
    public KStream<String, OrderEvent> orderStream(StreamsBuilder builder) {
        // 1. 주문 이벤트 스트림 읽기
        KStream<String, OrderEvent> orders = builder.stream(
                "order-events",
                Consumed.with(Serdes.String(), orderEventSerde()));

        // 2. 5분 윈도우로 가맹점별 매출 집계
        KTable<Windowed<String>, Long> salesPerStore = orders
                .filter((key, event) -> event.getType() == EventType.PAID)
                .groupBy(
                    (key, event) -> event.getStoreId(),
                    Grouped.with(Serdes.String(), orderEventSerde()))
                .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
                .aggregate(
                    () -> 0L,
                    (storeId, event, total) -> total + event.getAmount(),
                    Materialized.<String, Long, WindowStore<Bytes, byte[]>>as(
                        "store-sales-store")
                        .withValueSerde(Serdes.Long())
                );

        // 3. 결과를 출력 토픽으로 전송
        salesPerStore.toStream()
                .map((windowedKey, total) -> KeyValue.pair(
                    windowedKey.key(),
                    new SalesResult(
                        windowedKey.key(),
                        total,
                        windowedKey.window().startTime().toEpochMilli(),
                        windowedKey.window().endTime().toEpochMilli()
                    )))
                .to("store-sales-5min", Produced.with(Serdes.String(), salesResultSerde()));

        return orders;
    }
}
```

&nbsp;

### 상태 저장: RocksDB

Kafka Streams는 **RocksDB**를 내장하여 로컬에 상태를 저장한다.

```
┌────────────────────────────────────┐
│   Kafka Streams Instance          │
│                                    │
│   ┌──────────────────────┐         │
│   │    RocksDB           │         │
│   │    (로컬 디스크)      │         │
│   │                      │         │
│   │  storeId → 현재 합계  │         │
│   │  S001 → 1,500,000   │         │
│   │  S002 → 890,000     │         │
│   └──────────────────────┘         │
│                                    │
│   장애 시 → Kafka changelog        │
│   토픽에서 복구                     │
└────────────────────────────────────┘
```

&nbsp;

---

&nbsp;

## 3. Apache Flink

&nbsp;

Flink는 **분산 스트리밍 엔진**이다. Kafka Streams와 달리 **별도 클러스터가 필요**하지만, 더 큰 규모와 복잡한 처리를 할 수 있다.

```
┌─────────────────────────────────────────┐
│            Flink Cluster                │
│                                         │
│  ┌───────────┐  ┌───────────┐          │
│  │ JobManager│  │TaskManager│ x N      │
│  │ (마스터)   │  │ (워커)    │          │
│  │           │  │           │          │
│  │ 작업 분배  │  │ 실제 처리  │          │
│  │ 체크포인트 │  │ 상태 관리  │          │
│  └───────────┘  └───────────┘          │
│                                         │
│  Kafka Source → 처리 → Kafka/DB Sink   │
└─────────────────────────────────────────┘
```

&nbsp;

### DataStream API

```java
public class FraudDetectionJob {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env =
                StreamExecutionEnvironment.getExecutionEnvironment();

        // 1. Kafka에서 읽기
        KafkaSource<TransactionEvent> source = KafkaSource.<TransactionEvent>builder()
                .setBootstrapServers("localhost:9092")
                .setTopics("transactions")
                .setGroupId("fraud-detection")
                .setStartingOffsets(OffsetsInitializer.latest())
                .setValueOnlyDeserializer(new TransactionEventSchema())
                .build();

        DataStream<TransactionEvent> transactions = env.fromSource(
                source, WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(5)),
                "kafka-source");

        // 2. 이상 거래 탐지: 1분 안에 3건 이상 + 합계 100만원 이상
        DataStream<FraudAlert> alerts = transactions
                .keyBy(TransactionEvent::getUserId)
                .window(TumblingEventTimeWindows.of(Time.minutes(1)))
                .process(new FraudDetectionFunction());

        // 3. 알림 전송
        alerts.addSink(new AlertSink());

        env.execute("Fraud Detection");
    }
}
```

&nbsp;

### 이상 탐지 로직

```java
public class FraudDetectionFunction
        extends ProcessWindowFunction<TransactionEvent, FraudAlert, String, TimeWindow> {

    @Override
    public void process(String userId,
                        Context context,
                        Iterable<TransactionEvent> events,
                        Collector<FraudAlert> out) {

        List<TransactionEvent> txList = new ArrayList<>();
        long totalAmount = 0;

        for (TransactionEvent event : events) {
            txList.add(event);
            totalAmount += event.getAmount();
        }

        // 1분 안에 3건 이상 + 합계 100만원 이상
        if (txList.size() >= 3 && totalAmount >= 1_000_000) {
            out.collect(FraudAlert.builder()
                    .userId(userId)
                    .transactionCount(txList.size())
                    .totalAmount(totalAmount)
                    .windowStart(context.window().getStart())
                    .windowEnd(context.window().getEnd())
                    .riskLevel("HIGH")
                    .build());
        }
    }
}
```

&nbsp;

### Exactly-once 보장

Flink는 **체크포인팅**으로 exactly-once를 보장한다.

```java
// 체크포인트 설정
env.enableCheckpointing(60000);  // 1분마다 체크포인트

CheckpointConfig config = env.getCheckpointConfig();
config.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
config.setMinPauseBetweenCheckpoints(30000);  // 체크포인트 간 최소 30초
config.setCheckpointTimeout(120000);          // 타임아웃 2분
config.setMaxConcurrentCheckpoints(1);
config.setExternalizedCheckpointCleanup(
    ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);  // 취소 시에도 보관

// 상태 백엔드 설정 (대규모 상태 저장)
env.setStateBackend(new EmbeddedRocksDBStateBackend());
config.setCheckpointStorage("hdfs://namenode:8020/flink/checkpoints");
```

```
체크포인트 동작:
시점 1: [이벤트 1,2,3 처리 완료] → 체크포인트 저장 ✅
시점 2: [이벤트 4,5 처리 중] → 장애 발생! 💥
복구:   [시점 1로 복원] → 이벤트 4,5 다시 처리
        → 결과가 정확히 한 번만 반영됨
```

&nbsp;

---

&nbsp;

## 4. Kafka Streams vs Flink 비교

&nbsp;

| 항목 | Kafka Streams | Apache Flink |
|:---:|:---:|:---:|
| 형태 | 라이브러리 | 분산 엔진 (클러스터) |
| 배포 | Spring Boot에 내장 | 별도 Flink 클러스터 |
| 입력 소스 | Kafka만 | Kafka, 파일, DB, 소켓 등 |
| 상태 관리 | RocksDB (로컬) | RocksDB + 체크포인트 (분산) |
| Exactly-once | Kafka 트랜잭션 기반 | 체크포인트 기반 |
| 확장 | 인스턴스 수 = 파티션 수 | TaskManager 추가 |
| 복잡한 이벤트 처리 | 제한적 | CEP 라이브러리 지원 |
| 배치 처리 | 불가 | 가능 (통합 API) |
| 운영 난이도 | 낮음 | 높음 |
| 적합한 규모 | 초당 수만 건 | 초당 수백만 건 |

&nbsp;

**선택 기준:**

```
"Kafka에서 읽어서 간단한 집계/필터링?"     → Kafka Streams
"여러 소스에서 복잡한 이벤트 처리?"          → Flink
"별도 클러스터 운영할 여력이 없다?"           → Kafka Streams
"초당 100만 건 이상, exactly-once 필수?"    → Flink
```

&nbsp;

---

&nbsp;

## 5. 실시간 대시보드 구축

&nbsp;

```
┌──────┐    ┌───────┐    ┌────────────┐    ┌──────────┐    ┌─────────┐
│ 서비스 │──→│ Kafka │──→│ Kafka      │──→│ WebSocket│──→│ 브라우저 │
│      │    │       │    │ Streams    │    │ 서버     │    │ 대시보드 │
│ 주문  │    │       │    │ (5분 집계) │    │          │    │         │
│ 이벤트│    │       │    │            │    │          │    │ 실시간  │
│      │    │       │    │ → 결과 토픽│──→│ 구독     │──→│ 차트    │
└──────┘    └───────┘    └────────────┘    └──────────┘    └─────────┘
```

&nbsp;

### WebSocket으로 실시간 전달

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-dashboard")
                .setAllowedOrigins("*")
                .withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
    }
}

@Service
@RequiredArgsConstructor
public class DashboardConsumer {

    private final SimpMessagingTemplate messagingTemplate;

    @KafkaListener(topics = "store-sales-5min", groupId = "dashboard")
    public void onSalesUpdate(SalesResult result) {
        // Kafka에서 받은 집계 결과를 WebSocket으로 브라우저에 전달
        messagingTemplate.convertAndSend(
                "/topic/sales/" + result.getStoreId(),
                result);
    }
}
```

&nbsp;

### 프론트엔드 (React)

```typescript
import { Client } from '@stomp/stompjs';
import { useEffect, useState } from 'react';

function SalesDashboard({ storeId }: { storeId: string }) {
    const [sales, setSales] = useState<SalesResult | null>(null);

    useEffect(() => {
        const client = new Client({
            brokerURL: 'ws://localhost:8080/ws-dashboard',
            onConnect: () => {
                client.subscribe(`/topic/sales/${storeId}`, (message) => {
                    const result = JSON.parse(message.body);
                    setSales(result);
                });
            },
        });
        client.activate();

        return () => { client.deactivate(); };
    }, [storeId]);

    return (
        <div>
            <h2>실시간 매출</h2>
            {sales && (
                <div>
                    <p>최근 5분 매출: {sales.totalAmount.toLocaleString()}원</p>
                    <p>갱신 시간: {new Date(sales.windowEnd).toLocaleTimeString()}</p>
                </div>
            )}
        </div>
    );
}
```

&nbsp;

---

&nbsp;

## 6. CDC (Change Data Capture)

&nbsp;

CDC는 **DB 변경을 실시간 이벤트로 캡처**하는 기술이다. 애플리케이션 코드를 수정하지 않고도 DB 변경을 스트리밍할 수 있다.

&nbsp;

### Debezium: CDC의 대표 주자

```
┌──────────┐    ┌───────────┐    ┌────────┐    ┌──────────┐
│  MySQL   │──→│ Debezium  │──→│ Kafka  │──→│ Consumer │
│          │    │ Connector │    │        │    │          │
│ binlog   │    │           │    │ DB 변경│    │ 검색 엔진│
│ 변경 캡처│    │ binlog →  │    │ 이벤트 │    │ 캐시 갱신│
│          │    │ Kafka     │    │        │    │ 동기화   │
└──────────┘    └───────────┘    └────────┘    └──────────┘
```

&nbsp;

### Debezium 설정 (Kafka Connect)

```json
{
    "name": "mysql-connector",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.hostname": "localhost",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "dbz",
        "database.server.id": "1",
        "topic.prefix": "dbserver1",
        "database.include.list": "mydb",
        "table.include.list": "mydb.orders,mydb.members",
        "schema.history.internal.kafka.bootstrap.servers": "localhost:9092",
        "schema.history.internal.kafka.topic": "schema-changes"
    }
}
```

&nbsp;

### CDC 이벤트 형태

```json
{
    "before": {
        "id": 1001,
        "status": "CREATED",
        "amount": 50000
    },
    "after": {
        "id": 1001,
        "status": "PAID",
        "amount": 50000
    },
    "source": {
        "db": "mydb",
        "table": "orders",
        "ts_ms": 1712620800000
    },
    "op": "u"
}
```

| op | 의미 |
|:---:|:---|
| c | CREATE (INSERT) |
| u | UPDATE |
| d | DELETE |
| r | READ (스냅샷) |

&nbsp;

### CDC Consumer

```java
@Service
@Slf4j
public class OrderCdcConsumer {

    @KafkaListener(topics = "dbserver1.mydb.orders", groupId = "search-sync")
    public void onOrderChange(ConsumerRecord<String, String> record) {
        JsonNode payload = objectMapper.readTree(record.value()).get("payload");
        String op = payload.get("op").asText();
        JsonNode after = payload.get("after");

        switch (op) {
            case "c", "u":
                // Elasticsearch에 동기화
                elasticsearchClient.index(
                    IndexRequest.of(i -> i
                        .index("orders")
                        .id(after.get("id").asText())
                        .document(mapToOrderDocument(after))));
                break;
            case "d":
                // Elasticsearch에서 삭제
                elasticsearchClient.delete(
                    DeleteRequest.of(d -> d
                        .index("orders")
                        .id(payload.get("before").get("id").asText())));
                break;
        }
    }
}
```

&nbsp;

**CDC의 장점:**
- 애플리케이션 코드 수정 없이 DB 변경을 캡처
- 이벤트 발행을 깜빡해도 DB 변경은 항상 캡처됨
- DB 간 동기화, 캐시 무효화, 검색 인덱싱에 최적

&nbsp;

---

&nbsp;

## 정리

&nbsp;

| 기술 | 역할 | 적합한 경우 |
|:---:|:---|:---|
| Kafka Streams | 가벼운 스트리밍 처리 | Kafka 중심, 간단한 집계 |
| Apache Flink | 대규모 스트리밍 엔진 | 복잡한 이벤트, exactly-once |
| Debezium (CDC) | DB 변경 캡처 | 시스템 간 동기화 |
| WebSocket | 실시간 전달 | 브라우저 대시보드 |

&nbsp;

스트리밍은 배치를 대체하는 것이 아니라 **보완**한다. 실시간으로 근사 결과를 보여주고, 배치로 정확한 결과를 확정하는 조합이 현실에서 가장 많이 쓰인다.

&nbsp;

---

&nbsp;

**다음 편 예고:**

7편 (마지막 편)에서는 **10억 건 아키텍처**를 다룬다. RDBMS의 한계를 넘어, Spark, 데이터 레이크, OLAP 데이터베이스로 10억 건을 처리하는 현실적인 아키텍처를 설계해본다.

&nbsp;

---

`#실시간스트리밍` `#KafkaStreams` `#ApacheFlink` `#CDC` `#Debezium` `#WebSocket` `#대규모데이터` `#이벤트처리` `#데이터엔지니어링`
