# 코어 친화도(Affinity) — 핀닝이 캐시 지역성을 지킨다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- OS 스케줄러가 스레드를 다른 코어로 옮길 때 무엇이 손실되는가?
- `taskset` / `sched_setaffinity` / `pthread_setaffinity_np`의 차이는?
- 같은 L2/L3를 공유하는 코어에 협력 스레드를 묶는 게 왜 유리한가?
- 스레드 친화도가 NUMA 메모리 위치(Ch6-02)와 어떻게 상호작용하는가?
- JVM이 친화도와 GC, 그리고 거대 페이지 설정과 어떻게 얽히는가?
- 친화도 설정이 오히려 성능을 떨어뜨릴 수 있는 경우는?

---

## 🔍 왜 이 개념이 중요한가

### "코어 16개인데 처리량이 8 스레드부터 안 늘어난다"

```
일반적인 진단 흐름:
  ① CPU 사용률 확인 → 코어들이 골고루 ~50% 사용
  ② perf로 캐시 미스 확인 → 평소 대비 2~3배 높음
  ③ 처리량은 코어 수 늘려도 정체

원인 후보:
  a) 락 경합 (Ch6-04)
  b) 메모리 대역폭 한계 (Ch7-03)
  c) **스레드 마이그레이션으로 인한 캐시 warm-up 손실** ← 이 문서

OS 스케줄러의 사실:
  Linux CFS는 부하 균형 위해 주기적으로 스레드를 다른 코어로 이동
  → 새 코어의 L1/L2 캐시는 비어있음
  → 워밍업되는 동안 모든 접근이 L3 또는 DRAM
  → 짧은 작업이 자주 이동되면 캐시 워밍업 비용이 작업 비용을 넘김

핀닝(pinning):
  특정 스레드를 특정 코어에 고정 → 캐시·TLB·BTB 모두 그 코어에 유지
  성능 결정적 워크로드(DB, 게임 서버, 거래 시스템)의 기본 기법
```

---

## 😱 잘못된 이해

### Before: "스케줄러가 알아서 잘 할 것이다"

```
잘못된 가정 1:
  "OS는 알아서 가장 좋은 코어에 배치한다"
  
실제:
  CFS의 목표는 "공정성 + 부하 균형"이지 "캐시 지역성"이 아님
  → 빈 코어가 있으면 그쪽으로 옮기는 게 우선
  → 캐시 손실 고려는 보조적 (cache hot/cold heuristic 정도)

잘못된 가정 2:
  "코어를 많이 쓰면 빨라진다"
  
실제:
  스레드 수 > 물리 코어 → context switch + 마이그레이션 폭증
  특히 SMT(Hyper-Threading) 형제 코어는 같은 실행 유닛 공유
  → 친화도 없이 운영하면 두 스레드가 같은 SMT 형제에 몰리는 일 발생

잘못된 가정 3:
  "친화도를 걸면 무조건 빠르다"
  
실제:
  너무 빡빡하게 핀닝하면 OS가 부하 균형을 못 함
  → 특정 코어가 다른 워크로드(인터럽트, 시스템 데몬)와 경쟁
  → 오히려 latency spike

올바른 접근:
  핀닝 대상 결정 → 격리 코어 확보 → 측정 → 조정
  성능 결정적 워크로드만 핀닝, 일반 워크로드는 OS에 맡김
```

---

## ✨ 올바른 이해

### After: 친화도는 워크로드별로 신중하게 설계한다

