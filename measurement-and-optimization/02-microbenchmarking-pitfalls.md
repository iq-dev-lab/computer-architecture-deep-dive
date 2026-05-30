# 마이크로벤치마킹의 함정 — 측정이 거짓말을 하는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 벤치마크가 "0 nanoseconds"를 출력하는 이유는 무엇이고, 어떻게 막는가?
- 컴파일러가 데드코드 제거와 상수 폴딩으로 측정 대상을 없애버리는 시나리오는?
- 워밍업 없이 측정하면 어떤 세 가지 요소가 결과를 왜곡하는가?
- Google Benchmark의 `DoNotOptimize`와 `ClobberMemory`는 정확히 무슨 역할인가?
- JMH가 Java 벤치마킹에서 필수인 이유는 JIT의 어떤 특성 때문인가?
- 측정 분산이 클 때 평균 대신 중앙값을 써야 하는 통계적 이유는?
- "이 결과가 의미 있는 수치인가"를 판단하는 기준은 무엇인가?

---

## 🔍 왜 이 개념이 중요한가

### "최적화 후 10배 빨라졌다 — 정말?"

```
마이크로벤치마킹의 함정이 실무에서 일으키는 사고:

사고 1: 0ns 벤치마크
  auto start = high_resolution_clock::now();
  for (int i = 0; i < 1000000; i++)
      result = compute(i);         // 컴파일러: result 안 씀 → 삭제
  auto end = high_resolution_clock::now();
  // 출력: 2 ns → 컴파일러가 루프 자체를 없앤 것

사고 2: 콜드 캐시로 "너무 느린" 결과
  // 첫 번째 측정: 40ms (L3 미스 다발, DRAM 접근)
  // 두 번째 측정: 4ms (L3 히트)
  // 결론: 첫 번째가 "진짜 성능"이라고 보고 → 엉뚱한 곳 최적화

사고 3: JIT 미컴파일 상태 측정 (Java)
  // 처음 10회: 느림 (인터프리터 실행)
  // 11~1000회: 느림 (JIT C1 컴파일)
  // 1001회~: 빠름 (JIT C2/GraalVM 컴파일)
  // 워밍업 없이 10회만 측정 → "느린 Java" 결론

이런 함정을 모르면:
  최적화하지 않은 것을 최적화했다고 착각하거나
  이미 충분히 빠른 것을 느리다고 오판하거나
  잘못된 데이터 기반으로 아키텍처 결정을 내린다
```

---

## 😱 잘못된 이해

### Before: "간단한 루프로 시간을 재면 된다"

```cpp
// 흔히 보이는 잘못된 벤치마크 패턴

// 문제 1: 데드코드 제거
int result;
auto t0 = now();
for (int i = 0; i < N; i++) {
    result = expensive_compute(i);  // result가 이후에 쓰이지 않음
}                                   // → 컴파일러가 루프 전체 제거 가능
auto t1 = now();
// result를 출력하면 막을 수 있다고 생각하지만:
// expensive_compute가 순수 함수라면 루프를 한 번으로 줄임

// 문제 2: 상수 폴딩
const int x = 42;
auto t0 = now();
for (int i = 0; i < N; i++) {
    result = x * x + x;  // 컴파일 타임에 42*42+42 = 1806으로 계산 완료
}                        // → 루프가 "result = 1806" 대입 1회로 변환
auto t1 = now();

// 문제 3: 워밍업 없는 초기 측정
clock_t start = clock();
process_large_data(data, N);  // 처음 실행 → 콜드 캐시
clock_t end = clock();
// 다음에 같은 함수를 프로덕션에서 호출하면 5배 빠를 수 있음
// 왜냐하면 반복 실행 시 L3에 데이터가 상주하기 때문

// 문제 4: 시스템 노이즈 무시
// 한 번만 측정 → OS 스케줄러, 인터럽트, 다른 프로세스의 영향 반영
// 실제 값: 4.2ms / 측정값: 7.8ms (스케줄러가 중간에 선점)
```

