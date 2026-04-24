# [실전 삽질 2편] "잠깐만요, 데드락(Deadlock)입니다" — 트랜잭션 순서 꼬임과 InnoDB 락 메커니즘 해부

&nbsp;

대규모 이벤트 오픈 첫날, 결제 완료 후 유저의 포인트를 차감하고 파트너사의 정산금을 올려주는 단순한 이체(Transfer) 로직에서 대량의 에러가 쏟아지기 시작했다. 알람 채널을 붉게 물들인 범인은 `Deadlock found when trying to get lock; try restarting transaction` 이었다.

&nbsp;

개발 환경에서는 수십 번의 QA를 거쳐도 단 한 번도 재현되지 않았던 버그였다. 하지만 동시 접속자가 수천 명 단위로 몰리면서 숨어 있던 동시성(Concurrency) 결함이 이빨을 드러낸 것이다. 흔히 데드락이라고 하면 여러 개의 테이블을 복잡하게 엮어서 업데이트할 때 발생한다고 생각하지만, 우리의 로직은 단 하나의 `points` 테이블에서 두 개의 로우(Row)를 순차적으로 수정하는 것이 전부였다. 

&nbsp;

도대체 단일 테이블, 단일 트랜잭션에서 왜 데드락이 발생한 것일까? 이 미스터리를 풀기 위해 MySQL InnoDB 엔진의 깊숙한 곳, 레코드 락(Record Lock)과 갭 락(Gap Lock)의 세계로 들어가 본다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 장애 현상: 순환 대기(Circular Wait)의 수렁

&nbsp;

문제가 발생한 코드의 구조는 아주 고전적이고 직관적인 계좌 이체 패턴이었다.

&nbsp;

```java
@Transactional
public void transferPoints(Long senderId, Long receiverId, BigDecimal amount) {
    // 1. 송금자 포인트 차감
    pointRepository.decreasePoint(senderId, amount);
    
    // 2. 수신자 포인트 증가
    pointRepository.increasePoint(receiverId, amount);
}
```

&nbsp;

단일 스레드 환경에서는 완벽하게 동작한다. 하지만 수십 개의 스레드가 동시에 이 로직을 실행할 때, 교차 전송이 발생하면 상황이 달라진다.

&nbsp;

- **Thread A (트랜잭션 1)**: User 1에서 User 2로 송금 요청.
  - `decreasePoint(1)` 실행 → **User 1 레코드에 배타적 락(X-Lock) 획득.**
- **Thread B (트랜잭션 2)**: User 2에서 User 1로 송금 요청 (이벤트 리워드 반환 등 교차 상황).
  - `decreasePoint(2)` 실행 → **User 2 레코드에 배타적 락(X-Lock) 획득.**
- **Thread A 진행**: `increasePoint(2)` 실행 시도 → 이미 Thread B가 User 2의 락을 쥐고 있으므로 **대기 (Wait)**.
- **Thread B 진행**: `increasePoint(1)` 실행 시도 → 이미 Thread A가 User 1의 락을 쥐고 있으므로 **대기 (Wait)**.

&nbsp;

양쪽 스레드가 서로가 가진 락을 내놓기를 영원히 기다리는 '순환 대기(Circular Wait)' 상태에 빠진다. 이때 똑똑한 InnoDB 엔진의 데드락 감지기(Deadlock Detector)가 개입하여, 트랜잭션 규모가 더 작은 쪽을 강제로 롤백(Rollback)시키고 락을 풀어버리면서 데드락 에러를 발생시킨 것이다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 로그 파헤치기: `SHOW ENGINE INNODB STATUS`

&nbsp;

정말 순환 대기가 맞는지 엔진의 로그를 통해 직접 확인해야 한다. MySQL에서 `SHOW ENGINE INNODB STATUS` 쿼리를 실행하여 LATEST DETECTED DEADLOCK 섹션을 추출했다.

&nbsp;

```text
------------------------
LATEST DETECTED DEADLOCK
------------------------
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 0 sec fetching rows
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
UPDATE points SET amount = amount - 100 WHERE user_id = 1;

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 50 page no 4 n bits 72 index idx_user_id of table `test`.`points` 
trx id 12345 lock_mode X locks rec but not gap waiting

*** (2) TRANSACTION:
TRANSACTION 12346, ACTIVE 0 sec fetching rows
UPDATE points SET amount = amount + 100 WHERE user_id = 1;

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 50 page no 4 n bits 72 index idx_user_id of table `test`.`points` 
trx id 12346 lock_mode X locks rec but not gap

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
...
```

&nbsp;

로그를 보면 명확하게 두 트랜잭션이 동일한 인덱스(`idx_user_id`)의 레코드 락(`lock_mode X locks rec but not gap`)을 두고 경합하고 있음을 알 수 있다. 이처럼 데드락 로그는 락의 종류(레코드, 갭, 넥스트 키)와 대기 중인 쿼리를 정확히 지목해 주므로, 디버깅의 가장 중요한 단서가 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 근본 원인 해결: 자원 획득 순서의 정렬 (Resource Ordering)

&nbsp;

운영체제 공학에서 데드락을 방지하는 가장 고전적이면서도 확실한 방법론은 "모든 트랜잭션이 동일한 순서로 자원에 접근하게 만드는 것"이다. 
위 사례의 비극은 Thread A는 1번 → 2번 순서로 락을 잡으려 했고, Thread B는 2번 → 1번 순서로 락을 잡으려 했다는 방향의 엇갈림에 있다.

