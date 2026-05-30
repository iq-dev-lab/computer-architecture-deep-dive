# 분기 vs 분기 없는 코드 — branchless·cmov·SIMD 마스킹

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 데이터 의존 분기(data-dependent branch)가 분기 예측을 무력화하는 이유는 무엇인가?
- x86의 `cmov`(조건 이동, conditional move) 명령은 분기를 어떻게 제거하는가?
- 절댓값, min, max, clamp를 branchless로 구현하는 패턴은 무엇인가?
- branchless 코드가 항상 빠른 것이 아닌 이유는 무엇인가?
- 분기 예측 비용(Ch1-05)과 branchless 선택 사이의 트레이드오프를 어떻게 판단하는가?
- SIMD 마스킹(masking)을 이용한 벡터 수준 분기 제거는 어떻게 동작하는가?

---

## 🔍 왜 이 개념이 중요한가

### "if 문 하나가 왜 성능 문제가 되는가?"

```
CPU 파이프라인과 분기의 관계 (Ch1-05 복습):

  현대 CPU 파이프라인 깊이: 14~20 단계 (Intel Skylake: 14~19)
  분기 명령 도달 시 CPU의 선택:
    ① 결과를 기다린다 → 파이프라인 버블 ~14~19 사이클
    ② 결과를 추측한다 (투기 실행, speculative execution)

  추측이 맞으면 (branch predicted):
    → 버블 없음, 성능 손실 없음
    → 잘 예측되는 분기: if (i < n), for 루프 조건 등

  추측이 틀리면 (branch mispredicted):
    → 파이프라인 플러시 → 14~19 사이클 손실
    → 잘못 실행한 명령어들 폐기
    → 올바른 경로로 다시 시작

데이터 의존 분기의 문제:
  if (data[i] >= 128) ...
  → data[i] 값이 i마다 다르면 예측 패턴이 없음
  → 랜덤 입력: 예측 성공률 ~50% → 절반이 미스
  → 미스 1회 × ~15 cy × N회 반복 = 큰 비용

  그 유명한 SO 질문 (Ch1-05 연결):
    정렬된 배열:   if (data[i] >= 128) → BTB 학습 완료 → 빠름
    비정렬 배열:   if (data[i] >= 128) → 랜덤 → 느림
    → 정렬 하나로 4~5배 차이 발생

branchless의 아이디어:
  "if 문을 아예 없애면 예측 실패도 없다"
  → 조건을 산술/논리 연산으로 대체
  → 항상 양쪽을 실행 (무조건 실행)
  → 결과를 조건에 따라 선택
```

---

## 😱 잘못된 이해

### Before: "if 문을 branchless로 바꾸면 항상 빠르다"

```
잘못된 일반화:
  "if 문 = 느리다, branchless = 빠르다"
  → 무조건 cmov로 교체하자!

반례 1 — 잘 예측되는 분기:
  // 거의 항상 true (데이터의 95%가 threshold 이상)
  if (sensor_reading > MIN_THRESHOLD)
      process(sensor_reading);

  분기 예측기 관점:
    → 95% 확률로 taken → BTB가 "항상 taken"으로 학습
    → 예측 성공률 ~95% → 미스율 ~5% → 사실상 비용 없음

  branchless로 바꾸면:
    → 항상 process() 결과를 계산하고 선택
    → process()가 비싼 연산이라면 5% 케이스에서 헛된 계산
    → 오히려 느려질 수 있음

반례 2 — 파이프라인 호환성:
  x86 cmov는 두 값을 읽고 하나를 선택하는 구조
  → 두 경로의 값을 모두 계산해야 함
  → 값 계산에 부작용(side effect)이 있으면 적용 불가

반례 3 — 코드 가독성:
  branchless 코드는 종종 읽기 어려움
  int abs_val = (x ^ (x >> 31)) - (x >> 31);  // 비트 트릭
  vs
  int abs_val = (x < 0) ? -x : x;             // 명확함
  → 성능 차이가 없다면 가독성이 우선

올바른 사고:
  "분기 예측 미스율이 높은가?" → 측정 먼저
  분기 미스율 > 5~10% → branchless 고려
  분기 미스율 < 1%    → 그냥 if 문이 더 나음
```

