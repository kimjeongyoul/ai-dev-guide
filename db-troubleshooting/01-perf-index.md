# 인덱스 — SELECT가 느릴 때 개발자가 할 수 있는 것

&nbsp;

"목록 페이지가 5초 걸려요."

&nbsp;

어느 날 기획자에게 이 말을 들었다. 목록 조회인데 5초? 뭔가 잘못됐다.

&nbsp;

쿼리를 열어봤다. 별거 없는 SELECT였다. 테이블에 데이터가 300만 건 쌓여 있었을 뿐.

&nbsp;

문제는 **인덱스가 없었다.**

&nbsp;

&nbsp;

---

&nbsp;

## 1. 일단 EXPLAIN부터 찍자

&nbsp;

느린 쿼리를 만나면 가장 먼저 할 일. EXPLAIN을 앞에 붙인다.

&nbsp;

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 12345 ORDER BY created_at DESC;
```

&nbsp;

```
+----+-------------+--------+------+---------------+------+---------+------+---------+-----------------------------+
| id | select_type | table  | type | possible_keys | key  | key_len | ref  | rows    | Extra                       |
+----+-------------+--------+------+---------------+------+---------+------+---------+-----------------------------+
|  1 | SIMPLE      | orders | ALL  | NULL          | NULL | NULL    | NULL | 3120458 | Using where; Using filesort |
+----+-------------+--------+------+---------------+------+---------+------+---------+-----------------------------+
```

&nbsp;

이 결과를 읽을 줄 알아야 한다. 핵심 컬럼 3개만 보면 된다.

&nbsp;

### type — 어떻게 읽는가

&nbsp;

| type | 의미 | 속도 |
|------|------|------|
| `const` | PK 또는 유니크로 1건 조회 | 가장 빠름 |
| `ref` | 인덱스로 여러 건 조회 | 빠름 |
| `range` | 인덱스 범위 스캔 (BETWEEN, >, <) | 괜찮음 |
| `index` | 인덱스 전체 스캔 | 느림 |
| `ALL` | 테이블 풀스캔 | **가장 느림** |

&nbsp;

위 결과에서 type이 `ALL`이다. **300만 건을 처음부터 끝까지 다 읽고 있다.**

&nbsp;

### rows — 몇 건을 스캔하는가

&nbsp;

`rows: 3,120,458` — 300만 건 전부 스캔. 10건만 필요한데 300만 건을 뒤지고 있다.

&nbsp;

### Extra — 추가 작업

&nbsp;

| Extra | 의미 |
|-------|------|
| `Using where` | WHERE 조건으로 필터링 중 |
| `Using index` | **커버링 인덱스** 사용 (좋음) |
| `Using filesort` | **정렬을 위해 추가 작업** (느림) |
| `Using temporary` | **임시 테이블 생성** (느림) |

&nbsp;

`Using filesort`가 보인다. ORDER BY를 인덱스 없이 하고 있다는 뜻이다.

&nbsp;

&nbsp;

---

&nbsp;

## 2. 인덱스의 원리 — B-Tree

&nbsp;

인덱스를 이해하는 가장 쉬운 비유.

&nbsp;

```
500페이지짜리 책에서 "트랜잭션"이라는 단어를 찾아야 한다.

방법 1: 1페이지부터 500페이지까지 한 장씩 넘기며 찾기 → 풀스캔
방법 2: 뒤쪽 색인(Index)에서 "트" 찾기 → 해당 페이지로 점프 → 인덱스 스캔
```

&nbsp;

DB의 인덱스도 같은 원리다. B-Tree라는 자료구조로 데이터를 **정렬된 상태로** 유지한다.

&nbsp;

```
도서관 비유:

인덱스 없는 테이블 = 책이 아무렇게나 꽂혀있는 서가
  → "김영희"의 책 찾기: 처음부터 끝까지 다 봐야 함

인덱스 있는 테이블 = 저자 이름순으로 정렬된 서가
  → "김" 섹션으로 가서 "김영희" 찾기: 몇 초면 끝
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. 인덱스 종류

&nbsp;

### 단일 인덱스

&nbsp;

하나의 컬럼에 인덱스를 건다.

&nbsp;

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

&nbsp;

`WHERE user_id = 12345` 같은 쿼리가 빨라진다.

&nbsp;

### 복합 인덱스

&nbsp;

여러 컬럼을 묶어서 인덱스를 건다. **순서가 중요하다.**

&nbsp;

```sql
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);
```

&nbsp;

`WHERE user_id = ? AND created_at > ?` — 이 쿼리에 최적화된다.

&nbsp;

### 커버링 인덱스

&nbsp;

인덱스만 읽고 응답할 수 있는 경우. **테이블을 아예 안 간다.**

&nbsp;

```sql
-- 인덱스: (user_id, created_at)
SELECT user_id, created_at FROM orders WHERE user_id = 12345;
-- → 인덱스에 필요한 데이터가 다 있으므로 테이블 접근 불필요
-- → EXPLAIN Extra: "Using index"
```

