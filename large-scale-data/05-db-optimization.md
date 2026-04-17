# 대규모 데이터 처리 시리즈 (5) — DB 최적화, 대량 데이터에서 살아남는 SQL

&nbsp;

4편에서 Kafka + Worker 패턴으로 1억 건을 병렬 처리하는 방법을 다뤘다. 하지만 아무리 Worker를 늘려도 **DB가 느리면 전체가 느리다.**

이번 편에서는 대량 데이터에서 DB 성능을 끌어올리는 실전 기법을 다룬다. 쿼리 한 줄이 30초에서 0.3초로 바뀌는 경험을 해보자.

&nbsp;

---

&nbsp;

## 1. EXPLAIN으로 실행 계획 읽기

&nbsp;

쿼리 최적화의 시작은 **EXPLAIN**이다. DB가 쿼리를 어떻게 실행하는지 보여준다.

&nbsp;

### 느린 쿼리 예시

```sql
-- 3,000만 건 테이블에서 실행: 28초
SELECT * FROM orders
WHERE store_id = 'S001'
  AND order_date BETWEEN '2026-01-01' AND '2026-03-31'
  AND status = 'CONFIRMED';
```

&nbsp;

### EXPLAIN 실행

```sql
EXPLAIN SELECT * FROM orders
WHERE store_id = 'S001'
  AND order_date BETWEEN '2026-01-01' AND '2026-03-31'
  AND status = 'CONFIRMED';
```

```
+----+------+------+------+---------+------+----------+-------------+
| id | type | table| key  | key_len | ref  | rows     | Extra       |
+----+------+------+------+---------+------+----------+-------------+
|  1 | ALL  |orders| NULL | NULL    | NULL | 30245831 | Using where |
+----+------+------+------+---------+------+----------+-------------+
```

&nbsp;

**읽는 법:**

| 컬럼 | 의미 | 위 결과 해석 |
|:---:|:---|:---|
| type | 접근 방식 | `ALL` = 풀스캔 (최악) |
| key | 사용한 인덱스 | `NULL` = 인덱스 안 씀 |
| rows | 예상 스캔 행 수 | 3,024만 행 전부 스캔 |
| Extra | 추가 정보 | `Using where` = 조건 필터링만 |

&nbsp;

### 인덱스 추가 후

```sql
CREATE INDEX idx_orders_store_date_status
ON orders (store_id, order_date, status);

EXPLAIN SELECT * FROM orders
WHERE store_id = 'S001'
  AND order_date BETWEEN '2026-01-01' AND '2026-03-31'
  AND status = 'CONFIRMED';
```

```
+----+-------+------+--------------------------------+---------+------+-------+-------------+
| id | type  | table| key                            | key_len | ref  | rows  | Extra       |
+----+-------+------+--------------------------------+---------+------+-------+-------------+
|  1 | range |orders| idx_orders_store_date_status   | 108     | NULL | 45230 | Using where |
+----+-------+------+--------------------------------+---------+------+-------+-------------+
```

&nbsp;

| 변경점 | 변경 전 | 변경 후 |
|:---:|:---:|:---:|
| type | ALL (풀스캔) | range (범위 스캔) |
| key | NULL | idx_orders_store_date_status |
| rows | 30,245,831 | 45,230 |
| 실행시간 | 28초 | 0.12초 |

&nbsp;

**EXPLAIN type 순서 (좋은 순 → 나쁜 순):**

```
system > const > eq_ref > ref > range > index > ALL
  ↑                                              ↑
 최고                                           최악
(1건)                                       (풀스캔)
```

&nbsp;

---

&nbsp;

## 2. 인덱스 전략

&nbsp;

### B-Tree 인덱스 (기본)

```
인덱스 없이:
orders 테이블 3,000만 건 → store_id = 'S001' 찾기 → 3,000만 건 전부 확인

B-Tree 인덱스:
                    [M]
                   /   \
              [D,H]     [R,V]
             / | \      / | \
          [A,C][E,G][I,K][N,Q][S,U][W,Z]
                              ↑
                          S001은 여기!
                          → 3~4회 탐색으로 도달
```

&nbsp;

### 복합 인덱스 (Composite Index)

