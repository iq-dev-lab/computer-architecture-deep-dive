# 메모리 계층 — 레지스터에서 디스크까지의 지연

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- L1/L2/L3 캐시와 DRAM의 지연(latency)은 사이클 단위로 각각 얼마인가?
- "캐시 미스 한 번 = 명령 수십 개 실행 시간"이 구체적으로 무엇을 의미하는가?
- 레지스터, L1, DRAM이 각각 다른 속도를 가지는 물리적·경제적 이유는 무엇인가?
- lat_mem_rd와 Intel MLC로 내 머신의 실제 메모리 지연을 어떻게 측정하는가?
- 메모리 계층이 깊어질수록 용량은 커지고 속도는 느려지는 이유는 무엇인가?
- 코드의 알고리즘 복잡도가 같아도 메모리 접근 패턴에 따라 성능이 10배 차이나는 이유는?

---

## 🔍 왜 이 개념이 중요한가

### "같은 O(N) 코드인데 왜 한쪽이 10배 느리지?"

```
흔한 오해:
  알고리즘 복잡도가 같으면 성능도 비슷하다

실제:
  for (int i = 0; i < N; i++) sum += a[i];          // 빠름
  for (int i = 0; i < N; i++) sum += a[rand() % N]; // 느림

  둘 다 O(N), 연산 수도 동일
  차이는 오직 메모리 접근 패턴

  왜 차이가 나는가:
    순차 접근 → 캐시라인 64B씩 미리 로드 → L1/L2 히트
    랜덤 접근 → 매번 캐시 미스 → DRAM까지 200 사이클 대기

  DRAM 접근 한 번 동안 CPU가 할 수 있는 일:
    4 GHz CPU + 200 사이클 = 50 나노초
    이 시간에 단순 덧셈: ~200개
    즉, "DRAM 미스 1번 = 덧셈 200번 버림"

메모리 계층을 이해하면:
  왜 캐시 친화적 코드가 필요한지 (Ch2-03 지역성)
  왜 AoS보다 SoA가 빠른지 (Ch2-05 데이터 구조)
  왜 TLB 미스가 비싼지 (Ch2-06 TLB)
  — 모두 이 계층 구조에서 파생된다
```

---

## 😱 잘못된 이해

### Before: "메모리 접근은 다 비슷한 속도다"

```
잘못된 모델:
  CPU → 메모리 → 데이터 반환
  "메모리 접근 = 1~2 나노초"

실제로 놓치는 것:
  메모리 접근에는 계층이 있다
  각 계층의 속도 차이는 수십~수백 배

  코드에서 array[i] 하나를 읽을 때:
  ① 레지스터에 있으면: 0 사이클 (이미 있음)
  ② L1 캐시 히트:     ~4 사이클
  ③ L2 캐시 히트:     ~12 사이클
  ④ L3 캐시 히트:     ~40 사이클
  ⑤ DRAM 미스:        ~200 사이클

  "메모리 접근은 다 같다"고 생각하면
  → 왜 행 우선/열 우선이 다른지 설명 못 함
  → 왜 연결 리스트가 배열보다 느린지 이해 못 함
  → 프로파일러의 "cache-miss 80%"가 뭘 의미하는지 모름

  5 사이클짜리 코드와 200 사이클짜리 코드가
  소스에서는 똑같이 보인다
```

---

## ✨ 올바른 이해

### After: 메모리 계층은 속도-용량-비용의 삼각 트레이드오프

