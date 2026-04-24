# 캐시 한 줄이 DB 부하를 90% 줄인다 — Redis 실전 적용

&nbsp;

"DB CPU가 90%예요."

&nbsp;

모니터링 대시보드를 열었다. CPU 90%, 커넥션 풀 포화, 응답 시간 급등.

&nbsp;

슬로우 쿼리 로그를 열었다. 느린 쿼리는 없었다. 전부 0.05초 이내의 빠른 쿼리들.

&nbsp;

그런데 같은 쿼리가 **초당 1,200번** 실행되고 있었다.

&nbsp;

상품 상세 페이지. 인기 상품 하나에 접속이 몰리면서, 같은 SELECT를 1,200번씩 날리고 있었다. 쿼리가 빠르든 느리든, 양이 많으면 DB는 죽는다.

&nbsp;

**답은 캐시였다.**

&nbsp;

&nbsp;

---

&nbsp;

## 1. 캐시란

&nbsp;

```
캐시 없을 때:
  사용자 1 → DB 조회 → 응답
  사용자 2 → DB 조회 → 응답  (같은 데이터)
  사용자 3 → DB 조회 → 응답  (같은 데이터)
  ...
  사용자 1000 → DB 조회 → 응답  (같은 데이터)
  = DB 1,000번 호출

캐시 있을 때:
  사용자 1 → DB 조회 → 캐시에 저장 → 응답
  사용자 2 → 캐시에서 꺼냄 → 응답  (DB 안 감)
  사용자 3 → 캐시에서 꺼냄 → 응답  (DB 안 감)
  ...
  사용자 1000 → 캐시에서 꺼냄 → 응답  (DB 안 감)
  = DB 1번 호출
```

&nbsp;

자주 읽히는 데이터를 메모리에 올려두고, DB까지 가지 않는 것. 그게 캐시다.

&nbsp;

&nbsp;

---

&nbsp;

## 2. 어디에 캐시할 수 있는가

&nbsp;

| 계층 | 위치 | 특징 | 예시 |
|------|------|------|------|
| 브라우저 | 클라이언트 | 네트워크 요청 자체를 안 함 | Cache-Control 헤더 |
| CDN | 엣지 서버 | 정적 파일, 이미지 | CloudFront, CloudFlare |
| 앱 메모리 | 서버 프로세스 | 가장 빠름, 서버 간 공유 안 됨 | Map, HashMap |
| Redis | 별도 서버 | 서버 간 공유, TTL 지원 | Redis, Memcached |
| DB 쿼리 캐시 | DB 서버 | 자동, 하지만 제한적 | MySQL Query Cache |

&nbsp;

실무에서 가장 많이 쓰는 건 **Redis 캐시**다. 서버가 여러 대여도 공유되고, TTL로 자동 만료된다.

&nbsp;

&nbsp;

---

&nbsp;

## 3. Redis 캐시 실전 구현

&nbsp;

### Spring Boot (@Cacheable)

&nbsp;

```java
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    // ✅ 캐시 적용: 같은 id로 호출하면 DB 안 감
    @Cacheable(value = "product", key = "#id")
    public ProductDTO getProduct(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new NotFoundException("상품 없음"));
        return ProductDTO.from(product);
    }

    // ✅ 상품 수정 시 캐시 삭제
    @CacheEvict(value = "product", key = "#id")
    public void updateProduct(Long id, ProductUpdateRequest request) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new NotFoundException("상품 없음"));
        product.update(request);
        productRepository.save(product);
    }
}
```

&nbsp;

```java
// Redis 설정
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))           // TTL 10분
                .serializeValuesWith(
                    SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())
                );

        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
    }
}
```

&nbsp;

### Node.js (ioredis)

&nbsp;