```
스레드 마이그레이션 시 손실되는 것:

  ┌──────────────────────────────────────────┐
  │ Core 0 → Core 4로 이동                    │
  ├──────────────────────────────────────────┤
  │ L1d 캐시 (32KB)    → 콜드, 0 hit          │
  │ L1i 캐시 (32KB)    → 콜드                 │
  │ L2 캐시 (256KB-1MB)→ 콜드                 │
  │ L1 dTLB (~64)      → 콜드                 │
  │ L1 iTLB (~128)     → 콜드                 │
  │ Branch Predictor   → 콜드                 │
  │ Store Buffer       → 비워짐                │
  │                                            │
  │ L3 캐시 (소켓 공유) → 같은 소켓이면 hit 유지│
  │ DRAM 데이터        → 그대로 (NUMA 이슈 별개)│
  └──────────────────────────────────────────┘

워밍업 비용:
  L1/L2 캐시 워밍업: ~수십~수백 µs (워킹셋 크기에 따라)
  Branch Predictor 학습: 수천 분기 실행 필요
  → 짧은 task(<1ms)가 빈번히 마이그레이션되면 큰 손실

친화도 전략:
  ① 단일 코어 핀: 특정 스레드 = 특정 코어
  ② NUMA 노드 핀: NUMA 노드 범위 내에서만 이동 가능
  ③ 캐시 도메인 핀: 같은 L2/L3 공유 코어 내에서만 이동
  ④ 코어 격리(isolcpus): OS 스케줄러가 절대 사용 못 하는 코어 확보
```

---

## 🔬 내부 동작 원리

### 1. Linux CFS와 부하 균형

```
CFS (Completely Fair Scheduler) 부하 균형:

  스케줄링 도메인 계층:
    Smt   (SMT 형제)
    MC    (같은 L2/L3 공유 — Multi-Core 도메인)
    NUMA  (같은 노드)
    System(전체)
  
  각 도메인마다 부하 균형 주기 다름:
    Smt   → 가장 자주 (수십 ms)
    MC    → 그 다음 (수백 ms)
    NUMA  → 더 드물게 (초 단위)
    System→ 가장 드물게
  
  부하 균형 트리거:
    ① 코어가 IDLE이면 빈 큐 발견 → 다른 코어에서 task 끌어옴
    ② periodic balancing
    ③ fork/exec 시 새 task 배치
  
  cache hot heuristic:
    최근 실행된 task는 "hot"으로 표시 → 옮기지 않음
    `sysctl kernel.sched_migration_cost_ns` (기본 ~500µs)
    이보다 짧게 실행됐으면 마이그레이션 회피

문제:
  부하 균형이 캐시 지역성보다 우선시되는 경우 존재
  특히 짧은 task가 많은 워크로드에서
  → 명시적 친화도가 필요한 이유
```

### 2. SMT(Hyper-Threading)와 친화도

```
SMT — 한 물리 코어 = 2개 논리 코어:

  Physical Core 0
    ├─ Logical CPU 0  (HT thread 0)
    └─ Logical CPU 8  (HT thread 1)  ← /sys 보면 8번이 형제

  같은 물리 코어의 SMT 형제는:
    L1/L2 캐시 공유
    실행 유닛(ALU/FPU/SIMD) 공유
    Branch Predictor 공유

좋은 경우:
  형제 둘 다 메모리 대기 → 한 쪽 작업, 다른 쪽 대기
  → 자원 활용도 ↑

나쁜 경우:
  형제 둘 다 SIMD 명령 폭주
  → 실행 유닛 경쟁 → 두 형제 모두 느려짐

  형제 한 쪽이 캐시 hot, 다른 쪽이 cold
  → 한 쪽이 다른 쪽의 캐시 라인을 evict → 둘 다 느려짐

CPU 토폴로지 확인:
  $ lscpu --extended
  CPU NODE SOCKET CORE L1d:L1i:L2:L3
  0   0    0      0    0:0:0:0       ← CPU 0과 CPU 8이 같은 Core 0
  1   0    0      1    1:1:1:0
  ...
  8   0    0      0    0:0:0:0
  9   0    0      1    1:1:1:0

또는:
  $ cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
  0,8

성능 결정적 워크로드:
  같은 물리 코어 안에서 스레드 2개 배치 금지
  → 짝수번 CPU만 사용 (또는 홀수번만)
  → SMT를 BIOS에서 꺼버리기도 함 (DB, HFT)
```

### 3. `taskset` / `sched_setaffinity` / `pthread_setaffinity_np`

```bash
# 셸 레벨 — taskset
taskset -c 0 ./a.out               # CPU 0에만 실행
taskset -c 0,2,4,6 ./a.out         # CPU 0,2,4,6 사용 가능
taskset -c 0-3 ./a.out             # CPU 0~3
taskset -p 0x1 <PID>               # 실행 중 프로세스 변경

# 확인
taskset -p <PID>
# pid 12345's current affinity mask: f  (= 0,1,2,3)
```