컬럼 순서가 중요하다. **선택도가 높은(값이 다양한) 컬럼을 앞에** 놓는다.

```sql
-- 좋은 순서: store_id(1만 종류) → order_date(365 종류) → status(5 종류)
CREATE INDEX idx_orders_composite
ON orders (store_id, order_date, status);

-- 이 인덱스가 사용되는 쿼리들:
WHERE store_id = 'S001'                                    -- ✅ 첫 컬럼
WHERE store_id = 'S001' AND order_date = '2026-01-01'      -- ✅ 첫 + 둘째
WHERE store_id = 'S001' AND order_date BETWEEN ... AND status = 'CONFIRMED'  -- ✅ 전부

-- 이 인덱스가 사용되지 않는 쿼리:
WHERE order_date = '2026-01-01'                            -- ❌ 첫 컬럼 누락
WHERE status = 'CONFIRMED'                                 -- ❌ 첫 컬럼 누락
WHERE order_date = '2026-01-01' AND status = 'CONFIRMED'   -- ❌ 첫 컬럼 누락
```

&nbsp;

### 커버링 인덱스 (Covering Index)

쿼리가 필요한 모든 컬럼이 인덱스에 포함되면, **테이블에 접근하지 않고 인덱스만으로 응답**한다.

```sql
-- 인덱스: (store_id, order_date, status, amount)
CREATE INDEX idx_orders_covering
ON orders (store_id, order_date, status, amount);

-- 이 쿼리는 테이블에 접근하지 않음 (Using index)
SELECT store_id, order_date, SUM(amount)
FROM orders
WHERE store_id = 'S001'
  AND order_date BETWEEN '2026-01-01' AND '2026-03-31'
  AND status = 'CONFIRMED'
GROUP BY store_id, order_date;

-- EXPLAIN에 "Using index"가 나오면 커버링 인덱스
```

&nbsp;

---

&nbsp;

## 3. 풀스캔이 더 빠른 경우

&nbsp;

인덱스가 항상 좋은 건 아니다. **전체의 20~30% 이상을 읽을 때는 풀스캔이 더 빠르다.**

```
인덱스 사용 시:
인덱스 → 주소 확인 → 테이블 랜덤 접근 (한 건씩, 디스크 여기저기)
→ 건수가 많으면 랜덤 I/O가 너무 많아짐

풀스캔 시:
테이블 처음부터 끝까지 순차 읽기 (Sequential I/O)
→ 어차피 대부분 읽어야 하면 이게 더 효율적
```

&nbsp;

```sql
-- status가 'CONFIRMED'인 데이터가 전체의 80%라면?
-- 인덱스 보다 풀스캔이 빠름 → DB 옵티마이저가 알아서 풀스캔 선택

-- 반대로, status가 'REFUNDED'인 데이터가 전체의 0.1%라면?
-- 인덱스가 압도적으로 빠름
```

**규칙:** 인덱스는 **선택도가 높을 때(결과가 전체의 5~10% 이하)** 효과적이다.

&nbsp;

---

&nbsp;

## 4. 파티셔닝

&nbsp;

테이블이 너무 크면 **물리적으로 분할**하는 것이 파티셔닝이다.

&nbsp;

### Range 파티셔닝 (가장 많이 사용)

```sql
-- MySQL
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    store_id VARCHAR(20),
    order_date DATE,
    amount BIGINT,
    status VARCHAR(20),
    PRIMARY KEY (id, order_date)
) PARTITION BY RANGE (YEAR(order_date) * 100 + MONTH(order_date)) (
    PARTITION p202601 VALUES LESS THAN (202602),
    PARTITION p202602 VALUES LESS THAN (202603),
    PARTITION p202603 VALUES LESS THAN (202604),
    PARTITION p202604 VALUES LESS THAN (202605),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

```
orders 테이블 (3,000만 건)
├── p202601 (500만 건) ← 1월 데이터만 스캔
├── p202602 (600만 건)
├── p202603 (550만 건)
├── p202604 (500만 건)
└── p_future (...)