```typescript
import Redis from 'ioredis';

const redis = new Redis({ host: '127.0.0.1', port: 6379 });

async function getProduct(id: string): Promise<Product> {
    const cacheKey = `product:${id}`;

    // 1. 캐시 확인
    const cached = await redis.get(cacheKey);
    if (cached) {
        return JSON.parse(cached);  // 캐시 히트 → DB 안 감
    }

    // 2. 캐시 미스 → DB 조회
    const product = await db.query('SELECT * FROM products WHERE id = ?', [id]);

    // 3. 캐시에 저장 (TTL 600초 = 10분)
    await redis.set(cacheKey, JSON.stringify(product), 'EX', 600);

    return product;
}

async function updateProduct(id: string, data: Partial<Product>): Promise<void> {
    await db.query('UPDATE products SET ... WHERE id = ?', [id]);

    // 캐시 삭제 (다음 조회 시 새 데이터로 캐시됨)
    await redis.del(`product:${id}`);
}
```

&nbsp;

### 캐시 적용 전후 흐름

&nbsp;

```
Before (캐시 없음):
  요청 → Controller → Service → Repository → DB → 응답
  매번 DB 접근. 초당 1,200번.

After (캐시 적용):
  요청 → Controller → Service → Redis에 있나? → 있으면 바로 응답 (95%)
                                              → 없으면 DB → Redis 저장 → 응답 (5%)
  DB 접근: 초당 1,200번 → 초당 60번
```

&nbsp;

&nbsp;

---

&nbsp;

## 4. 캐시 전략

&nbsp;

### Cache-Aside (가장 일반적)

&nbsp;

```
읽기:
  1. 캐시에서 찾는다
  2. 있으면 반환 (Cache Hit)
  3. 없으면 DB에서 읽고, 캐시에 저장 후 반환 (Cache Miss)

쓰기:
  1. DB에 쓴다
  2. 캐시를 삭제한다 (다음 읽기 시 새 데이터로 캐시됨)
```

&nbsp;

```typescript
// Cache-Aside 패턴 구현
async function getCachedData(key: string, fetchFn: () => Promise<any>, ttl: number) {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);

    const data = await fetchFn();
    await redis.set(key, JSON.stringify(data), 'EX', ttl);
    return data;
}

// 사용
const product = await getCachedData(
    `product:${id}`,
    () => productRepository.findById(id),
    600  // 10분
);
```

&nbsp;

### Write-Through (쓰기 동시)

&nbsp;

```
쓰기:
  1. 캐시에 쓴다
  2. DB에도 쓴다
  3. 둘 다 성공하면 완료

장점: 캐시와 DB가 항상 일치
단점: 쓰기가 느림 (두 번 쓰니까)
```

&nbsp;

### Write-Behind (쓰기 지연)

&nbsp;

```
쓰기:
  1. 캐시에만 쓴다 → 바로 응답
  2. 나중에 비동기로 DB에 반영

장점: 쓰기가 매우 빠름
단점: 캐시 서버 죽으면 데이터 유실 위험
```

&nbsp;

**실무에서는 Cache-Aside를 90% 쓴다.** 가장 안전하고 구현이 단순하다.

&nbsp;

### TTL 설정 가이드

&nbsp;

| 데이터 유형 | TTL | 이유 |
|-------------|-----|------|
| 상품 상세 | 5~10분 | 가격 변경이 잦지 않음 |
| 카테고리 목록 | 1시간 | 거의 안 바뀜 |
| 인기 검색어 | 1~5분 | 적당히 실시간 |
| 사용자 프로필 | 5분 | 수정 빈도 낮음 |
| 설정/코드 테이블 | 1일 | 거의 안 바뀜 |

&nbsp;

&nbsp;

---

&nbsp;

## 5. 캐시 주의사항 3가지

&nbsp;

### (1) 캐시 무효화 — 가장 어려운 문제

&nbsp;

> "컴퓨터 과학에서 어려운 것은 두 가지뿐이다: 캐시 무효화와 이름 짓기."
> — Phil Karlton

&nbsp;

```
문제 상황:
  1. 상품 가격 10,000원 → 캐시에 저장됨
  2. 관리자가 가격을 15,000원으로 수정
  3. 캐시를 안 지우면? → 사용자는 10,000원을 본다
  4. 10,000원으로 결제하면? → 장애
```

&nbsp;