```
메모리 계층 (Skylake/Zen2 기준 대략적 수치):

  계층            용량          지연(사이클)   지연(나노초)   특징
  ─────────────────────────────────────────────────────────────────
  레지스터         수십 개 × 8B   0 cy          ~0 ns         CPU 내부
  L1d 캐시        32KB          ~4 cy          ~1 ns         코어 전용
  L2 캐시         256KB         ~12 cy         ~3 ns         코어 전용
  L3 캐시         8~32MB        ~40 cy         ~10 ns        소켓 공유
  DRAM            8~64GB        ~200 cy        ~50 ns        NUMA 노드
  NVMe SSD        TB 급         ~수만 cy       ~100 µs       스토리지
  HDD             TB 급         ~수십만 cy     ~10 ms        스토리지
  ─────────────────────────────────────────────────────────────────

왜 이런 계층이 존재하는가:
  레지스터/L1은 빠른 SRAM — 비싸고 열이 많이 남, 용량 제한
  DRAM은 느린 DRAM — 싸고 용량 크지만 병렬 접근 불가
  캐시는 그 중간에서 자주 쓰는 데이터를 DRAM 속도가 아닌
  SRAM 속도로 제공하는 버퍼

핵심 숫자를 외워라:
  L1 미스 → L2:    +8 사이클
  L2 미스 → L3:    +28 사이클
  L3 미스 → DRAM:  +160 사이클
  → DRAM 미스 한 번 = L1 히트 50번
```

---

## 🔬 내부 동작 원리

### 1. CPU가 메모리를 읽는 과정 — 계층을 타고 내려가기

```
mov eax, [rdi]  ← 이 한 줄에서 일어나는 일

Step 1: L1d 캐시 조회 (1 사이클 이내)
  ┌─────────────────────────────────────┐
  │  L1d (32KB, 8-way set-associative)  │
  │  주소 → 태그/인덱스/오프셋 분해        │
  │  해당 캐시라인(64B) 존재?             │
  │    ✅ 히트 → 4 사이클에 데이터 반환    │
  │    ❌ 미스 → L2로 요청 전달            │
  └─────────────────────────────────────┘

Step 2: L2 캐시 조회 (L1 미스 시)
  ┌─────────────────────────────────────┐
  │  L2 (256KB, 4-way)                  │
  │    ✅ 히트 → 12 사이클               │
  │    ❌ 미스 → L3로 요청 전달            │
  └─────────────────────────────────────┘

Step 3: L3 캐시 조회 (L2 미스 시)
  ┌─────────────────────────────────────┐
  │  L3 (8~32MB, 16-way)               │
  │  소켓 내 모든 코어가 공유             │
  │    ✅ 히트 → 40 사이클               │
  │    ❌ 미스 → 메모리 컨트롤러로 요청    │
  └─────────────────────────────────────┘

Step 4: DRAM 접근 (L3 미스 시)
  ┌─────────────────────────────────────┐
  │  DRAM (수 GB~수 TB)                 │
  │  메모리 컨트롤러 → DRAM 행(Row) 활성  │
  │  → 열(Column) 읽기 → 데이터 전송     │
  │  → 200 사이클 후 L3→L2→L1에 채워짐  │
  └─────────────────────────────────────┘

중요: 캐시 미스는 로드를 멈추지 않는다
  비순차 실행(OoO)이 미스 동안 다른 명령 실행
  하지만 의존하는 명령은 데이터가 올 때까지 대기
  → 200 사이클 동안 CPU가 놀지는 않지만 해당 값에
     의존하는 모든 명령은 스톨(stall)
```

### 2. 사이클 비용이 만드는 실제 차이

