# Entity를 API 응답으로 직접 반환하면 생기는 일

&nbsp;

Controller에서 Entity를 그대로 반환한 적 있는가?

&nbsp;

```java
@GetMapping("/members/{id}")
public Member getMember(@PathVariable Long id) {
    return memberRepository.findById(id).orElseThrow();
}
```

&nbsp;

"잘 되는데? DTO 만들기 귀찮은데 이대로 쓰면 안 돼?"

&nbsp;

**안 된다. 5가지 이유로 반드시 터진다.**

&nbsp;

&nbsp;

---

&nbsp;

# 1. 택배 상자로 이해하기

&nbsp;

```
Entity = 회사 내부 서류 원본
DTO = 외부 전달용 복사본

Entity 직접 반환 = 내부 서류 원본을 고객에게 보내는 것
  → 주민번호가 적혀 있음 (민감 정보 노출)
  → 서류 양식 바꾸면 고객이 읽을 수 없음 (API 스펙 깨짐)
  → 서류에 다른 서류가 첨부돼 있고, 그 서류에 또 첨부가... (무한 재귀)

DTO 반환 = 필요한 내용만 복사해서 전달
  → 이름, 이메일만 적어서 보냄
  → 원본 양식이 바뀌어도 복사본은 그대로
  → 안전
```

&nbsp;

&nbsp;

---

&nbsp;

# 2. 문제 1 — 민감 정보 노출

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private String email;
    private String password;      // ← 비밀번호!
    private String socialNumber;  // ← 주민번호!
    private int failCount;        // ← 로그인 실패 횟수
}
```

&nbsp;

```java
@GetMapping("/members/{id}")
public Member getMember(@PathVariable Long id) {
    return memberRepository.findById(id).orElseThrow();
}
```

&nbsp;

```json
// API 응답
{
  "id": 1,
  "name": "홍길동",
  "email": "hong@example.com",
  "password": "$2a$10$xyz...",     // 해시값이라도 노출됨
  "socialNumber": "900101-1234567", // 주민번호 그대로 노출!
  "failCount": 3                    // 내부 정보 노출
}
```

&nbsp;

"@JsonIgnore 붙이면 되잖아?"

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private String email;

    @JsonIgnore
    private String password;

    @JsonIgnore
    private String socialNumber;

    @JsonIgnore
    private int failCount;
}
```

&nbsp;

되긴 된다. 근데 **Entity가 Jackson(JSON 직렬화)에 종속된다.** 새 필드를 추가할 때마다 "이거 노출해도 되나?" 확인해야 한다. 까먹으면 민감 정보가 그대로 나간다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 문제 2 — 양방향 관계 무한 재귀

&nbsp;

이게 가장 많이 터지는 문제다.

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
}

@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "team")
    private List<Member> members;
}
```

&nbsp;

```java
@GetMapping("/members/{id}")
public Member getMember(@PathVariable Long id) {
    return memberRepository.findById(id).orElseThrow();
}
```

&nbsp;

```
Jackson이 Member를 JSON으로 변환할 때:

Member → team 필드 직렬화
  → Team → members 필드 직렬화
    → Member → team 필드 직렬화
      → Team → members 필드 직렬화
        → Member → team 필드 직렬화
          → ...
            → StackOverflowError!
```

&nbsp;

```
에러 로그:
com.fasterxml.jackson.databind.JsonMappingException:
  Infinite recursion (StackOverflowError)
    at com.fasterxml.jackson.databind.ser.BeanSerializer.serialize(...)
    at com.fasterxml.jackson.databind.ser.BeanPropertyWriter.serializeAsField(...)
    ...
```

&nbsp;

"@JsonIgnore나 @JsonManagedReference 붙이면 되잖아?"

&nbsp;

```java
@Entity
public class Team {
    @JsonManagedReference
    @OneToMany(mappedBy = "team")
    private List<Member> members;
}

