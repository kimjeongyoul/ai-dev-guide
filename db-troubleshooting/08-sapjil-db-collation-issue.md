# [실전 삽질 4편] 분명 local에선 됐는데... — DB 정렬 기준(Collation)이 부른 대참사와 조인 성능 파괴

&nbsp;

"로컬 환경에서는 테스트가 완벽하게 통과했습니다. 그런데 운영 서버에 배포하자마자 로그인 로직이 완전히 꼬여버렸어요."

&nbsp;

개발자라면 누구나 한 번쯤 겪어봤을 법한 환경 파편화 이슈. 하지만 이번 버그의 양상은 유독 기괴했다. `Admin` 이라는 아이디로 로그인을 시도했는데, 정작 DB에는 `admin` 이라는 소문자 아이디로 가입된 유저의 정보가 반환되거나, 반대로 새로운 아이디 가입 로직에서 명백한 중복 아이디가 그대로 통과되어 Insert되는 현상이 산발적으로 발생했다. 

&nbsp;

애플리케이션 코드는 깃허브(GitHub)의 커밋 해시(Hash)까지 똑같은 완전한 동일 버전이었다. 유일한 차이는 개발자의 로컬 PC(Mac)에 띄운 Docker MySQL 컨테이너와 AWS 클라우드의 운영 환경 RDS 인스턴스라는 인프라적 차이뿐이었다. 버그의 근본 원인은 코드 로직이 아니라, 눈에 잘 띄지 않는 데이터베이스 스키마 설정의 밑바닥, 바로 **문자셋(Character Set)**과 **콜레이션(Collation)**의 불일치에 있었다. 본 글에서는 무심코 지나치기 쉬운 Collation 설정이 어떻게 데이터 무결성을 파괴하고, 심지어 수억 건의 조인(Join) 성능을 나락으로 떨어뜨리는지 심층 분석한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 현상 파악: 'A'와 'a'는 과연 같은 문자인가?

&nbsp;

RDBMS에서 두 문자열을 비교할 때 잣대가 되는 규칙을 콜레이션(Collation)이라고 부른다. 
우리의 로컬 DB와 운영 DB의 설정을 직접 까보며 진실을 대면했다.

&nbsp;

```sql
-- 테이블 스키마 정보 조회
SELECT table_name, table_collation 
FROM information_schema.tables 
WHERE table_schema = 'our_service_db';
```

&nbsp;

- **로컬 DB 설정**: `utf8mb4_general_ci`
- **운영 DB 설정**: `utf8mb4_bin`

&nbsp;

이 알파벳 몇 글자의 차이가 참사를 불러왔다.

&nbsp;

## 1-1. Case Insensitive (ci)의 함정
접미사 `_ci`는 Case Insensitive의 약자로, 대소문자를 구분하지 않는다는 뜻이다. 
로컬 환경에서 `WHERE username = 'Admin'` 쿼리를 날리면, DB는 친절하게도 소문자 `admin` 레코드까지 동일한 값으로 취급하여 반환한다. 이는 검색의 유연성을 제공하지만, 유일성(Uniqueness)을 엄격하게 보장해야 하는 회원가입이나 인증 로직에서는 악몽이 된다.

&nbsp;

## 1-2. Binary (bin)의 엄격함
운영 환경에 설정되어 있던 `_bin`은 Binary의 약자다. 텍스트를 사람이 읽는 문자가 아닌, 메모리에 저장된 바이너리 바이트 코드 그대로 비교한다. 
당연히 대문자 'A'와 소문자 'a'의 아스키코드(ASCII) 값이 다르므로 철저하게 다른 데이터로 판별한다. 따라서 로컬에서 작성한 '대소문자 무시' 기반의 느슨한 로그인 검증 로직이 운영 환경에서는 완전히 다르게 작동하며 데이터 불일치를 일으킨 것이다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 보이지 않는 성능 살인마: Illegal mix of collations

&nbsp;

대소문자 구분 문제로 끝났다면 다행이련만, 진정한 대참사는 백오피스(Back-office) 어드민 페이지에서 벌어졌다. 두 테이블을 엮어서 통계를 뽑아내는 조인(Join) 쿼리의 응답 속도가 30초를 넘어가며 타임아웃을 뿜어냈다.

&nbsp;

```sql
-- 조인 성능 테스트 (orders 500만 건, users 100만 건)
SELECT o.order_id, u.username
FROM orders o
JOIN users u ON o.user_id = u.user_id; -- 참사의 진원지
```

&nbsp;

실행 계획(EXPLAIN)을 확인해보니 놀랍게도 양쪽 테이블의 `user_id` 컬럼에 인덱스가 완벽하게 걸려있음에도 불구하고, MySQL 옵티마이저가 인덱스를 무시하고 Full Table Scan을 타며 중첩 루프(Nested Loop)를 돌고 있었다. 
에러 로그에는 간헐적으로 `Illegal mix of collations for operation '='` 경고가 찍혀 있었다.

&nbsp;

## 2-1. 조인 조건의 타입 미스매치 (Collation 불일치)
원인은 각 테이블이 생성된 시점이 달라, `users` 테이블의 `user_id`는 `utf8mb4_unicode_ci`로, 새로 추가된 `orders` 테이블의 `user_id`는 `utf8mb4_general_ci`로 콜레이션이 다르게 설정되어 있었던 것이다.

&nbsp;

