# [Java vs Node.js 1편] Java에서는 @Service, Node.js에서는? — DI 없이 레이어를 나누는 법

&nbsp;

Java/Spring 개발자가 Node.js 프로젝트를 처음 열면 가장 먼저 느끼는 위화감이 있다.

&nbsp;

**"@Controller, @Service, @Repository가 없는데... 레이어를 어떻게 나누지?"**

&nbsp;

Spring에서는 DI 컨테이너가 모든 걸 관리해준다.

Node.js에서는? **직접 import하고, 직접 연결한다.**

&nbsp;

이게 불편한 건지, 자유로운 건지. 둘 다 써본 입장에서 솔직하게 비교한다.

&nbsp;

&nbsp;

---

&nbsp;

## 1. Java의 레이어 구조 — 어노테이션이 다 해준다

&nbsp;

Spring Boot의 전형적인 3계층 구조다.

&nbsp;

```java
// Controller
@RestController
@RequestMapping("/api/members")
public class MemberController {

    private final MemberService memberService;

    // 생성자 주입 — Spring이 알아서 MemberService 인스턴스를 넣어준다
    public MemberController(MemberService memberService) {
        this.memberService = memberService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<MemberDto> getMember(@PathVariable Long id) {
        return ResponseEntity.ok(memberService.findById(id));
    }
}
```

&nbsp;

```java
// Service
@Service
public class MemberService {

    private final MemberRepository memberRepository;

    public MemberService(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }

    public MemberDto findById(Long id) {
        Member member = memberRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("Member not found"));
        return MemberDto.from(member);
    }
}
```

&nbsp;

```java
// Repository
@Repository
public interface MemberRepository extends JpaRepository<Member, Long> {
    Optional<Member> findByCardNumber(String cardNumber);
}
```

&nbsp;

**핵심:**
- `@Controller`, `@Service`, `@Repository` — 어노테이션만 붙이면 Spring이 빈으로 등록
- 생성자에 타입만 맞으면 자동 주입
- 개발자는 "누가 누구를 만들고 연결하는지" 신경 안 써도 됨

&nbsp;

&nbsp;

---

&nbsp;

## 2. Node.js의 레이어 구조 — import가 곧 DI다

&nbsp;

Node.js에는 Spring 같은 DI 컨테이너가 없다. 대신 **ES Module의 import/export가 의존성 연결을 대신한다.**

&nbsp;

```typescript
// router (= Controller)
// routes/member.ts
import { Router } from 'express';
import { memberService } from '../services/member.service';

const router = Router();

router.get('/:id', async (req, res) => {
  const member = await memberService.findById(req.params.id);
  res.json(member);
});

export default router;
```

&nbsp;

```typescript
// service
// services/member.service.ts
import { memberRepository } from '../repositories/member.repository';

class MemberService {
  async findById(id: string) {
    const member = await memberRepository.findById(id);
    if (!member) throw new Error('Member not found');
    return member;
  }
}

export const memberService = new MemberService();
```

&nbsp;

```typescript
// repository
// repositories/member.repository.ts
import { dataSource } from '../database';
import { Member } from '../entities/member';

class MemberRepository {
  private repo = dataSource.getRepository(Member);

  async findById(id: string) {
    return this.repo.findOneBy({ id });
  }

  async findByCardNumber(cardNumber: string) {
    return this.repo.findOneBy({ cardNumber });
  }
}

export const memberRepository = new MemberRepository();
```

&nbsp;

**핵심:**
- DI 컨테이너 없음. `import`로 직접 가져옴
- 싱글톤이 필요하면 `export const instance = new Class()`로 해결
- Node.js의 모듈 시스템이 자연스러운 싱글톤을 보장 (같은 모듈은 한 번만 실행)

&nbsp;

&nbsp;

---

&nbsp;

## 3. DI가 없어서 불편한 점 — 테스트에서 드러난다

&nbsp;

**Java에서 테스트:**

```java
@ExtendWith(MockitoExtension.class)
class MemberServiceTest {

    @Mock
    private MemberRepository memberRepository;

    @InjectMocks
    private MemberService memberService;

    @Test
    void findById_success() {
        // given
        Member member = new Member(1L, "홍길동");
        when(memberRepository.findById(1L)).thenReturn(Optional.of(member));

        // when
        MemberDto result = memberService.findById(1L);

        // then
        assertEquals("홍길동", result.getName());
    }
}
```

&nbsp;

생성자 주입 덕분에 `@Mock`으로 가짜 객체를 손쉽게 넣을 수 있다.

&nbsp;

**Node.js에서 테스트 — 문제 발생:**

```typescript
// memberService 안에서 memberRepository를 import로 직접 가져왔기 때문에
// 테스트에서 바꿔치기가 어렵다

import { memberService } from '../services/member.service';
// memberRepository가 이미 연결된 상태!
```

&nbsp;

**해결 방법 1: jest.mock()**

