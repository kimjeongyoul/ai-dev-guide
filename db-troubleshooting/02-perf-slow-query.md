# 슬로우 쿼리 잡기 — 쿼리 하나가 서비스를 멈추는 순간

&nbsp;

"갑자기 서비스가 느려졌어요."

&nbsp;

코드 변경한 적 없다. 서버 재시작해도 똑같다. 뭐가 문제일까?

&nbsp;

DB에 들어가서 슬로우 쿼리 로그를 열었다. 쿼리 하나가 28초 걸리고 있었다. 그리고 그 쿼리가 1초에 50번씩 실행되고 있었다.

&nbsp;

**쿼리 하나가 서비스 전체를 죽이고 있었다.**

&nbsp;

&nbsp;

---

&nbsp;

## 1. 슬로우 쿼리 로그 켜기

&nbsp;

느린 쿼리를 잡으려면 일단 기록해야 한다.

&nbsp;

### MySQL

&nbsp;

```sql
-- 현재 설정 확인
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 슬로우 쿼리 로그 활성화
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;  -- 1초 이상 걸리는 쿼리 기록
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

&nbsp;

```ini
# my.cnf (영구 설정)
[mysqld]
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /var/log/mysql/slow.log
log_queries_not_using_indexes = 1   # 인덱스 안 타는 쿼리도 기록
```

&nbsp;

### PostgreSQL

&nbsp;

```ini
# postgresql.conf
log_min_duration_statement = 1000   # 1000ms = 1초 이상 기록
log_statement = 'none'              # 전체 로그는 끄고
```

&nbsp;

### 로그 분석 도구

&nbsp;

```bash
# MySQL: mysqldumpslow로 요약
mysqldumpslow -t 10 /var/log/mysql/slow.log
# → 상위 10개 슬로우 쿼리를 빈도/시간 순으로 정리

# 출력 예시:
# Count: 4820  Time=28.30s  Lock=0.00s  Rows=15.2
# SELECT * FROM orders WHERE user_id = N ORDER BY created_at DESC
```

&nbsp;

&nbsp;

---

&nbsp;

## 2. 슬로우 쿼리 원인 Top 5

&nbsp;

### 원인 1: 풀스캔 (인덱스 없음)

&nbsp;

1편에서 다뤘다. EXPLAIN 찍어서 type이 ALL이면 인덱스를 건다.

&nbsp;

이건 넘어가고, 인덱스로 안 풀리는 것들을 보자.

&nbsp;

&nbsp;

---

&nbsp;

### 원인 2: 서브쿼리 지옥

&nbsp;

```sql
-- ❌ 서브쿼리: 주문이 있는 사용자 조회
SELECT *
FROM users u
WHERE u.id IN (
    SELECT o.user_id
    FROM orders o
    WHERE o.total_amount > 100000
);
```

&nbsp;

EXPLAIN을 찍어보면:

&nbsp;

```
+----+--------------------+--------+------+---------+---------+
| id | select_type        | table  | type | rows    | Extra   |
+----+--------------------+--------+------+---------+---------+
|  1 | PRIMARY            | users  | ALL  | 520000  |         |
|  2 | DEPENDENT SUBQUERY | orders | ALL  | 3100000 |         |
+----+--------------------+--------+------+---------+---------+
```

&nbsp;

**`DEPENDENT SUBQUERY`가 보이면 위험하다.** users의 매 행마다 서브쿼리를 실행한다는 뜻이다.

52만 x 310만 = 조 단위 연산.

&nbsp;

```sql
-- ✅ JOIN으로 변환
SELECT DISTINCT u.*
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.total_amount > 100000;
```

&nbsp;

```
+----+-------------+--------+------+-------------------+---------+
| id | select_type | table  | type | rows              | Extra   |
+----+-------------+--------+------+-------------------+---------+
|  1 | SIMPLE      | o      | range| 45000             |         |
|  1 | SIMPLE      | u      | eq_ref| 1                |         |
+----+-------------+--------+------+-------------------+---------+
```

&nbsp;

DEPENDENT SUBQUERY가 사라지고, SIMPLE JOIN으로 바뀌었다.

&nbsp;

&nbsp;

---

&nbsp;

### 원인 3: ORDER BY + LIMIT 함정

&nbsp;

```sql
SELECT * FROM articles ORDER BY created_at DESC LIMIT 10;
```

&nbsp;

10건만 가져오니까 빠를 것 같다? **아니다.**

&nbsp;

인덱스가 없으면:

&nbsp;

```
1. 테이블 전체를 읽는다 (100만 건)
2. created_at으로 전부 정렬한다 (메모리 또는 디스크에서)
3. 상위 10건만 반환한다
```

&nbsp;

```
EXPLAIN 결과:
  type: ALL
  rows: 1,050,000
  Extra: Using filesort
```

&nbsp;

**100만 건을 정렬하고 10건만 주는 꼴이다.**

&nbsp;

```sql
CREATE INDEX idx_articles_created ON articles(created_at);
```

&nbsp;

```
EXPLAIN 결과 (인덱스 추가 후):
  type: index
  rows: 10
  Extra: NULL
