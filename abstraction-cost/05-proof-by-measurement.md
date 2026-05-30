# 측정으로 증명 — 같은 로직 두 구현의 사이클 비교

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 추상화 수준만 다른 두 구현을 정확하게 비교하는 실험을 어떻게 설계하는가?
- 컴파일러의 데드코드 제거(dead code elimination)와 상수 폴딩(constant folding)이 벤치마크를 무효화하는 방법은 무엇인가?
- `volatile`, `benchmark::DoNotOptimize`, `benchmark::ClobberMemory`로 측정 무효화를 어떻게 방지하는가?
- IPC, cache-miss rate, branch-miss rate를 한 표에 정리해 추상화 비용을 사이클로 환산하는 방법은?
- Ch7 측정 방법론에서 이 챕터의 실험이 어떻게 확장되는가?
- JMH(Java Microbenchmark Harness)에서 동일한 함정을 방지하는 방법은?

---

## 🔍 왜 이 개념이 중요한가

### "그냥 time으로 재면 안 되나요?"

```
나쁜 측정의 사례들:

문제 1 — 컴파일러가 측정 대상을 통째로 삭제:
  int sum = 0;
  for (int i = 0; i < 1000000; i++)
      sum += i;
  // sum을 이후에 사용하지 않으면:
  // → 컴파일러: "sum은 결과에 영향 없음" → 루프 전체 삭제
  // → time: 0.0 ns (루프가 없음!)
  // → "와, 정말 빠른데요?" → 측정 무의미

문제 2 — 상수 폴딩:
  int result = compute(42, 100);
  // compute(42, 100)의 결과가 상수라면:
  // → 컴파일러: 컴파일 타임에 계산 완료 → 런타임에 mov eax, <상수> 1개
  // → 런타임 비용 측정 불가

문제 3 — 워밍업 없는 측정:
  auto start = clock();
  result = my_function(data);  // 첫 번째 실행: 콜드 캐시, 콜드 BTB
  auto end = clock();
  // → 캐시 미스, 분기 예측기 미학습 상태
  // → 실제 워크로드와 완전히 다른 환경
  // → "콜드 캐시 성능"을 측정한 것 (의도한 것이 아님)

문제 4 — 측정 노이즈:
  time ./program  // OS 스케줄링, IRQ, 캐시 경합
  → 단일 실행은 분산이 매우 큼
  → 통계적으로 유의미한 결과 아님

올바른 측정:
  Google Benchmark 또는 JMH:
    ① 워밍업 이터레이션 (캐시, BTB, JIT 안정화)
    ② 반복 측정 (수백~수천 번)
    ③ 통계 계산 (평균, 중앙값, 표준편차)
    ④ DoNotOptimize로 컴파일러 최적화 방지
    ⑤ perf counters로 하드웨어 이벤트 연동
```

이 문서는 Chapter 4에서 다룬 함수 호출 비용(01), 가상 함수(02), 포인터 추적(03), branchless(04)를 **하나의 실험 설계**로 통합해 사이클 단위로 증명합니다. 그리고 Ch7의 측정 방법론이 이 실험을 어떻게 체계화하는지를 연결합니다.

---

## 😱 잘못된 이해

### Before: "time으로 재서 한쪽이 더 빠르면 그게 증명 아닌가?"

```
잘못된 측정 시나리오:

// 측정 코드 (잘못된 예)
#include <chrono>
int main() {
    auto start = std::chrono::high_resolution_clock::now();

    int result = 0;
    for (int i = 0; i < 1000000; i++)
        result += abstract_function(i);

    auto end = std::chrono::high_resolution_clock::now();
    auto ms = duration_cast<microseconds>(end - start).count();
    std::cout << ms << " us\n";
    // result를 출력하지 않음
    return 0;
}

이 코드의 문제들:
  1. result가 return에 영향 없음
     → -O2: abstract_function 루프 전체 삭제
     → 출력: "0 us" (루프 없음)

  2. 첫 실행이 콜드 캐시
     → data 배열이 캐시에 없음 → 첫 번째 실행이 느림
     → 두 번째 실행은 빠름 → 어느 것을 측정했나?

  3. 단 1회 실행 → 측정 노이즈 제거 불가
     → OS 인터럽트, TLB 미스, 컨텍스트 스위치 등

  4. 두 버전의 실행 환경이 다름
     → CPU 주파수 변동 (Turbo Boost)
     → 메모리 상태 다름
     → 공정한 비교 불가

올바른 도구:
  C++: Google Benchmark
  Java: JMH (Java Microbenchmark Harness)
  Rust: Criterion.rs
  + perf/VTune으로 하드웨어 카운터 연동
```

---

## ✨ 올바른 이해

### After: 방어적 벤치마크 설계 + 하드웨어 카운터로 근본 원인 확인

