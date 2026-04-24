# [Java vs Node.js 3편] @Valid 없이 어떻게 검증하지? — Zod, 커스텀 미들웨어, 그리고 보안

&nbsp;

Java/Spring에서는 `@Valid`를 붙이면 끝이다.

&nbsp;

```java
@PostMapping("/members")
public ResponseEntity<Void> create(@Valid @RequestBody CreateMemberDto dto) {
    // 여기 도착했으면 dto는 이미 검증 완료
}
```

&nbsp;

Node.js에서는? **스키마를 정의하고, 미들웨어를 걸고, 에러를 잡아야 한다.**

한 줄이면 될 걸 세 단계로 나눠야 한다. 하지만 그만큼 **무엇이 어떻게 검증되는지** 명확하게 보인다.

&nbsp;

&nbsp;

---

&nbsp;

## 1. Java의 검증 — Bean Validation이 다 해준다

&nbsp;

```java
public class CreateMemberDto {

    @NotBlank(message = "이름은 필수입니다")
    private String name;

    @NotBlank
    @Email(message = "올바른 이메일 형식이 아닙니다")
    private String email;

    @NotBlank
    @Size(min = 8, max = 20, message = "비밀번호는 8~20자입니다")
    private String password;

    @NotNull
    @Pattern(regexp = "^\\d{8}$", message = "생년월일은 8자리 숫자입니다")
    private String birthDate;

    @NotNull
    @Min(1) @Max(150)
    private Integer age;
}
```

&nbsp;

```java
@PostMapping("/members")
public ResponseEntity<MemberResponse> create(
    @Valid @RequestBody CreateMemberDto dto
) {
    // @Valid 덕분에 여기 도착하면 모든 필드가 검증 완료
    Member member = memberService.create(dto);
    return ResponseEntity.status(201).body(MemberResponse.from(member));
}
```

&nbsp;

검증 실패 시 Spring이 자동으로 `MethodArgumentNotValidException`을 던지고, `@ControllerAdvice`에서 잡으면 된다.

&nbsp;

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<Map<String, String>> handleValidation(
    MethodArgumentNotValidException e
) {
    Map<String, String> errors = new HashMap<>();
    e.getBindingResult().getFieldErrors().forEach(error ->
        errors.put(error.getField(), error.getDefaultMessage())
    );
    return ResponseEntity.badRequest().body(errors);
}
```

&nbsp;

**Java의 흐름:** DTO 클래스에 어노테이션 → 컨트롤러에 `@Valid` → 전역 에러 핸들러. 끝.

&nbsp;

&nbsp;

---

&nbsp;

## 2. Node.js의 검증 — Zod + 미들웨어

&nbsp;

### 2-1. Zod 스키마 정의

```typescript
import { z } from 'zod';

const createMemberSchema = z.object({
  name: z.string().min(1, '이름은 필수입니다'),
  email: z.string().email('올바른 이메일 형식이 아닙니다'),
  password: z.string().min(8, '비밀번호는 8자 이상입니다').max(20, '비밀번호는 20자 이하입니다'),
  birthDate: z.string().regex(/^\d{8}$/, '생년월일은 8자리 숫자입니다'),
  age: z.number().int().min(1).max(150),
});

// 타입 자동 추론 — 인터페이스 따로 안 만들어도 됨
type CreateMemberDto = z.infer<typeof createMemberSchema>;
```

&nbsp;

### 2-2. 검증 미들웨어

```typescript
import { ZodSchema, ZodError } from 'zod';

function validate(schema: ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body);
      next();
    } catch (err) {
      if (err instanceof ZodError) {
        const errors: Record<string, string> = {};
        err.errors.forEach((e) => {
          const field = e.path.join('.');
          errors[field] = e.message;
        });
        return res.status(400).json({ errors });
      }
      next(err);
    }
  };
}
```

&nbsp;

### 2-3. 라우트에 적용

```typescript
router.post('/members',
  validate(createMemberSchema),
  async (req, res) => {
    // 여기 도착하면 req.body는 검증 + 타입 안전
    const member = await memberService.create(req.body);
    res.status(201).json(member);
  }
);
```

&nbsp;

### 2-4. Next.js App Router에서

```typescript
// app/api/members/route.ts
import { NextRequest, NextResponse } from 'next/server';

const createMemberSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  password: z.string().min(8).max(20),
});

