# 멀티코어 토폴로지 — 코어·L3·소켓의 실제 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 물리 코어(Physical Core)와 논리 코어(Logical Core)의 차이는 무엇이고, SMT/Hyper-Threading은 왜 존재하는가?
- L1·L2가 코어 전용이고 L3가 공유라는 사실이 프로그램 성능에 어떤 의미를 갖는가?
- 소켓(Socket)·다이(Die)·CCX(AMD Core Complex)란 무엇이고, 이 단위가 바뀔 때 어떤 비용이 생기는가?
- `lstopo --of console`과 `lscpu`로 내 머신의 실제 캐시·NUMA 구조를 어떻게 읽어야 하는가?
- 같은 소켓 내 코어들이라도 CCX가 다르면 왜 L3 접근 비용이 달라지는가?
- 토폴로지를 모르고 스레드를 배치할 때 어떤 숨은 비용이 발생하는가?

---

## 🔍 왜 이 개념이 중요한가

### "코어가 많으면 빠르다 — 그런데 왜 이 배치에서 더 빠른가?"

```
일반적인 인식:
  16코어 서버 → 16배 빠름
  스레드 16개 → 코어 16개를 꽉 채움
  → 최대 성능

실제 (토폴로지를 무시할 때):
  스레드 A와 B가 같은 데이터를 공유하며 협력
  OS 스케줄러가 A를 소켓 0, B를 소켓 1에 배치
  → A가 쓴 데이터가 B에게 QPI/UPI 버스를 통해 전달
  → NUMA 원격 접근: 로컬 L3 대비 2~3배 느림

  같은 두 스레드를 같은 CCX(L3 공유) 내 코어에 배치하면:
  → L3 캐시를 통한 직접 전달 (40 사이클 이내)

토폴로지를 아는 것과 모르는 것의 차이:
  알고 있다 → taskset/numactl로 명시적 배치
  모른다    → OS 스케줄러의 임의 배치 → 로터리

결론: 코어 개수 못지않게 "어떤 코어끼리 협력하는가"가 중요하다
```

---

## 😱 잘못된 이해

### Before: CPU는 균질한 코어들의 집합이다

```
잘못된 모델:
  [Core0] [Core1] [Core2] [Core3] [Core4] [Core5] [Core6] [Core7]
     ↑         ↑
   "모두 동일한 속도로 서로 통신"

이 모델이 틀린 이유:
  1. 논리 코어와 물리 코어를 구분 안 함
     → "8코어"가 실제로 4물리×2논리인지 8물리인지 모름
     → SMT 환경에서 두 논리 코어는 같은 실행 유닛 공유

  2. L3가 소켓 단위임을 모름
     → 두 코어가 서로 다른 소켓에 있으면 L3를 공유하지 않음
     → 캐시 일관성 유지에 훨씬 비싼 크로스-소켓 트래픽 발생

  3. AMD CCX 구조를 모름
     → Zen2/Zen3: 8코어 소켓도 CCX 단위로 L3가 쪼개짐
     → 같은 소켓이라도 CCX가 다른 코어끼리는 L3 공유 안 됨

  4. `lscpu`에서 "CPU(s): 32"를 보고 32물리 코어로 착각
     → 실제: 2 소켓 × 8 물리코어 × 2 HT = 32 논리 코어
```

---

## ✨ 올바른 이해

### After: 토폴로지는 계층적이고 거리마다 비용이 다르다

