# 프롬프트 체인 실전 — 이커머스 고객 문의 자동 처리 시스템 만들기

&nbsp;

쇼핑몰 고객센터에 하루 1,000건의 문의가 들어온다.

&nbsp;

"주문한 지 3일인데 아직 안 왔어요"
"색상이 사진이랑 달라요 환불해주세요"
"쿠폰 적용이 안 돼요"

&nbsp;

상담원 5명이 처리한다. 한 명당 하루 200건. 바쁘다.

&nbsp;

AI로 자동화하면 어떻게 될까?

"고객 문의 자동 처리해줘"라고 AI에게 한 번에 시키면?

&nbsp;

**실패한다.**

&nbsp;

&nbsp;

---

&nbsp;

# 1. 한 덩어리로 시키면 실패하는 이유

&nbsp;

```python
# v1: 한 번에 다 시키기
response = llm.chat(f"""
  고객 문의: "{inquiry}"
  
  이 문의를 분류하고, 주문 정보를 확인하고, 
  환불 가능 여부를 판단하고, 답변을 작성해줘.
""")
```

&nbsp;

결과:

```
"고객님의 문의에 대해 확인해보겠습니다.
 배송 관련 문의로 보이며, 환불도 가능할 수 있습니다.
 주문 내역을 확인해주시기 바랍니다."
```

&nbsp;

뭐가 문제인가:

- **분류가 모호하다** — "배송 관련으로 보이며" (확실하지 않음)
- **주문 정보를 못 봤다** — DB를 조회 안 했으니까
- **환불 판단이 틀렸다** — 정책을 모르니까 "가능할 수 있습니다"
- **답변이 뻔하다** — 구체적 정보 없이 템플릿 답변

&nbsp;

AI는 **한 번에 4가지 역할**을 시키면 다 얕게 한다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 프롬프트 체인으로 나누기

&nbsp;

한 덩어리를 **5단계**로 나눈다.

&nbsp;

```
고객 문의 (자연어)
  ↓
Step 1: 분류 → {"category": "refund", "urgency": "high"}
  ↓
Step 2: 정보 수집 → 주문 DB 조회 → {"order": {...}, "shipping": {...}}
  ↓
Step 3: 정책 판단 → 환불 규정 RAG 검색 → {"refundable": true, "reason": "..."}
  ↓
Step 4: 답변 생성 → 구체적인 맞춤 답변
  ↓
Step 5: 품질 검증 → 답변이 정책에 맞는지 확인
  ↓
최종 답변 (또는 상담원 전달)
```

&nbsp;

각 단계를 하나씩 만들어보자.

&nbsp;

&nbsp;

---

&nbsp;

# 3. Step 1 — 문의 분류

&nbsp;

AI에게 "분류만" 시킨다. 답변은 시키지 않는다.

&nbsp;

```python
classify_prompt = """
고객 문의를 분류하세요. 반드시 아래 JSON 형식으로만 응답하세요.

카테고리:
- shipping: 배송 관련 (배송 지연, 배송 조회, 배송지 변경)
- refund: 환불/반품 (상품 불량, 단순 변심, 오배송)
- product: 상품 문의 (사이즈, 색상, 재입고)
- payment: 결제 문의 (결제 실패, 쿠폰, 포인트)
- other: 기타

긴급도:
- high: 결제 오류, 오배송, 상품 파손
- medium: 배송 지연, 환불 요청
- low: 단순 문의, 재입고 문의

고객 문의: "{inquiry}"

응답 형식:
{
  "category": "카테고리",
  "sub_category": "세부 카테고리", 
  "urgency": "high/medium/low",
  "key_info": {
    "order_id": "문의에서 추출한 주문번호 (없으면 null)",
    "product_name": "언급된 상품명 (없으면 null)",
    "issue": "핵심 문제 한 줄"
  }
}
"""
```

&nbsp;

```python
# 실행
inquiry = "주문번호 ORD-2024-1234인데 3일 전에 주문했는데 아직 배송이 안 왔어요. 급하게 필요한 건데."

result = llm.chat(classify_prompt.format(inquiry=inquiry))
```

&nbsp;

```json
{
  "category": "shipping",
  "sub_category": "배송 지연",
  "urgency": "medium",
  "key_info": {
    "order_id": "ORD-2024-1234",
    "product_name": null,
    "issue": "주문 후 3일 경과, 배송 미시작"
  }
}
```