```

&nbsp;

인덱스가 이미 정렬되어 있으니까, 뒤에서 10건만 읽으면 된다.

&nbsp;

&nbsp;

---

&nbsp;

### 원인 4: JOIN 순서 문제

&nbsp;

```sql
SELECT o.*, u.name
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE u.grade = 'VIP';
```

&nbsp;

옵티마이저가 orders(300만 건)를 먼저 읽고 users와 매칭하면 느리다.

users에서 VIP(500명)를 먼저 뽑고 orders와 매칭하면 빠르다.

&nbsp;

보통 옵티마이저가 알아서 잘 하지만, 통계 정보가 오래되면 잘못된 순서를 선택한다.

&nbsp;

```sql
-- MySQL: 통계 정보 갱신
ANALYZE TABLE orders;
ANALYZE TABLE users;

-- 그래도 안 되면 순서 강제 (MySQL 전용)
SELECT o.*, u.name
FROM users u
STRAIGHT_JOIN orders o ON o.user_id = u.id
WHERE u.grade = 'VIP';
```

&nbsp;

&nbsp;

---

&nbsp;

### 원인 5: SELECT * 남용

&nbsp;

```sql
-- ❌ 모든 컬럼
SELECT * FROM articles WHERE category_id = 3;

-- ✅ 필요한 컬럼만
SELECT id, title, created_at FROM articles WHERE category_id = 3;
```

&nbsp;

왜 차이가 나는가?

&nbsp;

```
articles 테이블:
  id          INT           4 bytes
  title       VARCHAR(200)  ~200 bytes
  content     TEXT          ~5,000 bytes (평균)
  thumbnail   BLOB          ~50,000 bytes (평균)
  created_at  DATETIME      8 bytes
  ...

SELECT * → 행당 ~55KB 읽기 → 1,000건 = 55MB
필요 컬럼만 → 행당 ~212B 읽기 → 1,000건 = 212KB
```

&nbsp;

**응답 크기가 250배 차이.**

특히 TEXT, BLOB 컬럼이 있는 테이블에서 SELECT *는 치명적이다.

&nbsp;

&nbsp;

---

&nbsp;

## 3. 실전 사례 3개

&nbsp;

### 사례 1: 관리자 대시보드 — 30초 → 0.5초

&nbsp;

```sql
-- 원래 쿼리: 이번 달 매출 + 주문 수 + 신규 회원 수
SELECT
    (SELECT SUM(total_amount) FROM orders WHERE MONTH(created_at) = MONTH(NOW())) AS monthly_sales,
    (SELECT COUNT(*) FROM orders WHERE MONTH(created_at) = MONTH(NOW())) AS monthly_orders,
    (SELECT COUNT(*) FROM users WHERE MONTH(created_at) = MONTH(NOW())) AS new_users;
