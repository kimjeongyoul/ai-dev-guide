# 클라우드 비용 비교 — AWS, NCP, GCP, Azure 트래픽 규모별 실전 견적

&nbsp;

클라우드를 선택할 때 가장 먼저 드는 질문이 있다. **"한 달에 얼마나 나올까?"**

공식 가격표를 보면 항목이 수십 개다. 인스턴스, 스토리지, 네트워크 전송, IOPS, 로드밸런서, CDN... 하나하나 계산하다 보면 오후가 지나간다. 그래서 이 글에서는 실제 서비스에서 흔히 쓰는 아키텍처를 기준으로, 트래픽 규모별로 4개 클라우드의 월 비용을 정리했다.

환율은 **$1 = ₩1,430** 기준이다. NCP는 원화 과금이므로 달러 환산을 괄호로 표기한다.

&nbsp;

---

&nbsp;

## 1. 기본 아키텍처

&nbsp;

이 글에서 비교하는 아키텍처는 다음과 같다:

```
┌──────────┐     ┌──────────┐     ┌──────────────┐     ┌──────────┐
│   CDN    │────►│    LB    │────►│   Backend    │────►│ MariaDB  │
│ (정적파일) │     │ (L7 LB)  │     │ (Node.js or  │     │ (RDS /   │
│          │     │          │     │  Spring Boot)│     │ Managed) │
└──────────┘     └──────────┘     └──────┬───────┘     └──────────┘
                                         │
                                         ▼
                                  ┌──────────┐
                                  │  Redis   │
                                  │ (캐시/세션)│
                                  └──────────┘
```

&nbsp;

**구성 요소:**

| 계층 | 역할 | 서비스 예시 |
|:---:|:---|:---|
| CDN | 정적 파일 (JS, CSS, 이미지) 배포 | CloudFront, CDN+, Cloud CDN, Azure CDN |
| LB | HTTPS 종단, 트래픽 분산 | ALB, NCP LB, Cloud LB, Azure LB |
| Backend | API 서버 (Node.js 또는 Spring Boot) | EC2, NCP Server, GCE, Azure VM |
| DB | MariaDB (Managed) | RDS, Cloud DB, Cloud SQL, Azure DB |
| Redis | 세션, 캐시, Rate Limiting | ElastiCache, Cloud Redis, Memorystore, Azure Cache |

&nbsp;

모든 규모에서 **CDN 전송량 월 100GB**, **네트워크 아웃바운드 월 100GB**를 기준으로 한다. 실제로는 트래픽 패턴에 따라 크게 달라질 수 있다.

&nbsp;

---

&nbsp;

## 2. 일 10만 건 — 소규모

&nbsp;

스타트업 초기, 사내 서비스, MVP 단계. 서버 한 대로 충분한 규모다.

&nbsp;

### 공통 스펙

| 항목 | 스펙 |
|:---:|:---|
| Backend | 2 vCPU, 4GB RAM × 1대 |
| DB | 2 vCPU, 4GB RAM, 50GB SSD |
| Redis | 1GB, 단일 노드 |
| CDN | 100GB 전송 |
| LB | 1개 |

&nbsp;

### AWS

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend | t3.medium (On-Demand) | $30 (약 4.3만원) |
| DB | RDS db.t3.medium, 50GB gp3 | $52 (약 7.4만원) |
| Redis | ElastiCache cache.t3.micro | $12 (약 1.7만원) |
| LB | ALB | $22 (약 3.1만원) |
| CDN | CloudFront 100GB | $9 (약 1.3만원) |
| **합계** | | **$125 (약 17.9만원)** |

&nbsp;

### NCP (네이버 클라우드)

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend | Standard g3 (2vCPU, 4GB) | ₩44,000 (약 $31) |
| DB | Cloud DB for MySQL (2vCPU, 4GB) | ₩65,000 (약 $45) |
| Redis | Cloud Redis (1GB) | ₩22,000 (약 $15) |
| LB | Load Balancer | ₩18,000 (약 $13) |
| CDN | CDN+ 100GB | ₩5,000 (약 $3) |
| **합계** | | **₩154,000 (약 $108)** |

&nbsp;