```
올바른 측정 체계:

  ① 컴파일러 최적화 방지:
     DoNotOptimize(result) → 값을 레지스터에서 살아있게 유지
     ClobberMemory()        → 메모리 내용이 바뀐 것처럼 가정

  ② 워밍업:
     Google Benchmark: --benchmark_min_time=1s
     JMH: @Warmup(iterations = 5, time = 1)
     → 캐시 워밍업, BTB 학습 완료 후 측정

  ③ 반복 측정 + 통계:
     Google Benchmark: 자동으로 충분한 반복 횟수 결정
     → 평균, 중앙값, 표준편차 제공

  ④ 하드웨어 카운터:
     perf stat -e cycles,instructions,cache-misses,branch-misses
     → "이 코드가 느린 이유"를 사이클 단위로 설명
     → IPC < 1.0: memory-bound or branch-miss
     → cache-miss > 10%: 캐시 지역성 문제
     → branch-miss > 5%: 분기 예측 실패

  ⑤ 어셈블리 확인:
     godbolt 또는 objdump -d
     → 측정하려는 코드가 실제로 실행되는지 확인
     → 의도치 않은 최적화(인라이닝, 상수 폴딩) 확인

최종 목표:
  "이 추상화는 X 사이클이 더 든다" — 숫자로 증명
```

---

## 🔬 내부 동작 원리

### 1. 컴파일러 최적화가 벤치마크를 무효화하는 방법

```cpp
// 실험 코드 작성 시 반드시 알아야 할 함정

// ============================================================
// 함정 1: Dead Code Elimination (데드코드 제거)
// ============================================================

// 잘못된 벤치마크:
void bad_benchmark() {
    int result = 0;
    for (int i = 0; i < N; i++)
        result += compute(i);
    // result를 사용하지 않음 → 컴파일러가 루프 전체 삭제!
}

// 어셈블리 확인 (-O2):
// bad_benchmark():
//     ret                    // 루프 없음!

// 해결: DoNotOptimize 또는 volatile
void good_benchmark() {
    int result = 0;
    for (int i = 0; i < N; i++)
        result += compute(i);
    benchmark::DoNotOptimize(result);  // result가 살아있음을 보장
    // 또는: volatile int sink = result;
}

// ============================================================
// 함정 2: Constant Folding (상수 폴딩)
// ============================================================

// 잘못된 벤치마크:
void bad_const_benchmark() {
    for (auto _ : state) {
        int result = heavy_compute(42, 100);  // 42, 100 상수
        // 컴파일러: heavy_compute(42, 100) = 4200 (컴파일 타임 계산)
        // → 런타임에 mov eax, 4200 한 줄
        benchmark::DoNotOptimize(result);  // result를 방어해도 compute 자체가 사라짐
    }
}

// 해결: 입력을 state에서 받거나 opaque 포인터 사용
void good_const_benchmark(benchmark::State& state) {
    int a = state.range(0);  // 런타임에 결정되는 값
    int b = state.range(1);
    for (auto _ : state) {
        int result = heavy_compute(a, b);
        benchmark::DoNotOptimize(result);
    }
}
BENCHMARK(good_const_benchmark)->Args({42, 100});

// ============================================================
// 함정 3: Loop Hoisting (루프 불변 코드 끌어올림)
// ============================================================

// 잘못된 벤치마크:
void bad_loop_benchmark(benchmark::State& state) {
    int x = 42;
    for (auto _ : state) {
        for (int i = 0; i < N; i++)
            result += x * 2;  // x * 2는 루프 불변
        // → 컴파일러: result = N * (x*2)  (루프 사라짐!)
        benchmark::DoNotOptimize(result);
    }
}

// 해결: 루프 본체가 i에 의존하도록
void good_loop_benchmark(benchmark::State& state) {
    for (auto _ : state) {
        long result = 0;
        for (int i = 0; i < N; i++)
            result += i * i;  // i에 의존 → 루프 불변 코드 없음
        benchmark::DoNotOptimize(result);
    }
}
```

### 2. `volatile`, `DoNotOptimize`, `ClobberMemory`의 동작 원리

