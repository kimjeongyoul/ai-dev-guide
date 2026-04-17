# 대규모 데이터 처리 시리즈 (7) — 10억 건 아키텍처, Spark, 데이터 레이크, 그리고 현실

&nbsp;

시리즈의 마지막 편이다. 1편에서 배치와 실시간의 차이를 배웠고, Spring Batch, Kafka, DB 최적화, 스트리밍을 거쳐 여기까지 왔다.

이번 편에서는 **10억 건**이라는 규모에 도전한다. RDBMS 한 대로는 불가능한 세계. Spark, 데이터 레이크, OLAP 데이터베이스가 필요한 세계. 그리고 가장 중요한 질문 — **"우리에게 이게 정말 필요한가?"**

&nbsp;

---

&nbsp;

## 1. RDBMS의 한계

&nbsp;

10억 건이 들어있는 MySQL 테이블에서 벌어지는 일:

```sql
-- 10억 건 테이블
SELECT COUNT(*) FROM access_logs;
-- 결과: 1,073,741,824
-- 소요: 32초

SELECT DATE(created_at), COUNT(*)
FROM access_logs
WHERE created_at >= '2026-01-01'
GROUP BY DATE(created_at);
-- 소요: 4분 28초

ALTER TABLE access_logs ADD COLUMN region VARCHAR(10);
-- 소요: 2시간 15분 (테이블 잠금)
```

&nbsp;

**10억 건에서 RDBMS가 힘든 이유:**

| 문제 | 설명 |
|:---:|:---|
| 풀스캔 | 인덱스가 없는 집계 쿼리는 전체 스캔 |
| 인덱스 크기 | 인덱스 자체가 수십 GB → 메모리에 안 올라감 |
| DDL 잠금 | 스키마 변경 시 테이블 락 → 서비스 중단 |
| 백업/복구 | 덤프만 수 시간 |
| 단일 노드 | 수직 확장의 한계 |

&nbsp;

```
RDBMS의 적정 규모:

MySQL/PostgreSQL:     ~1억 건 (잘 튜닝하면)
읽기 분리 + 파티셔닝: ~5억 건 (어렵지만 가능)
그 이상:              다른 기술이 필요
```

&nbsp;

---

&nbsp;

## 2. 데이터 레이크

&nbsp;

데이터 레이크는 **모든 데이터를 원본 그대로 저장하는 저장소**다.

```
┌─────────────────────────────────────────────┐
│              데이터 레이크                    │
│              (S3 / HDFS)                    │
│                                             │
│  ┌──────────────┐  ┌──────────────┐         │
│  │  원본 데이터   │  │  가공 데이터   │         │
│  │  (Raw Zone)  │  │ (Processed)  │         │
│  │              │  │              │         │
│  │  CSV, JSON   │  │  Parquet     │         │
│  │  로그 파일    │  │  집계 결과    │         │
│  │  DB 덤프     │  │  ML 피처     │         │
│  └──────────────┘  └──────────────┘         │
│                                             │
│  데이터 웨어하우스와 달리:                     │
│  - 스키마 없이 저장 (Schema-on-Read)         │
│  - 정형/비정형 데이터 모두 수용                │
│  - 저비용 대용량 (S3: 1TB에 월 $23)          │
└─────────────────────────────────────────────┘
```

&nbsp;

**데이터 레이크 vs 데이터 웨어하우스:**

| 항목 | 데이터 레이크 | 데이터 웨어하우스 |
|:---:|:---:|:---:|
| 스키마 | 읽을 때 결정 | 쓸 때 결정 |
| 데이터 형태 | 정형 + 비정형 | 정형만 |
| 저장 비용 | 매우 저렴 | 비쌈 |
| 쿼리 속도 | 느림 (엔진 필요) | 빠름 (최적화됨) |
| 용도 | 원본 보관, ML, 탐색 | 비즈니스 분석, 리포트 |

&nbsp;

---

&nbsp;

## 3. Parquet이 왜 빠른가

&nbsp;