### GCP

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend | e2-medium (On-Demand) | $25 (약 3.6만원) |
| DB | Cloud SQL db-custom-2-4096, 50GB | $51 (약 7.3만원) |
| Redis | Memorystore Basic 1GB | $35 (약 5.0만원) |
| LB | Cloud Load Balancing | $20 (약 2.9만원) |
| CDN | Cloud CDN 100GB | $9 (약 1.3만원) |
| **합계** | | **$140 (약 20.0만원)** |

&nbsp;

### Azure

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend | B2s (2vCPU, 4GB) | $35 (약 5.0만원) |
| DB | Azure DB for MariaDB Basic (2vCPU) | $58 (약 8.3만원) |
| Redis | Azure Cache Basic C0 (250MB) | $16 (약 2.3만원) |
| LB | Application Gateway | $25 (약 3.6만원) |
| CDN | Azure CDN 100GB | $10 (약 1.4만원) |
| **합계** | | **$144 (약 20.6만원)** |

&nbsp;

> 소규모에서는 클라우드 간 차이가 월 2~3만원 수준이다. **NCP가 가장 저렴**하고, AWS와 GCP가 비슷하며, Azure가 약간 비싸다. 이 규모에서는 비용보다 **익숙한 플랫폼**을 고르는 게 낫다.

&nbsp;

---

&nbsp;

## 3. 일 100만 건 — 중규모

&nbsp;

본격적으로 트래픽이 들어오는 단계. Backend 2대에 Redis를 캐시로 적극 활용해야 한다.

&nbsp;

### 공통 스펙

| 항목 | 스펙 |
|:---:|:---|
| Backend | 4 vCPU, 8GB RAM × 2대 |
| DB | 4 vCPU, 16GB RAM, 100GB SSD |
| Redis | 4GB, 단일 노드 |
| CDN | 500GB 전송 |
| LB | 1개 |

&nbsp;

### AWS

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 2 | t3.xlarge | $120 (약 17.2만원) |
| DB | RDS db.r6g.large, 100GB gp3 | $150 (약 21.5만원) |
| Redis | ElastiCache cache.r6g.large (4GB) | $95 (약 13.6만원) |
| LB | ALB | $30 (약 4.3만원) |
| CDN | CloudFront 500GB | $43 (약 6.1만원) |
| **합계** | | **$438 (약 62.6만원)** |

&nbsp;

### NCP

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 2 | Standard g3 (4vCPU, 8GB) × 2 | ₩176,000 (약 $123) |
| DB | Cloud DB for MySQL (4vCPU, 16GB) | ₩198,000 (약 $138) |
| Redis | Cloud Redis (4GB) | ₩66,000 (약 $46) |
| LB | Load Balancer | ₩25,000 (약 $17) |
| CDN | CDN+ 500GB | ₩20,000 (약 $14) |
| **합계** | | **₩485,000 (약 $339)** |

&nbsp;

### GCP

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 2 | e2-standard-4 × 2 | $97 (약 13.9만원) |
| DB | Cloud SQL db-custom-4-16384, 100GB | $175 (약 25.0만원) |
| Redis | Memorystore Basic 4GB | $110 (약 15.7만원) |
| LB | Cloud Load Balancing | $25 (약 3.6만원) |
| CDN | Cloud CDN 500GB | $40 (약 5.7만원) |
| **합계** | | **$447 (약 63.9만원)** |

&nbsp;

### Azure

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 2 | D4s v5 (4vCPU, 16GB) × 2 | $140 (약 20.0만원) |
| DB | Azure DB for MariaDB GP (4vCPU) | $195 (약 27.9만원) |
| Redis | Azure Cache Standard C2 (6GB) | $100 (약 14.3만원) |
| LB | Application Gateway | $30 (약 4.3만원) |
| CDN | Azure CDN 500GB | $40 (약 5.7만원) |
| **합계** | | **$505 (약 72.2만원)** |

&nbsp;

> 중규모부터 NCP의 가격 경쟁력이 눈에 띈다. AWS 대비 **약 46% 저렴**. GCP의 인스턴스 가격이 가장 낮지만, Managed Redis(Memorystore)가 비싸서 총합은 AWS와 비슷하다.

&nbsp;

---

&nbsp;

## 4. 일 1,000만 건 — 대규모

&nbsp;

하루 1,000만 건이면 초당 약 115건. 피크 타임에는 초당 500건 이상 나올 수 있다. DB Read Replica와 Redis 클러스터가 필수다.