```
실제 구조 (2소켓 Intel Xeon 예시):

Socket 0                           Socket 1
┌─────────────────────────────┐    ┌─────────────────────────────┐
│  Core0    Core1    Core2    │    │  Core8    Core9    Core10   │
│  L1+L2   L1+L2   L1+L2     │    │  L1+L2   L1+L2   L1+L2     │
│  (전용)   (전용)  (전용)    │    │  (전용)  (전용)   (전용)   │
│  ────────────────────────   │    │  ────────────────────────   │
│           L3 (공유)         │    │          L3 (공유)          │
│           (소켓 내 전체)    │    │          (소켓 내 전체)     │
└──────────────┬──────────────┘    └──────────────┬──────────────┘
               └────── QPI/UPI ─────────────────────┘
                       (소켓 간 연결: ~40~80ns 추가 비용)

AMD Zen3 (단일 소켓, 8코어):
┌────────────────────────────────────────┐
│  Socket 0                              │
│  ┌──────────────────────────────────┐  │
│  │  CCX0 (4코어)                    │  │
│  │  Core0 Core1 Core2 Core3         │  │
│  │  L1+L2 × 4 (전용)               │  │
│  │  L3 32MB (CCX0 내 공유)          │  │
│  └──────────────────────────────────┘  │
│  ┌──────────────────────────────────┐  │
│  │  CCX1 (4코어)                    │  │
│  │  Core4 Core5 Core6 Core7         │  │
│  │  L1+L2 × 4 (전용)               │  │
│  │  L3 32MB (CCX1 내 공유)          │  │
│  └──────────────────────────────────┘  │
│  Infinity Fabric (CCX 간 연결)         │
└────────────────────────────────────────┘

Core0과 Core1: 같은 CCX → L3 공유 → ~40 사이클
Core0과 Core4: 다른 CCX → Infinity Fabric → ~60 사이클
Core0과 Socket1 Core: 다른 소켓 → QPI → ~150 사이클 이상
```

---

## 🔬 내부 동작 원리

### 1. SMT(Simultaneous Multi-Threading) / Hyper-Threading의 실체

```
물리 코어 1개에 논리 코어(HT) 2개가 공존하는 방식:

        물리 코어 (Physical Core)
  ┌───────────────────────────────────────┐
  │  프론트엔드 (Fetch/Decode)             │
  │   ↗ Thread 0의 IP (명령 포인터)        │
  │   ↘ Thread 1의 IP (명령 포인터)        │
  │                                       │
  │  레지스터 파일 × 2                     │
  │   [Thread 0 registers]                │
  │   [Thread 1 registers]                │
  │                                       │
  │  실행 유닛 (공유!)                     │
  │   ALU0  ALU1  FPU  Load  Store        │
  │                                       │
  │  L1d/L1i 캐시 (공유)                  │
  │  L2 캐시 (공유)                       │
  └───────────────────────────────────────┘

HT의 의도:
  Thread 0이 L2 미스로 대기 중일 때
  → 실행 유닛이 유휴 상태
  → Thread 1의 명령을 실행하여 파이프라인 활용률 향상

HT의 현실적 이득:
  compute-bound 워크로드: 거의 이득 없음 (이미 실행 유닛 포화)
  memory-bound 워크로드: 20~30% 처리량 향상 (대기 중에 타 스레드 실행)
  DB/웹서버 혼합 부하: 15~25% 향상

HT의 함정:
  L1/L2 캐시가 두 스레드에 분할됨
  → 스레드 하나가 쓸 수 있는 L1 용량이 반으로 줄어드는 효과
  → 캐시 집약적 코드에서 HT가 오히려 느릴 수 있음

perf로 HT 효과 측정:
  taskset -c 0    ./a.out  # HT 형제 코어 비어 있음
  taskset -c 0,1  ./a.out  # HT 형제 코어 동시 사용 (0,1이 같은 물리 코어)
  → 두 결과를 비교해 HT 이득/손실 정량화
```

### 2. L1/L2 전용 vs L3 공유의 의미

```
각 캐시 계층의 역할:

L1d (Data Cache, 코어 전용):
  크기: 32~64KB
  지연: ~4 사이클
  역할: 해당 코어의 가장 최근 데이터 보관
  특이점: 코어가 쓴 데이터를 다른 코어가 읽으려면
          MESI 프로토콜로 먼저 무효화 필요 (Ch3-01 참조)

L2 (코어 전용, Victim Cache 역할):
  크기: 256KB~4MB (세대마다 다름)
  지연: ~12 사이클
  역할: L1에서 쫓겨난 라인을 임시 보관
  특이점: 여전히 코어 전용 → 코어 간 직접 공유 안 됨

L3 (소켓 공유, Last Level Cache):
  크기: 8~64MB
  지연: ~40 사이클
  역할: 소켓 내 모든 코어가 공유 → 코어 간 데이터 전달의 고속도로
  특이점:
    MESI 상태가 L3 수준에서 조율됨 (Inclusive LLC의 경우)
    코어 A가 쓴 데이터를 코어 B가 읽을 때:
      코어 A L1(M) → L3(S) → 코어 B L1(S)
      → L3가 중재자 역할

L3 공유가 만드는 이점:
  ✅ 같은 소켓 내 코어 간 데이터 전달이 DRAM을 거치지 않음
  ✅ 협력하는 스레드가 같은 데이터를 공유할 때 효율적

L3 공유가 만드는 단점:
  ❌ 고대역폭 L3 접근이 경합할 경우 대역폭 포화
  ❌ LLC 오염(Pollution): 한 코어의 스트리밍 워크로드가 전체 L3를 오염
```