---

## ✨ 올바른 이해

### After: 측정 프레임워크는 컴파일러·CPU·OS의 개입을 체계적으로 차단한다

```
올바른 마이크로벤치마킹의 세 축:

축 1 — 최적화 차단:
  컴파일러가 측정 대상을 제거하거나 단순화하지 못하도록
  benchmark::DoNotOptimize(result)  // C++ Google Benchmark
  Blackhole.consume(result)         // Java JMH

축 2 — 워밍업 충분히 수행:
  콜드 캐시 → 워밍업 N회 → 측정
  JIT 미컴파일 → 충분히 호출하여 C2 컴파일 유도 → 측정
  BTB(분기 타겟 버퍼) 빌드 → 측정

축 3 — 통계적 처리:
  최소 30회 이상 반복 측정
  이상치 제거 (상위 5%, 하위 5% 제거)
  평균 대신 중앙값 또는 p95 사용
  표준편차 / CV(변동 계수) 함께 보고

이 세 축 없이는 어떤 벤치마크도 신뢰할 수 없다.
```

---

## 🔬 내부 동작 원리

### 1. 컴파일러 최적화가 벤치마크를 파괴하는 메커니즘

```c
// 예제: 소수 판별 벤치마크 (C)
bool is_prime(int n) {
    if (n < 2) return false;
    for (int i = 2; i * i <= n; i++)
        if (n % i == 0) return false;
    return true;
}

// ---- 잘못된 벤치마크 ----
void bad_bench() {
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 1000000; i++) {
        bool r = is_prime(9999991);  // 상수 인자! 컴파일러가 결과 미리 계산
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    // 실제로는 is_prime(9999991) → true 를 컴파일 타임에 판단 가능
    // r이 사용되지 않으면 루프 전체 제거
    // → 측정 시간 ≈ 0
}

// 컴파일러가 만드는 실제 어셈블리 (-O2 기준):
// bad_bench():
//   sub    rsp, 8
//   call   clock_gettime   ← t0
//   # 루프 없음! 컴파일러가 dead store 제거
//   call   clock_gettime   ← t1
//   # 결과: 0~2 ns
//   ret

// ---- 올바른 벤치마크 ----
#include <benchmark/benchmark.h>

static void BM_IsPrime(benchmark::State& state) {
    for (auto _ : state) {
        bool result = is_prime(9999991);
        benchmark::DoNotOptimize(result);  // 컴파일러에게: result를 실제로 씀
    }
}
BENCHMARK(BM_IsPrime);

// DoNotOptimize의 구현 (GCC/Clang):
// template <class Tp>
// inline void DoNotOptimize(Tp& value) {
//     asm volatile("" : "+r,m"(value) : : "memory");
// }
// "+r,m": 값이 레지스터/메모리에서 수정될 수 있음을 컴파일러에게 알림
// "memory": 메모리 배리어 역할 → 전후 메모리 재정렬 차단
// → 컴파일러는 value 계산을 생략할 수 없게 됨
```

### 2. 상수 폴딩과 루프 최적화 탈출