&nbsp;

### 공통 스펙

| 항목 | 스펙 |
|:---:|:---|
| Backend | 4 vCPU, 16GB RAM × 4대 |
| DB Primary | 8 vCPU, 32GB RAM, 500GB SSD |
| DB Read Replica | 4 vCPU, 16GB RAM × 1대 |
| Redis | 클러스터 모드, 3노드, 각 4GB |
| CDN | 2TB 전송 |
| LB | 1개 |

&nbsp;

### AWS

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 4 | m6i.xlarge × 4 | $560 (약 80.1만원) |
| DB Primary | RDS db.r6g.2xlarge, 500GB gp3 | $520 (약 74.4만원) |
| DB Read Replica | RDS db.r6g.large | $130 (약 18.6만원) |
| Redis 클러스터 | ElastiCache 3노드 r6g.large | $285 (약 40.7만원) |
| LB | ALB | $45 (약 6.4만원) |
| CDN | CloudFront 2TB | $170 (약 24.3만원) |
| **합계** | | **$1,710 (약 244.5만원)** |

&nbsp;

### NCP

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 4 | Standard g3 (4vCPU, 16GB) × 4 | ₩560,000 (약 $392) |
| DB Primary | Cloud DB for MySQL (8vCPU, 32GB) | ₩495,000 (약 $346) |
| DB Read Replica | Cloud DB (4vCPU, 16GB) | ₩198,000 (약 $138) |
| Redis 클러스터 | Cloud Redis HA 3노드 (4GB) | ₩264,000 (약 $185) |
| LB | Load Balancer | ₩30,000 (약 $21) |
| CDN | CDN+ 2TB | ₩65,000 (약 $45) |
| **합계** | | **₩1,612,000 (약 $1,127)** |

&nbsp;

### GCP

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 4 | e2-standard-4 × 4 | $390 (약 55.8만원) |
| DB Primary | Cloud SQL db-custom-8-32768, 500GB | $525 (약 75.1만원) |
| DB Read Replica | Cloud SQL db-custom-4-16384 | $175 (약 25.0만원) |
| Redis 클러스터 | Memorystore Standard HA 3노드 | $320 (약 45.8만원) |
| LB | Cloud Load Balancing | $30 (약 4.3만원) |
| CDN | Cloud CDN 2TB | $150 (약 21.5만원) |
| **합계** | | **$1,590 (약 227.4만원)** |

&nbsp;

### Azure

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 4 | D4s v5 × 4 | $560 (약 80.1만원) |
| DB Primary | Azure DB for MySQL (8vCPU, 32GB) | $580 (약 82.9만원) |
| DB Read Replica | Azure DB (4vCPU, 16GB) | $195 (약 27.9만원) |
| Redis 클러스터 | Azure Cache Premium P2 (3노드) | $450 (약 64.4만원) |
| LB | Application Gateway WAF v2 | $55 (약 7.9만원) |
| CDN | Azure CDN 2TB | $160 (약 22.9만원) |
| **합계** | | **$2,000 (약 286.0만원)** |

&nbsp;

> 대규모에서 클라우드 간 차이가 확실히 벌어진다. NCP가 AWS 대비 **약 34% 저렴**. GCP는 인스턴스에서 절감하지만 Memorystore가 여전히 비싸다. Azure는 Redis Premium 과금이 치명적.

&nbsp;

---

&nbsp;

## 5. 일 1억 건 — 엔터프라이즈

&nbsp;

초당 1,150건. 피크 타임 초당 5,000건 이상. Auto Scaling, HA DB, 메시지 큐가 필수인 규모다.

&nbsp;

### 공통 스펙

| 항목 | 스펙 |
|:---:|:---|
| Backend | 8 vCPU, 32GB RAM × 8대 + Auto Scaling |
| DB Primary | 16 vCPU, 64GB RAM, 1TB SSD (HA 구성) |
| DB Read Replica | 8 vCPU, 32GB RAM × 2대 |
| Redis | 클러스터 모드, 6노드, 각 8GB |
| Kafka (또는 MQ) | 3 브로커 클러스터 |
| CDN | 10TB 전송 |
| LB | 1개 |

&nbsp;

