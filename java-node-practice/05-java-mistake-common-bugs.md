# 실무에서 자주 터지는 Spring 버그 모음 — 면접에도 나오는 것들

&nbsp;

Spring으로 개발하면 한 번씩은 꼭 만나는 버그들이 있다.

&nbsp;

에러 메시지를 보고 "아 이거..." 하는 순간이 반드시 온다.

시니어도 가끔 실수하고, 면접에서도 자주 나오는 주제들만 모았다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 순환 참조 (Circular Dependency)

&nbsp;

## 증상

&nbsp;

```
애플리케이션 시작 시:

***************************
APPLICATION FAILED TO START
***************************

The dependencies of some of the beans in the application context
form a cycle:

┌─────┐
|  memberService
↑     ↓
|  orderService
└─────┘
```

&nbsp;

## 원인

&nbsp;

```java
@Service
@RequiredArgsConstructor
public class MemberService {
    private final OrderService orderService;  // MemberService → OrderService

    public void deleteMember(Long id) {
        orderService.cancelAllOrders(id);
        // ...
    }
}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final MemberService memberService;  // OrderService → MemberService

    public void createOrder(OrderDto dto) {
        memberService.findById(dto.getMemberId());
        // ...
    }
}
```

&nbsp;

```
MemberService 생성하려면 → OrderService 필요
OrderService 생성하려면 → MemberService 필요
→ 무한 루프 → 앱 시작 실패
```

&nbsp;

## 해결

&nbsp;

```java
// 해결법 1: 설계 개선 — 공통 로직을 별도 서비스로 분리 (권장)
@Service
@RequiredArgsConstructor
public class MemberService {
    private final MemberRepository memberRepository;
    // OrderService 의존 제거

    public Member findById(Long id) {
        return memberRepository.findById(id).orElseThrow();
    }
}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final MemberService memberService;  // 단방향

    public void createOrder(OrderDto dto) {
        Member member = memberService.findById(dto.getMemberId());
        // ...
    }
}

// 회원 삭제 + 주문 취소는 별도 서비스에서
@Service
@RequiredArgsConstructor
public class MemberDeletionService {
    private final MemberService memberService;
    private final OrderService orderService;

    @Transactional
    public void deleteMember(Long id) {
        orderService.cancelAllOrders(id);
        memberService.delete(id);
    }
}
```

&nbsp;

```java
// 해결법 2: @Lazy (임시방편)
@Service
public class MemberService {
    private final OrderService orderService;

    public MemberService(@Lazy OrderService orderService) {
        this.orderService = orderService;
    }
}
// → 실제 사용 시점에 프록시로 주입
// → 순환은 해결되지만 설계 문제가 남아있음
```

&nbsp;

## 예방

&nbsp;

```
1. 의존 방향은 항상 단방향으로
2. A → B → A 구조가 보이면 설계를 다시 생각
3. 공통 로직은 별도 서비스로 분리
4. @Lazy는 급할 때만 — 근본적 해결은 설계 개선
```

&nbsp;

&nbsp;

---

&nbsp;

# 2. Bean 등록이 안 됨

&nbsp;

## 증상

&nbsp;

```
Parameter 0 of constructor in com.example.service.MemberService
required a bean of type 'com.example.repository.MemberRepository'
that could not be found.

Action:
Consider defining a bean of type 'com.example.repository.MemberRepository'
in your configuration.
```

&nbsp;

## 원인들

&nbsp;

```java
// 원인 1: @Component 빠뜨림
public class EmailService {  // ← @Service 없음!
    public void send(String to, String subject) { ... }
}

// 해결
@Service  // ← 추가
public class EmailService {
    public void send(String to, String subject) { ... }
}
```

&nbsp;

```java
// 원인 2: 패키지 스캔 범위 밖
// 메인 클래스가 com.example.app에 있는데
@SpringBootApplication  // com.example.app 하위만 스캔
public class Application { ... }

// Bean이 com.other.service에 있으면 → 스캔 안 됨
package com.other.service;

@Service
public class ExternalService { ... }  // 못 찾음!
```

&nbsp;

```java
// 해결: 스캔 범위 지정
@SpringBootApplication(scanBasePackages = {
    "com.example",
    "com.other"
})
public class Application { ... }
```

&nbsp;

```java
// 원인 3: @Configuration에서 반환 타입 잘못
@Configuration
public class AppConfig {

    @Bean
    public Object emailService() {  // ← 반환 타입이 Object!
        return new EmailService();
    }
    // EmailService 타입으로 주입 시 못 찾음
}

// 해결
@Configuration
public class AppConfig {

    @Bean
    public EmailService emailService() {  // ← 정확한 타입
        return new EmailService();
    }
}
```

