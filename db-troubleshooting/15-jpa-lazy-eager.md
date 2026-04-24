# JPA 즉시 로딩 vs 지연 로딩 — EAGER는 왜 있는데 왜 쓰면 안 되는가

&nbsp;

Member를 조회할 때 Team도 같이 가져와야 할까?

&nbsp;

"항상 같이 쓰니까 같이 가져오면 되지" → 이게 EAGER.

"필요할 때만 가져오자" → 이게 LAZY.

&nbsp;

간단해 보이는데, 이 선택 하나가 **쿼리 1번과 101번의 차이**를 만든다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 식당으로 이해하기

&nbsp;

```
코스 요리를 시켰다.

즉시 로딩 (EAGER):
  "전부 한꺼번에 가져다 주세요"
  → 전채, 수프, 메인, 디저트가 한 번에 나옴
  → 디저트 안 먹을 건데도 테이블에 올라옴
  → 테이블(메모리) 꽉 참

지연 로딩 (LAZY):
  "일단 전채만 주세요. 나머지는 먹을 때 가져다 주세요"
  → 전채만 나옴
  → 수프 먹고 싶으면 그때 "수프 주세요" → 그때 가져옴
  → 디저트 안 먹으면 안 가져옴
  → 테이블 깔끔
```

&nbsp;

&nbsp;

---

&nbsp;

# 2. 코드로 보기

&nbsp;

## EAGER — 즉시 로딩

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.EAGER)  // 즉시 로딩
    private Team team;
}
```

&nbsp;

```java
Member member = em.find(Member.class, 1L);
```

&nbsp;

```sql
-- 실행되는 SQL: JOIN으로 한 번에 가져옴
SELECT m.*, t.*
FROM Member m
LEFT JOIN Team t ON m.team_id = t.id
WHERE m.id = 1
```

&nbsp;

```java
member.getTeam().getName();  // 이미 가져왔으니까 추가 쿼리 없음
member.getTeam().getClass(); // Team (실제 객체)
```

&nbsp;

## LAZY — 지연 로딩

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)  // 지연 로딩
    private Team team;
}
```

&nbsp;

```java
Member member = em.find(Member.class, 1L);
```

&nbsp;

```sql
-- 실행되는 SQL: Member만 가져옴
SELECT m.*
FROM Member m
WHERE m.id = 1
```

&nbsp;

```java
member.getTeam();             // 프록시(가짜 객체) 반환, 쿼리 안 나감
member.getTeam().getClass();  // Team$HibernateProxy$e97rdqZR (프록시)

member.getTeam().getName();   // ← 이 순간! Team 쿼리 실행
```

&nbsp;

```sql
-- getName() 호출 시점에 실행
SELECT t.*
FROM Team t
WHERE t.id = 1
```

&nbsp;

**LAZY는 "진짜 필요한 순간"까지 쿼리를 미룬다.**

&nbsp;

&nbsp;

---

&nbsp;

# 3. 그러면 EAGER가 편하지 않나?

&nbsp;

한 명 조회할 때는 그렇다.

**여러 명 조회하면 폭발한다.**

&nbsp;

## N+1 문제

&nbsp;

```java
// 회원 100명 조회 (JPQL)
List<Member> members = em.createQuery(
    "SELECT m FROM Member m", Member.class
).getResultList();
```

&nbsp;

### EAGER일 때:

&nbsp;

```sql
-- 1. 먼저 Member 전체 조회 (1번)
SELECT * FROM Member

-- 2. 각 Member의 Team을 하나씩 조회 (N번)
SELECT * FROM Team WHERE id = 1
SELECT * FROM Team WHERE id = 2
SELECT * FROM Team WHERE id = 3
...
SELECT * FROM Team WHERE id = 100

-- 총 101번 쿼리
```

&nbsp;

JPA가 내부적으로 이렇게 동작한다:

```
1. "SELECT m FROM Member m"을 SQL로 변환 → Member만 가져옴
2. 가져와 보니 Team이 EAGER네?
3. 반환하기 전에 Team이 채워져 있어야 하니까
4. Member 100명의 Team을 하나씩 조회
5. = N+1 (1번 쿼리 + N번 추가 쿼리)
```

&nbsp;

### LAZY일 때:

&nbsp;

```sql
-- Member만 조회 (1번)
SELECT * FROM Member

-- Team? 프록시로 넣어둠. 쿼리 안 나감.
-- 총 1번 쿼리
```

&nbsp;

```
회원 100명, 팀 100개:
  EAGER: 쿼리 101번
  LAZY:  쿼리 1번

회원 10,000명이면:
  EAGER: 쿼리 10,001번
  LAZY:  쿼리 1번
```

&nbsp;

**이게 N+1 문제다.** 쿼리 1번 날렸는데 추가로 N번이 따라오는 것.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 근데 Team도 같이 필요하면?

&nbsp;

LAZY로 설정하되, **필요할 때만 fetch join**으로 한 번에 가져온다.

&nbsp;

```java
// LAZY인데 Team도 같이 필요한 경우: fetch join
List<Member> members = em.createQuery(
    "SELECT m FROM Member m JOIN FETCH m.team", Member.class
).getResultList();
```

&nbsp;

```sql
-- 실행되는 SQL: JOIN으로 한 번에
SELECT m.*, t.*
FROM Member m
JOIN Team t ON m.team_id = t.id

-- 총 1번 쿼리. EAGER처럼 동작하지만 내가 원할 때만.
```

&nbsp;

```
EAGER:           항상 같이 가져옴 (선택 불가)
LAZY + fetch join: 필요할 때만 같이 가져옴 (선택 가능)
```

&nbsp;

**같은 결과인데 제어권이 개발자에게 있다.** 이게 핵심이다.

&nbsp;

&nbsp;

---

&nbsp;

# 5. 그러면 EAGER는 왜 존재하는 거야?

&nbsp;

없어도 될 것 같은데 있는 이유가 있다.

&nbsp;

## 이유 1: 항상 같이 쓰는 1:1 관계

&nbsp;

```java
User → UserProfile (1:1)
  → 유저 조회할 때 프로필 안 쓰는 경우가 없음
  → LAZY면 매번 쿼리 2번
  → EAGER면 JOIN으로 1번

Order → Payment (1:1)
  → 주문 보면서 결제정보 안 보는 경우가 없음
```

&nbsp;

## 이유 2: em.find() 단건 조회에서는 N+1이 안 생긴다

&nbsp;

```java
// 이건 괜찮음 — JPA가 JOIN으로 최적화
Member member = em.find(Member.class, 1L);
// → SELECT member JOIN team (쿼리 1번)

// 이게 문제 — JPQL은 SQL 그대로 변환
List<Member> members = em.createQuery("SELECT m FROM Member m").getResultList();
// → SELECT member (1번) + SELECT team × N번
```

&nbsp;

N+1은 **EAGER + JPQL 조합**에서 터지는 거지, `em.find()` 단건에서는 괜찮다.

&nbsp;

## 이유 3: JPA 초기 설계의 편의 기능

&nbsp;

JPA가 처음 만들어질 때 "자주 같이 쓰는 건 같이 가져오면 편하지 않을까?"라는 의도로 넣었다.

틀린 생각은 아닌데, **실무에서 규모가 커지면 독이 된다**는 걸 다들 경험으로 배웠다.

&nbsp;

&nbsp;

---

&nbsp;

# 6. EAGER를 실무에서 안 쓰는 진짜 이유

&nbsp;

```
처음: "User랑 Profile은 항상 같이 쓰니까 EAGER로 하자"
  ↓
3개월 후: "User 목록 페이지 만들었는데 왜 이렇게 느리지?"
  ↓
확인: User 100명 조회 → Profile 100번 추가 조회 (N+1)
  ↓
"EAGER 때문이네... LAZY로 바꿔야지"
  ↓
다른 곳에서 user.getProfile() 호출하던 코드가
LazyInitializationException 터짐
  ↓
전부 수정...
```