### AWS

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 8 | m6i.2xlarge × 8 (+ ASG) | $2,240 (약 320.3만원) |
| DB Primary | Aurora MySQL r6g.4xlarge, Multi-AZ | $2,100 (약 300.3만원) |
| DB Read Replica × 2 | Aurora r6g.2xlarge × 2 | $1,040 (약 148.7만원) |
| Redis 클러스터 | ElastiCache 6노드 r6g.xlarge | $1,140 (약 163.0만원) |
| Kafka | Amazon MSK 3브로커 m5.large | $650 (약 93.0만원) |
| LB | ALB | $75 (약 10.7만원) |
| CDN | CloudFront 10TB | $850 (약 121.6만원) |
| **합계** | | **$8,095 (약 1,157.6만원)** |

&nbsp;

### NCP

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 8 | Standard g3 (8vCPU, 32GB) × 8 | ₩2,640,000 (약 $1,846) |
| DB Primary | Cloud DB for MySQL HA (16vCPU, 64GB) | ₩1,980,000 (약 $1,385) |
| DB Read Replica × 2 | Cloud DB (8vCPU, 32GB) × 2 | ₩990,000 (약 $692) |
| Redis 클러스터 | Cloud Redis HA 6노드 (8GB) | ₩792,000 (약 $554) |
| Kafka | Cloud Hadoop (Kafka 3브로커) | ₩650,000 (약 $455) |
| LB | Load Balancer | ₩45,000 (약 $31) |
| CDN | CDN+ 10TB | ₩250,000 (약 $175) |
| **합계** | | **₩7,347,000 (약 $5,138)** |

&nbsp;

### GCP

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 8 | e2-standard-8 × 8 (+ MIG) | $1,560 (약 223.1만원) |
| DB Primary | Cloud SQL HA db-custom-16-65536, 1TB | $2,200 (약 314.6만원) |
| DB Read Replica × 2 | Cloud SQL db-custom-8-32768 × 2 | $1,050 (약 150.2만원) |
| Redis 클러스터 | Memorystore Standard HA 6노드 | $1,280 (약 183.0만원) |
| Kafka | Confluent on GCP (3브로커) | $700 (약 100.1만원) |
| LB | Cloud Load Balancing | $50 (약 7.2만원) |
| CDN | Cloud CDN 10TB | $750 (약 107.3만원) |
| **합계** | | **$7,590 (약 1,085.4만원)** |

&nbsp;

### Azure

| 항목 | 서비스 | 월 비용 |
|:---:|:---|---:|
| Backend × 8 | D8s v5 × 8 (+ VMSS) | $2,240 (약 320.3만원) |
| DB Primary | Azure DB for MySQL HA (16vCPU, 64GB) | $2,400 (약 343.2만원) |
| DB Read Replica × 2 | Azure DB (8vCPU, 32GB) × 2 | $1,160 (약 165.9만원) |
| Redis 클러스터 | Azure Cache Premium P3 (6노드) | $1,800 (약 257.4만원) |
| Kafka | Azure Event Hubs Premium (3CU) | $870 (약 124.4만원) |
| LB | Application Gateway WAF v2 | $80 (약 11.4만원) |
| CDN | Azure CDN 10TB | $800 (약 114.4만원) |
| **합계** | | **$9,350 (약 1,337.1만원)** |

&nbsp;

> 엔터프라이즈 규모에서는 **연간 수억 원** 차이가 난다. NCP와 GCP가 비슷한 수준이고, AWS가 그 위, Azure가 가장 비싸다. 다만 NCP는 Kafka를 Cloud Hadoop으로 대체해야 하는 등 서비스 선택지가 제한적이다.

&nbsp;

---

&nbsp;

## 6. 한눈에 비교 표

&nbsp;

### 월 비용 비교 (On-Demand 기준)

| 규모 | AWS | NCP | GCP | Azure |
|:---:|:---:|:---:|:---:|:---:|
| 일 10만 건 | $125 (17.9만원) | $108 (15.4만원) | $140 (20.0만원) | $144 (20.6만원) |
| 일 100만 건 | $438 (62.6만원) | $339 (48.5만원) | $447 (63.9만원) | $505 (72.2만원) |
| 일 1,000만 건 | $1,710 (244.5만원) | $1,127 (161.2만원) | $1,590 (227.4만원) | $2,000 (286.0만원) |
| 일 1억 건 | $8,095 (1,157.6만원) | $5,138 (734.7만원) | $7,590 (1,085.4만원) | $9,350 (1,337.1만원) |