```c
// 예제: 같은 합산, 다른 접근 패턴
// gcc -O2 -o sum_test sum_test.c

#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N (1 << 24)  // 16M 정수 = 64MB

int a[N];

long long seq_sum() {
    long long s = 0;
    for (int i = 0; i < N; i++)
        s += a[i];  // 순차 접근: L1/L2 히트율 높음
    return s;
}

long long rand_sum(int* idx) {
    long long s = 0;
    for (int i = 0; i < N; i++)
        s += a[idx[i]];  // 랜덤 접근: 매번 캐시 미스
    return s;
}

int main() {
    // 초기화
    for (int i = 0; i < N; i++) a[i] = i;
    int* idx = malloc(N * sizeof(int));
    for (int i = 0; i < N; i++) idx[i] = rand() % N;

    struct timespec t0, t1;

    clock_gettime(CLOCK_MONOTONIC, &t0);
    volatile long long s1 = seq_sum();
    clock_gettime(CLOCK_MONOTONIC, &t1);
    printf("순차: %ld ms\n",
        (t1.tv_sec - t0.tv_sec) * 1000 +
        (t1.tv_nsec - t0.tv_nsec) / 1000000);

    clock_gettime(CLOCK_MONOTONIC, &t0);
    volatile long long s2 = rand_sum(idx);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    printf("랜덤: %ld ms\n",
        (t1.tv_sec - t0.tv_sec) * 1000 +
        (t1.tv_nsec - t0.tv_nsec) / 1000000);

    free(idx);
    return 0;
}

// 예상 결과 (modern x86-64, 16MB 배열):
//   순차:  ~15 ms  (L3/DRAM 순차 로드, prefetch 효과)
//   랜덤: ~300 ms  (대부분 DRAM 미스, ~20배 차이)
```

### 3. 계층 깊이가 비용 곡선을 만드는 이유

```
SRAM vs DRAM의 물리적 차이:

SRAM (L1/L2/L3 캐시):
  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
  │T1│ │T2│ │T3│ │T4│ │T5│ │T6│  ← 1 bit = 6 트랜지스터
  └──┘ └──┘ └──┘ └──┘ └──┘ └──┘
  • 빠름: 1~4 사이클에 접근
  • 비쌈: bit당 ~6 트랜지스터
  • 전력 소비 높음 (누설 전류)
  • 칩 면적 대비 용량 작음 → GB 수준 불가

DRAM (메인 메모리):
  ┌──┐
  │T1│  ← 1 bit = 1 트랜지스터 + 1 커패시터
  └──┘
  • 느림: ~200 사이클 (행 활성화, 충전/방전)
  • 쌈: bit당 ~1 트랜지스터
  • 용량: 수십~수백 GB 가능
  • 주기적 리프레시 필요 (데이터 휘발성)

경제적 결론:
  빠른 SRAM으로 GB 수준 캐시를 만들 수 없음
  → 작은 SRAM + 큰 DRAM의 계층 구조가 최선
  → 자주 쓰는 데이터를 SRAM에 올려두는 것이 핵심

계층의 용량-지연 곡선:
    지연(사이클)
    200|                              * DRAM
       |
    40 |              * L3
       |
    12 |      * L2
     4 |  * L1
     0 |─────────────────────────→ 용량
          32KB 256KB 8MB  64GB
```

### 4. 비순차 실행과 메모리 병렬성

```
메모리 접근이 스톨을 피하는 방식:

// 두 독립 load: CPU가 둘을 동시에 발행 가능
int x = a[i];   // Miss → 200 사이클 대기 중...
int y = b[j];   // 독립적이므로 즉시 발행 → 병렬 진행
int z = x + y;  // x와 y가 모두 도착해야 실행 가능

메모리 레벨 병렬성(MLP: Memory Level Parallelism):
  L3 미스가 여러 개 동시에 진행 중일 때
  총 지연 = max(각 미스 지연) ← 병렬 겹침
  
  순차 의존:  200 + 200 + 200 = 600 사이클 (최악)
  완전 독립: max(200, 200, 200) = 200 사이클 (이상적)

prefetch (하드웨어 / 소프트웨어):
  // 컴파일러/CPU가 순차 패턴을 감지해 미리 로드
  // __builtin_prefetch(&a[i+16], 0, 1);
  → 접근 전에 미리 캐시에 올려둠 → 미스 비용 숨김
  → 랜덤 접근에는 효과 없음 (패턴 예측 불가)
```

---

## 💻 실전 실험

### 실험 1: perf stat으로 캐시 미스율 측정