WHERE order_date = '2026-01-15'
→ p202601 파티션만 스캔 (500만 건)
→ 나머지 2,500만 건은 건드리지 않음 (Partition Pruning)
```

&nbsp;

### Hash 파티셔닝

```sql
-- 균등하게 분배할 때
CREATE TABLE user_logs (
    id BIGINT AUTO_INCREMENT,
    user_id BIGINT,
    action VARCHAR(50),
    created_at DATETIME,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH(user_id) PARTITIONS 16;
```

&nbsp;

### List 파티셔닝

```sql
-- 특정 값 기준으로 분할
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    region VARCHAR(10),
    amount BIGINT,
    PRIMARY KEY (id, region)
) PARTITION BY LIST COLUMNS(region) (
    PARTITION p_kr VALUES IN ('KR'),
    PARTITION p_jp VALUES IN ('JP'),
    PARTITION p_us VALUES IN ('US'),
    PARTITION p_etc VALUES IN ('CN', 'EU', 'OTHER')
);
```

&nbsp;

---

&nbsp;

## 5. 벌크 연산

&nbsp;

대량 데이터를 건건이 처리하면 느리다. **벌크 연산**으로 한 번에 처리해야 한다.

&nbsp;

### INSERT SELECT

```sql
-- 주문 테이블에서 정산 테이블로 한 번에 삽입
INSERT INTO daily_settlement (store_id, settlement_date, total_amount, order_count)
SELECT
    store_id,
    order_date,
    SUM(amount),
    COUNT(*)
FROM orders
WHERE order_date = '2026-04-08'
  AND status = 'CONFIRMED'
GROUP BY store_id, order_date;

-- 100만 건 주문 → 1만 건 정산 (가맹점별 집계) → 약 3초
```

&nbsp;

### UPDATE JOIN

```sql
-- 가맹점 등급에 따라 수수료율 일괄 업데이트
UPDATE orders o
JOIN stores s ON o.store_id = s.store_id
SET o.commission_rate = s.grade_commission_rate
WHERE o.order_date = '2026-04-08'
  AND o.commission_rate IS NULL;
```

&nbsp;

### 임시 테이블 활용

```sql
-- 대량 UPDATE: 직접 UPDATE보다 임시 테이블이 빠른 경우

-- 1. 임시 테이블에 계산 결과 저장
CREATE TEMPORARY TABLE tmp_settlement AS
SELECT
    store_id,
    order_date,
    SUM(amount) AS total_amount,
    SUM(amount * commission_rate) AS commission
FROM orders
WHERE order_date = '2026-04-08'
GROUP BY store_id, order_date;

-- 2. 임시 테이블로 메인 테이블 업데이트
UPDATE settlement s
JOIN tmp_settlement t
  ON s.store_id = t.store_id AND s.settlement_date = t.order_date
SET s.total_amount = t.total_amount,
    s.commission = t.commission,
    s.status = 'CALCULATED';

-- 3. 정리
DROP TEMPORARY TABLE tmp_settlement;
```

&nbsp;

---

&nbsp;

## 6. 커넥션 풀: HikariCP 설정

&nbsp;

DB 커넥션은 비싸다. 매번 새로 만들면 느리고, 너무 많이 열면 DB가 죽는다.

&nbsp;

```yaml
spring:
  datasource:
    hikari:
      # 핵심 설정
      maximum-pool-size: 20        # 최대 커넥션 수
      minimum-idle: 5              # 유휴 커넥션 최소 유지
      connection-timeout: 3000     # 커넥션 못 얻으면 3초 후 에러
      idle-timeout: 600000         # 유휴 커넥션 10분 후 제거
      max-lifetime: 1800000        # 커넥션 최대 수명 30분
      leak-detection-threshold: 60000  # 60초 이상 반납 안 하면 경고 로그

      # 성능 설정
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
```

&nbsp;

**pool-size 공식:**

```
connections = (core_count * 2) + effective_spindle_count