---

## ✨ 올바른 이해

### After: 분기 예측 미스율을 측정하고, 높으면 branchless를 적용하라

```
판단 기준:

  1단계: perf stat으로 분기 미스율 측정
    perf stat -e branches,branch-misses ./program
    → branch-misses / branches = 미스율
    → 미스율 > 10%이면 branchless 고려

  2단계: 해당 분기가 핫 루프 안에 있는가?
    → 100만 번 이상 실행되는 루프 안의 분기만 중요
    → 드물게 실행되는 분기는 미스율이 높아도 절대적 비용이 작음

  3단계: 해당 분기가 데이터 의존적인가?
    → if (i < n): 루프 카운터 → BTB가 학습 가능 → 미스 드뭄
    → if (data[i] > threshold): 데이터 값에 따라 → 미스 빈발

  4단계: branchless 변환 후 측정으로 비교
    → 반드시 before/after 측정 (가정으로 결정하지 말 것)

도구:
  perf stat -e branch-misses → 전체 미스
  perf record -e branch-misses -g → 핫스팟 위치
  godbolt → cmov 생성 여부 확인
  Google Benchmark → 실제 ns/iter 비교
```

---

## 🔬 내부 동작 원리

### 1. x86 `cmov` 명령의 동작 방식

```
일반 조건 분기 (jcc) vs 조건 이동 (cmov):

if (a > b) x = a;  else x = b;

--- jcc (분기) 방식 어셈블리 ---
    cmp  eax, ebx          ; a - b 비교 (플래그 설정)
    jle  .else             ; a <= b 이면 .else로 점프 (분기!)
    mov  ecx, eax          ; x = a
    jmp  .end
  .else:
    mov  ecx, ebx          ; x = b
  .end:
    ; 분기가 있으므로 BTB/분기 예측기 개입
    ; 예측 실패 시 ~15 사이클 파이프라인 플러시

--- cmov (조건 이동) 방식 ---
    cmp  eax, ebx          ; a - b 비교
    mov  ecx, ebx          ; x = b (무조건 실행)
    cmovg ecx, eax         ; if (a > b): x = a
    ; 분기 명령 없음!
    ; 두 값 모두 준비 → 조건에 따라 한쪽 선택 → 파이프라인 연속

cmov의 특성:
  - 조건 플래그(EFLAGS)를 읽어 src를 dst로 이동할지 결정
  - 플래그가 맞지 않아도 명령 자체는 완료 (no-op처럼 동작)
  - 분기 예측기를 전혀 거치지 않음
  - 단점: 두 경로 모두 값이 계산되어야 함 (양쪽 실행)

cmov 변형 (Intel 조건 코드):
  cmove   / cmovz   : 같으면 이동   (ZF=1)
  cmovne  / cmovnz  : 다르면 이동   (ZF=0)
  cmovg   / cmovnle : 큰 경우 (부호 있음)
  cmovge  / cmovnl  : 크거나 같은 경우
  cmovl   / cmovnge : 작은 경우
  cmovle  / cmovng  : 작거나 같은 경우
  cmova   / cmovnbe : 큰 경우 (부호 없음, unsigned)
  cmovb   / cmovnae : 작은 경우 (부호 없음, unsigned)
```

### 2. godbolt로 보는 branchless 컴파일

```cpp
// 예제 1: max(a, b)

// 일반 if 방식
int max_branch(int a, int b) {
    if (a > b) return a;
    return b;
}

// 명시적 branchless (컴파일러 힌트)
int max_branchless(int a, int b) {
    int result = b;
    if (a > b) result = a;  // 컴파일러가 cmov로 최적화 가능
    return result;
}
```