```cpp
// ============================================================
// volatile: 메모리를 통한 관찰 강제
// ============================================================

volatile int sink;   // 이 변수에 쓸 때마다 실제 메모리 쓰기 발생

void use_volatile(int result) {
    sink = result;
    // 컴파일러: "누군가 메모리를 관찰할 수 있다" → result를 메모리에 써야 함
    // → result를 계산한 코드가 삭제되지 않음
}

// 단점: 실제 메모리 쓰기가 발생 → 측정에 메모리 쓰기 오버헤드 포함될 수 있음

// ============================================================
// benchmark::DoNotOptimize (Google Benchmark)
// ============================================================

// 내부 구현 (Clang/GCC):
template <class Tp>
inline BENCHMARK_ALWAYS_INLINE void DoNotOptimize(Tp& value) {
    asm volatile("" : "+r,m"(value) : : "memory");
    // asm volatile: 컴파일러에 "이 시점에 레지스터/메모리를 건드림"
    // "+r,m": value가 레지스터 또는 메모리로 읽히거나 쓰임
    // "memory": 메모리 모델 배리어 (다른 메모리 접근도 재배치 금지)
}

// 효과: value를 "살아있는 값"으로 표시 → value 계산하는 코드 보존
// volatile보다 가벼움: 실제 메모리 쓰기 없이 레지스터 유지 가능

// ============================================================
// benchmark::ClobberMemory
// ============================================================

inline BENCHMARK_ALWAYS_INLINE void ClobberMemory() {
    asm volatile("" : : : "memory");
    // "memory" clobber: "메모리 어딘가가 바뀌었다"고 가정
    // → 캐시된 메모리 값 무효화 → 다음 읽기는 실제 메모리에서
}

// 용도: 메모리 상태를 "더럽혀서" 반복 측정 간 캐시 효과를 일정하게
// 예: 배열을 쓴 후 ClobberMemory() → 다음 읽기가 항상 캐시 미스처럼 동작
```

### 3. Chapter 4의 모든 추상화 비용을 하나의 실험으로 통합

```cpp
// ============================================================
// 추상화 수준 비교 벤치마크 — Ch4 통합 실험
// ============================================================
// 동일한 로직: N개 정수 조건부 합산
// 조건: data[i] >= threshold 인 원소들의 합
//
// 추상화 레벨:
//   Level 0: 직접 배열 + branchless (최소 추상화)
//   Level 1: 직접 배열 + 분기
//   Level 2: std::function 콜백 (함수 포인터 오버헤드)
//   Level 3: 가상 함수 다형성 (vtable)
//   Level 4: 연결 리스트 + 가상 함수 (포인터 추적 + vtable)
// ============================================================

#include <benchmark/benchmark.h>
#include <vector>
#include <list>
#include <functional>
#include <numeric>
#include <random>

const int N = 1'000'000;
const int THRESHOLD = 500000;

// 공통 데이터 생성
std::vector<int> make_random_data(int n) {
    std::vector<int> v(n);
    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(0, 999999);
    for (auto& x : v) x = dist(rng);
    return v;
}

// ── Level 0: 직접 배열 + branchless ──────────────────────
static void BM_L0_ArrayBranchless(benchmark::State& state) {
    auto data = make_random_data(N);
    for (auto _ : state) {
        long sum = 0;
        for (int x : data) {
            int mask = -(x >= THRESHOLD);
            sum += x & mask;
        }
        benchmark::DoNotOptimize(sum);
    }
    state.SetItemsProcessed(state.iterations() * N);
}

// ── Level 1: 직접 배열 + 분기 ────────────────────────────
static void BM_L1_ArrayBranch(benchmark::State& state) {
    auto data = make_random_data(N);
    for (auto _ : state) {
        long sum = 0;
        for (int x : data)
            if (x >= THRESHOLD) sum += x;
        benchmark::DoNotOptimize(sum);
    }
    state.SetItemsProcessed(state.iterations() * N);
}

// ── Level 2: std::function 콜백 ──────────────────────────
static void BM_L2_StdFunction(benchmark::State& state) {
    auto data = make_random_data(N);
    std::function<bool(int)> pred = [](int x){ return x >= THRESHOLD; };
    for (auto _ : state) {
        long sum = 0;
        for (int x : data)
            if (pred(x)) sum += x;
        benchmark::DoNotOptimize(sum);
    }
    state.SetItemsProcessed(state.iterations() * N);
}

// ── Level 3: 가상 함수 다형성 ────────────────────────────
struct Filter {
    virtual bool accept(int x) const = 0;
    virtual ~Filter() = default;
};
struct ThresholdFilter : Filter {
    int thr;
    ThresholdFilter(int t) : thr(t) {}
    bool accept(int x) const override { return x >= thr; }
};

static void BM_L3_VirtualFunc(benchmark::State& state) {
    auto data = make_random_data(N);
    ThresholdFilter f(THRESHOLD);
    Filter* filter = &f;   // 다형 포인터 (devirt 방지)
    for (auto _ : state) {
        long sum = 0;
        for (int x : data)
            if (filter->accept(x)) sum += x;
        benchmark::DoNotOptimize(sum);
    }
    state.SetItemsProcessed(state.iterations() * N);
}

// ── Level 4: 연결 리스트 + 가상 함수 ────────────────────
// 의도적으로 단편화된 연결 리스트
struct LNode {
    int value;
    LNode* next;
};

static void BM_L4_ListVirtual(benchmark::State& state) {
    // 힙 단편화 유도: 다른 할당과 섞음
    std::vector<int*> fillers;
    std::vector<LNode*> nodes;
    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(0, 999999);
    for (int i = 0; i < N; i++) {
        nodes.push_back(new LNode{dist(rng), nullptr});
        if (i % 5 == 0) fillers.push_back(new int(i));
    }
    // 링크 연결
    for (int i = 0; i + 1 < N; i++) nodes[i]->next = nodes[i+1];

    ThresholdFilter f(THRESHOLD);
    Filter* filter = &f;

    LNode* head = nodes[0];
    for (auto _ : state) {
        long sum = 0;
        for (LNode* p = head; p; p = p->next)
            if (filter->accept(p->value)) sum += p->value;
        benchmark::DoNotOptimize(sum);
    }
    state.SetItemsProcessed(state.iterations() * N);

    for (auto p : nodes) delete p;
    for (auto p : fillers) delete p;
}

BENCHMARK(BM_L0_ArrayBranchless)->Unit(benchmark::kMillisecond);
BENCHMARK(BM_L1_ArrayBranch)->Unit(benchmark::kMillisecond);
BENCHMARK(BM_L2_StdFunction)->Unit(benchmark::kMillisecond);
BENCHMARK(BM_L3_VirtualFunc)->Unit(benchmark::kMillisecond);
BENCHMARK(BM_L4_ListVirtual)->Unit(benchmark::kMillisecond);
BENCHMARK_MAIN();
```

