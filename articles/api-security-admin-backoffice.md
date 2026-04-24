# 백오피스 관리자 API도 OWASP API Security를 지켜야 할까

결론부터 말하면, 그렇다. 그리고 보통 더 엄격하게 지켜야 한다.

많은 팀이 관리자용 백오피스를 "내부 시스템", "운영자만 쓰는 화면", "사내망 전용"이라고 생각해서 일반 사용자 API보다 보안 우선순위를 낮게 둔다. 그런데 실제 사고는 오히려 이런 관리자 API에서 더 크게 터진다. 이유는 간단하다. 관리자 API는 조회만 하는 API가 아니라, 계정 생성, 권한 변경, 설정 수정, 배포, 서비스 재시작, 데이터 삭제 같은 고위험 기능을 갖고 있기 때문이다.

즉, 일반 사용자 API가 뚫리면 일부 사용자 정보가 노출될 수 있지만, 관리자 API가 뚫리면 운영 전체가 흔들릴 수 있다.

## 왜 관리자라서 예외가 아니냐

백오피스 API는 보통 다음 성격을 가진다.

- 권한이 크다
- 데이터 범위가 넓다
- 수정, 삭제, 배포 같은 파괴적 기능이 있다
- 내부망에만 있다고 믿고 방어를 덜 넣기 쉽다

문제는 이 전제가 자주 깨진다는 점이다.

- VPN 계정이 탈취될 수 있다
- 내부 직원 계정이 피싱당할 수 있다
- 잘못된 프록시 설정으로 외부에 노출될 수 있다
- 개발/운영 환경이 섞이면서 테스트 엔드포인트가 살아남을 수 있다

그래서 관리자 API는 외부 사용자용 API보다 보안 요구사항이 낮다가 아니라, 오히려 더 높은 수준의 인증, 인가, 감사, 제한 정책이 필요하다.

## 관리자 API에서 특히 중요한 보안 원칙

### 1. 인증은 기본이고, 권한 분리는 필수다

관리자 로그인만 되어 있으면 모든 API를 호출할 수 있게 두는 경우가 많다. 하지만 실제 운영에서는 최소한 역할 기반 권한 분리가 있어야 한다.

예를 들면 이런 식이다.

- 운영자: 조회, 일부 수정
- 관리자: 설정 변경, 계정 관리
- 슈퍼 관리자: 배포, 서비스 제어, 삭제

즉, 로그인했는가만 확인하면 안 되고, 이 사용자가 이 기능을 실행해도 되는가까지 확인해야 한다.

### 2. 객체 단위 권한 검사가 필요하다

관리자라고 해서 모든 멤버, 모든 점포, 모든 키오스크, 모든 패치를 다 다뤄도 되는 것은 아니다.

예를 들어:

- A 운영자는 A 지점 데이터만 봐야 할 수 있다
- 특정 관리자는 특정 장비군만 제어할 수 있어야 할 수 있다
- 특정 배포는 특정 서비스 담당자만 실행해야 할 수 있다

즉, 단순히 `id`를 받아 조회 또는 수정하는 API는 위험하다. 항상 이 사용자가 이 객체에 접근할 수 있는가를 별도로 확인해야 한다.

### 3. 입력 검증과 필드 화이트리스트는 별개다

많은 개발자가 입력 검증을 넣으면 충분하다고 생각한다. 하지만 실제로는 두 단계가 필요하다.

1. 요청 형식이 맞는지 검증
2. 수정 가능한 필드만 골라서 반영

이 차이가 중요한 이유는, 클라이언트가 원래 보내면 안 되는 필드를 같이 보내는 순간 문제가 생길 수 있기 때문이다. 예를 들어 `role`, `status`, `isDeleted`, `approvedAt` 같은 필드가 의도치 않게 수정되면 큰 사고가 된다.

그래서 안전한 방식은 항상:

- 스키마로 요청을 검증하고
- 서버에서 허용 필드만 다시 추려서
- 업데이트에 넘기는 것

이다.

### 4. 민감 기능은 관리자 권한만으로 끝내지 않는 게 좋다

특히 위험한 기능은 한 단계 더 보호하는 것이 좋다.

예를 들면:

- 배포 실행
- 계정 삭제
- 권한 변경
- 서비스 재시작
- 대량 데이터 수정
- 외부 연동 설정 변경