10억 건을 CSV로 저장하면 수백 GB다. Parquet으로 저장하면 수십 GB로 줄고, 쿼리도 빠르다.

&nbsp;

### 행 기반 vs 열 기반

```
행 기반 (CSV, MySQL):
┌────┬───────┬─────────┬────────┬────────┐
│ id │ name  │  date   │ amount │ status │
├────┼───────┼─────────┼────────┼────────┤
│  1 │ Kim   │ 2026-01 │  10000 │ PAID   │
│  2 │ Lee   │ 2026-01 │  20000 │ PAID   │
│  3 │ Park  │ 2026-02 │  15000 │ CANCEL │
└────┴───────┴─────────┴────────┴────────┘
→ 모든 컬럼을 읽어야 함

열 기반 (Parquet):
id:     [1, 2, 3, ...]
name:   [Kim, Lee, Park, ...]
date:   [2026-01, 2026-01, 2026-02, ...]
amount: [10000, 20000, 15000, ...]
status: [PAID, PAID, CANCEL, ...]
→ SUM(amount) 할 때 amount 컬럼만 읽으면 됨
→ 나머지 컬럼은 건드리지 않음
```

&nbsp;

### Parquet의 3가지 무기

```
1. 컬럼 기반 저장
   SELECT SUM(amount) → amount 열만 읽음 → I/O 80% 감소

2. 압축
   같은 열에는 비슷한 값이 모여있음 → 압축률 극대화
   status: [PAID, PAID, PAID, PAID, ...] → 매우 잘 압축됨
   CSV 100GB → Parquet 15GB (85% 감소)

3. Predicate Pushdown
   WHERE date = '2026-01' → 메타데이터에서 해당 행 그룹만 골라 읽음
   Row Group 1: date min=2026-01, max=2026-01 → 읽음 ✅
   Row Group 2: date min=2026-03, max=2026-04 → 건너뜀 ❌
```

&nbsp;

---

&nbsp;

## 4. Apache Spark

&nbsp;

Spark는 **분산 데이터 처리 엔진**이다. 10억 건을 수십~수백 대의 머신에 나눠서 동시에 처리한다.

```
┌─────────────────────────────────────────┐
│            Spark Cluster                │
│                                         │
│  ┌───────────┐                          │
│  │  Driver   │  작업 분배, 결과 취합     │
│  └─────┬─────┘                          │
│        │                                │
│  ┌─────┼─────┬─────────┬─────────┐     │
│  │     │     │         │         │     │
│  ▼     ▼     ▼         ▼         ▼     │
│ ┌───┐ ┌───┐ ┌───┐   ┌───┐   ┌───┐    │
│ │W1 │ │W2 │ │W3 │   │W4 │   │W5 │    │
│ │   │ │   │ │   │   │   │   │   │    │
│ │2억│ │2억│ │2억│   │2억│   │2억│    │
│ │건 │ │건 │ │건 │   │건 │   │건 │    │
│ └───┘ └───┘ └───┘   └───┘   └───┘    │
│  각 Worker가 자기 몫을 병렬 처리         │
└─────────────────────────────────────────┘
```

&nbsp;

### RDD → DataFrame → Dataset 진화

```
RDD (초기):
  - 저수준 API, 타입 안전
  - 최적화 없음 (개발자가 직접 튜닝)

DataFrame (현재 주류):
  - SQL과 유사한 API
  - Catalyst 옵티마이저가 자동 최적화
  - Python에서도 빠름

Dataset (Java/Scala):
  - DataFrame + 타입 안전성
  - 컴파일 타임 에러 체크
```

&nbsp;

### PySpark 코드 예시: 10억 건 집계

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, count, date_format

# Spark 세션 생성
spark = SparkSession.builder \
    .appName("BillionRowAggregation") \
    .config("spark.sql.shuffle.partitions", 200) \
    .config("spark.executor.memory", "8g") \
    .config("spark.executor.cores", 4) \
    .config("spark.executor.instances", 20) \
    .getOrCreate()

# 1. S3에서 Parquet 파일 읽기 (10억 건)
df = spark.read.parquet("s3a://data-lake/access_logs/year=2026/")