### 4. perf counters로 추상화 비용을 사이클로 분해

```bash
# ============================================================
# 각 레벨별 하드웨어 카운터 측정
# ============================================================

# 빌드 (프레임 포인터 유지 → perf 스택 추적 정확)
g++ -O2 -std=c++17 -fno-omit-frame-pointer \
    ch4_benchmark.cpp -lbenchmark -lbenchmark_main -lpthread \
    -o ch4_bench

# 전체 카운터 측정 (각 레벨 별도 실행 필요)
for LEVEL in L0 L1 L2 L3 L4; do
    echo "=== Level: $LEVEL ==="
    perf stat -e \
        cycles,\
        instructions,\
        L1-dcache-loads,L1-dcache-load-misses,\
        LLC-loads,LLC-load-misses,\
        branches,branch-misses \
        ./ch4_bench --benchmark_filter="BM_${LEVEL}" \
                    --benchmark_min_time=3s 2>&1 | \
        grep -E 'cycles|instructions|cache|branch|miss'
    echo ""
done
```

```bash
# 예상 출력 및 분석 (N=1,000,000, 단위: 이벤트/반복):

Level   시간(ms) IPC    cache-miss%  branch-miss%  설명
──────────────────────────────────────────────────────────────────
L0      1.8      3.42   0.08%        0.05%         이상적: 브랜치리스+배열
L1      6.9      1.12   0.10%        42.3%         분기 예측 실패 급증
L2      18.4     0.88   0.12%        44.1%         std::function 오버헤드+분기
L3      22.1     0.76   0.18%        45.8%         vtable 역참조+분기
L4      410      0.38   52.4%        47.2%         포인터 추적+vtable+분기

사이클 비용 환산 (1회 반복 기준):
  L0 → L1: 분기 예측 실패 추가      = +5.1 ms (+283%)
  L1 → L2: std::function 호출 비용  = +11.5 ms (+167%)
  L2 → L3: vtable 오버헤드          = +3.7 ms  (+20%)
  L3 → L4: 포인터 추적(캐시 미스)   = +388 ms  (+1755%)
```

### 5. JMH(Java)에서 동일한 함정 방지

```java
// JMH 벤치마크 — 추상화 비용 Java 버전
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.util.*;

@State(Scope.Benchmark)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(2)
public class AbstractionCostBenchmark {

    private static final int N = 1_000_000;
    private static final int THRESHOLD = 500_000;
    private int[] data;
    private LinkedList<Integer> list;

    @Setup
    public void setup() {
        Random rng = new Random(42);
        data = new int[N];
        for (int i = 0; i < N; i++) data[i] = rng.nextInt(1_000_000);

        // LinkedList 초기화
        list = new LinkedList<>();
        for (int x : data) list.add(x);
    }

    // Level 0: 직접 배열 순회
    @Benchmark
    public void arrayDirect(Blackhole bh) {
        long sum = 0;
        for (int x : data)
            if (x >= THRESHOLD) sum += x;
        bh.consume(sum);  // DoNotOptimize 역할
    }

    // Level 1: 람다 함수형 인터페이스
    @Benchmark
    public void lambdaFilter(Blackhole bh) {
        // 람다 사용 → JIT가 인라이닝하면 빠름 (monomorphic)
        java.util.function.IntPredicate pred = x -> x >= THRESHOLD;
        long sum = 0;
        for (int x : data)
            if (pred.test(x)) sum += x;
        bh.consume(sum);
    }

    // Level 2: 다형성 인터페이스 (여러 구현)
    interface Predicate { boolean test(int x); }
    static class ThresholdPred implements Predicate {
        int t; ThresholdPred(int t) { this.t = t; }
        public boolean test(int x) { return x >= t; }
    }

    @Benchmark
    public void polymorphicPred(Blackhole bh) {
        Predicate pred = new ThresholdPred(THRESHOLD);
        long sum = 0;
        for (int x : data)
            if (pred.test(x)) sum += x;
        bh.consume(sum);
    }

    // Level 3: LinkedList 순회 (포인터 추적)
    @Benchmark
    public void linkedListTraversal(Blackhole bh) {
        long sum = 0;
        for (int x : list)
            if (x >= THRESHOLD) sum += x;
        bh.consume(sum);
    }
}
```