&nbsp;

### 유니크 인덱스

&nbsp;

중복을 방지하면서 조회도 빠르게.

&nbsp;

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

&nbsp;

`WHERE email = ?` 조회 시 최대 1건이므로 const 수준의 속도.

&nbsp;

&nbsp;

---

&nbsp;

## 4. 복합 인덱스 순서의 중요성

&nbsp;

이걸 모르면 인덱스를 걸어놓고도 안 타는 경우가 생긴다.

&nbsp;

```sql
CREATE INDEX idx_name_age ON users(name, age);
```

&nbsp;

```
인덱스는 왼쪽부터 매칭한다. 전화번호부와 같다.

전화번호부: 성 → 이름 순으로 정렬
  "김" + "영희" → 바로 찾음 ✅
  "김" → "김"씨 전체 범위 찾음 ✅
  "영희" → 성을 모르면 처음부터 다 봐야 함 ❌
```

&nbsp;

| 쿼리 | 인덱스 사용 여부 |
|-------|------------------|
| `WHERE name = '김영희' AND age = 30` | 사용 (두 컬럼 모두 매칭) |
| `WHERE name = '김영희'` | 사용 (왼쪽 컬럼 매칭) |
| `WHERE age = 30` | **사용 안 함** (왼쪽이 빠짐) |
| `WHERE age = 30 AND name = '김영희'` | 사용 (옵티마이저가 순서 변경) |

&nbsp;

**규칙: 복합 인덱스는 왼쪽부터 빠짐없이 사용해야 탄다.**

&nbsp;

&nbsp;

---

&nbsp;

## 5. 인덱스가 안 타는 경우 7가지

&nbsp;

인덱스를 걸어놨는데 EXPLAIN 찍어보면 ALL(풀스캔). 왜?

&nbsp;

### (1) LIKE '%검색어%' — 앞에 와일드카드

&nbsp;

```sql
-- ❌ 인덱스 안 탐
SELECT * FROM users WHERE name LIKE '%김%';

-- ✅ 인덱스 탐
SELECT * FROM users WHERE name LIKE '김%';
```

&nbsp;

앞에 `%`를 붙이면 "어디서 시작하는지 모르니까" 처음부터 다 봐야 한다.

&nbsp;

### (2) 컬럼에 함수를 씌움

&nbsp;

```sql
-- ❌ 인덱스 안 탐
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- ✅ 인덱스 탐
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

&nbsp;

함수를 씌우면 인덱스의 정렬 순서가 깨진다. **컬럼을 가공하지 말고, 값을 가공하라.**

&nbsp;

### (3) 부정 조건 (!=, NOT IN)

&nbsp;

```sql
-- ❌ 인덱스 비효율적
SELECT * FROM users WHERE status != 'DELETED';

-- ✅ 대안
SELECT * FROM users WHERE status IN ('ACTIVE', 'SUSPENDED');
```

&nbsp;

### (4) IS NULL (설정에 따라)

&nbsp;

```sql
-- 인덱스에 NULL이 포함되지 않는 DB도 있다 (Oracle)
SELECT * FROM users WHERE phone IS NULL;
```

&nbsp;

MySQL의 InnoDB는 NULL도 인덱스에 포함하지만, 습관적으로 NOT NULL + 기본값을 쓰는 게 안전하다.

&nbsp;

### (5) OR 조건

&nbsp;

```sql
-- ❌ 인덱스 비효율적 (name에만 인덱스 있을 때)
SELECT * FROM users WHERE name = '김영희' OR email = 'kim@test.com';

-- ✅ 각각 인덱스가 있으면 UNION으로
SELECT * FROM users WHERE name = '김영희'
UNION
SELECT * FROM users WHERE email = 'kim@test.com';
```

&nbsp;

### (6) 타입 불일치

&nbsp;

```sql
-- user_id가 VARCHAR인데 숫자로 비교
-- ❌ 암묵적 형변환 → 인덱스 안 탐
SELECT * FROM users WHERE user_id = 12345;

-- ✅ 타입 맞추기
SELECT * FROM users WHERE user_id = '12345';
```

&nbsp;

이거 의외로 많이 발생한다. 특히 MyBatis에서 파라미터 타입을 안 맞추는 경우.

&nbsp;

### (7) 데이터의 20% 이상을 읽을 때

&nbsp;

```sql
-- status = 'ACTIVE'가 전체의 90%인 경우
SELECT * FROM users WHERE status = 'ACTIVE';
-- → 옵티마이저: "어차피 거의 다 읽을 거, 풀스캔이 더 빠르겠다"
-- → 인덱스 무시하고 풀스캔
```

&nbsp;

옵티마이저가 "인덱스 타는 것보다 풀스캔이 빠르다"고 판단하면 인덱스를 안 탄다. 이건 정상이다.

&nbsp;

&nbsp;

---

&nbsp;

## 6. 실전 Before/After

&nbsp;

### 사례 1: 주문 목록 조회

&nbsp;

```sql
-- 원래 쿼리
SELECT * FROM orders WHERE user_id = 12345 ORDER BY created_at DESC LIMIT 20;
```

&nbsp;

**Before — 인덱스 없음:**

&nbsp;

```
EXPLAIN 결과:
  type: ALL
  rows: 3,120,458
  Extra: Using where; Using filesort

