# @Transactional의 함정 — 붙였는데 왜 안 돼?

&nbsp;

@Transactional은 Spring에서 가장 많이 쓰는 어노테이션 중 하나다.

&nbsp;

"트랜잭션 필요하면 @Transactional 붙이면 되지."

&nbsp;

맞는데, **붙여도 안 되는 경우가 5가지**나 있다.

시니어도 가끔 실수하고, 면접에서도 자주 나온다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 은행으로 이해하기

&nbsp;

```
@Transactional = 은행 창구 번호표

번호표를 받으면:
  → 모든 업무가 한 묶음으로 처리됨
  → 중간에 실패하면 전부 취소 (롤백)
  → 끝나면 한 번에 반영 (커밋)

근데 번호표를 받아도 안 되는 경우가 있다:
  1. 창구 뒤에서 몰래 처리 (private)
  2. 같은 직원이 자기한테 번호표 넘김 (self-invocation)
  3. 에러를 삼켜버림 (try-catch)
  4. "이 에러는 취소 대상 아닙니다" (checked exception)
  5. "읽기 전용인데 수정하려고요?" (readOnly=true)
```

&nbsp;

&nbsp;

---

&nbsp;

# 2. 함정 1 — private 메서드에 붙이면 안 된다

&nbsp;

```java
@Service
public class MemberService {

    @Transactional  // ← 이거 안 먹힘!
    private void updateMember(Long id, String name) {
        Member member = memberRepository.findById(id).orElseThrow();
        member.setName(name);
        // 트랜잭션 없이 실행됨 → 더티 체킹 안 됨
    }
}
```

&nbsp;

## 왜 안 되는가?

&nbsp;

Spring의 @Transactional은 **프록시(Proxy)** 기반으로 동작한다.

&nbsp;

```
외부 호출 → [프록시] → [실제 객체]
             ↓
         트랜잭션 시작
         실제 메서드 호출
         트랜잭션 커밋/롤백
```

&nbsp;

프록시가 메서드를 가로채서 트랜잭션을 감싸는 구조다.

근데 **private 메서드는 프록시가 오버라이드할 수 없다.**

&nbsp;

```java
// Spring이 내부적으로 만드는 프록시 (개념 코드)
public class MemberService$$Proxy extends MemberService {

    @Override
    public void updateMember(Long id, String name) {  // public만 오버라이드 가능
        // 트랜잭션 시작
        super.updateMember(id, name);
        // 트랜잭션 커밋
    }

    // private은 오버라이드 불가 → 트랜잭션 적용 불가
}
```

&nbsp;

## 해결

&nbsp;

```java
@Transactional  // public으로 바꾸면 된다
public void updateMember(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.setName(name);
}
```

&nbsp;

**규칙: @Transactional은 반드시 public 메서드에만 붙인다.**

&nbsp;

&nbsp;

---

&nbsp;

# 3. 함정 2 — 같은 클래스 내부 호출 (self-invocation)

&nbsp;

이게 가장 많이 실수하는 경우다.

&nbsp;

```java
@Service
public class MemberService {

    public void register(MemberDto dto) {
        // 회원 저장
        saveMember(dto);

        // 환영 메일 발송
        sendWelcomeEmail(dto.getEmail());
    }

    @Transactional  // ← 이거 안 먹힘!
    public void saveMember(MemberDto dto) {
        Member member = new Member(dto.getName(), dto.getEmail());
        memberRepository.save(member);
    }
}
```

&nbsp;

## 왜 안 되는가?

&nbsp;

```
[외부] → register() 호출
  → 프록시를 거침? NO!
  → register()에 @Transactional 없으니 그냥 통과
  → register() 안에서 saveMember() 호출
  → 이건 this.saveMember() = 실제 객체의 메서드 직접 호출
  → 프록시를 안 거침 → @Transactional 무시됨
```

&nbsp;

```java
// 이렇게 동작하는 거다
public class MemberService$$Proxy extends MemberService {

    @Override
    public void register(MemberDto dto) {
        // register에는 @Transactional 없으니 그냥 호출
        super.register(dto);
        // super.register() 안에서 saveMember()를 this로 호출
        // → 프록시를 안 거침!
    }

    @Override
    public void saveMember(MemberDto dto) {
        // 트랜잭션 시작
        super.saveMember(dto);
        // 트랜잭션 커밋
        // → 이건 외부에서 직접 호출해야 작동
    }
}
```

