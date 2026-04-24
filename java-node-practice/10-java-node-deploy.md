# [Java vs Node.js 5편] JAR 하나 vs node dist/index.js — 배포와 운영의 현실적 차이

&nbsp;

Java는 `java -jar app.jar`로 끝난다. JVM만 있으면 어디서든 돌아간다.

Node.js는 `node dist/index.js`로 끝난다. Node.js만 있으면 어디서든 돌아간다.

&nbsp;

비슷해 보이지만, **빌드 과정, 이미지 크기, 시작 시간, 메모리 사용량** 전부 다르다.

&nbsp;

&nbsp;

---

&nbsp;

## 1. 환경 설정 — Profile vs dotenv

&nbsp;

### Java: application-{profile}.yml

```yaml
# application.yml (공통)
spring:
  application:
    name: my-service

# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://dev-db:3306/mydb
    username: dev_user
    password: dev_pass

# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/mydb
    username: prod_user
    password: ${DB_PASSWORD}  # 환경변수 참조
```

```bash
# 실행 시 프로필 지정
java -jar app.jar --spring.profiles.active=prod
```

&nbsp;

Spring은 **프로필 시스템이 프레임워크에 내장**되어 있다. `application-{profile}.yml` 파일만 만들면 자동으로 로딩된다.

&nbsp;

### Node.js: .env.{profile} + dotenv

```bash
# .env.local
DB_TYPE=sqlite
DB_DATABASE=dev.sqlite
API_URL=http://localhost:3000

# .env.prod
DB_TYPE=mssql
DB_HOST=192.168.0.97
DB_PORT=1443
DB_USERNAME=admin
DB_PASSWORD=secret
API_URL=https://api.example.com
```

```typescript
// config/index.ts
import dotenv from 'dotenv';

const profile = process.env.PROFILE || 'local';
dotenv.config({ path: `.env.${profile}` });

export const config = {
  db: {
    type: process.env.DB_TYPE as 'sqlite' | 'mssql',
    host: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT || '1433'),
  },
  apiUrl: process.env.API_URL,
};
```

```bash
# 실행
PROFILE=prod node dist/index.js
```

&nbsp;

Node.js는 프레임워크가 환경 분리를 해주지 않는다. **dotenv + 스크립트로 직접 구현한다.**

&nbsp;

Next.js는 자체 환경변수 시스템이 있다:

```bash
# .env.local → 자동 로딩 (gitignore 대상)
# .env.production → next build 시 자동 로딩
# .env.development → next dev 시 자동 로딩

# 빌드 전 환경 복사 스크립트
node scripts/set-env.js prod  # .env.prod → .env.local 복사
npm run build
```

&nbsp;

| 환경 설정 | Java | Node.js |
|:---|:---|:---|
| 프로필 시스템 | 프레임워크 내장 | dotenv + 수동 관리 |
| 설정 파일 | YAML/Properties | .env 파일 |
| 프로필 전환 | --spring.profiles.active | PROFILE 환경변수 |
| 설정 병합 | 자동 (공통 + 프로필) | 수동 (파일 하나만 로딩) |
| 설정값 검증 | @ConfigurationProperties + @Validated | 직접 구현 (Zod 등) |

&nbsp;

&nbsp;

---

&nbsp;

## 2. 빌드 — Fat JAR vs 번들

&nbsp;

### Java: Gradle/Maven → Fat JAR

```bash
# Gradle 빌드
./gradlew bootJar

# 결과물
build/libs/app-1.0.0.jar  # 50~100MB (의존성 전부 포함)
```

```groovy
// build.gradle
plugins {
    id 'org.springframework.boot' version '3.2.0'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.mysql:mysql-connector-j'
}
```

&nbsp;

Fat JAR 하나에 **모든 의존성이 다 들어있다.** 이 파일 하나만 있으면 어디서든 실행 가능.

&nbsp;

### Node.js: tsc/esbuild → dist/

```bash
# TypeScript 컴파일
npx tsc

# 결과물
dist/
├── index.js
├── routes/
├── services/
└── config/

node_modules/  # 이게 문제. 수백 MB
```

```json
// package.json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

&nbsp;

Node.js의 문제: **node_modules가 별도로 필요하다.** Java의 Fat JAR처럼 하나로 묶이지 않는다.

&nbsp;

해결 방법:
```bash
# 프로덕션 의존성만 설치
npm ci --omit=dev