```typescript
jest.mock('../repositories/member.repository', () => ({
  memberRepository: {
    findById: jest.fn(),
  },
}));

import { memberService } from '../services/member.service';
import { memberRepository } from '../repositories/member.repository';

test('findById 성공', async () => {
  (memberRepository.findById as jest.Mock).mockResolvedValue({
    id: '1', name: '홍길동'
  });

  const result = await memberService.findById('1');
  expect(result.name).toBe('홍길동');
});
```

&nbsp;

**해결 방법 2: 생성자 주입 패턴 (Java 스타일)**

```typescript
class MemberService {
  constructor(private memberRepo: MemberRepository) {}

  async findById(id: string) {
    return this.memberRepo.findById(id);
  }
}

// 프로덕션
export const memberService = new MemberService(memberRepository);

// 테스트
const mockRepo = { findById: jest.fn() };
const testService = new MemberService(mockRepo as any);
```

&nbsp;

두 번째 방법이 Java 스타일에 가깝다. 하지만 Node.js 생태계에서는 첫 번째(jest.mock)가 더 흔하다.

&nbsp;

&nbsp;

---

&nbsp;

## 4. DI를 흉내내는 방법들

&nbsp;

Node.js에도 DI 라이브러리가 있다. 다만 **주류가 아니다.**

&nbsp;

### 4-1. tsyringe (Microsoft)

```typescript
import { injectable, inject, container } from 'tsyringe';

@injectable()
class MemberService {
  constructor(
    @inject('MemberRepository') private memberRepo: MemberRepository
  ) {}
}

container.register('MemberRepository', { useClass: MemberRepository });
const service = container.resolve(MemberService);
```

&nbsp;

### 4-2. InversifyJS

```typescript
import { injectable, inject, Container } from 'inversify';

@injectable()
class MemberService {
  constructor(
    @inject(TYPES.MemberRepository) private memberRepo: MemberRepository
  ) {}
}

const container = new Container();
container.bind(TYPES.MemberRepository).to(MemberRepository);
container.bind(TYPES.MemberService).to(MemberService);
```

&nbsp;

### 4-3. NestJS (프레임워크 레벨)

```typescript
// NestJS는 Spring과 거의 동일한 DI를 제공한다
@Injectable()
export class MemberService {
  constructor(private memberRepo: MemberRepository) {}
}

@Controller('members')
export class MemberController {
  constructor(private memberService: MemberService) {}

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.memberService.findById(id);
  }
}
```

&nbsp;

NestJS가 Spring과 가장 비슷하다. 하지만 **Express + 수동 레이어가 Node.js의 주류**다.

&nbsp;

&nbsp;

---

&nbsp;

## 5. 현실 — 대부분 직접 import로 충분하다

&nbsp;

| 상황 | 추천 |
|:---|:---|
| 소규모~중규모 프로젝트 | 직접 import (DI 불필요) |
| 대규모 + 복잡한 의존성 | NestJS 또는 tsyringe 고려 |
| 테스트 격리가 중요 | 생성자 주입 패턴 or jest.mock |
| Spring 출신 팀 | NestJS가 적응 비용 최소 |

&nbsp;

솔직히 말하면, DI 프레임워크 없이도 **Node.js 프로젝트의 90%는 잘 돌아간다.** 모듈 시스템 자체가 싱글톤을 보장하고, 의존성 그래프가 Java만큼 복잡해지는 경우가 드물기 때문이다.

&nbsp;

DI가 빛나는 건 "인터페이스를 바꿔 끼워야 할 때"인데, Node.js에서는 그런 상황 자체가 덜 생긴다.

&nbsp;

&nbsp;

---

&nbsp;

## 6. DTO — 클래스 vs 인터페이스 + 런타임 검증

&nbsp;

**Java: 클래스가 곧 DTO**

```java
public class CreateMemberDto {
    @NotBlank
    private String name;

    @Email
    private String email;

    @Size(min = 8, max = 20)
    private String password;

    // getter, setter (또는 record)
}
```

&nbsp;

Java에서는 DTO 클래스에 검증 어노테이션을 붙이면, 컨트롤러에서 `@Valid`로 자동 검증된다.

&nbsp;

**Node.js: 인터페이스 + Zod**

```typescript
// 타입 정의 (컴파일 타임)
interface CreateMemberDto {
  name: string;
  email: string;
  password: string;
}

// 런타임 검증 (Zod)
import { z } from 'zod';

const createMemberSchema = z.object({
  name: z.string().min(1, '이름은 필수입니다'),
  email: z.string().email('올바른 이메일 형식이 아닙니다'),
  password: z.string().min(8).max(20),
});

// 사용
const validated = createMemberSchema.parse(req.body);
// 실패 시 ZodError throw
```

&nbsp;

**차이:**
- Java: 클래스 하나가 타입 + 검증을 동시에 담당
- Node.js: 인터페이스(타입)와 Zod 스키마(검증)가 분리됨
- Zod는 스키마에서 타입을 추론할 수 있어서, 한 곳에서 관리 가능

```typescript
// Zod에서 타입 자동 추론 — 인터페이스 따로 안 만들어도 됨
type CreateMemberDto = z.infer<typeof createMemberSchema>;
```