### 3. AMD CCX(Core Complex) — 소켓 내 또 다른 경계

```
AMD Zen2 (EPYC Rome, 최대 64코어/소켓):

  소켓
  ┌──────────────────────────────────────────────────────────────┐
  │  CCD0 (Core Chiplet Die)                                     │
  │  ┌────────────────┐  ┌────────────────┐                     │
  │  │  CCX0 (4코어)  │  │  CCX1 (4코어)  │                     │
  │  │  L3: 16MB      │  │  L3: 16MB      │                     │
  │  └────────┬───────┘  └───────┬────────┘                     │
  │           └─── Infinity Fabric ────┘                         │
  │                                                              │
  │  CCD1 ~ CCD7 (동일 구조 × 7개 더)                           │
  │                                                              │
  │  I/O Die (중앙 허브)                                         │
  │  ┌───────────────────────────────────────────────────────┐  │
  │  │  Infinity Fabric 스위치 + 메모리 컨트롤러 + PCIe      │  │
  │  └───────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────┘

Zen3에서의 변화:
  CCX → CCD 전체가 L3를 공유 (8코어가 32MB L3 공유)
  → Zen2의 크로스-CCX 지연 제거
  → Redis, DB 같은 공유 캐시 의존 워크로드에서 큰 향상

결론: 같은 소켓이라도 CCD/CCX 경계를 넘으면 추가 지연 발생
      lstopo로 이 경계를 시각화해야 최적 스레드 배치 가능
```

### 4. `lscpu` 출력 읽기

```bash
# lscpu 출력 예시 (2소켓 Intel Xeon, Hyper-Threading 활성):
Architecture:                    x86_64
CPU(s):                          64          ← 총 논리 코어 수
On-line CPU(s) list:             0-63
Thread(s) per core:              2           ← HT 활성 (물리 1 = 논리 2)
Core(s) per socket:              16          ← 물리 코어 16개/소켓
Socket(s):                       2           ← 소켓 2개
NUMA node(s):                    2           ← NUMA 노드 2개
Vendor ID:                       GenuineIntel
Model name:                      Intel(R) Xeon(R) Gold 6226R

L1d cache:                       32K         ← 물리코어별 32KB
L1i cache:                       32K
L2 cache:                        1024K       ← 물리코어별 1MB
L3 cache:                        22528K      ← 소켓 공유 22MB

NUMA node0 CPU(s):               0-15,32-47  ← 소켓0: 물리코어 0-15 + HT 파트너
NUMA node1 CPU(s):               16-31,48-63 ← 소켓1: 물리코어 16-31 + HT 파트너

# 물리 코어 매핑 확인:
# CPU 0 (논리) ↔ 물리코어 0, NUMA 노드 0
# CPU 32 (논리) ↔ 물리코어 0의 HT 파트너, NUMA 노드 0
# → CPU 0과 CPU 32가 같은 물리코어를 공유함

# /sys 파일시스템으로 더 정밀하게:
cat /sys/devices/system/cpu/cpu0/topology/core_id        # 물리코어 ID
cat /sys/devices/system/cpu/cpu0/topology/physical_package_id # 소켓 ID
ls /sys/devices/system/cpu/cpu0/cache/                   # 각 캐시 레벨 정보
cat /sys/devices/system/cpu/cpu0/cache/index2/shared_cpu_map # L3 공유 코어 맵
```

---

## 💻 실전 실험

### 실험 1: lstopo로 캐시·NUMA 구조 시각화