```asm
; godbolt.org — GCC 13.2 -O2

; max_branch — 컴파일러가 이미 cmov로 변환!
max_branch(int, int):
    cmp    edi, esi
    mov    eax, esi
    cmovg  eax, edi    ; if (edi > esi): eax = edi
    ret

; 흥미로운 사실: -O2에서 GCC/Clang은 이미 단순 max/min을 cmov로 생성
; → 컴파일러가 자동으로 branchless를 선택한 것
; 문제는 컴파일러가 branchless를 선택하지 못하는 복잡한 경우

; 예제 2: 컴파일러가 분기를 유지하는 경우
; (함수 호출이 포함된 경우, 부작용이 있는 경우)
int conditional_work(int x, int threshold) {
    if (x > threshold) {
        return expensive_compute(x);  // 비싼 함수
    }
    return 0;
}
; → expensive_compute()를 항상 호출할 수 없으므로 jcc 사용
; → branchless 불가: 한쪽 경로는 항상 실행되어선 안 됨
```

### 3. branchless 패턴 모음

```cpp
// ============================================================
// 패턴 1: 절댓값 (abs)
// ============================================================

// 표준 방식 (컴파일러가 최적화):
int abs_standard(int x) { return x < 0 ? -x : x; }

// 비트 트릭 (역사적 방법, 현대 컴파일러 불필요):
int abs_bitmask(int x) {
    int mask = x >> 31;    // 음수: 모든 비트 1 (0xFFFFFFFF), 양수: 0
    return (x ^ mask) - mask;
    // x 음수:  (~x) - (-1) = ~x + 1 = -x   (2의 보수 부정)
    // x 양수:  (x ^ 0) - 0 = x             (변화 없음)
}
// godbolt -O2: 두 버전 모두 동일한 어셈블리 생성 (sar + xor + sub)

// ============================================================
// 패턴 2: min/max
// ============================================================

// std::min/max는 이미 cmov로 컴파일됨 (-O2 기준)
int my_max(int a, int b) { return a > b ? a : b; }
// 생성 어셈블리:
//   cmp edi, esi  /  cmovle edi, esi  /  mov eax, edi  /  ret

// ============================================================
// 패턴 3: clamp (범위 제한)
// ============================================================

int clamp(int x, int lo, int hi) {
    // 두 번의 cmov:
    if (x < lo) x = lo;  // cmovl
    if (x > hi) x = hi;  // cmovg
    return x;
}
// -O2 생성:
//   cmp  edi, esi  /  cmovl  edi, esi
//   cmp  edi, edx  /  cmovg  edi, edx
//   mov  eax, edi  /  ret
// → 분기 없음

// ============================================================
// 패턴 4: 조건부 누산 (conditional accumulation)
// ============================================================

// 분기 버전 (예측 어려운 경우 느림):
long sum_branch(const int* data, int n, int threshold) {
    long sum = 0;
    for (int i = 0; i < n; i++)
        if (data[i] >= threshold) sum += data[i];  // 분기!
    return sum;
}

// branchless 버전:
long sum_branchless(const int* data, int n, int threshold) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        int val = data[i];
        int mask = -(val >= threshold);  // 조건 true: 0xFFFFFFFF, false: 0
        sum += val & mask;               // 조건 true: val, false: 0
    }
    return sum;
}
// 이 루프는 분기가 없어서 자동 벡터화 가능
// → SIMD로 8개씩 처리 가능 (분기 버전은 벡터화 어려움)
```

### 4. SIMD 마스킹 — 벡터 수준 branchless