```c
// 프로세스 전체 — sched_setaffinity
#define _GNU_SOURCE
#include <sched.h>

cpu_set_t mask;
CPU_ZERO(&mask);
CPU_SET(0, &mask);
CPU_SET(2, &mask);
sched_setaffinity(0, sizeof(mask), &mask);
```

```c
// 특정 스레드 — pthread_setaffinity_np
#include <pthread.h>

cpu_set_t mask;
CPU_ZERO(&mask);
CPU_SET(0, &mask);

pthread_t tid = pthread_self();
pthread_setaffinity_np(tid, sizeof(mask), &mask);
```

```cpp
// C++20 jthread + custom affinity helper
auto pin_thread = [](std::thread& t, int cpu) {
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(cpu, &mask);
    pthread_setaffinity_np(t.native_handle(), sizeof(mask), &mask);
};
```

```java
// Java — 직접 지원 없음, JNI 또는 OpenHFT Affinity 라이브러리
// build.gradle: implementation 'net.openhft:affinity:3.x'
import net.openhft.affinity.AffinityLock;

try (AffinityLock al = AffinityLock.acquireCore()) {
    // 이 블록 내 코드는 핀닝된 코어에서 실행
    runHotLoop();
}
```

### 4. 캐시 도메인을 활용한 협력 스레드 묶기

```
시나리오: Producer-Consumer

  Producer  → 큐에 데이터 push
  Consumer  → 큐에서 데이터 pop

  메모리 패턴:
    Producer가 쓴 캐시라인 → Consumer가 읽음
    → 캐시 일관성(MESI, Ch3-01)으로 라인 전송 필요

  같은 L2 공유 코어에 두 스레드 배치:
    L2 캐시에서 직접 전송 (~12cy)
    → 빠른 코어 간 통신

  다른 NUMA 노드에 두 스레드 배치:
    DRAM을 통해 전송 (~수백cy + 원격 접근 비용)
    → 100배 이상 느림

확인 방법 — 코어 그룹 매핑:

  $ lstopo --of console
  Machine (total memory)
    NUMANode L#0 (Package L#0)
      L3 L#0 (8MB)
        L2 L#0 (256KB) + L1 → Core L#0
          PU L#0
          PU L#8         ← Core 0 (SMT pair: 0, 8)
        L2 L#1 + L1 → Core L#1
          PU L#1
          PU L#9
        ...
    NUMANode L#1 (Package L#1)
      ...

  → 같은 NUMA 노드 + 같은 L3 안에서 협력 스레드 배치
  → 같은 Core(SMT 형제)는 워크로드 따라 결정
```

### 5. `isolcpus` — OS로부터 코어 격리

```bash
# 부팅 시 커널 파라미터 (GRUB 설정)
# /etc/default/grub
GRUB_CMDLINE_LINUX="isolcpus=4-7 nohz_full=4-7 rcu_nocbs=4-7"

sudo update-grub && sudo reboot

# 효과:
#   CPU 4~7은 일반 스케줄러에서 제외
#   사용자 프로세스가 taskset으로 명시적으로 지정해야만 사용
#   nohz_full: 해당 코어에서 타이머 인터럽트 최소화 (latency 결정적)
#   rcu_nocbs: RCU 콜백을 다른 코어로 옮김

# 확인
cat /sys/devices/system/cpu/isolated
# 출력: 4-7

# HFT, 게임 서버, 거래 시스템에서 사용:
# - 핵심 워크로드를 isolcpus 코어에 핀닝
# - 일반 워크로드는 자동 배치되는 코어에서 동작
# - 핵심 워크로드의 latency spike 거의 제거
```

### 6. NUMA + Affinity — 메모리도 같이 묶기