```bash
# hwloc 패키지 설치
sudo apt-get install hwloc   # Ubuntu/Debian
sudo yum install hwloc       # CentOS/RHEL

# 텍스트 형식으로 전체 토폴로지 출력
lstopo --of console

# 예상 출력 (단일 소켓 8코어 Zen3):
# Machine (62GB total)
#   NUMANode L#0 (P#0 62GB)
#   Package L#0
#     L3 L#0 (32MB)
#       L2 L#0 (512KB) + L1d L#0 (32KB) + L1i L#0 (32KB) + Core L#0
#         PU L#0 (P#0)    ← 논리 코어 0 (물리코어 0, HT 없음 또는 첫 번째 스레드)
#         PU L#1 (P#8)    ← 논리 코어 8 (같은 물리코어의 HT 파트너)
#       L2 L#1 (512KB) + L1d L#1 (32KB) + L1i L#1 (32KB) + Core L#1
#         PU L#2 (P#1)
#         PU L#3 (P#9)
#       ...

# SVG 이미지로 저장 (그래픽 확인)
lstopo --of svg topology.svg

# 2소켓 NUMA 시스템에서의 출력 차이:
# NUMANode L#0 (P#0) → Package L#0 → L3 L#0 → [코어 목록]
# NUMANode L#1 (P#1) → Package L#1 → L3 L#1 → [코어 목록]
# → 두 NUMANode 사이에 QPI/UPI 연결 표시
```

### 실험 2: HT 형제 코어 식별 및 공유 코어 매핑

```bash
# HT 형제 코어 확인
for cpu in $(ls /sys/devices/system/cpu/ | grep "^cpu[0-9]"); do
    core=$(cat /sys/devices/system/cpu/$cpu/topology/core_id 2>/dev/null)
    pkg=$(cat /sys/devices/system/cpu/$cpu/topology/physical_package_id 2>/dev/null)
    if [ -n "$core" ]; then
        echo "$cpu: socket=$pkg, physical_core=$core"
    fi
done | sort -t= -k2,2n -k3,3n

# 예상 출력 (4물리코어 × 2HT = 8논리코어):
# cpu0: socket=0, physical_core=0
# cpu4: socket=0, physical_core=0   ← cpu0과 같은 물리코어!
# cpu1: socket=0, physical_core=1
# cpu5: socket=0, physical_core=1   ← cpu1과 같은 물리코어!
# cpu2: socket=0, physical_core=2
# cpu6: socket=0, physical_core=2
# cpu3: socket=0, physical_core=3
# cpu7: socket=0, physical_core=3

# L3 캐시 공유 범위 확인 (비트맵으로 표시)
cat /sys/devices/system/cpu/cpu0/cache/index3/shared_cpu_map
# 예: 000000ff → 코어 0~7이 L3 공유 (단일 소켓)
# 예: 0000000f → 코어 0~3만 L3 공유 (AMD CCX)
```

### 실험 3: 토폴로지를 모를 때 vs 알 때의 캐시 공유 효율