@Entity
public class Member {
    @JsonBackReference
    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
}
```

&nbsp;

해결은 되지만, **Entity에 JSON 관련 어노테이션이 쌓인다.** Entity는 DB 매핑용이지, API 응답용이 아니다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 문제 3 — Entity 변경 = API 스펙 변경

&nbsp;

```java
// 처음 설계
@Entity
public class Member {
    private String name;     // API 응답: { "name": "홍길동" }
}

// 3개월 후 요구사항 변경: 이름을 성/이름으로 분리
@Entity
public class Member {
    private String firstName;  // name 삭제!
    private String lastName;   // 새 필드!
}
```

&nbsp;

```json
// 기존 API 응답 (클라이언트가 쓰고 있던 것)
{ "name": "홍길동" }

// 변경 후 API 응답 (클라이언트 깨짐!)
{ "firstName": "길동", "lastName": "홍" }
```

&nbsp;

Entity를 직접 반환하면 **DB 스키마 변경이 곧 API 스펙 변경**이 된다.

프론트엔드, 모바일 앱, 외부 연동 — 전부 깨진다.

&nbsp;

DTO를 쓰면 Entity가 바뀌어도 API 스펙은 유지할 수 있다.

&nbsp;

```java
// Entity는 바뀌었지만
@Entity
public class Member {
    private String firstName;
    private String lastName;
}

// DTO는 기존 스펙 유지
public class MemberResponse {
    private String name;  // firstName + lastName 합쳐서 반환

    public MemberResponse(Member member) {
        this.name = member.getLastName() + member.getFirstName();
    }
}
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. 문제 4 — @JsonIgnore 떡칠

&nbsp;

Entity에 @JsonIgnore가 3~4개 붙기 시작하면 이미 잘못된 거다.

&nbsp;

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private String email;

    @JsonIgnore
    private String password;

    @JsonIgnore
    private String socialNumber;

    @JsonIgnore
    private int failCount;

    @JsonIgnore
    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;

    @JsonIgnore
    @OneToMany(mappedBy = "member")
    private List<Order> orders;

    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDateTime createdAt;
}
```

&nbsp;

```
문제점:
  1. Entity가 JSON 직렬화에 종속됨
  2. Entity의 역할이 불명확해짐 (DB 매핑? API 응답?)
  3. API마다 숨겨야 할 필드가 다를 수 있음
     → 목록 API: name, email만
     → 상세 API: name, email, createdAt
     → 관리자 API: 전부 다
  4. @JsonIgnore 하나로는 이런 케이스별 대응 불가
```

&nbsp;

&nbsp;

---

&nbsp;

# 6. 문제 5 — LazyInitializationException

&nbsp;

```java
@GetMapping("/members/{id}")
public Member getMember(@PathVariable Long id) {
    Member member = memberService.findById(id);
    return member;  // Controller에서 반환 → JSON 변환 시도
}
```

&nbsp;

```java
@Service
@Transactional(readOnly = true)
public class MemberService {
    public Member findById(Long id) {
        return memberRepository.findById(id).orElseThrow();
        // team은 LAZY → 프록시 상태로 반환
    }
    // 여기서 트랜잭션 종료 → 영속성 컨텍스트 닫힘
}
```

&nbsp;

```
Controller에서 JSON 변환 시:
  Jackson이 member.getTeam()에 접근
  → LAZY 프록시 초기화 시도
  → 영속성 컨텍스트 이미 닫힘
  → LazyInitializationException!

org.hibernate.LazyInitializationException:
  could not initialize proxy - no Session
```

&nbsp;

"OSIV(Open Session In View) 켜면 되잖아?"

&nbsp;

```yaml
spring:
  jpa:
    open-in-view: true  # 기본값이 true
