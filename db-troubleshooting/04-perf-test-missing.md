# 테스트 코드 — 안 짜면 어떻게 되는지 겪어본 이야기

&nbsp;

이번 편은 DB 성능 이야기가 아니다.

&nbsp;

**"개발 생산성 성능"** 이야기다.

&nbsp;

테스트 코드 없이 2년을 개발했다. 처음엔 괜찮았다. 기능이 적으니까. 그런데 코드가 10만 줄을 넘어가면서부터 모든 게 느려졌다. 코드 수정이 느려지고, 배포가 느려지고, 신입 온보딩이 느려졌다.

&nbsp;

쿼리 튜닝으로는 안 풀리는 느림. 개발자 자체가 느려지는 문제.

&nbsp;

&nbsp;

---

&nbsp;

## 1. 테스트 없는 프로젝트의 일상

&nbsp;

### "이거 수정하면 저쪽 깨지지 않을까?"

&nbsp;

```
할인 로직을 수정해야 한다.

할인율 계산 → 주문 금액 계산 → 결제 → 정산 → 리포트

할인율 공식을 바꾸면 뒤쪽 4개가 전부 영향받는다.
어디까지 영향받는지 정확히 모른다.
코드를 따라가며 눈으로 확인? 10개 파일, 2,000줄.
무서워서 못 고친다.

결과: "일단 하드코딩으로 때우자" → 기술 부채 +1
```

&nbsp;

### "리팩토링 하고 싶은데..."

&nbsp;

```
UserService가 3,000줄이다.
메서드가 80개. 절반은 중복 로직.
쪼개고 싶다. 정리하고 싶다.

그런데 검증 수단이 없다.
쪼개고 나서 뭐가 깨졌는지 어떻게 알지?
수동으로 전부 테스트? 화면이 50개다.
일주일 걸린다.

결과: 안 한다. 3,000줄이 5,000줄이 된다.
```

&nbsp;

### "배포 후 CS 폭탄"

&nbsp;

```
금요일 오후 배포.
"별거 아닌 수정이에요" — 버튼 텍스트 변경.

월요일 아침 출근.
"결제가 안 돼요" CS 47건.

버튼 텍스트 변경하면서 옆에 있던 결제 버튼의 onClick 핸들러를 실수로 지웠다.
사람이 눈으로 테스트하니까 놓친 것.
```

&nbsp;

### "신입이 왔는데 코드 못 건드림"

&nbsp;

```
신입 개발자가 합류했다.
"이 함수 수정해주세요."

신입: "수정했는데 이거 다른 데 영향 없나요?"
시니어: "음... 아마 괜찮을 거예요."
신입: "아마요?"
시니어: "네, 아마..."

신입은 한 줄 고치는 데 반나절이 걸린다.
영향 범위를 파악하느라.
```

&nbsp;

&nbsp;

---

&nbsp;

## 2. 테스트 있는 프로젝트의 일상

&nbsp;

```
할인 로직 수정:
  코드 수정 → 테스트 실행 → 3개 실패
  → "아, 정산 쪽에서 이 값을 쓰고 있었구나"
  → 정산 로직도 수정 → 테스트 전부 통과
  → 자신감 있게 배포

리팩토링:
  UserService 3,000줄 → 5개 클래스로 분리
  → 테스트 200개 전부 통과
  → "기능은 동일하다"는 증명 완료

배포:
  CI에서 테스트 자동 실행 → 전부 초록불 → 배포
  → 결제 버튼 삭제 실수? → 테스트가 잡아줌

신입 온보딩:
  신입: "이 함수 뭐하는 거예요?"
  시니어: "테스트 코드 보세요."
  신입: (테스트 읽음) "아, 이 입력이 들어가면 이 결과가 나오는 거군요"
  신입: 수정 → 테스트 통과 → PR
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. 테스트 종류 — 피라미드

&nbsp;

```
        /\
       /  \        E2E Test (적게)
      /    \       브라우저에서 클릭, 느림, 비쌈
     /------\
    /        \     Integration Test (중간)
   /          \    API 전체 흐름, DB 포함
  /------------\
 /              \  Unit Test (많이)
