# JPA 더티 체킹 — save() 안 해도 되는데 왜? 그리고 언제 안 될까?

&nbsp;

JPA를 처음 쓸 때 가장 놀라는 순간이 있다.

&nbsp;

"어? save() 안 했는데 UPDATE가 나갔어?"

&nbsp;

이게 **더티 체킹(Dirty Checking)**이다.

편하지만, **원리를 모르면 왜 되는지도 왜 안 되는지도 모른다.**

&nbsp;

&nbsp;

---

&nbsp;

# 1. 메모장으로 이해하기

&nbsp;

```
JPA의 영속성 컨텍스트 = 메모장

1. DB에서 회원을 꺼내옴
   → 메모장에 "이름: 홍길동" 이라고 적어둠 (스냅샷)

2. 코드에서 이름을 "김철수"로 바꿈
   → 메모장에 적힌 건 여전히 "홍길동"

3. 트랜잭션 끝날 때
   → 메모장 확인: "홍길동" ≠ "김철수" → 달라졌네!
   → UPDATE 쿼리 자동 실행

이게 더티 체킹이다.
"메모장(스냅샷)이랑 현재 상태를 비교해서, 다르면 자동 업데이트."
```

&nbsp;

&nbsp;

---

&nbsp;

# 2. 코드로 보기

&nbsp;

## save() 없이 UPDATE가 되는 경우

&nbsp;

```java
@Service
public class MemberService {

    @Transactional
    public void updateName(Long id, String newName) {
        Member member = memberRepository.findById(id).orElseThrow();
        // member는 영속 상태 (영속성 컨텍스트가 관리 중)

        member.setName(newName);
        // save() 안 함!
        // 그런데 UPDATE 나감!
    }
}
```

&nbsp;

```sql
-- 트랜잭션 커밋 시점에 자동 실행됨
UPDATE member SET name = '김철수' WHERE id = 1
```

&nbsp;

```java
// save()를 호출해도 결과는 같다
member.setName(newName);
memberRepository.save(member);  // 이미 영속 상태면 save()는 의미 없음
// JPA가 어차피 더티 체킹으로 UPDATE 날림
```

&nbsp;

**영속 상태 엔티티는 set만 해도 UPDATE가 나간다. save()는 불필요하다.**

&nbsp;

&nbsp;

---

&nbsp;

# 3. 동작 원리 — 영속성 컨텍스트와 스냅샷

&nbsp;

```
[영속성 컨텍스트 (1차 캐시)]

┌─────────────────────────────────────────────────┐
│ ID    │ Entity (현재 상태)  │ Snapshot (최초 상태) │
├───────┼────────────────────┼─────────────────────┤
│ 1     │ Member(김철수)      │ Member(홍길동)       │ ← 다르다!
│ 2     │ Member(이영희)      │ Member(이영희)       │ ← 같다
└───────┴────────────────────┴─────────────────────┘

트랜잭션 커밋 시점:
  ID=1: 현재 ≠ 스냅샷 → UPDATE 실행
  ID=2: 현재 = 스냅샷 → UPDATE 안 함
```

&nbsp;

## 상세 흐름

&nbsp;

```java
@Transactional
public void updateName(Long id, String newName) {

    // 1. DB에서 조회 → 영속성 컨텍스트에 저장 + 스냅샷 생성
    Member member = memberRepository.findById(id).orElseThrow();
    // 1차 캐시: { entity: Member(홍길동), snapshot: Member(홍길동) }

    // 2. 엔티티 값 변경
    member.setName(newName);
    // 1차 캐시: { entity: Member(김철수), snapshot: Member(홍길동) }

    // 3. 트랜잭션 커밋 시점 (메서드 종료)
    // → flush() 호출
    // → 1차 캐시의 entity와 snapshot을 필드 단위로 비교
    // → name이 다르다 → UPDATE 쿼리 생성 & 실행
}
```

&nbsp;

```
flush() 타이밍:
  1. 트랜잭션 커밋 직전 (자동)
  2. JPQL 쿼리 실행 직전 (자동)
  3. em.flush() 직접 호출 (수동)
```

&nbsp;

&nbsp;

---

&nbsp;

# 4. 더티 체킹이 안 되는 경우

&nbsp;

## 경우 1 — @Transactional이 없을 때

&nbsp;

```java
@Service
public class MemberService {

    // @Transactional 없음!
    public void updateName(Long id, String newName) {
        Member member = memberRepository.findById(id).orElseThrow();
        member.setName(newName);
        // UPDATE 안 나감!
    }
}
```

&nbsp;