&nbsp;

## 해결법 1 — 클래스 분리 (권장)

&nbsp;

```java
@Service
@RequiredArgsConstructor
public class MemberService {

    private final MemberWriter memberWriter;

    public void register(MemberDto dto) {
        memberWriter.saveMember(dto);  // 외부 Bean 호출 → 프록시 거침
        sendWelcomeEmail(dto.getEmail());
    }
}

@Service
public class MemberWriter {

    @Transactional  // ← 이제 작동함!
    public void saveMember(MemberDto dto) {
        Member member = new Member(dto.getName(), dto.getEmail());
        memberRepository.save(member);
    }
}
```

&nbsp;

## 해결법 2 — 자기 자신 주입

&nbsp;

```java
@Service
public class MemberService {

    @Lazy
    @Autowired
    private MemberService self;  // 프록시 객체를 주입받음

    public void register(MemberDto dto) {
        self.saveMember(dto);  // self = 프록시 → @Transactional 작동
        sendWelcomeEmail(dto.getEmail());
    }

    @Transactional
    public void saveMember(MemberDto dto) {
        Member member = new Member(dto.getName(), dto.getEmail());
        memberRepository.save(member);
    }
}
```

&nbsp;

**실무에서는 클래스 분리가 더 깔끔하다.**

&nbsp;

&nbsp;

---

&nbsp;

# 4. 함정 3 — try-catch로 예외를 삼키면 롤백 안 된다

&nbsp;

```java
@Transactional
public void transferMoney(Long fromId, Long toId, int amount) {
    try {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        from.withdraw(amount);
        to.deposit(amount);

        // 여기서 에러 발생!
        externalApiClient.notifyTransfer(fromId, toId, amount);

    } catch (Exception e) {
        log.error("이체 알림 실패", e);
        // 예외를 삼켜버림 → 롤백 안 됨!
        // from에서 돈은 빠졌는데, 알림만 실패한 줄 알고 커밋됨
    }
}
```

&nbsp;

## 왜 안 되는가?

&nbsp;

```
@Transactional의 롤백 조건:
  → 메서드에서 예외가 던져져야 함 (throw)
  → catch로 잡아버리면 "정상 종료"로 판단
  → 커밋됨
```

&nbsp;

```java
// Spring 내부 동작 (개념 코드)
try {
    트랜잭션_시작();
    실제_메서드_호출();  // 예외가 여기서 나와야 함
    트랜잭션_커밋();     // 예외 없으면 커밋
} catch (Exception e) {
    트랜잭션_롤백();     // 예외 나오면 롤백
}
// → catch 안에서 삼키면 예외가 밖으로 안 나옴 → 커밋됨
```

&nbsp;

## 해결

&nbsp;

```java
@Transactional
public void transferMoney(Long fromId, Long toId, int amount) {
    try {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        from.withdraw(amount);
        to.deposit(amount);

        externalApiClient.notifyTransfer(fromId, toId, amount);

    } catch (Exception e) {
        log.error("이체 실패", e);
        throw e;  // ← 다시 던져야 롤백됨!
    }
}
```

&nbsp;

또는 알림은 트랜잭션 밖에서 처리한다.

&nbsp;

```java
@Transactional
public void transferMoney(Long fromId, Long toId, int amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();
    from.withdraw(amount);
    to.deposit(amount);
    // 여기까지만 트랜잭션
}

// 알림은 별도로
public void transferAndNotify(Long fromId, Long toId, int amount) {
    transferMoney(fromId, toId, amount);
    try {
        externalApiClient.notifyTransfer(fromId, toId, amount);
    } catch (Exception e) {
        log.error("알림 실패 (이체는 완료됨)", e);
    }
}
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. 함정 4 — checked exception은 롤백 안 된다

&nbsp;

```java
@Transactional
public void createOrder(OrderDto dto) throws IOException {
    Order order = new Order(dto.getProductId(), dto.getQuantity());
    orderRepository.save(order);

    // IOException = checked exception
    fileService.saveReceipt(order);  // IOException 발생!
    // → 롤백 안 됨! order는 DB에 저장됨!
}
```

&nbsp;

## 왜 안 되는가?

&nbsp;

```
Spring @Transactional 기본 롤백 규칙:
  RuntimeException (unchecked)  → 롤백 O
  Error                          → 롤백 O
  Exception (checked)           → 롤백 X  ← !!!!

