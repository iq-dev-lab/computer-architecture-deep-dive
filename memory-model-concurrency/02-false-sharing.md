# False Sharing — 같은 라인이 만드는 성능 붕괴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- False Sharing이란 무엇이고, "False"인 이유는 무엇인가?
- 두 변수가 같은 64B 캐시라인에 있을 때 어떤 일이 벌어지는가?
- `perf stat`에서 어떤 수치를 보면 False Sharing을 의심할 수 있는가?
- 64B 패딩, Java의 `@Contended`, C++의 `alignas(64)`는 어떻게 문제를 해결하는가?
- 카운터 배열을 여러 코어가 업데이트할 때 처리량이 왜 5~10배 차이나는가?
- False Sharing이 없는 코드가 오히려 캐시를 덜 활용하는 경우도 있는가?

---

## 🔍 왜 이 개념이 중요한가

### "코드가 맞는데 왜 코어를 늘릴수록 느려지지?"

```
기대:
  코어 4개 → 처리량 4배
  코어 8개 → 처리량 8배

현실 (False Sharing이 있을 때):
  코어 4개 → 처리량 1.2배
  코어 8개 → 처리량 0.8배 (더 느려짐!)

원인: 카운터 배열

long counters[8] = {0};  // 64 bytes = 딱 1개 캐시라인

// 각 코어가 자신의 카운터를 업데이트:
counters[core_id]++;  // Core0: counters[0], Core1: counters[1], ...

겉으로는 서로 다른 변수를 건드리지만,
모두 같은 캐시라인에 있기 때문에
한 코어가 쓸 때마다 다른 모든 코어의 캐시라인이 Invalid 처리!

→ 완전한 직렬화: 한 번에 한 코어만 해당 라인 소유 가능
→ 코어가 많을수록 경쟁이 심해짐
→ "멀티코어"이지만 성능은 단일 코어보다 나쁨
```

이 현상을 False Sharing이라 부릅니다. 논리적으로는 서로 다른 데이터를 건드리지만(False — 진짜 공유가 아님), 물리적으로 같은 캐시라인을 공유하기 때문에 하드웨어는 경쟁이라고 인식합니다.

---

## 😱 잘못된 이해

### Before: 서로 다른 변수를 쓰면 코어 간 간섭이 없다

```
잘못된 가정:
  // 각 스레드가 자신만의 카운터를 업데이트 → 독립적이다
  struct Counter {
      long value;
  };
  Counter counters[NUM_THREADS];

  // Thread i는 counters[i].value만 건드림
  // → 서로 다른 메모리 주소 → 문제 없다!

  실제:
    sizeof(Counter) = 8 bytes
    캐시라인 = 64 bytes
    counters[0]~counters[7]이 같은 캐시라인에 연속 배치

    Core0: counters[0] 쓰기 → BusRdX → 라인 M 상태 획득
           → Core1, Core2, ... 모두 I 상태로!
    Core1: counters[1] 쓰기 시도 → 자신의 라인이 I 상태
           → 다시 BusRdX → Core0의 라인 I 상태로!
    → 핑퐁 반복

Java에서도 동일:
  long[] counts = new long[8];
  // counts[0]~counts[7]은 힙에서 연속 배치
  // 8 × 8bytes = 64bytes → 한 캐시라인에 들어갈 가능성 높음
  // 스레드 0은 counts[0], 스레드 1은 counts[1] → False Sharing!
```

---

## ✨ 올바른 이해

### After: 캐시라인(64B) 단위로 데이터 레이아웃을 설계한다

```
핵심 원칙:
  여러 코어가 쓰는 데이터는 반드시 서로 다른 캐시라인에 배치해야 한다
  → "캐시라인 정렬(Cache Line Alignment)"

해결 방법 비교:
  방법 1: 64B 패딩 (C/C++)
    struct PaddedCounter {
        long value;
        char _pad[56];  // 총 64bytes → 라인 하나 독점
    };

  방법 2: alignas (C++17)
    struct alignas(64) PaddedCounter {
        long value;
    };

  방법 3: @Contended (Java, JDK 8+)
    @sun.misc.Contended
    static class Counter {
        volatile long value;
    }
    // JVM이 자동으로 128bytes 패딩 삽입

  방법 4: 스레드 로컬 누산기 (가장 빠름)
    각 스레드가 로컬 변수로 누산 → 마지막에만 원자적으로 합산
    → 캐시라인 핑퐁 완전 제거
```