```cpp
// SIMD에서 분기를 처리하는 방법: 마스크(mask)

// 스칼라:
int result = (a > b) ? a : b;   // cmov로 1개 처리

// AVX2로 8개 동시 처리:
#include <immintrin.h>

// 8개의 int를 동시에 max 계산
__m256i simd_max_8(const int* a, const int* b) {
    __m256i va = _mm256_loadu_si256((__m256i*)a);
    __m256i vb = _mm256_loadu_si256((__m256i*)b);
    return _mm256_max_epi32(va, vb);  // 하드웨어 지원 max (AVX2)
    // 8개 int를 동시에 비교 → 마스크 생성 → 선택 → 1 명령
}

// 조건부 누산의 SIMD 버전:
// "threshold 이상인 원소만 sum에 더하라"
long simd_conditional_sum(const int* data, int n, int threshold) {
    __m256i vsum = _mm256_setzero_si256();
    __m256i vthresh = _mm256_set1_epi32(threshold);

    int i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256i v = _mm256_loadu_si256((__m256i*)(data + i));
        // 비교: 각 레인에서 v >= threshold → 마스크 (0 또는 0xFFFFFFFF)
        __m256i mask = _mm256_cmpgt_epi32(v, _mm256_sub_epi32(vthresh,
                           _mm256_set1_epi32(1)));  // v > threshold-1
        // 마스킹: 조건 false인 원소를 0으로
        __m256i masked = _mm256_and_si256(v, mask);
        // 누산
        vsum = _mm256_add_epi32(vsum, masked);
    }
    // 수평 합으로 스칼라 환원
    // (수평 합 코드 생략)
    return 0; // 단순화
}
```

```asm
; 컴파일러 자동 벡터화된 조건부 누산 (GCC -O3 -mavx2)
; int sum_cond(int* data, int n, int thresh):
;   루프 내 vpslld, vpcmpeqd, vpand, vpaddd 등 YMM 레지스터 명령
;   → 분기 없이 8개 int를 동시에 처리
;   → 분기 버전 대비 약 4~8배 처리량 향상 (8way SIMD + 분기 제거)
```

### 5. 컴파일러가 branchless를 선택하는 조건

```
컴파일러(GCC/Clang -O2)의 branchless 선택 기준:

자동으로 cmov 생성 (branchless):
  ✅ 단순 ternary: a > b ? a : b
  ✅ 단순 min/max 패턴
  ✅ 플래그 기반 단순 대입: if (cond) x = y;
  ✅ 함수 호출 없음, 부작용 없음

branchless 생성 안 함 (jcc 유지):
  ❌ 함수 호출 포함: if (cond) f(); (f가 항상 실행되면 안 됨)
  ❌ 루프 안의 break/continue
  ❌ 예외 처리(throw)
  ❌ 컴파일러가 mispredict 비용 < compute 비용으로 판단 시

강제로 branchless 유도하는 방법:
  // 방법 1: __builtin_expect 제거 (컴파일러 힌트 제거)
  // 방법 2: 비트 마스크 패턴 사용
  // 방법 3: std::min/max 사용 (컴파일러가 cmov로 번역)
  // 방법 4: -O3 + profile feedback (PGO) → 예측 실패 케이스 감지
  
  // Clang의 경우 __builtin_unpredictable() 힌트:
  if (__builtin_unpredictable(data[i] >= threshold))
      sum += data[i];
  // "이 분기는 예측 불가" 힌트 → 컴파일러가 branchless 생성 우선
```

---

## 💻 실전 실험

### 실험 1: 정렬 vs 비정렬 배열에서 분기 비용 측정

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <algorithm>
#include <random>

const int N = 1'000'000;

static void BM_Sorted_Branch(benchmark::State& state) {
    std::vector<int> data(N);
    std::iota(data.begin(), data.end(), 0);
    // 0..999999 정렬 → 절반이 >= 500000 → 예측 가능 (경계 근처만 미스)
    for (auto _ : state) {
        long sum = 0;
        for (int x : data)
            if (x >= 500000) sum += x;
        benchmark::DoNotOptimize(sum);
    }
}

static void BM_Random_Branch(benchmark::State& state) {
    std::vector<int> data(N);
    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(0, 999999);
    for (auto& x : data) x = dist(rng);
    // 랜덤 → >= 500000 여부가 예측 불가
    for (auto _ : state) {
        long sum = 0;
        for (int x : data)
            if (x >= 500000) sum += x;
        benchmark::DoNotOptimize(sum);
    }
}

static void BM_Random_Branchless(benchmark::State& state) {
    std::vector<int> data(N);
    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(0, 999999);
    for (auto& x : data) x = dist(rng);
    for (auto _ : state) {
        long sum = 0;
        for (int x : data) {
            int mask = -(x >= 500000);  // true: 0xFFFFFFFF, false: 0
            sum += x & mask;
        }
        benchmark::DoNotOptimize(sum);
    }
}