# 또는 esbuild로 번들링 (node_modules 없이 실행 가능)
npx esbuild src/index.ts --bundle --platform=node --outfile=dist/index.js
```

&nbsp;

Next.js standalone 모드:

```javascript
// next.config.js
module.exports = {
  output: 'standalone', // 필요한 node_modules만 복사
};
```

```bash
# 결과물
.next/standalone/
├── server.js           # 엔트리포인트
├── node_modules/       # 최소 의존성만
└── .next/              # 빌드 결과
```

&nbsp;

&nbsp;

---

&nbsp;

## 3. Docker — 이미지 크기 차이

&nbsp;

### Java Dockerfile

```dockerfile
# 멀티스테이지 빌드
FROM gradle:8.5-jdk21 AS builder
WORKDIR /app
COPY . .
RUN gradle bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/app-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

&nbsp;

### Node.js Dockerfile

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

&nbsp;

### 이미지 크기 비교

| 항목 | Java | Node.js |
|:---|:---|:---|
| 베이스 이미지 | eclipse-temurin:21-jre-alpine (150MB) | node:20-alpine (50MB) |
| 앱 크기 | JAR 50~100MB | dist + node_modules 30~80MB |
| **최종 이미지** | **200~300MB** | **80~150MB** |
| 최적화 시 | GraalVM native: 50~100MB | esbuild 번들: 50~80MB |

&nbsp;

Node.js가 이미지 크기에서 **확실히 가볍다.** JVM 자체가 무겁기 때문이다.

&nbsp;

&nbsp;

---

&nbsp;

## 4. 시작 시간 — 가장 체감되는 차이

&nbsp;

```bash
# Java Spring Boot
$ time java -jar app.jar
Started MyApplication in 12.345 seconds
# 10~30초 (DB 마이그레이션 포함 시 더 길어짐)

# Node.js
$ time node dist/index.js
Server started on port 3000
# 0.5~2초
```

&nbsp;

| 항목 | Java | Node.js |
|:---|:---|:---|
| 콜드 스타트 | 10~30초 | 0.5~2초 |
| 원인 | JVM 초기화, 클래스 로딩, DI 컨테이너, 빈 스캔 | 모듈 로딩만 |
| 핫 리로드 | Spring DevTools (3~5초) | nodemon/tsx (1초 미만) |
| 개발 체감 | 코드 수정 → 5초 대기 | 코드 수정 → 즉시 |

&nbsp;

개발 중 생산성은 **Node.js가 압도적이다.** 코드 한 줄 고치고 5초 기다리는 것과 1초도 안 기다리는 것의 차이는 하루 종일 누적된다.

&nbsp;

&nbsp;

---

&nbsp;

## 5. 메모리 사용량

&nbsp;

```bash
# Java — 아무것도 안 하고 올리기만 해도
$ java -jar app.jar
# RSS: 300~500MB (기본)
# -Xmx256m 설정해도 300MB 전후

# Node.js — 아무것도 안 하고 올리기만 해도
$ node dist/index.js
# RSS: 30~60MB
```

&nbsp;

| 항목 | Java | Node.js |
|:---|:---|:---|
| 기본 메모리 | 300~500MB | 30~60MB |
| 부하 시 | 500MB~2GB | 100~300MB |
| 메모리 제어 | -Xms, -Xmx | --max-old-space-size |
| GC | G1GC, ZGC (튜닝 옵션 풍부) | V8 GC (튜닝 제한적) |

&nbsp;

같은 서버에 **Node.js는 5~10배 더 많은 인스턴스를 띄울 수 있다.** 마이크로서비스에서 인스턴스 수가 많을수록 이 차이가 비용으로 직결된다.

&nbsp;

&nbsp;

---

&nbsp;

## 6. 프로세스 관리

&nbsp;

### Java: systemd

```ini
# /etc/systemd/system/my-service.service
[Unit]
Description=My Java Service
After=network.target

[Service]
Type=simple
User=app
ExecStart=/usr/bin/java -jar /opt/app/app.jar --spring.profiles.active=prod
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

&nbsp;

### Node.js: PM2

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'my-service',
    script: 'dist/index.js',
    instances: 'max',        // CPU 코어 수만큼
    exec_mode: 'cluster',    // 클러스터 모드
    env_production: {
      NODE_ENV: 'production',
      PROFILE: 'prod',
    },
    max_memory_restart: '500M',
  }],
};
```