/________________\ 함수 하나, 빠름, 쌈
```

&nbsp;

| 종류 | 테스트 대상 | 속도 | 비용 | 비중 |
|------|-------------|------|------|------|
| Unit | 함수, 메서드 1개 | 1ms | 낮음 | 70% |
| Integration | API, DB 연동 | 100ms~1s | 중간 | 20% |
| E2E | 브라우저 전체 시나리오 | 5~30s | 높음 | 10% |

&nbsp;

**Unit Test를 가장 많이 짠다.** 빠르고, 정확하고, 유지보수가 쉽다.

&nbsp;

&nbsp;

---

&nbsp;

## 4. "테스트 짤 시간이 없어요"

&nbsp;

이 말을 가장 많이 들었다. 그리고 이 말을 하는 팀이 가장 바빴다.

&nbsp;

```
테스트 없이 개발:

  기능 개발:     3시간
  수동 테스트:   1시간
  배포:          30분
  버그 발견:     (다음 날)
  원인 파악:     2시간
  수정:          1시간
  재배포:        30분
  재테스트:      1시간
  ────────────
  합계:          9시간

테스트 있는 개발:

  기능 개발:     3시간
  테스트 작성:   1시간
  테스트 실행:   10초
  버그 발견:     (즉시, 테스트 실패로)
  수정:          30분
  테스트 재실행: 10초
  배포:          30분
  ────────────
  합계:          5시간
```

&nbsp;

처음에는 테스트 작성 시간이 추가된다. 그런데 **버그 수정 + 재배포 + 재테스트 시간이 사라진다.**

&nbsp;

```
"테스트 짤 시간이 없는 게 아니라,
 테스트 안 짜서 시간이 없는 것이다."
```

&nbsp;

&nbsp;

---

&nbsp;

## 5. 처음 시작하기 — 완벽하지 않아도 된다

&nbsp;

모든 걸 테스트하려고 하면 시작도 못 한다. 세 가지만 기억하자.

&nbsp;

### 규칙 1: 버그 고칠 때 그 버그에 대한 테스트 1개 추가

&nbsp;

```
할인 계산에서 버그 발견:
  - 10% 할인인데 100% 할인이 적용됨
  - 원인: 0.1을 곱해야 하는데 1을 곱함

수정하기 전에 테스트부터 짠다:
```

&nbsp;

```java
@Test
void 할인율_10퍼센트_적용() {
    // given
    int originalPrice = 10000;
    double discountRate = 0.1;

    // when
    int discountedPrice = calculator.applyDiscount(originalPrice, discountRate);

    // then
    assertThat(discountedPrice).isEqualTo(9000);
}
```

&nbsp;

이 테스트가 실패하는 걸 확인 → 코드 수정 → 테스트 통과. 이 버그는 두 번 다시 안 나온다.

이걸 **회귀 테스트(Regression Test)** 라고 한다.

&nbsp;

### 규칙 2: 핵심 비즈니스 로직만

&nbsp;

```
테스트 우선순위:
  1순위: 결제, 정산, 권한 체크  ← 틀리면 돈이 왔다갔다
  2순위: 회원가입, 로그인       ← 틀리면 서비스 못 씀
  3순위: 목록 조회, 검색        ← 틀려도 큰일은 안 남
  4순위: UI 표시, 포맷팅        ← 나중에

전부 다 짤 필요 없다. 1순위부터.
```

&nbsp;

### 규칙 3: 한 테스트에 한 가지만 검증

&nbsp;

```java
// ❌ 나쁜 테스트: 한 번에 너무 많이 검증
@Test
void 주문_테스트() {
    Order order = orderService.create(request);
    assertThat(order.getStatus()).isEqualTo("CREATED");
    assertThat(order.getTotalAmount()).isEqualTo(15000);
    assertThat(order.getItems()).hasSize(3);
    assertThat(userService.getPoint(userId)).isEqualTo(500);
    assertThat(stockService.getStock(itemId)).isEqualTo(97);
}

// ✅ 좋은 테스트: 한 가지만 검증
@Test
void 주문_생성_시_상태는_CREATED() {
    Order order = orderService.create(request);
    assertThat(order.getStatus()).isEqualTo("CREATED");
}

@Test
void 주문_생성_시_포인트_적립() {
    orderService.create(request);
    assertThat(userService.getPoint(userId)).isEqualTo(500);
}

