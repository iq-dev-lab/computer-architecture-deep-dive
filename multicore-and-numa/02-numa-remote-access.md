# NUMA 원격 메모리 접근 비용 — 로컬 대비 1.5~3배 느린 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- NUMA(Non-Uniform Memory Access)란 무엇이고, 왜 UMA 대신 사용하는가?
- NUMA 노드 간 메모리 접근이 로컬보다 1.5~3배 느린 이유는 물리적으로 무엇인가?
- Intel QPI/UPI, AMD Infinity Fabric의 역할과 대역폭 한계는 무엇인가?
- `numactl --hardware`의 거리(distance) 행렬을 어떻게 읽어야 하는가?
- `--cpunodebind=0 --membind=0` vs `--membind=1`로 로컬/원격 접근을 실제 측정하는 방법은?
- first-touch 정책이란 무엇이고, 잘못된 첫 접근이 어떤 함정을 만드는가?

---

## 🔍 왜 이 개념이 중요한가

### "두 배로 키운 서버에서 처리량이 두 배가 안 나온다"

```
일반적인 기대 (2소켓 서버 도입 시):
  소켓 1개: 16코어, 64GB DRAM → 처리량 X
  소켓 2개: 32코어, 128GB DRAM → 처리량 2X

실제:
  소켓 1개: 처리량 X
  소켓 2개: 처리량 1.3X ~ 1.6X

원인 중 하나: NUMA 원격 접근

  JVM 프로세스가 힙을 초기화할 때:
    main 스레드가 소켓0에서 실행
    → 힙의 first-touch 모두 소켓0의 DRAM에 할당
    → 나중에 소켓1에서 실행되는 GC/워커 스레드가
      소켓0의 DRAM을 원격으로 접근
    → 모든 힙 접근이 UPI를 경유 → 1.5~2배 느려짐

  데이터베이스 서버 (PostgreSQL, MySQL):
    연결 처리 스레드가 소켓1
    공유 버퍼 캐시가 소켓0 메모리에 할당됨
    → 모든 캐시 접근에 NUMA 원격 비용 발생

  결론: 단순히 소켓을 늘려도 NUMA를 인식하지 않으면
         메모리 접근 비용이 병목이 됨
```

---

## 😱 잘못된 이해

### Before: 64GB RAM은 어디서 접근해도 같은 속도다

```
잘못된 모델:
  [CPU Core 0] ─────┐
  [CPU Core 1] ─────┤
  [CPU Core 2] ─────┼──── 메모리 버스 ──── [128 GB DRAM]
  [CPU Core 3] ─────┤
  ...               │
  [CPU Core 31] ────┘
  → "모든 코어가 같은 DRAM에 동일한 속도로 접근"

실제 2소켓 구조:
  소켓 0                    소켓 1
  ┌─────────────┐           ┌─────────────┐
  │  Core 0-15  │ ←──QPI──→ │  Core 16-31 │
  │             │           │             │
  │ 메모리       │           │ 메모리       │
  │ 컨트롤러 0   │           │ 컨트롤러 1   │
  └──────┬──────┘           └──────┬──────┘
         │                         │
      DRAM 0                    DRAM 1
     (64GB)                    (64GB)

Core 0이 DRAM 0 접근: 로컬 → ~80 ns
Core 0이 DRAM 1 접근: 원격 → QPI 경유 → ~150~200 ns
                            → 1.8~2.5배 느림

잘못된 인식이 만드는 문제:
  numactl 없이 프로세스 배치 → OS가 임의의 NUMA 노드에 메모리 할당
  → 접근 패턴에 따라 대부분 원격 접근이 될 수 있음
  → 성능 예측 불가
```

---

## ✨ 올바른 이해

### After: NUMA는 거리가 다른 메모리 계층의 또 다른 단계다

