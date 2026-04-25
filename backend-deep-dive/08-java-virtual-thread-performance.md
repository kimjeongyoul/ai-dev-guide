# [백엔드 8편] Java Virtual Thread — Thread-per-Request 모델의 한계를 극복하는 새로운 동시성 패러다임

&nbsp;

높은 동시성을 확보하기 위해 그동안 Java 개발자들은 리소스 소모가 큰 Thread-per-Request 모델과 코드 복잡도가 높은 Reactive Programming 사이에서 고민해 왔습니다. 
 Java 21에서 도입된 Virtual Thread(Project Loom)는 이러한 고민에 대한 새로운 해답을 제시합니다. 본 아티클에서는 Virtual Thread의 내부 메커니즘과 도입 시 고려해야 할 사항들을 분석합니다.

---

## 1. Platform Thread의 한계와 진화

기존 Java의 Thread는 OS 스레드를 래핑한 Platform Thread입니다.

*   높은 리소스 비용: OS 스레드는 생성 시 약 1MB의 스택 메모리를 할당받으며 컨텍스트 스위칭 비용이 발생합니다.
*   I/O 블로킹 문제: I/O 작업 동안 OS 스레드는 대기 상태가 되어 리소스가 낭비됩니다. 이를 해결하기 위해 등장한 리액티브 스택은 성능은 좋으나 학습 곡선이 높고 디버깅이 어렵다는 단점이 있습니다.

---

## 2. Virtual Thread의 내부 구조: Carrier Thread와 Continuation

Virtual Thread는 OS 스레드에 직접 매핑되지 않는 경량 사용자 수준 스레드입니다.

### 2.1 Carrier Thread와 스케줄링
Virtual Thread는 실제로 ForkJoinPool 기반의 Carrier Thread 위에서 실행됩니다. Virtual Thread가 I/O 작업을 만나 블로킹되면, 런타임은 해당 Virtual Thread의 상태를 힙(Heap) 메모리에 저장하고 Carrier Thread에서 언마운트(Unmount)합니다. Carrier Thread는 즉시 다른 Virtual Thread를 실행할 수 있게 되어 소수의 OS 스레드로 많은 동시 요청을 처리할 수 있습니다.

### 2.2 Continuation
Virtual Thread의 핵심은 실행 중인 코드 지점과 로컬 변수 상태를 캡처하여 저장했다가 나중에 다시 실행할 수 있게 해주는 저수준 기술입니다.

---

## 3. 실무 도입 시의 핵심 이슈: Pinning (피닝)

Virtual Thread 도입 시 가장 주의해야 할 현상이 Pinning입니다.

*   발생 조건: synchronized 블록 내부에서 I/O 작업을 수행하거나 Native Method를 호출할 때 발생합니다.
*   현상: Virtual Thread가 Carrier Thread에서 언마운트되지 못하고 OS 스레드를 점유하게 되어 시스템 가용성이 떨어집니다.
*   해결책: synchronized 대신 ReentrantLock을 사용하여 블로킹 시 언마운트가 원활하게 일어나도록 코드를 수정해야 합니다.

---

## 4. 성능 분석: WebFlux vs Virtual Thread

*   처리량: I/O 밀집형 작업에서 Virtual Thread는 WebFlux와 대등하거나 그 이상의 성능을 보여줍니다.
*   메모리 효율: 수만 개의 요청이 몰려도 Virtual Thread는 힙 메모리를 유연하게 사용하므로 효율적입니다.
*   개발 경험: Virtual Thread의 이점은 가독성입니다. 리액티브 연산자 없이 익숙한 코드로 높은 성능을 구현할 수 있습니다.

---

## 5. 실무 도입 및 운영 가이드라인

1.  스레드 풀 지양: Virtual Thread는 생성 비용이 저렴하므로 기존처럼 스레드 풀에 가두는 대신 필요할 때마다 생성하여 사용하는 방식이 권장됩니다.
2.  ThreadLocal 사용 주의: 수백만 개의 스레드가 무거운 ThreadLocal 변수를 가지게 되면 메모리 압박이 올 수 있으므로 Scoped Values 도입을 검토해야 합니다.
3.  라이브러리 호환성 체크: 사용하는 라이브러리가 내부적으로 synchronized를 사용하는지 확인해야 하며, 최근 라이브러리들은 점진적으로 Virtual Thread를 지원하고 있습니다.

&nbsp;
Java21, VirtualThread, Loom프로젝트, 동시성모델, 성능최적화, 처리량향상, 백엔드설계