```c
// topology_bench.c
// 두 스레드가 공유 배열을 읽는 패턴
// 같은 물리코어 HT vs 다른 물리코어 vs 다른 소켓에 배치 시 비교
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <time.h>
#include <sched.h>

#define ARRAY_SIZE (1 << 22)  // 16MB — L3에 들어가는 크기
#define ITERATIONS 20

volatile int shared_array[ARRAY_SIZE];
volatile long sum_result[2];

typedef struct { int thread_id; int cpu_id; } thread_arg_t;

void* reader_thread(void* arg) {
    thread_arg_t* ta = (thread_arg_t*)arg;

    // 지정된 CPU에 고정
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(ta->cpu_id, &cpuset);
    pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);

    long s = 0;
    for (int iter = 0; iter < ITERATIONS; iter++)
        for (int i = 0; i < ARRAY_SIZE; i++)
            s += shared_array[i];
    sum_result[ta->thread_id] = s;
    return NULL;
}

double run_with_cpus(int cpu0, int cpu1) {
    pthread_t t[2];
    thread_arg_t args[2] = {{0, cpu0}, {1, cpu1}};

    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    pthread_create(&t[0], NULL, reader_thread, &args[0]);
    pthread_create(&t[1], NULL, reader_thread, &args[1]);
    pthread_join(t[0], NULL);
    pthread_join(t[1], NULL);

    clock_gettime(CLOCK_MONOTONIC, &end);
    return (end.tv_sec - start.tv_sec) * 1e3 +
           (end.tv_nsec - start.tv_nsec) / 1e6;
}

int main() {
    // 배열 초기화
    for (int i = 0; i < ARRAY_SIZE; i++) shared_array[i] = i;

    // CPU 번호는 lstopo/lscpu로 확인 후 조정 필요
    // 예: cpu0=0, cpu1=1 (같은 물리코어 HT 파트너)
    //     cpu0=0, cpu1=2 (같은 소켓 다른 물리코어)
    //     cpu0=0, cpu1=8 (다른 소켓, 2소켓 시스템)

    printf("같은 HT 코어 (cpu 0,1):    %.1f ms\n", run_with_cpus(0, 1));
    printf("같은 소켓 (cpu 0,2):       %.1f ms\n", run_with_cpus(0, 2));
    // 2소켓 시스템인 경우:
    // printf("다른 소켓 (cpu 0,8):     %.1f ms\n", run_with_cpus(0, 8));
    return 0;
}
// gcc -O2 -pthread -o topology_bench topology_bench.c
// 결과 해석: 같은 L3를 공유할수록 두 스레드의 L3 로드 비용이 중복 제거됨
```

### 실험 4: AMD CCX 경계를 넘는 비용 측정

```bash
# AMD 시스템에서 CCX 경계 확인
# Zen3 (CCX = CCD 전체, 8코어가 L3 공유):
lstopo --of console | grep -E "L3|Core|PU"

# CCX 내부 통신 (같은 L3 공유):
taskset -c 0,1 ./topology_bench   # 같은 CCX

# CCX 외부 통신 (Infinity Fabric 경유):
taskset -c 0,4 ./topology_bench   # 다른 CCD (Zen2) 또는 다른 CCX

# 지연 차이를 numactl로 직접 확인 (Zen2 EPYC):
numactl --hardware
# Node 0: 거리 = 10 (로컬), 12 (같은 소켓 다른 CCD), 32 (다른 소켓)
```

---

## 📊 성능 비교

```
스레드 배치에 따른 공유 데이터 접근 처리량 비교 (2스레드, 16MB 공유 배열):

  배치                      L3 히트율   처리량(GB/s)   상대 성능
  ─────────────────────────────────────────────────────────────
  같은 물리코어(HT 파트너)    ~99%       ~55 GB/s       1.0x
  같은 소켓, 다른 물리코어    ~98%       ~50 GB/s       0.9x
  같은 소켓, CCX 경계 넘음    ~95%       ~40 GB/s       0.7x  (Zen2)
  다른 소켓 (NUMA 원격)       ~30%       ~18 GB/s       0.33x

  ※ HT 파트너가 L3를 공유하지만 실행 유닛도 공유 → compute 경합 발생
  ※ 최적 배치: 같은 소켓, 다른 물리코어 (L3 공유 + 실행 유닛 독립)

lscpu 숫자 해석 예시:
  CPU(s): 64
  Thread(s) per core: 2   → 물리코어 = 64/2 = 32
  Core(s) per socket: 16  → 소켓 수 = 32/16 = 2
  Socket(s): 2            ✓
  NUMA node(s): 2         ✓ (소켓당 NUMA 노드 1개, 이 경우)
```

---

## ⚖️ 트레이드오프