왜? @Transactional이 없으면 영속성 컨텍스트의 생명주기가 **findById() 호출 때 시작되고 즉시 끝난다.** 반환된 member는 **준영속(detached)** 상태다.

&nbsp;

```
영속(managed):  영속성 컨텍스트가 관리 중 → 더티 체킹 O
준영속(detached): 영속성 컨텍스트에서 분리됨 → 더티 체킹 X
비영속(transient): 아직 저장 안 됨 → 더티 체킹 X
```

&nbsp;

## 경우 2 — detach로 분리했을 때

&nbsp;

```java
@Transactional
public void updateName(Long id, String newName) {
    Member member = memberRepository.findById(id).orElseThrow();

    entityManager.detach(member);  // 영속성 컨텍스트에서 분리!

    member.setName(newName);
    // UPDATE 안 나감! detach 됐으니까.
}
```

&nbsp;

```java
// clear()도 마찬가지
entityManager.clear();  // 영속성 컨텍스트 전체 초기화
member.setName(newName);
// UPDATE 안 나감!
```

&nbsp;

## 경우 3 — 벌크 연산 후

&nbsp;

이게 실무에서 가장 위험한 케이스다.

&nbsp;

```java
@Transactional
public void bulkUpdate() {
    // 1. 회원 조회 → 영속성 컨텍스트에 저장
    Member member = memberRepository.findById(1L).orElseThrow();
    System.out.println(member.getAge());  // 20

    // 2. 벌크 연산: DB에서 직접 UPDATE
    entityManager.createQuery(
        "UPDATE Member m SET m.age = m.age + 1"
    ).executeUpdate();

    // DB에는 age = 21
    // 영속성 컨텍스트에는 age = 20 (반영 안 됨!)

    // 3. 다시 조회해도 1차 캐시에서 가져옴
    Member member2 = memberRepository.findById(1L).orElseThrow();
    System.out.println(member2.getAge());  // 20 ← DB는 21인데!

    // DB와 영속성 컨텍스트 불일치!
}
```

&nbsp;

```
벌크 연산의 문제:
  DB에 직접 쿼리를 날림 → 영속성 컨텍스트를 건너뜀
  → DB와 1차 캐시가 다른 상태
  → 이후 조회가 틀린 값을 반환
```

&nbsp;

## 해결: 벌크 연산 후 em.clear()

&nbsp;

```java
@Transactional
public void bulkUpdate() {
    Member member = memberRepository.findById(1L).orElseThrow();
    System.out.println(member.getAge());  // 20

    entityManager.createQuery(
        "UPDATE Member m SET m.age = m.age + 1"
    ).executeUpdate();

    entityManager.flush();  // 아직 안 보낸 변경사항 보내기
    entityManager.clear();  // 1차 캐시 초기화!

    // 이제 DB에서 새로 가져옴
    Member member2 = memberRepository.findById(1L).orElseThrow();
    System.out.println(member2.getAge());  // 21 ← 정확!
}
```

&nbsp;

Spring Data JPA에서는 더 간단하게 할 수 있다.

&nbsp;

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    @Modifying(clearAutomatically = true)  // 자동 clear
    @Query("UPDATE Member m SET m.age = m.age + 1")
    int bulkAgePlusOne();
}
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. merge() vs 더티 체킹

&nbsp;

초보가 자주 헷갈리는 부분이다.

&nbsp;

```java
// 더티 체킹: 영속 상태에서 set → 자동 UPDATE
@Transactional
public void updateByDirtyChecking(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();  // 영속
    member.setName(name);  // 변경 감지 → UPDATE
}

// merge(): 준영속 객체를 다시 영속화
@Transactional
public void updateByMerge(MemberDto dto) {
    Member detachedMember = new Member();
    detachedMember.setId(dto.getId());
    detachedMember.setName(dto.getName());
    // detachedMember는 비영속/준영속 상태

    Member mergedMember = entityManager.merge(detachedMember);
    // mergedMember = 영속 상태의 새 객체
    // detachedMember는 여전히 준영속!
}
```

&nbsp;

## merge()의 함정

&nbsp;

```java
@Transactional
public void mergeTrap() {
    Member detached = new Member();
    detached.setId(1L);
    detached.setName("새이름");

    Member merged = entityManager.merge(detached);

    detached.setName("또다른이름");
    // ← 이건 반영 안 됨! detached는 여전히 준영속!

    merged.setName("또다른이름");
    // ← 이건 반영됨! merged가 영속 상태!
}
```

&nbsp;