```bash
# JMH 실행
java -jar target/benchmarks.jar -rf json -rff results.json

# 예상 출력 (us/op):
# arrayDirect:         ~1800 us
# lambdaFilter:        ~1900 us (JIT 인라이닝으로 거의 동일)
# polymorphicPred:     ~2100 us (invokeinterface)
# linkedListTraversal: ~95000 us (포인터 추적, GC 압박)

# JVM 연결:
# jvm-deep-dive 레포에서 JIT의 invokeinterface 최적화 상세 참조
```

---

## 💻 실전 실험

### 실험 1: 측정 코드 검증 — 어셈블리 확인

```bash
# 벤치마크 코드가 실제로 실행되는지 어셈블리로 확인
g++ -O2 -std=c++17 -S -masm=intel ch4_benchmark.cpp -o ch4_bench.s

# DoNotOptimize 없는 버전과 있는 버전 비교
grep -A 30 'BM_L1_ArrayBranch' ch4_bench.s | head -40
# → 루프가 어셈블리에 존재하는지 확인
# → DoNotOptimize 없으면 루프가 사라질 수 있음

grep -c 'vpadd\|add\|lea' ch4_bench.s
# → 루프 내 연산 명령어 개수 확인
```

### 실험 2: perf stat 워크플로우

```bash
# 1단계: 빌드
g++ -O2 -std=c++17 -fno-omit-frame-pointer -g \
    ch4_benchmark.cpp -lbenchmark -lbenchmark_main -lpthread \
    -o ch4_bench

# 2단계: 기본 성능 카운터 (IPC, 캐시, 분기)
perf stat -e \
    cycles:u,\
    instructions:u,\
    cache-references:u,\
    cache-misses:u,\
    branches:u,\
    branch-misses:u \
    ./ch4_bench --benchmark_filter="BM_L1_ArrayBranch" \
                --benchmark_min_time=5s \
                2>&1

# 3단계: L1 캐시 미스 상세
perf stat -e \
    L1-dcache-loads:u,\
    L1-dcache-load-misses:u,\
    L1-dcache-prefetch-misses:u \
    ./ch4_bench --benchmark_filter="BM_L4_ListVirtual" \
                --benchmark_min_time=5s

# 4단계: 핫스팟 프로파일
perf record -g ./ch4_bench --benchmark_filter="BM_L4_ListVirtual" \
    --benchmark_min_time=10s
perf report --call-graph dwarf
# → 어느 명령어에서 stall이 발생하는지 확인
# → L4: 포인터 역참조 명령어에서 cache-miss stall 집중
```

### 실험 3: 결과를 표로 정리 — 추상화 비용 환산

```bash
# 모든 레벨 일괄 측정 스크립트
cat > measure_all.sh << 'EOF'
#!/bin/bash
for LEVEL in L0 L1 L2 L3 L4; do
    echo -n "BM_${LEVEL}: "
    perf stat -e cycles:u,instructions:u,branch-misses:u,cache-misses:u \
        ./ch4_bench --benchmark_filter="BM_${LEVEL}" \
                    --benchmark_min_time=3s 2>&1 | \
        awk '
            /cycles:/       { cyc=$1 }
            /instructions:/ { ins=$1 }
            /branch-misses/ { bm=$1 }
            /cache-misses/  { cm=$1 }
            END { printf "cycles=%s IPC=%.2f branch-miss=%s cache-miss=%s\n",
                  cyc, ins/cyc, bm, cm }
        '
done
EOF
chmod +x measure_all.sh
./measure_all.sh
```

### 실험 4: Google Benchmark 내장 perf counters 활용

```cpp
// Google Benchmark 3.x 이상: BENCHMARK_PERF_COUNTERS
// (Linux perf_event_open 연동)
#include <benchmark/benchmark.h>

static void BM_WithPerfCounters(benchmark::State& state) {
    auto data = make_random_data(N);
    for (auto _ : state) {
        long sum = 0;
        for (int x : data)
            if (x >= THRESHOLD) sum += x;
        benchmark::DoNotOptimize(sum);
    }
    // 커스텀 카운터 추가
    state.counters["items/s"] = benchmark::Counter(
        state.iterations() * N,
        benchmark::Counter::kIsRate
    );
}

// 실행 시 --benchmark_perf_counters="CYCLES,INSTRUCTIONS" 옵션
```

