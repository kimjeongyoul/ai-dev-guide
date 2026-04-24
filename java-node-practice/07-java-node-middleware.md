# [Java vs Node.js 2편] AOP가 없는 세상 — Node.js 미들웨어로 횡단 관심사 처리하기

&nbsp;

Java/Spring에서 로깅, 인증, 트랜잭션은 어노테이션 한 줄이면 끝난다.

&nbsp;

```java
@Transactional
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }
```

&nbsp;

Node.js에서는? **직접 감싸야 한다.**

&nbsp;

이게 불편한 건 맞다. 하지만 **"무슨 일이 일어나는지 코드에 다 보인다"**는 건 장점이기도 하다.

&nbsp;

&nbsp;

---

&nbsp;

## 1. Java의 횡단 관심사 — 선언적으로 끝

&nbsp;

Spring에서 횡단 관심사(Cross-Cutting Concerns)를 처리하는 방법은 크게 세 가지다.

&nbsp;

### 1-1. AOP (@Aspect)

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example..service..*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();

        Object result = joinPoint.proceed();

        long duration = System.currentTimeMillis() - start;
        log.info("{} executed in {}ms",
            joinPoint.getSignature().getName(), duration);

        return result;
    }
}
```

&nbsp;

이 코드 하나로 **모든 서비스 메서드의 실행 시간이 자동으로 로깅된다.** 서비스 코드를 한 줄도 건드리지 않고.

&nbsp;

### 1-2. HandlerInterceptor

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        String token = request.getHeader("Authorization");
        if (!jwtService.isValid(token)) {
            response.setStatus(401);
            return false;
        }
        return true;
    }
}
```

&nbsp;

### 1-3. @ControllerAdvice (전역 에러 핸들링)

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(NotFoundException e) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("NOT_FOUND", e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception e) {
        log.error("Unhandled exception", e);
        return ResponseEntity.status(500)
            .body(new ErrorResponse("INTERNAL_ERROR", "서버 오류"));
    }
}
```

&nbsp;

**공통점:** 비즈니스 코드에 침투하지 않는다. 선언만 하면 프레임워크가 끼워넣어준다.

&nbsp;

&nbsp;

---

&nbsp;

## 2. Node.js의 횡단 관심사 — 미들웨어와 함수 래핑

&nbsp;

Node.js(Express)에서는 `app.use()`와 함수 래핑이 AOP를 대신한다.

&nbsp;

### 2-1. 미들웨어 (= Interceptor)

```typescript
// 로깅 미들웨어
function requestLogger(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.path} ${res.statusCode} ${duration}ms`);
  });

  next();
}

app.use(requestLogger);
```

&nbsp;

### 2-2. 인증 미들웨어

```typescript
function authMiddleware(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'Token required' });
  }

  try {
    const decoded = jwt.verify(token, SECRET_KEY);
    req.user = decoded;
    next();
  } catch {
    return res.status(401).json({ error: 'Invalid token' });
  }
}

// 특정 라우트에만 적용
router.get('/admin/users', authMiddleware, adminController.getUsers);

// 전체 적용
app.use('/api/admin', authMiddleware);
```

&nbsp;

### 2-3. 에러 핸들링 미들웨어

```typescript
// Express 에러 미들웨어 — 인자 4개가 시그니처
function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  if (err instanceof NotFoundException) {
    return res.status(404).json({
      error: 'NOT_FOUND',
      message: err.message,
    });
  }

  console.error('Unhandled error:', err);
  res.status(500).json({
    error: 'INTERNAL_ERROR',
    message: '서버 오류',
  });
}

// 반드시 라우트 등록 이후에 선언
app.use(errorHandler);
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. 1:1 비교 — 같은 기능, 다른 표현

&nbsp;

### 인증

&nbsp;

**Java: SecurityFilterChain**

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
        .build();
}
```

&nbsp;

**Node.js: 미들웨어 수동 적용**

```typescript
// 공개 API — 미들웨어 없음
app.use('/api/public', publicRouter);

// 인증 필요 API
app.use('/api', authMiddleware);

// 관리자 API — 인증 + 권한 체크
app.use('/api/admin', authMiddleware, roleMiddleware('ADMIN'));

function roleMiddleware(requiredRole: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (req.user?.role !== requiredRole) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}
```

&nbsp;

### 에러 핸들링

&nbsp;

**Java: @ExceptionHandler 한 곳에서 전부 처리**

```java
@ExceptionHandler(ValidationException.class)
public ResponseEntity<ErrorResponse> handleValidation(ValidationException e) {
    return ResponseEntity.badRequest()
        .body(new ErrorResponse("VALIDATION_ERROR", e.getMessage()));
}
```