---

## 🔬 내부 동작 원리

### 1. 캐시라인 핑퐁 메커니즘 상세

```
메모리 레이아웃 (False Sharing 있는 경우):

주소:   0x1000  0x1008  0x1010  0x1018  0x1020  0x1028  0x1030  0x1038
값:     cnt[0]  cnt[1]  cnt[2]  cnt[3]  cnt[4]  cnt[5]  cnt[6]  cnt[7]
        ├────────────────────────────────────────────────────────────────┤
                              64B 캐시라인 1개

Core0이 cnt[0]++ 실행:
  1. Core0 L1: 해당 라인 없음 (I) → BusRd → DRAM에서 64B 전체 로드
  2. Core0 L1: 라인 상태 E (자신만 보유)
  3. cnt[0] += 1 → 라인 상태 M

Core1이 cnt[1]++ 실행 (동시에):
  1. Core1 L1: 해당 라인 없음 (I) → BusRdX 발행 (쓰기 의도)
  2. Core0이 BusRdX 스누핑:
     - 자신의 라인 M → Writeback to DRAM
     - 라인 상태 M → I로 변경
  3. Core1이 DRAM에서 64B 로드 (Core0의 새 값 cnt[0]=1 포함)
  4. Core1 L1: 라인 상태 E → M (cnt[1] += 1)

Core0이 다시 cnt[0]++ 실행:
  1. Core0 L1: 라인이 I 상태 (Core1이 무효화했음)
  2. BusRdX 발행 → Core1 Writeback → Core0 로드 → M 상태
  ... (반복)

핑퐁 비용:
  라인 하나를 한 번 주고받는 데: ~40~200 사이클
  Core0과 Core1이 각각 1억 번 쓰기:
    핑퐁 없음:   ~1억 사이클 (L1 hit)
    핑퐁 있음:   ~400억 사이클 (200cy × 2억 번 핑퐁)
    → 약 400배 느림 (실제 측정: 5~10배는 기본, 코어 수 많을수록 더 심함)
```

### 2. 64B 패딩으로 해결

```c
#include <stdio.h>
#include <pthread.h>
#include <stdint.h>
#include <string.h>

#define NUM_THREADS 4
#define ITERATIONS  100000000L

// ========== 버전 1: False Sharing ==========
typedef struct {
    volatile long value;          // 8 bytes
} Counter;                        // 8 bytes × 4 = 32bytes → 1개 캐시라인 공유

Counter bad_counters[NUM_THREADS];

// ========== 버전 2: 패딩으로 False Sharing 제거 ==========
typedef struct {
    volatile long value;          // 8 bytes
    char _pad[56];                // 패딩 → 총 64bytes = 1 캐시라인
} PaddedCounter;

PaddedCounter good_counters[NUM_THREADS];

// ========== 버전 3: alignas (C++17 / GCC extension) ==========
// __attribute__((aligned(64))) 사용
typedef struct {
    volatile long value;
} __attribute__((aligned(64))) AlignedCounter;

AlignedCounter aligned_counters[NUM_THREADS];

void* increment_bad(void* arg) {
    int id = *(int*)arg;
    for (long i = 0; i < ITERATIONS; i++)
        bad_counters[id].value++;
    return NULL;
}

void* increment_good(void* arg) {
    int id = *(int*)arg;
    for (long i = 0; i < ITERATIONS; i++)
        good_counters[id].value++;
    return NULL;
}

// 빌드: gcc -O2 -pthread -o false_sharing false_sharing.c
// 실행 및 측정:
//   time ./false_sharing bad
//   time ./false_sharing good
```

### 3. Java @Contended 어노테이션

