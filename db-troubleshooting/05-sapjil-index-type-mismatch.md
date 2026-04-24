# [실전 삽질 1편] 인덱스는 죄가 없다 — 타입 불일치가 부른 Full Table Scan의 비극과 옵티마이저 트레이스 분석

&nbsp;

"CPU 99%, DB 서버 응답 불가. 원인은 슬로우 쿼리."

&nbsp;

대규모 트래픽이 몰리는 커머스 플랫폼에서, 특정 이벤트 알림톡 발송 직후 데이터베이스 서버가 완전히 주저앉는 장애가 발생했다. 트래픽 스파이크(Spike)가 원인이었을까? 하지만 모니터링 대시보드가 가리키는 지표는 달랐다. 커넥션 풀(Connection Pool)은 가득 찼고, I/O Wait는 하늘을 찔렀다. 

&nbsp;

장애 복구 후 슬로우 쿼리 로그(Slow Query Log)를 열어보니, 500만 건 규모의 `orders` 테이블에서 단일 주문 건을 조회하는 아주 단순한 `SELECT` 문 하나가 4초 이상의 실행 시간을 기록하며 모든 병목의 근원이 되고 있었다.

&nbsp;

해당 컬럼에는 분명히 Unique Index가 걸려 있었고 카디널리티(Cardinality) 역시 완벽했다. 하지만 MySQL 옵티마이저는 이 완벽한 인덱스를 무시하고 'Full Table Scan'을 선택했다. 그 이유는 비즈니스 로직 어딘가에서 섞여 들어간 단순한 '따옴표' 하나, 즉 타입 불일치(Type Mismatch) 때문이었다. 본 글에서는 이 허무한 장애의 원인을 MySQL의 옵티마이저 내부 동작 메커니즘과 함께 심도 있게 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 범죄 현장 재구성: BIGINT와 String의 어긋난 만남

&nbsp;

장애를 일으킨 테이블 구조와 쿼리의 실체는 다음과 같았다.

&nbsp;

```sql
-- 테이블 스키마 (MySQL 8.0)
CREATE TABLE orders (
    order_no BIGINT NOT NULL,        -- 주문 번호 (숫자형)
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (order_no)           -- PK (Clustered Index)
) ENGINE=InnoDB;

-- 장애 발생 쿼리
SELECT * FROM orders WHERE order_no = '2024042100010522';
```

&nbsp;

개발자들은 흔히 "MySQL은 알아서 캐스팅(Casting)을 해주니까, 따옴표가 붙든 안 붙든 값이 같으면 인덱스를 태워주겠지"라고 낙관한다. 실제로 작은 테이블에서는 속도 차이를 체감할 수 없기에 이 치명적인 안티 패턴은 코드 리뷰를 무사히 통과한다. 하지만 이 '암시적 형변환(Implicit Conversion)'은 데이터가 수백만 건 단위로 쌓였을 때 시스템을 붕괴시키는 시한폭탄이 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 기술적 원인: Optimizer Trace로 들여다본 엔진의 속마음

&nbsp;

왜 인덱스를 타지 않았는지 추측이 아닌 데이터로 증명하기 위해, MySQL의 `optimizer_trace`를 활성화하여 실행 계획을 생성하는 과정을 낱낱이 해부해 보았다.

&nbsp;

```sql
SET optimizer_trace="enabled=on";
SELECT * FROM orders WHERE order_no = '2024042100010522';
SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
```

&nbsp;

추출된 트레이스 로그 중 `row_estimation` 섹션에서 인덱스가 배제된 정확한 사유가 나타났다.

&nbsp;

```json
{
  "potential_range_indices": [
    {
      "index": "PRIMARY",
      "usable": false,
      "cause": "cannot_convert_string_to_number_safely"
    }
  ]
}
```

&nbsp;

## 2-1. 형변환의 주체와 인덱스 무력화 (Sargability 파괴)
MySQL의 비교 연산 규칙에 따르면, 숫자(BIGINT)와 문자열(VARCHAR)을 비교할 때 우선순위는 **숫자**에 있다. 즉, 양쪽 모두를 숫자로 변환한 뒤 비교한다. 파라미터로 넘어온 `'2024042100010522'`를 숫자로 바꾸는 것은 상수 변환 1회로 끝나므로 문제가 되지 않는다.