```bash
./ch4_bench \
    --benchmark_perf_counters="CYCLES,INSTRUCTIONS,CACHE-MISSES,BRANCH-MISSES" \
    --benchmark_format=console

# 출력 예시:
# BM_L0  1.80ms  556M items/s  cycles=1.8G  instructions=6.2G
# BM_L1  6.90ms  145M items/s  cycles=6.9G  instructions=7.8G
# ...
```

---

## 📊 성능 비교

```
Chapter 4 추상화 비용 종합표 (N=1,000,000, x86-64 Skylake, -O2):

추상화 레벨            시간(ms)  배율    IPC   cache-miss%  branch-miss%
────────────────────────────────────────────────────────────────────────
L0: 배열 + branchless   1.8     1.0x   3.42    0.08%         0.05%
L1: 배열 + 분기          6.9     3.8x   1.12    0.10%        42.3%
L2: std::function        18.4    10.2x  0.88    0.12%        44.1%
L3: 가상 함수(단일)       22.1    12.3x  0.76    0.18%        45.8%
L4: 연결 리스트+가상함수  410     228x   0.38   52.4%         47.2%

비용 분해 분석:
  L0→L1: 분기 예측 실패 비용
    branch-miss 42% × ~15cy × 500K 미스 ≈ +5.1ms (+3.8배)
    → "랜덤 분기 하나가 3.8배 느리게 만든다"

  L1→L2: std::function 간접 호출 비용
    vtable 1단계 + type erasure 오버헤드 ≈ +11.5ms
    → std::function은 단순 함수 포인터보다 비쌈
    → 람다를 template 파라미터로 전달하면 인라이닝 가능 (L1과 동등)

  L2→L3: vtable 간접 참조 비용
    추가 메모리 역참조 2회 + 간접 분기 ≈ +3.7ms
    → vtable 자체 비용은 생각보다 작음 (L2에 이미 유사 구조 있음)

  L3→L4: 포인터 추적 비용 (가장 큰 차이)
    cache-miss 52% × ~200cy (DRAM) × 1M 노드 ≈ +388ms
    → "메모리 레이아웃이 모든 것을 지배한다"

원인별 사이클 비용 순서 (이 실험 기준):
  ① 포인터 추적(캐시 미스): +388ms (228배 주범)
  ② 랜덤 분기 예측 실패:   +5.1ms  (3.8배)
  ③ std::function 오버헤드: +3.7ms  (2배)
  ④ vtable 역참조:          +3.7ms  (1.2배)
```

---

## ⚖️ 트레이드오프

```
측정 설계의 트레이드오프:

마이크로벤치마크의 한계 (Ch7-02 연결):
  ✅ 특정 패턴의 비용을 격리해서 측정 가능
  ❌ 실제 애플리케이션에서는 여러 효과가 중첩됨
  ❌ 데이터가 캐시에 들어가는 실제 워크로드와 다를 수 있음
  ❌ CPU 동적 주파수(Turbo Boost) 변동으로 결과 흔들림
  → perf stat과 함께 하드웨어 이벤트로 절대적 사이클 측정이 더 신뢰성 높음

Google Benchmark vs 직접 측정:
  Google Benchmark 장점:
    ✅ 워밍업 자동화
    ✅ 반복 횟수 자동 결정 (통계 안정화)
    ✅ DoNotOptimize/ClobberMemory 제공
    ✅ --benchmark_min_time, --benchmark_repetitions 옵션

  직접 측정(rdtsc/perf)이 필요한 경우:
    → 단일 실행 지연시간 측정 (벤치마크 프레임워크 오버헤드 제거)
    → 나노초 이하 정밀도 필요
    → 특정 CPU 이벤트를 루프 단위로 측정

perf stat 측정 주의사항:
  --perf-stat-min_time을 충분히 길게 (3s 이상):
    → 측정 시간이 짧으면 perf 오버헤드 비율이 커짐
  CPU 주파수 고정 권장:
    cpupower frequency-set -g performance
    또는: taskset -c 0 perf stat ...  (단일 코어 고정)
  SMT(Hyper-Threading) 비활성화 권장:
    echo off > /sys/devices/system/cpu/smt/control
    → 코어 간 경합 제거로 재현성 향상

Ch7 측정 방법론으로의 다리:
  이 챕터의 실험 설계 원칙 = Ch7의 방법론 예고편
  Ch7-01: perf 완전 활용 (perf record -g, FlameGraph)
  Ch7-02: 마이크로벤치마킹의 함정 (더 상세한 방어 패턴)
  Ch7-03: 루프라인 모델 (compute-bound vs memory-bound 판단)
  Ch7-04: 최적화 우선순위 (알고리즘 → 메모리 → 벡터화 순)
  Ch7-05: 케이스 스터디 (end-to-end before/after 10배 가속)
  → 이 문서에서 "왜 측정해야 하는가"를 이해했다면
    Ch7에서 "어떻게 체계적으로 측정하는가"를 학습
```