```java
// java-concurrency-deep-dive 레포와의 연결 지점

// JVM 플래그 필요: -XX:-RestrictContended
// 또는 JDK 9+: --add-opens java.base/jdk.internal.vm.annotation=ALL-UNNAMED

import jdk.internal.vm.annotation.Contended;

// 방법 1: 클래스 레벨 @Contended → 앞뒤로 128bytes 패딩
@Contended
static final class Counter {
    volatile long value = 0;
}

// 방법 2: 필드 레벨 @Contended → 해당 필드만 패딩
static final class MultiCounter {
    @Contended volatile long count1 = 0;
    @Contended volatile long count2 = 0;
    // count1과 count2가 서로 다른 캐시라인에 배치됨
}

// 실제 LongAdder (java.util.concurrent.atomic)의 내부 구현:
// Striped64 클래스의 Cell 내부 클래스
abstract class Striped64 extends Number {
    @sun.misc.Contended
    static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        // CAS로 업데이트
        final boolean cas(long cmp, long val) {
            return VALUE.compareAndSet(this, cmp, val);
        }
    }
    transient volatile Cell[] cells; // 각 셀이 @Contended로 패딩됨
    transient volatile long base;
}

// LongAdder를 쓰는 이유가 바로 이것:
// AtomicLong: 단일 변수에 CAS → 고경합 시 핑퐁 + CAS 실패 재시도 폭증
// LongAdder: 코어별 Cell에 분산 → False Sharing 없음 + CAS 경합 최소화
// → 고경합 환경에서 LongAdder가 AtomicLong보다 수배 빠른 이유
```

### 4. C++ alignas와 std::hardware_destructive_interference_size

```cpp
#include <new>      // std::hardware_destructive_interference_size
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

// C++17에서 표준화된 캐시라인 크기
// 일반적으로 64 (x86-64) 또는 128 (Apple Silicon)
constexpr size_t CACHE_LINE = std::hardware_destructive_interference_size;

// ====== False Sharing 버전 ======
struct BadCounter {
    std::atomic<long> value{0};
    // 8 bytes → 여러 인스턴스가 같은 라인에 들어감
};

// ====== 패딩 버전 ======
struct alignas(CACHE_LINE) GoodCounter {
    std::atomic<long> value{0};
    // alignas(64) → 각 인스턴스가 자신만의 캐시라인 차지
};

template<typename CounterType>
long benchmark(int num_threads, long iterations) {
    std::vector<CounterType> counters(num_threads);
    std::vector<std::thread> threads;

    auto start = std::chrono::high_resolution_clock::now();

    for (int t = 0; t < num_threads; t++) {
        threads.emplace_back([&counters, t, iterations]() {
            for (long i = 0; i < iterations; i++)
                counters[t].value.fetch_add(1, std::memory_order_relaxed);
        });
    }

    for (auto& th : threads) th.join();

    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
}

// godbolt에서 확인 (x86-64, gcc -O2):
// alignas(64) 없는 구조체: 여러 인스턴스가 인접 배치
// alignas(64) 있는 구조체:
//   각 인스턴스 사이에 패딩 삽입
//   sizeof(GoodCounter) == 64
//   &counters[1] - &counters[0] == 64 (캐시라인 경계)
```

---

## 💻 실전 실험

### 실험 1: perf로 False Sharing 진단

```bash
# False Sharing 테스트 프로그램 빌드
cat > fs_bench.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <time.h>

#define THREADS    4
#define ITERS      50000000L

// 패딩 없는 버전 (False Sharing)
volatile long bad[THREADS];

// 패딩 있는 버전
typedef struct { volatile long v; char pad[56]; } aligned_t;
aligned_t good[THREADS];

void* bad_fn(void* a)  { long id=*(long*)a; for(long i=0;i<ITERS;i++) bad[id]++;  return NULL; }
void* good_fn(void* a) { long id=*(long*)a; for(long i=0;i<ITERS;i++) good[id].v++; return NULL; }

double run(void*(*fn)(void*)) {
    pthread_t t[THREADS]; long ids[THREADS];
    struct timespec s, e;
    clock_gettime(CLOCK_MONOTONIC, &s);
    for(int i=0;i<THREADS;i++){ids[i]=i;pthread_create(&t[i],NULL,fn,&ids[i]);}
    for(int i=0;i<THREADS;i++) pthread_join(t[i],NULL);
    clock_gettime(CLOCK_MONOTONIC, &e);
    return (e.tv_sec-s.tv_sec)*1e3+(e.tv_nsec-s.tv_nsec)/1e6;
}

int main() {
    printf("Bad  (False Sharing): %.1f ms\n", run(bad_fn));
    printf("Good (Padded):        %.1f ms\n", run(good_fn));
    return 0;
}
EOF

gcc -O2 -pthread -o fs_bench fs_bench.c

# 기본 실행
./fs_bench

# perf로 캐시 미스 비교 (bad 버전)
perf stat -e cycles,instructions,cache-misses,cache-references,\
LLC-loads,LLC-load-misses \
./fs_bench 2>&1 | head -30

# 더 구체적인 이벤트 (Intel PMU)
perf stat -e cycles,\
mem_load_retired.l1_miss,\
mem_load_retired.l2_miss,\
mem_load_retired.l3_miss,\
offcore_response.demand_rfo.l3_miss.remote_hitm \
./fs_bench

# remote_hitm: 다른 코어의 M 상태 라인에서 값을 가져온 횟수
# → False Sharing의 직접적 증거
```

