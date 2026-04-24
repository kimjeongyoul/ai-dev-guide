# [Java vs Node.js 4편] JPA vs TypeORM/Prisma — ORM의 성숙도 차이가 체감되는 순간

&nbsp;

Java의 JPA는 20년 넘게 검증된 ORM이다.

Node.js의 TypeORM은 JPA에서 영감을 받았다. Prisma는 완전히 다른 접근을 한다.

&nbsp;

셋 다 "SQL 안 쓰고 DB 다루기"라는 같은 목표를 갖고 있지만, **실무에서 쓰면 성숙도 차이가 체감된다.**

&nbsp;

&nbsp;

---

&nbsp;

## 1. 엔티티 정의 — 비슷한데 미묘하게 다르다

&nbsp;

### Java JPA

```java
@Entity
@Table(name = "members")
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Enumerated(EnumType.STRING)
    private MemberGrade grade;

    @OneToMany(mappedBy = "member", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<MemberCard> cards = new ArrayList<>();

    @CreationTimestamp
    private LocalDateTime createdAt;

    // getter, setter, constructor
}
```

&nbsp;

### TypeORM

```typescript
@Entity('members')
export class Member {

  @PrimaryGeneratedColumn()
  id: number;

  @Column({ nullable: false, length: 100 })
  name: string;

  @Column({ unique: true, nullable: false })
  email: string;

  @Column({ type: 'varchar', length: 20 })
  grade: MemberGrade;

  @OneToMany(() => MemberCard, (card) => card.member, { cascade: true })
  cards: MemberCard[];

  @CreateDateColumn()
  createdAt: Date;
}
```

&nbsp;

### Prisma

```prisma
// schema.prisma — 코드가 아니라 스키마 파일
model Member {
  id        Int          @id @default(autoincrement())
  name      String       @db.VarChar(100)
  email     String       @unique
  grade     MemberGrade
  cards     MemberCard[]
  createdAt DateTime     @default(now())

  @@map("members")
}

enum MemberGrade {
  BRONZE
  SILVER
  GOLD
  PLATINUM
}
```

&nbsp;

**차이점:**
- JPA, TypeORM: 데코레이터 기반, 클래스가 곧 엔티티
- Prisma: 별도 스키마 파일(DSL), `npx prisma generate`로 타입 자동 생성
- Prisma는 엔티티 클래스가 없다. 타입만 있다.

&nbsp;

&nbsp;

---

&nbsp;

## 2. 관계 매핑 — @OneToMany는 같은데...

&nbsp;

### 1:N 관계

&nbsp;

**Java JPA:**

```java
// Member (1) → MemberCard (N)
@Entity
public class Member {
    @OneToMany(mappedBy = "member", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<MemberCard> cards;
}

@Entity
public class MemberCard {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;
}
```

&nbsp;

**TypeORM:**

```typescript
@Entity()
export class Member {
  @OneToMany(() => MemberCard, (card) => card.member, { cascade: true })
  cards: MemberCard[];
}

@Entity()
export class MemberCard {
  @ManyToOne(() => Member, (member) => member.cards)
  @JoinColumn({ name: 'member_id' })
  member: Member;
}
```

&nbsp;

**Prisma:**

```prisma
model Member {
  cards MemberCard[]
}

model MemberCard {
  memberId Int    @map("member_id")
  member   Member @relation(fields: [memberId], references: [id])
}
```

&nbsp;

문법은 비슷하다. 차이가 나는 건 **Lazy Loading과 영속성 컨텍스트**다.

&nbsp;

&nbsp;

---

&nbsp;

## 3. 영속성 컨텍스트 — JPA만의 세계

&nbsp;

JPA에는 **영속성 컨텍스트(Persistence Context)**가 있다. TypeORM과 Prisma에는 없다.

&nbsp;

```java
// JPA — 영속성 컨텍스트가 동작하는 예
@Transactional
public void updateName(Long id, String newName) {
    Member member = memberRepository.findById(id).orElseThrow();
    member.setName(newName);
    // save() 호출 안 해도 트랜잭션 커밋 시 자동 UPDATE
    // → 더티 체킹(Dirty Checking)
}
```

&nbsp;

```java
// JPA — 1차 캐시
@Transactional
public void example(Long id) {
    Member m1 = memberRepository.findById(id).orElseThrow(); // DB 조회
    Member m2 = memberRepository.findById(id).orElseThrow(); // 1차 캐시에서 리턴
    // m1 == m2 → true (같은 인스턴스)
}
```

&nbsp;

**TypeORM — 이런 거 없다:**