&nbsp;

우리는 비즈니스 로직에 **'ID 오름차순 정렬'**이라는 아주 간단한 알고리즘을 도입했다.

&nbsp;

```java
@Transactional
public void transferPoints(Long senderId, Long receiverId, BigDecimal amount) {
    // 1. 데드락 방지를 위해 항상 ID가 작은 유저부터 락을 획득하도록 강제 정렬
    Long firstId = senderId < receiverId ? senderId : receiverId;
    Long secondId = senderId < receiverId ? receiverId : senderId;

    // 2. 정렬된 순서대로 배타적 락(Pessimistic Lock) 선점
    Point firstPoint = pointRepository.findByIdForUpdate(firstId);
    Point secondPoint = pointRepository.findByIdForUpdate(secondId);

    // 3. 비즈니스 로직 (금액 검증 및 차감/증가)
    if (firstId.equals(senderId)) {
        firstPoint.decrease(amount);
        secondPoint.increase(amount);
    } else {
        secondPoint.decrease(amount);
        firstPoint.increase(amount);
    }
}
```

&nbsp;

이 로직을 배포한 이후의 시나리오를 다시 시뮬레이션 해보자.
- **Thread A (User 1 → User 2 송금)**: `firstId = 1`, `secondId = 2`. User 1의 락을 먼저 획득한다.
- **Thread B (User 2 → User 1 송금)**: `firstId = 1`, `secondId = 2`. User 1의 락을 획득하려 시도한다.
- **결과**: Thread B는 애초에 User 2의 락을 잡기도 전에 User 1의 락을 대기하게 된다. Thread A는 방해 없이 User 2의 락까지 확보하고 처리를 완료한 뒤 락을 모두 반납한다. 순환 대기 고리가 원천적으로 끊어지는 마법이 일어난다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 보이지 않는 덫: 보조 인덱스와 Gap Lock

&nbsp;

자원 정렬만으로 대부분의 데드락을 막을 수 있지만, InnoDB 환경에서 조건절에 PK가 아닌 보조 인덱스(Secondary Index)를 사용할 경우 예상치 못한 변수가 발생한다.

&nbsp;

```sql
-- status가 인덱스인 경우
UPDATE points SET amount = 0 WHERE status = 'DEACTIVATED';
```

&nbsp;

만약 트랜잭션 내에서 위와 같이 범위 조건이나 유니크하지 않은 보조 인덱스로 `UPDATE`를 수행하면, InnoDB는 해당 데이터뿐만 아니라 그 데이터 앞뒤의 '존재하지 않는 빈 공간(Gap)'에까지 락을 걸어버린다. 이를 **갭 락(Gap Lock)** 혹은 넥스트 키 락(Next-Key Lock)이라 부른다. 

&nbsp;

이는 다른 트랜잭션이 해당 범위 사이에 새로운 데이터를 `INSERT`하는 팬텀 리드(Phantom Read)를 막기 위한 조치지만, 범위가 넓게 잡힐 경우 전혀 무관해 보이는 데이터 삽입 시도들끼리 데드락을 유발하는 끔찍한 연쇄 반응을 일으킨다.
따라서 가급적 `UPDATE`나 `DELETE` 쿼리는 사전에 `SELECT id ...`로 PK 값을 획득한 뒤, **오직 PK만을 조건으로 사용하여 단일 레코드 락(Record Lock)으로 범위를 극한으로 좁히는 아키텍처**를 구성해야 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 5. 결론: 동시성은 운에 맡기는 것이 아니다

&nbsp;

로컬에서 혼자 포스트맨(Postman) 버튼을 누를 때는 결코 만날 수 없는 에러가 데드락이다. "설마 1초도 안 되는 찰나에 두 유저가 서로 교차 송금을 하겠어?"라는 개발자의 안일함이 서비스 장애를 부른다.

&nbsp;

데이터베이스는 우리가 짠 코드의 동시성(Concurrency) 결함을 가장 잔인한 방식으로 피드백한다. 
- 복수의 리소스를 점유할 때는 반드시 '자원 정렬(Resource Ordering)' 원칙을 준수하라.
- 업데이트는 가급적 유니크한 프라이머리 키(PK)를 기준으로 수행하여 락의 범위를 최소화하라.
- `innodb_print_all_deadlocks` 설정을 켜고 에러 로그를 분석하는 능력을 길러라.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 삽질 3편] Connection Pool이 왜 마를까? — HikariCP 설정과 누수(Leak) 추적기**

&nbsp;

데이터베이스 성능은 쌩쌩하고 CPU도 10% 미만인데, 앱 서버에서 계속해서 `Connection is not available` 타임아웃 에러를 뿜어내며 전체 서비스가 먹통이 되는 기이한 현상. 범인은 코드 어딘가에 숨어 있는 커넥션 누수(Leak)였다. 스프링 부트 환경에서 HikariCP의 `leakDetectionThreshold`를 활용해 원인을 추적하고, 네트워크 I/O가 트랜잭션을 물고 늘어질 때 벌어지는 최악의 참사를 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

데드락, Deadlock, InnoDB락, 트랜잭션정렬, MySQL장애분석, 갭락, 백엔드성능, 동시성제어