```
완전한 메모리 계층 (NUMA 포함):

  레지스터   →  L1   →  L2   →  L3   →  로컬 DRAM  →  원격 DRAM
  0 cy         4 cy   12 cy   40 cy    ~200 cy       ~300~400 cy

  원격 DRAM = 로컬 DRAM + QPI/UPI 왕복 지연

QPI/UPI 경로:
  Core0 (소켓0) → 메모리 요청 발행
  → 로컬 캐시 미스 → 로컬 메모리 컨트롤러 미스
  → QPI/UPI 패킷으로 소켓1 전송 (~40ns)
  → 소켓1 메모리 컨트롤러에서 DRAM 접근 (~80ns)
  → 결과를 QPI로 소켓0에 전송 (~40ns)
  → 총 ~160~200ns (로컬 ~80ns 대비 2~2.5배)

numactl distance 행렬 예시:
  node distances:
  node   0   1
     0:  10  21    ← 소켓0에서: 로컬(10), 원격(21)
     1:  21  10    ← 소켓1에서: 원격(21), 로컬(10)

  거리 10 = 기준값 (로컬)
  거리 21 = 2.1배 느림
  4소켓 시스템에서는 거리 31~41 이상도 가능

올바른 접근:
  numactl로 명시적으로 로컬 노드 메모리 사용 강제
  first-touch를 올바른 코어에서 수행
  NUMA-aware 메모리 할당자 사용 (jemalloc, glibc의 mbind)
```

---

## 🔬 내부 동작 원리

### 1. QPI/UPI(Intel)와 Infinity Fabric(AMD)의 동작

```
Intel UPI (Ultra Path Interconnect, Skylake+):

  소켓 0                              소켓 1
  ┌─────────────────────┐            ┌─────────────────────┐
  │  Ring Bus / Mesh    │            │  Ring Bus / Mesh    │
  │  L3 캐시 (22MB)     │            │  L3 캐시 (22MB)     │
  │                     │            │                     │
  │  Home Agent (HA)    │←── UPI ───→│  Home Agent (HA)    │
  │  (캐시 일관성 중재)  │  ~10.4GT/s │  (캐시 일관성 중재)  │
  │                     │   ×2 링크  │                     │
  │  iMC                │            │  iMC                │
  │  (통합 메모리 컨트롤러)│            │  (통합 메모리 컨트롤러)│
  └──────────┬──────────┘            └──────────┬──────────┘
             │                                   │
           DRAM 채널 ×4                        DRAM 채널 ×4
             │                                   │
    DDR4 (최대 128GB)                  DDR4 (최대 128GB)

UPI 데이터 흐름 (원격 읽기):
  1. Core0: 주소 0x...가 소켓1 메모리 범위임을 물리 주소로 인식
  2. 소켓0 Home Agent: UPI로 소켓1에 Read Request 전송
  3. 소켓1 Home Agent: 자신의 L3 조회 → 미스 → iMC로 전달
  4. 소켓1 iMC → DRAM → 64바이트 캐시라인 반환
  5. 소켓1 Home Agent → UPI로 소켓0에 Data Response
  6. 소켓0: L3에 캐시라인 채움 → Core0에 전달

UPI 대역폭 한계:
  Intel Xeon Platinum: UPI 3링크 × 10.4 GT/s × 8B = ~250 GB/s (이론)
  실제 양방향 포화 대역폭: ~100~150 GB/s
  vs 로컬 DRAM 대역폭: ~200~300 GB/s/소켓
  → 원격 접근은 대역폭도 절반 이하

AMD Infinity Fabric:
  EPYC: XGMI(Cross-GPU/CPU Memory Interface) 링크
  Zen3: CCD 간 + 소켓 간 모두 Infinity Fabric 사용
  대역폭: ~최대 400 GB/s (소켓 내) vs ~100 GB/s (소켓 간)
```

### 2. first-touch 메모리 정책의 함정