```bash
# 친화도만 묶고 메모리 위치는 안 묶으면 NUMA 원격 접근 비용 발생 (Ch6-02)

# 잘못된 예 — CPU만 핀
taskset -c 0-7 ./a.out  
# 메모리는 어디든 할당 → NUMA 원격 접근 가능성

# 올바른 예 — CPU + 메모리 묶기
numactl --cpunodebind=0 --membind=0 ./a.out
# 메모리도 NUMA 노드 0에 강제

# 더 세밀하게
numactl --physcpubind=0,2,4,6 --membind=0 ./a.out

# Linux의 자동 NUMA 균형
echo 1 > /proc/sys/kernel/numa_balancing
# 이걸 키면 OS가 자동으로 메모리를 코어 근처로 옮김
# 일반 워크로드는 좋지만 결정적 latency 워크로드는 끄기 권장
```

---

## 💻 실전 실험

### 실험 1: 마이그레이션이 캐시에 미치는 영향 측정

```c
// affinity_demo.c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sched.h>
#include <unistd.h>
#include <time.h>

#define N (1 << 20)  // 4MB worth of int

void busy_work(int* arr) {
    long s = 0;
    for (int iter = 0; iter < 1000; iter++) {
        for (int i = 0; i < N; i++) s += arr[i];
    }
}

void migrate() {
    cpu_set_t mask;
    int cur = sched_getcpu();
    int next = (cur + 1) % 8;
    CPU_ZERO(&mask); CPU_SET(next, &mask);
    sched_setaffinity(0, sizeof(mask), &mask);
}

int main(int argc, char** argv) {
    int* arr = malloc(N * sizeof(int));
    memset(arr, 0, N * sizeof(int));
    
    struct timespec t1, t2;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    
    if (argc > 1 && argv[1][0] == 'm') {
        for (int i = 0; i < 10; i++) {
            busy_work(arr);
            migrate();  // 매번 코어 이동
        }
    } else {
        // 한 코어 고정
        cpu_set_t mask; CPU_ZERO(&mask); CPU_SET(0, &mask);
        sched_setaffinity(0, sizeof(mask), &mask);
        for (int i = 0; i < 10; i++) busy_work(arr);
    }
    
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms = (t2.tv_sec - t1.tv_sec) * 1000.0 + (t2.tv_nsec - t1.tv_nsec) / 1e6;
    printf("Time: %.2f ms\n", ms);
    free(arr);
}
```

```bash
gcc -O2 affinity_demo.c -o affinity_demo

# 마이그레이션 없음
perf stat -e cache-misses,cache-references,context-switches \
    ./affinity_demo pinned

# 마이그레이션 매번
perf stat -e cache-misses,cache-references,context-switches \
    ./affinity_demo m

# 예상: 마이그레이션 버전이 cache-misses 5~10배, 실행시간 30~100% 증가
```

### 실험 2: SMT 형제 vs 다른 물리 코어

```bash
# 토폴로지 확인
lscpu --extended | head -10

# 같은 물리 코어의 SMT 형제 (보통 0, 8이 짝)
taskset -c 0 ./worker1 & 
taskset -c 8 ./worker2 &  # CPU 8 = CPU 0의 SMT 형제
wait

# 다른 물리 코어 (CPU 0, CPU 1)
taskset -c 0 ./worker1 & 
taskset -c 1 ./worker2 & 
wait

# 두 워크로드 합계 시간 비교
# SIMD 등 실행 유닛 경쟁이 심한 워크로드: 형제 배치는 ~30% 손해
# 메모리 대기 많은 워크로드: 형제 배치가 ~10% 이득 (자원 공유 효과)
```

### 실험 3: Producer-Consumer 통신 비용

```bash
# 같은 L2 공유 코어 (예: CPU 0과 CPU 1이 같은 다이의 인접 코어)
taskset -c 0,1 ./prod_cons

# 다른 NUMA 노드 (CPU 0과 CPU 16이 다른 소켓)
taskset -c 0,16 ./prod_cons

# 측정 결과 비교
# perf c2c (cache-to-cache transfer) 분석
perf c2c record -F 99 ./prod_cons
perf c2c report

# 출력에서 HITM(modified line hit) 통계 확인
# 같은 L3 안: HITM 비용 ~40cy
# NUMA 원격: HITM 비용 ~수백cy
```

### 실험 4: isolcpus를 활용한 latency 결정적 워크로드