```typescript
// 해결: 데이터 수정 시 반드시 캐시 삭제
async function updateProductPrice(id: string, newPrice: number) {
    // 1. DB 업데이트
    await db.query('UPDATE products SET price = ? WHERE id = ?', [newPrice, id]);

    // 2. 캐시 삭제 (필수!)
    await redis.del(`product:${id}`);

    // 3. 목록 캐시도 삭제해야 할 수 있다
    await redis.del(`product-list:category:${categoryId}`);
}
```

&nbsp;

### (2) 캐시 스탬피드 — 동시에 몰리는 문제

&nbsp;

```
TTL 만료 순간:
  09:00:00.000 — 캐시 만료됨
  09:00:00.001 — 요청 100개가 동시에 캐시 미스
  09:00:00.002 — 100개 요청이 전부 DB 조회
  09:00:00.003 — DB에 같은 쿼리 100개가 동시에 도착
  → DB 부하 폭증
```

&nbsp;

```typescript
// 해결: 뮤텍스 (락) 패턴
async function getProductWithLock(id: string): Promise<Product> {
    const cacheKey = `product:${id}`;
    const lockKey = `lock:product:${id}`;

    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    // 락 획득 시도 (NX: 없을 때만, EX: 5초 후 자동 해제)
    const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 5);

    if (acquired) {
        // 락 획득 성공 → DB 조회 후 캐시
        const product = await db.query('SELECT * FROM products WHERE id = ?', [id]);
        await redis.set(cacheKey, JSON.stringify(product), 'EX', 600);
        await redis.del(lockKey);
        return product;
    } else {
        // 락 획득 실패 → 잠시 후 캐시에서 재시도
        await sleep(100);
        return getProductWithLock(id);
    }
}
```

&nbsp;

### (3) 캐시 웜업 — 서버 시작 시 빈 캐시

&nbsp;

```
서버 재시작 → 캐시 비어있음 → 모든 요청이 DB로 → DB 과부하

해결: 서버 시작 시 주요 데이터를 미리 캐시에 로드
```

&nbsp;

```typescript
// 서버 시작 시 캐시 웜업
async function warmUpCache() {
    console.log('캐시 웜업 시작...');

    // 인기 상품 100개 미리 캐시
    const popularProducts = await db.query(
        'SELECT * FROM products ORDER BY view_count DESC LIMIT 100'
    );

    for (const product of popularProducts) {
        await redis.set(`product:${product.id}`, JSON.stringify(product), 'EX', 600);
    }

    // 카테고리 목록 캐시
    const categories = await db.query('SELECT * FROM categories WHERE active = 1');
    await redis.set('categories:all', JSON.stringify(categories), 'EX', 3600);

    console.log(`캐시 웜업 완료: 상품 ${popularProducts.length}개, 카테고리 ${categories.length}개`);
}
```

&nbsp;

&nbsp;

---

&nbsp;

## 6. 실전 Before/After

&nbsp;

### 사례 1: 상품 목록 — DB 직접 조회 → Redis 캐시

&nbsp;

```sql
-- 카테고리별 상품 목록 (매번 DB)
SELECT id, name, price, thumbnail
FROM products
WHERE category_id = 5 AND status = 'ACTIVE'
ORDER BY sort_order ASC;
```

&nbsp;

**Before:**

&nbsp;

```
응답 시간: 200ms (쿼리 자체는 빠름)
초당 요청: 800회
DB CPU: 78%
DB 커넥션: 45/50 (거의 포화)
```

&nbsp;

```typescript
// 캐시 적용
const cacheKey = `products:category:${categoryId}`;
const products = await getCachedData(cacheKey, () => {
    return db.query('SELECT ... FROM products WHERE category_id = ?', [categoryId]);
}, 300);  // 5분 TTL
```

&nbsp;

**After:**

&nbsp;

```
응답 시간: 5ms (Redis에서 바로 응답)
초당 요청: 800회 (동일)
DB CPU: 12%
DB 커넥션: 8/50
캐시 히트율: 96%
```

&nbsp;

**응답 200ms → 5ms (40배 빨라짐), DB CPU 78% → 12%, DB 부하 95% 감소.**