```
Linux의 기본 NUMA 메모리 정책: first-touch

  가상 메모리 공간을 먼저 할당 (물리 메모리 매핑 없음)
  → 최초로 해당 주소를 접근(터치)하는 코어의 NUMA 노드에
    물리 페이지 할당

코드 예시:
  // main 스레드: CPU 0 (소켓0)에서 실행
  char* buf = malloc(1 GB);   // 가상 메모리만 예약

  // 초기화: main 스레드가 직접 초기화
  memset(buf, 0, 1 GB);       // ← 소켓0의 메모리에 물리 페이지 할당!

  // 워커 스레드 생성: CPU 16-31 (소켓1)에서 실행
  for (int i = 0; i < 16; i++)
      pthread_create(..., worker, buf);  // 소켓1 스레드들이 buf 접근

  // 결과: buf의 모든 페이지가 소켓0 메모리에 있음
  //        소켓1 스레드가 접근할 때마다 UPI 원격 접근 발생!

올바른 해결책:
  방법 1: 워커 스레드가 직접 자신의 영역 초기화
    void* worker(void* arg) {
        int id = get_thread_id();
        size_t chunk = TOTAL / NUM_WORKERS;
        // 내 영역을 직접 초기화 → 내 소켓 메모리에 할당
        memset(buf + id * chunk, 0, chunk);
        // 이후 buf 접근은 로컬
    }

  방법 2: numactl --localalloc
    numactl --localalloc ./a.out
    → 각 스레드가 현재 NUMA 노드의 메모리를 할당받음

  방법 3: mbind/numa_alloc_onnode 직접 사용
    #include <numaif.h>
    numa_alloc_onnode(size, node_id);  // 특정 노드에 강제 할당

JVM에서의 first-touch 함정:
  JVM 시작 시 main 스레드가 힙 전체를 -Xms 크기만큼 초기화
  → 모든 힙 페이지가 main 스레드의 소켓에 배치
  → 멀티 스레드 GC가 원격 노드 메모리를 청소
  해결: -XX:+UseNUMA (G1/Parallel GC에서 NUMA 인식 힙 관리)
         numactl --interleave=all (페이지를 노드 간 인터리브)
```

### 3. NUMA-aware vs NUMA-oblivious 메모리 할당

```c
// NUMA 할당 API 비교

// 1. 기본 malloc (NUMA 무지)
char* p1 = malloc(1024 * 1024 * 1024);
// → first-touch 정책 → 할당 시점의 현재 스레드 노드

// 2. numa_alloc_local (현재 스레드의 NUMA 노드)
#include <numa.h>
char* p2 = numa_alloc_local(1024 * 1024 * 1024);
// → 명시적으로 현재 스레드의 노드에 할당

// 3. numa_alloc_onnode (특정 노드 지정)
char* p3 = numa_alloc_onnode(size, 0);  // 노드 0에 강제 할당

// 4. mbind (기존 메모리의 정책 변경)
#include <numaif.h>
unsigned long nodemask = 1UL;  // 노드 0
mbind(ptr, size,
      MPOL_BIND,        // 특정 노드에 바인드
      &nodemask, 2,     // 노드 0 (비트 0 세트)
      MPOL_MF_MOVE);    // 기존 페이지도 이동

// 5. numactl CLI (프로그램 수정 없이)
// numactl --membind=0 ./a.out          → 노드 0 메모리만 사용
// numactl --interleave=all ./a.out     → 모든 노드에 라운드로빈
// numactl --preferred=0 ./a.out        → 가능하면 노드 0, 부족하면 다른 노드

// 설치: sudo apt-get install libnuma-dev
// 컴파일: gcc -O2 -o numa_test numa_test.c -lnuma
```

### 4. NUMA 거리 행렬 이해

```
numactl --hardware 출력 예시 (4소켓 서버):

available: 4 nodes (0-3)
node 0 cpus: 0 1 2 3 4 5 6 7 32 33 34 35 36 37 38 39
node 0 size: 95420 MB
node 0 free: 87234 MB
node 1 cpus: 8 9 10 11 12 13 14 15 40 41 42 43 44 45 46 47
node 1 size: 96020 MB
...

node distances:
node   0   1   2   3
   0:  10  21  31  31   ← 노드0 기준: 로컬(10), 노드1(21), 노드2/3(31)
   1:  21  10  31  31
   2:  31  31  10  21
   3:  31  31  21  10

해석:
  거리 10: 로컬 접근 (동일 소켓의 DRAM) → 기준 1배
  거리 21: UPI 1홉 → ~2.1배 지연
  거리 31: UPI 2홉 (소켓0 → 소켓1 → 소켓2) → ~3.1배 지연

  4소켓에서 노드0의 코어가 노드2/3 메모리 접근:
  소켓0 → UPI → 소켓1 → UPI → 소켓2/3
  → 2홉 = 원격 지연 × 2 + UPI 오버헤드

  최악의 배치:
    프로세스 코어: 소켓0 (노드0)
    메모리: 소켓2 또는 소켓3 (노드2/3)
    → 모든 L3 미스가 2홉 UPI 트래픽 → ~3배 느린 DRAM
```

