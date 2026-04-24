# N+1 문제 총정리 — EAGER만 문제가 아니다

&nbsp;

이전 글에서 EAGER의 N+1 문제를 다뤘다.

&nbsp;

"그러면 LAZY로 바꾸면 되는 거 아니야?"

&nbsp;

**아니다. LAZY에서도 N+1은 터진다.**

&nbsp;

오히려 LAZY의 N+1이 더 위험하다. 코드에서 안 보이니까.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 택배로 이해하기

&nbsp;

```
회원 100명의 팀 이름을 출력해야 한다.

EAGER의 N+1:
  "회원 목록 주세요" → 회원 100명 도착
  → 근데 택배 기사가 각 회원의 팀 정보도 따로따로 배달
  → 택배차 101번 옴 (1번 + 100번)

LAZY에서 N+1 안 터지는 경우:
  "회원 목록 주세요" → 회원 100명 도착
  → 팀 정보? 안 열어봄 → 택배차 1번만 옴

LAZY에서 N+1 터지는 경우:
  "회원 목록 주세요" → 회원 100명 도착
  → 근데 각 회원의 팀 이름을 하나씩 열어봄
  → 열어볼 때마다 택배차가 옴
  → 택배차 101번 옴 (1번 + 100번)
```

&nbsp;

**LAZY는 "안 쓰면 안 가져온다"일 뿐, "쓰면 하나씩 가져온다."**

&nbsp;

&nbsp;

---

&nbsp;

# 2. EAGER에서의 N+1 (복습)

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.EAGER)
    private Team team;
}
```

&nbsp;

```java
List<Member> members = em.createQuery(
    "SELECT m FROM Member m", Member.class
).getResultList();
```

&nbsp;

```sql
-- 1번: Member 전체 조회
SELECT * FROM member

-- N번: 각 Member의 Team 조회
SELECT * FROM team WHERE id = 1
SELECT * FROM team WHERE id = 2
SELECT * FROM team WHERE id = 3
...
SELECT * FROM team WHERE id = 100
```

&nbsp;

JPQL은 SQL을 그대로 만들기 때문에, EAGER여도 JOIN을 자동으로 넣지 않는다.

Member를 가져온 뒤, "어? EAGER네?" 하고 Team을 하나씩 추가 조회한다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. LAZY에서도 N+1이 터지는 경우

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)  // LAZY로 바꿨다
    private Team team;
}
```

&nbsp;

```java
List<Member> members = em.createQuery(
    "SELECT m FROM Member m", Member.class
).getResultList();

// 여기까지는 쿼리 1번. 문제없다.
```

&nbsp;

```sql
SELECT * FROM member
-- 끝. Team 쿼리 안 나감. LAZY니까.
```

&nbsp;

**근데 이렇게 하면?**

&nbsp;

```java
// 각 회원의 팀 이름을 출력
for (Member member : members) {
    System.out.println(member.getTeam().getName());  // ← 여기!
}
```

&nbsp;

```sql
-- for문이 돌면서 Team 쿼리가 N번 나감
SELECT * FROM team WHERE id = 1   -- 1번째 회원의 팀
SELECT * FROM team WHERE id = 2   -- 2번째 회원의 팀
SELECT * FROM team WHERE id = 3   -- 3번째 회원의 팀
...
SELECT * FROM team WHERE id = 100 -- 100번째 회원의 팀
```

&nbsp;

**LAZY인데 N+1이 터졌다.**

&nbsp;

```
EAGER N+1: 조회하는 순간 바로 터짐 (눈에 보임)
LAZY N+1:  for문에서 getter 호출할 때 터짐 (코드에서 안 보임)
```

&nbsp;

LAZY의 N+1이 더 위험한 이유는, **쿼리 로그를 안 보면 모른다**는 것이다.

코드만 보면 그냥 getter 호출인데, 내부에서는 SELECT가 N번 나가고 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 해결법 1 — fetch join

&nbsp;

가장 많이 쓰는 방법이다.

&nbsp;

```java
List<Member> members = em.createQuery(
    "SELECT m FROM Member m JOIN FETCH m.team", Member.class
).getResultList();
```

&nbsp;

```sql
-- 실행되는 SQL: JOIN으로 한 번에 가져옴
SELECT m.*, t.*
FROM member m
INNER JOIN team t ON m.team_id = t.id
```

&nbsp;

```java
// 이제 for문 돌아도 추가 쿼리 안 나감
for (Member member : members) {
    System.out.println(member.getTeam().getName());  // 이미 가져옴
}
```

&nbsp;