```

&nbsp;

EXPLAIN:

&nbsp;

```
3개의 DEPENDENT SUBQUERY, 각각 풀스캔
orders: rows 3,100,000 (x2)
users: rows 520,000
실행 시간: 30.2초
```

&nbsp;

**문제:** MONTH() 함수가 인덱스를 막고, 매번 풀스캔.

&nbsp;

```sql
-- 개선: 날짜 범위 조건 + JOIN 제거
SELECT
    SUM(o.total_amount) AS monthly_sales,
    COUNT(o.id) AS monthly_orders,
    (SELECT COUNT(*) FROM users
     WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01') AS new_users
FROM orders o
WHERE o.created_at >= '2024-03-01' AND o.created_at < '2024-04-01';
```

&nbsp;

```sql
-- 인덱스 추가
CREATE INDEX idx_orders_created ON orders(created_at);
CREATE INDEX idx_users_created ON users(created_at);
```

&nbsp;

```
EXPLAIN (개선 후):
  orders: type=range, rows=48,500
  users: type=range, rows=5,200
실행 시간: 0.5초
```

&nbsp;

**30.2초 → 0.5초. 60배 개선.**

&nbsp;

### 사례 2: 목록 페이지 ORDER BY — 2.1초 → 0.1초

&nbsp;

```sql
SELECT id, title, author, created_at
FROM posts
WHERE status = 'PUBLISHED'
ORDER BY created_at DESC
LIMIT 20;
```

&nbsp;

```
EXPLAIN:
  type: ALL
  rows: 850,000
  Extra: Using where; Using filesort
실행 시간: 2.1초
```

&nbsp;

```sql
CREATE INDEX idx_posts_status_created ON posts(status, created_at);
```

&nbsp;

```
EXPLAIN (개선 후):
  type: ref
  rows: 20
  Extra: Using where; Backward index scan
실행 시간: 0.1초
```

&nbsp;

**2.1초 → 0.1초. 21배 개선.**

&nbsp;

### 사례 3: SELECT * → 필요 컬럼만 — 응답 크기 80% 감소

&nbsp;

```sql
-- Before
SELECT * FROM products WHERE category_id = 7;
-- 응답: 2.8MB, 실행: 0.8초

-- After
SELECT id, name, price, stock FROM products WHERE category_id = 7;
-- 응답: 0.5MB, 실행: 0.3초
```

&nbsp;

description(TEXT)과 images(JSON) 컬럼이 응답의 80%를 차지하고 있었다.

목록에서는 필요 없는 데이터.

&nbsp;

&nbsp;

---

&nbsp;

## 4. 페이지네이션: OFFSET의 함정

&nbsp;

목록 페이지에서 흔히 쓰는 OFFSET 방식.

&nbsp;

```sql
-- 1페이지
SELECT * FROM articles ORDER BY id DESC LIMIT 20 OFFSET 0;

-- 100페이지
SELECT * FROM articles ORDER BY id DESC LIMIT 20 OFFSET 1980;

-- 5000페이지
SELECT * FROM articles ORDER BY id DESC LIMIT 20 OFFSET 99980;
```

&nbsp;

OFFSET이 커질수록 느려진다. 왜?

&nbsp;

```
OFFSET 99980, LIMIT 20의 실행 과정:
  1. 인덱스에서 99,980 + 20 = 100,000건을 읽는다
  2. 앞의 99,980건을 버린다
  3. 나머지 20건을 반환한다

  → 99,980건을 읽고 버리는 낭비
```

&nbsp;

```
EXPLAIN:
  type: index
  rows: 100,000
실행 시간: 1.2초
```

&nbsp;

### 커서 기반 페이지네이션

&nbsp;

```sql
-- 이전 페이지의 마지막 ID를 기억해서 사용
SELECT * FROM articles
WHERE id < 500021
ORDER BY id DESC
LIMIT 20;
```

&nbsp;

```
EXPLAIN:
  type: range
  rows: 20
실행 시간: 0.01초
```

&nbsp;

**1.2초 → 0.01초. 120배 빨라졌다.**

&nbsp;

```
OFFSET 방식:
  페이지 1 → 20건 읽기 (빠름)
  페이지 100 → 2,000건 읽기 (괜찮음)
  페이지 5000 → 100,000건 읽기 (느림)
  페이지 50000 → 1,000,000건 읽기 (죽음)

커서 방식:
  어떤 페이지든 → 20건만 읽기 (항상 빠름)
```

&nbsp;

### 구현 비교

&nbsp;

```javascript
// OFFSET 방식 API
// GET /articles?page=5000&size=20
const offset = (page - 1) * size;
const query = `SELECT * FROM articles ORDER BY id DESC LIMIT ${size} OFFSET ${offset}`;

// 커서 방식 API
// GET /articles?cursor=500021&size=20
const query = `SELECT * FROM articles WHERE id < ${cursor} ORDER BY id DESC LIMIT ${size}`;
```

&nbsp;

| 방식 | 장점 | 단점 |
|------|------|------|
| OFFSET | 특정 페이지 점프 가능 | 뒤로 갈수록 느림 |
| 커서 | 항상 일정한 속도 | 특정 페이지 점프 불가 |

&nbsp;

무한 스크롤, 타임라인 같은 UI는 커서 방식이 맞다. 게시판처럼 "10페이지로 이동"이 필요하면 OFFSET을 쓰되, 최대 페이지를 제한하는 게 현실적이다.

&nbsp;

&nbsp;

---

&nbsp;

## 5. 쿼리 튜닝 체크리스트

&nbsp;

```
□ EXPLAIN 찍었는가?
□ type이 ALL이 아닌가?
□ DEPENDENT SUBQUERY가 없는가?
□ Using filesort가 없는가? (또는 인덱스로 정렬 가능한가?)
□ SELECT *를 쓰고 있지 않은가?
□ 컬럼에 함수를 씌우고 있지 않은가?
□ OFFSET이 1만 이상 되지 않는가?
□ 불필요한 서브쿼리를 JOIN으로 바꿀 수 있는가?
□ 인덱스 추가 후 EXPLAIN으로 확인했는가?
□ 통계 정보가 최신인가? (ANALYZE TABLE)
```

&nbsp;

&nbsp;

---

&nbsp;

다음 편 예고: **캐시 한 줄이 DB 부하를 90% 줄인다 — Redis 실전 적용**

쿼리를 아무리 튜닝해도 한계가 있다. 같은 데이터를 초당 1,000번 읽으면 DB가 버틴다고? 안 버틴다. 캐시를 넣으면 DB 접근 자체를 95% 줄일 수 있다. Cache-Aside, TTL, 캐시 무효화 전략을 실전 코드와 함께 정리한다.

&nbsp;

&nbsp;

---

슬로우쿼리, slow query, EXPLAIN, 서브쿼리, JOIN, ORDER BY, LIMIT, OFFSET, 커서페이지네이션, SELECT *, 쿼리튜닝, MySQL, PostgreSQL, 성능최적화, 데이터베이스