---

## 💻 실전 실험

### 실험 1: numactl --hardware로 현재 시스템 NUMA 구조 확인

```bash
# NUMA 하드웨어 정보 확인
numactl --hardware

# NUMA 정책 현재 상태 확인
numactl --show

# 노드별 메모리 통계 (이미 원격 접근이 얼마나 발생했는지)
numastat
# 출력 예시:
# Per-node numastat info (in MBs):
#                           Node 0          Node 1
# numa_hit               123456.789      98765.432
# numa_miss               12345.678       9876.543  ← 원격 할당 발생!
# numa_foreign             9876.543      12345.678
# interleave_hit              12.345         11.234
# local_node             120000.000      96000.000
# other_node              12345.678       9876.543  ← 이게 많으면 NUMA 비효율

# 프로세스별 NUMA 접근 통계
numastat -p <PID>

# 페이지별 NUMA 위치 확인
cat /proc/<PID>/numa_maps | head -20
# 예: 7f1234560000 default file=... mapped=512 N0=400 N1=112
#     N0=400: 400페이지가 노드0, N1=112: 112페이지가 노드1
```

### 실험 2: 로컬 vs 원격 메모리 접근 처리량 측정

```c
// numa_bandwidth_test.c
// numactl로 로컬/원격 조건을 바꿔가며 같은 코드 실행
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <stdint.h>

#define SIZE_MB  512
#define SIZE     ((size_t)SIZE_MB * 1024 * 1024)
#define ITER     5

double measure_bandwidth(char* buf, size_t size) {
    // 순차 읽기 처리량 측정
    volatile long long sum = 0;
    long long* p = (long long*)buf;
    size_t n = size / sizeof(long long);

    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    for (int iter = 0; iter < ITER; iter++)
        for (size_t i = 0; i < n; i++)
            sum += p[i];

    clock_gettime(CLOCK_MONOTONIC, &end);

    double sec = (end.tv_sec - start.tv_sec) +
                 (end.tv_nsec - start.tv_nsec) * 1e-9;
    double total_gb = (double)SIZE_MB * ITER / 1024.0;
    return total_gb / sec;  // GB/s
}

int main() {
    char* buf = (char*)malloc(SIZE);
    if (!buf) { perror("malloc"); return 1; }

    // first-touch: 현재 프로세스의 NUMA 노드에 할당
    memset(buf, 0xAB, SIZE);

    printf("버퍼 주소: %p, 크기: %zu MB\n", buf, SIZE / (1024*1024));
    printf("처리량: %.2f GB/s\n", measure_bandwidth(buf, SIZE));

    free(buf);
    return 0;
}
// gcc -O2 -o numa_bw numa_bandwidth_test.c

// ========== 실행 방법 ==========
// 로컬 접근 (CPU와 메모리 모두 노드0):
// numactl --cpunodebind=0 --membind=0 ./numa_bw

// 원격 접근 (CPU는 노드0, 메모리는 노드1):
// numactl --cpunodebind=0 --membind=1 ./numa_bw

// 인터리브 (두 노드에 분산):
// numactl --cpunodebind=0 --interleave=all ./numa_bw

// 예상 결과 (Intel 2소켓 Xeon):
// 로컬:    ~45 GB/s
// 원격:    ~25 GB/s (약 1.8배 차이)
// 인터리브: ~35 GB/s (평균값)
```

### 실험 3: first-touch 함정 재현