@Test
void 주문_생성_시_재고_차감() {
    orderService.create(request);
    assertThat(stockService.getStock(itemId)).isEqualTo(97);
}
```

&nbsp;

한 가지만 검증하면 실패했을 때 원인이 바로 보인다.

&nbsp;

&nbsp;

---

&nbsp;

## 6. 코드 예시 — Java (JUnit 5 + MockMvc)

&nbsp;

### Unit Test

&nbsp;

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @InjectMocks
    private OrderService orderService;

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private StockService stockService;

    @Test
    void 주문_생성_성공() {
        // given
        OrderRequest request = new OrderRequest(1L, 3);
        when(stockService.getStock(1L)).thenReturn(10);
        when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        // when
        Order order = orderService.create(request);

        // then
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CREATED);
        assertThat(order.getQuantity()).isEqualTo(3);
        verify(stockService).decrease(1L, 3);
    }

    @Test
    void 재고_부족_시_예외() {
        // given
        OrderRequest request = new OrderRequest(1L, 100);
        when(stockService.getStock(1L)).thenReturn(5);

        // when & then
        assertThatThrownBy(() -> orderService.create(request))
                .isInstanceOf(InsufficientStockException.class)
                .hasMessage("재고가 부족합니다. 현재: 5, 요청: 100");
    }
}
```

&nbsp;

### Integration Test (MockMvc)

&nbsp;

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
class OrderApiTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void 주문_생성_API() throws Exception {
        // given
        String requestBody = """
            {
                "itemId": 1,
                "quantity": 3
            }
            """;

        // when & then
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.status").value("CREATED"))
                .andExpect(jsonPath("$.quantity").value(3));

        // DB 검증
        List<Order> orders = orderRepository.findAll();
        assertThat(orders).hasSize(1);
        assertThat(orders.get(0).getStatus()).isEqualTo(OrderStatus.CREATED);
    }

    @Test
    void 존재하지_않는_상품_주문_시_404() throws Exception {
        String requestBody = """
            {
                "itemId": 99999,
                "quantity": 1
            }
            """;

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.message").value("상품을 찾을 수 없습니다"));
    }
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 7. 코드 예시 — Node.js (Jest + Supertest)

&nbsp;

### Unit Test

&nbsp;

```typescript
// orderService.test.ts
describe('OrderService', () => {
    let orderService: OrderService;
    let mockOrderRepo: jest.Mocked<OrderRepository>;
    let mockStockService: jest.Mocked<StockService>;

    beforeEach(() => {
        mockOrderRepo = { save: jest.fn(), findById: jest.fn() } as any;
        mockStockService = { getStock: jest.fn(), decrease: jest.fn() } as any;
        orderService = new OrderService(mockOrderRepo, mockStockService);
    });

    test('주문 생성 성공', async () => {
        // given
        mockStockService.getStock.mockResolvedValue(10);
        mockOrderRepo.save.mockResolvedValue({ id: 1, status: 'CREATED', quantity: 3 });

        // when
        const order = await orderService.create({ itemId: 1, quantity: 3 });

        // then
        expect(order.status).toBe('CREATED');
        expect(order.quantity).toBe(3);
        expect(mockStockService.decrease).toHaveBeenCalledWith(1, 3);
    });

    test('재고 부족 시 예외', async () => {
        // given
        mockStockService.getStock.mockResolvedValue(5);

        // when & then
        await expect(orderService.create({ itemId: 1, quantity: 100 }))
            .rejects
            .toThrow('재고가 부족합니다');
    });
});
```

&nbsp;

### Integration Test (Supertest)

&nbsp;

```typescript
// order.api.test.ts
import request from 'supertest';
import { app } from '../app';
import { db } from '../database';

describe('POST /api/orders', () => {
    beforeEach(async () => {
        await db.query('DELETE FROM orders');
        await db.query('INSERT INTO items (id, name, stock) VALUES (1, "테스트상품", 10)');
    });

    afterAll(async () => {
        await db.close();
    });

    test('주문 생성 성공', async () => {
        const response = await request(app)
            .post('/api/orders')
            .send({ itemId: 1, quantity: 3 })
            .expect(201);

        expect(response.body.status).toBe('CREATED');
        expect(response.body.quantity).toBe(3);

        // DB 검증
        const orders = await db.query('SELECT * FROM orders');
        expect(orders).toHaveLength(1);
    });

    test('존재하지 않는 상품 주문 시 404', async () => {
        const response = await request(app)
            .post('/api/orders')
            .send({ itemId: 99999, quantity: 1 })
            .expect(404);

        expect(response.body.message).toBe('상품을 찾을 수 없습니다');
    });

    test('수량 0 이하 시 400', async () => {
        await request(app)
            .post('/api/orders')
            .send({ itemId: 1, quantity: 0 })
            .expect(400);
    });
});
```