```bash
pm2 start ecosystem.config.js --env production
pm2 save       # 재부팅 시 자동 시작
pm2 startup    # 시스템 서비스 등록
```

&nbsp;

### Windows: NSSM

```batch
:: Java
nssm install MyJavaService "C:\Program Files\Java\jdk-21\bin\java.exe"
nssm set MyJavaService AppParameters "-jar D:\server\app.jar"

:: Node.js
nssm install MyNodeService "C:\Program Files\nodejs\node.exe"
nssm set MyNodeService AppParameters "D:\server\dist\index.js"
nssm set MyNodeService AppDirectory "D:\server"
```

&nbsp;

| 프로세스 관리 | Java | Node.js |
|:---|:---|:---|
| Linux | systemd | PM2 또는 systemd |
| Windows | NSSM | NSSM 또는 PM2 |
| 클러스터링 | 외부 (Nginx, K8s) | PM2 cluster 또는 Node.js cluster |
| 자동 재시작 | systemd Restart=always | PM2 자동 |
| 로그 관리 | Log4j2, Logback | PM2 log 또는 winston |

&nbsp;

&nbsp;

---

&nbsp;

## 7. 서버리스 — 콜드 스타트가 갈린다

&nbsp;

```
AWS Lambda 콜드 스타트:
├── Java (Spring):     3~15초 ← 치명적
├── Java (GraalVM):    0.1~0.5초 ← 네이티브 이미지
├── Node.js:           0.1~0.5초 ← 기본적으로 빠름
└── Node.js (esbuild): 0.05~0.2초 ← 번들링 시 더 빠름
```

&nbsp;

Java의 서버리스 콜드 스타트는 **사용자가 체감할 수 있는 수준이다.** API Gateway 타임아웃(29초)에 걸릴 수도 있다.

&nbsp;

### GraalVM Native Image — Java의 해결책

```bash
# GraalVM으로 네이티브 바이너리 생성
./gradlew nativeCompile

# 결과물: 단일 실행 파일, JVM 불필요
./build/native/nativeCompile/app

# 시작 시간: 0.05~0.1초 (JVM 대비 100배 빠름)
# 메모리: 50~100MB (JVM 대비 5배 적음)
```

&nbsp;

하지만 GraalVM Native Image의 한계:
- 빌드 시간이 매우 길다 (5~15분)
- 리플렉션, 동적 프록시 등에 제한이 있어 Spring 일부 기능이 안 됨
- 아직 모든 라이브러리가 호환되지 않음
- Spring Boot 3.0+에서 공식 지원하지만, 실무 적용 사례는 제한적

&nbsp;

&nbsp;

---

&nbsp;

## 8. 모니터링 — Actuator vs 직접 구현

&nbsp;

### Java: Spring Actuator (올인원)

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: always
```

```bash
# 바로 사용 가능
GET /actuator/health
GET /actuator/metrics
GET /actuator/prometheus  # Grafana 연동
```

&nbsp;

의존성 하나 추가하면 **헬스체크, 메트릭, 프로메테우스 익스포터**가 자동으로 생긴다.

&nbsp;

### Node.js: 직접 구현

```typescript
// 헬스체크
app.get('/health', async (req, res) => {
  const health = {
    status: 'ok',
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
    memory: process.memoryUsage(),
    checks: {
      db: await checkDatabase(),
    },
  };

  const isHealthy = health.checks.db;
  res.status(isHealthy ? 200 : 503).json(health);
});

async function checkDatabase(): Promise<boolean> {
  try {
    await dataSource.query('SELECT 1');
    return true;
  } catch {
    return false;
  }
}
```

```typescript
// 메트릭 (prom-client)
import client from 'prom-client';

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
});

app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    end({ method: req.method, route: req.path, status: res.statusCode });
  });
  next();
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});
```

&nbsp;

| 모니터링 | Java | Node.js |
|:---|:---|:---|
| 헬스체크 | Actuator 자동 | 직접 구현 |
| 메트릭 | Micrometer 자동 | prom-client 수동 |
| 프로메테우스 | 의존성 추가만 | prom-client + 미들웨어 |
| 구현 비용 | 거의 없음 | 중간 |

&nbsp;

&nbsp;

---

&nbsp;

## 9. CI/CD 파이프라인 비교

&nbsp;

### Java

```yaml
# GitHub Actions
name: Java CI/CD

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build with Gradle
        run: ./gradlew bootJar

      - name: Build Docker image
        run: docker build -t my-service:${{ github.sha }} .

      - name: Push to registry
        run: docker push my-service:${{ github.sha }}
