# controller vs api, service vs application — 패키지 이름이 설계 수준을 보여준다

&nbsp;

Spring Boot 프로젝트를 만들 때 패키지 구조를 잡는다.

&nbsp;

대부분 이렇게 시작한다:

```
src/
├── controller/
├── service/
├── repository/
├── entity/
└── dto/
```

&nbsp;

근데 어떤 프로젝트를 보면 이렇게 되어있다:

```
src/
├── api/
├── application/
├── domain/
└── infrastructure/
```

&nbsp;

같은 Spring Boot인데 이름이 다르다.

뭐가 다른 걸까? 그냥 이름 취향 차이일까?

&nbsp;

**아니다. 설계 관점이 다르다.**

&nbsp;

&nbsp;

---

&nbsp;

# 1. 일반적인 구조 — "기술 기준"

&nbsp;

```
src/
├── controller/     ← 스프링 컨트롤러
│   ├── MemberController.java
│   └── OrderController.java
├── service/        ← 비즈니스 로직
│   ├── MemberService.java
│   └── OrderService.java
├── repository/     ← DB 접근
│   ├── MemberRepository.java
│   └── OrderRepository.java
├── entity/         ← DB 테이블
│   ├── Member.java
│   └── Order.java
└── dto/            ← 요청/응답
    ├── MemberDto.java
    └── OrderDto.java
```

&nbsp;

**"이 클래스가 어떤 Spring 기술을 쓰는가?"** 기준으로 나눈 것이다.

- controller = `@RestController` 쓰는 클래스
- service = `@Service` 쓰는 클래스
- repository = `@Repository` 쓰는 클래스

&nbsp;

**장점:** 직관적이다. Spring 배운 사람이면 바로 이해한다.

**단점:** 기능이 많아지면 service에 모든 게 몰린다.

&nbsp;

```java
// service/OrderService.java — 점점 뚱뚱해진다
@Service
public class OrderService {
    public void createOrder() { ... }        // 주문 생성
    public void cancelOrder() { ... }        // 주문 취소
    public void calculateFee() { ... }       // 수수료 계산
    public void validateStock() { ... }      // 재고 검증
    public void sendNotification() { ... }   // 알림 전송
    public void processRefund() { ... }      // 환불 처리
    public void generateReceipt() { ... }    // 영수증 생성
    // ... 500줄
}
```

&nbsp;

"비즈니스 로직"이라는 이유로 **전부 service에 넣는다.** 이게 문제다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 엔터프라이즈 구조 — "역할 기준"

&nbsp;

```
src/
├── api/                ← 외부와 소통하는 접점
│   ├── MemberApi.java
│   ├── MemberRequest.java
│   └── MemberResponse.java
├── application/        ← 흐름 제어 (오케스트레이션)
│   ├── MemberFacade.java
│   └── OrderFacade.java
├── domain/             ← 순수 비즈니스 규칙
│   ├── Member.java
│   ├── Order.java
│   └── OrderValidator.java
└── infrastructure/     ← 외부 시스템 연결
    ├── MemberRepository.java
    ├── PaymentGateway.java
    └── NotificationSender.java
```

&nbsp;

**"이 코드의 역할(책임)이 무엇인가?"** 기준으로 나눈 것이다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 왜 controller 대신 api인가

&nbsp;

```
controller: "여기에 @RestController 클래스가 있다" (기술)
api:        "여기는 외부와 소통하는 접점이다" (역할)
```

&nbsp;

api 폴더에는 컨트롤러만 있는 게 아니다:

```
api/
├── MemberApi.java          ← 컨트롤러
├── MemberRequest.java      ← 이 API 전용 요청 DTO
├── MemberResponse.java     ← 이 API 전용 응답 DTO
└── MemberApiExceptionHandler.java  ← 이 API 전용 에러 처리
```

&nbsp;

**"외부에 노출되는 규격"을 한 곳에서 관리한다.**

controller 폴더에 넣으면 DTO는 dto 폴더에, 에러 핸들러는 exception 폴더에 흩어진다.

&nbsp;