```
merge()의 동작:
  1. 1차 캐시에서 id로 엔티티 찾기
  2. 없으면 DB에서 조회
  3. 찾은 엔티티에 전달받은 값을 복사
  4. 찾은 엔티티(영속)를 반환

주의: 원본 객체가 영속이 되는 게 아니라, 새 객체가 반환됨!
```

&nbsp;

**실무에서는 merge()보다 더티 체킹이 안전하다. 가능하면 merge() 대신 "조회 → 값 변경" 패턴을 쓰자.**

&nbsp;

&nbsp;

---

&nbsp;

# 6. save()가 필요한 경우 vs 불필요한 경우

&nbsp;

```java
// save() 불필요: 이미 영속 상태
@Transactional
public void update(Long id, String name) {
    Member member = memberRepository.findById(id).orElseThrow();  // 영속
    member.setName(name);
    // memberRepository.save(member);  ← 불필요! 더티 체킹이 해줌
}

// save() 필요: 새 엔티티 저장
@Transactional
public void create(String name) {
    Member member = new Member();  // 비영속
    member.setName(name);
    memberRepository.save(member);  // ← 필요! persist() 호출
}

// save()가 헷갈리는 경우: id가 있는 새 객체
@Transactional
public void upsert(MemberDto dto) {
    Member member = new Member();
    member.setId(dto.getId());     // id 세팅
    member.setName(dto.getName());
    memberRepository.save(member);
    // id가 있으면 → merge() 호출 (기존 데이터 덮어쓰기)
    // id가 없으면 → persist() 호출 (새로 저장)
}
```

&nbsp;

```
Spring Data JPA의 save() 내부:

if (entity.isNew()) {
    em.persist(entity);   // INSERT
} else {
    em.merge(entity);     // SELECT + UPDATE
}

isNew() 판단: @Id 값이 null이면 new, 아니면 기존
```

&nbsp;

&nbsp;

---

&nbsp;

# 7. 성능 관점

&nbsp;

더티 체킹은 편하지만, 모든 상황에서 최선은 아니다.

&nbsp;

```java
// 더티 체킹: 한 건씩 UPDATE
@Transactional
public void updateAllAges() {
    List<Member> members = memberRepository.findAll();  // 1000건
    for (Member member : members) {
        member.setAge(member.getAge() + 1);
    }
    // 커밋 시 UPDATE 1000번 실행!
}

// 벌크 연산: 한 번에 UPDATE
@Transactional
public void bulkUpdateAllAges() {
    memberRepository.bulkAgePlusOne();  // UPDATE 1번!
    entityManager.clear();
}
```

&nbsp;

```
더티 체킹:
  ✅ 코드 간결, 엔티티 단위 로직 처리
  ❌ 대량 데이터 시 느림 (건건이 UPDATE)

벌크 연산:
  ✅ 대량 데이터 빠름 (한 번에 UPDATE)
  ❌ 영속성 컨텍스트 불일치 주의 (clear 필수)
```

&nbsp;

**1~10건: 더티 체킹, 100건 이상: 벌크 연산을 고려하자.**

&nbsp;

&nbsp;

---

&nbsp;

# 8. 정리

&nbsp;

```
더티 체킹이란:
  → @Transactional 안에서 영속 엔티티의 필드를 바꾸면
  → 트랜잭션 커밋 시 스냅샷과 비교
  → 다르면 UPDATE 자동 실행
  → save() 불필요

더티 체킹이 안 되는 경우:
  1. @Transactional 없음 → 준영속 상태
  2. detach/clear로 분리 → 관리 안 됨
  3. 벌크 연산 후 → 영속성 컨텍스트 불일치

실무 규칙:
  1. 수정은 "조회 → set" 패턴 (merge 지양)
  2. 벌크 연산 후 em.clear() 또는 @Modifying(clearAutomatically = true)
  3. 대량 데이터는 벌크 연산 사용
  4. save()는 새 엔티티에만 (기존 엔티티는 더티 체킹)
```

&nbsp;

&nbsp;

---

&nbsp;

다음 편 예고: **Entity를 API 응답으로 직접 반환하면 생기는 일**

"귀찮으니까 Entity 그대로 반환하면 안 돼?" 안 된다. 무한 재귀, 비밀번호 노출, LazyInitializationException... Entity를 절대 API 응답으로 쓰면 안 되는 5가지 이유와, 올바른 DTO 패턴을 정리한다.

&nbsp;

&nbsp;

---

JPA, 더티체킹, Dirty Checking, 영속성 컨텍스트, 스냅샷, flush, merge, detach, 벌크연산, Spring Data JPA, Hibernate, save, persist, 변경감지