&nbsp;

**Node.js: 에러 미들웨어 + 커스텀 에러 클래스**

```typescript
// 커스텀 에러
class AppError extends Error {
  constructor(
    public statusCode: number,
    public code: string,
    message: string
  ) {
    super(message);
  }
}

class ValidationError extends AppError {
  constructor(message: string) {
    super(400, 'VALIDATION_ERROR', message);
  }
}

// 에러 미들웨어
function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: err.code,
      message: err.message,
    });
  }

  console.error(err);
  res.status(500).json({ error: 'INTERNAL_ERROR', message: '서버 오류' });
}
```

&nbsp;

### 로깅

&nbsp;

**Java: AOP @Around — 코드 한 줄 안 건드림**

```java
@Around("@annotation(Loggable)")
public Object logMethod(ProceedingJoinPoint jp) throws Throwable {
    log.info("→ {}", jp.getSignature());
    Object result = jp.proceed();
    log.info("← {} returned", jp.getSignature());
    return result;
}

// 사용
@Loggable
public MemberDto findById(Long id) { ... }
```

&nbsp;

**Node.js: 래퍼 함수**

```typescript
function withLogging<T extends (...args: any[]) => Promise<any>>(
  name: string,
  fn: T
): T {
  return (async (...args: any[]) => {
    console.log(`→ ${name}`, args);
    const result = await fn(...args);
    console.log(`← ${name} returned`);
    return result;
  }) as T;
}

// 사용
class MemberService {
  findById = withLogging('MemberService.findById', async (id: string) => {
    return this.repo.findById(id);
  });
}
```

&nbsp;

어노테이션 한 줄 vs 래퍼 함수 감싸기. **Java가 더 우아하다.** 이건 인정해야 한다.

&nbsp;

&nbsp;

---

&nbsp;

## 4. 트랜잭션 — 가장 차이가 큰 부분

&nbsp;

**Java: @Transactional 한 줄**

```java
@Transactional
public void transferPoints(Long fromId, Long toId, int points) {
    Member from = memberRepository.findById(fromId).orElseThrow();
    Member to = memberRepository.findById(toId).orElseThrow();

    from.deductPoints(points);
    to.addPoints(points);

    // 예외 발생 시 자동 롤백. 명시적 commit/rollback 불필요
}
```

&nbsp;

**Node.js: 수동 트랜잭션**

```typescript
async function transferPoints(fromId: string, toId: string, points: number) {
  const queryRunner = dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    const from = await queryRunner.manager.findOneBy(Member, { id: fromId });
    const to = await queryRunner.manager.findOneBy(Member, { id: toId });

    if (!from || !to) throw new Error('Member not found');

    from.points -= points;
    to.points += points;

    await queryRunner.manager.save(from);
    await queryRunner.manager.save(to);

    await queryRunner.commitTransaction();
  } catch (err) {
    await queryRunner.rollbackTransaction();
    throw err;
  } finally {
    await queryRunner.release();
  }
}
```

&nbsp;

**10배 이상 코드가 길어진다.** 이걸 줄이려면 래퍼를 만들어야 한다:

```typescript
async function withTransaction<T>(
  fn: (manager: EntityManager) => Promise<T>
): Promise<T> {
  const queryRunner = dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    const result = await fn(queryRunner.manager);
    await queryRunner.commitTransaction();
    return result;
  } catch (err) {
    await queryRunner.rollbackTransaction();
    throw err;
  } finally {
    await queryRunner.release();
  }
}

// 사용 — 그나마 깔끔해짐
async function transferPoints(fromId: string, toId: string, points: number) {
  await withTransaction(async (manager) => {
    const from = await manager.findOneBy(Member, { id: fromId });
    const to = await manager.findOneBy(Member, { id: toId });

    from!.points -= points;
    to!.points += points;

    await manager.save([from, to]);
  });
}
```

&nbsp;

결국 Node.js에서는 **Spring이 자동으로 해주는 걸 직접 유틸로 만들어야 한다.**

&nbsp;

&nbsp;

---

&nbsp;

## 5. 커스텀 어노테이션 vs 래퍼 함수

&nbsp;

**Java: 커스텀 어노테이션으로 재사용**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Auditable {
    String action();
}

@Aspect
@Component
public class AuditAspect {
    @Around("@annotation(auditable)")
    public Object audit(ProceedingJoinPoint jp, Auditable auditable) throws Throwable {
        Object result = jp.proceed();
        auditService.log(auditable.action(), jp.getArgs());
        return result;
    }
}