예시:
- 4코어 서버 + SSD 1개 = (4 * 2) + 1 = 9
- 8코어 서버 + SSD 1개 = (8 * 2) + 1 = 17
- 대부분의 경우 10~20이 적당
```

&nbsp;

**흔한 실수:** pool size를 100으로 올리면 빨라질 거라 생각하지만, DB 쪽에서 동시 쿼리 100개를 감당 못 하면 오히려 느려진다.

&nbsp;

---

&nbsp;

## 7. 읽기/쓰기 분리

&nbsp;

하나의 DB로 읽기와 쓰기를 모두 감당하면 한계가 온다. **Master/Slave Replication**으로 분리한다.

```
             ┌──────────────┐
             │   Master DB  │  ← 쓰기 전용
             │  (Read/Write)│
             └──────┬───────┘
                    │ Replication (비동기)
          ┌─────────┼─────────┐
          ▼         ▼         ▼
     ┌─────────┐┌─────────┐┌─────────┐
     │ Slave 1 ││ Slave 2 ││ Slave 3 │  ← 읽기 전용
     │ (Read)  ││ (Read)  ││ (Read)  │
     └─────────┘└─────────┘└─────────┘
```

&nbsp;

### Spring에서 읽기/쓰기 분리

```java
// 라우팅 DataSource
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        boolean isReadOnly = TransactionSynchronizationManager
                .isCurrentTransactionReadOnly();
        return isReadOnly ? "slave" : "master";
    }
}

// 설정
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource routingDataSource(
            @Qualifier("masterDataSource") DataSource master,
            @Qualifier("slaveDataSource") DataSource slave) {

        RoutingDataSource routing = new RoutingDataSource();
        Map<Object, Object> targets = new HashMap<>();
        targets.put("master", master);
        targets.put("slave", slave);

        routing.setTargetDataSources(targets);
        routing.setDefaultTargetDataSource(master);
        return routing;
    }
}

// 사용
@Service
public class OrderService {

    @Transactional                // Master로 라우팅
    public void createOrder(Order order) { ... }

    @Transactional(readOnly = true)  // Slave로 라우팅
    public List<Order> findOrders(String storeId) { ... }
}
```

&nbsp;

---

&nbsp;

## 8. 락 최적화

&nbsp;

### 비관적 락 (Pessimistic Lock)

```java
// "다른 트랜잭션이 건드리지 못하게 잠근다"
@Query("SELECT s FROM Stock s WHERE s.productId = :productId")
@Lock(LockModeType.PESSIMISTIC_WRITE)
Stock findByProductIdForUpdate(@Param("productId") String productId);

// SQL: SELECT ... FOR UPDATE
// 장점: 충돌이 잦을 때 안전
// 단점: 락 대기 → 동시성 저하, 데드락 가능
```

&nbsp;

### 낙관적 락 (Optimistic Lock)

```java
@Entity
public class Stock {
    @Id
    private String productId;
    private int quantity;

    @Version   // 이게 핵심
    private Long version;
}

// 동작 방식:
// 1. SELECT productId, quantity, version FROM stock WHERE productId = 'P001'
//    → version = 5
// 2. UPDATE stock SET quantity = 99, version = 6
//    WHERE productId = 'P001' AND version = 5
//    → version이 5가 아니면 (다른 트랜잭션이 먼저 수정) → OptimisticLockException
```

```java
// 재시도 로직
@Retryable(
    retryFor = OptimisticLockingFailureException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 100)
)
@Transactional
public void decreaseStock(String productId, int quantity) {
    Stock stock = stockRepository.findById(productId)
            .orElseThrow();
    stock.decrease(quantity);
}
```

&nbsp;

**어떤 락을 쓸까?**

| 상황 | 추천 |
|:---|:---:|
| 충돌이 자주 발생 (재고 차감 등) | 비관적 락 |
| 충돌이 거의 없음 (프로필 수정 등) | 낙관적 락 |
| 배치 처리 (대량 UPDATE) | 비관적 락 또는 락 없이 범위 분리 |

&nbsp;

---

&nbsp;

## 9. 페이지네이션: OFFSET 대신 커서 기반

&nbsp;

### OFFSET 방식의 문제

```sql
-- 100만 번째부터 20건: 앞의 100만 건을 스캔한 후 버린다
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 1000000;
-- 실행시간: 3.2초 (뒤로 갈수록 느려짐)
```

&nbsp;

### 커서 기반 (Keyset Pagination)

```sql
-- "마지막으로 본 ID 다음부터 20건"
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 20;
-- 실행시간: 0.003초 (항상 일정)
```

&nbsp;

### Spring에서 구현

```java
// OFFSET 방식 (느림)
@GetMapping("/orders")
public Page<Order> getOrders(@RequestParam int page, @RequestParam int size) {
    return orderRepository.findAll(PageRequest.of(page, size, Sort.by("id")));
}