```
SMT/Hyper-Threading 활성화 트레이드오프:

활성화가 유리한 경우:
  ✅ memory-bound 워크로드: 한 스레드 대기 중 다른 스레드 실행
  ✅ I/O 대기가 많은 서버 워크로드: 처리량 15~30% 향상
  ✅ 혼합 부하 (DB 쿼리 + 백그라운드 작업): 전반적 활용률 향상

비활성화가 유리한 경우:
  ❌ compute-bound HPC: 실행 유닛 경합으로 오히려 느림
  ❌ 캐시 집약적 코드: L1/L2를 절반씩 나눔 → 히트율 저하
  ❌ 보안 민감 환경: Spectre/MDS 취약점 일부가 HT를 통해 발생

BIOS에서 HT 비활성화:
  /sys/devices/system/cpu/cpuX/online → 0 (런타임 비활성화)
  또는 GRUB: nosmt (전체 비활성화)

CCX/CCD 인식 스레드 배치의 트레이드오프:
  ✅ 협력 스레드를 같은 CCX에 묶기: 공유 캐시 효율 최대화
  ❌ 모든 스레드를 한 CCX에 몰기: CCX 내 L3 대역폭 포화
  → 적정 배치: 독립적 워크로드는 각 CCX에 분산
                공유 데이터 집약 워크로드는 같은 CCX에 집중

lstopo 기반 배치 자동화:
  hwloc-bind --cpubind numa:0.core:0-3 ./a.out
  → NUMA 노드 0의 코어 0~3에 프로세스 바인딩
```

---

## 📌 핵심 정리

```
멀티코어 토폴로지 핵심:

물리 코어 vs 논리 코어:
  HT/SMT 활성 시: 논리 코어 = 물리 코어 × 2
  같은 물리코어의 두 논리 코어는 실행 유닛과 L1/L2 공유
  → compute-bound에서 HT 이득 없음, memory-bound에서 ~20% 이득

캐시 소유 구조:
  L1d/L1i:  각 물리코어 전용 (~32KB)
  L2:       각 물리코어 전용 (256KB~4MB)
  L3(LLC):  소켓 내 전체 공유 (8~64MB)
  AMD CCX:  CCX 내 코어끼리만 L3 공유 (Zen2), Zen3+는 CCD 전체

AMD 계층:
  물리코어 → CCX → CCD → 소켓
  CCX 경계를 넘으면: Infinity Fabric으로 ~20ns 추가
  소켓 경계를 넘으면: QPI/UPI로 ~40~80ns 추가

진단 명령어:
  lscpu                      → 소켓/코어/HT 구조 파악
  lstopo --of console        → 캐시·NUMA 계층 트리 시각화
  cat /sys/.../shared_cpu_map → L3 공유 코어 비트맵
  numactl --hardware         → NUMA 노드 간 거리 행렬

다음 단계:
  소켓 간 거리(NUMA)가 만드는 비용 → Ch6-02
  같은 라인을 두 코어가 쓸 때 → Ch3-02 False Sharing
  협력 스레드를 같은 캐시에 묶기 → Ch6-05 Thread Affinity
```

---

## 🤔 생각해볼 문제

**Q1.** `lscpu` 출력에서 `CPU(s): 32`, `Thread(s) per core: 2`, `Core(s) per socket: 8`, `Socket(s): 2`를 보았다. 물리코어는 몇 개이고, HT 비활성화 시 논리코어는 몇 개가 되는가? 또한 이 시스템에서 두 스레드가 같은 L3를 공유하려면 어느 논리코어 번호에 배치해야 하는가?

<details>
<summary>해설 보기</summary>

**물리코어 계산:**
- 총 논리코어: 32
- HT 비율: Thread(s) per core = 2
- 물리코어 = 32 / 2 = **16개** (소켓 2개 × 코어 8개/소켓)

**HT 비활성화 시:**
- 논리코어 = 물리코어 = 16개
- `echo 0 > /sys/devices/system/cpu/cpuX/online`으로 HT 파트너 코어를 오프라인화

**L3 공유 범위 확인:**
- `NUMA node0 CPU(s): 0-7,16-23` — 소켓0의 물리코어(0-7)와 HT 파트너(16-23)
- `NUMA node1 CPU(s): 8-15,24-31` — 소켓1

같은 L3를 공유하려면 **같은 NUMA 노드 내 코어**에 배치해야 합니다.
- CPU 0과 CPU 3 → 둘 다 소켓0 → L3 공유 O
- CPU 0과 CPU 8 → 소켓0 vs 소켓1 → L3 공유 X (NUMA 원격 접근)
- CPU 0과 CPU 16 → 같은 물리코어의 HT 파트너 → L3 공유 O (하지만 실행 유닛도 공유)