```c
// first_touch_trap.c
// main 스레드가 초기화하고 워커 스레드가 접근하는 패턴
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <sched.h>

#define SIZE_MB   256
#define SIZE      ((size_t)SIZE_MB * 1024 * 1024)
#define WORKERS   4
#define ITERS     10

char* shared_buf;

typedef struct { int worker_id; int cpu_id; double result_gbps; } worker_arg_t;

void pin_to_cpu(int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
}

void* bad_worker(void* arg) {
    worker_arg_t* wa = (worker_arg_t*)arg;
    pin_to_cpu(wa->cpu_id);  // 소켓1 코어에 고정

    volatile long long sum = 0;
    long long* p = (long long*)shared_buf;
    size_t n = SIZE / sizeof(long long) / WORKERS;
    size_t offset = wa->worker_id * n;

    struct timespec s, e;
    clock_gettime(CLOCK_MONOTONIC, &s);
    for (int iter = 0; iter < ITERS; iter++)
        for (size_t i = 0; i < n; i++)
            sum += p[offset + i];
    clock_gettime(CLOCK_MONOTONIC, &e);

    double sec = (e.tv_sec - s.tv_sec) + (e.tv_nsec - s.tv_nsec) * 1e-9;
    wa->result_gbps = (double)SIZE_MB * ITERS / 1024.0 / WORKERS / sec;
    return NULL;
}

int main() {
    shared_buf = (char*)malloc(SIZE);

    // 나쁜 패턴: main 스레드(소켓0)가 전체 초기화
    // → 모든 페이지가 소켓0 DRAM에 할당
    memset(shared_buf, 0xAB, SIZE);
    printf("[BAD]  main 스레드(소켓0)가 먼저 초기화\n");

    // 워커는 소켓1 코어에서 실행 (→ 원격 접근 발생)
    pthread_t threads[WORKERS];
    worker_arg_t args[WORKERS];

    // cpu 16-19: 소켓1 코어 (시스템에 따라 번호 조정 필요)
    for (int i = 0; i < WORKERS; i++) {
        args[i].worker_id = i;
        args[i].cpu_id = 16 + i;  // 소켓1 코어
    }

    for (int i = 0; i < WORKERS; i++)
        pthread_create(&threads[i], NULL, bad_worker, &args[i]);
    for (int i = 0; i < WORKERS; i++)
        pthread_join(threads[i], NULL);

    double total = 0;
    for (int i = 0; i < WORKERS; i++) total += args[i].result_gbps;
    printf("총 처리량: %.2f GB/s (소켓1 코어가 소켓0 메모리 접근)\n", total);

    // numastat으로 원격 접근 확인:
    // numastat -p $(pgrep first_touch_trap)

    free(shared_buf);
    return 0;
}
// gcc -O2 -pthread -o first_touch_trap first_touch_trap.c
// 실행 후 numastat으로 numa_miss 값 확인
```

### 실험 4: perf stat으로 NUMA 원격 접근 이벤트 측정

```bash
# NUMA 원격 접근 하드웨어 이벤트 측정 (Intel)
perf stat -e \
  cycles,instructions,\
  offcore_response.demand_data_rd.l3_miss.remote_hitm,\
  offcore_response.demand_data_rd.l3_miss.remote_hit_forward,\
  mem_load_l3_miss_retired.remote_dram,\
  mem_load_l3_miss_retired.local_dram \
  numactl --cpunodebind=0 --membind=1 ./numa_bw

# 핵심 지표:
# mem_load_l3_miss_retired.remote_dram  ← 원격 DRAM에서 가져온 횟수
# mem_load_l3_miss_retired.local_dram   ← 로컬 DRAM에서 가져온 횟수
# remote_dram / (remote_dram + local_dram) = 원격 접근 비율

# AMD 시스템:
perf stat -e \
  cycles,instructions,\
  ls_not_halted_cyc,\
  l3_cache_misses_all \
  numactl --cpunodebind=0 --membind=1 ./numa_bw

# /proc/vmstat로 NUMA 접근 카운터 확인
grep numa /proc/vmstat
# numa_hit        → 로컬 노드에 제대로 할당된 페이지
# numa_miss       → 원하는 노드에 할당 실패, 다른 노드에서 할당
# numa_foreign    → 다른 노드가 요청했으나 현재 노드에 할당됨
# numa_interleave → 인터리브 정책으로 할당
# numa_local      → 로컬 노드에서 접근된 페이지
# numa_other      → 원격 노드에서 접근된 페이지 ← 이 값이 크면 경고
```