```cpp
// 상수 폴딩 (Constant Folding) 예시

// 문제: 인자가 컴파일 타임에 알려진 경우
void bench_with_const() {
    const int SIZE = 1024;
    std::vector<int> v(SIZE, 1);

    auto t0 = std::chrono::high_resolution_clock::now();
    int sum = 0;
    for (int x : v) sum += x;  // sum = 1024 (컴파일러가 알 수 있음)
    auto t1 = std::chrono::high_resolution_clock::now();
    // 컴파일러: sum = 1024; 루프 없음!
}

// 해결책 1: volatile — 컴파일러 최적화 완전 차단 (but 매우 비쌈)
volatile int prevent_opt = 0;
sum += prevent_opt;  // 컴파일러: 값을 모름 → 최적화 차단

// 해결책 2: DoNotOptimize + ClobberMemory (Google Benchmark)
static void BM_Sum(benchmark::State& state) {
    std::vector<int> v(1024, 1);
    for (auto _ : state) {
        int sum = 0;
        for (int x : v) sum += x;
        benchmark::DoNotOptimize(sum);
        // ClobberMemory: v의 내용이 외부에서 변경 가능하다고 알림
        benchmark::ClobberMemory();
    }
}

// 해결책 3: 런타임 데이터 사용 (가장 현실적)
static void BM_SumRuntime(benchmark::State& state) {
    std::vector<int> v(state.range(0));
    std::iota(v.begin(), v.end(), 0);  // 런타임 초기화

    for (auto _ : state) {
        int sum = 0;
        for (int x : v) sum += x;
        benchmark::DoNotOptimize(sum);
    }
    state.SetBytesProcessed(state.iterations() * v.size() * sizeof(int));
}
BENCHMARK(BM_SumRuntime)->Range(64, 1 << 20);

// godbolt에서 확인:
// -O2로 컴파일 후 sum += x 루프가 SIMD로 벡터화되는지 확인
// DoNotOptimize 있을 때 vs 없을 때 어셈블리 비교
// → DoNotOptimize 없으면 루프 사라짐, 있으면 ymm 레지스터로 벡터화 확인
```

### 3. 워밍업의 세 가지 차원

```
워밍업이 필요한 세 가지 이유:

──────────────────────────────────────────────────────────

차원 1: 캐시 워밍업 (Cold Cache vs Hot Cache)

콜드 캐시 (첫 실행):
  - 전체 워킹셋이 DRAM에 있음
  - 첫 N 접근: L3/DRAM 미스 다발
  - 지연: 200 사이클 × miss_count

핫 캐시 (워밍업 후):
  - 워킹셋이 L3 (또는 L2/L1)에 상주
  - 반복 실행: L3/L2 히트 위주
  - 지연: 4~40 사이클 × access_count

실제 측정값 차이:
  함수 A: 콜드 40ms / 핫 4ms → 10배 차이
  → 콜드만 측정하면 "느리다"고 오판

어느 것이 맞는 기준인가?
  → 서비스 시작 직후 한 번: 콜드가 현실적
  → 지속적으로 호출되는 서비스: 핫이 현실적
  → 반드시 시나리오를 명시해야 함

──────────────────────────────────────────────────────────

차원 2: BTB(Branch Target Buffer) 워밍업

첫 실행:
  - BTB가 비어있음 (분기 이력 없음)
  - 거의 모든 분기가 "콜드 미스" (중립 예측)
  - 분기 미스 패널티: 10~20 사이클/미스

워밍업 후:
  - BTB에 분기 이력 축적
  - 잘 예측되는 패턴: 미스율 <1%
  - 측정 시간 현저히 감소

──────────────────────────────────────────────────────────

차원 3: JIT 컴파일 워밍업 (Java/JVM)

Java 실행 단계:
  단계 0 (인터프리터):   호출 1~100회    → 최저 속도
  단계 1 (C1 컴파일):   호출 100~1000회  → 중간 속도 (빠른 컴파일, 기본 최적화)
  단계 2 (C1 프로파일): 호출 1000~2000회 → 중간 속도 (프로파일 데이터 수집)
  단계 4 (C2 컴파일):   호출 2000+회     → 최고 속도 (인라이닝, 탈출 분석, SIMD)

JMH 기본 워밍업: 5 fork × 5 warmup iteration
→ 대략 10,000+ 호출 이후 C2 컴파일 완료 후 측정

JMH 없이 측정 시:
  처음 1000번: 인터프리터/C1 → "Java 느림" 결론
  2000번 이후: C2로 최적화된 JIT → "빠른 Java"
  → JVM 워밍업 미고려가 Java 성능 오해의 주요 원인
```

### 4. 측정 분산의 통계적 처리