```bash
# /etc/default/grub에 isolcpus=6,7 추가하고 재부팅

# 일반 코어에서 (다른 워크로드와 경쟁)
taskset -c 0 ./latency_test
# p99: 80µs, max: 500µs (가끔 spike)

# 격리 코어에서
taskset -c 6 ./latency_test
# p99: 25µs, max: 30µs (spike 거의 없음)

# 추가 안정화
sudo cpupower frequency-set -g performance     # 클럭 변동 방지
sudo cset shield -c 6,7 --kthread=on            # cset으로 더 엄격히 격리

# nohz_full + rcu_nocbs 조합으로 인터럽트까지 차단하면
# 마이크로초 단위 latency 결정성 확보 가능
```

---

## 📊 성능 비교

```
워크로드별 친화도 효과:

워크로드                       기본 OS    핀닝 후     개선
─────────────────────────────────────────────────────
HFT 주문 핸들러                300µs      80µs       3~4배 (스파이크 거의 제거)
인메모리 DB (Redis)            avg p99    avg p99    p99 30~50% 감소
대용량 분석 (Spark executor)   100s       95s        5% (대부분 메모리 BW 한계)
범용 웹 서버                   응답시간   응답시간    거의 변화 없음 (I/O bound)
SIMD-heavy 배치                100s       70s        30% (SMT 형제 회피)
프로듀서-컨슈머 큐 (다른 NUMA) ~1µs/msg   ~50ns/msg  20배 (같은 L3 도메인)
GC 집약 JVM                    GC pause   GC pause   10~20% 감소

규칙:
  결정적 latency 워크로드 → 친화도 + isolcpus 큰 효과
  처리량 워크로드 → 효과 작음 (메모리 대역폭이 결국 한계)
  I/O bound 워크로드 → 효과 거의 없음
```

---

## ⚖️ 트레이드오프

```
친화도 도입 트레이드오프:

✅ 좋은 경우:
  - latency 결정적 워크로드 (HFT, 거래 시스템, 게임 서버)
  - 협력 스레드 통신이 핫스팟 (producer/consumer, pipeline)
  - DB 인메모리 워크로드
  - GC 결정성 필요한 JVM

❌ 나쁜 경우 / 주의:
  - 일반 웹 워크로드 — OS가 더 잘 분산
  - 너무 빡빡한 핀닝 → 부하 불균형
  - 핀닝 코어에 인터럽트 / 시스템 데몬 몰림
  - 컨테이너 환경에서 다른 컨테이너와 충돌

NUMA 고려:
  CPU 친화도 ≠ 메모리 친화도
  반드시 `numactl --cpunodebind --membind`로 둘 다 묶기
  first-touch 정책 활용 (Ch6-02)

운영 환경 차이:
  베어메탈: 자유롭게 isolcpus 사용
  VM: 호스트 스케줄러가 가상 코어를 다시 옮길 수 있음
  컨테이너: cgroup의 cpuset으로 친화도 제한
  
JVM 특수성:
  GC 스레드는 별도 친화도 설정 필요
  -XX:ParallelGCThreads, ConcGCThreads 조정
  GC 스레드와 애플리케이션 스레드를 다른 코어에 배치

복잡도 비용:
  운영 자동화 필요 — 코어 수 변화에 대응
  CPU 토폴로지가 시스템마다 다름 → 환경별 조정
  → 운영 코드에서 lstopo / numactl --hardware 정보 기반으로 동적 결정
```

---

## 📌 핵심 정리