&nbsp;

```java
// 원인 4: 인터페이스 vs 구현체 혼동
public interface PaymentGateway { ... }

@Service
public class StripeGateway implements PaymentGateway { ... }

// 주입할 때
@RequiredArgsConstructor
public class OrderService {
    private final StripeGateway gateway;  // ← 구현체로 주입하면
    // 프록시(AOP) 적용 시 못 찾을 수 있음
}

// 해결: 인터페이스로 주입
private final PaymentGateway gateway;  // ← 인터페이스로
```

&nbsp;

&nbsp;

---

&nbsp;

# 3. equals/hashCode 미구현

&nbsp;

## 증상

&nbsp;

```java
Set<Member> memberSet = new HashSet<>();

Member member1 = memberRepository.findById(1L).orElseThrow();
Member member2 = memberRepository.findById(1L).orElseThrow();

memberSet.add(member1);
memberSet.add(member2);

System.out.println(memberSet.size());
// 기대: 1 (같은 회원이니까)
// 실제: 2 (다른 객체로 판단!)
```

&nbsp;

## 원인

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;
    // equals/hashCode 미구현
    // → Object의 기본 equals 사용 = 참조(주소) 비교
    // → 같은 id라도 다른 객체면 다르다고 판단
}
```

&nbsp;

```
Object.equals() 기본 동작:
  member1 == member2  →  false (다른 객체)
  → Set에 2개 들어감

원하는 동작:
  member1.id == member2.id  →  true (같은 회원)
  → Set에 1개만 들어가야 함
```

&nbsp;

## 해결

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Member)) return false;  // instanceof 사용!
        Member member = (Member) o;
        return id != null && id.equals(member.getId());
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();  // 고정값!
    }
}
```

&nbsp;

## 왜 instanceof인가?

&nbsp;

```java
// getClass()로 비교하면:
if (o == null || getClass() != o.getClass()) return false;

// JPA 프록시 문제:
Member member = em.find(Member.class, 1L);
Member proxy = em.getReference(Member.class, 1L);

member.getClass();  // Member
proxy.getClass();   // Member$HibernateProxy$abc123

member.getClass() == proxy.getClass();  // false!
// → 같은 엔티티인데 다르다고 판단

// instanceof로 비교하면:
proxy instanceof Member;  // true!
// → 프록시도 Member의 하위 클래스이므로 true
```

&nbsp;

## 왜 hashCode()에 고정값을 쓰는가?

&nbsp;

```java
// id로 hashCode를 만들면:
@Override
public int hashCode() {
    return Objects.hash(id);
}

// 문제:
Member member = new Member();  // id = null → hashCode = X
memberSet.add(member);

em.persist(member);  // id = 1 → hashCode = Y
memberSet.contains(member);  // false! hashCode가 바뀌었으니까!

// 고정값이면:
@Override
public int hashCode() {
    return getClass().hashCode();  // 항상 같은 값
}
// → id가 바뀌어도 hashCode 동일 → Set/Map에서 안전
// → 단점: 같은 버킷에 몰리므로 성능 O(n). 하지만 실무에서 문제되는 경우 드묾.
```

&nbsp;

&nbsp;

---

&nbsp;

# 4. LocalDateTime 직렬화 문제

&nbsp;

## 증상

&nbsp;

```json
// 기대한 응답
{ "createdAt": "2024-04-20T10:30:00" }

// 실제 응답
{ "createdAt": [2024, 4, 20, 10, 30, 0] }
```

&nbsp;

또는

&nbsp;

```
com.fasterxml.jackson.databind.exc.InvalidDefinitionException:
Java 8 date/time type `java.time.LocalDateTime` not supported by default:
add Module "com.fasterxml.jackson.datatype:jackson-datatype-jsr310"
```

&nbsp;

## 원인

&nbsp;

Jackson은 기본적으로 Java 8 날짜 타입(LocalDateTime, LocalDate 등)을 모른다. 배열로 직렬화하거나 에러가 난다.

&nbsp;

## 해결

&nbsp;

```java
// 방법 1: 필드에 @JsonFormat (필드별 적용)
public class MemberResponse {

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createdAt;

    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate birthDate;
}
```

&nbsp;

```java
// 방법 2: 글로벌 설정 (전체 적용, 권장)
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        return mapper;
    }
}
```

&nbsp;