---

## 📊 성능 비교

```
NUMA 로컬 vs 원격 접근 성능 비교 (Intel Xeon 2소켓):

순차 읽기 처리량 (512MB 버퍼):
  접근 패턴                          처리량(GB/s)   지연(ns)   상대 성능
  ────────────────────────────────────────────────────────────────────
  로컬 DRAM (cpubind=0, membind=0)   ~45 GB/s       ~80 ns     1.0x
  원격 DRAM (cpubind=0, membind=1)   ~25 GB/s       ~150 ns    0.56x
  인터리브 (interleave=all)           ~35 GB/s       ~110 ns    0.78x
  ────────────────────────────────────────────────────────────────────

4소켓 시스템 (2홉 원격):
  로컬 DRAM:   ~45 GB/s   (거리 10)
  1홉 원격:    ~25 GB/s   (거리 21)
  2홉 원격:    ~15 GB/s   (거리 31)  ← 3배 저하!

랜덤 접근 (작은 청크, 지연 측정):
  로컬 DRAM:   ~80 ns
  원격 DRAM:   ~150 ns  (1.88배)
  2홉 원격:    ~250 ns  (3.1배)

first-touch 잘못된 패턴 vs 올바른 패턴 (4스레드, 소켓1 코어):
  main 스레드가 초기화 (소켓0 메모리):  ~18 GB/s
  각 워커 스레드가 초기화 (로컬 메모리): ~42 GB/s  (2.3배 향상)

numastat numa_other (원격 접근 페이지 수):
  NUMA 무시 JVM:  numa_other ~80% of total
  numactl 적용:   numa_other ~5% of total
```

---

## ⚖️ 트레이드오프

```
NUMA 정책 선택의 트레이드오프:

--membind=N (특정 노드 고정):
  ✅ 로컬 접근 보장 → 최저 지연
  ✅ 예측 가능한 성능
  ❌ 해당 노드 메모리 고갈 시 OOM 발생 (다른 노드 여유 있어도)
  ❌ 단일 노드 대역폭에 제한됨

--interleave=all (라운드로빈 분산):
  ✅ 전체 NUMA 메모리 대역폭 활용 가능
  ✅ 메모리 고갈 위험 분산
  ❌ 모든 접근이 평균 50% 원격 → 중간 정도의 지연
  ✅ 접근 패턴이 랜덤하고 넓은 경우 효과적
  → 데이터베이스의 대형 버퍼 풀 등에 적합

--localalloc (기본, first-touch):
  ✅ 코드 수정 없음
  ✅ 접근 패턴에 따라 자연스럽게 로컬 배치
  ❌ 잘못된 first-touch → 영구적인 원격 접근 문제
  ❌ 스레드 마이그레이션 후 메모리는 원래 노드에 남음

--preferred=N (선호 노드, 부족 시 다른 노드):
  ✅ OOM 위험 없음
  ✅ 가능한 경우 로컬 접근
  ❌ 메모리 부족 시 예측 불가한 원격 접근 증가

인터리브가 오히려 좋은 경우:
  대용량 스트리밍 워크로드 (단일 노드 대역폭 포화)
  NUMA 노드 수 × 노드 대역폭 > 단일 노드 대역폭
  → 4소켓에서 interleave로 이론상 4배 대역폭

주의: 마이그레이션 후 데이터 이동
  프로세스가 다른 코어로 이동해도 메모리 페이지는 원래 노드에 남음
  → sched_setaffinity로 코어를 바꿀 때 NUMA 정책도 재검토 필요
  → numa_migrate_pages()로 페이지 이동 가능 (비용 발생)
```

---

## 📌 핵심 정리