**쿼리 1번으로 끝난다.**

&nbsp;

## fetch join 주의점

&nbsp;

```java
// 1:N 컬렉션 fetch join 시 데이터 중복 가능
List<Team> teams = em.createQuery(
    "SELECT t FROM Team t JOIN FETCH t.members", Team.class
).getResultList();

// Team A에 Member가 3명이면 → Team A가 3번 나옴
// 해결: DISTINCT 사용
List<Team> teams = em.createQuery(
    "SELECT DISTINCT t FROM Team t JOIN FETCH t.members", Team.class
).getResultList();
```

&nbsp;

```java
// 컬렉션 fetch join은 페이징 불가
List<Team> teams = em.createQuery(
    "SELECT t FROM Team t JOIN FETCH t.members", Team.class
)
.setFirstResult(0)
.setMaxResults(10)  // ← 경고! 메모리에서 페이징함
.getResultList();

// Hibernate 경고:
// HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
// → 전체 데이터를 메모리에 올린 뒤 잘라냄 → OutOfMemoryError 위험
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. 해결법 2 — @EntityGraph

&nbsp;

fetch join을 어노테이션으로 쓸 수 있다. Spring Data JPA에서 편하다.

&nbsp;

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    // 기본 findAll()은 Team을 안 가져옴 (LAZY)
    // @EntityGraph를 붙이면 fetch join과 같은 효과
    @EntityGraph(attributePaths = {"team"})
    @Override
    List<Member> findAll();
}
```

&nbsp;

```sql
-- 실행되는 SQL
SELECT m.*, t.*
FROM member m
LEFT JOIN team t ON m.team_id = t.id
```

&nbsp;

```java
// 메서드 이름 쿼리에도 사용 가능
@EntityGraph(attributePaths = {"team"})
List<Member> findByName(String name);

// 여러 연관관계 동시에
@EntityGraph(attributePaths = {"team", "orders"})
List<Member> findAll();
```

&nbsp;

**장점:** JPQL 안 써도 됨, 간결함

**단점:** 복잡한 조건이 필요하면 결국 JPQL fetch join을 써야 함

&nbsp;

&nbsp;

---

&nbsp;

# 6. 해결법 3 — @BatchSize

&nbsp;

fetch join이 안 되는 상황이 있다. **컬렉션 2개 이상 동시 fetch join은 불가능**하다.

&nbsp;

```java
// 이렇게 하면 MultipleBagFetchException 터짐
List<Team> teams = em.createQuery(
    "SELECT t FROM Team t JOIN FETCH t.members JOIN FETCH t.projects",
    Team.class
).getResultList();
// → org.hibernate.loader.MultipleBagFetchException:
//   cannot simultaneously fetch multiple bags
```

&nbsp;

이때 @BatchSize를 쓴다.

&nbsp;

```java
@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @BatchSize(size = 100)
    @OneToMany(mappedBy = "team")
    private List<Member> members;
}
```

&nbsp;

```java
List<Team> teams = em.createQuery(
    "SELECT t FROM Team t", Team.class
).getResultList();

// Team이 150개라면?
for (Team team : teams) {
    team.getMembers().size();  // 이 시점에 IN 쿼리 발생
}
```

&nbsp;

```sql
-- @BatchSize 없으면: Team 150개 → Member 쿼리 150번
SELECT * FROM member WHERE team_id = 1
SELECT * FROM member WHERE team_id = 2
...
SELECT * FROM member WHERE team_id = 150

-- @BatchSize(size = 100)이면: IN 절로 묶어서 2번
SELECT * FROM member WHERE team_id IN (1, 2, 3, ... , 100)   -- 100개
SELECT * FROM member WHERE team_id IN (101, 102, ... , 150)   -- 50개
```

&nbsp;

```
쿼리 150번 → 쿼리 2번으로 줄어든다.
```

&nbsp;

## 글로벌 설정 (권장)

&nbsp;

엔티티마다 @BatchSize 붙이기 귀찮으면 글로벌로 설정한다.

&nbsp;

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

&nbsp;

이러면 모든 지연 로딩에 자동으로 IN 절이 적용된다.

**실무에서 가장 먼저 해야 할 설정이다.**

&nbsp;

&nbsp;

---

&nbsp;

# 7. 해결법 4 — DTO 직접 조회

&nbsp;

엔티티가 아니라 필요한 데이터만 뽑아오는 방법이다.

&nbsp;

```java
// DTO 정의
public class MemberTeamDto {
    private String memberName;
    private String teamName;

    public MemberTeamDto(String memberName, String teamName) {
        this.memberName = memberName;
        this.teamName = teamName;
    }
}
```