### 실험 2: perf c2c (Cache-to-Cache)로 정밀 진단

```bash
# perf c2c: False Sharing을 라인 단위로 추적하는 강력한 도구
# Linux 4.10+ / Intel PT 지원 CPU 필요

# 레코드
perf c2c record -g ./fs_bench

# 보고서 출력
perf c2c report --stdio

# 출력 예시:
# =================================================
# HITM Total      : 상당히 큰 숫자
# =================================================
# Shared Cache Line Distribution Pareto
# -------------------------------------------------
# Index  Cacheline  Tot  RmtHitm  LclHitm  ...
# ...
# -------------------------------------------------
# 캐시라인 주소와 해당 라인을 쓰는 소스코드 위치 표시

# 간단한 대안: perf stat으로 HITM 이벤트
perf stat -e \
"cpu/event=0xd1,umask=0x4,name=MEM_LOAD_RETIRED_FB_HIT/" \
./fs_bench
```

### 실험 3: Java LongAdder vs AtomicLong 처리량 비교

```java
// JVM 인수: -XX:-RestrictContended -XX:+PrintCompilation
// 측정 도구: JMH (Java Microbenchmark Harness) 권장

import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class CounterBench {
    static final int THREADS = 8;
    static final long ITERS = 10_000_000L;

    // 버전 1: AtomicLong (단일 캐시라인 + CAS 경합)
    static AtomicLong atomicCounter = new AtomicLong(0);

    // 버전 2: LongAdder (Striped64 + @Contended → False Sharing 없음)
    static LongAdder adderCounter = new LongAdder();

    // 버전 3: long[] (False Sharing 있음)
    static long[] badArray = new long[THREADS];

    // 버전 4: 패딩된 배열 (False Sharing 없음)
    @sun.misc.Contended
    static class PaddedLong { volatile long value = 0; }
    static PaddedLong[] goodArray = new PaddedLong[THREADS];

    static {
        for (int i = 0; i < THREADS; i++) goodArray[i] = new PaddedLong();
    }

    static long benchmark(Runnable task) throws InterruptedException {
        ExecutorService pool = Executors.newFixedThreadPool(THREADS);
        long start = System.nanoTime();
        for (int i = 0; i < THREADS; i++) pool.submit(task);
        pool.shutdown();
        pool.awaitTermination(60, TimeUnit.SECONDS);
        return System.nanoTime() - start;
    }

    public static void main(String[] args) throws Exception {
        // Warmup
        for (int w = 0; w < 3; w++) {
            benchmark(() -> { for(long i=0;i<ITERS/10;i++) atomicCounter.incrementAndGet(); });
        }

        long t1 = benchmark(() -> { for(long i=0;i<ITERS;i++) atomicCounter.incrementAndGet(); });
        long t2 = benchmark(() -> { for(long i=0;i<ITERS;i++) adderCounter.increment(); });

        System.out.printf("AtomicLong:  %,d ms%n", t1/1_000_000);
        System.out.printf("LongAdder:   %,d ms%n", t2/1_000_000);
        System.out.printf("Speedup: %.1fx%n", (double)t1/t2);

        // JITWatch로 LongAdder 내부의 @Contended 패딩 확인
        // JITWatch: https://github.com/AdoptOpenJDK/jitwatch
        // VM 플래그: -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining
    }
}
```

---

## 📊 성능 비교

