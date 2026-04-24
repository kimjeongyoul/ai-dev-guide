# [실전 삽질 3편] Connection Pool이 왜 마를까? — HikariCP 설정과 누수(Leak) 추적기

&nbsp;

새벽 2시, PagerDuty 알람이 미친 듯이 울리기 시작했다. 에러 로그 대시보드에는 `java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available` 메시지가 폭포수처럼 쏟아지고 있었다. 

&nbsp;

직감적으로 데이터베이스(DB) 인프라의 장애를 의심하고 RDS 대시보드를 열었지만, 황당하게도 DB의 CPU 사용률은 5%, Active Connection 수는 평소와 다를 바 없었다. DB는 너무나 평온하게 쉬고 있는데, 애플리케이션(Spring Boot) 서버들만 홀로 "커넥션을 달라"며 비명을 지르다 타임아웃(Timeout)으로 쓰러져가는 기괴한 상황이었다. 

&nbsp;

문제의 원인은 DB 자체의 부하가 아니라, 애플리케이션 코드 내부에서 커넥션을 빌려 간 뒤 돌려주지 않는 '커넥션 누수(Connection Leak)'와 잘못 튜닝된 커넥션 풀(Pool) 설정에 있었다. 본 글에서는 HikariCP의 내부 동작 원리를 파헤치고, 트랜잭션 스코프 설계 결함으로 인한 서비스 마비 상황을 어떻게 디버깅하고 해결했는지 상세한 엔지니어링 리포트를 공유한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 장애 메커니즘: 상태 전이 장애와 풀(Pool)의 고갈

&nbsp;

HikariCP를 비롯한 대부분의 데이터베이스 커넥션 풀은 크기가 고정되어 있다(예: `maximum-pool-size=20`). 서버에 들어온 요청(Thread)은 쿼리를 실행하기 위해 풀에서 커넥션을 하나 빌려(`borrow`) 가고, 처리가 끝나면 즉시 반납(`return`)해야 한다.

&nbsp;

장애 당시 로그를 보면, HikariPool이 커넥션을 반환받지 못하고 `connectionTimeout` (기본 30초) 동안 처절하게 대기하다가 결국 포기하는 패턴이 반복되었다.

&nbsp;

```text
java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30005ms.
    at com.zaxxer.hikari.pool.HikariPool.createTimeoutException(HikariPool.java:696)
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:197)
```

&nbsp;

이 에러가 무서운 이유는 서비스 전체를 마비시키는 '전염성' 때문이다. 특정 API 로직 하나가 커넥션을 반납하지 않으면, 톰캣(Tomcat)의 가용 워커 스레드들이 모두 HikariPool 앞에서 커넥션이 나기를 기다리며 멈춰 선다(Blocking). 결국 커넥션을 아예 쓰지 않는 단순한 캐시 조회 API조차 스레드 부족으로 인해 응답하지 못하게 되는 전면 장애(Cascading Failure)로 번지게 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 범인 검거: `leakDetectionThreshold`의 마법

&nbsp;

코드 수만 줄 속에서 어떤 메서드가 커넥션을 쥐고 안 놔주는지 눈으로 찾는 것은 불가능하다. 이때 빛을 발하는 것이 HikariCP의 누수 감지 기능이다. 이 옵션은 기본적으로 꺼져 있으나, 운영 환경에서 반드시 켜두어야 하는 필수 가드레일이다.

&nbsp;

```yaml
# application.yml
spring:
  datasource:
    hikari:
      # 커넥션을 빌려간 지 5초(5000ms)가 넘도록 반납하지 않으면 누수로 간주하고 스택 트레이스를 로깅함
      leak-detection-threshold: 5000
```

&nbsp;

이 옵션을 활성화하고 배포를 진행하자, 5초 뒤 범인의 정확한 위치가 콜 스택(Stack Trace)과 함께 로그에 선명하게 찍혔다.

&nbsp;

```text
WARN  com.zaxxer.hikari.pool.ProxyLeakTask - Connection leak detection triggered for connection org.postgresql.jdbc.PgConnection@1234abcd, stack trace follows
java.lang.Exception: Apparent connection leak detected
    at com.example.service.PaymentService.callExternalPaymentGateway(PaymentService.java:85)
    at com.example.service.OrderFacade.processOrder(OrderFacade.java:42)
    at ...
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. 기술적 분석: @Transactional과 Network I/O의 치명적 동거

&nbsp;

로그가 지목한 `processOrder` 메서드의 구조를 뜯어보니 전형적이고도 치명적인 안티 패턴이 자리 잡고 있었다.

&nbsp;

```java
@Transactional // 1. 여기서 DB 커넥션 획득을 시도하거나 점유함
public void processOrder(OrderRequest request) {
    // 2. 주문 정보 DB 저장 (순식간에 끝남)
    orderRepository.save(new Order(request));

    // 3. ❌ 치명적 오류: 외부 결제 PG사 API 호출 (네트워크 I/O 발생)
    paymentResponse = paymentService.callExternalPaymentGateway(request);

    // 4. 결제 결과에 따라 주문 상태 업데이트
    order.updateStatus(paymentResponse.getStatus());
    // 5. 트랜잭션 종료, 이 시점에 커넥션 반납
}
```

&nbsp;

스프링의 `@Transactional` 어노테이션은 진입 시점에 트랜잭션을 시작하며, 기본 동작 모드에서는 이때 이미 DB 커넥션을 풀에서 가져와 스레드에 바인딩한다.

&nbsp;

문제는 3번 단계의 외부 네트워크 호출이었다. 외부 PG사의 서버가 응답이 느려져서 타임아웃(Read Timeout)인 10초 동안 쓰레드가 대기한다고 가정해 보자. DB 작업은 이미 끝났지만 트랜잭션 스코프가 닫히지 않았으므로, 이 귀중한 DB 커넥션은 아무 일도 하지 않으면서 외부 API 응답만 기다린 채 10초 동안 허공에 묶여 있게 된다. 
이런 결제 요청이 동시에 20개만 들어와도, `maximum-pool-size=20`인 풀은 즉시 100% 점유율을 찍으며 마비되어 버린다. 외부 API의 성능 저하가 내부 DB 아키텍처를 파괴해 버린 전형적인 횡적 장애 전파 모델이다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 아키텍처 리팩토링: 트랜잭션 스코프 최소화

&nbsp;

이 문제를 해결하는 유일한 방법은 데이터베이스 커넥션 점유 시간과 트랜잭션의 유지 범위(Scope)를 극한으로 쪼개는 것이다. 외부 네트워크 I/O나 무거운 CPU 연산은 절대로 트랜잭션 바운더리 내부에 두어서는 안 된다.

&nbsp;

파사드(Facade) 패턴을 도입하여 로직을 재배치했다.

&nbsp;

```java
@Component
public class OrderFacade {
    