---

## 📌 핵심 정리

```
측정으로 증명하기 핵심:

컴파일러 최적화 함정 3가지:
  1. 데드코드 제거: 결과를 사용 안 하면 루프 삭제
     → DoNotOptimize(result) 또는 volatile로 방지
  2. 상수 폴딩: 컴파일 타임 상수 입력 → 런타임 0 사이클
     → 런타임 입력(state.range) 또는 랜덤 데이터 사용
  3. 루프 호이스팅: 루프 불변 계산 → 루프 바깥으로
     → 루프 인덱스(i)에 의존하는 계산 사용

Google Benchmark 핵심:
  DoNotOptimize(val):   val의 계산 코드 삭제 방지 (레지스터 생존)
  ClobberMemory():      메모리 캐시된 값 무효화 (콜드 캐시 시뮬레이션)
  state.range(n):       런타임 파라미터 전달 (상수 폴딩 방지)
  --benchmark_min_time: 충분한 측정 시간 (최소 3~5s 권장)

perf 카운터로 원인 분석:
  IPC < 1.0   : memory-bound 또는 branch-miss 주도
  cache-miss > 10%: 캐시 지역성 문제 (포인터 추적, OOP)
  branch-miss > 10%: 분기 예측 실패 (데이터 의존 분기)
  → 숫자 없이 "느리다"고 말하지 말 것

Chapter 4 추상화 비용 결론:
  배열 branchless (기준)      1.0배
  배열 + 랜덤 분기             3.8배  (branch-miss 42%)
  std::function + 분기         10.2배 (간접 호출 + 분기)
  가상 함수 + 분기             12.3배 (vtable + 분기)
  연결 리스트 + 가상 함수      228배  (캐시 미스 52% 주도)

  → 가장 비싼 추상화: 포인터 추적 (메모리 레이아웃)
  → 두 번째: 예측 불가 분기
  → 상대적으로 저렴: vtable 자체 (메모리가 뒷받침될 때)

Ch7 연결:
  이 문서: 격리된 마이크로벤치마크로 추상화 비용 측정
  Ch7-02:  실제 애플리케이션에서의 함정 (워크로드 대표성)
  Ch7-03:  루프라인 모델로 compute/memory-bound 분류
  Ch7-05:  end-to-end 케이스 스터디 (~10배 가속)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 벤치마크 코드에는 세 가지 측정 오류가 있다. 각각을 찾아내고 수정 방법을 제시하라.

```cpp
static void BM_Broken(benchmark::State& state) {
    const int N = 1000;
    int arr[N];
    for (int i = 0; i < N; i++) arr[i] = i;  // 항상 같은 데이터

    for (auto _ : state) {
        int sum = 0;
        for (int i = 0; i < N; i++)
            sum += arr[i] * 2;
        // sum을 반환하지 않음
    }
}
BENCHMARK(BM_Broken);
```

<details>
<summary>해설 보기</summary>

**오류 1 — 데드코드 제거**: `sum`을 루프 밖에서 사용하지 않으므로 컴파일러가 내부 루프 전체를 삭제할 수 있습니다. 어셈블리로 확인하면 루프가 없고 벤치마크 시간이 거의 0ns가 됩니다.
수정: `benchmark::DoNotOptimize(sum);`을 내부 루프 끝에 추가합니다.

**오류 2 — 상수 폴딩 + 루프 호이스팅**: `arr[i] * 2`에서 `arr[i] = i`로 값이 고정되어 있으므로 컴파일러가 `sum = 0 + 0*2 + 1*2 + ... + 999*2 = 999000`을 컴파일 타임에 계산해 `mov eax, 999000`으로 치환할 수 있습니다.
수정: `arr`에 랜덤 데이터를 채우거나 `@Setup`에서 초기화하고, 값을 `state.range(0)` 기반으로 만들어 컴파일러가 예측하지 못하게 합니다.

**오류 3 — 벤치마크 루프 외부 데이터 초기화 OK, 그러나 스택 배열 문제**: `arr`가 `BM_Broken` 내부 스택에 선언되어 각 벤치마크 반복마다 동일한 메모리 주소를 재사용합니다. 실제 워크로드는 다양한 데이터를 다루는데, 이 코드는 항상 캐시에 따뜻하게 올라간 소량 데이터만 측정합니다. 큰 데이터셋의 성능을 측정하려면 `std::vector`로 충분히 크게 만들거나 `@State`에서 `Setup`으로 초기화해야 합니다.

</details>

---

**Q2.** `perf stat`으로 다음 표를 얻었다. 각 케이스의 병목이 무엇인지 진단하고, 어떤 최적화를 시도해야 하는지 설명하라.

```
케이스   IPC    cache-miss%   branch-miss%   시간(ms)
A       0.42     51.3%           2.1%          450
B       0.88      0.8%          43.7%          180
C       3.41      0.2%           0.9%           22
```

<details>
<summary>해설 보기</summary>

**케이스 A (IPC=0.42, cache-miss=51.3%)**: 전형적인 **memory-bound** 워크로드입니다. cache-miss가 51%라는 것은 절반 이상의 메모리 접근이 L1/L2 캐시를 벗어나 L3 또는 DRAM에 접근한다는 의미입니다. IPC가 0.42로 낮은 이유는 CPU가 메모리 대기 중 실행 유닛이 놀기 때문입니다.

진단: 포인터 추적, 연결 리스트, 무작위 접근 패턴이 원인일 가능성이 높습니다.
최적화: AoS → SoA 전환, Arena 할당기로 지역성 개선, 연결 리스트 → `std::vector` 전환, ECS 패턴 적용.

**케이스 B (IPC=0.88, branch-miss=43.7%)**: 전형적인 **branch-miss bound** 워크로드입니다. cache-miss는 낮아 메모리 레이아웃은 양호하지만, 43.7%의 분기 예측 실패로 IPC가 낮습니다.

진단: 데이터 의존적 조건 분기(예: `if (data[i] >= threshold)`)가 랜덤 입력에서 실행되는 경우입니다.
최적화: branchless 변환(`mask = -(cond); sum += val & mask`), 데이터 정렬 후 처리, `__builtin_unpredictable()` 힌트, `-O3` + PGO 적용.

**케이스 C (IPC=3.41, cache-miss=0.2%, branch-miss=0.9%)**: **이상적인 상태**입니다. IPC가 3 이상이면 CPU가 슈퍼스칼라 파이프라인을 충분히 활용 중이고, 캐시 미스와 분기 예측 실패가 모두 낮습니다. 이 코드는 이미 compute-bound 상태로, 추가 최적화는 SIMD 적용으로 처리량을 4~8배 늘리거나 알고리즘 개선 정도입니다.

진단 기준 요약:
- IPC < 1.0 + cache-miss > 10%: memory-bound → 데이터 레이아웃 개선
- IPC < 1.5 + branch-miss > 10%: branch-bound → branchless 또는 정렬
- IPC > 2.5 + 낮은 miss: compute-bound → SIMD 또는 알고리즘 개선

</details>

---

**Q3.** 이 문서의 실험에서 `L3 (가상 함수)` 대신 `final` 키워드를 추가하면 성능이 어떻게 달라지는가? 그리고 이 변화가 표의 어떤 카운터에 영향을 미치는가?

<details>
<summary>해설 보기</summary>

`ThresholdFilter`에 `final`을 추가하고, 포인터 타입을 `ThresholdFilter* filter = &f;`로 바꾸면 컴파일러가 devirtualization을 수행할 수 있습니다. (Ch4-02에서 다룬 내용)

**코드 변경**:
```cpp
struct ThresholdFilter final : Filter {  // final 추가
    bool accept(int x) const override { return x >= thr; }
};
ThresholdFilter f(THRESHOLD);
ThresholdFilter* filter = &f;   // Base* → Derived* 로 변경
```

**예상되는 변화**:

1. **branch-miss rate 감소**: 가상 함수 호출이 직접 호출 또는 인라이닝으로 변환됩니다. `filter->accept(x)` 간접 분기가 사라지므로 BTB 오염이 줄어듭니다. 그러나 `data[i] >= threshold` 데이터 의존 분기는 여전히 남아 있으므로 branch-miss는 완전히 사라지지 않습니다.

2. **IPC 소폭 개선**: vtable 역참조(`mov rax, [rdi]` + `call [rax]`)가 직접 call 또는 인라이닝으로 교체되면 명령어 수가 줄고 실행 유닛의 의존성 체인이 짧아집니다.

3. **추가 최적화 가능성**: 인라이닝이 성공하면 `accept(x)` 본체가 루프 안에 펼쳐지고, 컴파일러가 추가 상수 전파나 branchless 변환을 적용할 수 있습니다. 이 경우 L3가 L1에 가까운 성능으로 개선될 수 있습니다.

결론: `final` 하나로 vtable 비용 제거 + 인라이닝 기회 확보 → L3가 L1~L2 범위로 내려올 수 있습니다. 표에서 IPC가 0.76 → 1.5 이상으로 오르고, branch-miss도 45.8% → 42%로 소폭 개선될 것으로 예상됩니다. perf stat으로 직접 측정해 확인하는 것이 최선입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 분기 없는 코드](./04-branchless-code.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: SIMD와 데이터 병렬성 ➡️](../simd-data-parallelism/01-simd-basics.md)**

</div>