RDBMS에서 인덱스를 기반으로 두 컬럼을 빠르게 매칭(Join)하려면 비교 대상의 '데이터 타입'은 물론 '정렬 규칙(Collation)'까지 100% 동일해야 한다. 비교 기준이 다르면 DB 엔진은 한쪽 컬럼의 문자열 데이터를 다른 쪽 기준에 맞춰 런타임에 변환(Implicit Cast)해야 한다. 이 변환 작업이 매 로우(Row)마다 함수 실행을 강제하게 되므로, 인덱스 트리를 탐색하는 B-Tree의 이점(Sargability)이 완벽히 파괴되어 버리는 것이다. 고작 정렬 설정 하나가 수천만 건의 풀 스캔을 유발하는 치명적 뇌관이었다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 기술적 마이그레이션: 전사적 인프라 통일 전략

&nbsp;

이 사태를 수습하기 위해서는 애플리케이션 코드를 수정하는 것이 아니라, 인프라스트럭처 레벨에서 DB의 Character Set과 Collation을 전면 재조정해야 했다.

&nbsp;

## 3-1. 최적의 표준 포맷 선정
최신 이모지(Emoji, 4바이트 크기)를 완벽하게 저장하고 다국어 정렬을 가장 표준적으로 지원하기 위해 전사 표준을 다음과 같이 확정했다.
- **Character Set**: `utf8mb4` (기존 utf8은 3바이트까지만 지원하여 이모지 저장 시 에러 발생)
- **Collation**: `utf8mb4_0900_ai_ci` (MySQL 8.0 기준 가장 진보된 유니코드 9.0 정렬 알고리즘. 대소문자와 억음 기호를 스마트하게 무시하여 검색 편의성 증대)

&nbsp;

*단, 회원 아이디, 패스워드 해시값 등 바이너리 수준의 엄격한 비교가 필수적인 단일 컬럼에 한해서만 예외적으로 `utf8mb4_bin`을 부여하는 하이브리드 전략을 채택했다.*

&nbsp;

## 3-2. 무중단 ALTER 테이블 전략
이미 라이브 중인 수천만 건의 테이블에 대해 Collation을 통일하는 `ALTER TABLE` 쿼리를 날리는 것은 위험하다. 전체 테이블에 메타데이터 락(Metadata Lock)이 걸려 서비스 장애를 유발할 수 있다.
이를 극회하기 위해 Percona Toolkit의 `pt-online-schema-change`와 같은 외부 도구를 도입했다.
1. 목표 스키마와 동일하지만 Collation만 수정한 임시(Ghost) 테이블을 생성.
2. 기존 데이터를 청크 단위로 백그라운드 복사.
3. 복사 중 변경되는 데이터는 트리거(Trigger)를 통해 실시간 동기화.
4. 순식간에 테이블 이름을 Swap하여 다운타임 없이 마이그레이션을 완료했다.

&nbsp;

## 3-3. DDL 스크립트 중앙 통제
향후 동일한 이슈를 방지하기 위해 JPA나 TypeORM의 `auto_update` 스키마 생성 기능을 실서버에서 원천 차단하고, Flyway나 Liquibase 같은 데이터베이스 마이그레이션 도구를 도입하여 모든 CREATE/ALTER 스크립트에 반드시 `DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci` 구문이 하드코딩되도록 형상 관리 파이프라인을 구축했다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. 결론: 인프라의 기본 설정은 코드보다 무겁다

&nbsp;

개발 환경과 운영 환경의 인프라 설정이 1바이트라도 다르면, 그것은 시한폭탄을 안고 코드를 작성하는 것과 같다. 

&nbsp;

- 문자열 비교에서 대소문자가 구별되지 않는 기이한 현상을 마주했다면 가장 먼저 테이블의 Collation 설정을 조회하라.
- 성능 튜닝의 기본은 복잡한 쿼리를 고치는 것이 아니라, 조인되는 컬럼 간의 데이터 타입과 콜레이션을 100% 일치시켜 인덱스의 효용성을 되살리는 데 있다.
- Docker를 활용하여 개발 환경, CI 테스트 환경, 상용 환경의 DB 이미지를 완벽하게 동기화(Infrastructure as Code)하라.

&nbsp;

보이지 않는 설정 하나가 서비스의 멱살을 쥐고 흔들 수 있음을 잊지 말아야 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[실전 삽질 5편] 대량 데이터 삭제(Delete)의 역습 — Undo Log와 MVCC 병목의 파국**

&nbsp;

"디스크가 부족합니다. 지난 3년 치 로그 데이터 5천만 건 좀 한 번에 지워주세요." 이 단순하고 순진한 `DELETE` 쿼리 한 줄이 어떻게 데이터베이스의 디스크를 오히려 가득 채우게 만들고, CPU를 100%로 마비시키며 서비스 전체를 응답 불가 상태로 몰고 갔을까? InnoDB 엔진의 치명적인 매커니즘인 Undo Log 폭증과 MVCC(Multi-Version Concurrency Control) 찌꺼기(Purge Lag)의 끔찍한 연쇄 작용을 해부하고, 무중단으로 수억 건의 데이터를 안전하게 썰어내는(Chunking) 대규모 마이그레이션 아키텍처를 공개한다.

&nbsp;

&nbsp;

---

&nbsp;

Collation, 데이터베이스정렬, 조인성능최적화, MySQL장애분석, utf8mb4, 환경파편화, 백엔드삽질기, 인덱스무력화