&nbsp;

```java
List<MemberTeamDto> result = em.createQuery(
    "SELECT new com.example.dto.MemberTeamDto(m.name, t.name) " +
    "FROM Member m JOIN m.team t", MemberTeamDto.class
).getResultList();
```

&nbsp;

```sql
-- 실행되는 SQL: 필요한 컬럼만 가져옴
SELECT m.name, t.name
FROM member m
JOIN team t ON m.team_id = t.id
```

&nbsp;

**장점:** 필요한 것만 가져오니까 성능 최고, N+1 원천 차단

**단점:** DTO 만들어야 하고, 패키지명까지 써야 해서 번거로움

&nbsp;

&nbsp;

---

&nbsp;

# 8. 해결법 비교

&nbsp;

```
┌──────────────────┬────────────┬──────────────┬──────────────┐
│ 방법             │ 쿼리 수    │ 편의성       │ 제한사항     │
├──────────────────┼────────────┼──────────────┼──────────────┤
│ fetch join       │ 1번        │ ★★★★       │ 컬렉션 2개↑  │
│                  │            │              │ 동시 불가    │
│                  │            │              │ 페이징 주의  │
├──────────────────┼────────────┼──────────────┼──────────────┤
│ @EntityGraph     │ 1번        │ ★★★★★     │ 복잡한 조건  │
│                  │            │              │ 처리 어려움  │
├──────────────────┼────────────┼──────────────┼──────────────┤
│ @BatchSize       │ 1 + α번   │ ★★★★★     │ 완전한 1번은 │
│                  │ (매우 적음) │              │ 아님         │
├──────────────────┼────────────┼──────────────┼──────────────┤
│ DTO 직접 조회    │ 1번        │ ★★★        │ DTO 작성     │
│                  │            │              │ 필요         │
└──────────────────┴────────────┴──────────────┴──────────────┘
```

&nbsp;

&nbsp;

---

&nbsp;

# 9. 실무 권장 전략

&nbsp;

```
1단계: 글로벌 BatchSize 설정
  → application.yml에 default_batch_fetch_size: 100
  → 이것만으로 대부분의 N+1이 사라짐

2단계: 핵심 API에 fetch join 적용
  → 자주 호출되는 API는 fetch join으로 쿼리 1번으로 줄이기

3단계: 성능이 중요한 곳은 DTO 직접 조회
  → 화면에 필요한 데이터만 뽑아오기
```

&nbsp;

```java
// 이 순서로 적용하면 된다

// 1. 먼저 글로벌 설정 (전체 적용)
spring.jpa.properties.hibernate.default_batch_fetch_size=100

// 2. 핫 API에 fetch join
@Query("SELECT m FROM Member m JOIN FETCH m.team WHERE m.status = :status")
List<Member> findActiveMembers(@Param("status") String status);

// 3. 대시보드 같은 곳은 DTO
@Query("SELECT new MemberSummaryDto(m.name, t.name, COUNT(o)) " +
       "FROM Member m JOIN m.team t LEFT JOIN m.orders o " +
       "GROUP BY m.name, t.name")
List<MemberSummaryDto> getMemberSummary();
```

&nbsp;

&nbsp;

---

&nbsp;

# 10. 정리

&nbsp;

```
N+1 문제:
  ❌ EAGER만의 문제가 아니다
  ❌ LAZY로 바꿔도 getter 호출하면 똑같이 터진다
  ✅ LAZY + fetch join 조합이 정답
  ✅ 글로벌 BatchSize 설정이 첫 번째 할 일
  ✅ 성능 중요한 곳은 DTO 직접 조회
```

&nbsp;

## 핵심 한 줄

&nbsp;

**N+1은 "로딩 전략"의 문제가 아니라 "조회 전략"의 문제다.**

&nbsp;

EAGER든 LAZY든, 연관 데이터를 어떻게 가져올지 명시하지 않으면 N+1은 언제든 터진다.

&nbsp;

&nbsp;

---

&nbsp;

다음 편 예고: **@Transactional의 함정 — 붙였는데 왜 안 돼?**

@Transactional을 붙이면 다 되는 줄 알았는데, 5가지 경우에 안 된다. private 메서드, self-invocation, checked exception... 하나씩 코드로 확인한다.

&nbsp;

&nbsp;

---

N+1, JPA, JPQL, fetch join, EntityGraph, BatchSize, DTO, Hibernate, Spring Data JPA, 지연로딩, 즉시로딩, 성능최적화, 쿼리최적화, ORM