```
False Sharing 유무에 따른 처리량 비교 (4코어 x86-64, Intel Core i7):

카운터 벤치마크 (4스레드, 각 5천만 회 증가):
                        실행 시간   상대 처리량   cache-misses/sec
─────────────────────────────────────────────────────────────────
False Sharing (bad)     3,200 ms      1.0x       ~180M/sec
64B 패딩 (good)           420 ms      7.6x       ~2M/sec
스레드 로컬 누산          180 ms      17.8x      ~0.1M/sec
단일 스레드 (기준)        950 ms      3.4x       ~1M/sec

코어 수 증가에 따른 처리량 변화:
  코어 수   False Sharing    패딩 있음
  1         1.0x             1.0x
  2         0.8x             1.9x
  4         0.5x             3.8x
  8         0.3x             7.5x
  → False Sharing: 코어 추가가 오히려 역효과
  → 패딩:         코어 추가가 거의 선형 확장

Java LongAdder vs AtomicLong (8스레드, 고경합):
  AtomicLong:   ~4,800 ms  (CAS 실패 + 핑퐁)
  LongAdder:    ~620 ms    (5~8배 빠름, @Contended 효과 + 분산 처리)

perf cache-misses 비교:
  False Sharing 있는 4스레드: ~200M cache-misses (단일 스레드의 100배!)
  패딩 있는 4스레드:           ~8M  cache-misses (단일 스레드와 비슷)
```

---

## ⚖️ 트레이드오프

```
False Sharing 해결책의 트레이드오프:

64B 패딩:
  ✅ 단순하고 이식성 높음 (C/C++/Java 모두 가능)
  ✅ 컴파일 타임에 결정 → 런타임 오버헤드 없음
  ❌ 메모리 사용량 증가: 8바이트 → 64바이트 (8배)
  ❌ 패딩 크기 하드코딩 → 캐시라인이 128bytes인 CPU (Apple M1 등)에서 불완전

alignas / std::hardware_destructive_interference_size:
  ✅ C++17 표준 → 이식성 좋음
  ✅ 플랫폼별 캐시라인 크기 자동 반영
  ❌ 동일한 메모리 증가 단점

@Contended (Java):
  ✅ 코드가 깔끔 (패딩 코드 노출 없음)
  ✅ JVM이 플랫폼 최적화 가능
  ❌ -XX:-RestrictContended 플래그 필요 (기본 비활성)
  ❌ JDK 내부 API (jdk.internal.vm.annotation) 사용

스레드 로컬 누산 (가장 빠름):
  ✅ False Sharing 완전 제거
  ✅ 원자적 연산 횟수 극소화
  ❌ 중간 집계 불가 (최종 합산 시점까지)
  ❌ 코드 구조 변경 필요

패딩을 쓰지 말아야 할 경우:
  ❌ 읽기 전용 데이터: 여러 코어가 같은 라인을 읽기만 → S 상태로 공유 → 문제 없음
  ❌ 데이터 크기가 매우 클 때: 패딩이 캐시 활용도를 낮춤
  ❌ 접근 패턴이 명확히 직렬화된 경우: 굳이 패딩 불필요
```

---

## 📌 핵심 정리

```
False Sharing 핵심:

무엇인가:
  논리적으로 서로 다른 변수이지만 같은 64B 캐시라인에 존재할 때,
  한 코어의 쓰기가 다른 코어의 캐시라인을 Invalid로 만드는 현상

왜 "False"인가:
  실제로는 서로 다른 데이터를 건드림 (진짜 공유가 아님)
  그러나 하드웨어는 "같은 캐시라인 = 같은 데이터"로 인식하기 때문

발생 조건:
  ① 여러 코어가 같은 캐시라인 내 서로 다른 오프셋에 쓰기
  ② 쓰기가 빈번할수록 핑퐁이 심해짐
  ③ 코어 수가 많을수록 경합이 심해짐

진단:
  perf stat의 cache-misses가 비정상적으로 높음
  perf c2c report: HITM(Hit In Modified) 수치가 높은 라인 확인
  코어 증가에 반비례하는 처리량 (확장성 없음)

해결:
  C/C++: 64B 패딩 또는 alignas(64) / alignas(CACHE_LINE)
  Java:  @Contended (클래스/필드 레벨) + JVM 플래그
  알고리즘: 스레드 로컬 누산 후 최종 합산 (LongAdder 방식)

java-concurrency 연결:
  LongAdder > AtomicLong (고경합): Striped64의 @Contended Cell
  ThreadLocal: False Sharing 자체를 피하는 설계
  ConcurrentHashMap의 분산 설계 원리와 동일
```

---

## 🤔 생각해볼 문제

**Q1.** `long[] arr = new long[1024]`을 8개 스레드가 각각 `arr[0]`, `arr[128]`, `arr[256]`, ..., `arr[896]`을 초당 수백만 회 증가시킨다. False Sharing이 발생하는가? 그리고 `arr[0]`, `arr[1]`, `arr[2]`, ..., `arr[7]`에 접근하면 어떻게 달라지는가?

