# 대규모 데이터 처리 시리즈 (3) — Kafka 기초, 메시지 큐가 왜 대규모 처리의 핵심인가

&nbsp;

2편에서 Spring Batch로 1,000만 건까지 처리하는 방법을 다뤘다. 하지만 배치만으로는 한계가 있다. 시스템 간 데이터를 주고받아야 하고, 처리를 분산해야 하며, 실시간성도 요구된다.

이때 등장하는 것이 **메시지 큐**, 그리고 그 정점에 있는 **Apache Kafka**다.

&nbsp;

---

&nbsp;

## 1. 메시지 큐란

&nbsp;

메시지 큐는 시스템 사이에 놓인 **우체통**이다.

```
직접 호출 (동기):
┌──────────┐  HTTP  ┌──────────┐
│ 주문 서비스 │ ────→ │ 재고 서비스 │    재고 서비스가 죽으면? 주문도 실패.
└──────────┘       └──────────┘

메시지 큐 (비동기):
┌──────────┐       ┌──────────┐       ┌──────────┐
│ 주문 서비스 │ ──→  │ 메시지 큐  │ ──→  │ 재고 서비스 │
│ (Producer)│ 넣기  │  (Queue) │ 꺼내기│ (Consumer)│
└──────────┘       └──────────┘       └──────────┘
                    ↑
                    재고 서비스가 죽어도 메시지는 큐에 남아있다.
                    재고 서비스가 복구되면 밀린 메시지를 처리한다.
```

&nbsp;

**메시지 큐를 쓰는 이유:**

| 이유 | 설명 |
|:---:|:---|
| **비동기 처리** | 보내는 쪽은 큐에 넣고 바로 리턴 |
| **시스템 분리** | 보내는 쪽과 받는 쪽이 서로 몰라도 됨 |
| **부하 완충** | 트래픽 폭증 시 큐가 버퍼 역할 |
| **실패 복구** | 메시지가 보관되므로 재처리 가능 |
| **확장 용이** | Consumer를 늘리면 처리량 증가 |

&nbsp;

---

&nbsp;

## 2. RabbitMQ vs Kafka

&nbsp;

메시지 큐 하면 RabbitMQ와 Kafka가 양대 산맥이다. 하지만 근본적으로 다르다.

&nbsp;

| 항목 | RabbitMQ | Kafka |
|:---:|:---:|:---:|
| 정체성 | 메시지 브로커 | 이벤트 스트리밍 플랫폼 |
| 메시지 보관 | 소비하면 삭제 | 소비해도 보관 (기간 설정) |
| 소비 방식 | Push (브로커가 밀어줌) | Pull (컨슈머가 당겨감) |
| 순서 보장 | 큐 단위 | 파티션 단위 |
| 처리량 | 수만 건/초 | 수백만 건/초 |
| 재처리 | 불가 (이미 삭제됨) | 가능 (offset 되감기) |
| 적합한 용도 | 작업 큐, RPC | 이벤트 스트리밍, 로그 수집 |

&nbsp;

```
RabbitMQ: "편지를 우체통에 넣으면 배달부가 가져다줌. 받으면 끝."
Kafka:    "모든 기록이 남는 게시판. 누구든 원하는 시점부터 읽을 수 있음."
```

&nbsp;

대규모 데이터 처리에서는 **Kafka가 압도적**이다. 처리량, 재처리 가능성, 확장성 모두 Kafka가 유리하다.

&nbsp;

---

&nbsp;

## 3. Kafka 핵심 개념

&nbsp;

### 아키텍처 전체 그림

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│Producer 1│  │Producer 2│  │Producer 3│
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     ▼             ▼             ▼
┌─────────────────────────────────────────┐
│            Kafka Cluster                │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  Topic: order-events            │    │
│  │  ┌──────────┐ ┌──────────┐     │    │
│  │  │Partition 0│ │Partition 1│    │    │
│  │  │[0][1][2]  │ │[0][1][2] │    │    │
│  │  │[3][4][5]  │ │[3][4]   │    │    │
│  │  └──────────┘ └──────────┘     │    │
│  └─────────────────────────────────┘    │
│                                         │
│  Broker 1    Broker 2    Broker 3       │
└────────────────┬────────────────────────┘
                 │
     ┌───────────┼───────────┐
     ▼           ▼           ▼