```

&nbsp;

OSIV를 켜면 Controller까지 영속성 컨텍스트가 살아있어서 LazyInitializationException은 안 터진다. 근데 그러면 **Controller에서 LAZY 로딩이 발생** → 예측 불가능한 쿼리가 나감 → 성능 문제.

&nbsp;

```
OSIV = true (기본값):
  ✅ LazyInitializationException 안 터짐
  ❌ Controller에서 쿼리 나감 → 성능 예측 불가
  ❌ DB 커넥션을 오래 잡고 있음

OSIV = false (권장):
  ✅ 트랜잭션 범위 명확
  ✅ DB 커넥션 빨리 반환
  ❌ LazyInitializationException 주의 필요
  → 해결: DTO로 변환해서 반환
```

&nbsp;

&nbsp;

---

&nbsp;

# 7. 해결 — Entity를 DTO로 변환

&nbsp;

## 방법 1: 수동 변환

&nbsp;

```java
// Response DTO
public class MemberResponse {
    private Long id;
    private String name;
    private String email;
    private String teamName;

    public MemberResponse(Member member) {
        this.id = member.getId();
        this.name = member.getName();
        this.email = member.getEmail();
        this.teamName = member.getTeam().getName();
    }
}
```

&nbsp;

```java
// Controller
@GetMapping("/members/{id}")
public MemberResponse getMember(@PathVariable Long id) {
    Member member = memberService.findById(id);
    return new MemberResponse(member);
}

// Service — 트랜잭션 안에서 DTO 변환 (LAZY 로딩 안전)
@Service
@Transactional(readOnly = true)
public class MemberService {
    public MemberResponse findById(Long id) {
        Member member = memberRepository.findById(id).orElseThrow();
        return new MemberResponse(member);  // 여기서 team.getName() 호출 → 안전
    }
}
```

&nbsp;

## 방법 2: Java Record (Java 16+)

&nbsp;

```java
public record MemberResponse(
    Long id,
    String name,
    String email,
    String teamName
) {
    public static MemberResponse from(Member member) {
        return new MemberResponse(
            member.getId(),
            member.getName(),
            member.getEmail(),
            member.getTeam().getName()
        );
    }
}
```

&nbsp;

```java
// 사용
MemberResponse response = MemberResponse.from(member);
```

&nbsp;

## 방법 3: MapStruct (자동 매핑)

&nbsp;

```java
@Mapper(componentModel = "spring")
public interface MemberMapper {

    @Mapping(source = "team.name", target = "teamName")
    MemberResponse toResponse(Member member);
}
```

&nbsp;

```java
// 사용
@Service
@RequiredArgsConstructor
public class MemberService {
    private final MemberMapper memberMapper;

    public MemberResponse findById(Long id) {
        Member member = memberRepository.findById(id).orElseThrow();
        return memberMapper.toResponse(member);
    }
}
```

&nbsp;

```
수동 변환:   간단, 완전 제어, 매핑 코드 많아짐
Record:      간결, 불변, Java 16+ 필요
MapStruct:   자동 매핑, 컴파일 타임 검증, 설정 필요
```

&nbsp;

&nbsp;

---

&nbsp;

# 8. 요청도 DTO — Entity를 RequestBody로 쓰지 말 것

&nbsp;

```java
// 나쁜 예: Entity를 Request로 사용
@PostMapping("/members")
public void create(@RequestBody Member member) {
    memberRepository.save(member);
    // 클라이언트가 id, role, status 같은 필드도 보내면?
    // → 의도치 않은 값이 저장될 수 있음
}
```

&nbsp;

```java
// 좋은 예: Request DTO 사용
public class CreateMemberRequest {
    @NotBlank
    private String name;

    @Email
    private String email;

    @Size(min = 8)
    private String password;