```typescript
async function updateName(id: number, newName: string) {
  const member = await memberRepository.findOneBy({ id });
  if (!member) throw new Error('Not found');

  member.name = newName;
  await memberRepository.save(member); // 반드시 save() 호출해야 함

  // 1차 캐시도 없음
  const m1 = await memberRepository.findOneBy({ id }); // DB 조회
  const m2 = await memberRepository.findOneBy({ id }); // 또 DB 조회
  // m1 === m2 → false (다른 인스턴스)
}
```

&nbsp;

**Prisma — 더 명시적:**

```typescript
async function updateName(id: number, newName: string) {
  // update를 직접 호출. 조회 후 수정이 아니라, 바로 UPDATE 쿼리
  await prisma.member.update({
    where: { id },
    data: { name: newName },
  });
}
```

&nbsp;

| 기능 | JPA | TypeORM | Prisma |
|:---|:---|:---|:---|
| 영속성 컨텍스트 | O | X | X |
| 더티 체킹 | O (자동 UPDATE) | X (save 필수) | X (update 호출) |
| 1차 캐시 | O | X | X |
| 동일성 보장 | O (같은 트랜잭션) | X | X |

&nbsp;

JPA의 영속성 컨텍스트는 **강력하지만 마법 같아서** 디버깅이 어려운 원인이 되기도 한다. TypeORM과 Prisma는 이런 마법이 없어서 **예측 가능하다.**

&nbsp;

&nbsp;

---

&nbsp;

## 4. N+1 문제 — 해결 방법이 다르다

&nbsp;

N+1 문제: 1개 조회 + N개 연관 엔티티 각각 조회 = N+1개 쿼리

&nbsp;

**Java JPA — fetch join:**

```java
// JPQL fetch join
@Query("SELECT m FROM Member m JOIN FETCH m.cards WHERE m.grade = :grade")
List<Member> findByGradeWithCards(@Param("grade") MemberGrade grade);

// EntityGraph
@EntityGraph(attributePaths = {"cards"})
List<Member> findByGrade(MemberGrade grade);
```

&nbsp;

**TypeORM — relations 옵션:**

```typescript
// 방법 1: find 옵션
const members = await memberRepository.find({
  where: { grade: 'GOLD' },
  relations: ['cards'], // LEFT JOIN으로 가져옴
});

// 방법 2: QueryBuilder
const members = await memberRepository
  .createQueryBuilder('m')
  .leftJoinAndSelect('m.cards', 'card')
  .where('m.grade = :grade', { grade: 'GOLD' })
  .getMany();
```

&nbsp;

**Prisma — include:**

```typescript
const members = await prisma.member.findMany({
  where: { grade: 'GOLD' },
  include: { cards: true }, // 별도 쿼리로 가져옴 (JOIN 아님)
});
```

&nbsp;

Prisma의 `include`는 JOIN이 아니라 **두 개의 SELECT 쿼리**를 실행한다. 이게 N+1은 아니지만, JOIN보다 쿼리 수가 많다. 대신 결과 조합이 더 단순하고 예측 가능하다.

&nbsp;

&nbsp;

---

&nbsp;

## 5. 트랜잭션 — 선언적 vs 수동

&nbsp;

**Java:**

```java
@Transactional
public void transfer(Long fromId, Long toId, int amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();

    from.withdraw(amount);
    to.deposit(amount);
    // 메서드 끝나면 자동 commit, 예외 시 자동 rollback
}
```

&nbsp;

**TypeORM:**

```typescript
// 방법 1: queryRunner (가장 명시적)
async function transfer(fromId: number, toId: number, amount: number) {
  const queryRunner = dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();

  try {
    const from = await queryRunner.manager.findOneBy(Account, { id: fromId });
    const to = await queryRunner.manager.findOneBy(Account, { id: toId });

    from!.balance -= amount;
    to!.balance += amount;

    await queryRunner.manager.save([from, to]);
    await queryRunner.commitTransaction();
  } catch (err) {
    await queryRunner.rollbackTransaction();
    throw err;
  } finally {
    await queryRunner.release();
  }
}

// 방법 2: dataSource.transaction (약간 간결)
async function transfer(fromId: number, toId: number, amount: number) {
  await dataSource.transaction(async (manager) => {
    const from = await manager.findOneBy(Account, { id: fromId });
    const to = await manager.findOneBy(Account, { id: toId });

    from!.balance -= amount;
    to!.balance += amount;

    await manager.save([from, to]);
  });
}
```

&nbsp;

**Prisma:**

```typescript
async function transfer(fromId: number, toId: number, amount: number) {
  await prisma.$transaction(async (tx) => {
    const from = await tx.account.update({
      where: { id: fromId },
      data: { balance: { decrement: amount } },
    });

    // 잔액 체크
    if (from.balance < 0) {
      throw new Error('Insufficient balance');
      // throw 하면 자동 rollback
    }

    await tx.account.update({
      where: { id: toId },
      data: { balance: { increment: amount } },
    });
  });
}
```