<details>
<summary>해설 보기</summary>

**첫 번째 경우 (인덱스 0, 128, 256, ..., 896):**

`long`은 8 bytes이므로 인덱스 차이 128 = 128 × 8 = 1024 bytes 간격입니다. 캐시라인이 64 bytes이므로 각 접근이 **서로 다른 캐시라인**(16개 라인 이상 간격)에 있습니다. False Sharing이 발생하지 않으며, 8개 스레드가 거의 선형 확장을 보입니다.

**두 번째 경우 (인덱스 0, 1, 2, ..., 7):**

인덱스 0~7은 오프셋 0~56 bytes로, 정확히 **64 bytes 한 캐시라인** 안에 들어갑니다. 전형적인 False Sharing 시나리오입니다. 스레드 하나가 쓸 때마다 나머지 7개 스레드의 캐시라인이 Invalid가 되어 핑퐁이 발생합니다.

Java에서 배열 헤더(16 bytes 내외)가 있기 때문에 실제 첫 번째 원소의 오프셋이 약간 달라질 수 있지만, 인접 원소들이 같은 라인에 있다는 사실은 변하지 않습니다. `JOL(Java Object Layout)`로 실제 오프셋을 확인할 수 있습니다.

</details>

---

**Q2.** `@Contended`가 붙은 객체를 수백만 개 생성하는 경우(예: 대용량 맵의 값으로) 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

`@Contended`는 필드 앞뒤로 **128 bytes** (x86 기준, JVM이 결정)의 패딩을 삽입합니다. 따라서 원래 8 bytes짜리 `long` 하나를 담는 객체가 128 + 8 + 128 = 264 bytes 이상을 차지하게 됩니다.

이를 100만 개 생성하면: 264MB (패딩 포함) vs 8MB (패딩 없음). 메모리 사용량이 약 33배 증가합니다.

실질적 문제:
1. **힙 공간 부족**: `-Xmx` 설정보다 빨리 메모리를 소진
2. **GC 압력 증가**: 큰 객체가 많아질수록 GC 빈도와 STW 시간 증가
3. **캐시 효율 저하**: 패딩이 L3 캐시를 낭비하여 오히려 cache-miss 증가

따라서 `@Contended`는 **소수의 고경합 카운터/플래그**에만 사용해야 합니다. 대량 생성이 필요한 객체에는 스레드 로컬 설계나 작업 분할 방식을 택하는 것이 적절합니다. `LongAdder`가 내부적으로 `Cell[]` 배열 크기를 CPU 수에 맞게 제한하는 이유도 같은 맥락입니다.

</details>

---

**Q3.** False Sharing을 피하기 위해 패딩을 많이 추가한 경우, 반대로 캐시 효율이 떨어지는 경우가 있는가? 언제 패딩이 오히려 역효과를 내는가?

<details>
<summary>해설 보기</summary>

패딩이 역효과를 내는 대표적인 상황은 **읽기 전용 또는 읽기 다중/쓰기 단일** 패턴입니다.

예를 들어, 설정값 배열 `Config[] configs = new Config[1000]`이 있고 모든 스레드가 이를 읽기만 한다고 가정합니다. 패딩 없이 연속 배치하면 64 bytes 한 라인에 여러 Config가 들어가 프리페처(prefetcher)가 효율적으로 동작합니다.

반면 각 Config를 64 bytes로 패딩하면:
1. 라인 하나에 Config 하나만 → 프리페처 효율 저하
2. 배열 전체 순회에 필요한 캐시라인 수가 대폭 증가
3. L1/L2/L3 캐시에 담을 수 있는 Config 수가 줄어들어 cache-miss 증가

**판단 기준:**
- 해당 데이터를 **여러 코어가 동시에 쓰는가?** → 패딩 필요
- **읽기 전용 또는 단일 코어가 쓰는가?** → 패딩 불필요, 오히려 연속 배치가 유리

실무에서는 프로파일러(perf c2c, async-profiler의 allocation 뷰)로 실제 핑퐁이 발생하는 라인을 확인한 후, 해당 자료구조에만 선별적으로 패딩을 적용하는 것이 권장됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: MESI 캐시 일관성](./01-mesi-cache-coherence.md)** | **[홈으로 🏠](../README.md)** | **[다음: 메모리 재정렬 ➡️](./03-memory-reordering.md)**

</div>