이런 기능은 다음 같은 보호를 추가할 수 있다.

- 슈퍼 관리자만 허용
- 재인증 필요
- 확인용 입력 필요
- 승인 워크플로우 필요
- 감사 로그 강제 기록

즉, 민감한 기능일수록 버튼 하나 누르면 바로 실행되는 구조를 피해야 한다.

### 5. 감사 로그는 선택이 아니라 운영 안전장치다

백오피스 API에서는 누가, 언제, 무엇을, 어떤 값으로 바꿨는지가 반드시 남아야 한다.

감사 로그가 필요한 이유는 세 가지다.

- 사고 조사
- 오남용 추적
- 운영 복구

조회 API까지 전부 기록할 필요는 없지만, 최소한 다음은 남겨야 한다.

- 로그인과 로그아웃
- 계정 생성, 삭제, 권한 변경
- 설정 변경
- 배포 실행
- 명령 발행
- 실패한 권한 시도
- 민감 데이터 수정

감사 로그는 개발자 편의를 위한 로그가 아니라, 운영 통제 장치로 봐야 한다.

### 6. 내부망이니까 괜찮다는 설계는 오래 못 간다

관리자 API는 네트워크 레벨에서도 제한해야 한다.

좋은 운영 환경이라면 보통 이런 식으로 간다.

- 관리자 API는 VPN 또는 사내망에서만 접근 가능
- 장비 제어 API는 허용된 IP에서만 접근 가능
- 업로드와 배포 API는 별도 게이트웨이 뒤에 둠
- 개발용 또는 mock API는 운영 빌드에서 제거

코드 수준 보안이 중요하지만, 네트워크 레벨에서 한 번 더 막는 것이 현실적으로 매우 효과적이다.

## 관리자 API에 적용하기 좋은 코드 패턴

아래 예시는 특정 프로젝트와 무관한, 평범한 관리자 API 패턴이다.

### 인증과 권한 검사를 공통 래퍼로 묶기

```ts
import { NextRequest, NextResponse } from 'next/server';

type AdminUser = {
  id: string;
  role: 'super' | 'manager' | 'operator';
  permissions: string[];
};

async function getAuthenticatedUser(request: NextRequest): Promise<AdminUser | null> {
  const session = request.cookies.get('admin_session')?.value;
  if (!session) return null;

  return {
    id: 'u1',
    role: 'manager',
    permissions: ['member:read', 'member:update'],
  };
}

function hasPermission(user: AdminUser, permission: string) {
  return user.permissions.includes(permission);
}

export function withAdminAuth(
  handler: (request: NextRequest, user: AdminUser) => Promise<NextResponse>,
  permission?: string,
) {
  return async (request: NextRequest) => {
    const user = await getAuthenticatedUser(request);

    if (!user) {
      return NextResponse.json({ success: false, error: 'Unauthorized' }, { status: 401 });
    }

    if (permission && !hasPermission(user, permission)) {
      return NextResponse.json({ success: false, error: 'Forbidden' }, { status: 403 });
    }

    return handler(request, user);
  };
}
```

이 패턴의 장점은 인증과 인가 코드가 각 API에 흩어지지 않고 공통화된다는 점이다.

### 입력 검증은 스키마로 명확하게

```ts
import { z } from 'zod';

export const updateMemberSchema = z.object({
  phone: z.string().min(1).optional(),
  email: z.string().email().optional(),
  address: z.string().min(1).optional(),
  status: z.enum(['active', 'inactive']).optional(),
  grade: z.string().min(1).optional(),
});
```

그리고 API에서는 이렇게 쓴다.

```ts
const body = updateMemberSchema.parse(await request.json());
```

이렇게 하면 입력 형식이 틀린 요청을 초기에 잘라낼 수 있다.

### 화이트리스트 업데이트로 과도한 필드 수정 방지

```ts
const updateData = {
  ...(body.phone !== undefined && { phone: body.phone }),
  ...(body.email !== undefined && { email: body.email }),
  ...(body.address !== undefined && { address: body.address }),
  ...(body.status !== undefined && { status: body.status }),
  ...(body.grade !== undefined && { grade: body.grade }),
};

await memberService.update(memberId, updateData);
```