```python
# 벤치마크 결과의 통계적 분석 예시 (Python pseudo-code)

import numpy as np

# 100회 측정값 (ms)
measurements = [4.2, 4.1, 4.3, 4.2, 12.8, 4.1, 4.4, 4.2, 4.0, 4.3, ...]
#                                    ^^ 이상치: OS 스케줄러 선점, 인터럽트

# 나쁜 방법: 산술 평균
mean = np.mean(measurements)
# 이상치 12.8ms가 평균을 4.2 → 4.7로 끌어올림

# 좋은 방법 1: 중앙값
median = np.median(measurements)
# 이상치에 영향 없음 → 4.2ms

# 좋은 방법 2: Trimmed Mean (상·하위 5% 제거)
trimmed = np.mean(sorted(measurements)[5:-5])

# 좋은 방법 3: p50/p95/p99 함께 보기 (서비스 지연 관점)
p50  = np.percentile(measurements, 50)   # 중앙값
p95  = np.percentile(measurements, 95)   # 상위 5%
p99  = np.percentile(measurements, 99)   # 상위 1%

print(f"p50={p50:.1f}ms  p95={p95:.1f}ms  p99={p99:.1f}ms")
# p50=4.2ms  p95=5.1ms  p99=12.8ms
# → p99만 높다면: 간헐적 OS 선점 또는 GC 일시정지

# CV (변동 계수) = 표준편차 / 평균
# CV < 5%: 안정적인 벤치마크 → 결과 신뢰
# CV > 10%: 불안정 → 환경 노이즈 제거 필요
cv = np.std(measurements) / np.mean(measurements) * 100
print(f"CV={cv:.1f}%")

# 이상치 감지: IQR 방법
q1, q3 = np.percentile(measurements, [25, 75])
iqr = q3 - q1
lower = q1 - 1.5 * iqr
upper = q3 + 1.5 * iqr
outliers = [x for x in measurements if x < lower or x > upper]
print(f"이상치: {outliers}")
```

---

## 💻 실전 실험

### 실험 1: Google Benchmark 기본 사용

```cpp
// install: apt install libbenchmark-dev
// build:   g++ -O2 -o bench bench.cpp -lbenchmark -lpthread
// run:     ./bench --benchmark_filter=BM_.*

#include <benchmark/benchmark.h>
#include <vector>
#include <numeric>
#include <algorithm>

// ---- 잘못된 버전 (컴파일러가 제거할 수 있음) ----
static void BM_BadSum(benchmark::State& state) {
    std::vector<int> v(state.range(0), 1);
    for (auto _ : state) {
        int sum = 0;
        for (int x : v) sum += x;
        // sum 사용 안 함 → 컴파일러가 루프 제거
    }
}

// ---- 올바른 버전 ----
static void BM_GoodSum(benchmark::State& state) {
    std::vector<int> v(state.range(0), 1);
    for (auto _ : state) {
        int sum = 0;
        for (int x : v) sum += x;
        benchmark::DoNotOptimize(sum);   // sum 강제로 "사용"
        benchmark::ClobberMemory();      // 메모리 내용이 외부에서 변경 가능함을 알림
    }
    // 처리량 보고
    state.SetBytesProcessed(
        int64_t(state.iterations()) * int64_t(state.range(0)) * sizeof(int));
}

// ---- 랜덤 접근 (메모리 병목 재현) ----
static void BM_RandomAccess(benchmark::State& state) {
    int n = state.range(0);
    std::vector<int> data(n);
    std::iota(data.begin(), data.end(), 0);

    // 랜덤 인덱스 미리 생성
    std::vector<int> indices(n);
    std::iota(indices.begin(), indices.end(), 0);
    std::shuffle(indices.begin(), indices.end(), std::mt19937{42});

    for (auto _ : state) {
        int sum = 0;
        for (int idx : indices) {
            sum += data[idx];
        }
        benchmark::DoNotOptimize(sum);
    }
}

// 테스트 크기 등록 (1KB ~ 64MB)
BENCHMARK(BM_BadSum)->Range(256, 1 << 24);
BENCHMARK(BM_GoodSum)->Range(256, 1 << 24);
BENCHMARK(BM_RandomAccess)->Range(256, 1 << 24);
BENCHMARK_MAIN();

// 실행 시 추가 옵션:
// --benchmark_repetitions=5       반복 횟수
// --benchmark_report_aggregates_only  median/mean/stddev 요약
// --benchmark_out=result.json     JSON으로 결과 저장
// --benchmark_out_format=json
```

