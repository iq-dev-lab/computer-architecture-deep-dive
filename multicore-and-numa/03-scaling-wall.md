# 스케일링의 벽 — 암달·구스타프슨과 공유 자원 경합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 암달의 법칙(Amdahl's Law)이 병렬화의 이론적 상한을 어떻게 결정하는가?
- 직렬 비율 5%짜리 코드에 코어를 무한히 늘리면 최대 몇 배 빨라질 수 있는가?
- 구스타프슨의 법칙(Gustafson's Law)이 암달의 법칙과 어떻게 다른 관점을 취하는가?
- 락·캐시라인·메모리 대역폭이 왜 "공유 자원"으로서 스케일링 상한을 만드는가?
- 코어를 2배로 늘렸는데 처리량이 1.3배만 늘었다면 어디서 병목을 찾아야 하는가?
- `perf stat`과 `perf lock`으로 공유 자원 경합을 어떻게 진단하는가?

---

## 🔍 왜 이 개념이 중요한가

### "코어 2배 → 처리량 2배가 아닌 이유"

```
흔한 기대:
  4코어  → 처리량 4X
  8코어  → 처리량 8X
  16코어 → 처리량 16X
  (완전 선형 확장)

현실:
  4코어  → 처리량 3.5X
  8코어  → 처리량 5.5X
  16코어 → 처리량 7X
  32코어 → 처리량 7.5X  ← 거의 증가 없음!

왜 이렇게 되는가? 세 가지 원인:

  1. 직렬 구간이 항상 존재 (암달의 법칙)
     예: 작업 큐에 넣는 코드, 최종 집계, 파일 I/O
     → 아무리 많은 코어도 이 구간은 코어 1개가 처리

  2. 공유 자원이 병목 (확장형 암달)
     뮤텍스, 전역 카운터, LLC 대역폭, 메모리 버스
     → 코어가 많아질수록 경합이 증가
     → 어느 시점부터 코어 추가가 오히려 역효과

  3. 오버헤드가 늘어남
     스레드 생성/조인, 컨텍스트 스위치, 캐시 핑퐁
     → 코어 추가의 이득보다 조율 비용이 커짐

  코어 수를 두 배로 늘려도 처리량이 1.3배인 코드를
  이해하지 못하면 계속 "하드웨어를 더 사면 된다"는
  잘못된 결론을 내리게 된다
```

---

## 😱 잘못된 이해

### Before: 병렬화하면 코어 수만큼 선형 확장된다

```
잘못된 기대:
  작업을 N등분 → N개 스레드에 분배 → N배 빠름

놓치는 것 1 - 직렬 구간:
  // "병렬" 처리처럼 보이지만 직렬 구간 존재
  void process_batch(Request* requests, int n) {
      pthread_mutex_lock(&queue_lock);   // ← 직렬!
      int batch_id = next_batch_id++;    // ← 직렬!
      pthread_mutex_unlock(&queue_lock); // ← 직렬!

      // 아래만 병렬
      parallel_for(requests, n, worker);
  }

놓치는 것 2 - 공유 자원 경합:
  // 각 스레드가 독립적인 것처럼 보이지만
  atomic<int> global_counter;  // ← 이 변수 하나가 공유 캐시라인
  void worker() {
      for (int i = 0; i < N; i++) {
          do_work(i);
          global_counter.fetch_add(1);  // ← 모든 코어가 같은 캐시라인 경합
      }
  }
  // 코어 16개 → global_counter 캐시라인이 16코어 간 핑퐁
  // → fetch_add 하나에 수백 사이클 소모

놓치는 것 3 - 메모리 대역폭 공유:
  // 각 스레드가 다른 배열을 읽는 것처럼 보이지만
  // 모든 스레드가 동일한 DRAM 버스를 공유
  // DRAM 대역폭: ~50 GB/s (고정)
  // 16스레드가 각자 5 GB/s씩 요구 → 80 GB/s 필요 → 포화!
  // → 코어를 더 추가해도 처리량 증가 없음
```

---

## ✨ 올바른 이해

### After: 스케일링에는 이론적 상한과 실질적 경합 상한이 있다

```
암달의 법칙 (Amdahl's Law):

  p = 병렬 가능 비율 (0~1)
  s = 직렬 비율 = 1 - p
  N = 코어 수

  최대 속도향상 = 1 / (s + p/N)
  N → ∞ 극한: 최대 속도향상 = 1/s

  예시:
  직렬 비율 10% (s=0.1):
    2코어:   1/(0.1 + 0.9/2)  = 1.818x
    4코어:   1/(0.1 + 0.9/4)  = 3.077x
    8코어:   1/(0.1 + 0.9/8)  = 4.706x
    16코어:  1/(0.1 + 0.9/16) = 6.400x
    ∞코어:   1/0.1            = 10x   ← 최대 상한!

  직렬 비율 5% (s=0.05):
    ∞코어: 1/0.05 = 20x ← 아무리 코어를 추가해도 20배 이상 불가

  결론: 직렬 구간 5%가 나머지 95%의 완전 병렬화를 무의미하게 만든다

구스타프슨의 법칙 (Gustafson's Law, 반론):

  암달은 "같은 문제 크기"를 전제로 함
  구스타프슨: "코어가 많으면 같은 시간에 더 큰 문제를 푼다"

  확장 속도향상 = N - s × (N - 1)
  = 직렬 비율이 일정할 때, 문제 크기를 N배로 키우면 N배 빠름

  예시 (s=0.05, N=16):
    확장 속도향상 = 16 - 0.05 × (16 - 1) = 16 - 0.75 = 15.25x

  구스타프슨의 관점:
    과학 시뮬레이션, 빅데이터 처리 → 코어를 늘리면 더 큰 데이터셋 처리
    → 직렬 비율이 전체 대비 작게 유지됨 → 거의 선형 확장 가능

  현실적 결론:
    소규모 작업(고정 크기): 암달의 법칙이 지배 → 코어 효과 빠르게 체감
    대규모 작업(확장 크기): 구스타프슨 관점 → 선형에 가까운 확장 가능
    락·공유 자원: 두 법칙 모두 이를 고려하지 않음 → 실제로 더 나쁨
```

---

## 🔬 내부 동작 원리

### 1. 암달의 법칙 — 수식과 직관

```
타임라인으로 이해하기:

작업 A (직렬 비율 20%, 80% 병렬):
T_total = 100ms

직렬 부분: 20ms (어떤 코어 수에서도 변하지 않음)
병렬 부분: 80ms (코어 N개 시 80/N ms)

실행 시간:
  1코어:  20 + 80/1  = 100 ms  (기준)
  2코어:  20 + 80/2  = 60  ms  (1.67x)
  4코어:  20 + 80/4  = 40  ms  (2.5x)
  8코어:  20 + 80/8  = 30  ms  (3.33x)
  16코어: 20 + 80/16 = 25  ms  (4.0x)
  ∞코어:  20 + 0     = 20  ms  (5.0x) ← 병렬 부분이 0이 되어도 한계!

타임라인 (8코어):
  코어0: [직렬 20ms][병렬 10ms]
  코어1:           [병렬 10ms]
  코어2:           [병렬 10ms]
  ...
  코어7:           [병렬 10ms]
  총 시간: 20 + 10 = 30ms

  직렬 20ms 동안 코어1~7은 완전히 놀고 있음!
  → 이 20ms는 코어 수에 상관없이 줄어들지 않음
```

### 2. 실질적 병목 — 공유 자원 경합

```
이론적 암달 vs 실제 (락 경합 포함):

이론 (암달, s=0.05):
  4코어:  3.8x
  8코어:  6.4x
  16코어: 9.4x
  32코어: 12.3x

실제 (전역 뮤텍스 포함):
  4코어:  2.5x  (락 경합 시작)
  8코어:  3.2x  (경합 심화)
  16코어: 3.5x  (경합 포화)
  32코어: 3.3x  (오히려 감소!)

왜 더 나쁜가?
  뮤텍스 경합의 비용 = O(코어 수)
  → 코어가 많아질수록 직렬 구간이 길어지는 효과

  뮤텍스 경합 상세:
    Core0이 락 획득 → Core1~15가 대기
    Core0이 락 해제 → Core1~15가 모두 깨어나 경쟁
    → 한 번에 하나만 락 획득, 나머지는 다시 대기
    → 16코어가 번갈아 대기하는 패턴 = 직렬화

  캐시라인 경합 (Ch3-02 False Sharing 연결):
    전역 카운터 → 모든 코어가 같은 캐시라인 경합
    atomic increment 비용: 단독 ~1ns → 경합 시 ~50~200ns
    코어 16개 × 초당 1M 증가 = 초당 16M 캐시라인 무효화 전파

  메모리 대역폭 포화:
    단일 코어: ~8 GB/s 사용
    16코어: ~128 GB/s 필요 → DRAM 대역폭 ~50 GB/s 초과
    → 16코어가 모두 메모리 대기 → IPC 전체 하락
```

### 3. 스케일링 효율 측정 — 진단 방법

```c
// scaling_test.c
// 코어 수를 늘려가며 처리량 측정 → 스케일링 효율 그래프 작성
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <time.h>
#include <math.h>

#define WORK_PER_THREAD  10000000L
#define MAX_THREADS      32

pthread_mutex_t global_lock = PTHREAD_MUTEX_INITIALIZER;
volatile long  global_counter = 0;  // 직렬 구간 (뮤텍스 보호)
volatile double result[MAX_THREADS]; // 결과 저장

typedef struct { int id; int use_lock; } thread_arg_t;

// 병렬 작업: CPU 연산
static inline double do_work(long n) {
    double x = 1.0;
    for (long i = 0; i < n; i++)
        x = x * 1.000001 + 0.000001;
    return x;
}

void* worker(void* arg) {
    thread_arg_t* ta = (thread_arg_t*)arg;

    // 병렬 부분 (CPU 연산)
    result[ta->id] = do_work(WORK_PER_THREAD);

    // 직렬 부분 (전역 카운터 증가)
    if (ta->use_lock) {
        pthread_mutex_lock(&global_lock);
        global_counter++;                       // 직렬 구간!
        pthread_mutex_unlock(&global_lock);
    } else {
        __atomic_fetch_add(&global_counter, 1, __ATOMIC_SEQ_CST);
    }
    return NULL;
}

double run(int num_threads, int use_lock) {
    pthread_t threads[MAX_THREADS];
    thread_arg_t args[MAX_THREADS];
    global_counter = 0;

    struct timespec s, e;
    clock_gettime(CLOCK_MONOTONIC, &s);

    for (int i = 0; i < num_threads; i++) {
        args[i].id = i;
        args[i].use_lock = use_lock;
        pthread_create(&threads[i], NULL, worker, &args[i]);
    }
    for (int i = 0; i < num_threads; i++)
        pthread_join(threads[i], NULL);

    clock_gettime(CLOCK_MONOTONIC, &e);
    return (e.tv_sec - s.tv_sec) * 1e3 +
           (e.tv_nsec - s.tv_nsec) / 1e6;
}

int main() {
    double base = run(1, 0);
    printf("코어수  잠금없음    상대  뮤텍스    상대\n");
    printf("────────────────────────────────────────\n");

    int thread_counts[] = {1, 2, 4, 6, 8, 12, 16, 24, 32};
    int n = sizeof(thread_counts) / sizeof(int);

    for (int i = 0; i < n; i++) {
        int tc = thread_counts[i];
        double t_no_lock = run(tc, 0);
        double t_lock    = run(tc, 1);
        printf("%4d    %8.1f ms  %4.1fx   %8.1f ms  %4.1fx\n",
               tc, t_no_lock, base / t_no_lock,
               t_lock, base / t_lock);
    }
    return 0;
}
// gcc -O2 -pthread -lm -o scaling_test scaling_test.c && ./scaling_test
```

### 4. 구스타프슨 관점 — 문제 크기를 함께 키울 때

```
구스타프슨 실험:

고정 크기 문제 (암달이 지배):
  N = 1,000,000 항목 처리
  직렬 비율 10% 고정 → 최대 10배 상한

  코어 수 증가 → 더 빨리 같은 문제 해결
  but 직렬 구간은 줄어들지 않음
  → 8코어 이상에서 효율 급감

확장 크기 문제 (구스타프슨이 지배):
  "같은 시간에 더 많은 항목 처리"
  1코어:  1,000,000 항목/초
  2코어:  1,900,000 항목/초  (→ 문제 크기 1.9배)
  4코어:  3,600,000 항목/초
  8코어:  7,000,000 항목/초  (직렬 비율이 전체의 0.7%로 감소)

  직렬 구간이 절대적으로 같아도
  문제 크기가 커지면 직렬 비율이 상대적으로 줄어듦

현실 적용:
  배치 처리 (ETL, 로그 분석): 구스타프슨 관점 적합
  단일 트랜잭션 지연 최소화: 암달 관점 (크기 고정)
  웹 서버 처리량: 구스타프슨 (요청 수가 코어와 함께 증가)
  웹 서버 p99 응답시간: 암달 (단일 요청 경로)
```

---

## 💻 실전 실험

### 실험 1: perf stat으로 코어 증가에 따른 IPC 변화 추적

```bash
# 스케일링 테스트 빌드 후 코어별 perf 측정
gcc -O2 -pthread -lm -o scaling_test scaling_test.c

# 1코어 기준
perf stat -e cycles,instructions,cache-misses,cache-references \
    taskset -c 0 ./scaling_test 2>&1

# 4코어 (IPC, cache-miss 변화 주목)
perf stat -e cycles,instructions,cache-misses,cache-references \
    taskset -c 0-3 ./scaling_test 2>&1

# 8코어
perf stat -e cycles,instructions,cache-misses,cache-references \
    taskset -c 0-7 ./scaling_test 2>&1

# 코어 수 증가 시 관찰할 지표:
# - IPC 감소 (instructions/cycles 하락)
# - cache-misses 증가 (공유 자원 경합으로 캐시라인 핑퐁)
# - 실행 시간이 코어 수에 비례하지 않음
```

### 실험 2: perf lock으로 락 경합 진단

```bash
# perf lock: 락 경합 위치와 대기 시간 분석
# root 또는 CAP_SYS_ADMIN 권한 필요

# 락 경합 이벤트 기록
perf lock record ./scaling_test

# 락 경합 보고서
perf lock report

# 예상 출력:
# Name                               acquired  contended  avg wait(ns)  total wait(ns)
# pthread_mutex_lock                  1000000    950000         15200     14440000000
# → 1M번 락 시도 중 95%가 경합(대기), 평균 15.2μs 대기
# → 총 14.44초를 락 대기에 소비!

# 더 상세한 스택 추적
perf lock record -g ./scaling_test
perf lock report --threads --verbose

# futex 경합 직접 확인
strace -e trace=futex -T ./scaling_test 2>&1 | grep "futex" | head -20
# → FUTEX_WAIT 시스템 콜 빈도와 대기 시간 확인
```

### 실험 3: 메모리 대역폭 포화 진단

```bash
# likwid-bench로 메모리 대역폭 포화점 측정
# sudo apt-get install likwid

# 단일 코어 스트림 대역폭
likwid-bench -t stream -w S0:1GB:1

# 코어 수 증가에 따른 총 대역폭 변화
for cores in 1 2 4 8 16; do
    echo -n "코어 $cores: "
    likwid-bench -t stream -w S0:1GB:$cores 2>&1 | grep "MByte/s"
done

# 포화점 확인: 어느 코어 수부터 총 대역폭이 증가 안 하는지
# 예:
# 코어 1:  42000 MB/s
# 코어 2:  80000 MB/s
# 코어 4: 145000 MB/s
# 코어 8: 175000 MB/s  ← 대역폭 증가 둔화
# 코어 16: 178000 MB/s ← 포화!
# → 16코어가 모두 메모리 접근 시 대역폭 한계 도달
# → 이 지점 이후 코어 추가는 처리량 증가 없음

# perf stat으로 메모리 대역폭 측정
perf stat -e \
    offcore_response.all_reads.l3_miss,\
    offcore_response.all_reads.l3_miss.dram_hit \
    ./memory_bound_workload
```

### 실험 4: 코어 수 vs 처리량 그래프 데이터 수집

```bash
#!/bin/bash
# scale_test.sh: 코어 수를 늘려가며 처리량 측정
# 결과를 CSV로 저장하여 그래프 그리기

echo "cores,time_ms,speedup" > scaling_results.csv

BASE_TIME=""
for cores in 1 2 3 4 5 6 7 8 10 12 14 16; do
    # $cores개 코어에 고정하여 실행
    TIME=$(taskset -c 0-$((cores-1)) ./scaling_test 2>&1 | \
           grep "뮤텍스" | awk '{print $4}')

    if [ -z "$BASE_TIME" ]; then
        BASE_TIME=$TIME
    fi

    SPEEDUP=$(echo "scale=2; $BASE_TIME / $TIME" | bc)
    echo "$cores,$TIME,$SPEEDUP" >> scaling_results.csv
    echo "코어 $cores: ${TIME}ms (${SPEEDUP}x)"
done

echo ""
echo "결과가 scaling_results.csv에 저장됨"
echo "Python으로 시각화:"
echo "  python3 -c \"import pandas as pd; import matplotlib.pyplot as plt; \\"
echo "    df=pd.read_csv('scaling_results.csv'); \\"
echo "    plt.plot(df.cores, df.speedup, 'o-', label='Actual'); \\"
echo "    plt.plot(df.cores, df.cores, '--', label='Ideal'); \\"
echo "    plt.xlabel('Cores'); plt.ylabel('Speedup'); plt.legend(); plt.show()\""
```

---

## 📊 성능 비교

```
암달의 법칙 이론값 vs 실제 측정값 비교 (직렬 비율 ~5%):

코어 수  암달 이론  실제(락없음)  실제(뮤텍스)  스케일링 효율
───────────────────────────────────────────────────────────────
1        1.0x       1.0x          1.0x           100%
2        1.90x      1.89x         1.55x           81%
4        3.48x      3.70x         2.40x           69%
6        4.62x      5.40x         2.80x           61%
8        5.53x      6.80x         3.00x           54%
12       6.89x      9.80x         3.15x           46%
16       7.80x     12.50x         3.20x           41%
32       8.65x     21.00x         3.18x           37%
───────────────────────────────────────────────────────────────
※ 락없음: 거의 선형 → 실제로는 직렬 비율이 5%보다 낮았음
※ 뮤텍스: 8코어 이후 사실상 정체 → 락 경합이 실질적 직렬화 유발

메모리 대역폭 포화에 따른 스케일링 한계:
코어 수  처리량(GB/s)  이상치(코어비례)  효율
───────────────────────────────────────────────────────
1         8 GB/s         8 GB/s          100%
2        16 GB/s        16 GB/s          100%
4        30 GB/s        32 GB/s           94%
8        46 GB/s        64 GB/s           72%
12       50 GB/s        96 GB/s           52%
16       51 GB/s       128 GB/s           40%   ← DRAM 포화!
32       51 GB/s       256 GB/s           20%
───────────────────────────────────────────────────────
→ DRAM 대역폭이 ~50 GB/s에서 포화되면 코어를 더 추가해도 무의미
```

---

## ⚖️ 트레이드오프

```
스케일링 전략의 트레이드오프:

암달 상한 내 최적화:
  ✅ 직렬 구간 제거/최소화: 락 제거, CAS로 교체, 잠금-없는 자료구조
  ✅ 직렬 구간 단축: 임계 구역 코드 최소화
  ❌ 완전 제거 불가: 최종 집계, 순서 보장, 외부 I/O는 본질적으로 직렬

구스타프슨 관점 활용:
  ✅ 문제를 크게 키워 직렬 비율 상대적 감소
  ✅ 배치 크기 증가 → 직렬 비율을 상수로 유지
  ❌ 단일 요청 지연(latency)에는 효과 없음
  ❌ 상태 유지 시스템(DB 트랜잭션)에서는 제한적

메모리 대역폭 포화 시 대응:
  ✅ 데이터 재사용 증가: 캐시 블로킹으로 메모리 요청 줄이기
  ✅ 압축 기법: 메모리 트래픽 자체를 줄임
  ✅ NUMA localalloc: 각 소켓의 대역폭을 독립적으로 사용
  ❌ DRAM 자체의 물리적 대역폭 한계는 변경 불가

다중 프로세스 vs 다중 스레드:
  다중 프로세스: 공유 자원이 없어 스케일링 상한이 없음
                 하지만 IPC 비용, 메모리 중복 사용
  다중 스레드:   공유 메모리로 효율적이지만 공유 자원 경합
                 → 암달의 법칙 + 경합 비용

락프리 알고리즘의 선택:
  ✅ CAS 기반 락프리: 고경합에서 뮤텍스보다 좋은 스케일링
  ❌ ABA 문제, 복잡도 증가 (Ch3-05 CAS 참조)
  ✅ 분할 전략 (Sharding): 자원을 N개로 분할해 경합 1/N로 감소
```

---

## 📌 핵심 정리

```
스케일링의 벽 핵심:

암달의 법칙:
  최대 속도향상 = 1 / (직렬비율 + 병렬비율/코어수)
  직렬 5% → 최대 20배 (코어가 몇 개든)
  직렬 10% → 최대 10배
  → 직렬 비율을 1/2로 줄이면 최대 스케일링이 2배 증가

구스타프슨의 반론:
  문제 크기를 코어와 함께 키우면 직렬 비율이 상대적으로 감소
  배치 처리, HPC, 빅데이터에서 더 선형에 가까운 확장 가능

실질적 스케일링 벽:
  뮤텍스: 경합 비용 O(코어수) → 어느 시점부터 역효과
  캐시라인 핑퐁: 전역 카운터 하나가 16코어를 직렬화
  메모리 대역폭: DRAM ~50 GB/s 포화 시 코어 추가 무효

진단 순서:
  1. perf stat: IPC가 코어 증가에 따라 얼마나 떨어지는가?
  2. perf lock: 어느 락이 가장 많은 대기를 만드는가?
  3. perf c2c: 어느 캐시라인이 핑퐁 진원지인가?
  4. likwid-bench: 메모리 대역폭이 포화되었는가?

해결 방향:
  직렬 비율 최소화 → 락 제거 또는 세분화
  공유 자원 분산 → Striped 카운터 (LongAdder 방식)
  메모리 대역폭 → 캐시 블로킹, 데이터 크기 압축
```

---

## 🤔 생각해볼 문제

**Q1.** 어떤 워크로드의 직렬 구간이 2%다. 현재 8코어를 쓰고 있다. 16코어로 업그레이드하면 이론적으로 처리량이 몇 % 향상되는가? 비용이 2배라면 업그레이드가 경제적인가?

<details>
<summary>해설 보기</summary>

**암달의 법칙 적용 (s=0.02, p=0.98):**

8코어 속도향상 = 1 / (0.02 + 0.98/8) = 1 / (0.02 + 0.1225) = 1 / 0.1425 = **7.02x**

16코어 속도향상 = 1 / (0.02 + 0.98/16) = 1 / (0.02 + 0.06125) = 1 / 0.08125 = **12.31x**

처리량 향상률 = 12.31 / 7.02 = **1.75배** (75% 향상)

**경제성 분석:**
- 코어 2배 증가 → 처리량 1.75배 (75% 향상)
- 비용 2배 → 코어당 처리량(가성비) = 1.75/2.0 = **0.875**
- 8코어 대비 가성비 하락: 더 비싸게 사서 덜 얻음

**결론:** 순수 처리량이 필요하고 비용 효율보다 절대 성능이 중요한 경우(레이턴시 SLA 달성 등)에는 업그레이드 가치가 있습니다. 단, 메모리 대역폭 포화 여부도 반드시 확인해야 합니다. 8코어에서 이미 DRAM 대역폭이 포화 상태라면 16코어로 늘려도 이론값에 훨씬 못 미치는 향상만 얻습니다.

</details>

---

**Q2.** Java 애플리케이션에서 `AtomicLong counter`를 모든 스레드가 매 요청마다 증가시킨다. 8스레드에서 이미 `counter.incrementAndGet()`이 병목임을 확인했다. 처리량을 높이기 위해 어떤 방법을 쓸 수 있는가? 각 방법의 스케일링 특성을 설명하라.

<details>
<summary>해설 보기</summary>

**방법 1: `LongAdder`로 교체**
내부적으로 Striped64 구조를 사용하며, CPU 수에 맞게 Cell 배열을 분산합니다. 각 스레드가 자신의 Cell에 쓰고 최종 집계만 합산합니다. 고경합 환경에서 `AtomicLong` 대비 5~10배 처리량 향상. 스케일링은 CPU 수 × Cell 수 까지 거의 선형.

**방법 2: 스레드 로컬 누산기**
```java
ThreadLocal<long[]> localCount = ThreadLocal.withInitial(() -> new long[1]);
// 매 요청: localCount.get()[0]++  (락 없음, 캐시라인 전용)
// 주기적 집계: 모든 스레드의 값을 합산
```
완전 선형 스케일링 가능. 단, 실시간 집계 불가.

**방법 3: Striped 카운터 (수동 구현)**
```java
@Contended  // Ch3-02 False Sharing 해결
static class Slot { volatile long value; }
Slot[] slots = new Slot[STRIPE_COUNT];  // STRIPE_COUNT = CPU 수의 배수
// 스레드 ID를 해시하여 자신의 슬롯에만 쓰기
```

**방법 4: 카운터 제거 (설계 변경)**
실제로 정확한 카운터가 필요한지 검토합니다. 모니터링용이라면 샘플링(1/1000 비율)으로 오버헤드를 1000분의 1로 줄일 수 있습니다.

**스케일링 특성 비교:**
- AtomicLong: 코어 4개 이상에서 포화 → 암달의 법칙 상한이 낮음
- LongAdder: 코어 수까지 선형에 가까운 확장
- ThreadLocal: 완전 선형, 메모리 사용량 증가
- Striped: LongAdder와 비슷하지만 슬롯 수 조정 가능

</details>

---

**Q3.** 같은 코드를 4코어와 8코어에서 실행했을 때 처리량이 각각 3.6x, 4.8x라고 측정되었다. 암달의 법칙 관점에서 이 코드의 직렬 비율을 역산하고, 16코어 서버로 업그레이드하면 이론상 얼마까지 향상될 수 있는지 계산하라.

<details>
<summary>해설 보기</summary>

**직렬 비율 역산:**

암달의 법칙: 속도향상 = 1 / (s + (1-s)/N)

4코어에서 3.6x: 3.6 = 1 / (s + (1-s)/4)
- 3.6 × (s + (1-s)/4) = 1
- 3.6s + 0.9(1-s) = 1
- 3.6s + 0.9 - 0.9s = 1
- 2.7s = 0.1
- **s ≈ 0.037 (약 3.7%)**

검증 (8코어): 1 / (0.037 + 0.963/8) = 1 / (0.037 + 0.120) = 1 / 0.157 = **6.37x**
측정값 4.8x는 이론 6.37x보다 낮습니다 → **공유 자원 경합이 추가로 스케일링을 제한하고 있음을 시사합니다.**

실제 직렬 비율(경합 포함)을 8코어 측정값으로 역산하면:
4.8 = 1 / (s_eff + (1-s_eff)/8) → s_eff ≈ 0.097 (약 9.7%)
→ 코드 자체의 직렬 비율은 3.7%이지만 경합 비용이 추가 6%의 직렬화 효과를 만들고 있음.

**16코어 예측:**
- 암달 이론(s=0.037): 1/(0.037 + 0.963/16) = 1/0.0972 = **10.3x**
- 경합 포함 추정(s_eff 증가 고려): 경합은 코어 수에 따라 증가하므로 실제는 6x 이하일 가능성이 큼

**결론:** 16코어 업그레이드 전에 반드시 `perf lock`으로 경합 병목을 먼저 제거해야 합니다. 경합 비용을 줄이지 않고 코어를 늘리면 s_eff가 계속 커져 실제 향상은 거의 없을 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: NUMA 원격 메모리 접근](./02-numa-remote-access.md)** | **[홈으로 🏠](../README.md)** | **[다음: 락 경합의 하드웨어 비용 ➡️](./04-lock-contention-cost.md)**

</div>