┌──────────┐┌──────────┐┌──────────┐
│Consumer 1││Consumer 2││Consumer 3│
│(Group A) ││(Group A) ││(Group B) │
└──────────┘└──────────┘└──────────┘
```

&nbsp;

### 개념 정리

| 개념 | 설명 | 비유 |
|:---:|:---|:---|
| **Topic** | 메시지의 카테고리 | "주문" 게시판, "결제" 게시판 |
| **Partition** | Topic을 나눈 단위 | 게시판의 섹션 (A~Z) |
| **Offset** | 파티션 안에서 메시지 순번 | 게시글 번호 (0, 1, 2, 3...) |
| **Broker** | Kafka 서버 1대 | 게시판 서버 |
| **Producer** | 메시지를 보내는 쪽 | 글 작성자 |
| **Consumer** | 메시지를 읽는 쪽 | 글 읽는 사람 |
| **Consumer Group** | Consumer들의 묶음 | 같은 팀에서 나눠서 읽기 |

&nbsp;

### Partition과 Offset

```
Partition 0: [msg0] [msg1] [msg2] [msg3] [msg4] [msg5]
                                          ↑
                                    현재 offset=4
                                    (여기까지 읽었음)

Partition 1: [msg0] [msg1] [msg2] [msg3]
                              ↑
                        현재 offset=2
```

**핵심:** 같은 파티션 안에서는 순서가 보장된다. 다른 파티션 간에는 보장되지 않는다.

&nbsp;

---

&nbsp;

## 4. 왜 Kafka가 빠른가

&nbsp;

Kafka는 단일 브로커 기준 초당 수십만 건을 처리한다. 비결은 세 가지다.

&nbsp;

### 순차 I/O (Sequential I/O)

```
랜덤 I/O:   디스크 헤드가 여기저기 이동 → 느림
순차 I/O:   파일 끝에 계속 이어 쓰기 → 빠름 (SSD보다 빠를 수 있음)

Kafka는 메시지를 파일 끝에 append만 한다.
읽을 때도 순서대로 읽는다.
→ 디스크 기반인데도 메모리 큐에 버금가는 속도
```

&nbsp;

### Zero-copy

```
일반적인 전송:
디스크 → 커널 버퍼 → 애플리케이션 버퍼 → 커널 소켓 버퍼 → 네트워크
                 ↑ 복사               ↑ 복사

Zero-copy:
디스크 → 커널 버퍼 ──────────────────→ 네트워크
         (sendfile 시스템 콜로 직접 전송)

Kafka는 Consumer에게 데이터를 보낼 때 애플리케이션 메모리를 거치지 않는다.
```

&nbsp;

### 배치 전송

```java
// Producer 설정: 메시지를 바로 보내지 않고 모아서 보냄
Properties props = new Properties();
props.put("batch.size", 16384);          // 16KB까지 모음
props.put("linger.ms", 5);              // 5ms 대기 후 전송
props.put("compression.type", "snappy"); // 압축해서 전송
```

```
메시지 1개씩 전송:   [msg1] [msg2] [msg3] → 네트워크 3번
배치 전송:           [msg1, msg2, msg3]   → 네트워크 1번, 압축까지
```

&nbsp;

---

&nbsp;

## 5. Producer 코드 예시 (Spring Kafka)

&nbsp;

### 의존성

```groovy
// build.gradle
dependencies {
    implementation 'org.springframework.kafka:spring-kafka'
}
```

&nbsp;

### 설정

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all          # 모든 Replica에 기록 확인
      retries: 3         # 실패 시 3회 재시도
      properties:
        linger.ms: 10
        batch.size: 32768
        compression.type: snappy
```

&nbsp;

### Producer 코드