```bash
# 실행 및 출력 예시
./bench --benchmark_repetitions=5 --benchmark_report_aggregates_only

# Run on (8 X 3600 MHz CPU s)
# -------------------------------------------------------------------
# Benchmark                          Time      CPU   Iterations
# -------------------------------------------------------------------
# BM_BadSum/1024                   0.003 ns  0.003 ns  (삭제됨!)
# BM_GoodSum/1024                  132 ns    131 ns    5316040
# BM_GoodSum/65536                 6893 ns   6840 ns     102261
# BM_GoodSum/1048576               212345 ns 210000 ns    3302
# BM_RandomAccess/65536            156823 ns 155000 ns    4521
#   BM_GoodSum_median              212345 ns 210000 ns
#   BM_GoodSum_stddev               4123 ns   3890 ns   ← CV = 1.9%
#   BM_RandomAccess_median         156823 ns 155000 ns
#   BM_RandomAccess_stddev          12456 ns  12100 ns  ← CV = 7.8%

# BM_BadSum이 0.003ns → DoNotOptimize 없으면 루프가 제거됨!
# RandomAccess의 CV가 높음 → 메모리 대역폭 노이즈, 더 많은 반복 필요
```

### 실험 2: JMH로 Java 벤치마킹

```java
// JMH (Java Microbenchmark Harness) 사용법
// Maven: org.openjdk.jmh:jmh-core:1.37

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.options.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(2)  // 2번 JVM 재시작 (JVM 자체 편향 제거)
public class SumBenchmark {

    @Param({"1024", "65536", "1048576"})
    int size;

    int[] data;

    @Setup
    public void setup() {
        data = new int[size];
        for (int i = 0; i < size; i++) data[i] = i;
    }

    // 잘못된 벤치마크 — JIT이 Dead Code 제거 가능
    @Benchmark
    public void badSum() {
        int sum = 0;
        for (int x : data) sum += x;
        // sum 반환 안 함 → JIT: 루프 전체 제거 가능!
    }

    // 올바른 벤치마크 — 반환값으로 사용
    @Benchmark
    public int goodSum() {
        int sum = 0;
        for (int x : data) sum += x;
        return sum;  // 반환 → JIT이 루프를 제거할 수 없음
    }

    // Blackhole 사용 (명시적 소비)
    @Benchmark
    public void blackholeSum(Blackhole bh) {
        int sum = 0;
        for (int x : data) sum += x;
        bh.consume(sum);  // JMH Blackhole: JIT 최적화 차단
    }
}

// JMH 실행 후 예상 결과 (size=1048576):
// Benchmark              (size)  Mode  Cnt     Score     Error  Units
// SumBenchmark.badSum   1048576  avgt   20     0.234 ±   0.012  ns/op ← JIT 제거!
// SumBenchmark.goodSum  1048576  avgt   20  212456.3 ± 4123.4  ns/op ← 실제 측정
// SumBenchmark.bh       1048576  avgt   20  213102.7 ± 3987.1  ns/op ← 일치 확인
```

### 실험 3: 시스템 노이즈 줄이기