```
NUMA 원격 메모리 접근 핵심:

왜 느린가:
  QPI/UPI/Infinity Fabric 인터커넥트 경유
  로컬 DRAM: ~80 ns
  원격 DRAM: ~150~250 ns (1.5~3배 느림, 홉 수에 따라)
  대역폭도 절반: 인터커넥트 최대 대역폭이 로컬 DRAM 대역폭의 절반

확인 명령어:
  numactl --hardware     → 노드 수, 거리 행렬, 메모리 크기
  numactl --show         → 현재 프로세스의 NUMA 정책
  numastat               → 노드별 할당/접근 통계 (numa_miss 주목)
  cat /proc/<PID>/numa_maps → 페이지별 NUMA 위치

측정 명령어:
  numactl --cpunodebind=0 --membind=0 ./a.out  → 로컬 기준선
  numactl --cpunodebind=0 --membind=1 ./a.out  → 원격 비교
  perf stat -e mem_load_l3_miss_retired.remote_dram  → HW 카운터

first-touch 함정:
  main 스레드가 초기화하면 소켓0 메모리
  워커 스레드가 소켓1 코어에서 실행 → 원격 접근
  해결: 워커 스레드가 자신의 영역을 직접 초기화
       또는 numactl --localalloc / --interleave

JVM 대응:
  -XX:+UseNUMA : G1/Parallel GC에서 NUMA 인식 힙
  numactl --interleave=all java ... : 힙 인터리브 배치

다음 단계:
  코어 수 증가의 수익 체감 → Ch6-03 스케일링의 벽
  코어 고정으로 NUMA 효과 유지 → Ch6-05 Thread Affinity
```

---

## 🤔 생각해볼 문제

**Q1.** `numactl --hardware`에서 node distances가 `10, 21, 31, 41`인 4소켓 시스템이 있다. 소켓0의 코어에서 소켓3의 메모리에 접근할 때 예상되는 지연 배율은? 또한 이 시스템에서 메모리 인터리브(`--interleave=all`)를 사용할 때 평균 지연 배율을 계산하라.

<details>
<summary>해설 보기</summary>

**소켓0 → 소켓3 접근 지연:**
거리 41은 로컬(10)의 4.1배를 의미합니다. 즉, 로컬 DRAM이 80ns라면 원격은 80 × (41/10) = **328ns**가 됩니다. 3홉을 거치는 셈입니다(소켓0 → 소켓1 → 소켓2 → 소켓3).

**인터리브 평균 지연 계산:**
4소켓에 균등하게 분산되면 각 노드로의 접근 확률이 25%씩:
- 소켓0(로컬): 25% × 10 = 2.5
- 소켓1(1홉): 25% × 21 = 5.25
- 소켓2(2홉): 25% × 31 = 7.75
- 소켓3(3홉): 25% × 41 = 10.25
- 평균 거리: 2.5 + 5.25 + 7.75 + 10.25 = **25.75**

로컬(10) 대비 **2.575배** 지연이 예상됩니다. 다만 인터리브의 이점은 4소켓의 총 대역폭을 활용할 수 있다는 점입니다. 단일 노드 대역폭이 100 GB/s라면 인터리브 시 ~300~350 GB/s가 이론상 가능합니다(인터커넥트 대역폭 제약에 걸리기 전까지).

**결론:** 레이턴시가 중요한 워크로드(캐시 미스 지연 민감)에는 `--membind=0`이 유리하고, 처리량이 중요한 대용량 스트리밍 워크로드에는 `--interleave=all`이 유리합니다.

</details>

---

**Q2.** Java Spring Boot 애플리케이션이 2소켓 서버에서 예상보다 느리게 실행된다. `numastat -p <pid>`를 보니 `other_node` 값이 `local_node`의 3배다. 어떤 조치를 취할 수 있는가? 각 조치의 효과와 한계를 설명하라.

<details>
<summary>해설 보기</summary>

`other_node`가 `local_node`의 3배라는 것은 JVM의 힙/메타스페이스 접근의 75%가 원격 NUMA 노드에서 이루어진다는 의미입니다. 전형적인 first-touch 함정입니다.