```java
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void send(OrderEvent event) {
        // key를 orderId로 → 같은 주문의 이벤트는 같은 파티션으로
        kafkaTemplate.send("order-events", event.getOrderId(), event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("Kafka 전송 실패: orderId={}", event.getOrderId(), ex);
                    } else {
                        log.debug("Kafka 전송 성공: topic={}, partition={}, offset={}",
                                result.getRecordMetadata().topic(),
                                result.getRecordMetadata().partition(),
                                result.getRecordMetadata().offset());
                    }
                });
    }
}
```

&nbsp;

### 이벤트 클래스

```java
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderEvent {
    private String orderId;
    private EventType type;      // CREATED, PAID, SHIPPED, CANCELLED
    private String productId;
    private int quantity;
    private long amount;
    private long timestamp;

    public enum EventType {
        CREATED, PAID, SHIPPED, CANCELLED
    }
}
```

&nbsp;

---

&nbsp;

## 6. Consumer 코드 예시 (Spring Kafka)

&nbsp;

### 설정

```yaml
spring:
  kafka:
    consumer:
      group-id: inventory-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest   # 처음부터 읽기
      enable-auto-commit: false     # 수동 커밋
      properties:
        spring.json.trusted.packages: "*"
        max.poll.records: 500       # 한 번에 최대 500건
```

&nbsp;

### Consumer 코드

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class InventoryConsumer {

    private final InventoryService inventoryService;

    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        concurrency = "3"          // 3개 스레드 = 3개 파티션 동시 처리
    )
    public void consume(OrderEvent event, Acknowledgment ack) {
        try {
            switch (event.getType()) {
                case CREATED:
                    inventoryService.reserve(event.getProductId(), event.getQuantity());
                    break;
                case CANCELLED:
                    inventoryService.release(event.getProductId(), event.getQuantity());
                    break;
                case PAID:
                    inventoryService.confirm(event.getProductId(), event.getQuantity());
                    break;
            }
            ack.acknowledge();  // 처리 성공 → offset 커밋
            log.info("처리 완료: orderId={}, type={}", event.getOrderId(), event.getType());

        } catch (Exception e) {
            log.error("처리 실패: orderId={}", event.getOrderId(), e);
            // ack 안 하면 다시 읽힘 (재처리)
        }
    }
}
```

&nbsp;

---

&nbsp;

## 7. Partition과 Consumer Group

&nbsp;

이것이 Kafka 병렬 처리의 핵심이다.

&nbsp;

### 규칙

```
1. 하나의 파티션은 같은 Consumer Group 안에서 하나의 Consumer만 읽는다.
2. Consumer가 파티션보다 많으면 놀고 있는 Consumer가 생긴다.
3. 파티션을 늘리면 Consumer도 늘릴 수 있다 → 처리량 증가.
```

&nbsp;

### 시각화

```
Topic: order-events (6개 파티션)

Consumer Group A (주문 서비스):
  Consumer A-1: Partition 0, 1   ← 2개 담당
  Consumer A-2: Partition 2, 3   ← 2개 담당
  Consumer A-3: Partition 4, 5   ← 2개 담당

Consumer Group B (분석 서비스):
  Consumer B-1: Partition 0, 1, 2, 3, 4, 5  ← 혼자 6개 전부

→ Group A는 3명이 나눠서 빠르게 처리
→ Group B는 1명이 전부 처리 (느리지만 전체 데이터 필요)
→ 두 Group은 서로 독립 (각자의 offset을 관리)
```

&nbsp;

```java
// Consumer 수와 파티션 수 맞추기
// 파티션이 6개면 concurrency도 6으로

@KafkaListener(
    topics = "order-events",
    groupId = "inventory-service",
    concurrency = "6"    // 파티션 수와 맞춤
)
public void consume(OrderEvent event) {
    // 각 스레드가 1개 파티션을 전담
}
```

&nbsp;

---

&nbsp;

## 8. Offset 관리

&nbsp;

### Auto Commit (기본값)

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: true      # 기본값
      auto-commit-interval-ms: 5000 # 5초마다 자동 커밋
```