&nbsp;

### 가격 순위 (저렴한 순)

| 순위 | 소규모 | 중규모 | 대규모 | 엔터프라이즈 |
|:---:|:---:|:---:|:---:|:---:|
| 1위 | NCP | NCP | NCP | NCP |
| 2위 | AWS | AWS | GCP | GCP |
| 3위 | GCP | GCP | AWS | AWS |
| 4위 | Azure | Azure | Azure | Azure |

&nbsp;

> NCP가 전 구간에서 가장 저렴하다. 다만 **비용만으로 클라우드를 선택하면 안 된다.** 서비스 다양성, 글로벌 리전, 생태계, 기술 지원까지 고려해야 한다.

&nbsp;

---

&nbsp;

## 7. 클라우드별 특징

&nbsp;

### AWS — 서비스가 가장 많다. 비싸다.

- 200개 이상의 서비스. 뭐든 있다
- Aurora, DynamoDB, SQS, Lambda 등 관리형 서비스 최강
- 한국 리전(서울) 존재. 다만 글로벌 리전에 비해 약간 비쌈
- 커뮤니티와 레퍼런스가 가장 풍부
- **단점:** On-Demand 가격이 비쌈. 예약 인스턴스 없이는 월말에 놀란다

&nbsp;

### NCP — 한국 리전 빠르다. 원화 결제. 저렴하다.

- 원화 결제로 환율 리스크 없음
- 한국 리전 네트워크 지연이 가장 낮음 (1~2ms)
- 정부/공공기관 사업에서 필수 (CSAP 인증)
- 기술 지원이 한국어로 빠르게 가능
- **단점:** 글로벌 리전 부족. 서비스 종류가 AWS 대비 1/3 수준. Kafka, Managed Kubernetes 등 일부 서비스가 없거나 제한적

&nbsp;

### GCP — 네트워크 빠르다. BigQuery가 강력하다.

- Google 프리미엄 네트워크. 글로벌 LB 성능 최상
- BigQuery로 대규모 분석이 간편하고 저렴
- GKE(Kubernetes)가 가장 완성도 높음
- Sustained Use Discount(지속 사용 할인) 자동 적용
- **단점:** Managed Redis(Memorystore)가 비쌈. 엔터프라이즈 기술 지원이 AWS 대비 약함

&nbsp;

### Azure — MS 생태계. 가격은 가장 비싸다.

- Active Directory, Office 365 연동이 필요하면 사실상 유일한 선택
- .NET, SQL Server 워크로드에 최적화
- 하이브리드 클라우드(Azure Arc, Azure Stack) 잘 되어 있음
- 엔터프라이즈 계약(EA) 시 대폭 할인 가능
- **단점:** On-Demand 가격이 전반적으로 가장 비쌈. Redis Premium 과금 구조가 불합리. UI/UX가 다소 복잡

&nbsp;

---

&nbsp;

## 8. 비용 절감 팁

&nbsp;

On-Demand 가격만 보면 겁이 난다. 실제로는 아래 방법으로 **30~70%** 절감할 수 있다.

&nbsp;

### 8-1. 예약 인스턴스 (Reserved Instance / Committed Use)

1년 또는 3년 약정으로 인스턴스를 예약하면 대폭 할인된다.

| 클라우드 | 1년 약정 할인 | 3년 약정 할인 |
|:---:|:---:|:---:|
| AWS (Reserved Instance) | 약 30~40% | 약 55~60% |
| NCP (약정 할인) | 약 25~30% | 약 40~50% |
| GCP (Committed Use) | 약 37% | 약 55% |
| Azure (Reserved VM) | 약 35~40% | 약 55~65% |

&nbsp;

**일 1,000만 건 기준, 1년 예약 시 월 비용 변화:**

```
AWS:   $1,710 → $1,110  (월 $600 절감, 연 $7,200 = 약 1,030만원)
NCP:   $1,127 → $820    (월 $307 절감, 연 $3,684 = 약 527만원)
GCP:   $1,590 → $1,000  (월 $590 절감, 연 $7,080 = 약 1,012만원)
Azure: $2,000 → $1,280  (월 $720 절감, 연 $8,640 = 약 1,236만원)
```

&nbsp;