```bash
# 벤치마크 환경 정제 체크리스트

# 1. CPU 주파수 고정 (Turbo Boost, DVFS 비활성화)
sudo cpupower frequency-set -g performance
# 또는
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 2. Hyper-Threading 비활성화 (일부 벤치마크에서 필요)
echo 0 | sudo tee /sys/devices/system/cpu/cpu*/online  # 짝수 코어만 남기기
# 더 정확히: BIOS에서 HT 비활성화

# 3. 특정 코어에 벤치마크 고정 (OS 스케줄러 이동 방지)
taskset -c 2 ./bench  # 코어 2번에 고정

# 4. NUMA 노드 고정 (메모리 지역성 고정)
numactl --cpunodebind=0 --membind=0 ./bench

# 5. 백그라운드 프로세스 최소화
# 불필요한 데몬, 브라우저, IDE 종료

# 6. ASLR(주소 공간 무작위화) 비활성화 (재현성 향상)
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
# 벤치마크 후 복원: echo 2 | sudo tee /proc/sys/kernel/randomize_va_space

# 7. IRQ 어피니티 조정 (인터럽트가 벤치마크 코어에 오지 않도록)
# 인터럽트를 코어 0으로 집중, 벤치마크를 코어 2~N에 고정

# 이 모든 설정을 적용한 벤치마크 명령 예시:
sudo cpupower frequency-set -g performance && \
  numactl --cpunodebind=0 --membind=0 \
  taskset -c 2 \
  ./bench --benchmark_repetitions=10 \
          --benchmark_report_aggregates_only
```

---

## 📊 성능 비교

```
DoNotOptimize 유무에 따른 측정 차이 (Google Benchmark, x86-64):

함수               DoNotOptimize 없음   DoNotOptimize 있음   실제 소요
─────────────────────────────────────────────────────────────────────
is_prime(const)    0.003 ns (삭제됨)    2,134 ns             2,134 ns
array_sum(1M)      0.001 ns (삭제됨)    212,456 ns           212,456 ns
matrix_mul(128)    0.002 ns (삭제됨)    45,234,123 ns        45,234,123 ns
─────────────────────────────────────────────────────────────────────

워밍업 단계별 Java 성능 (JMH, 정수 정렬 1만 개):

반복 횟수       모드            시간/op
─────────────────────────────────────
1~100          인터프리터       8,450 ns
101~1,000      C1 (tier1)      1,230 ns
1,001~2,000    C1+프로파일     980 ns
2,001+         C2 (tier4)      185 ns   ← 실제 프로덕션 성능
─────────────────────────────────────

워밍업 5회 iteration 후 측정 vs 워밍업 없이 측정:
  워밍업 없음: 2,100 ns/op (인터프리터 + C1 혼합)
  워밍업 충분: 185 ns/op
  → 11배 차이! 워밍업 없이는 Java 성능을 11배 과소평가

통계 방법에 따른 결과 차이 (100회 측정, 5개 이상치 포함):

방법              결과    이상치 영향
─────────────────────────────────────
산술 평균          4.89ms  있음 (이상치 12.8ms가 끌어올림)
중앙값             4.20ms  없음
Trimmed mean(5%)   4.21ms  없음
p95                5.10ms  있음 (설계 의도)
p99               12.80ms  있음 (설계 의도)
─────────────────────────────────────
```

---

## ⚖️ 트레이드오프

```
마이크로벤치마킹 접근법별 트레이드오프:

Google Benchmark (C++):
  ✅ DoNotOptimize/ClobberMemory로 컴파일러 최적화 차단
  ✅ 자동 반복 횟수 조정 (통계적으로 충분한 샘플 확보)
  ✅ CSV/JSON 출력으로 자동화 가능
  ❌ 빌드 의존성 추가
  ❌ JVM이 아닌 C++ 전용

JMH (Java):
  ✅ JVM fork로 JIT warm-up 상태 제어
  ✅ Blackhole로 dead code 제거 방지
  ✅ @State로 공유/스레드 로컬 상태 명확히 구분
  ❌ 설정이 많고 복잡 (어노테이션 다수)
  ❌ 빌드가 무거움 (Gradle/Maven 통합 필요)

직접 시간 측정 (`clock_gettime`, `chrono`):
  ✅ 외부 의존성 없음, 빠른 프로토타이핑
  ❌ 컴파일러 최적화 차단 방법 직접 구현 필요
  ❌ 통계 처리 부재
  ❌ 워밍업 직접 관리 필요
  → 아이디어 검증에는 OK, 정식 벤치마크에는 부족

마이크로벤치마크 vs 매크로벤치마크:
  마이크로: 함수 하나의 성능을 격리해 측정
    ✅ 원인 특정이 쉬움
    ❌ 실제 시스템 컨텍스트(캐시 경쟁, 멀티스레드) 반영 안 됨

  매크로: 전체 시스템 성능 측정 (e.g., wrk/siege로 HTTP 서버 부하)
    ✅ 실제 사용 패턴 반영
    ❌ 병목 원인 파악이 어려움

  두 접근을 함께 사용하는 것이 표준:
    마이크로로 의심 함수를 검증 → 매크로로 전체 효과 확인

  performance-testing-deep-dive 레포의 "레이어별 부하 테스트" 방법론과
  이 챕터의 마이크로벤치마킹이 서로 보완적 관계에 있음
```