    public Member toEntity() {
        return Member.builder()
            .name(name)
            .email(email)
            .password(passwordEncoder.encode(password))
            .role(Role.USER)           // 서버에서 설정
            .status(Status.ACTIVE)     // 서버에서 설정
            .build();
    }
}
```

&nbsp;

```java
@PostMapping("/members")
public MemberResponse create(@RequestBody @Valid CreateMemberRequest request) {
    Member member = request.toEntity();
    memberRepository.save(member);
    return new MemberResponse(member);
}
```

&nbsp;

&nbsp;

---

&nbsp;

# 9. 계층별 정리

&nbsp;

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│ Controller  │ ←→  │   Service    │ ←→  │ Repository   │
│             │     │              │     │              │
│ Request DTO │     │   Entity     │     │   Entity     │
│ Response DTO│     │              │     │ (Query DTO)  │
└─────────────┘     └──────────────┘     └──────────────┘

Controller:
  - Request DTO 받기 (검증 포함)
  - Response DTO 반환

Service:
  - Entity로 비즈니스 로직 처리
  - DTO ↔ Entity 변환

Repository:
  - Entity 조회/저장
  - 성능 필요 시 Query DTO 직접 조회
```

&nbsp;

&nbsp;

---

&nbsp;

# 10. "Entity에 @Setter 열면 안 되는 이유"

&nbsp;

같은 맥락이다.

&nbsp;

```java
// 나쁜 예
@Entity
@Getter @Setter  // Setter 전체 오픈
public class Member {
    private String name;
    private String email;
    private Role role;
    private Status status;
}

// 아무 데서나 이렇게 할 수 있다
member.setRole(Role.ADMIN);     // 누가 어디서든 관리자로 변경 가능
member.setStatus(Status.DELETED); // 누가 어디서든 삭제 가능
```

&nbsp;

```java
// 좋은 예
@Entity
@Getter  // Setter 없음
public class Member {
    private String name;
    private String email;
    private Role role;
    private Status status;

    // 의미 있는 메서드로 상태 변경
    public void changeName(String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("이름은 필수입니다");
        }
        this.name = name;
    }

    public void promote() {
        this.role = Role.ADMIN;
    }

    public void deactivate() {
        this.status = Status.INACTIVE;
    }
}
```

&nbsp;

```
@Setter 전체 오픈:
  ❌ 누가, 어디서, 왜 바꿨는지 추적 불가
  ❌ 유효성 검증 없이 아무 값이나 세팅 가능
  ❌ Entity의 일관성 보장 불가

의미 있는 메서드:
  ✅ 변경 의도가 명확 (changeName, promote, deactivate)
  ✅ 유효성 검증 가능
  ✅ 비즈니스 로직 캡슐화
```

&nbsp;

&nbsp;

---

&nbsp;

# 11. 정리

&nbsp;

```
Entity 직접 반환의 문제 5가지:
  1. 민감 정보 노출 (password, socialNumber)
  2. 양방향 관계 → 무한 재귀 (StackOverflow)
  3. Entity 변경 → API 스펙 변경 (클라이언트 깨짐)
  4. @JsonIgnore 떡칠 → Entity가 API에 종속
  5. LazyInitializationException

해결:
  ✅ Entity → Response DTO 변환
  ✅ RequestBody → Request DTO 사용
  ✅ Entity에 @Setter 대신 의미 있는 메서드
  ✅ Service 레이어에서 DTO 변환 (트랜잭션 안에서)
```

&nbsp;

## 핵심 한 줄

&nbsp;

**Entity는 DB와의 계약이고, DTO는 클라이언트와의 계약이다. 두 계약을 하나로 합치면 양쪽 다 깨진다.**

&nbsp;

&nbsp;

---

&nbsp;

다음 편 예고: **실무에서 자주 터지는 Spring 버그 모음 — 면접에도 나오는 것들**

순환 참조, equals/hashCode 미구현, LocalDateTime 직렬화, 동시성 문제, 스프링 프로필 실수... 실무에서 한 번씩은 꼭 만나는 버그들을 정리한다. 면접에서도 단골 주제다.

&nbsp;

&nbsp;

---

Entity, DTO, API 응답, Jackson, JsonIgnore, 무한재귀, StackOverflow, LazyInitializationException, OSIV, MapStruct, Record, Setter, 캡슐화, Spring, JPA