print(f"총 레코드 수: {df.count():,}")
# 총 레코드 수: 1,073,741,824
# 소요: 약 45초 (20대 클러스터)

# 2. 일별 접속 통계
daily_stats = df \
    .filter(col("status_code").between(200, 299)) \
    .groupBy(date_format("created_at", "yyyy-MM-dd").alias("date")) \
    .agg(
        count("*").alias("total_requests"),
        sum("response_time_ms").alias("total_response_time"),
        count("distinct_user_id").alias("unique_users")
    ) \
    .orderBy("date")

# 3. 결과 저장 (Parquet으로, 날짜별 파티셔닝)
daily_stats.write \
    .mode("overwrite") \
    .partitionBy("date") \
    .parquet("s3a://data-lake/processed/daily_access_stats/")

# 4. 또는 DB에 직접 저장
daily_stats.write \
    .format("jdbc") \
    .option("url", "jdbc:mysql://analytics-db:3306/reports") \
    .option("dbtable", "daily_access_stats") \
    .option("user", "spark_user") \
    .option("password", "****") \
    .mode("overwrite") \
    .save()
```

&nbsp;

### Spark SQL

```python
# SQL로도 동일한 처리 가능
df.createOrReplaceTempView("access_logs")

result = spark.sql("""
    SELECT
        DATE(created_at) AS log_date,
        COUNT(*) AS total_requests,
        COUNT(DISTINCT user_id) AS unique_users,
        AVG(response_time_ms) AS avg_response_time,
        PERCENTILE_APPROX(response_time_ms, 0.99) AS p99_response_time
    FROM access_logs
    WHERE status_code BETWEEN 200 AND 299
      AND created_at >= '2026-01-01'
    GROUP BY DATE(created_at)
    ORDER BY log_date
""")

result.show()
```

&nbsp;

### 파티션 전략: repartition vs coalesce

```python
# 현재 파티션 수 확인
print(f"파티션 수: {df.rdd.getNumPartitions()}")  # 200

# repartition: 파티션 수 변경 (셔플 발생 — 데이터 재분배)
df_repartitioned = df.repartition(100)
# → 데이터가 100개 파티션으로 균등 분배
# → 비용 큼 (네트워크 전송)

# coalesce: 파티션 수 줄이기 (셔플 없음)
df_coalesced = df.coalesce(50)
# → 인접한 파티션을 합침
# → 비용 적음 (로컬 병합)
# → 늘리는 것은 불가 (줄이기만 가능)

# 실무 팁:
# - 파일 저장 전 coalesce로 파일 수 줄이기
# - GROUP BY 키로 repartition하면 셔플 최소화
df.repartition("store_id") \
  .write \
  .partitionBy("store_id") \
  .parquet("s3a://output/")
```

&nbsp;

---

&nbsp;

## 5. Airflow: 워크플로우 오케스트레이션

&nbsp;

Spark 작업, DB 적재, 알림 전송을 순서대로 실행해야 한다. 이걸 관리하는 것이 **Apache Airflow**다.

```
                    ┌─────────────────────┐
                    │    Airflow Scheduler │
                    │    (DAG 실행 관리)   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │    DAG: daily_etl    │
                    │                     │
                    │  [extract_logs]     │
                    │       │             │
                    │  [transform_spark]  │
                    │       │             │
                    │  [load_to_db]       │
                    │       │             │
                    │  [send_report]      │
                    └─────────────────────┘
```

&nbsp;

### DAG 코드 예시

```python
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.providers.mysql.operators.mysql import MySqlOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2026, 1, 1),
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['data-team@example.com'],
}