    // 외부에 노출되는 파사드 메서드는 @Transactional을 달지 않는다.
    public void processOrder(OrderRequest request) {
        // Step 1: 빠른 DB 준비 작업 (내부에서 트랜잭션 열고 닫음)
        Order savedOrder = orderCommandService.createInitialOrder(request);

        // Step 2: 트랜잭션(커넥션)이 없는 안전한 상태에서 외부 API 호출 대기
        PaymentResponse paymentResponse = paymentService.callExternalPaymentGateway(request);

        // Step 3: 외부 API 결과를 들고 다시 빠르게 DB 업데이트 (트랜잭션 열고 닫음)
        orderCommandService.completeOrder(savedOrder.getId(), paymentResponse);
    }
}
```

&nbsp;

이 구조 개편을 통해 각 스레드가 DB 커넥션을 점유하는 시간은 기존 평균 2~3초에서 **0.05초(50ms) 미만**으로 비약적으로 단축되었다. 똑같은 풀 사이즈로도 초당 수백 배 많은 동시 요청을 처리할 수 있는 탄탄한 체력이 확보된 것이다.

&nbsp;

&nbsp;

---

&nbsp;

# 5. 풀 튜닝: 많다고 좋은 것이 아니다

&nbsp;

종종 커넥션 타임아웃 에러가 나면 "커넥션 풀을 100개, 200개로 늘리면 되잖아?"라고 반문하는 개발자가 있다. 이는 최악의 선택이다.
관계형 데이터베이스 시스템은 디스크 기반의 거대한 C 프로그램이며, 각 커넥션은 무거운 OS 스레드 컨텍스트 스위칭 비용을 동반한다. 커넥션이 늘어날수록 DB CPU는 실제 쿼리 연산보다 스레드 전환 처리에 에너지를 다 써버리며 처리량(Throughput)이 수직 낙하한다.

&nbsp;

PostgreSQL 팀과 HikariCP 메인테이너들이 권장하는 수학적 공식은 다음과 같다.
`Connections = ((core_count * 2) + effective_spindle_count)`
하드 디스크(Spindle)가 아닌 현대의 SSD/NVMe 환경이라면 대략 `CPU 코어 수 * 2` 전후가 최적의 풀 사이즈다. 우리는 AWS RDS 인스턴스의 코어 수에 맞춰 풀 사이즈를 오히려 20개로 축소하고 고정(`minimum-idle`과 `maximum-pool-size`를 동일하게 맞춤)함으로써 컨텍스트 스위칭 오버헤드를 줄여 응답 지연을 최적화했다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 커넥션은 한정된 천연자원이다

&nbsp;

데이터베이스 커넥션은 애플리케이션 메모리처럼 가비지 컬렉터가 알아서 회수해 주는 자원이 아니다. 철저하게 설계된 스코프 안에서 빌려 쓰고 즉시 반납해야 하는 고비용의 한정판 자원이다.

&nbsp;

- 외부 연동 로직(HTTP Client, SQS Produce, S3 Upload)을 `@Transactional` 내부에 방치하지 마라.
- 풀 사이즈를 무식하게 늘리기 전에 `leak-detection-threshold`를 켜서 새는 곳을 먼저 막아라.
- 애플리케이션의 성능은 데이터베이스를 '얼마나 적게 귀찮게 하느냐'에 달려 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 삽질 4편] 분명 local에선 됐는데... — DB 정렬 기준(Collation)이 부른 대참사**

&nbsp;

맥북의 로컬 환경에서는 `admin`과 `Admin`이 중복 가입 에러를 뱉으며 정상 방어되었지만, 배포 서버인 Linux 환경의 RDS에서는 버젓이 두 개의 계정이 생성되는 공포의 현상. 대소문자 구분을 멋대로 무시하고, 테이블 조인(JOIN)을 시도할 때마다 인덱스를 무력화시켜 Full Scan을 유발하는 데이터베이스 Character Set과 Collation 설정의 함정을 깊이 파헤친다.

&nbsp;

&nbsp;

---

&nbsp;

HikariCP, 커넥션풀, 커넥션누수, 트랜잭션설계, 스프링부트최적화, DB장애, 백엔드성능, 아키텍처리팩토링