**조치 1: `-XX:+UseNUMA` JVM 플래그**
G1GC, Parallel GC에서 NUMA 인식 힙을 활성화합니다. JVM이 힙을 NUMA 노드별로 분리하여 각 GC 스레드가 자신의 노드 메모리에서 작동합니다.
- 효과: `other_node` 비율 크게 감소, 특히 Parallel GC에서 효과적
- 한계: ZGC는 아직 NUMA 인식이 제한적, 힙 전체가 아닌 Young Gen 중심

**조치 2: `numactl --interleave=all java ...`**
힙 페이지를 두 소켓에 균등 분산합니다.
- 효과: `other_node` 비율 ~50%로 고정되지만 단일 소켓 대역폭 한계를 넘음
- 한계: 개별 접근 지연은 여전히 평균 1.5배, 지연 민감 경로에는 불리

**조치 3: `numactl --cpunodebind=0 --membind=0 java ...`**
JVM 전체를 소켓0에 고정합니다.
- 효과: 모든 접근이 로컬 → 최저 지연, `other_node` 거의 0
- 한계: 소켓0의 코어(16개)와 메모리(64GB)만 사용, 나머지 절반 낭비

**권장 방법:** 소켓당 JVM 인스턴스를 하나씩 실행하고 로드 밸런서로 분산합니다(`numactl --cpunodebind=0 --membind=0 java`와 `numactl --cpunodebind=1 --membind=1 java`). 각 인스턴스가 로컬 NUMA 노드 내에서만 동작하므로 원격 접근이 거의 없습니다.

</details>

---

**Q3.** first-touch 정책을 활용하여 OpenMP 병렬 코드에서 배열을 올바르게 초기화하는 방법을 설명하라. 잘못된 패턴과 올바른 패턴의 코드를 제시하고, NUMA 시스템에서 각각의 동작 차이를 설명하라.

<details>
<summary>해설 보기</summary>

**잘못된 패턴 (NUMA 함정):**
```c
double* arr = malloc(N * sizeof(double));
// main 스레드가 단독으로 초기화 → 단일 NUMA 노드에 모든 페이지 할당
memset(arr, 0, N * sizeof(double));

// 병렬 계산: 각 스레드가 다른 NUMA 노드 코어에서 실행될 수 있음
#pragma omp parallel for
for (int i = 0; i < N; i++)
    arr[i] = compute(i);  // 많은 스레드가 단일 노드 메모리 원격 접근
```

**올바른 패턴 (first-touch 활용):**
```c
double* arr = malloc(N * sizeof(double));
// 병렬 초기화: 각 스레드가 자신의 청크를 먼저 터치
// → 각 스레드가 실행 중인 NUMA 노드의 메모리에 페이지 할당
#pragma omp parallel for schedule(static)
for (int i = 0; i < N; i++)
    arr[i] = 0.0;  // first-touch → NUMA 로컬 페이지 할당

// 이후 동일한 청크 분할로 계산
#pragma omp parallel for schedule(static)
for (int i = 0; i < N; i++)
    arr[i] = compute(i);  // 이제 로컬 NUMA 메모리 접근
```

**동작 차이:**
- 잘못된 패턴: `arr[0]~arr[N-1]`이 모두 main 스레드의 소켓(노드0)에 배치. 소켓1 코어의 스레드들은 모두 원격 접근.
- 올바른 패턴: `schedule(static)`으로 같은 청크 분할이 보장되어 초기화와 계산이 같은 코어(같은 NUMA 노드)에서 실행됨. 각 스레드가 자신이 쓸 페이지를 먼저 초기화하므로 계산 시 로컬 접근.

**핵심:** `schedule(static)`을 두 병렬 루프에 모두 사용하여 동일한 스레드-데이터 매핑을 보장하는 것이 중요합니다. `schedule(dynamic)`을 사용하면 초기화와 계산이 다른 스레드/NUMA 노드에서 실행될 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: 멀티코어 토폴로지](./01-multicore-topology.md)** | **[홈으로 🏠](../README.md)** | **[다음: 스케일링의 벽 ➡️](./03-scaling-wall.md)**

</div>