with DAG(
    'daily_access_log_etl',
    default_args=default_args,
    schedule_interval='0 3 * * *',  # 매일 새벽 3시
    catchup=False,
    tags=['etl', 'access_log'],
) as dag:

    # 1. S3에서 원본 로그 추출 + Spark로 변환
    transform = SparkSubmitOperator(
        task_id='transform_with_spark',
        application='s3://scripts/daily_access_aggregation.py',
        conn_id='spark_default',
        conf={
            'spark.executor.memory': '8g',
            'spark.executor.instances': '20',
        },
        application_args=[
            '--date', '{{ ds }}',
            '--input', 's3://data-lake/raw/access_logs/{{ ds }}/',
            '--output', 's3://data-lake/processed/daily_stats/{{ ds }}/',
        ],
    )

    # 2. 결과를 분석 DB에 적재
    load_to_db = MySqlOperator(
        task_id='load_to_analytics_db',
        mysql_conn_id='analytics_db',
        sql="""
            LOAD DATA LOCAL INFILE '/tmp/daily_stats_{{ ds }}.csv'
            INTO TABLE daily_access_stats
            FIELDS TERMINATED BY ','
            LINES TERMINATED BY '\\n'
            IGNORE 1 ROWS;
        """,
    )

    # 3. 완료 알림
    def send_notification(**context):
        date = context['ds']
        # Slack, 이메일 등으로 알림
        print(f"{date} ETL 완료")

    notify = PythonOperator(
        task_id='send_notification',
        python_callable=send_notification,
    )

    # DAG 의존성
    transform >> load_to_db >> notify
```

&nbsp;

---

&nbsp;

## 6. 데이터 웨어하우스

&nbsp;

데이터 레이크에 저장된 데이터를 **빠르게 분석**하려면 데이터 웨어하우스가 필요하다.

&nbsp;

### OLTP vs OLAP

| 항목 | OLTP | OLAP |
|:---:|:---:|:---:|
| 목적 | 트랜잭션 처리 | 분석/리포트 |
| 쿼리 | INSERT, UPDATE (건별) | SELECT + 집계 (대량) |
| 데이터 모델 | 정규화 (3NF) | 비정규화 (Star Schema) |
| 응답 시간 | 밀리초 | 초~분 |
| 동시 사용자 | 수천~수만 | 수십~수백 |
| 대표 DB | MySQL, PostgreSQL | ClickHouse, BigQuery, Redshift |

&nbsp;

```
OLTP (서비스 DB):
"주문번호 12345의 상태를 PAID로 변경" → 0.5ms

OLAP (분석 DB):
"2026년 1분기, 지역별 매출 합계와 전년 대비 증감률" → 2초 (10억 건 스캔)
```

&nbsp;

### ClickHouse: 오픈소스 OLAP의 강자

```sql
-- ClickHouse 테이블 생성
CREATE TABLE access_logs (
    id UInt64,
    user_id UInt32,
    path String,
    status_code UInt16,
    response_time_ms UInt32,
    created_at DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);

-- 10억 건에서 월별 집계: 약 3초
SELECT
    toStartOfMonth(created_at) AS month,
    count() AS total_requests,
    uniq(user_id) AS unique_users,
    avg(response_time_ms) AS avg_response_time,
    quantile(0.99)(response_time_ms) AS p99_response_time
FROM access_logs
WHERE created_at >= '2026-01-01'
GROUP BY month
ORDER BY month;
```

&nbsp;

**ClickHouse가 빠른 이유:**
- 컬럼 기반 저장 (필요한 컬럼만 읽음)
- 벡터화 실행 (SIMD 명령어로 한 번에 여러 값 처리)
- 파티셔닝 + 인덱스로 불필요한 데이터 스킵
- 압축률 90%+ (디스크 I/O 최소화)

&nbsp;

### 클라우드 데이터 웨어하우스

| 서비스 | 특징 |
|:---:|:---|
| **BigQuery** (GCP) | 서버리스, 스캔한 데이터 양만큼 과금 |
| **Redshift** (AWS) | 클러스터 기반, 예측 가능한 비용 |
| **Snowflake** | 멀티 클라우드, 컴퓨팅/스토리지 분리 |

```sql
-- BigQuery: 10억 건 집계 (약 5초, 스캔 50GB → 약 $0.25)
SELECT
    DATE(created_at) AS log_date,
    COUNT(*) AS total_requests,
    COUNT(DISTINCT user_id) AS unique_users