```
코어 친화도 핵심:

마이그레이션 손실:
  L1/L2 캐시, TLB, BTB 모두 콜드 → 워밍업 비용
  짧은 task가 자주 마이그레이션될수록 손실 큼

친화도 도구:
  taskset -c               : 셸 레벨 핀닝
  sched_setaffinity        : 프로세스 레벨
  pthread_setaffinity_np   : 스레드 레벨
  isolcpus + nohz_full     : OS로부터 코어 격리

토폴로지 활용:
  lscpu --extended         : SMT 형제 확인
  lstopo --of console      : 캐시 도메인·NUMA 구조
  /sys/devices/system/cpu  : 세부 토폴로지

전략:
  단일 코어 핀 → latency 결정적
  같은 L2/L3 도메인 핀 → 협력 스레드 통신 최적화
  NUMA 노드 핀 → 메모리 친화도와 함께
  isolcpus → 가장 엄격, OS 인터럽트도 차단

함정:
  SMT 형제에 동시 배치 → 실행 유닛 경쟁
  CPU만 핀, 메모리는 안 핀 → NUMA 원격 접근
  너무 빡빡한 핀닝 → 인터럽트와 경쟁
  자동 NUMA 균형(numa_balancing)이 핀닝과 충돌

JVM:
  OpenHFT Affinity 라이브러리
  -XX:ParallelGCThreads 조정으로 GC 스레드 분리
  거대 페이지(+UseLargePages)와 조합 시 큰 효과

연결 챕터:
  Ch3-02 False Sharing → 같은 L3 도메인 활용으로 완화
  Ch6-01 토폴로지 → 친화도 설계의 입력
  Ch6-02 NUMA → 메모리 친화도와 함께
  Ch6-04 락 경합 → 협력 스레드 같은 캐시 도메인 배치
  linux-for-backend-deep-dive → CFS, cgroup 상세
```

---

## 🤔 생각해볼 문제

**Q1.** 거래 시스템에서 주문 처리 스레드를 CPU 4에 핀닝했더니 평균 응답시간은 줄었지만 p99가 가끔 200µs까지 튄다. 무엇이 원인일 가능성이 높은가?

<details>
<summary>해설 보기</summary>

가장 흔한 원인 후보:

**1. 같은 코어에 시스템 인터럽트가 옴**:
- 네트워크 IRQ, 타이머 인터럽트, RCU 콜백 등이 CPU 4에서 처리될 수 있음
- 확인: `cat /proc/interrupts | grep -E "CPU4"` (4번 열 값이 늘어나는지)
- 해결: IRQ affinity 조정 (`/proc/irq/<N>/smp_affinity`)으로 인터럽트를 다른 코어로 옮김

**2. 시스템 데몬과 경쟁**:
- systemd, journald, kthreadd 등이 같은 코어에서 실행
- 확인: `top -p $(pgrep -d',' system)` 또는 `ps -o pid,psr,cmd -e | awk '$2==4'`
- 해결: `isolcpus=4` 또는 cgroup의 cpuset으로 격리

**3. SMT 형제의 영향**:
- CPU 4의 SMT 형제(예: CPU 12)에서 다른 워크로드 실행 시 실행 유닛 경쟁
- 해결: 형제 코어도 isolcpus에 포함, 또는 BIOS에서 SMT 비활성화

**4. C-state 진입**:
- 코어가 일시적으로 IDLE이면 깊은 C-state로 진입 → 다시 깨어나는 데 수µs~수십µs
- 해결: `cpupower idle-set -D 1` 또는 부팅 파라미터 `intel_idle.max_cstate=1`

**5. CPU frequency scaling**:
- governor가 ondemand면 클럭이 변동 → latency 변동
- 해결: `cpupower frequency-set -g performance`

진단 도구: `perf sched record`로 스케줄링 이벤트, `tracecmd` 또는 `bpftrace`로 인터럽트 이벤트 추적.

</details>

---

**Q2.** Java로 만든 백엔드 서버에서 OpenHFT Affinity로 핵심 스레드 4개를 4개 코어에 핀닝했더니, GC가 발생하면 응답시간이 더 크게 튄다. 왜이며 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**원인 — GC 스레드와 애플리케이션 스레드가 같은 코어에서 경쟁**:

- OpenHFT Affinity는 핵심 스레드를 코어에 고정했지만 JVM의 GC 스레드들은 기본 OS 스케줄링을 따름
- GC가 발생하면 GC 스레드들이 핀닝된 코어로 몰릴 수 있음
- 핵심 스레드와 GC 스레드가 같은 코어 큐에서 경쟁 → 응답시간 spike

**해결**:

1. **GC 스레드 수 명시**:
   ```
   -XX:ParallelGCThreads=4
   -XX:ConcGCThreads=2
   ```
   기본값은 CPU 코어 수 기반이라 핀닝과 충돌