```bash
# 위 C 코드를 컴파일
gcc -O2 -o sum_test sum_test.c

# 순차 접근 측정
perf stat -e cycles,instructions,L1-dcache-loads,L1-dcache-load-misses,\
LLC-loads,LLC-load-misses ./sum_test_seq

# 예상 출력 (순차):
#  Performance counter stats:
#     500,000,000   cycles
#     450,000,000   instructions          #  0.90  insn per cycle
#  16,777,216       L1-dcache-loads
#      52,428       L1-dcache-load-misses  #  0.31% of all L1 cache accesses
#      52,428       LLC-loads
#       3,000       LLC-load-misses       #  5.72% of all LLC cache accesses

# 랜덤 접근 측정
perf stat -e cycles,instructions,L1-dcache-loads,L1-dcache-load-misses,\
LLC-loads,LLC-load-misses ./sum_test_rand

# 예상 출력 (랜덤):
#   9,000,000,000   cycles
#     450,000,000   instructions          #  0.05  insn per cycle  ← IPC 급락!
#  16,777,216       L1-dcache-loads
#  15,000,000       L1-dcache-load-misses  # 89.4% ← 대부분 미스!
#  14,800,000       LLC-loads
#  14,750,000       LLC-load-misses       # 99.6% ← DRAM 직행
```

### 실험 2: lat_mem_rd로 메모리 계층 지연 직접 측정

```bash
# lat_mem_rd 설치 (lmbench 패키지)
sudo apt-get install lmbench  # Ubuntu/Debian
# 또는
sudo yum install lmbench      # CentOS/RHEL

# 다양한 워킹셋 크기에서 랜덤 읽기 지연 측정
# 워킹셋 < L1 → L1 히트 지연
# 워킹셋 < L2 → L2 히트 지연
# 워킹셋 < L3 → L3 히트 지연
# 워킹셋 > L3 → DRAM 지연

lat_mem_rd -t -s 128m 64

# 예상 출력 (Intel Core i7 기준):
# "stride=64"
# 0.00049   1.4   ← 512B: L1 (1.4 ns ≈ 4 cy @ 3GHz)
# 0.00098   1.4
# 0.002     1.5
# ...
# 0.032     3.8   ← 32KB: L1/L2 경계
# 0.064     4.1   ← 64KB: L2
# 0.256     5.2   ← 256KB: L2
# 0.512     11.0  ← 512KB: L2/L3 경계
# 1.0       13.0  ← 1MB: L3
# 8.0       14.0  ← 8MB: L3
# 16.0      50.0  ← 16MB: L3/DRAM 경계 ← 절벽!
# 32.0      50.2  ← 32MB: DRAM
# 64.0      50.5  ← 64MB: DRAM
# 128.0     51.0  ← 128MB: DRAM

# 그래프로 볼 때 나타나는 "계단":
#   4ns → 3ns → 11ns → 50ns 순으로 올라가는 계단
#   각 계단 = 캐시 경계
```

### 실험 2b: Intel MLC (Memory Latency Checker)로 정밀 측정

```bash
# Intel MLC 다운로드 (Intel 공식 배포)
# https://www.intel.com/content/www/us/en/developer/articles/tool/intelr-memory-latency-checker.html

./mlc --latency_matrix
# 출력 예시 (단일 소켓):
# Numa node
# Numa node        0
#        0        79.2  ← DRAM 지연 (ns)

./mlc --peak_injection_bandwidth
# Peak Injection Memory Bandwidths
# ALL Reads:         47588.3 MB/sec
# 3:1 Reads-Writes:  36877.1 MB/sec
# 2:1 Reads-Writes:  33617.5 MB/sec

# 캐시 레벨별 지연 측정
./mlc --loaded_latency
# Inject  Latency
# Delay   (ns)
# 00000   81.89  ← 무부하 DRAM 지연
# 00002   81.93
# ...
# 02000   100.5  ← 부하 증가 시 큐잉 지연 추가
```

### 실험 3: 워킹셋 크기별 성능 절벽 확인