Java 예외 구조:
  Throwable
  ├── Error (롤백 O)
  └── Exception
      ├── RuntimeException (롤백 O)
      │   ├── NullPointerException
      │   ├── IllegalArgumentException
      │   └── ...
      └── checked exceptions (롤백 X)
          ├── IOException
          ├── SQLException
          └── ...
```

&nbsp;

왜 이런 설계인가? EJB 시절 관례를 따른 것이다. "checked exception은 비즈니스 예외이므로 복구 가능하다"는 가정인데, 실무에서는 대부분 틀린 가정이다.

&nbsp;

## 해결

&nbsp;

```java
// 방법 1: rollbackFor 명시
@Transactional(rollbackFor = Exception.class)  // 모든 예외에 롤백
public void createOrder(OrderDto dto) throws IOException {
    Order order = new Order(dto.getProductId(), dto.getQuantity());
    orderRepository.save(order);
    fileService.saveReceipt(order);
}

// 방법 2: checked를 unchecked로 감싸기
@Transactional
public void createOrder(OrderDto dto) {
    try {
        Order order = new Order(dto.getProductId(), dto.getQuantity());
        orderRepository.save(order);
        fileService.saveReceipt(order);
    } catch (IOException e) {
        throw new RuntimeException("영수증 저장 실패", e);  // unchecked로 변환
    }
}
```

&nbsp;

**실무 팁: `@Transactional(rollbackFor = Exception.class)`를 습관적으로 쓰자.**

&nbsp;

&nbsp;

---

&nbsp;

# 6. 함정 5 — readOnly=true인데 수정하기

&nbsp;

```java
@Transactional(readOnly = true)
public void updateMemberName(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.setName(name);  // ← 변경 감지 안 됨! UPDATE 안 나감!
}
```

&nbsp;

## 왜 안 되는가?

&nbsp;

```
readOnly = true 설정 시:
  1. Hibernate: 스냅샷을 안 만듦 → 더티 체킹 비활성화
  2. JDBC: setReadOnly(true) → DB 드라이버 최적화
  3. 변경해도 UPDATE 쿼리 안 나감
  4. 에러도 안 남 → 조용히 무시됨 ← 위험!
```

&nbsp;

```java
// 이런 식으로 착각하기 쉽다
@Service
@Transactional(readOnly = true)  // 클래스 레벨에 readOnly
public class MemberService {

    public Member getMember(Long id) {
        return memberRepository.findById(id).orElseThrow();  // OK
    }

    // readOnly인데 수정하려고 함 → UPDATE 안 나감!
    public void updateMember(Long id, String name) {
        Member member = memberRepository.findById(id).orElseThrow();
        member.setName(name);
        // 에러 없이 그냥 무시됨... name 안 바뀜
    }
}
```

&nbsp;

## 해결

&nbsp;

```java
@Service
@Transactional(readOnly = true)  // 기본: 읽기 전용
public class MemberService {

    public Member getMember(Long id) {
        return memberRepository.findById(id).orElseThrow();
    }

    @Transactional  // 수정 메서드는 readOnly 없이 오버라이드
    public void updateMember(Long id, String name) {
        Member member = memberRepository.findById(id).orElseThrow();
        member.setName(name);  // 이제 UPDATE 나감
    }
}
```

&nbsp;

**패턴: 클래스에 `@Transactional(readOnly = true)`, 수정 메서드에만 `@Transactional`.**

&nbsp;

&nbsp;

---

&nbsp;

# 7. 보너스 — 트랜잭션 전파 (Propagation)

&nbsp;

면접에서 자주 나오는 주제다.

&nbsp;

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(OrderDto dto) {
        orderRepository.save(new Order(dto));
        paymentService.processPayment(dto);  // 다른 서비스 호출
    }
}

@Service
public class PaymentService {

    @Transactional  // 전파 속성에 따라 동작이 달라진다
    public void processPayment(OrderDto dto) {
        paymentRepository.save(new Payment(dto));
    }
}
```