실행 시간: 3.2초
```

&nbsp;

```sql
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);
```

&nbsp;

**After — 복합 인덱스 추가:**

&nbsp;

```
EXPLAIN 결과:
  type: ref
  rows: 47
  Extra: Using where; Using index

실행 시간: 0.05초
```

&nbsp;

**3.2초 → 0.05초. 64배 빨라졌다.**

&nbsp;

user_id로 필터링하고, created_at으로 정렬하니까 복합 인덱스 하나로 두 마리 토끼를 잡았다.

&nbsp;

### 사례 2: 복합 인덱스 순서 변경

&nbsp;

```sql
-- 쿼리
SELECT * FROM products WHERE category_id = 5 AND status = 'ACTIVE' ORDER BY price ASC;

-- 잘못된 인덱스 (순서가 안 맞음)
CREATE INDEX idx_products_bad ON products(status, price, category_id);
```

&nbsp;

**Before:**

&nbsp;

```
EXPLAIN 결과:
  type: index
  rows: 580,000
  Extra: Using where

실행 시간: 1.5초
```

&nbsp;

```sql
-- 올바른 인덱스 (WHERE 조건 → ORDER BY 순서)
DROP INDEX idx_products_bad ON products;
CREATE INDEX idx_products_good ON products(category_id, status, price);
```

&nbsp;

**After:**

&nbsp;

```
EXPLAIN 결과:
  type: ref
  rows: 1,240
  Extra: Using where; Using index

실행 시간: 0.03초
```

&nbsp;

**1.5초 → 0.03초. 50배 빨라졌다.**

&nbsp;

인덱스 순서 규칙: **등호(=) 조건 컬럼을 앞에, 범위/정렬 컬럼을 뒤에.**

&nbsp;

&nbsp;

---

&nbsp;

## 7. 인덱스를 과하게 넣으면?

&nbsp;

인덱스는 공짜가 아니다.

&nbsp;

```
인덱스 1개 = 정렬된 복사본 1개

테이블에 INSERT할 때:
  → 테이블에 1건 추가
  → 인덱스 A에도 1건 추가 (정렬 유지하면서)
  → 인덱스 B에도 1건 추가
  → 인덱스 C에도 1건 추가
  → ...

인덱스 10개 = INSERT 시 11번 쓰기
```

&nbsp;

| 상황 | 인덱스 많으면 |
|------|---------------|
| SELECT | 빨라짐 |
| INSERT | 느려짐 |
| UPDATE | 느려짐 (인덱스 컬럼 수정 시) |
| DELETE | 느려짐 |
| 디스크 | 더 많이 사용 |

&nbsp;

**읽기 중심 서비스** (조회 90%, 쓰기 10%): 인덱스 많이 걸어도 OK.

**쓰기 중심 서비스** (로그, IoT): 인덱스 최소한으로.

&nbsp;

&nbsp;

---

&nbsp;

## 8. 실무 팁 정리

&nbsp;

```
1. 느린 쿼리 → EXPLAIN 먼저 찍는다
2. type이 ALL이면 인덱스를 의심한다
3. WHERE, ORDER BY, JOIN ON에 쓰는 컬럼 위주로 인덱스를 건다
4. 복합 인덱스는 등호 조건을 앞에, 범위/정렬을 뒤에
5. 인덱스 걸고 나서 반드시 EXPLAIN으로 확인한다
6. 함수 씌우지 않는다 (YEAR(), DATE() 등)
7. 사용하지 않는 인덱스는 정기적으로 정리한다
```

&nbsp;

```sql
-- MySQL: 사용하지 않는 인덱스 확인
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = 'your_db';
```

&nbsp;

&nbsp;

---

&nbsp;

다음 편 예고: **슬로우 쿼리 잡기 — 쿼리 하나가 서비스를 멈추는 순간**

인덱스만으로는 부족하다. 서브쿼리 지옥, ORDER BY + LIMIT 함정, SELECT * 남용, 그리고 OFFSET 페이지네이션의 비밀. 실제 슬로우 쿼리 로그를 열어서 하나씩 잡아본다.

&nbsp;

&nbsp;

---

인덱스, B-Tree, EXPLAIN, 실행계획, 복합인덱스, 커버링인덱스, 풀스캔, 쿼리최적화, MySQL, 성능튜닝, 슬로우쿼리, WHERE, ORDER BY, 데이터베이스