&nbsp;

하지만 만약 파라미터가 숫자이고, 인덱스가 걸린 컬럼이 문자열(`VARCHAR`)이라면 어떻게 될까?
```sql
-- 컬럼이 VARCHAR이고, 파라미터가 숫자(BIGINT)인 경우의 내부 동작
SELECT * FROM orders WHERE CAST(varchar_order_no AS SIGNED) = 2024042100010522;
```
DB 엔진은 문자열 컬럼의 모든 로우를 일일이 숫자로 캐스팅(CAST)하여 비교해야 한다고 판단한다. B-Tree 인덱스는 '가공되지 않은 원본 데이터'를 기준으로 정렬되어 있다. 컬럼 데이터에 변형이 일어나는 순간, 정렬된 트리를 타고 내려가는 이진 탐색 알고리즘은 불가능해지며(Sargability 상실), 옵티마이저는 어쩔 수 없이 500만 건의 리프 노드를 모두 훑는 Full Table Scan을 선택한다.

&nbsp;

우리의 장애 사례는 반대(컬럼이 숫자, 파라미터가 문자)였으나, 최신 MySQL 버전의 엄격한 타입 세이프티 정책 하에 파라미터 문자열의 숫자 변환 가능성을 100% 확신할 수 없다고 판단한 옵티마이저가 인덱스 스캔을 포기해 버린 것이다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 사이드 이펙트: Buffer Pool 오염과 도미노 장애

&nbsp;

이 쿼리 하나가 단순히 '오래 걸리는' 것으로 끝나지 않고 전체 DB를 마비시킨 이유는 InnoDB의 메모리 캐싱 계층인 **Buffer Pool**을 완전히 오염시켰기 때문이다.

&nbsp;

## 3-1. LRU (Least Recently Used) 알고리즘의 붕괴
InnoDB는 디스크 I/O를 최소화하기 위해 자주 조회되는 데이터 페이지(Page)를 메모리(Buffer Pool)에 캐싱한다.
정상적인 Index Scan이라면 필요한 데이터 페이지 단 1~2개만 디스크에서 읽어와 Buffer Pool에 올린다. 하지만 500만 건을 Full Table Scan하게 되면, 엔진은 테이블의 전체 데이터 블록(수 GB 규모)을 디스크에서 읽어 들여 Buffer Pool에 밀어 넣는다.

&nbsp;

이 과정에서 원래 Buffer Pool에 얌전하게 캐싱되어 있던 다른 정상적인 API들의 Hot Data들이 쫓겨나게(LRU Eviction) 된다. 결국 이 멍청한 쿼리 하나 때문에 잘 돌고 있던 수십 개의 다른 쿼리들까지 캐시 히트(Cache Hit)에 실패하고 디스크를 읽게 되며, 시스템 전체의 I/O Wait가 폭증하는 '도미노 장애'로 발전한 것이다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. ORM 레벨에서의 은밀한 지뢰밭

&nbsp;

현대의 애플리케이션 개발 환경에서는 개발자가 쌩 쿼리(Raw Query)를 직접 작성하지 않기 때문에, 이런 타입 불일치 버그는 훨씬 은밀하게 발생한다.

&nbsp;

## 4-1. Node.js 생태계 (TypeORM / Prisma)
자바스크립트의 유연한 타입 시스템은 DB 연동 시 재앙이 된다.
API 컨트롤러가 쿼리 스트링(Query String)으로 넘어온 파라미터를 그대로 ORM에 넘기는 패턴이 가장 흔한 원인이다.
```typescript
// GET /api/orders?orderNo=2024042100010522
// req.query.orderNo는 무조건 String 타입이다.
const order = await orderRepository.findOne({
    where: { order_no: req.query.orderNo } // String이 그대로 바인딩됨
});
```
ORM 내부 쿼리 빌더는 `req.query.orderNo`가 문자열이므로 친절하게 `WHERE order_no = '2024...'`로 따옴표를 붙여서 쿼리를 전송한다.

&nbsp;