export async function POST(request: NextRequest) {
  const body = await request.json();

  const result = createMemberSchema.safeParse(body);
  if (!result.success) {
    return NextResponse.json(
      { errors: result.error.flatten().fieldErrors },
      { status: 400 }
    );
  }

  // result.data는 타입 안전
  const member = await memberService.create(result.data);
  return NextResponse.json(member, { status: 201 });
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. 같은 API, 나란히 비교

&nbsp;

회원가입 API 검증을 양쪽으로 구현하면:

&nbsp;

**Java (총 코드: DTO 클래스 1개 + @Valid 1개)**

```java
// 1. DTO — 이게 전부
public record CreateMemberDto(
    @NotBlank String name,
    @Email String email,
    @Size(min = 8, max = 20) String password
) {}

// 2. 컨트롤러 — @Valid만 추가
@PostMapping("/members")
public ResponseEntity<Void> create(@Valid @RequestBody CreateMemberDto dto) {
    memberService.create(dto);
    return ResponseEntity.status(201).build();
}
```

&nbsp;

**Node.js (총 코드: 스키마 + safeParse)**

```typescript
// 1. 스키마
const createMemberSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  password: z.string().min(8).max(20),
});

// 2. API Route
export async function POST(request: NextRequest) {
  const body = await request.json();
  const result = createMemberSchema.safeParse(body);

  if (!result.success) {
    return NextResponse.json(
      { errors: result.error.flatten().fieldErrors },
      { status: 400 }
    );
  }

  await memberService.create(result.data);
  return NextResponse.json(null, { status: 201 });
}
```

&nbsp;

코드 양은 비슷하다. 차이는:
- Java: 검증 로직이 DTO에 **선언적으로** 붙어있음
- Node.js: 검증 로직이 API 핸들러에서 **명시적으로** 실행됨

&nbsp;

&nbsp;

---

&nbsp;

## 4. Zod vs Joi vs class-validator

&nbsp;

Node.js 검증 라이브러리 비교:

&nbsp;

```typescript
// Zod — 타입 추론이 최강
const schema = z.object({
  name: z.string().min(1),
  age: z.number().int().positive(),
});
type Dto = z.infer<typeof schema>; // 자동 타입

// Joi — 가장 오래됨, 타입 추론 약함
const schema = Joi.object({
  name: Joi.string().required(),
  age: Joi.number().integer().positive().required(),
});
// 타입은 수동으로 interface 정의해야 함

// class-validator — Java 스타일 데코레이터
class CreateMemberDto {
  @IsNotEmpty()
  name: string;

  @IsInt()
  @Min(1)
  age: number;
}
```

&nbsp;

| 라이브러리 | 타입 추론 | Java 스타일 | 번들 크기 | 추천 |
|:---|:---|:---|:---|:---|
| Zod | 자동 (z.infer) | X | 13KB | TypeScript 프로젝트 |
| Joi | 수동 | X | 70KB+ | JavaScript 레거시 |
| class-validator | 수동 | O (데코레이터) | 40KB+ | NestJS 프로젝트 |

&nbsp;

**2024년 기준 Zod가 사실상 표준이다.** TypeScript와의 통합이 압도적이다.

&nbsp;

&nbsp;

---

&nbsp;

## 5. 보안 — Spring Security vs 직접 구현

&nbsp;

### 5-1. JWT 인증

&nbsp;

**Java: Spring Security 필터 체인**

```java
@Component
public class JwtFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws IOException, ServletException {
        String token = extractToken(request);
        if (token != null && jwtProvider.validate(token)) {
            Authentication auth = jwtProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(request, response);
    }
}
```

&nbsp;

**Node.js: 미들웨어 직접 구현**

```typescript
import jwt from 'jsonwebtoken';

interface JwtPayload {
  userId: string;
  role: string;
}

function jwtAuth(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Token required' });
  }

  try {
    const token = header.slice(7);
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload;
    req.user = payload;
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
}
```

&nbsp;

구조는 비슷하다. Spring Security가 제공하는 건 **체인 관리, 세션 통합, CSRF 방어 등의 부가 기능**이다. Node.js에서는 이것들을 하나하나 붙여야 한다.

&nbsp;

### 5-2. 권한 체크

&nbsp;

**Java:**

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/members/{id}")
public ResponseEntity<Void> deleteMember(@PathVariable Long id) {
    memberService.delete(id);
    return ResponseEntity.noContent().build();
}
```

&nbsp;

**Node.js:**

```typescript
function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

router.delete('/members/:id',
  jwtAuth,
  requireRole('ADMIN'),
  async (req, res) => {
    await memberService.delete(req.params.id);
    res.status(204).send();
  }
);
```

&nbsp;

### 5-3. CORS

&nbsp;

**Java:**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowCredentials(true);
    }
}
```

&nbsp;

**Node.js:**

```typescript
import cors from 'cors';

app.use(cors({
  origin: 'https://example.com',
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true,
}));
```

&nbsp;

CORS만큼은 Node.js가 더 간결하다.

&nbsp;

&nbsp;

---

&nbsp;

## 6. SQL Injection — ORM이 기본 방어

&nbsp;

**Java (JPA):**

```java
// 안전 — JPA가 파라미터 바인딩
memberRepository.findByName(name);

// 위험 — native query에 문자열 직접 삽입
@Query(value = "SELECT * FROM member WHERE name = '" + name + "'", nativeQuery = true)
// 절대 이렇게 하지 말 것
```

&nbsp;

**Node.js (TypeORM):**

```typescript
// 안전 — TypeORM 파라미터 바인딩
memberRepository.findOneBy({ name });

// 안전 — QueryBuilder 파라미터
memberRepository.createQueryBuilder('m')
  .where('m.name = :name', { name })
  .getOne();

// 위험 — raw query에 문자열 직접 삽입
dataSource.query(`SELECT * FROM member WHERE name = '${name}'`);
// 절대 이렇게 하지 말 것
```

&nbsp;

**둘 다 ORM을 정상적으로 쓰면 안전하다.** 위험한 건 raw query에 문자열을 직접 넣을 때다. 이건 Java든 Node.js든 동일하다.

&nbsp;

raw query가 필요할 때는 반드시 파라미터 바인딩을 사용한다:

```typescript
// 안전한 raw query
dataSource.query('SELECT * FROM member WHERE name = $1', [name]);
```

&nbsp;

&nbsp;

---

&nbsp;

## 7. AI가 만든 코드에서 자주 빠지는 보안

&nbsp;

AI에게 코드를 생성시키면, Java든 Node.js든 비슷한 보안 실수가 나온다.

&nbsp;

### 공통적으로 빠지는 것들

| 항목 | 설명 |
|:---|:---|
| Rate Limiting | 로그인 시도 횟수 제한 없이 생성 |
| Input Sanitization | HTML/XSS 필터링 누락 |
| Error Detail 노출 | 에러 메시지에 스택 트레이스 포함 |
| 환경변수 하드코딩 | JWT 시크릿을 코드에 직접 작성 |
| 파일 업로드 검증 | 확장자/크기 제한 없이 구현 |

&nbsp;

**Node.js에서 추가로 주의할 것:**

```typescript
// 위험 — prototype pollution
const merged = { ...defaultConfig, ...userInput };
// userInput에 __proto__ 키가 있으면 위험

// 안전 — Object.create(null) 또는 검증
const safeInput = Object.fromEntries(
  Object.entries(userInput).filter(([key]) => !key.startsWith('__'))
);
```

&nbsp;

```typescript
// 위험 — eval 계열
eval(userInput);
new Function(userInput)();
// 절대 사용 금지

// 위험 — RegExp DoS (ReDoS)
const regex = new RegExp(userInput); // 사용자 입력으로 정규식 생성 금지
```

&nbsp;

**Java에서 추가로 주의할 것:**

```java
// 위험 — 직렬화 취약점
ObjectInputStream ois = new ObjectInputStream(inputStream);
Object obj = ois.readObject(); // 신뢰할 수 없는 데이터 역직렬화

// 위험 — SSRF
URL url = new URL(userProvidedUrl);
url.openStream(); // 사용자가 내부 네트워크 URL 입력 가능
```

&nbsp;

&nbsp;

---

&nbsp;

## 8. 검증 레이어 위치 비교

&nbsp;

```
Java:
  Client → Controller(@Valid) → Service(비즈니스 검증) → Repository
           ↑ Bean Validation       ↑ 수동 검증

Node.js:
  Client → Middleware(Zod) → Handler → Service(비즈니스 검증) → Repository
           ↑ 스키마 검증              ↑ 수동 검증
```

&nbsp;

| 검증 단계 | Java | Node.js |
|:---|:---|:---|
| 입력 형식 | @Valid + Bean Validation | Zod 스키마 |
| 비즈니스 규칙 | Service 레이어에서 수동 | Service 레이어에서 수동 |
| DB 제약 조건 | @Column(unique) + DB | Entity 데코레이터 + DB |
| 보안 검증 | Spring Security | 커스텀 미들웨어 |

&nbsp;

입력 형식 검증 방식만 다르고, 비즈니스 로직 검증은 어차피 둘 다 서비스 레이어에서 직접 해야 한다.

&nbsp;

&nbsp;

---

&nbsp;

## 정리

&nbsp;

| 항목 | Java/Spring | Node.js |
|:---|:---|:---|
| 입력 검증 | @Valid + Bean Validation | Zod + 미들웨어 |
| 타입-검증 통합 | DTO 클래스 = 타입 + 검증 | z.infer = 타입 + 검증 |
| 보안 프레임워크 | Spring Security (올인원) | 직접 조립 (미들웨어) |
| CORS | WebMvcConfigurer | cors 미들웨어 |
| SQL Injection | JPA 파라미터 바인딩 | TypeORM 파라미터 바인딩 |
| 검증 생태계 | 성숙함 (20년+) | 빠르게 성장 중 (Zod) |

&nbsp;

**Java는 프레임워크가 많이 해주고, Node.js는 직접 조립해야 한다.**

이건 불편함이 아니라 트레이드오프다. Java는 "Spring이 해줄 거니까 신경 꺼"이고, Node.js는 "네가 직접 골라서 조립해"다.

&nbsp;

Zod의 타입 추론은 Java Bean Validation이 제공하지 못하는 장점이다. 스키마 하나로 런타임 검증과 컴파일 타입 안전성을 동시에 얻을 수 있다.

&nbsp;

다음 편에서는 **JPA vs TypeORM/Prisma, ORM 성숙도의 현실적인 차이**를 비교한다.

&nbsp;

&nbsp;

---

Java, Node.js, Spring, 검증, Validation, Zod, Joi, BeanValidation, SpringSecurity, JWT, CORS, SQLInjection, TypeScript, 보안, 미들웨어, AI코딩