&nbsp;

| 트랜잭션 | Java | TypeORM | Prisma |
|:---|:---|:---|:---|
| 선언 방식 | @Transactional (한 줄) | queryRunner 또는 callback | $transaction callback |
| 롤백 | 자동 (RuntimeException) | 수동 catch + rollback | 자동 (throw) |
| 전파 | REQUIRED, REQUIRES_NEW 등 | 지원 안 함 | 지원 안 함 |
| 격리 수준 | @Transactional(isolation=) | queryRunner.startTransaction(level) | $transaction({ isolationLevel }) |

&nbsp;

Java의 트랜잭션 전파(Propagation)는 Node.js ORM에서 지원하지 않는다. 중첩 트랜잭션이 필요하면 직접 관리해야 한다.

&nbsp;

&nbsp;

---

&nbsp;

## 6. 마이그레이션

&nbsp;

**Java: Flyway / Liquibase**

```sql
-- V1__create_member.sql
CREATE TABLE members (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- V2__add_grade.sql
ALTER TABLE members ADD COLUMN grade VARCHAR(20) DEFAULT 'BRONZE';
```

&nbsp;

Flyway는 SQL 파일 이름 규칙(V1__, V2__)으로 순서를 관리한다. **SQL을 직접 작성하므로 정확하다.**

&nbsp;

**TypeORM: migration generate**

```bash
npx typeorm migration:generate -n AddMemberGrade
```

```typescript
// 자동 생성된 마이그레이션
export class AddMemberGrade1234567890 implements MigrationInterface {
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `ALTER TABLE "members" ADD "grade" varchar(20) DEFAULT 'BRONZE'`
    );
  }

  async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE "members" DROP COLUMN "grade"`);
  }
}
```

&nbsp;

TypeORM은 엔티티 변경을 감지해서 마이그레이션을 **자동 생성**한다. 편하지만 가끔 이상한 쿼리를 만든다.

&nbsp;

**Prisma: prisma migrate**

```bash
npx prisma migrate dev --name add-member-grade
```

```sql
-- 자동 생성 (prisma/migrations/xxx_add_member_grade/migration.sql)
ALTER TABLE "members" ADD COLUMN "grade" TEXT NOT NULL DEFAULT 'BRONZE';
```

&nbsp;

Prisma는 스키마 파일 변경을 감지해서 SQL 마이그레이션을 자동 생성한다. **생성된 SQL을 직접 확인하고 수정할 수 있다.**

&nbsp;

| 마이그레이션 | Flyway | TypeORM | Prisma |
|:---|:---|:---|:---|
| 방식 | SQL 직접 작성 | 엔티티 비교 자동 생성 | 스키마 비교 자동 생성 |
| 제어력 | 최고 (직접 SQL) | 중간 (수동 수정 가능) | 높음 (SQL 확인/수정) |
| 안정성 | 검증됨 | 가끔 예상 외 쿼리 | 안정적 |
| 롤백 | 수동 SQL | down() 메서드 | 수동 SQL |

&nbsp;

&nbsp;

---

&nbsp;

## 7. 환경별 DB — H2 vs SQLite

&nbsp;

**Java: H2 (인메모리 DB)**

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: create-drop
```

&nbsp;

H2는 JVM 위에서 돌아가는 인메모리 DB라서, **테스트에서 DB 설치 없이 사용 가능하다.**

&nbsp;

**Node.js: SQLite**

```typescript
// data-source.ts
const dataSource = new DataSource({
  type: process.env.DB_TYPE === 'mssql' ? 'mssql' : 'better-sqlite3',
  database: process.env.DB_TYPE === 'mssql'
    ? process.env.DB_DATABASE
    : 'dev.sqlite',
  entities: [Member, MemberCard, Storage],
  synchronize: process.env.DB_TYPE !== 'mssql',
});
```

&nbsp;

```typescript
// Prisma — datasource 분리
// schema.prisma
datasource db {
  provider = env("DATABASE_PROVIDER") // "sqlite" 또는 "postgresql"
  url      = env("DATABASE_URL")
}
```

&nbsp;

| 환경 | Java | Node.js |
|:---|:---|:---|
| 개발/테스트 DB | H2 (인메모리) | SQLite (파일) |
| 프로덕션 DB | MySQL, PostgreSQL, Oracle | PostgreSQL, MySQL, SQL Server |
| DB 전환 | application-{profile}.yml | .env.{profile} 또는 환경변수 |
| ORM 방언 처리 | Hibernate Dialect 자동 | TypeORM: type 변경, Prisma: provider 변경 |

&nbsp;

&nbsp;

---

&nbsp;