BENCHMARK(BM_Sorted_Branch)->Unit(benchmark::kMillisecond);
BENCHMARK(BM_Random_Branch)->Unit(benchmark::kMillisecond);
BENCHMARK(BM_Random_Branchless)->Unit(benchmark::kMillisecond);
BENCHMARK_MAIN();
```

```bash
g++ -O2 -std=c++17 bench_branch.cpp -lbenchmark -lbenchmark_main -lpthread -o bench_branch
./bench_branch

# 예상 출력:
# BM_Sorted_Branch    ~2.1 ms   (정렬됨 → 예측 가능)
# BM_Random_Branch    ~8.5 ms   (랜덤 → 예측 불가)
# BM_Random_Branchless ~2.8 ms  (분기 없음 → 예측 문제 없음)
# → 랜덤 분기 vs branchless: 약 3배 차이
```

### 실험 2: perf로 분기 미스율 확인

```bash
# 각 버전의 분기 미스율 측정
perf stat -e cycles,instructions,branches,branch-misses \
    ./bench_sorted_only

perf stat -e cycles,instructions,branches,branch-misses \
    ./bench_random_only

# 예상 비교:
#                    정렬        랜덤       branchless
# branch-miss rate:  0.8%       48.2%        0.1%
# IPC:               3.1        0.9          2.4
# cycles:            ~100M      ~420M        ~140M
#
# 랜덤 분기: 48% 미스 → 절반이 파이프라인 플러시
# branchless: 미스 거의 없음 + 두 경로 모두 실행하는 약간의 추가 연산

# 핫스팟 확인
perf record -g ./bench_random_only
perf report
# → 루프 안 branch-miss가 집중되는 위치 확인
```

### 실험 3: godbolt에서 cmov 생성 여부 확인

```cpp
// https://godbolt.org — GCC 13.2 -O2

// 케이스 A: 컴파일러가 cmov 생성 (branchless)
int max_ab(int a, int b) { return a > b ? a : b; }

// 케이스 B: 컴파일러가 jcc 유지 (분기 있음)
int max_with_side_effect(int a, int b) {
    if (a > b) {
        log_access(a);   // 부작용 → 항상 실행 불가 → jcc 필요
        return a;
    }
    return b;
}

// 케이스 C: 수동 branchless로 컴파일러 유도
int max_manual(int a, int b) {
    int result = b;
    result += (a - b) & ~((a - b) >> 31);   // 비트 트릭
    return result;
    // 단 주의: overflow 가능성 있음
}
```

```bash
# 어셈블리 파일로 저장 후 비교
gcc -O2 -S -masm=intel branch_test.cpp -o branch_test.s
grep -E 'cmov|jle|jge|jg|jl' branch_test.s
# → max_ab: cmovg 확인 (branchless)
# → max_with_side_effect: jle 확인 (분기 있음)
```

### 실험 4: 잘 예측되는 분기 vs branchless 비교

```cpp
#include <benchmark/benchmark.h>

const int N = 1'000'000;

// 거의 항상 같은 경로 (90:10 비율)
static void BM_HighlyPredictable(benchmark::State& state) {
    std::vector<int> data(N);
    for (int i = 0; i < N; i++) data[i] = (i % 10 != 0) ? 100 : 5;
    // 90%: 100 (>=50 → sum 추가), 10%: 5 (<50 → 무시)
    for (auto _ : state) {
        long sum = 0;
        for (int x : data)
            if (x >= 50) sum += x;   // 90% taken → BTB 학습 완료
        benchmark::DoNotOptimize(sum);
    }
}

static void BM_HighlyPredictable_Branchless(benchmark::State& state) {
    std::vector<int> data(N);
    for (int i = 0; i < N; i++) data[i] = (i % 10 != 0) ? 100 : 5;
    for (auto _ : state) {
        long sum = 0;
        for (int x : data) {
            int mask = -(x >= 50);
            sum += x & mask;   // 항상 두 경로 실행
        }
        benchmark::DoNotOptimize(sum);
    }
}

