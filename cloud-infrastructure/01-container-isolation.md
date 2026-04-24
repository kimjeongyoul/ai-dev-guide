# [플랫폼 1편] 컨테이너 격리의 실체 — Linux Namespace와 Cgroups가 만드는 가상화의 마법

&nbsp;

"컨테이너는 가벼운 VM(Virtual Machine)이다." 이 비유는 초급자에게는 유용할지 모르나, 시스템을 설계하는 엔지니어에게는 위험한 오해다. 컨테이너는 운영체제 위에 독립적으로 떠 있는 섬이 아니라, 호스트 커널(Host Kernel)의 기능을 빌려 쓰고 있는 '격리된 프로세스'에 불과하다. 

&nbsp;

그렇다면 어떻게 하나의 프로세스가 마치 독립된 운영체제처럼 고유한 IP를 갖고, 자신만의 파일 시스템을 보며, 다른 프로세스를 볼 수 없게 격리되는 것일까? 이 마법의 정체는 리눅스 커널의 두 핵심 기능인 **Namespace**와 **Control Groups (Cgroups)**에 있다. 본 글에서는 컨테이너 가상화의 근간을 이루는 커널 레벨의 격리 메커니즘을 소스코드 수준에서 낱낱이 파헤친다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. Namespace: 프로세스의 '시야'를 제한하다

&nbsp;

네임스페이스는 프로세스가 시스템의 특정 자원을 바라보는 '관점'을 분리한다. 컨테이너 내부에서 `ps -ef`를 쳤을 때 PID 1번이 내 앱으로 나오는 이유는, 해당 프로세스가 독립된 **PID Namespace**에 갇혀 있기 때문이다.

&nbsp;

## 1-1. 핵심 네임스페이스의 6가지 유형
리눅스 커널은 현재 다음과 같은 주요 자원들을 네임스페이스로 격리한다.
- **UTS (Unix Timesharing System)**: 호스트 네임과 도메인 네임을 격리.
- **IPC (Inter-Process Communication)**: 공유 메모리나 세마포어 등 프로세스 간 통신 자원 격리.
- **PID (Process ID)**: 프로세스 번호를 격리하여 컨테이너 안의 1번 프로세스가 호스트의 수천 번 프로세스와 매핑되게 함.
- **NET (Network)**: 네트워크 인터페이스, IP 스택, 라우팅 테이블 등을 독립적으로 소유.
- **MNT (Mount)**: 파일 시스템 마운트 포인트를 격리하여 컨테이너가 고유한 `/` 루트 디렉토리를 보게 함 (chroot의 확장).
- **USER**: 사용자 ID(UID)와 그룹 ID(GID)를 격리하여 컨테이너 내부의 root가 호스트의 일반 사용자이게 함.

&nbsp;

## 1-2. 커널 시스템 콜(Syscall) 수준의 구현
컨테이너 런타임(RunC 등)은 새로운 프로세스를 생성할 때 `clone()` 시스템 콜에 특정 플래그를 전달하여 격리를 구현한다.

&nbsp;

```c
// C 언어를 이용한 단순 PID 네임스페이스 격리 예시
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <unistd.h>

int child_main(void* arg) {
    printf("Child process - PID: %d\n", getpid()); // 컨테이너 내부에서는 1이 출력됨
    return 0;
}

int main() {
    // CLONE_NEWPID 플래그를 통해 새로운 PID 네임스페이스를 할당
    int child_pid = clone(child_main, stack_ptr, CLONE_NEWPID | SIGCHLD, NULL);
    waitpid(child_pid, NULL, 0);
    return 0;
}
```

&nbsp;

엔지니어는 `unshare()` 명령어나 `/proc/[pid]/ns` 디렉토리를 통해 현재 프로세스가 어떤 네임스페이스 아이노드(inode)를 소유하고 있는지 확인할 수 있으며, 이를 통해 네트워크나 마운트 꼬임 현상을 추적할 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. Cgroups: 프로세스의 '식량'을 통제하다

&nbsp;

네임스페이스가 프로세스의 '시야'를 가린다면, **Cgroups (Control Groups)**는 프로세스가 소모할 수 있는 자원의 '양'을 물리적으로 제한한다. 특정 컨테이너가 호스트의 CPU를 100% 점유하여 다른 서비스에 영향을 주는 'Noisy Neighbor' 문제를 해결하는 핵심 기술이다.