## 8. 솔직한 비교 — JPA가 압도적으로 성숙하다

&nbsp;

JPA(Hibernate)는 2001년부터 20년 넘게 발전해왔다. TypeORM은 2016년, Prisma는 2019년 시작이다.

&nbsp;

**JPA에 있고, Node.js ORM에 없는 것들:**

| 기능 | JPA | TypeORM | Prisma |
|:---|:---|:---|:---|
| 영속성 컨텍스트 | O | X | X |
| 더티 체킹 | O | X | X |
| 1차 캐시 | O | X | X |
| 2차 캐시 (EhCache) | O | X | X |
| 트랜잭션 전파 | O (7가지) | X | X |
| JPQL (객체 쿼리) | O | QueryBuilder | X (Prisma Client) |
| Criteria API | O | X | X |
| Batch Insert 최적화 | O | 제한적 | createMany() |
| 낙관적 락 (@Version) | O | @VersionColumn | X |
| 감사 (@CreatedBy) | O (Spring Data) | @CreateDateColumn 정도 | X |

&nbsp;

**하지만 Node.js ORM의 장점도 있다:**

| 장점 | TypeORM | Prisma |
|:---|:---|:---|
| 타입 안전성 | 보통 | 최강 (자동 생성 타입) |
| 학습 곡선 | 낮음 | 매우 낮음 |
| 마법 없음 | O (명시적) | O (더 명시적) |
| 스키마 관리 | 데코레이터 | 별도 DSL (직관적) |
| Edge/Serverless | 가능 | Prisma Accelerate |

&nbsp;

&nbsp;

---

&nbsp;

## 9. TypeORM vs Prisma — 어떤 걸 쓸까

&nbsp;

```
TypeORM을 선택할 때:
├── JPA 경험이 있어서 데코레이터 패턴이 익숙하다
├── Active Record 또는 Repository 패턴을 쓰고 싶다
├── 엔티티 클래스에 비즈니스 로직을 넣고 싶다
└── 기존 프로젝트가 TypeORM이다

Prisma를 선택할 때:
├── 타입 안전성이 최우선이다
├── 스키마 퍼스트(Schema-first) 접근을 원한다
├── 마이그레이션 관리가 깔끔해야 한다
├── Serverless/Edge 환경이다
└── 새 프로젝트를 시작한다
```

&nbsp;

**2024년 기준 트렌드:**
- 새 프로젝트는 **Prisma**가 우세
- TypeORM은 NestJS와의 조합으로 여전히 많이 사용
- Drizzle ORM이 새로운 대안으로 부상 중 (SQL에 더 가까움)

&nbsp;

```typescript
// Drizzle — SQL을 TypeScript로 표현
import { eq } from 'drizzle-orm';

const members = await db
  .select()
  .from(membersTable)
  .where(eq(membersTable.grade, 'GOLD'))
  .leftJoin(cardsTable, eq(membersTable.id, cardsTable.memberId));
```

&nbsp;

Drizzle은 "ORM이 아니라 타입 안전한 SQL 빌더"를 표방한다. JPA보다는 MyBatis/jOOQ에 가깝다.

&nbsp;

&nbsp;

---

&nbsp;

## 정리

&nbsp;

| 항목 | JPA (Java) | TypeORM | Prisma |
|:---|:---|:---|:---|
| 성숙도 | 20년+ | 8년 | 5년 |
| 패턴 | Active Record + Repository | Repository + Active Record | Client (독자) |
| 타입 안전 | 보통 | 보통 | 최강 |
| 영속성 컨텍스트 | O | X | X |
| 트랜잭션 | @Transactional (선언적) | 수동 | $transaction |
| 학습 곡선 | 높음 | 중간 | 낮음 |
| 디버깅 | 어려움 (프록시/마법) | 보통 | 쉬움 |
| 엔터프라이즈 | 검증됨 | 가능 | 성장 중 |

&nbsp;

JPA의 성숙도는 인정해야 한다. 영속성 컨텍스트, 더티 체킹, 트랜잭션 전파 — 이런 건 Node.js ORM에서 아직 제공하지 않는다.

하지만 **"그게 꼭 필요한가?"**라는 질문도 해야 한다. Node.js의 ORM은 마법을 걷어내고 명시적으로 가는 대신, 예측 가능성과 타입 안전성을 얻었다.

&nbsp;

다음 편에서는 **JAR 하나로 배포하는 Java vs node dist/index.js로 배포하는 Node.js, 빌드와 운영의 현실적 차이**를 비교한다.

&nbsp;

&nbsp;

---

Java, Node.js, JPA, TypeORM, Prisma, Drizzle, ORM, 데이터베이스, 영속성컨텍스트, 트랜잭션, N+1, 마이그레이션, TypeScript, Hibernate, 더티체킹