BENCHMARK(BM_HighlyPredictable);
BENCHMARK(BM_HighlyPredictable_Branchless);
```

```bash
# 예상: 잘 예측되는 경우 branchless가 오히려 느릴 수 있음
# BM_HighlyPredictable:            ~1.8 ms  (예측 완벽 → 빠름)
# BM_HighlyPredictable_Branchless: ~2.1 ms  (항상 두 경로 → 약간 느림)
# → "branchless가 항상 빠르지 않다"는 반증
```

---

## 📊 성능 비교

```
분기 방식별 비용 비교 (x86-64, Skylake, N=1,000,000 루프):

시나리오                     방식         시간      branch-miss  IPC
────────────────────────────────────────────────────────────────────
정렬 배열, 예측 가능           if (분기)    ~2.1 ms    0.8%        3.1
정렬 배열, 예측 가능          branchless   ~2.3 ms    0.1%        2.8
랜덤 배열, 예측 불가           if (분기)    ~8.5 ms   48.2%        0.9
랜덤 배열, 예측 불가          branchless   ~2.8 ms    0.1%        2.4
90:10 비율, 고예측도           if (분기)    ~1.8 ms    1.2%        3.4
90:10 비율, 고예측도          branchless   ~2.1 ms    0.1%        2.7

핵심 관찰:
  랜덤 분기 → branchless 전환: 8.5 ms → 2.8 ms (3.0배 개선)
  정렬 분기 → branchless 전환: 2.1 ms → 2.3 ms (오히려 9% 느림)
  고예측 분기 → branchless:    1.8 ms → 2.1 ms (17% 느림)

결론: 분기 미스율이 핵심 지표
  미스율 >10%: branchless 적극 고려
  미스율  <5%: 현상 유지가 대체로 유리

cmov vs jcc 사이클 비용 (단일 명령 기준):
  cmov (예측 실패 없음): ~1 사이클 (레이턴시는 조건 코드 준비 후)
  jcc (예측 성공):        ~0~1 사이클 추가 없음
  jcc (예측 실패):        ~14~19 사이클 파이프라인 플러시
```

---

## ⚖️ 트레이드오프

```
branchless 적용 판단 기준:

branchless가 유리한 경우:
  ✅ 분기 미스율이 높은 경우 (>10%, perf로 측정)
  ✅ 데이터 의존 조건 (배열 값에 따른 분기)
  ✅ 핫 루프 안 (1,000만 번 이상 실행)
  ✅ SIMD 벡터화를 방해하는 분기 (vectorization 막는 if)
  ✅ 두 경로 모두 계산 비용이 비슷한 경우

branchless가 불리한 경우:
  ❌ 잘 예측되는 분기 (미스율 <2%)
  ❌ 한쪽 경로가 비싼 연산 (예측 성공 시 그 경로만 실행하면 됨)
  ❌ 함수 호출, 부작용이 있는 경우 (cmov 적용 불가)
  ❌ 코드 가독성이 중요한 경우 (비트 트릭은 이해하기 어려움)

코드 가독성 vs 성능:
  나쁜 예:
    int abs_val = (x ^ (x >> 31)) - (x >> 31);  // 비트 트릭
  좋은 예:
    int abs_val = std::abs(x);  // 컴파일러가 최적화 (동일한 어셈블리)

  → 표준 라이브러리 함수를 먼저 사용하라
  → 컴파일러가 이미 branchless로 번역한다

컴파일러 힌트 활용:
  [[likely]], [[unlikely]] (C++20):
    if (x > 0) [[likely]] { ... }
    → 컴파일러에 분기 확률 힌트 → 코드 배치 최적화 (cmov는 아님)

  __builtin_expect (GCC/Clang):
    if (__builtin_expect(x > 0, 1)) { ... }  // 거의 항상 true
    → 핫 경로를 코드의 직선 경로에 배치

  PGO(Profile-Guided Optimization):
    → 실제 실행 데이터로 분기 확률 자동 추정
    → 컴파일러가 최적의 branchless/jcc 선택