```
controller 구조:
  "MemberController의 요청 DTO가 뭐지?"
  → controller/ 열고 → dto/ 열고 → 찾기
  = 2곳을 봐야 함

api 구조:
  "Member API 관련 전부 보자"
  → api/member/ 열면 전부 있음
  = 1곳만 보면 됨
```

&nbsp;

&nbsp;

---

&nbsp;

# 4. 왜 service 대신 application인가

&nbsp;

이게 핵심이다.

&nbsp;

```
service:     "비즈니스 로직 다 넣는 곳" (모호)
application: "흐름을 제어하는 곳" (명확)
```

&nbsp;

## service의 문제

&nbsp;

```java
// "비즈니스 로직"이라는 이유로 전부 service에
@Service
public class OrderService {
    public Order createOrder(OrderDto dto) {
        // 1. 회원 확인 (비즈니스 규칙? 흐름 제어?)
        Member member = memberRepository.findById(dto.getMemberId());
        
        // 2. 재고 확인 (비즈니스 규칙)
        if (product.getStock() < dto.getQuantity()) {
            throw new OutOfStockException();
        }
        
        // 3. 가격 계산 (비즈니스 규칙)
        int price = product.getPrice() * dto.getQuantity();
        if (member.isVip()) price = (int)(price * 0.9);
        
        // 4. 결제 (외부 API 호출)
        paymentGateway.pay(price);
        
        // 5. 주문 저장 (DB)
        Order order = orderRepository.save(new Order(...));
        
        // 6. 알림 전송 (외부 시스템)
        notificationSender.send(member, order);
        
        return order;
    }
}
```

&nbsp;

이 코드에는 **3가지 성격이 섞여있다:**

- 비즈니스 규칙: 재고 확인, VIP 할인
- 흐름 제어: 확인 → 계산 → 결제 → 저장 → 알림
- 외부 연동: 결제 API, 알림 전송

&nbsp;

## application + domain으로 나누면

&nbsp;

```java
// domain/Order.java — 순수 비즈니스 규칙만
@Entity
public class Order {
    public static Order create(Member member, Product product, int quantity) {
        if (product.getStock() < quantity) {
            throw new OutOfStockException();
        }
        int price = product.getPrice() * quantity;
        if (member.isVip()) price = (int)(price * 0.9);
        return new Order(member, product, quantity, price);
    }
}
```

&nbsp;

```java
// application/OrderFacade.java — 흐름 제어만
@Service
@RequiredArgsConstructor
public class OrderFacade {
    private final MemberRepository memberRepository;
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final NotificationSender notificationSender;

    @Transactional
    public Order createOrder(OrderCommand command) {
        // 1. 데이터 조회
        Member member = memberRepository.findById(command.getMemberId());
        Product product = productRepository.findById(command.getProductId());
        
        // 2. 도메인 로직 실행 (비즈니스 규칙은 도메인이 가짐)
        Order order = Order.create(member, product, command.getQuantity());
        
        // 3. 결제
        paymentGateway.pay(order.getPrice());
        
        // 4. 저장
        orderRepository.save(order);
        
        // 5. 알림
        notificationSender.send(member, order);
        
        return order;
    }
}
```

&nbsp;

```
application: "주문 흐름은 조회 → 생성 → 결제 → 저장 → 알림이다" (오케스트레이션)
domain:      "재고가 부족하면 주문 못 한다, VIP는 10% 할인" (비즈니스 규칙)
```

&nbsp;

**application은 "무엇을 어떤 순서로"**, domain은 **"규칙이 뭔지".**

&nbsp;

&nbsp;

---

&nbsp;

# 5. infrastructure는 뭔가

&nbsp;

```
repository:      "DB에 접근하는 곳" (기술)
infrastructure:  "외부 시스템과 연결하는 곳" (역할)
```

&nbsp;

```
infrastructure/
├── persistence/
│   ├── MemberRepository.java       ← DB
│   └── MemberJpaRepository.java    ← JPA 구현
├── external/
│   ├── PaymentGateway.java          ← 결제 API
│   └── SmsNotificationSender.java   ← 문자 발송
└── config/
    ├── JpaConfig.java
    └── RedisConfig.java
```