&nbsp;

&nbsp;

---

&nbsp;

## 8. 테스트가 바꾸는 것

&nbsp;

### 코드 품질

&nbsp;

```
테스트를 짜려고 하면, 자연스럽게 코드가 좋아진다.

테스트 불가능한 코드:
  - 한 메서드에서 DB 조회 + 비즈니스 로직 + 외부 API 호출 + 이메일 발송
  - 전부 섞여있어서 Mock을 끼울 수가 없다

테스트 가능한 코드:
  - DB 조회는 Repository
  - 비즈니스 로직은 Service
  - 외부 호출은 Client
  - 각각 분리되어 있어서 독립적으로 테스트 가능

"테스트 가능한 코드 = 잘 설계된 코드"
```

&nbsp;

### 배포 자신감

&nbsp;

```
테스트 없을 때:
  "배포해도 될까..." → 금요일 배포 금지 → 수동 테스트 3시간 → "에이 그냥 하자"

테스트 있을 때:
  CI 실행 → 테스트 352개 통과 → 초록불 → 배포
  금요일이든 월요일이든 상관없다. 테스트가 보증한다.
```

&nbsp;

### 문서 역할

&nbsp;

```java
// 이 함수가 뭘 하는지 주석보다 테스트가 명확하다
@Test void 쿠폰_만료일이_지나면_사용_불가()
@Test void 쿠폰_최소_금액_미달_시_적용_불가()
@Test void 쿠폰_이미_사용된_경우_예외()
@Test void 쿠폰_적용_성공_시_할인_금액_반환()

// 테스트 이름만 읽으면 비즈니스 규칙이 보인다
```

&nbsp;

### 개발 속도

&nbsp;

```
1주차: 테스트 짜느라 개발 속도 30% 느려짐
2주차: 버그가 줄어들기 시작
3주차: 리팩토링이 빨라짐 (테스트가 보호)
4주차: 기능 추가 속도가 오히려 빨라짐
  → 수동 테스트 시간 절약
  → 버그 수정 시간 절약
  → "이거 고쳐도 되나?" 고민 시간 절약

2주만 버티면 된다.
```

&nbsp;

&nbsp;

---

&nbsp;

## 9. 실전 체크리스트

&nbsp;

```
□ 프로젝트에 테스트 프레임워크가 설정되어 있는가?
  → Java: JUnit 5 + Mockito
  → Node.js: Jest
  → Python: pytest

□ CI에서 테스트가 자동 실행되는가?
  → PR 올리면 테스트 돌아가야 한다

□ 핵심 비즈니스 로직에 테스트가 있는가?
  → 결제, 정산, 권한, 가입

□ 버그 수정 시 회귀 테스트를 추가하고 있는가?
  → "이 버그는 두 번 안 나온다"

□ 테스트 실행 시간이 5분 이내인가?
  → 느리면 안 돌린다. 빨라야 자주 돌린다.
```

&nbsp;

&nbsp;

---

&nbsp;

## 시리즈를 마치며

&nbsp;

4편에 걸쳐 "느려요"에 대응하는 방법을 정리했다.

&nbsp;

```
1편: 인덱스 — SELECT가 느릴 때
2편: 슬로우 쿼리 — 쿼리 하나가 서비스를 멈출 때
3편: 캐시 — DB 부하를 90% 줄일 때
4편: 테스트 — 개발자가 느려질 때
```

&nbsp;

1~3편은 시스템 성능, 4편은 개발 생산성. 둘 다 "느림"이고, 둘 다 해결해야 한다.

&nbsp;

핵심은 하나다. **측정하고, 원인을 찾고, 고치고, 결과를 확인한다.** EXPLAIN을 찍고, 슬로우 로그를 열고, 캐시 히트율을 보고, 테스트 통과율을 본다. 감이 아니라 숫자로 판단한다.

&nbsp;

&nbsp;

---

테스트코드, Unit Test, Integration Test, JUnit, Jest, Mockito, MockMvc, Supertest, 회귀테스트, TDD, 리팩토링, CI, 배포, 개발생산성, 코드품질