SIMD 마스킹 활용:
  루프 안 분기가 SIMD 자동 벡터화를 막는다면:
    → 분기를 마스크 연산으로 교체 → 벡터화 가능
    → 8배(AVX2) 처리량 향상 가능
    → branchless 전환 이유 중 가장 큰 이득
```

---

## 📌 핵심 정리

```
분기 vs branchless 핵심:

분기 예측 비용 (Ch1-05 연결):
  예측 성공: 비용 없음
  예측 실패: ~14~19 사이클 파이프라인 플러시
  데이터 의존 분기: 입력에 따라 미스율 50%까지 도달 가능

x86 cmov (조건 이동):
  "항상 두 값을 계산하고, 조건에 따라 하나 선택"
  → 분기 명령 없음 → 파이프라인 플러시 없음
  → -O2에서 컴파일러가 자동으로 단순 패턴에 적용

branchless 패턴:
  abs(x):   (x ^ (x >> 31)) - (x >> 31)  또는 std::abs(x)
  max(a,b): a > b ? a : b  → cmovg
  min(a,b): a < b ? a : b  → cmovl
  clamp:    두 번의 cmov (min/max 조합)
  조건 누산: mask = -(cond); sum += val & mask;

SIMD 마스킹:
  벡터 비교 → 마스크 생성 → AND → 분기 없이 8개 동시 처리
  분기 제거 + SIMD = 최대 8~16배 개선 (경우에 따라)

branchless가 항상 빠른 것은 아님:
  미스율 낮음(<5%): 분기 유지가 유리 (한쪽만 실행)
  미스율 높음(>10%): branchless 전환 효과 있음
  → perf stat -e branch-misses 로 먼저 측정하라

판단 순서:
  1. perf로 branch-miss rate 측정
  2. 핫 루프 내 고미스율 분기인지 확인
  3. 데이터 의존 분기인지 확인
  4. branchless 전환 후 before/after 측정으로 검증
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 두 함수를 GCC `-O2`로 컴파일했을 때, 어떤 어셈블리 차이가 발생하는가? godbolt에서 직접 확인하고 그 이유를 설명하라.

```cpp
// A
int clamp_a(int x, int lo, int hi) {
    if (x < lo) return lo;
    if (x > hi) return hi;
    return x;
}

// B
int clamp_b(int x, int lo, int hi) {
    return std::max(lo, std::min(x, hi));
}
```

<details>
<summary>해설 보기</summary>

두 버전 모두 GCC `-O2`에서 **동일한 또는 매우 유사한 branchless 어셈블리**를 생성합니다.

`clamp_a`의 경우 컴파일러는 두 번의 조건 분기를 두 번의 `cmov`로 변환합니다:
```asm
clamp_a:
    cmp  edi, esi      ; x vs lo
    cmovl edi, esi     ; x = max(x, lo)
    cmp  edi, edx      ; x vs hi
    cmovg edi, edx     ; x = min(x, hi)
    mov  eax, edi
    ret
```

`clamp_b`도 `std::max`와 `std::min`이 내부적으로 ternary로 구현되어 있어 동일한 패턴으로 컴파일됩니다.

핵심 통찰: 컴파일러는 단순 조건 대입 패턴(`if (cond) x = y;`)을 자동으로 `cmov`로 변환합니다. 개발자가 "branchless처럼 보이게" 코드를 작성하지 않아도 됩니다. 중요한 것은 **부작용 없는 단순 값 선택 패턴**을 유지하는 것입니다. `std::min/max` 같은 표준 함수를 사용하면 의도가 명확하고, 컴파일러도 잘 최적화합니다.

</details>

---

**Q2.** 다음 루프에서 분기를 제거하면 왜 SIMD 자동 벡터화가 가능해지는지 설명하라. 그리고 `mask = -(x >= threshold)` 표현식이 어떻게 0 또는 -1(0xFFFFFFFF)을 생성하는지 설명하라.