&nbsp;

&nbsp;

---

&nbsp;

## 7. 패키지 구조 비교

&nbsp;

**Java: 도메인 기반**

```
src/main/java/com/example/
├── member/
│   ├── MemberController.java
│   ├── MemberService.java
│   ├── MemberRepository.java
│   ├── Member.java
│   └── MemberDto.java
├── storage/
│   ├── StorageController.java
│   ├── StorageService.java
│   └── ...
└── common/
    ├── exception/
    └── config/
```

&nbsp;

**Node.js: 기능(역할) 기반**

```
src/
├── routes/
│   ├── member.ts
│   └── storage.ts
├── services/
│   ├── member.service.ts
│   └── storage.service.ts
├── repositories/
│   ├── member.repository.ts
│   └── storage.repository.ts
├── entities/
│   ├── member.ts
│   └── storage.ts
└── middleware/
    ├── auth.ts
    └── error-handler.ts
```

&nbsp;

| 구조 | 장점 | 단점 |
|:---|:---|:---|
| 도메인 기반 (Java) | 도메인 단위 응집도 높음 | 파일이 많아지면 폴더도 많아짐 |
| 기능 기반 (Node.js) | 같은 역할끼리 모여서 파악 쉬움 | 도메인 간 관계 파악이 어려울 수 있음 |

&nbsp;

물론 Node.js에서도 도메인 기반 구조를 쓸 수 있고, Java에서도 기능 기반을 쓸 수 있다. 다만 **커뮤니티의 관성**이 이렇다.

&nbsp;

Next.js App Router는 파일 시스템 라우팅이라 또 다른 패턴이다:

```
src/app/
├── api/
│   ├── members/
│   │   └── route.ts          // GET, POST
│   │   └── [id]/
│   │       └── route.ts      // GET, PUT, DELETE
│   └── storage/
│       └── route.ts
└── lib/
    ├── services/
    └── database.ts
```

&nbsp;

&nbsp;

---

&nbsp;

## 8. AI 관점 — 어떤 구조가 AI한테 더 친절한가

&nbsp;

AI에게 코드를 작성시키거나 분석시킬 때, 프로젝트 구조는 중요하다.

&nbsp;

**Java/Spring:**
- 어노테이션이 역할을 명시 → AI가 "이건 서비스, 이건 컨트롤러" 즉시 파악
- 정형화된 패턴 → Spring Boot 프로젝트는 거의 다 비슷해서 AI가 잘 맞춤
- 단점: 매직(AOP, 프록시)이 많아서 "실제로 어떻게 동작하는지"는 AI도 추적하기 어려움

&nbsp;

**Node.js:**
- import 체인을 따라가면 의존성이 명시적으로 보임
- 파일명에 `.service.ts`, `.repository.ts` 규칙만 지키면 AI가 잘 따라옴
- 단점: 구조 자유도가 높아서, 팀마다 다르면 AI가 혼란스러워함

&nbsp;

```
// AI한테 친절한 Node.js 구조 규칙
1. 파일명: member.service.ts, member.repository.ts (역할 명시)
2. export: 클래스 + 인스턴스 같이 export
3. index.ts: 각 폴더에 barrel file로 진입점 명시
4. 순환 참조 금지: A → B → A 절대 불가
```

&nbsp;

결론: **Java는 프레임워크 규칙 덕에 자동으로 정리되고, Node.js는 개발자가 규칙을 만들어야 한다.** AI 활용 관점에서도, 일관된 규칙이 있으면 둘 다 비슷하게 잘 동작한다.

&nbsp;

&nbsp;

---

&nbsp;

## 정리

&nbsp;

| 항목 | Java/Spring | Node.js |
|:---|:---|:---|
| 레이어 분리 | 어노테이션 (@Controller, @Service) | 파일/폴더 규칙 + import |
| 의존성 주입 | Spring DI 컨테이너 (자동) | import (수동) 또는 NestJS |
| 싱글톤 | @Component가 기본 싱글톤 | 모듈 시스템이 자연 싱글톤 |
| 테스트 Mock | @Mock + @InjectMocks | jest.mock() 또는 생성자 주입 |
| DTO | 클래스 + Bean Validation | 인터페이스 + Zod |
| 패키지 구조 | 도메인 기반 (관성) | 기능 기반 (관성) |

&nbsp;

Spring의 DI는 강력하지만, Node.js에서 반드시 필요한 건 아니다.

**중요한 건 레이어를 나누는 원칙이지, 프레임워크가 해주느냐 직접 하느냐의 차이일 뿐이다.**

&nbsp;

다음 편에서는 **AOP가 없는 Node.js에서 인증, 로깅, 에러 처리 같은 횡단 관심사를 어떻게 처리하는지** 비교한다.

&nbsp;

&nbsp;

---

Java, Node.js, Spring, Express, DI, 의존성주입, 레이어구조, 아키텍처, TypeScript, NestJS, tsyringe, Zod, 패키지구조, AI코딩, 바이브코딩