## 4-2. Java Spring 생태계 (Hibernate)
Java는 강타입(Strong Type) 언어라 비교적 안전하지만, JPA Native Query를 사용할 때 실수가 잦다.
```java
// DTO의 필드 타입을 String으로 잘못 선언한 경우
@Query(value = "SELECT * FROM orders WHERE order_no = :orderNo", nativeQuery = true)
Order findByOrderNoNative(@Param("orderNo") String orderNo);
```
Entity 필드가 `Long`이더라도, Native Query에서 파라미터를 `String`으로 받아서 넘기면 JDBC 드라이버는 이를 문자열 리터럴로 매핑하여 전송한다.

&nbsp;

&nbsp;

---

&nbsp;

# 5. 재발 방지를 위한 방어적 아키텍처

&nbsp;

장애 복구 후, 우리 팀은 이 어처구니없는 실수가 다시는 발생하지 않도록 세 겹의 방어망을 구축했다.

&nbsp;

### 1) DTO 기반의 엄격한 파이프라인 (Strict Validation)
API 진입점에서 `class-validator` (Node.js)나 `@Valid` (Spring)를 사용하여 외부로부터 들어오는 모든 파라미터의 타입을 강제 캐스팅한다. 비즈니스 로직 레이어에는 무조건 DB 스키마와 완벽하게 일치하는 타입(Integer/Long)만이 도달할 수 있도록 설계한다.

&nbsp;

### 2) 슬로우 쿼리 모니터링 고도화: `Rows_examined` 추적
단순히 쿼리 실행 시간(Duration)만으로 알람을 거는 것은 사후약방문이다. 성능 저하의 전조 증상인 `Rows_examined`(검사한 행 수)와 `Rows_sent`(반환된 행 수)의 비율을 추적해야 한다.
조회 결과는 1건인데 검사한 행 수가 10만 건이 넘어간다면, 100% 인덱스를 타지 않은 풀 스캔이다. Datadog과 연동하여 이 비율이 무너지는 쿼리가 포착되면 즉시 슬랙 알람을 발송하도록 구성했다.

&nbsp;

### 3) CI/CD 단계의 Query Plan 검증 자동화
가장 핵심적인 비즈니스 로직을 담은 쿼리들은 CI 테스트 단계에서 실제 테스트 DB를 향해 `EXPLAIN`을 수행하도록 테스트 코드를 작성했다. 실행 계획의 `type` 필드가 `ALL`이거나 `index`(Full Index Scan)로 나올 경우 빌드 파이프라인을 강제로 실패(Fail)시켜 배포를 차단한다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 데이터베이스는 입력된 대로만 답할 뿐이다

&nbsp;

"인덱스가 안 타요"라고 말하기 전에, 내가 데이터베이스에게 던진 파라미터의 형상이 스키마의 정의와 단 1비트라도 어긋나지 않았는지 스스로를 의심해야 한다. RDBMS의 옵티마이저는 생각보다 똑똑하지만, 반대로 타입의 엄격성 앞에서는 한 치의 융통성도 발휘하지 않는다. 

&nbsp;

고작 따옴표 두 개(`''`)가 수십 대의 서버를 마비시킬 수 있는 것이 백엔드 엔지니어링의 무서움이자 매력이다. 모든 버그의 단서는 항상 소스 코드의 가장 기초적인 부분에 숨어 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 삽질 2편] "잠깐만요, 데드락(Deadlock)입니다" — 트랜잭션 순서 꼬임과 InnoDB 락 메커니즘 해부**

&nbsp;

단일 로우(Row)를 업데이트하는 단순한 API인데 왜 트래픽이 몰리면 데드락(Deadlock)이 터질까? InnoDB가 보조 인덱스(Secondary Index)에 접근할 때 은밀하게 거는 갭 락(Gap Lock)과 넥스트 키 락(Next-Key Lock)의 실체를 파헤치고, `SHOW ENGINE INNODB STATUS` 로그를 해독하여 무한 대기 상태에 빠진 스레드들을 구출하는 락 스케줄링 설계 최적화 전략을 심도 있게 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

인덱스성능, 옵티마이저트레이스, 타입불일치, 풀테이블스캔, MySQL장애, 슬로우쿼리분석, 버퍼풀오염, 백엔드성능최적화