```c
// working_set_test.c
// 워킹셋을 점점 키우며 처리량 측정
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#define ITER 100000000ULL

double measure_bw(size_t size_bytes) {
    int* buf = (int*)aligned_alloc(64, size_bytes);
    memset(buf, 0, size_bytes);
    size_t n = size_bytes / sizeof(int);

    struct timespec t0, t1;
    volatile long long sum = 0;

    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (size_t iter = 0; iter < ITER; iter++) {
        sum += buf[iter % n];  // 순차 패턴
    }
    clock_gettime(CLOCK_MONOTONIC, &t1);

    double elapsed = (t1.tv_sec - t0.tv_sec) +
                     (t1.tv_nsec - t0.tv_nsec) * 1e-9;
    double bw = (double)ITER * sizeof(int) / elapsed / 1e9; // GB/s
    free(buf);
    return bw;
}

int main() {
    size_t sizes[] = {
        4*1024, 8*1024, 16*1024, 32*1024,   // L1 이하~L1 경계
        64*1024, 128*1024, 256*1024,          // L2
        512*1024, 1024*1024, 4096*1024,       // L3
        16*1024*1024, 64*1024*1024            // DRAM
    };
    for (int i = 0; i < 12; i++) {
        printf("%6zuKB: %.2f GB/s\n",
               sizes[i]/1024, measure_bw(sizes[i]));
    }
    return 0;
}

// gcc -O2 -o ws_test working_set_test.c && ./ws_test
// 예상 출력:
//      4KB: 180.5 GB/s  ← L1 (레지스터에 루프)
//      8KB: 175.3 GB/s
//     16KB: 168.0 GB/s
//     32KB: 120.0 GB/s  ← L1/L2 경계 (32KB L1)
//     64KB:  85.0 GB/s  ← L2
//    128KB:  80.0 GB/s
//    256KB:  75.0 GB/s  ← L2 경계 (256KB L2)
//    512KB:  45.0 GB/s  ← L3
//   1024KB:  42.0 GB/s
//   4096KB:  38.0 GB/s
//  16384KB:  18.0 GB/s  ← L3/DRAM 경계 절벽!
//  65536KB:  16.5 GB/s  ← DRAM (약 10배 느림)
```

---

## 📊 성능 비교

```
메모리 계층별 지연 및 대역폭 (Intel Skylake, 3.0 GHz 기준 근사치):

  계층          지연(사이클)  지연(ns)   대역폭(GB/s)  용량
  ──────────────────────────────────────────────────────────
  레지스터       0             0          ~2000         ~256B
  L1d 캐시      4             1.3        ~200          32KB
  L2 캐시       12            4.0        ~100          256KB
  L3 캐시       40            13.3       ~50           8MB
  DRAM          200           66.7       ~40           수십GB
  NVMe SSD      ~20,000       ~50µs      ~3            수TB
  ──────────────────────────────────────────────────────────

미스 비용 배율:
  L1 미스 → L2:    3배 느려짐
  L2 미스 → L3:    3.3배 더 느려짐
  L3 미스 → DRAM:  5배 더 느려짐
  누적: L1 대비 DRAM은 50배 느림

실험 결과 (순차 vs 랜덤 64MB 배열, 동일 N번 접근):
  순차 접근:  ~15ms  (L3 적중, prefetch 효과)
  랜덤 접근: ~300ms  (대부분 DRAM 미스)
  차이: ~20배

IPC로 본 영향:
  L1 히트 집약 코드:   IPC ~3.5 (거의 이론 최대)
  DRAM 미스 집약 코드: IPC ~0.05 (스톨로 CPU 낭비)
```

---

## ⚖️ 트레이드오프