&nbsp;

### 사례 2: 인기 검색어 — 매번 집계 → 1분 캐시

&nbsp;

```sql
-- 최근 1시간 인기 검색어 Top 10 (매 요청마다 집계)
SELECT keyword, COUNT(*) AS cnt
FROM search_logs
WHERE created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY keyword
ORDER BY cnt DESC
LIMIT 10;
```

&nbsp;

**Before:**

&nbsp;

```
쿼리 실행 시간: 1.5초 (100만 건 집계)
초당 요청: 200회
DB CPU: 82%
```

&nbsp;

```typescript
// 1분 캐시 적용 (인기 검색어는 1분 안에 크게 안 바뀜)
const popular = await getCachedData('search:popular', () => {
    return db.query('SELECT keyword, COUNT(*) ...');
}, 60);  // 1분 TTL
```

&nbsp;

**After:**

&nbsp;

```
응답 시간: 3ms (캐시 히트)
초당 요청: 200회 (동일)
DB CPU: 15%
실제 DB 집계: 1분에 1번만
```

&nbsp;

**DB CPU 82% → 15%. 집계 쿼리가 초당 200번에서 분당 1번으로 감소.**

&nbsp;

&nbsp;

---

&nbsp;

## 7. 캐시하면 안 되는 것

&nbsp;

모든 데이터를 캐시할 수 있는 건 아니다.

&nbsp;

| 데이터 | 캐시 가능 여부 | 이유 |
|--------|----------------|------|
| 상품 목록 | O | 잠깐 옛날 데이터 보여도 괜찮음 |
| 카테고리 | O | 거의 안 바뀜 |
| 인기 검색어 | O | 1분 오차 허용 |
| **계좌 잔액** | **X** | 실시간 정확성 필수 |
| **재고 수량** | **X** | 0개인데 "있음"으로 보이면 장애 |
| **결제 상태** | **X** | 중복 결제 위험 |
| **인증 토큰** | 조건부 | 캐시하되 즉시 무효화 가능해야 함 |

&nbsp;

```
핵심 판단 기준:
  "이 데이터가 10초 동안 옛날 값이어도 괜찮은가?"

  괜찮다 → 캐시 가능
  안 괜찮다 → 캐시하면 안 됨
```

&nbsp;

&nbsp;

---

&nbsp;

## 8. 캐시 모니터링

&nbsp;

캐시를 넣었으면 모니터링해야 한다.

&nbsp;

```bash
# Redis CLI로 상태 확인
redis-cli INFO stats

# 핵심 지표
keyspace_hits:   4,521,340   # 캐시 히트
keyspace_misses: 231,500     # 캐시 미스

# 히트율 = hits / (hits + misses) = 95.1%
```

&nbsp;

| 히트율 | 상태 |
|--------|------|
| 95% 이상 | 잘 되고 있음 |
| 80~95% | 보통, TTL 조정 검토 |
| 80% 미만 | 뭔가 잘못됨. TTL이 너무 짧거나 키 설계가 잘못됨 |

&nbsp;

```bash
# 메모리 사용량 확인
redis-cli INFO memory
# used_memory_human: 256.50M
# maxmemory_human: 1.00G

# 키 개수 확인
redis-cli DBSIZE
# (integer) 45230
```

&nbsp;

&nbsp;

---

&nbsp;

다음 편 예고: **테스트 코드 — 안 짜면 어떻게 되는지 겪어본 이야기**

마지막 편은 DB 성능이 아니다. "개발 생산성 성능"이다. 테스트 없이 개발하면 수정이 무섭고, 배포가 무섭고, 리팩토링이 불가능하다. 테스트 한 줄이 개발 속도를 어떻게 바꾸는지, 실제 경험을 기반으로 정리한다.

&nbsp;

&nbsp;

---

캐시, Redis, Cache-Aside, TTL, 캐시무효화, 캐시스탬피드, ioredis, Spring Cache, Cacheable, CacheEvict, 성능최적화, DB부하, 메모리캐시, 쿼리캐시, 히트율