```

&nbsp;

### Node.js

```yaml
# GitHub Actions
name: Node.js CI/CD

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Build Docker image
        run: docker build -t my-service:${{ github.sha }} .

      - name: Push to registry
        run: docker push my-service:${{ github.sha }}
```

&nbsp;

CI/CD 파이프라인 자체는 거의 동일하다. 차이는 **빌드 시간**:

| CI 단계 | Java | Node.js |
|:---|:---|:---|
| 의존성 설치 | Gradle 캐시 활용 (30초~2분) | npm ci (10초~1분) |
| 컴파일 | Gradle bootJar (1~3분) | tsc (5~30초) |
| Docker 빌드 | 멀티스테이지 (2~5분) | 멀티스테이지 (1~2분) |
| **총 CI 시간** | **3~10분** | **1~3분** |

&nbsp;

&nbsp;

---

&nbsp;

## 10. 어떤 걸 선택할까

&nbsp;

```
Node.js가 유리한 경우:
├── 빠른 시작/배포가 중요 (서버리스, 스타트업)
├── 메모리/비용 제한이 있음 (소규모 인프라)
├── 프론트엔드와 언어 통일 (풀스택 TypeScript)
├── 실시간 통신 (WebSocket, SSE)
├── 마이크로서비스가 많음 (인스턴스 수 x 메모리)
└── 개발 속도 우선 (핫 리로드, 적은 보일러플레이트)

Java가 유리한 경우:
├── 엔터프라이즈 안정성 (금융, 대기업)
├── 복잡한 비즈니스 로직 (DDD, 대규모 도메인)
├── 멀티스레드 처리 (CPU 집약적 작업)
├── 성숙한 에코시스템 필요 (Spring Security, Batch 등)
├── 팀에 Java 경험이 풍부
└── JVM 생태계 (Kotlin, Scala와 혼용)
```

&nbsp;

&nbsp;

---

&nbsp;

## 최종 비교표

&nbsp;

| 항목 | Java/Spring | Node.js |
|:---|:---|:---|
| 빌드 결과물 | Fat JAR (50~100MB) | dist/ + node_modules |
| Docker 이미지 | 200~300MB | 80~150MB |
| 시작 시간 | 10~30초 | 0.5~2초 |
| 메모리 (기본) | 300~500MB | 30~60MB |
| 환경 설정 | Spring Profile (내장) | dotenv (수동) |
| 프로세스 관리 | systemd | PM2 / systemd |
| 모니터링 | Actuator (자동) | 직접 구현 |
| 서버리스 | 콜드 스타트 문제 | 즉시 시작 |
| CI 빌드 시간 | 3~10분 | 1~3분 |
| 운영 안정성 | 검증됨 (20년+) | 검증됨 (10년+) |

&nbsp;

**작고 빠르면 Node.js, 크고 안정적이면 Java.**

하지만 이건 단순화한 결론이다. 실제로는 **팀의 경험, 기존 인프라, 비즈니스 요구사항**이 기술 선택보다 더 중요하다.

두 가지를 다 할 줄 아는 게 최선이다. 그래야 상황에 맞는 도구를 고를 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

이것으로 Java vs Node.js 대응 시리즈를 마친다.

&nbsp;

| 편 | 주제 |
|:---|:---|
| 1편 | DI와 레이어 구조 |
| 2편 | AOP와 미들웨어 |
| 3편 | 검증과 보안 |
| 4편 | ORM (JPA vs TypeORM/Prisma) |
| 5편 | 배포와 운영 |

&nbsp;

어느 쪽이 더 좋다는 게 아니다. **같은 문제를 다른 방식으로 푸는 두 가지 도구**를 이해하는 게 목표였다. 둘 다 써본 사람은, 새 프로젝트에서 "이건 Node.js가 낫겠다" 또는 "이건 Java가 맞겠다"를 자연스럽게 판단할 수 있게 된다.

&nbsp;

&nbsp;

---

Java, Node.js, Spring, 배포, Docker, 컨테이너, 서버리스, PM2, GraalVM, CI/CD, 콜드스타트, 메모리, 모니터링, Actuator, TypeScript