FROM `project.dataset.access_logs`
WHERE created_at >= '2026-01-01'
GROUP BY log_date
ORDER BY log_date;
```

&nbsp;

---

&nbsp;

## 7. 전체 아키텍처

&nbsp;

10억 건을 다루는 현실적인 아키텍처:

```
┌───────────────────────────────────────────────────────────────┐
│                     데이터 수집                                │
│                                                               │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                     │
│  │서비스A│  │서비스B│  │서비스C│  │로그   │                     │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘                     │
│     │         │         │         │                           │
│     ▼         ▼         ▼         ▼                           │
│  ┌──────────────────────────────────┐                         │
│  │          Kafka Cluster           │  ← 모든 이벤트의 허브    │
│  └──────────┬───────────┬──────────┘                         │
│             │           │                                     │
│     ┌───────▼───┐ ┌─────▼─────┐                              │
│     │ 실시간     │ │ 배치       │                              │
│     │ 처리       │ │ 적재       │                              │
│     │            │ │            │                              │
│     │ Flink /   │ │ Spark      │                              │
│     │ Kafka     │ │ (야간)     │                              │
│     │ Streams   │ │            │                              │
│     └─────┬─────┘ └─────┬─────┘                              │
│           │             │                                     │
│     ┌─────▼─────┐ ┌─────▼──────────────┐                     │
│     │ 서비스 DB  │ │ 데이터 레이크       │                     │
│     │ (MySQL)   │ │ (S3 + Parquet)     │                     │
│     │ OLTP      │ │                    │                     │
│     └───────────┘ └─────┬──────────────┘                     │
│                         │                                     │
│                   ┌─────▼──────────────┐                     │
│                   │ 데이터 웨어하우스    │                     │
│                   │ (ClickHouse /      │                     │
│                   │  BigQuery)         │                     │
│                   │ OLAP               │                     │
│                   └─────┬──────────────┘                     │
│                         │                                     │
│                   ┌─────▼──────────────┐                     │
│                   │ 대시보드 / 리포트    │                     │
│                   │ (Grafana, Metabase)│                     │
│                   └────────────────────┘                     │
└───────────────────────────────────────────────────────────────┘
```

&nbsp;

---

&nbsp;

## 8. 비용: AWS 기준 월 비용 예시

&nbsp;

"이거 도입하면 얼마나 드는데?" 현실적인 비용을 알아야 한다.

&nbsp;

### 일일 100만 건 (월 3,000만 건)

```
MySQL RDS (db.r6g.large):           월 $200
Redis (cache.t3.medium):            월 $50
Application (t3.medium x 2):        월 $70
S3 (50GB):                         월 $1.15
──────────────────────────────────
합계: 약 월 $320 (약 42만원)

→ Spring Batch + MySQL이면 충분
→ Kafka, Spark 불필요
```

&nbsp;

### 일일 1억 건 (월 30억 건)

```
Kafka (MSK kafka.m5.large x 3):    월 $600
MySQL RDS (db.r6g.2xlarge):         월 $600
MySQL Read Replica x 2:             월 $1,200
Application (c5.xlarge x 8):        월 $400
S3 (2TB):                          월 $46
Redis (cache.r6g.large):           월 $200
──────────────────────────────────
합계: 약 월 $3,050 (약 400만원)

→ Kafka + Worker 패턴
→ 읽기/쓰기 분리
→ DB 파티셔닝
```

&nbsp;

### 일일 10억 건 (월 300억 건)

```
Kafka (MSK kafka.m5.2xlarge x 6):  월 $3,600
Spark EMR (m5.2xlarge x 20, 야간): 월 $2,400
S3 (20TB):                         월 $460
ClickHouse (3노드):                월 $1,800
MySQL RDS (Multi-AZ):              월 $1,200
Application (c5.2xlarge x 16):      월 $1,600
Airflow (MWAA):                    월 $500
모니터링 (Grafana, Prometheus):     월 $200
──────────────────────────────────
합계: 약 월 $11,760 (약 1,530만원)