```yaml
# 방법 3: application.yml 설정
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false
    date-format: yyyy-MM-dd HH:mm:ss
```

&nbsp;

```json
// 설정 후 응답
{ "createdAt": "2024-04-20 10:30:00" }
```

&nbsp;

## 추가: 타임존 문제

&nbsp;

```java
// 서버 타임존과 DB 타임존이 다르면 시간이 어긋남
// application.yml에 타임존 명시
spring:
  jackson:
    time-zone: Asia/Seoul
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. 동시성 문제

&nbsp;

## 증상

&nbsp;

```
재고가 1개 남은 상품에 2명이 동시에 주문.
둘 다 성공함. 재고가 -1이 됨.
```

&nbsp;

## 원인

&nbsp;

```java
@Service
public class StockService {

    @Transactional
    public void decrease(Long productId) {
        Product product = productRepository.findById(productId).orElseThrow();

        if (product.getStock() <= 0) {
            throw new RuntimeException("재고 없음");
        }

        product.setStock(product.getStock() - 1);
    }
}
```

&nbsp;

```
Thread A                          Thread B
─────────────────────────────     ─────────────────────────────
SELECT stock FROM product         
→ stock = 1                       
                                  SELECT stock FROM product
                                  → stock = 1
if (stock <= 0) → false           
stock = 1 - 1 = 0                 if (stock <= 0) → false
UPDATE stock = 0                  stock = 1 - 1 = 0
                                  UPDATE stock = 0

결과: 재고 0. 근데 2건 주문 처리됨!
```

&nbsp;

## 해결법 1: 낙관적 락 (@Version)

&nbsp;

```java
@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;
    private int stock;

    @Version  // ← 버전 필드 추가
    private Long version;
}
```

&nbsp;

```sql
-- UPDATE 시 version 체크
UPDATE product
SET stock = 0, version = 2
WHERE id = 1 AND version = 1

-- Thread B가 같은 version으로 UPDATE 시도하면 0 rows affected
-- → OptimisticLockException 발생
```

&nbsp;

```java
// 재시도 로직 필요
@Service
public class StockService {

    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    @Transactional
    public void decrease(Long productId) {
        Product product = productRepository.findById(productId).orElseThrow();
        if (product.getStock() <= 0) {
            throw new RuntimeException("재고 없음");
        }
        product.setStock(product.getStock() - 1);
    }
}
```

&nbsp;

```
낙관적 락:
  ✅ 충돌이 적을 때 성능 좋음 (락 안 잡으니까)
  ❌ 충돌 많으면 재시도 비용
  → 사용 사례: 게시글 수정, 프로필 수정
```

&nbsp;

## 해결법 2: 비관적 락 (@Lock)

&nbsp;

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

&nbsp;

```sql
-- 실행되는 SQL
SELECT * FROM product WHERE id = 1 FOR UPDATE
-- → 다른 트랜잭션이 이 행을 읽거나 수정할 수 없음 (대기)
```

&nbsp;

```java
@Transactional
public void decrease(Long productId) {
    // 비관적 락으로 조회 → 다른 스레드는 대기
    Product product = productRepository.findByIdForUpdate(productId)
        .orElseThrow();

    if (product.getStock() <= 0) {
        throw new RuntimeException("재고 없음");
    }
    product.setStock(product.getStock() - 1);
    // 트랜잭션 종료 시 락 해제
}
```

&nbsp;

```
비관적 락:
  ✅ 충돌이 많아도 확실한 동기화
  ❌ 대기 시간 발생 (성능 저하)
  ❌ 데드락 가능성
  → 사용 사례: 재고 차감, 포인트 사용, 좌석 예약
```

&nbsp;

```
정리:
  읽기 많고 충돌 적음 → 낙관적 락 (@Version)
  쓰기 많고 충돌 많음 → 비관적 락 (@Lock)
  초고성능 필요       → Redis 분산 락 (별도 인프라)
```

&nbsp;

&nbsp;

---

&nbsp;

# 6. 스프링 프로필 실수

&nbsp;

## 증상

&nbsp;

```
application-dev.yml 만들었는데 적용이 안 됨.
로컬에서 dev 설정으로 돌리고 싶은데 prod 설정이 적용됨.
@Profile("dev") 붙인 Bean이 운영에서 안 뜸.
```

&nbsp;

## 원인 1: active profile 설정 안 함

&nbsp;

```yaml
# application.yml — 기본 설정
server:
  port: 8080

# application-dev.yml — dev 환경
server:
  port: 8081
  # 이 파일이 있어도 active profile 설정 안 하면 적용 안 됨!