2. **GC 스레드 친화도**:
   - JVM은 GC 스레드 친화도 직접 지원 X
   - 우회: GC 스레드들을 다른 코어에 격리 (예: 핵심 코어 4개 외의 코어)
   - JNI로 직접 친화도 설정하는 라이브러리 / 에이전트 사용

3. **GC 알고리즘 변경**:
   - G1 / ZGC / Shenandoah 등 동시 GC로 STW 최소화
   - ZGC는 ms 미만 STW → 핀닝과 충돌 영향 최소

4. **isolcpus + 분리 배치**:
   - 핵심 스레드 4개는 isolcpus 코어에
   - 나머지 (GC 포함)는 일반 코어에
   - 단점: JVM이 GC 스레드도 isolcpus에 배치하지 못하게 막아야 함

5. **거대 페이지 활용**:
   - `-XX:+UseLargePages`로 TLB 미스 줄이기 → GC 자체가 빨라짐
   - 단, Ch2-06에서 본 Huge Page 단편화 함정 주의

권장 운영 패턴: latency 결정적 서비스는 작은 heap + ZGC + 핀닝 + GC 스레드 분리. 처리량 우선이면 핀닝 안 하고 큰 heap + G1.

</details>

---

**Q3.** 컨테이너 환경(Kubernetes)에서 친화도를 어떻게 적용하며, 베어메탈과 다른 점은?

<details>
<summary>해설 보기</summary>

**Kubernetes 친화도 도구**:

1. **CPU Manager `static` 정책**:
   - kubelet의 `--cpu-manager-policy=static`
   - Guaranteed QoS Pod이고 정수형 `cpu` 요청이면 전용 코어 할당
   - 자동으로 cgroup cpuset 설정

   ```yaml
   resources:
     requests:
       cpu: "4"
       memory: "8Gi"
     limits:
       cpu: "4"
       memory: "8Gi"
   ```

2. **Topology Manager**:
   - `--topology-manager-policy=single-numa-node`
   - CPU와 메모리를 같은 NUMA 노드에 배치
   - 베어메탈의 `numactl --cpunodebind --membind`와 같은 효과

3. **`cpuset` cgroup 직접 활용**:
   - 컨테이너 시작 후 `/sys/fs/cgroup/cpuset/.../cpuset.cpus`에 코어 ID 쓰기
   - 일반적으로 kubelet이 자동 관리하지만 세밀 제어 필요시 수동

**베어메탈과의 차이점**:

| 항목 | 베어메탈 | 컨테이너/K8s |
|------|---------|--------------|
| 친화도 | taskset, sched_setaffinity | cpuset cgroup, CPU Manager |
| isolcpus | 부팅 파라미터로 자유롭게 | 호스트에서 설정, Pod이 사용 |
| 인터럽트 격리 | 자유 | 노드 관리자가 설정해야 함 |
| 동적 코어 변경 | 쉬움 | Pod 재시작 필요 (cpu 변경 시) |
| 다른 워크로드 영향 | 시스템 데몬만 | 같은 노드의 다른 Pod도 |

**주의사항**:

- **K8s scheduler가 다른 Pod을 같은 코어에 배치**: BestEffort/Burstable Pod이 Guaranteed Pod 코어를 침범할 수 있음 → static 정책으로 차단
- **Live migration / 노드 교체**: 친화도 정보가 노드 교체 후 사라짐 → DaemonSet으로 노드별 친화도 자동화
- **VM 안의 컨테이너**: 호스트 → VM → 컨테이너 3단계 → 친화도 효과 보장 어려움 → 베어메탈 K8s 권장 (latency 결정적 워크로드)

**실전 패턴 (latency 결정적 마이크로서비스)**:
```yaml
# Node 레벨에서 isolcpus + CPU Manager static
# Pod 레벨에서 Guaranteed QoS + 정수 CPU 요청
# Topology Manager로 NUMA 정렬
# 추가로 CPU Pinning을 위한 CRD (예: NRI) 활용
```

이 영역은 linux-for-backend-deep-dive 레포의 cgroup·네임스페이스·kubelet 챕터와 연결됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: 락 경합의 하드웨어 비용](./04-lock-contention-cost.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 측정과 최적화 방법론 ➡️](../measurement-and-optimization/01-perf-deep-dive.md)**

</div>