// 사용
@Auditable(action = "DELETE_MEMBER")
public void deleteMember(Long id) { ... }
```

&nbsp;

**Node.js: 데코레이터 (실험적) 또는 래퍼**

```typescript
// TypeScript 데코레이터 (experimentalDecorators 필요)
function Auditable(action: string) {
  return function (target: any, key: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    descriptor.value = async function (...args: any[]) {
      const result = await original.apply(this, args);
      await auditService.log(action, args);
      return result;
    };
  };
}

// 사용
class MemberService {
  @Auditable('DELETE_MEMBER')
  async deleteMember(id: string) { ... }
}
```

&nbsp;

TypeScript 데코레이터를 쓰면 Java와 비슷해진다. 다만 **TC39 데코레이터 표준이 아직 안정화되지 않아서** 프로덕션에서 쓰기 꺼려하는 팀이 많다.

&nbsp;

현실적으로는 고차 함수(Higher-Order Function)로 처리하는 경우가 더 많다:

```typescript
function withAudit(action: string, fn: Function) {
  return async (...args: any[]) => {
    const result = await fn(...args);
    await auditService.log(action, args);
    return result;
  };
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 6. Next.js App Router에서의 횡단 관심사

&nbsp;

Express가 아닌 Next.js App Router 환경에서는 또 다르다.

&nbsp;

```typescript
// Next.js middleware.ts — 전역 미들웨어
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  // 인증 체크
  const token = request.headers.get('authorization');
  if (request.nextUrl.pathname.startsWith('/api/admin') && !token) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  return NextResponse.next();
}

export const config = {
  matcher: '/api/:path*',
};
```

&nbsp;

```typescript
// API Route에서 래퍼 패턴
// lib/api-handler.ts
export function withApiLog(
  handler: (req: NextRequest) => Promise<NextResponse>
) {
  return async (req: NextRequest) => {
    const start = Date.now();
    try {
      const response = await handler(req);
      const duration = Date.now() - start;
      // 비동기로 로그 저장 (fire-and-forget)
      logService.save({
        endpoint: req.nextUrl.pathname,
        method: req.method,
        duration,
        status: response.status,
      });
      return response;
    } catch (err) {
      // 에러 로깅 + 에러 응답
      console.error(err);
      return NextResponse.json(
        { error: 'Internal Server Error' },
        { status: 500 }
      );
    }
  };
}

// 사용
// app/api/members/route.ts
export const GET = withApiLog(async (request) => {
  const members = await memberService.findAll();
  return NextResponse.json(members);
});
```

&nbsp;

Next.js의 `withApiLog` 패턴은 Java의 `@Around` AOP와 **역할이 같다.** 다만 어노테이션이 아니라 함수로 감싸는 것뿐이다.

&nbsp;

&nbsp;

---

&nbsp;

## 7. 솔직한 비교

&nbsp;

| 항목 | Java/Spring | Node.js |
|:---|:---|:---|
| 인증 | SecurityFilterChain (선언적) | 미들웨어 수동 적용 |
| 로깅 | AOP @Around (비침투적) | 미들웨어 or 래퍼 |
| 에러 핸들링 | @ControllerAdvice (전역) | 에러 미들웨어 |
| 트랜잭션 | @Transactional (한 줄) | 수동 try-commit-rollback |
| 감사 로그 | 커스텀 어노테이션 + AOP | 래퍼 함수 or 데코레이터 |
| 디버깅 | 프록시/매직이 많아 추적 어려움 | 코드에 다 보여서 추적 쉬움 |

&nbsp;

**Java가 더 우아한 건 사실이다.** 어노테이션 한 줄로 끝나는 세상은 편하다.

&nbsp;

하지만 Node.js의 **명시적 접근**도 장점이 있다:
- 미들웨어 체인이 어떤 순서로 실행되는지 코드에서 바로 보인다
- "이 API에 인증이 걸려있나?" — import 따라가면 즉시 확인
- AOP 프록시의 마법이 없어서 디버깅이 직관적
- 스택 트레이스가 깨끗하다

&nbsp;

**결론: Java는 "선언하면 알아서 해주는" 편안함, Node.js는 "다 보이는" 투명함.** 취향이 아니라 프로젝트 규모에 따라 선택하는 게 맞다.

&nbsp;

다음 편에서는 **@Valid 없이 요청 검증을 어떻게 하는지, 그리고 Spring Security 없이 보안을 어떻게 구현하는지** 비교한다.

&nbsp;

&nbsp;

---

Java, Node.js, Spring, Express, AOP, 미들웨어, 횡단관심사, 인증, 에러핸들링, 트랜잭션, 어노테이션, TypeScript, NextJS, 래퍼패턴, 데코레이터