&nbsp;

repository는 DB만 들어가는데, infrastructure는 **DB + 외부 API + 캐시 + 파일 + 메시지 큐** 전부 포함한다.

"외부 세계와 연결되는 모든 것"이 infrastructure다.

&nbsp;

&nbsp;

---

&nbsp;

# 6. 비교

&nbsp;

| | 일반 구조 | 엔터프라이즈 구조 |
|:---|:---|:---|
| **이름** | controller, service, repository | api, application, domain, infrastructure |
| **기준** | "어떤 기술을 쓰는가?" | "이 코드의 역할이 무엇인가?" |
| **service** | 비즈니스 로직 전부 | application(흐름) + domain(규칙) 분리 |
| **controller** | 컨트롤러만 | api(컨트롤러 + DTO + 에러 핸들러) |
| **repository** | DB만 | infrastructure(DB + 외부 API + 캐시) |
| **적합** | 소규모, 학습 | 중대규모, 장기 유지보수 |

&nbsp;

&nbsp;

---

&nbsp;

# 7. 도메인 기반으로 더 나누면

&nbsp;

기능이 많아지면 **레이어 기준**이 아니라 **도메인 기준**으로 나눈다:

&nbsp;

```
레이어 기준 (기능 10개 넘으면 혼란):
src/
├── api/          ← 10개 컨트롤러가 한 폴더에
├── application/  ← 10개 서비스가 한 폴더에
├── domain/       ← 10개 엔티티가 한 폴더에
└── infrastructure/

도메인 기준 (기능별로 독립):
src/
├── member/
│   ├── api/
│   ├── application/
│   ├── domain/
│   └── infrastructure/
├── order/
│   ├── api/
│   ├── application/
│   ├── domain/
│   └── infrastructure/
└── payment/
    ├── api/
    ├── application/
    ├── domain/
    └── infrastructure/
```

&nbsp;

```
"주문 관련 코드 전부 보고 싶어"
  레이어 기준: api/, application/, domain/, infrastructure/ 4곳을 다 열어야 함
  도메인 기준: order/ 하나만 열면 됨
```

&nbsp;

&nbsp;

---

&nbsp;

# 8. 그래서 뭘 써야 하나

&nbsp;

```
프로젝트 시작 or 소규모 (API 10개 이하):
  → controller / service / repository
  → 빨리 시작하기 좋고, 누구나 이해함

프로젝트 성장 or 중규모 (API 30개 이상):
  → api / application / domain / infrastructure
  → 레이어 기준으로 나눔

대규모 or 팀이 여러 개 (API 100개 이상):
  → 도메인 기준 + 레이어 하위
  → member/, order/, payment/ 각각 독립
```

&nbsp;

**가장 흔한 실수:**

```
❌ 소규모인데 엔터프라이즈 구조 → 오버엔지니어링, 파일만 많아짐
❌ 대규모인데 controller/service → service가 2,000줄, 누가 뭔지 모름
```

&nbsp;

**규모에 맞게.** 처음엔 단순하게 시작하고, 커지면 리팩토링하는 게 맞다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론

&nbsp;

```
controller → api:          "기술"이 아니라 "역할"로 이름 짓기
service → application:     "전부 넣는 통"이 아니라 "흐름 제어"로 한정
repository → infrastructure: "DB"만이 아니라 "외부 세계 전부"
domain:                     비즈니스 규칙은 여기에
```

&nbsp;

패키지 이름은 취향이 아니다.

**"이 코드가 뭘 하는지"를 이름만 보고 알 수 있느냐의 차이다.**

&nbsp;

controller라는 이름은 "Spring 컨트롤러가 있다"고 알려주고,

api라는 이름은 "외부와 소통하는 접점이 있다"고 알려준다.

&nbsp;

같은 코드인데, **이름이 설계 의도를 말해준다.**

&nbsp;

&nbsp;

---

Spring, 패키지구조, 클린아키텍처, DDD, 도메인주도설계, controller, service, 레이어드아키텍처, 헥사고날, Java, 설계패턴, 프로젝트구조