```
메모리 계층 구조의 설계 트레이드오프:

캐시 크기 증가 시:
  ✅ 캐시 히트율 향상 → 성능 향상
  ✅ 더 큰 워킹셋을 빠른 속도로 처리
  ❌ 칩 면적 증가 → 단가 상승
  ❌ 접근 지연 증가 (큰 캐시는 탐색에 더 많은 사이클 필요)
  ❌ 캐시 일관성 프로토콜 복잡도 증가 (Ch3-01 MESI)

캐시 계층 수 증가 시:
  ✅ 계층 간 속도 차이를 완충
  ✅ 코어 전용(L1/L2) + 공유(L3)로 역할 분리
  ❌ 미스 발생 시 탐색해야 하는 계층 수 증가
  ❌ 캐시 관리 회로 복잡도 증가

프로그래머가 할 수 있는 일:
  ✅ 캐시에 잘 맞는 워킹셋 설계 (데이터 크기 줄이기)
  ✅ 공간/시간 지역성 향상 (Ch2-03)
  ✅ 랜덤 접근 패턴 제거 → 순차/스트라이드 패턴으로
  ✅ 핫/콜드 데이터 분리 (Ch2-05 AoS/SoA)
  ❌ 캐시 하드웨어 자체는 변경 불가
  ❌ DRAM 지연 자체는 물리적 한계

측정 없이는 추측에 불과:
  "이 코드가 캐시 친화적이다" → perf stat으로 확인 필수
  LLC-load-misses / L1-dcache-load-misses 비율로 판단
```

---

## 📌 핵심 정리

```
메모리 계층 핵심:

지연 수치 (외워라):
  L1: ~4 사이클   (~1 ns)
  L2: ~12 사이클  (~4 ns)
  L3: ~40 사이클  (~13 ns)
  DRAM: ~200 사이클 (~67 ns)
  → L3 미스 한 번 = L1 히트 50번

왜 계층이 있는가:
  SRAM(빠름/비쌈/작음) + DRAM(느림/쌈/큼)의 타협
  자주 쓰는 데이터를 SRAM에 올려두는 전략
  경제적/물리적 한계로 SRAM만으로 GB 수준 캐시 불가

코드 성능에 주는 영향:
  같은 O(N) 알고리즘이라도 접근 패턴에 따라 20배 차이
  IPC: DRAM 미스 집약 코드는 0.05 수준으로 떨어짐
  perf stat의 LLC-load-misses가 핵심 지표

측정 도구:
  perf stat -e L1-dcache-load-misses,LLC-load-misses ./a.out
  lat_mem_rd -t -s 128m 64  ← 계층별 지연 계단 시각화
  Intel MLC                 ← DRAM 대역폭/지연 정밀 측정

다음 단계:
  이 계층이 어떻게 동작하는지 (Ch2-02 캐시 내부 구조)
  접근 패턴이 히트율을 결정하는 이유 (Ch2-03 지역성)
  미스를 줄이는 데이터 구조 설계 (Ch2-05 AoS/SoA)
```

---

## 🤔 생각해볼 문제

**Q1.** 4 GHz CPU에서 L3 캐시 미스가 초당 1,000만 번 발생하는 코드가 있다. 이 미스들로 인한 순수 대기 사이클은 얼마이며, 이는 전체 사이클의 몇 %를 차지하는가? (L3 미스 지연 = 200 사이클, 초당 총 사이클 = 4 × 10⁹)

<details>
<summary>해설 보기</summary>

**계산 과정**:

미스로 인한 대기 사이클 (상한):
- 미스 1번당 대기: 200 사이클
- 초당 미스 횟수: 10,000,000번
- 순수 대기 사이클 (직렬 가정): 200 × 10,000,000 = 20억 사이클/초

전체 사이클 대비 비율:
- 초당 총 사이클: 4,000,000,000
- 대기 비율: 2,000,000,000 / 4,000,000,000 = 50%

**실제는 다르다**: 비순차 실행(OoO)이 미스 동안 독립적인 다른 명령을 실행하므로, 실제 "낭비"된 사이클은 이보다 적습니다. 또한 MLP(메모리 레벨 병렬성)로 여러 미스가 병렬 진행될 수 있습니다.