**문제:** 메시지를 받았지만 처리 전에 커밋되면 → 장애 시 유실.

&nbsp;

### Manual Commit (권장)

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: false
    listener:
      ack-mode: MANUAL  # 수동 커밋
```

```java
@KafkaListener(topics = "order-events")
public void consume(OrderEvent event, Acknowledgment ack) {
    processEvent(event);  // 처리 완료 후
    ack.acknowledge();    // 명시적으로 커밋
}
```

&nbsp;

### Offset 커밋 전략 비교

| 전략 | 장점 | 단점 |
|:---:|:---|:---|
| Auto Commit | 간단 | 유실 가능 |
| Manual (건별) | 정확 | 커밋 빈번 → 느림 |
| Manual (배치) | 균형 | 배치 내 실패 시 재처리 |

&nbsp;

---

&nbsp;

## 9. 재처리: Offset 되감기

&nbsp;

Kafka의 강력한 기능 중 하나가 **offset을 되감아서 과거 데이터를 재처리**할 수 있다는 것이다.

```bash
# Consumer Group의 offset을 처음으로 되감기
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group inventory-service \
  --topic order-events \
  --reset-offsets --to-earliest \
  --execute

# 특정 시점으로 되감기
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group inventory-service \
  --topic order-events \
  --reset-offsets --to-datetime 2026-04-01T00:00:00.000 \
  --execute

# 특정 offset으로 이동
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group inventory-service \
  --topic order-events \
  --reset-offsets --to-offset 1000 \
  --execute
```

&nbsp;

**재처리가 필요한 상황:**
- 버그가 있어서 어제 데이터를 다시 처리해야 할 때
- 새로운 Consumer를 추가하고 과거 데이터부터 처리하고 싶을 때
- 분석 로직이 바뀌어서 전체 재계산이 필요할 때

&nbsp;

---

&nbsp;

## 10. Docker Compose로 Kafka 로컬 실행

&nbsp;

개발 환경에서 Kafka를 가장 쉽게 띄우는 방법이다.

```yaml
# docker-compose.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_LOG_RETENTION_HOURS: 168  # 7일 보관

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

&nbsp;

```bash
# 실행
docker-compose up -d

# Topic 생성
docker exec -it kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic order-events \
  --partitions 6 \
  --replication-factor 1

# 메시지 확인
docker exec -it kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --from-beginning

# Kafka UI 확인
# http://localhost:8080
```

&nbsp;

**KRaft 모드 (Zookeeper 없이):**

Kafka 3.3+부터 Zookeeper 없이 실행 가능하다.

```yaml
# docker-compose-kraft.yml
version: '3.8'

services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_LOG_RETENTION_HOURS: 168
```

&nbsp;

---

&nbsp;

## 정리

&nbsp;

| 개념 | 핵심 |
|:---:|:---|
| Topic | 메시지의 카테고리 |
| Partition | 병렬 처리 단위 (파티션 수 = 최대 Consumer 수) |
| Offset | 어디까지 읽었는지 기록 (되감기로 재처리 가능) |
| Consumer Group | 같은 그룹은 파티션을 나눠 읽고, 다른 그룹은 독립적 |
| Manual Commit | 처리 완료 후 커밋으로 유실 방지 |

&nbsp;

Kafka를 이해했으면 이제 **실전**에 들어갈 준비가 된 것이다. Kafka는 단순한 메시지 큐가 아니라, 대규모 데이터 처리의 **중앙 신경계**다.

&nbsp;

---

&nbsp;

**다음 편 예고:**

4편에서는 Kafka와 Worker Pool을 결합하여 **1억 건을 병렬로 처리하는 패턴**을 다룬다. 작업 분할 전략, JDBC 배치 처리, 멱등성 보장, Dead Letter Queue까지 실전 코드와 함께 살펴본다.

&nbsp;

---

`#Kafka` `#메시지큐` `#대규모데이터` `#SpringKafka` `#이벤트스트리밍` `#Producer` `#Consumer` `#DockerCompose` `#데이터엔지니어링`