&nbsp;

## 2-1. 하이재킹 없는 자원 할당 메커니즘
Cgroups는 커널 레벨에서 프로세스 그룹별로 CPU 사이클, 메모리 한도, 블록 I/O 대역폭 등을 할당한다.
- **CPU Limit**: `cpu.cfs_quota_us` 설정을 통해 프로세스가 특정 시간(period) 동안 사용할 수 있는 마이크로초 단위의 연산 시간을 강제한다. 만약 할당량을 초과하면 커널은 해당 프로세스를 스케줄링에서 제외(Throttling)시킨다.
- **Memory Limit**: `memory.limit_in_bytes`를 통해 사용량을 제어한다. 만약 프로세스가 할당된 메모리를 초과하여 할당받으려 하면 커널의 OOM(Out of Memory) Killer가 개입하여 해당 프로세스를 즉시 사살한다.

&nbsp;

## 2-2. Cgroups v1 vs v2의 아키텍처 변화
현대적인 K8s 노드(v1.25 이상)는 Cgroups v2를 표준으로 채택하고 있다.
- **v1**: 각 자원(CPU, Memory)마다 별도의 계층 구조를 가져 관리가 복잡하고 자원 간 상관관계 파악이 어려웠다.
- **v2**: 통합된 단일 계층 구조(Unified Hierarchy)를 사용하여 메모리 사용량에 따른 CPU 압력(Pressure Stall Information, PSI) 등을 정확하게 계산할 수 있게 되었다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. 레이어드 파일 시스템 (OverlayFS) — 격리의 마지막 퍼즐

&nbsp;

네임스페이스와 Cgroups가 프로세스를 가두었다면, 컨테이너가 1초 만에 실행될 수 있게 만드는 속도의 비결은 **OverlayFS**에 있다. 수 기가바이트의 이미지를 복사하는 대신, 읽기 전용(Read-only) 이미지 레이어 위에 얇은 쓰기 가능(Writable) 레이어를 얹는 방식이다.

&nbsp;

- **Lower Dir**: 베이스 이미지 레이어 (변경 불가).
- **Upper Dir**: 컨테이너 실행 중 발생하는 변경 사항이 기록되는 레이어.
- **Merged**: 사용자가 컨테이너 내부에서 최종적으로 바라보는 통합 뷰.

&nbsp;

이 구조 덕분에 우리는 수백 개의 컨테이너를 띄워도 실제 디스크 용량은 베이스 이미지 한 장만큼만 소모하는 압도적인 효율성을 얻게 된다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 컨테이너는 커널의 투영이다

&nbsp;

컨테이너 기술의 본질은 "커널 자원을 얼마나 세밀하게 쪼개어 프로세스에게 할당할 수 있는가"에 있다. Docker나 K8s는 이러한 복잡한 커널 파라미터 제어를 추상화한 도구일 뿐이다. 

&nbsp;

인프라가 꼬이고 네트워크 병목이 발생했을 때, YAML 파일을 수정하는 수준을 넘어 커널의 Namespace 아이노드를 확인하고 Cgroups의 Throttling 로그를 분석할 수 있는 능력이 진정한 플랫폼 엔지니어를 가르는 기준점이 된다. 가상화는 하이퍼바이저의 전유물이 아니라, 우리 운영체제 커널이 이미 수십 년 전부터 준비해 온 논리적 완성품이다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[플랫폼 2편] K8s 스케줄링 메커니즘 — Resource 오버커밋(Overcommit)과 Bin-packing 최적화 전략**

&nbsp;

내가 배포한 파드(Pod)는 어떤 근거로 특정 노드에 배치되는 것일까? 쿠버네티스 스케줄러(`kube-scheduler`)의 내부 필터링(Filtering)과 스코어링(Scoring) 알고리즘을 소스코드 레벨에서 분석한다. 또한, 자원 효율을 극대화하기 위해 노드를 꽉꽉 채우는 Bin-packing 전략과, 그 이면에서 발생하는 자원 경합(Contention) 문제를 해결하는 스케줄링 튜닝 기법을 심층적으로 다룬다.

&nbsp;

&nbsp;

---

&nbsp;

Namespace, Cgroups, 커널격리, OverlayFS, 컨테이너가상화, 리눅스인터널, 플랫폼엔지니어링, Docker원리