```

&nbsp;

```yaml
# 해결: application.yml에 active profile 설정
spring:
  profiles:
    active: dev  # ← 이게 있어야 application-dev.yml이 로딩됨
```

&nbsp;

```bash
# 또는 실행 시 지정
java -jar app.jar --spring.profiles.active=dev

# 또는 환경변수
SPRING_PROFILES_ACTIVE=dev java -jar app.jar
```

&nbsp;

## 원인 2: @Profile Bean이 안 뜸

&nbsp;

```java
@Configuration
@Profile("dev")  // dev 프로필에서만 등록
public class DevConfig {

    @Bean
    public DataSource dataSource() {
        // H2 인메모리 DB
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

@Configuration
@Profile("prod")  // prod 프로필에서만 등록
public class ProdConfig {

    @Bean
    public DataSource dataSource() {
        // 실제 DB
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://prod-server:3306/mydb");
        return ds;
    }
}
```

&nbsp;

```
active profile이 없으면:
  → @Profile("dev")도 안 뜸
  → @Profile("prod")도 안 뜸
  → DataSource Bean 없음 → 앱 시작 실패

active = dev 이면:
  → DevConfig만 뜸 → H2 사용

active = prod 이면:
  → ProdConfig만 뜸 → MySQL 사용
```

&nbsp;

## 원인 3: yml 파일명 오타

&nbsp;

```
application-dev.yml    ← 올바름
application_dev.yml    ← 틀림! (언더스코어)
application-Dev.yml    ← 틀림! (대문자)
aplication-dev.yml     ← 틀림! (오타)
```

&nbsp;

## 실무 팁

&nbsp;

```yaml
# application.yml — 공통 설정 + 기본 프로필
spring:
  profiles:
    active: local  # 기본값: local

server:
  port: 8080

# application-local.yml — 로컬 개발
spring:
  datasource:
    url: jdbc:h2:mem:testdb

# application-dev.yml — 개발 서버
spring:
  datasource:
    url: jdbc:mysql://dev-server:3306/mydb

# application-prod.yml — 운영 서버
spring:
  datasource:
    url: jdbc:mysql://prod-server:3306/mydb
```

&nbsp;

```bash
# 운영 배포 시
java -jar app.jar --spring.profiles.active=prod

# 여러 프로필 동시 적용
java -jar app.jar --spring.profiles.active=prod,monitoring
```

&nbsp;

&nbsp;

---

&nbsp;

# 7. 한눈에 정리

&nbsp;

```
┌─────────────────────────┬────────────────────────────────────┐
│ 버그                    │ 핵심 해결                          │
├─────────────────────────┼────────────────────────────────────┤
│ 1. 순환 참조            │ 단방향 의존, 서비스 분리           │
│ 2. Bean 등록 안 됨      │ @Component 확인, 스캔 범위 확인    │
│ 3. equals/hashCode      │ instanceof + id 비교, 고정 hash   │
│ 4. LocalDateTime        │ JavaTimeModule + timestamps=false  │
│ 5. 동시성 문제          │ 낙관적/비관적 락                   │
│ 6. 프로필 실수          │ spring.profiles.active 설정        │
└─────────────────────────┴────────────────────────────────────┘
```

&nbsp;

&nbsp;

---

&nbsp;

# 8. 마무리

&nbsp;

이 버그들의 공통점이 있다.

&nbsp;

**"동작 원리를 모르면 에러 메시지만 보고 헤맨다."**

&nbsp;

```
순환 참조     → Spring IoC 컨테이너의 Bean 생성 순서
Bean 등록     → Component Scan 범위와 프록시 메커니즘
equals        → JPA 프록시와 Object의 기본 동작
직렬화        → Jackson의 타입 처리 방식
동시성        → DB 트랜잭션 격리 수준과 락
프로필        → Spring Boot 설정 파일 로딩 순서
```

&nbsp;

에러가 나면 "어떻게 고치지?"보다 **"왜 이렇게 되지?"**를 먼저 생각하자.

원리를 알면 비슷한 버그를 미리 예방할 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

이전 시리즈:
- 1편: N+1 문제 총정리
- 2편: @Transactional의 함정
- 3편: JPA 더티 체킹
- 4편: Entity를 API 응답으로 직접 반환하면 생기는 일

&nbsp;

&nbsp;

---

Spring, 순환참조, Circular Dependency, Bean, equals, hashCode, LocalDateTime, Jackson, 동시성, 낙관적락, 비관적락, Version, Lock, Profile, 면접, Spring Boot, JPA, 실무