&nbsp;

**한 번 EAGER로 설정하면 나중에 LAZY로 바꾸기 어렵다.**

반대로 LAZY에서 fetch join 추가하는 건 쉽다.

&nbsp;

```
EAGER → LAZY 전환: 기존 코드 다 깨질 수 있음 (어려움)
LAZY → fetch join 추가: 쿼리 한 줄 추가 (쉬움)
```

&nbsp;

&nbsp;

---

&nbsp;

# 7. 주의: @ManyToOne, @OneToOne은 기본이 EAGER

&nbsp;

```java
@ManyToOne   // 기본값: FetchType.EAGER ← 위험!
@OneToOne    // 기본값: FetchType.EAGER ← 위험!

@OneToMany   // 기본값: FetchType.LAZY (안전)
@ManyToMany  // 기본값: FetchType.LAZY (안전)
```

&nbsp;

**@ManyToOne, @OneToOne을 쓸 때는 반드시 LAZY를 명시하자.**

&nbsp;

```java
// 위험: 기본값이 EAGER
@ManyToOne
private Team team;

// 안전: LAZY 명시
@ManyToOne(fetch = FetchType.LAZY)
private Team team;
```

&nbsp;

&nbsp;

---

&nbsp;

# 8. Node.js TypeORM에서는?

&nbsp;

같은 개념이 있다.

&nbsp;

```typescript
// 기본: 관계를 안 가져옴 (LAZY와 비슷)
const member = await memberRepo.findOne({
  where: { id: 1 }
});
// member.team → undefined (가져오지 않음)

// 필요할 때 명시 (fetch join과 비슷)
const member = await memberRepo.findOne({
  where: { id: 1 },
  relations: ['team'],  // 명시적으로 요청
});
// member.team → Team 객체 (같이 가져옴)
```

&nbsp;

```typescript
// eager 설정도 가능 (비추천)
@ManyToOne({ eager: true })  // 항상 같이 가져옴
team: Team;

// 추천
@ManyToOne()  // 기본: 안 가져옴
team: Team;
// → relations: ['team']으로 필요할 때만
```

&nbsp;

TypeORM은 기본이 "안 가져옴"이라 JPA보다 안전하다.

JPA는 @ManyToOne 기본이 EAGER라서 실수하기 쉽다.

&nbsp;

&nbsp;

---

&nbsp;

# 9. 정리

&nbsp;

```
EAGER:
  ✅ 단건 조회 시 편리 (JOIN 1번)
  ❌ 목록 조회 시 N+1 폭발
  ❌ 안 쓰는 데이터도 항상 가져옴
  ❌ 나중에 LAZY로 바꾸기 어려움
  ❌ 쿼리 예측 불가

LAZY:
  ✅ 필요할 때만 가져옴
  ✅ N+1 방지
  ✅ fetch join으로 유연하게 제어
  ✅ 쿼리 예측 가능
  ❌ 매번 relations/fetch join 명시해야 함 (귀찮음)
```

&nbsp;

## 결론

&nbsp;

```
1. 전부 LAZY로 설정한다
2. @ManyToOne, @OneToOne에 반드시 fetch = FetchType.LAZY 명시
3. 같이 가져와야 할 때는 fetch join 사용
4. EAGER는 쓰지 않는다
```

&nbsp;

EAGER가 존재하는 이유는 "편의 기능"이었다.

근데 **편한 게 항상 좋은 건 아니다.**

&nbsp;

LAZY + fetch join이 한 줄 더 쓰지만,

그 한 줄이 **쿼리 100번과 1번의 차이**를 만든다.

&nbsp;

&nbsp;

---

JPA, 즉시로딩, 지연로딩, EAGER, LAZY, N+1, fetch join, Spring, Hibernate, ORM, 프록시, TypeORM, 데이터베이스, 성능최적화