&nbsp;

**자연어 문의 → 구조화된 JSON.** 다음 단계에서 정확하게 쓸 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 4. Step 2 — 정보 수집 (DB 조회)

&nbsp;

이 단계는 AI가 아니라 **코드가 한다.** 주문번호로 DB를 조회한다.

&nbsp;

```python
# Step 1에서 추출한 주문번호로 DB 조회
async def collect_info(classification):
    order_id = classification['key_info']['order_id']
    
    if not order_id:
        return {"error": "주문번호 없음 → 상담원 연결 필요"}
    
    # 주문 정보
    order = await db.query("""
        SELECT order_id, product_name, price, status, 
               ordered_at, paid_at
        FROM orders WHERE order_id = :id
    """, {"id": order_id})
    
    # 배송 정보
    shipping = await db.query("""
        SELECT tracking_number, carrier, status,
               shipped_at, estimated_delivery
        FROM shipping WHERE order_id = :id
    """, {"id": order_id})
    
    # 고객 정보
    customer = await db.query("""
        SELECT name, grade, total_orders, 
               total_spent, joined_at
        FROM customers WHERE id = :id
    """, {"id": order['customer_id']})
    
    return {
        "order": order,
        "shipping": shipping,
        "customer": customer
    }
```

&nbsp;

```json
{
  "order": {
    "order_id": "ORD-2024-1234",
    "product_name": "나이키 에어맥스 270",
    "price": 159000,
    "status": "paid",
    "ordered_at": "2024-04-14 10:30:00"
  },
  "shipping": {
    "tracking_number": null,
    "carrier": null,
    "status": "pending",
    "shipped_at": null,
    "estimated_delivery": null
  },
  "customer": {
    "name": "김민수",
    "grade": "VIP",
    "total_orders": 47,
    "total_spent": 3250000
  }
}
```

&nbsp;

**AI가 추측하는 게 아니라 실제 데이터를 가져온다.** 이게 핵심이다.

&nbsp;

&nbsp;

---

&nbsp;

# 5. Step 3 — 정책 판단 (RAG)

&nbsp;

환불/배송 정책 문서를 검색해서 AI에게 판단시킨다.

&nbsp;

```python
# 벡터DB에서 관련 정책 검색
relevant_policies = vector_db.search(
    query=f"{classification['category']} {classification['sub_category']}",
    top_k=3
)

# 검색된 정책 예시:
# - "배송 지연 시 주문일 기준 3영업일 초과 시 고객 요청에 따라 환불 가능"
# - "VIP 고객 배송 지연 시 적립금 5,000원 자동 지급"
# - "배송 전 단계에서는 즉시 취소 가능"

judge_prompt = f"""
고객 문의 분류: {json.dumps(classification, ensure_ascii=False)}
주문/배송 정보: {json.dumps(order_info, ensure_ascii=False)}
관련 정책: {relevant_policies}

위 정보를 바탕으로 판단하세요. 반드시 JSON으로 응답하세요.

{{
  "can_resolve": true/false,
  "action": "즉시취소/환불접수/적립금지급/배송조회안내/상담원연결",
  "reason": "판단 근거 (정책 인용)",
  "compensation": {{
    "type": "적립금/쿠폰/없음",
    "amount": 금액
  }},
  "requires_human": false/true,
  "human_reason": "상담원이 필요한 이유 (필요 시)"
}}
"""

judgment = llm.chat(judge_prompt)
```

&nbsp;

```json
{
  "can_resolve": true,
  "action": "배송조회안내",
  "reason": "주문일 기준 3영업일 미초과 (4/14~4/17). 정책상 배송 지연 기준 미해당. 다만 VIP 고객이므로 적립금 지급 권장.",
  "compensation": {
    "type": "적립금",
    "amount": 5000
  },
  "requires_human": false
}
```

&nbsp;

**AI가 정책 문서를 근거로 판단한다.** 추측이 아니라 **정책 기반 판단.**

&nbsp;

&nbsp;

---

&nbsp;

# 6. Step 4 — 답변 생성

&nbsp;

분류 + 실제 데이터 + 정책 판단 = **정확한 맞춤 답변.**

&nbsp;