겉보기엔 조금 번거롭지만, 실제로는 이게 가장 안전하다. 관리자 API에서는 특히 요청 body 전체를 그대로 update에 전달하는 방식을 피해야 한다.

### 객체 접근 권한도 별도로 확인하기

```ts
async function canAccessMember(userId: string, memberId: string) {
  return true;
}
```

```ts
if (!(await canAccessMember(user.id, memberId))) {
  return NextResponse.json({ success: false, error: 'Forbidden' }, { status: 403 });
}
```

이 단계가 빠지면 로그인한 관리자라면 다른 조직 데이터까지 건드릴 수 있게 된다.

### 감사 로그 남기기

```ts
await auditLogService.write({
  actorId: user.id,
  action: 'member.update',
  targetType: 'member',
  targetId: memberId,
  metadata: updateData,
  ipAddress: request.headers.get('x-forwarded-for') ?? 'unknown',
});
```

감사 로그는 나중에 추가하려 하면 항상 늦는다. 민감한 관리자 기능부터 먼저 넣는 것이 좋다.

### 민감 API는 별도 보호 추가

```ts
const deploySchema = z.object({
  releaseId: z.string().min(1),
  targetIds: z.array(z.string()).min(1),
  confirmText: z.literal('DEPLOY'),
});
```

```ts
if (user.role !== 'super') {
  return NextResponse.json({ success: false, error: 'Forbidden' }, { status: 403 });
}
```

배포, 계정 삭제, 서비스 제어 같은 기능은 보통 여기까지 가는 것이 좋다.

## 관리자 API에서 자주 놓치는 항목

### Rate Limit

관리자라고 해도 brute force, 대량 호출, 실수로 인한 반복 요청은 발생한다. 로그인 API, 업로드 API, 명령 발행 API 정도는 최소한의 rate limit이 필요하다.

### 개발용 엔드포인트 정리

운영 환경에서 mock 로그인, 테스트용 우회 로직, 디버그 API가 남아 있지 않아야 한다.

### 업로드와 다운로드 제한

파일 업로드 API는 확장자만 체크해서는 부족하다.

- 크기 제한
- 저장 위치 제한
- 파일명 정규화
- 다운로드 권한 확인

이 기본이 필요하다.

### 외부 연동 응답 검증

관리자 API는 다른 시스템을 프록시하는 경우가 많다. 이때 외부 API 응답을 그대로 신뢰하면 안 된다. 응답 구조, 상태값, 허용 필드를 다시 검증하는 습관이 필요하다.

## 현실적인 적용 순서

모든 걸 한 번에 다 넣으려 하면 오히려 진도가 안 나간다. 실무에서는 보통 아래 순서가 가장 효율적이다.

### 1단계

- 인증 공통화
- 권한 체크 공통화
- 민감 API부터 적용

### 2단계

- 입력 검증 스키마 도입
- 화이트리스트 업데이트 방식 정착
- 감사 로그 추가

### 3단계

- 객체 접근 권한 분리
- rate limit
- 네트워크 레벨 제한
- 운영과 개발 API inventory 정리

## 정리

백오피스 관리자 API도 OWASP API Security 원칙을 지켜야 한다. 정확히는, 관리자 API이기 때문에 더 엄격하게 지켜야 한다.

핵심은 어렵지 않다.

- 로그인만 확인하지 말고 권한까지 본다
- ID만 받았다고 바로 조회 또는 수정하지 않는다
- body 전체를 그대로 update하지 않는다
- 민감 기능은 한 단계 더 보호한다
- 감사 로그를 남긴다
- 내부망이라는 가정에 의존하지 않는다

보안은 특별한 기능이 아니라, 관리자 API의 기본 품질이다.

## 추천 체크리스트

- 인증이 없는 관리자 API는 없는가
- 역할별 권한 분리가 되어 있는가
- 객체 단위 접근 권한 검사가 있는가
- 수정 API가 화이트리스트 방식인가
- 배포, 삭제, 재시작 같은 민감 기능에 추가 보호가 있는가
- 감사 로그가 남는가
- 업로드와 다운로드 API 제한이 있는가
- mock, dev, debug API가 운영에 남아 있지 않은가
- 관리자 API가 내부망 또는 VPN 뒤에 있는가