```cpp
// 분기 버전 (벡터화 불가)
for (int i = 0; i < n; i++)
    if (data[i] >= threshold) sum += data[i];

// branchless 버전 (벡터화 가능)
for (int i = 0; i < n; i++) {
    int mask = -(data[i] >= threshold);
    sum += data[i] & mask;
}
```

<details>
<summary>해설 보기</summary>

**SIMD 벡터화와 분기의 충돌**:

SIMD(AVX2 등)는 8개의 정수를 동시에 처리하는 레지스터(`ymm`)를 사용합니다. 루프 안에 `if`가 있으면 8개의 원소가 각각 다른 경로를 취할 수 있어 SIMD가 "이 8개를 동시에 처리할 수 없다"고 판단합니다. 컴파일러는 벡터화를 포기하고 스칼라 루프를 생성합니다.

`mask = -(data[i] >= threshold)` 분석:
- `data[i] >= threshold`: C++의 비교 연산자 결과는 `true`(1) 또는 `false`(0)입니다.
- `-(true)` = `-1` = `0xFFFFFFFF` (2의 보수에서 -1의 비트 표현)
- `-(false)` = `-0` = `0x00000000`

따라서 `mask`는 조건 true 시 모든 비트 1, false 시 모든 비트 0이 됩니다.

`data[i] & mask`의 결과:
- 조건 true: `data[i] & 0xFFFFFFFF = data[i]`
- 조건 false: `data[i] & 0x00000000 = 0`

SIMD 관점: 비교(`vpcmpgtd`), AND(`vpand`), 덧셈(`vpaddd`)은 모두 8개 원소에 균일하게 적용할 수 있는 연산입니다. 분기가 없으므로 컴파일러가 `ymm` 레지스터를 사용해 8개씩 처리할 수 있습니다. 결과적으로 순수 branchless 전환만으로 8배 SIMD 처리량이 추가됩니다.

</details>

---

**Q3.** 아래 코드에서 `-O2` 컴파일 시 `process(x)`가 함수 호출을 포함한다면 branchless(cmov)가 적용될 수 없는 이유를 설명하고, 이 경우의 최적화 전략을 제시하라.

```cpp
int result = 0;
for (int x : data) {
    if (x > threshold)
        result += process(x);   // process()는 외부 함수
}
```

<details>
<summary>해설 보기</summary>

`cmov` 명령은 "이미 계산된 두 값 중 하나를 선택"하는 명령입니다. `process(x)`를 branchless로 처리하려면 **조건과 무관하게 항상 `process(x)`를 실행한 후** 그 결과를 선택해야 합니다.

그러나 `process(x)`가 외부 함수라면:
1. **부작용(side effect) 가능성**: 파일 쓰기, 전역 변수 수정 등 — 항상 실행하면 동작이 달라집니다.
2. **비용 문제**: `process(x)`가 비싼 연산이라면 불필요한 실행이 오히려 손해입니다.
3. **컴파일러의 보수적 선택**: 함수 내부를 알 수 없으면(다른 TU) 컴파일러는 안전하게 `jcc`를 사용합니다.

최적화 전략:

1. **`process(x)`를 인라이닝**: 함수 본체가 작고 부작용이 없다면 `__attribute__((always_inline))`이나 LTO로 인라이닝 → 컴파일러가 cmov 판단 가능

2. **데이터를 미리 필터링**: `std::partition`이나 조건에 맞는 인덱스를 먼저 수집 → `process()`를 항상 taken 경로에서 실행 → 분기 예측 성공률 100%
   ```cpp
   std::vector<int> filtered;
   std::copy_if(data.begin(), data.end(), std::back_inserter(filtered),
                [&](int x){ return x > threshold; });
   for (int x : filtered) result += process(x);
   ```

3. **분기 확률 힌트**: `[[likely]]`/`[[unlikely]]` → 핫 경로를 직선 코드에 배치 → `jcc` 자체는 남지만 브랜치 레이아웃 최적화

</details>

---

<div align="center">

**[⬅️ 이전: 포인터 추적](./03-pointer-chasing.md)** | **[홈으로 🏠](../README.md)** | **[다음: 측정으로 증명 ➡️](./05-proof-by-measurement.md)**

</div>