### 8-2. 스팟 / 선점형 인스턴스

언제든 회수될 수 있는 대신 최대 **90%** 할인. Stateless 워크로드에 적합하다.

| 클라우드 | 이름 | 할인율 |
|:---:|:---|:---:|
| AWS | Spot Instance | 60~90% |
| NCP | - (미지원) | - |
| GCP | Spot VM (구 Preemptible) | 60~91% |
| Azure | Spot VM | 60~90% |

&nbsp;

Backend 서버 4대 중 2대를 스팟으로 돌리면:

```
AWS m6i.xlarge On-Demand: $140/월
AWS m6i.xlarge Spot:       $42/월 (70% 절감)

4대 전부 On-Demand:  $560/월
2대 On-Demand + 2대 Spot: $364/월 (35% 절감)
```

&nbsp;

### 8-3. Auto Scaling

트래픽이 일정하지 않으면 Auto Scaling이 필수다. 낮에 4대, 새벽에 1대.

```
24시간 4대 고정 운영:   $560/월
Auto Scaling (평균 2.5대): $350/월 (37% 절감)
```

&nbsp;

### 8-4. Graviton / ARM 인스턴스

AWS Graviton, GCP Tau T2A 등 ARM 기반 인스턴스는 x86 대비 **20~40%** 저렴하면서 성능은 비슷하거나 더 좋다.

```
AWS m6i.xlarge (x86):    $140/월
AWS m6g.xlarge (Graviton): $112/월 (20% 절감)
```

&nbsp;

Node.js, Java, Python 워크로드는 ARM에서 문제없이 동작한다. Docker 이미지만 multi-arch로 빌드하면 된다.

&nbsp;

### 8-5. 복합 적용 예시

일 1,000만 건 기준, AWS에서 모든 절감 기법을 적용:

```
원래:                              $1,710/월

1년 예약 (DB, Redis):             -$400
Graviton 전환 (Backend, Redis):   -$200
Auto Scaling (Backend 4 → 평균 2.5): -$150
스팟 혼합 (Backend 일부):          -$80
────────────────────────────────
절감 후:                           $880/월 (48% 절감)

연간 절감액: $9,960 (약 1,424만원)
```

&nbsp;

---

&nbsp;

## 9. 주의사항

&nbsp;

이 글의 모든 금액은 **2026년 4월 기준 대략적인 추정치**다. 실제 비용은 아래 요인에 따라 크게 달라질 수 있다.

&nbsp;

**변동 요인:**

- **네트워크 전송량:** 이 글에서는 100GB~10TB로 가정했지만, 동영상 서비스 등은 수십 TB일 수 있다
- **IOPS:** DB 스토리지의 IOPS 요구량에 따라 gp3 → io2 전환 시 비용이 2~5배 증가
- **리전:** 서울 리전과 미국 리전의 가격이 10~30% 차이남
- **데이터 전송 방향:** 같은 리전 내 전송은 무료/저렴하지만, 리전 간 전송은 비쌈
- **Managed 서비스 vs Self-managed:** RDS 대신 EC2에 직접 설치하면 30~50% 절감 가능 (운영 비용은 증가)
- **Support Plan:** AWS Business Support만 추가해도 월 비용의 10% 추가
- **환율 변동:** $1 = ₩1,430 기준이지만, ±100원 변동 시 대규모에서 월 수십만원 차이

&nbsp;

**각 클라우드 비용 계산기:**

- AWS: [https://calculator.aws](https://calculator.aws)
- NCP: [https://www.ncloud.com/charge/calc](https://www.ncloud.com/charge/calc)
- GCP: [https://cloud.google.com/products/calculator](https://cloud.google.com/products/calculator)
- Azure: [https://azure.microsoft.com/pricing/calculator](https://azure.microsoft.com/pricing/calculator)

&nbsp;

반드시 실제 워크로드 패턴으로 **직접 계산해보고** 결정하자. 이 글은 대략적인 감을 잡기 위한 참고 자료일 뿐이다.

&nbsp;

---

`#클라우드비용비교` `#AWS` `#NCP` `#GCP` `#Azure` `#인프라비용` `#클라우드견적` `#예약인스턴스` `#스팟인스턴스` `#AutoScaling` `#Graviton` `#CDN` `#Redis` `#MariaDB` `#트래픽규모별아키텍처`