// 커서 방식 (빠름)
@GetMapping("/orders")
public List<Order> getOrders(
        @RequestParam(required = false) Long lastId,
        @RequestParam(defaultValue = "20") int size) {

    if (lastId == null) {
        return orderRepository.findTop20ByOrderByIdAsc();
    }
    return orderRepository.findByIdGreaterThanOrderByIdAsc(lastId, PageRequest.of(0, size));
}
```

```java
// Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByIdGreaterThanOrderByIdAsc(Long lastId, Pageable pageable);
}
```

&nbsp;

---

&nbsp;

## 10. N+1 해결

&nbsp;

### 문제

```java
// 주문 10건을 조회하면...
List<Order> orders = orderRepository.findAll();  // 쿼리 1회

for (Order order : orders) {
    order.getItems().size();  // 쿼리 10회 (주문별로 1번씩)
}
// 총 11회 쿼리 실행 → "N+1 문제"
```

&nbsp;

### 해결 1: Fetch Join

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.storeId = :storeId")
List<Order> findByStoreIdWithItems(@Param("storeId") String storeId);
// 쿼리 1회로 주문 + 아이템 모두 로딩
```

&nbsp;

### 해결 2: @EntityGraph

```java
@EntityGraph(attributePaths = {"items", "store"})
@Query("SELECT o FROM Order o WHERE o.storeId = :storeId")
List<Order> findByStoreIdWithGraph(@Param("storeId") String storeId);
```

&nbsp;

### 해결 3: 배치 사이즈

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
        # items를 100건씩 IN 쿼리로 묶어서 조회
        # SELECT * FROM order_items WHERE order_id IN (1,2,3,...,100)
```

```java
// 엔티티에 직접 설정
@Entity
public class Order {
    @OneToMany(mappedBy = "order")
    @BatchSize(size = 100)
    private List<OrderItem> items;
}
```

&nbsp;

**해결 방법 비교:**

| 방법 | 장점 | 단점 |
|:---:|:---|:---|
| Fetch Join | 쿼리 1회 | 페이징 불가, 카르테시안 곱 주의 |
| @EntityGraph | 선언적, 깔끔 | 복잡한 조건 어려움 |
| 배치 사이즈 | 설정만으로 해결 | 최적은 아닐 수 있음 |

&nbsp;

---

&nbsp;

## 정리

&nbsp;

| 기법 | 적용 시점 | 효과 |
|:---:|:---|:---|
| EXPLAIN | 항상 | 문제 진단의 시작 |
| 복합 인덱스 | 조건절에 2개 이상 컬럼 | 10~100배 |
| 커버링 인덱스 | SELECT 컬럼이 적을 때 | 5~50배 |
| 파티셔닝 | 1,000만 건 이상 | 5~20배 |
| 벌크 연산 | 대량 INSERT/UPDATE | 10~50배 |
| 읽기/쓰기 분리 | 읽기 비율 높을 때 | 2~3배 |
| 커서 페이지네이션 | 깊은 페이지 접근 | 100~1000배 |
| N+1 해결 | JPA 사용 시 항상 | 10~100배 |

&nbsp;

DB 최적화는 끝이 없지만, 위의 기법만 알아도 대부분의 상황을 커버할 수 있다. 핵심은 **측정 → 진단 → 개선** 순서를 지키는 것이다. 감으로 최적화하지 말고, EXPLAIN부터 찍어보자.

&nbsp;

---

&nbsp;

**다음 편 예고:**

6편에서는 **실시간 스트리밍**을 다룬다. Kafka Streams와 Apache Flink로 실시간 데이터를 윈도우 집계하고, CDC(Change Data Capture)로 DB 변경을 실시간 스트리밍하는 방법을 코드와 함께 살펴본다.

&nbsp;

---

`#DB최적화` `#SQL튜닝` `#인덱스` `#파티셔닝` `#N+1` `#HikariCP` `#JPA` `#페이지네이션` `#대규모데이터` `#데이터엔지니어링`