&nbsp;

## REQUIRED (기본값)

&nbsp;

```java
@Transactional(propagation = Propagation.REQUIRED)  // 기본값
public void processPayment(OrderDto dto) { ... }
```

&nbsp;

```
OrderService.createOrder() → 트랜잭션 A 시작
  └→ PaymentService.processPayment() → 트랜잭션 A에 참여 (같은 트랜잭션)

결과:
  - 어디서든 에러 → 전부 롤백 (Order + Payment)
  - 하나의 트랜잭션으로 묶임
```

&nbsp;

## REQUIRES_NEW

&nbsp;

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void processPayment(OrderDto dto) { ... }
```

&nbsp;

```
OrderService.createOrder() → 트랜잭션 A 시작
  └→ PaymentService.processPayment() → 트랜잭션 A 일시 중단, 트랜잭션 B 새로 시작

결과:
  - Payment에서 에러 → 트랜잭션 B만 롤백 (Payment만)
  - Order는 트랜잭션 A이므로 별개로 판단
  - 단, Payment 에러가 createOrder()까지 올라가면 A도 롤백됨
```

&nbsp;

## "부모가 롤백되면 자식도 롤백되나?"

&nbsp;

```
REQUIRED (같은 트랜잭션):
  부모 롤백 → 자식도 롤백 (O)
  자식 롤백 → 부모도 롤백 (O)
  → 한 몸이니까

REQUIRES_NEW (별도 트랜잭션):
  부모 롤백 → 자식은 이미 커밋됨 (X, 자식은 유지)
  자식 롤백 → 부모는 별개 (에러 전파 안 하면 부모 유지)
```

&nbsp;

## 실무에서 REQUIRES_NEW를 쓰는 경우

&nbsp;

```java
// 로그는 무조건 남겨야 한다 — 비즈니스 로직이 실패해도
@Service
public class OrderService {

    @Transactional
    public void createOrder(OrderDto dto) {
        try {
            orderRepository.save(new Order(dto));
            // 에러 발생!
        } catch (Exception e) {
            auditService.saveLog("주문 실패: " + e.getMessage());  // 로그는 남겨야 함
            throw e;
        }
    }
}

@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)  // 새 트랜잭션
    public void saveLog(String message) {
        auditRepository.save(new AuditLog(message));
        // 부모가 롤백돼도 이건 커밋됨
    }
}
```

&nbsp;

&nbsp;

---

&nbsp;

# 8. 정리

&nbsp;

```
@Transactional이 안 되는 5가지:

  1. private 메서드      → public으로
  2. 같은 클래스 내부 호출 → 클래스 분리
  3. try-catch 예외 삼킴  → throw 다시 던지기
  4. checked exception   → rollbackFor = Exception.class
  5. readOnly = true     → 수정 메서드에 @Transactional 별도 선언

전파 규칙:
  REQUIRED (기본):     같은 트랜잭션에 참여 → 한 몸
  REQUIRES_NEW:        새 트랜잭션 생성 → 별개
```

&nbsp;

## 실무 팁

&nbsp;

```
1. @Transactional은 Service 레이어에만 붙인다
   → Controller, Repository에는 X

2. 클래스에 @Transactional(readOnly = true), 수정 메서드에 @Transactional

3. rollbackFor = Exception.class 습관적으로

4. self-invocation 주의 — 같은 클래스 안에서 호출하면 안 먹힘

5. 트랜잭션 범위는 최소한으로 — 외부 API 호출은 트랜잭션 밖에서
```

&nbsp;

&nbsp;

---

&nbsp;

다음 편 예고: **JPA 더티 체킹 — save() 안 해도 되는데 왜? 그리고 언제 안 될까?**

@Transactional 안에서 엔티티 필드를 바꾸면 save() 없이도 UPDATE가 나간다. 초보가 가장 놀라는 JPA의 마법, 그 원리와 함정을 파헤친다.

&nbsp;

&nbsp;

---

Transactional, Spring, 트랜잭션, 프록시, self-invocation, 롤백, checked exception, propagation, REQUIRED, REQUIRES_NEW, readOnly, AOP, 면접