```python
reply_prompt = f"""
고객 정보: {classification['key_info']}
주문 정보: {json.dumps(order_info, ensure_ascii=False)}
판단 결과: {json.dumps(judgment, ensure_ascii=False)}

위 정보를 바탕으로 고객 답변을 작성하세요.

규칙:
- 고객 이름을 사용할 것
- 주문번호, 상품명 등 구체적 정보 포함
- 보상 내역이 있으면 안내
- 공감 → 상황 설명 → 조치 안내 → 추가 문의 안내 순서
- 존댓말, 200자 이내
"""

reply = llm.chat(reply_prompt)
```

&nbsp;

```
김민수 고객님, 안녕하세요.

주문번호 ORD-2024-1234 (나이키 에어맥스 270) 건에 대해 
확인하였습니다.

현재 출고 준비 중이며, 영업일 기준 1~2일 내 발송 예정입니다. 
발송 시 배송 조회 가능한 운송장 번호를 문자로 안내드리겠습니다.

기다려주신 점 감사드리며, VIP 고객님께 감사의 의미로 
적립금 5,000원을 지급해드렸습니다.

추가 문의 사항이 있으시면 언제든 연락주세요.
```

&nbsp;

**v1(한 덩어리)의 "주문 내역을 확인해주시기 바랍니다"와 비교해보자.**

차원이 다르다.

&nbsp;

&nbsp;

---

&nbsp;

# 7. Step 5 — 품질 검증

&nbsp;

답변을 보내기 전에 **다른 AI가 검증**한다.

&nbsp;

```python
verify_prompt = f"""
고객 문의: "{inquiry}"
생성된 답변: "{reply}"
적용된 정책: {judgment['reason']}
주문 정보: {json.dumps(order_info, ensure_ascii=False)}

아래 항목을 검증하세요. JSON으로 응답하세요.

{{
  "factually_correct": true/false,
  "policy_compliant": true/false,
  "tone_appropriate": true/false,
  "contains_sensitive_info": false/true,
  "issues": ["문제점 목록"],
  "approved": true/false
}}
"""

verification = llm.chat(verify_prompt)
```

&nbsp;

```json
{
  "factually_correct": true,
  "policy_compliant": true,
  "tone_appropriate": true,
  "contains_sensitive_info": false,
  "issues": [],
  "approved": true
}
```

&nbsp;

```python
# 검증 통과하면 자동 발송, 실패하면 상담원에게 전달
if verification['approved']:
    await send_reply(customer_id, reply)
    await log_auto_resolved(inquiry_id, classification, judgment, reply)
else:
    await escalate_to_agent(inquiry_id, {
        "inquiry": inquiry,
        "draft_reply": reply,
        "issues": verification['issues'],
        "order_info": order_info
    })
```

&nbsp;

**AI가 만든 답변을 다른 AI가 검증.** 자기 답변을 자기가 검증하는 것보다 정확하다.

&nbsp;

&nbsp;

---

&nbsp;

# 8. 전체 파이프라인 코드

&nbsp;

```python
async def handle_inquiry(inquiry: str, customer_id: str):
    
    # Step 1: 분류 (AI)
    classification = await classify(inquiry)
    
    # Step 2: 정보 수집 (DB)
    order_info = await collect_info(classification)
    if order_info.get('error'):
        return await escalate_to_agent(inquiry, order_info['error'])
    
    # Step 3: 정책 판단 (AI + RAG)
    judgment = await judge(classification, order_info)
    if judgment['requires_human']:
        return await escalate_to_agent(inquiry, judgment['human_reason'])
    
    # Step 4: 답변 생성 (AI)
    reply = await generate_reply(classification, order_info, judgment)
    
    # Step 5: 품질 검증 (AI)
    verification = await verify(inquiry, reply, judgment, order_info)
    
    if verification['approved']:
        await send_reply(customer_id, reply)
        
        # 보상이 있으면 자동 처리
        if judgment['compensation']['type'] == '적립금':
            await add_points(customer_id, judgment['compensation']['amount'])
        
        return {"status": "auto_resolved", "reply": reply}
    else:
        # 검증 실패 → 상담원에게 초안과 함께 전달
        return await escalate_to_agent(inquiry, {
            "draft": reply,
            "issues": verification['issues']
        })
```

&nbsp;

```
전체 흐름:
고객 문의 → 분류(AI) → DB 조회(코드) → 정책 판단(AI+RAG) 
→ 답변 생성(AI) → 품질 검증(AI) → 자동 발송 or 상담원 전달

소요 시간: 3~5초
상담원 처리 시간: 5~10분
```