---

## 📌 핵심 정리

```
마이크로벤치마킹의 함정과 방어:

함정 1 — 데드코드 제거:
  증상: 측정 시간이 0~수 ns
  원인: 컴파일러가 사용되지 않는 계산을 제거
  방어: DoNotOptimize(result) / return result / Blackhole.consume()

함정 2 — 상수 폴딩:
  증상: 측정 시간이 비현실적으로 빠름
  원인: 컴파일 타임에 결과 계산 완료
  방어: 런타임 데이터 사용, ClobberMemory()

함정 3 — 콜드 캐시 편향:
  증상: 반복 측정 시 처음이 비정상적으로 느림
  원인: L1/L2/L3 미스가 첫 실행에 집중
  방어: 워밍업 반복 후 측정, 측정값 중앙값 사용

함정 4 — JIT 미컴파일 편향 (Java):
  증상: 초반 10~100회가 느리고 이후 갑자기 빨라짐
  원인: C2 컴파일은 충분한 호출 카운트 후 실행
  방어: JMH의 @Warmup, 최소 2000+ 호출 워밍업

함정 5 — 시스템 노이즈:
  증상: 동일 벤치마크의 측정값 편차가 큼 (CV > 10%)
  원인: OS 스케줄러 선점, 인터럽트, DVFS, 다른 프로세스
  방어: 코어 고정(taskset), 주파수 고정, 반복 측정 + 중앙값

통계 처리 원칙:
  평균 대신 중앙값 (이상치 영향 차단)
  p50/p95/p99 함께 보고 (SLA 설계 연결)
  CV < 5%: 신뢰 가능 / CV > 10%: 환경 재검토
  최소 30회 이상 반복
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 C++ 벤치마크 코드에서 측정값이 "1 ns" 이하로 나온다. 왜 그런가? 올바르게 고쳐라.

```cpp
static void BM_HashCompute(benchmark::State& state) {
    std::string key = "benchmark_test_key";
    for (auto _ : state) {
        std::hash<std::string> hasher;
        size_t h = hasher(key);
    }
}
BENCHMARK(BM_HashCompute);
```

<details>
<summary>해설 보기</summary>

**원인**: `h`가 루프 이후에 사용되지 않으므로 컴파일러가 `hasher(key)` 호출 자체를 제거할 수 있습니다. `std::hash<std::string>`가 부작용 없는 순수 함수(pure function)이기 때문입니다.

**추가 문제**: `key`가 루프 바깥에 있어 반복마다 같은 값을 해시합니다. 상수 폴딩이 발생할 수 있습니다.

**올바른 버전**:
```cpp
static void BM_HashCompute(benchmark::State& state) {
    std::string key = "benchmark_test_key";
    for (auto _ : state) {
        std::hash<std::string> hasher;
        size_t h = hasher(key);
        benchmark::DoNotOptimize(h);       // h 소비
        benchmark::ClobberMemory();         // key가 외부에서 변경 가능함을 알림
    }
}
```

`ClobberMemory()`는 메모리 배리어 역할을 하여 컴파일러가 `key`의 내용을 캐싱하거나 `hasher(key)` 호출을 생략하지 못하게 합니다. `DoNotOptimize(h)`는 `h`의 계산이 완료되어야 함을 강제합니다.

godbolt에서 `-O2`로 확인하면, `DoNotOptimize` 없을 때는 루프가 없고, 있을 때는 `std::hash<std::string>::operator()` 호출 어셈블리가 실제로 생성됩니다.

</details>

---

**Q2.** JMH 벤치마크를 `@Fork(0)`으로 설정하면 어떤 문제가 생기는가? `@Fork(2)`와 `@Fork(5)`의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

**`@Fork(0)` 문제:**

Fork를 0으로 설정하면 현재 JVM 프로세스에서 직접 벤치마크를 실행합니다. 이 경우:

1. **이전 코드의 JIT 컴파일 상태가 영향을 미칩니다.** 테스트 코드나 다른 벤치마크가 JIT 캐시를 오염시켜 결과가 달라집니다.
2. **JVM 힙 상태가 공유됩니다.** 이전 테스트가 만든 쓰레기 데이터가 GC를 유발하여 측정값이 불안정해집니다.
3. **인라이닝 판단이 바뀔 수 있습니다.** JIT는 호출 이력을 기반으로 인라이닝을 결정하는데, 다른 코드가 동일 메서드를 호출했다면 판단이 달라집니다.

`@Fork(0)`은 IDE에서 빠른 확인용으로만 사용하고, 정식 측정에는 반드시 1 이상을 설정해야 합니다.

**`@Fork(2)` vs `@Fork(5)`:**

각 fork는 완전히 새로운 JVM 인스턴스를 시작합니다. Fork가 많을수록:
- 서로 다른 JVM 초기화 상태에서의 결과를 평균 → JVM 자체 편향 감소
- 전체 측정 시간 증가 (fork당 warmup + measurement)

`@Fork(2)`: 빠른 반복 개발 시. 기본 편향 제거.
`@Fork(5)`: 정확한 비교가 필요한 성능 회귀 테스트. 결과의 신뢰도가 높아짐.

일반적으로 `@Fork(2)` + `@Warmup(iterations=5)` + `@Measurement(iterations=10)`이 균형 잡힌 설정입니다.

</details>

---

**Q3.** 두 구현을 비교하는 벤치마크를 만들었다. A가 100번 측정 결과 평균 4.2ms, B가 4.1ms로 나왔다. "B가 A보다 빠르다"고 결론 내릴 수 있는가?

<details>
<summary>해설 보기</summary>

**결론 내리기 이릅니다.** 2.4% 차이는 측정 노이즈 범위 안에 있을 수 있습니다.

확인해야 할 사항:

**1. 표준편차 확인:**
- A: 평균 4.2ms, 표준편차 0.8ms → CV = 19% → 불안정
- B: 평균 4.1ms, 표준편차 0.05ms → CV = 1.2% → 안정적

이 경우 A의 측정이 불안정하여 4.2ms를 신뢰하기 어렵습니다.

**2. 통계적 유의성 검정:**
t-test 또는 Mann-Whitney U-test로 두 분포가 통계적으로 다른지 확인해야 합니다. Google Benchmark의 `--benchmark_enable_random_interleaving`와 `--benchmark_repetitions=50`을 함께 사용하면 두 벤치마크를 섞어 측정하여 시스템 상태 편향을 제거합니다.

**3. 실질적 유의성:**
2.4%의 차이가 SLA 관점에서 의미 있는가? p99 지연이 10% 이상 차이나지 않는다면 코드 가독성을 우선할 수 있습니다.

**올바른 결론 방식:**
- 표준편차가 차이보다 작고 (e.g., 표준편차 0.05ms, 차이 0.1ms)
- CV < 5%이고
- 통계 검정 p-value < 0.05

이 세 조건을 모두 만족할 때 "B가 통계적으로 유의미하게 빠르다"고 말할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: perf 완전 활용](./01-perf-deep-dive.md)** | **[홈으로 🏠](../README.md)** | **[다음: 루프라인 모델 ➡️](./03-roofline-model.md)**

</div>