하지만 미스에 의존하는 연산이 많다면 50%의 사이클이 실질적으로 낭비됩니다. `perf stat`에서 IPC가 0.5 미만으로 떨어지는 코드가 이 상황입니다.

</details>

---

**Q2.** 같은 알고리즘을 구현한 두 버전이 있다. 버전 A는 L3 미스율이 높지만 연산이 단순하고, 버전 B는 연산이 복잡하지만 캐시 미스가 거의 없다. perf stat 결과 A는 IPC 0.08, B는 IPC 1.2이다. 어느 쪽이 더 빠른가?

<details>
<summary>해설 보기</summary>

**IPC만으로는 판단할 수 없다**. 실제 실행 시간은 **총 사이클 수** = 명령 수 / IPC 로 결정됩니다.

예를 들어:
- 버전 A: 명령 수 = 100M, IPC = 0.08 → 총 사이클 = 1,250M
- 버전 B: 명령 수 = 800M, IPC = 1.2  → 총 사이클 = 667M

이 경우 B가 더 빠릅니다. IPC가 낮아도 명령 수가 적으면 사이클이 적을 수 있습니다.

반대로:
- 버전 A: 명령 수 = 50M, IPC = 0.08 → 총 사이클 = 625M
- 버전 B: 명령 수 = 2000M, IPC = 1.2 → 총 사이클 = 1,667M

이 경우 A가 더 빠릅니다.

**핵심**: `perf stat`에서 `cycles` 수치를 직접 비교하는 것이 가장 정확합니다. IPC는 병목의 성격을 설명하는 지표이지 절대 성능 지표가 아닙니다.

</details>

---

**Q3.** L3 캐시가 32MB인 머신에서 40MB 배열을 처리하는 코드가 있다. 이 코드를 수정해서 배열 크기는 그대로 유지하면서 캐시 히트율을 올릴 수 있는 방법을 두 가지 제시하라.

<details>
<summary>해설 보기</summary>

**방법 1: 캐시 블로킹(Cache Blocking / Tiling)**

배열 전체를 한 번에 처리하는 대신, L3에 들어가는 크기(예: 24MB)로 나누어 처리합니다.

```c
// Before: 40MB 배열 전체 스캔 → 대부분 DRAM 미스
for (int i = 0; i < N; i++)
    process(a[i]);

// After: 블록 단위 처리 → 각 블록이 L3에 들어감
#define BLOCK (24 * 1024 * 1024 / sizeof(int))
for (int b = 0; b < N; b += BLOCK)
    for (int i = b; i < min(b + BLOCK, N); i++)
        process(a[i]);
```

블록 하나가 L3에 캐싱된 동안 여러 번 재사용 가능한 알고리즘(예: 행렬 연산)에 특히 효과적입니다.

**방법 2: 데이터 크기 줄이기 (Hot/Cold 분리)**

40MB 중 실제로 자주 접근하는 "핫(hot)" 필드만 별도 배열로 분리합니다.

```c
// Before: 큰 구조체 배열 → 캐시에 필요 없는 데이터도 로드
struct Item { int key; char data[80]; }; // 84B × 500K = 40MB
Item items[500000];
for (int i = 0; i < N; i++)
    if (items[i].key > threshold) count++;

// After: 핫 필드만 별도 배열
int keys[500000];  // 4B × 500K = 2MB → L3에 들어감!
char data[500000][80];
for (int i = 0; i < N; i++)
    if (keys[i] > threshold) count++;
```

`key`만 자주 접근하는 경우, 2MB 배열은 L3에 완전히 들어가 히트율이 극적으로 향상됩니다. 이것이 Ch2-05에서 다루는 Hot/Cold 필드 분리 전략의 핵심입니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: ILP의 한계](../execution-and-pipeline/06-ilp-limits.md)** | **[홈으로 🏠](../README.md)** | **[다음: 캐시 동작 원리 ➡️](./02-cache-internals.md)**

</div>