`cat /sys/devices/system/cpu/cpu0/cache/index3/shared_cpu_map`으로 정확한 공유 범위 확인이 가능합니다.

</details>

---

**Q2.** AMD Zen2 EPYC 서버에서 Redis 같은 인메모리 DB를 운영할 때, 프로세스를 어떤 CCX 배치 전략으로 실행하는 것이 유리한가? Zen3에서 이 전략이 어떻게 달라지는가?

<details>
<summary>해설 보기</summary>

**Zen2 EPYC에서의 문제:**
Zen2는 4코어 CCX 단위로 L3가 분리됩니다(각 CCX가 독립된 16MB L3 보유). Redis는 해시 테이블 전체를 캐시에 올려두는 것이 이상적인데, Zen2에서 프로세스가 여러 CCX를 넘나들면 L3 캐시 일관성 유지를 위한 Infinity Fabric 트래픽이 발생해 지연이 늘어납니다.

**Zen2 최적 전략:**
- `numactl --cpunodebind=0 --membind=0` 으로 단일 NUMA 노드에 고정
- 더 나아가 `taskset -c 0-7`로 특정 CCD(2개 CCX)에 고정
- Redis 인스턴스를 CCX 수만큼 여러 개 띄우고 각자 다른 CCX를 담당하게 하는 샤딩 전략도 유효

**Zen3에서의 변화:**
Zen3는 CCD 전체(8코어)가 하나의 32MB L3를 공유합니다. 크로스-CCX Infinity Fabric 홉이 사라져 동일 CCD 내 코어 간 L3 공유가 훨씬 빠릅니다. 따라서 Zen3에서는 CCX 경계를 신경 쓸 필요 없이 CCD 단위로만 묶어주면 됩니다. 실제로 Zen3 출시 후 Redis 벤치마크에서 Zen2 대비 L3 히트율이 크게 향상된 것이 보고되었습니다.

</details>

---

**Q3.** HT(Hyper-Threading) 환경에서 CPU-bound 암호화 워크로드(AES-NI)를 실행 중이다. 논리코어 0과 논리코어 1(같은 물리코어의 HT 파트너)에 각각 암호화 스레드를 하나씩 배치했을 때 예상되는 성능과, 논리코어 0과 논리코어 2(다른 물리코어)에 배치했을 때의 성능 차이를 설명하라.

<details>
<summary>해설 보기</summary>

**같은 물리코어(HT 파트너, cpu 0 + cpu 1):**
AES-NI는 전용 실행 유닛(AES unit)을 사용합니다. 같은 물리코어 내 두 논리 스레드는 이 유닛을 공유합니다. 한 사이클에 하나의 AES 명령만 실행 가능하므로, 두 스레드가 경합합니다.

결과: 단일 스레드 대비 처리량 증가가 거의 없거나, L1/L2 캐시 경합까지 더해지면 오히려 단일 스레드보다 느려질 수 있습니다. `perf stat -e cpu-cycles ./encrypt_bench`로 IPC를 확인하면 0.5 근처가 나옵니다.

**다른 물리코어(cpu 0 + cpu 2):**
각 코어가 독립된 AES 실행 유닛을 보유합니다. 두 스레드가 동시에 각자의 AES 명령을 실행할 수 있습니다.

결과: 처리량이 거의 2배에 가깝게 증가합니다. IPC도 각 코어에서 독립적으로 측정되므로 단일 스레드와 비슷한 수준을 유지합니다.

**실측 방법:**
```bash
taskset -c 0,1 ./aes_bench   # HT 파트너
taskset -c 0,2 ./aes_bench   # 다른 물리코어
```
일반적으로 compute-bound 워크로드에서 HT를 다른 물리코어로 분리하는 것이 2배에 가까운 성능을 냅니다. 이는 `linux-for-backend-deep-dive` 레포의 CPU 스케줄러 섹션에서 다루는 SMT 인식 스케줄링(SMT-aware scheduling)의 근거이기도 합니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: 처리량 vs 지연시간](../simd-data-parallelism/05-throughput-vs-latency.md)** | **[홈으로 🏠](../README.md)** | **[다음: NUMA 원격 메모리 접근 ➡️](./02-numa-remote-access.md)**

</div>