→ 전체 아키텍처 필요
→ 데이터 엔지니어 2~3명 인건비 별도
→ 정말 필요한지 신중하게 판단
```

&nbsp;

---

&nbsp;

## 9. 현실적 조언

&nbsp;

### "우리에게 이게 필요한가?"

기술에 취하기 쉽다. Spark, Flink, ClickHouse... 들어보면 다 도입하고 싶다. 하지만 현실을 직시해야 한다.

&nbsp;

**도입하지 않아도 되는 신호:**

```
- 일일 데이터가 100만 건 미만
- 팀에 데이터 엔지니어가 없음
- "혹시 나중에 필요할 수도 있어서"
- 현재 쿼리가 5초 안에 끝남
- 비용을 감당할 수 없음
```

&nbsp;

**도입해야 하는 신호:**

```
- 야간 배치가 아침까지 안 끝남
- DB CPU가 상시 80% 이상
- 분석 쿼리 때문에 서비스 DB가 느려짐
- 데이터 보관 기간을 줄여야 할 만큼 디스크 부족
- 실시간 분석이 비즈니스에 직접적 영향
```

&nbsp;

### 단계별 도입 전략

```
[1단계] 지금 당장
├── EXPLAIN으로 느린 쿼리 찾기
├── 인덱스 최적화
├── N+1 해결
└── 비용: $0

[2단계] 데이터가 1,000만 건을 넘으면
├── 읽기/쓰기 분리 (Read Replica)
├── Spring Batch 정비
├── DB 파티셔닝
└── 비용: +$200/월

[3단계] 데이터가 1억 건을 넘으면
├── Kafka 도입
├── 이벤트 기반 아키텍처
├── OLAP DB (ClickHouse) 분리
└── 비용: +$1,500/월

[4단계] 데이터가 10억 건을 넘으면
├── Spark + 데이터 레이크
├── Airflow 워크플로우
├── 데이터 엔지니어 채용
└── 비용: +$10,000/월 + 인건비
```

&nbsp;

### 마지막으로

```
"가장 좋은 아키텍처는 팀이 운영할 수 있는 아키텍처다."

10억 건 아키텍처를 설계할 수 있는 것과
10억 건 아키텍처를 운영할 수 있는 것은 완전히 다르다.

기술 도입보다 중요한 것:
1. 측정하라 — 진짜 느린 지점이 어딘지
2. 단순하게 시작하라 — 복잡한 것은 나중에
3. 팀의 역량을 고려하라 — 아무도 모르는 기술은 장애의 시작
4. 비용을 계산하라 — 인프라비 + 인건비 + 학습 비용
```

&nbsp;

---

&nbsp;

## 시리즈 전체 정리

&nbsp;

| 편 | 주제 | 핵심 |
|:---:|:---|:---|
| 1편 | 배치 vs 실시간 | 언제 무엇을, Lambda/Kappa |
| 2편 | Spring Batch | 1,000만 건까지의 정석 |
| 3편 | Kafka 기초 | 메시지 큐와 이벤트 스트리밍 |
| 4편 | Kafka + Worker | 1억 건 병렬 처리 패턴 |
| 5편 | DB 최적화 | 인덱스, 파티셔닝, N+1 |
| 6편 | 스트리밍 | Kafka Streams, Flink, CDC |
| 7편 | 10억 건 아키텍처 | Spark, 데이터 레이크, 현실 |

&nbsp;

이 시리즈를 통해 전달하고 싶었던 메시지는 하나다:

> **규모에 맞는 기술을 선택하고, 단계적으로 성장하라.**

100만 건에서 시작해서 10억 건까지, 각 단계마다 적합한 도구가 있다. 모든 것을 한 번에 도입할 필요 없다. 현재의 병목을 해결하고, 다음 병목을 준비하는 것. 그것이 엔지니어링이다.

&nbsp;

---

`#10억건아키텍처` `#ApacheSpark` `#데이터레이크` `#Parquet` `#ClickHouse` `#Airflow` `#OLAP` `#데이터웨어하우스` `#대규모데이터` `#데이터엔지니어링`