&nbsp;

&nbsp;

---

&nbsp;

# 9. 각 단계에서 전달되는 데이터

&nbsp;

```
Step 1 출력 (JSON):
  {"category": "shipping", "urgency": "medium", ...}
      ↓ 구조화된 데이터
Step 2 출력 (JSON):
  {"order": {...}, "shipping": {...}, "customer": {...}}
      ↓ 실제 DB 데이터  
Step 3 출력 (JSON):
  {"can_resolve": true, "action": "배송조회안내", ...}
      ↓ 구조화된 판단
Step 4 출력 (텍스트):
  "김민수 고객님, 안녕하세요..."
      ↓ 자연어 답변
Step 5 출력 (JSON):
  {"approved": true, "issues": []}
      ↓ 구조화된 검증
최종: 발송 or 상담원 전달
```

&nbsp;

**자연어는 Step 4(답변 생성)에서만.** 나머지는 전부 JSON.

이게 "자연어로 넘어오면 부정확하지 않나?"의 답이다.

**단계 간 전달은 JSON, 최종 고객 응답만 자연어.**

&nbsp;

&nbsp;

---

&nbsp;

# 10. 성과

&nbsp;

| 지표 | 상담원만 | AI 체인 도입 후 |
|:---|:---|:---|
| 일 처리량 | 1,000건 (상담원 5명) | 1,000건 (AI 700건 + 상담원 300건) |
| 평균 응답 시간 | 15분 | 5초 (AI) / 10분 (상담원) |
| 상담원 수 | 5명 | 2명 (복잡한 건만) |
| 고객 만족도 | 78% | 85% (빠른 응답) |
| 월 인건비 | 약 1,500만원 | 약 600만원 + AI 50만원 |

&nbsp;

```
비용 절감: 월 850만원 (약 $5,944)
연간: 약 1억원 절감

AI 비용 (월):
  분류: 1,000건 × $0.01 = $10
  판단: 1,000건 × $0.03 = $30
  답변: 1,000건 × $0.02 = $20
  검증: 1,000건 × $0.01 = $10
  합계: $70/월 (약 10만원)
```

&nbsp;

**AI 비용 월 10만원으로 상담원 3명분의 업무를 처리한다.**

&nbsp;

&nbsp;

---

&nbsp;

# 11. 이 패턴이 적용되는 다른 산업

&nbsp;

| 산업 | Step 1 (분류) | Step 2 (정보) | Step 3 (판단) | Step 4 (생성) |
|:---|:---|:---|:---|:---|
| **이커머스** | 문의 분류 | 주문 DB | 환불 정책 | 답변 |
| **금융** | 문의 분류 | 계좌/거래 DB | 금융 규정 | 답변 |
| **의료** | 증상 분류 | 환자 기록 | 진료 가이드 | 안내 (진단X) |
| **법률** | 사건 분류 | 판례 검색 | 법률 해석 | 의견서 초안 |
| **HR** | 문의 분류 | 사원 정보 | 인사 규정 | 답변 |
| **IT 헬프데스크** | 이슈 분류 | 시스템 로그 | 해결 KB | 가이드 |

&nbsp;

**구조는 동일하다.** 분류 → 정보 수집 → 정책 판단 → 생성 → 검증.

달라지는 건 DB와 정책 문서뿐이다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론

&nbsp;

프롬프트 체인의 핵심:

&nbsp;

1. **한 번에 다 시키지 않는다** — 역할별로 나눈다
2. **단계 간 데이터는 JSON으로** — 자연어는 마지막 답변만
3. **AI가 못 하는 건 코드가 한다** — DB 조회, API 호출
4. **AI가 만든 건 AI가 검증한다** — 다른 세션에서
5. **못 풀면 사람에게 넘긴다** — 100% 자동화가 목표가 아니다

&nbsp;

"AI한테 한 번에 시키면 되지 않나?"라는 생각이 들 때,

**신입한테 "이거 다 해"라고 하면 어떻게 되는지** 떠올려보자.

단계별로 알려줘야 정확하게 한다.

AI도 똑같다.

&nbsp;

&nbsp;

---

프롬프트체인, AI자동화, 이커머스, 고객상담, RAG, LLM, ChatGPT, Claude, 벡터DB, 프롬프트엔지니어링, 챗봇, 자동화, 파이프라인, AI에이전트
