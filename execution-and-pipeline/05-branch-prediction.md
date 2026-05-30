# 분기 예측 — 파이프라인을 비우지 않으려는 투기 실행

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 분기 예측이 없으면 파이프라인에서 얼마나 많은 사이클을 낭비하는가?
- 2-bit 포화 카운터(Saturating Counter)는 어떤 원리로 예측하는가?
- BTB(Branch Target Buffer)는 무엇이고 직접 분기와 간접 분기에서 어떻게 다르게 작동하는가?
- 정렬된 배열과 랜덤 배열에서 `if (data[i] >= 128) sum += data[i];`의 실행 속도가 다른 이유는?
- Spectre 취약점은 투기 실행의 어떤 특성에서 비롯되는가?
- `perf stat`의 `branch-misses / branches` 비율로 무엇을 진단할 수 있는가?

---

## 🔍 왜 이 개념이 중요한가

### "정렬 하나로 5배 빠른 코드 — 이게 다 분기 예측 때문이다"

```
Stack Overflow 역사상 가장 많은 추천을 받은 질문:
  "왜 정렬된 배열을 처리하는 게 정렬 안 된 것보다 빠른가?"
  https://stackoverflow.com/questions/11227809

  // 이 코드:
  for (int i = 0; i < 100000; i++) {
      if (data[i] >= 128) sum += data[i];
  }

  정렬된 배열:   ~0.8초
  랜덤 배열:     ~4.0초
  → 5배 차이, 같은 O(N) 코드, 같은 하드웨어!

이것을 이해하지 못하면:
  "캐시 미스인가? 메모리 대역폭인가? 컴파일러 최적화?"
  무작위로 원인을 추측

이것을 이해하면:
  "랜덤 배열은 분기 방향이 무작위 → 예측기 적중률 ~50%
   매 미스마다 ~15 사이클 파이프라인 플러시
   100,000번 반복 × 50% 미스 × 15 사이클 = 750,000 사이클 낭비"
  단 한 줄로 진단 가능:
    perf stat -e branches,branch-misses ./a.out

연결:
  분기 예측 이해 → 비순차 실행의 투기적 실행 (Ch1-04)
  분기 예측 이해 → Spectre 취약점의 근본 원인
  분기 예측 이해 → branchless 코드의 필요성 (Ch4-04)
```

---

## 😱 잘못된 이해

### Before: "CPU가 분기를 만나면 결과가 나올 때까지 기다린다"

```
잘못된 모델:
  cmp eax, 128
  jl  .skip        ← "CPU가 여기서 멈춰 결과를 기다린다"
  add rbx, rax
.skip:
  ...

  → 분기마다 3~5 사이클 대기 (파이프라인 깊이만큼)
  → 코드의 20%가 분기라면 전체 성능의 20% × 3 사이클 = 60% 낭비

현실:
  CPU는 멈추지 않는다.
  분기 예측기(Branch Predictor)가 "아마 jl는 not-taken일 것"이라고 예측
  예측 방향으로 즉시 다음 명령어를 Fetch/Decode/Execute 시작
  분기 결과 확정 시:
    예측 맞음  → 이미 실행 중인 명령어 그대로 계속 (낭비 없음!)
    예측 틀림  → 파이프라인 플러시 + 재시작 (~15 사이클 낭비)

  → 예측이 잘 맞으면 분기 비용 ≈ 0
  → 예측이 잘 틀리면 분기 비용 = ~15 사이클
  → 예측기의 정확도가 성능을 결정한다
```

---

## ✨ 올바른 이해

### After: 분기 예측기는 확률 기반 패턴 학습 기계

```
분기 예측기의 목표:
  분기 명령어를 만났을 때
    1. 방향(Direction): taken(분기함) vs not-taken(계속 진행)
    2. 목적지(Target): 분기 시 어느 주소로 점프하는가
  두 가지를 Execute 단계 전에 미리 맞춰야 함

예측기 정확도가 성능에 미치는 영향:
  현대 CPU의 분기 예측 적중률: 95~99% (잘 예측되는 코드)
  분기 미스 패널티: ~15 사이클 (Intel Skylake 기준)

  분기 빈도 10% (100 명령어 중 10개가 분기):
    예측 적중률 99%: 손실 = 10 × 0.01 × 15 = 1.5 사이클/100명령어
    예측 적중률 50%: 손실 = 10 × 0.50 × 15 = 75 사이클/100명령어
    → 같은 코드, 예측 적중률만 다른데 50배 차이!

  이것이 정렬 배열 vs 랜덤 배열의 5배 차이를 만드는 메커니즘
```

---

## 🔬 내부 동작 원리

### 1. 2-bit 포화 카운터(Saturating Counter): 방향 예측의 기본

```
1-bit 예측기의 문제:
  마지막 방향을 기억 (T=taken, N=not-taken)
  루프: T T T T T N T T T T T N ...
  → 루프 마지막 반복에서 항상 틀림 (T→N, N→T 전환)
  → 적중률 ≈ 99% (루프 10000번이면 2번 틀림)
  → 하지만 중첩 루프에서 내부 루프 끝 후 상태가 뒤바뀜

2-bit 포화 카운터:
  상태 4가지:
    00 = 강하게 not-taken (SN, Strongly Not-Taken)
    01 = 약하게 not-taken (WN, Weakly Not-Taken)
    10 = 약하게 taken     (WT, Weakly Taken)
    11 = 강하게 taken     (ST, Strongly Taken)

  상태 전이:
       분기 taken
    ←──────────────→
  SN  WN  WT  ST
    ←──────────────→
       분기 not-taken

  예측 규칙:
    상태 10 또는 11 → "taken" 예측
    상태 00 또는 01 → "not-taken" 예측

  루프 예시 (10번 반복):
    초기 상태: ST (11)
    반복 1~9 (taken): ST → ST → ... → ST  예측: taken ✅×9
    반복 10  (not-taken): ST → WT         예측: taken ❌ (미스 1회)
    다음 실행:
    반복 1 (taken): WT → ST               예측: taken ✅
    ...
    → 루프 10번 중 1번만 틀림, 적중률 90%
    → 1-bit보다 안정적 (중첩 루프에서도)

2-bit 카운터의 한계:
  예측 컨텍스트가 없음 → 패턴을 인식하지 못함
  예: T N T N T N ... → 항상 50% 적중
  예: T T N T T N ... → 이전 2개 기록이 있으면 예측 가능하지만 2-bit는 불가
```

### 2. 히스토리 기반 예측기: 패턴 인식

```
Global History Register (GHR):
  최근 N개 분기 방향을 비트열로 기록
  예: N=8 → "TTNTTNN T" = 11011001 (마지막 8개 분기 방향)

2-level Adaptive Predictor (Yeh & Patt, 1992):
  GHR(N비트) × PHT (Pattern History Table, 2^N 엔트리)

  예측 과정:
    GHR = 현재까지의 N개 분기 역사
    PHT[GHR] = 2-bit 포화 카운터
    → "이 역사 패턴 다음에는 어떻게 됐나"를 학습

  예시: T T N 패턴이 반복될 때
    GHR = 110 (최근 3개: T, T, N 중 현재 2개가 TT)
    PHT[110] = ST (강하게 taken) → 다음은 T 예측
    GHR = 100 (TN 후)
    PHT[100] = SN → 다음은 N 예측
    → 패턴 T T N을 학습하여 100% 적중 가능

현대 예측기 (Tournament, TAGE):
  TAGE (Tagged Geometric History Length Predictor, Seznec 2006):
    여러 길이의 GHR (4, 8, 16, 32, 64비트) 동시 사용
    더 긴 역사가 최신 예측을 담당 (hit-miss tournament)
  Intel: Neural Branch Predictor (Alder Lake 이후, 세부 비공개)
  AMD Zen3+: 세부 비공개, ~98~99% 적중률 달성
```

### 3. BTB (Branch Target Buffer): 분기 목적지 예측

```
방향 예측 vs 목적지 예측:
  조건부 분기 (jne, jl, je 등):
    방향 예측 (taken / not-taken)이 핵심
    taken이면 목적지 주소도 필요 → BTB 참조

  간접 분기 (jmp rax, call rax):
    방향은 항상 taken
    목적지 주소가 레지스터에 → 실행 전까지 모름
    → BTB가 없으면 항상 미스!

BTB 구조:
  ┌─────────────────────────────────────────────┐
  │  Branch Target Buffer (BTB)                 │
  │  Key: 분기 명령어의 PC 주소                 │
  │  Value: 마지막에 점프한 목적지 주소          │
  │                                              │
  │  PC[0x401020] → 0x401150   (마지막 목적지)  │
  │  PC[0x401035] → 0x401060   (마지막 목적지)  │
  │  PC[0x401080] → 0x402000   (마지막 목적지)  │
  └─────────────────────────────────────────────┘
  → Fetch 단계에서 PC로 BTB 조회 → 목적지 예측
  → 목적지로 즉시 Fetch 시작 (분기 결과 기다리지 않음)

간접 분기의 어려움 (가상 함수, 함수 포인터):
  // C++ 가상 함수 호출
  obj->method();  →  jmp qword ptr [rax + vtable_offset]
  
  목적지 주소가 매번 다를 수 있음 (다형성)
  BTB는 마지막 목적지만 기억
  여러 다형 객체가 섞이면 BTB 적중률 저하
  → 가상 함수 비용의 한 원인 (Ch4-02에서 상세)

Return Address Stack (RAS):
  함수 호출(call)의 반환 주소를 CPU 내부 스택에 캐시
  ret 명령어 시 RAS에서 목적지 예측
  깊은 함수 호출 스택에서 BTB 대신 RAS로 정확하게 예측
```

### 4. 투기적 실행(Speculative Execution)과 파이프라인 플러시

```
투기 실행 전체 흐름:

  ① 분기 명령어 Decode
  ② 예측기: "taken, 목적지 0x401150" 예측
  ③ 0x401150부터 Fetch/Decode/Execute 시작 (투기적 실행)
  ④ 분기 명령어 Execute 완료 → 실제 결과 확정
     Case A: 예측 맞음 (0x401150으로 분기)
             → 투기적으로 실행 중인 명령어 그대로 계속 ✅
             → 낭비 없음!
     Case B: 예측 틀림 (실제 목적지 0x401200)
             → 투기적으로 실행한 명령어들 ROB에서 전부 폐기
             → 0x401200부터 다시 Fetch 시작
             → ~15 사이클 낭비 (파이프라인 깊이만큼)

파이프라인 플러시 비용 계산:
  Fetch → Decode → Rename → Dispatch → Execute:
  분기 결과 확정까지 ~15개 단계 = ~15 사이클 낭비
  (파이프라인이 깊을수록 더 많은 사이클 낭비)

투기 실행의 부작용 — Spectre (2018):
  투기적으로 실행된 명령어가 캐시를 변경함
  ROB 폐기 시 레지스터는 복원되지만 캐시 상태는 복원되지 않음
  → 캐시 타이밍 측정으로 투기적으로 읽힌 데이터 추론 가능
  
  // Spectre 패턴:
  if (untrusted_index < size) {
      // 예측기가 "true"로 예측 → 투기적 실행
      char secret = secret_array[untrusted_index]; // 투기적 비밀 로드
      dummy = probe_array[secret * 64];             // 캐시 오염
  }
  // 예측 틀림 → 폐기, 하지만 캐시는 이미 오염됨
  
  완화책:
    IBRS/IBPB: 커널/유저 모드 전환 시 BTB 초기화
    Retpoline: 간접 분기를 예측기가 무력화되는 패턴으로 교체
    → 성능 5~30% 저하 (워크로드 의존)
```

### 5. 정렬된 배열 vs 랜덤 배열: 예측기 해부

```c
// 유명한 예제: 배열 조건부 합산
for (int i = 0; i < N; i++) {
    if (data[i] >= 128) sum += data[i];
}
```

```
정렬된 배열 [0, 1, 2, ..., 127, 128, 129, ..., 255]:
  data[i] < 128 구간: 반복적으로 not-taken
  data[i] >= 128 구간: 반복적으로 taken
  
  분기 패턴:
    N N N N ... N N T T T T ... T T
    (128번 N) (128번 T)
  
  예측기 적중률: ~99% (구간이 길어 2-bit 카운터가 완벽 추적)
  perf 측정: branch-misses ≈ 2 (N→T 전환 1번, T→N 전환 1번)

랜덤 배열 [199, 40, 182, 72, 238, 15, ...]:
  data[i] >= 128은 무작위로 ~50% 확률
  
  분기 패턴:
    T N T T N N T N T N T N ...
  
  예측기 적중률: ~50% (랜덤 패턴 = 학습 불가)
  perf 측정: branch-misses ≈ N/2 = 50,000 misses (N=100,000)
  
  비용: 50,000 misses × 15 사이클 = 750,000 사이클 낭비
  (3GHz CPU에서 0.25ms 낭비)

perf로 확인:
  정렬된 배열:  branch-miss rate ≈ 0.002%
  랜덤 배열:    branch-miss rate ≈ 50%

해결책:
  1. 정렬 후 처리 (sort-then-filter):
     sort(data, data + N)
     for (...) if (data[i] >= 128) sum += data[i];
     → 분기 예측 최적화

  2. Branchless (예측기 필요 없음):
     for (...) sum += data[i] & -(data[i] >= 128);
     // 분기 없는 마스킹 → 예측 미스 0
     (Ch4-04에서 상세)
```

### 6. 분기 미스 패널티의 실제 측정

```
분기 예측 미스의 정확한 비용:

  분기 미스 1회:
    → 파이프라인에서 투기적으로 실행 중인 명령어 폐기
    → 올바른 주소에서 다시 Fetch 시작
    → 소요 시간: "파이프라인 깊이" 사이클

  Intel Skylake 기준:
    파이프라인 깊이 ~14~20단계
    분기 미스 패널티 ~15~20 사이클

  전체 프로그램에서의 영향:
    branches 수 × branch-miss-rate × 패널티 사이클 = 총 낭비 사이클
    
    예: 10억 분기 × 5% 미스율 × 15 사이클
       = 7.5억 사이클 낭비
       3GHz에서 = 0.25초
       전체 실행 시간이 1초라면 → 25% 성능 손실

분기 미스 역산 (perf로 파이프라인 깊이 추정):
  perf stat -e branches,branch-misses,cycles ./a.out
  
  대략적 패널티 = 
    (cycles 중 분기 미스로 낭비된 cycles) / (branch-misses 수)
  
  더 정확한 방법: CPU 전용 perf 이벤트
  perf stat -e br_misp_retired.all_branches,machine_clears.count ./a.out
```

---

## 💻 실전 실험

### 실험 1: 정렬 배열 vs 랜덤 배열 분기 미스 직접 측정

```c
// branch_prediction_demo.c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#define N 100000

// 배열 셔플 (Fisher-Yates)
void shuffle(int *arr, int n) {
    for (int i = n - 1; i > 0; i--) {
        int j = rand() % (i + 1);
        int tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
    }
}

int compare_int(const void *a, const void *b) {
    return (*(int*)a - *(int*)b);
}

long test_branch(int *data, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        if (data[i] >= 128) sum += data[i];
    }
    return sum;
}

int main(void) {
    srand(42);
    int data[N];
    for (int i = 0; i < N; i++) data[i] = rand() & 0xFF;  // 0~255

    // 정렬된 버전
    int sorted[N];
    memcpy(sorted, data, sizeof(data));
    qsort(sorted, N, sizeof(int), compare_int);

    // 워밍업
    volatile long dummy = test_branch(sorted, N);
    dummy = test_branch(data, N);

    struct timespec t1, t2;
    const int REPS = 10000;

    // 정렬된 배열 측정
    clock_gettime(CLOCK_MONOTONIC, &t1);
    long sum_sorted = 0;
    for (int r = 0; r < REPS; r++) sum_sorted += test_branch(sorted, N);
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_sorted = ((t2.tv_sec-t1.tv_sec)*1e3 + (t2.tv_nsec-t1.tv_nsec)*1e-6) / REPS;

    // 랜덤 배열 측정
    clock_gettime(CLOCK_MONOTONIC, &t1);
    long sum_random = 0;
    for (int r = 0; r < REPS; r++) sum_random += test_branch(data, N);
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_random = ((t2.tv_sec-t1.tv_sec)*1e3 + (t2.tv_nsec-t1.tv_nsec)*1e-6) / REPS;

    printf("결과: %ld / %ld (최적화 방지)\n", sum_sorted, sum_random);
    printf("정렬된 배열:  %.3f ms\n", ms_sorted);
    printf("랜덤 배열:    %.3f ms\n", ms_random);
    printf("속도비:       %.1fx\n", ms_random / ms_sorted);

    return 0;
}
```

```bash
gcc -O2 -o branch_demo branch_prediction_demo.c
./branch_demo

# 예상 출력:
# 정렬된 배열:  ~0.08 ms
# 랜덤 배열:    ~0.40 ms
# 속도비:       ~5x

# perf로 분기 미스 측정
perf stat -e branches,branch-misses \
          -e cycles,instructions \
          ./branch_demo

# 두 버전을 분리해서 측정하려면 각각 별도 바이너리로 컴파일:
# sorted_only.c (sorted 배열만 테스트)
# random_only.c (random 배열만 테스트)
# perf stat -e branch-misses ./sorted_only
# perf stat -e branch-misses ./random_only

# 예상:
# sorted: branch-misses ≈ 0.002%
# random: branch-misses ≈ 49~50%
```

### 실험 2: Branchless로 분기 미스 제거

```c
// branchless_demo.c
#include <stdio.h>
#include <time.h>

#define N 100000

// 분기 버전
long with_branch(int *data, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        if (data[i] >= 128) sum += data[i];
    }
    return sum;
}

// Branchless 버전: cmov 유도
long branchless(int *data, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        // 조건이 참이면 data[i], 거짓이면 0 (분기 없음)
        sum += (data[i] >= 128) ? data[i] : 0;
        // 또는: sum += data[i] & -(data[i] >= 128);
    }
    return sum;
}

int main(void) {
    int data[N];
    unsigned int seed = 42;
    for (int i = 0; i < N; i++) {
        seed ^= seed << 13; seed ^= seed >> 17; seed ^= seed << 5;
        data[i] = seed & 0xFF;
    }

    struct timespec t1, t2;
    const int REPS = 10000;
    volatile long dummy;

    clock_gettime(CLOCK_MONOTONIC, &t1);
    for (int r = 0; r < REPS; r++) dummy = with_branch(data, N);
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_branch = ((t2.tv_sec-t1.tv_sec)*1e3+(t2.tv_nsec-t1.tv_nsec)*1e-6)/REPS;

    clock_gettime(CLOCK_MONOTONIC, &t1);
    for (int r = 0; r < REPS; r++) dummy = branchless(data, N);
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_bl = ((t2.tv_sec-t1.tv_sec)*1e3+(t2.tv_nsec-t1.tv_nsec)*1e-6)/REPS;

    printf("분기 버전:      %.3f ms\n", ms_branch);
    printf("Branchless:     %.3f ms\n", ms_bl);
    printf("속도비(랜덤 입력): %.1fx\n", ms_branch / ms_bl);

    return 0;
}
```

```bash
gcc -O2 -o branchless_demo branchless_demo.c

# 어셈블리 확인 (cmov 또는 setge + and 패턴)
gcc -O2 -S -masm=intel branchless_demo.c -o branchless_demo.s
grep -E "cmov|setge|test|jl|jge" branchless_demo.s
# branchless 함수에서 jl/jge 대신 cmov 또는 산술 마스킹이 나와야 함

perf stat -e branches,branch-misses ./branchless_demo
# branchless 버전: branch-misses ≈ 0 (분기 자체가 없으므로)

# 주의: branchless가 항상 빠른 건 아님
# 잘 예측되는 분기(정렬된 배열)에서는 branch 버전이 더 빠를 수 있음
# → Ch4-04에서 트레이드오프 상세
```

### 실험 3: BTB 오염과 간접 분기 비용

```c
// indirect_branch.c: 가상 함수 / 함수 포인터의 분기 예측
#include <stdio.h>
#include <time.h>

typedef long (*func_t)(long);

long add_one(long x)  { return x + 1; }
long add_two(long x)  { return x + 2; }
long add_three(long x){ return x + 3; }

#define N 100000000L

int main(void) {
    struct timespec t1, t2;

    // 단형(monomorphic): 항상 같은 함수 → BTB가 쉽게 예측
    func_t mono_fn = add_one;
    long r = 0;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    for (long i = 0; i < N; i++) r = mono_fn(r);
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_mono = ((t2.tv_sec-t1.tv_sec)*1e3+(t2.tv_nsec-t1.tv_nsec)*1e-6);

    // 다형(polymorphic): 3개 함수가 순환 → BTB가 패턴 학습
    func_t poly_fns[3] = {add_one, add_two, add_three};
    r = 0;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    for (long i = 0; i < N; i++) r = poly_fns[i % 3](r);
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_poly = ((t2.tv_sec-t1.tv_sec)*1e3+(t2.tv_nsec-t1.tv_nsec)*1e-6);

    printf("r=%ld (최적화 방지)\n", r);
    printf("단형 호출:  %.1f ms\n", ms_mono);
    printf("다형 호출:  %.1f ms\n", ms_poly);
    printf("차이:       %.1fx\n", ms_poly / ms_mono);
    return 0;
}
```

```bash
gcc -O1 -o indirect_branch indirect_branch.c
# -O1: 인라이닝 방지 (O2는 함수 포인터를 직접 호출로 최적화할 수 있음)

perf stat -e branches,branch-misses \
          -e instructions,cycles \
          ./indirect_branch

# 단형: branch-misses ≈ 0% (BTB가 완벽 예측)
# 다형 순환: BTB가 학습하면 적중률 높아짐
# 진짜 랜덤 다형: branch-misses 높음 → 성능 저하
```

---

## 📊 성능 비교

```
분기 예측 시나리오별 비용 (Intel Skylake, 3 GHz 기준):

시나리오                  branch-miss율   사이클 손실/분기   설명
──────────────────────────────────────────────────────────────────
루프 카운터 (항상 taken)   0.01%          0.002 cy          완벽 예측
정렬된 배열 임계값         0.01%          0.002 cy          구간 패턴 학습
순환 패턴 T T N            1~2%           0.2~0.3 cy        TAGE 예측기 학습
랜덤 50/50 분기            50%            7.5 cy            예측 불가
랜덤 간접 분기(함수 포인터) 10~30%         2~5 cy            BTB 오염

실측 비교 (N=100,000 배열 합산):
                   실행 시간    branch-misses    IPC
정렬된 배열         0.08 ms      ~200            3.5
랜덤 배열           0.40 ms      ~50,000         1.2
Branchless          0.12 ms      ~0              2.8

주석:
  branchless는 cmov가 추가 계산을 수반 → 정렬 배열보다 느림
  branchless는 랜덤 배열보다 3.3배 빠름
  → "예측 불가 분기가 있을 때만 branchless 적용"

분기 미스 패널티 측정:
  perf stat 출력에서:
    total cycles     = C
    branch-misses    = M
    
  대략적 패널티 = C_with_misses - C_without_misses (이론)
  실제 측정: 두 버전(예측 가능 vs 불가능) 비교
  
  또는 Intel PEBS (Precise Event-Based Sampling):
    perf record -e br_misp_retired.all_branches:ppp ./a.out
    perf report → 정확히 어떤 분기에서 미스가 많은지 확인
```

---

## ⚖️ 트레이드오프

```
분기 예측의 이득과 비용:
  ✅ 예측 적중 시: 분기 비용 ≈ 0 (최대 IPC 유지)
  ✅ 현대 예측기: 95~99% 적중률 → 대부분의 코드에서 효과적
  ❌ 예측 실패 시: ~15 사이클 낭비
  ❌ 예측기 구조 자체: 수십~수백 KB의 칩 면적 + 전력 소비
  ❌ 보안 취약점 (Spectre, BHI): 예측기 훈련 공격 가능

분기 예측 vs Branchless:
  분기 예측 유리:
    ✅ 분기가 항상 같은 방향 (루프 조건, 초기화 체크)
    ✅ 패턴이 규칙적 (정렬된 배열)
    ❌ 랜덤 데이터 의존 분기 → branchless가 나음

  Branchless 유리:
    ✅ 데이터가 무작위로 분기 방향을 결정할 때
    ✅ 간단한 min/max, abs, clamp 연산
    ❌ 복잡한 조건 → 코드 가독성 저하
    ❌ 잘 예측되는 분기에 적용 시 오히려 느려짐
       (불필요한 계산이 추가되기 때문)

정렬 전략의 트레이드오프:
  정렬 비용: O(N log N)
  분기 예측 이득: O(N) 루프에서 미스 0으로 감소
  
  손익분기점:
    루프를 K번 반복 실행한다면:
    K × (랜덤 시간 - 정렬 시간) > 정렬 비용
    K가 클수록 정렬 투자가 값어치함 (DB 쿼리 최적화의 핵심)

Spectre 완화의 성능 비용:
  Retpoline: 간접 분기 ~10~30% 느려짐
  IBRS: 컨텍스트 스위치 비용 증가
  → 보안과 성능의 직접적인 트레이드오프

  워크로드별 영향:
    유저 공간 집약적 코드: Spectre 완화 영향 적음 (~1~5%)
    syscall 집약적 코드: Retpoline 영향 큼 (~5~30%)
    가상화 환경: Spectre/Meltdown 완화 합산 ~30% 영향 가능

Ch3 메모리 배리어와의 연결:
  분기 예측 + 투기 실행이 Spectre의 근본
  Ch3-04에서 lfence가 투기 실행을 막는 방법 설명
  (lfence는 이전 모든 명령어 완료 후 다음 Fetch 허용)
```

---

## 📌 핵심 정리

```
분기 예측 핵심:

왜 필요한가:
  분기 결과는 Execute 단계에서야 확정 (Fetch보다 수 사이클 늦음)
  기다리면 매 분기마다 파이프라인 단계 수만큼 낭비
  → 예측으로 미리 실행, 틀리면 폐기하는 투기 실행

2-bit 포화 카운터:
  ST/WT(taken 예측) vs WN/SN(not-taken 예측) 4상태
  한 번 틀려도 예측 바뀌지 않는 히스테리시스
  루프 패턴에 효과적

역사 기반 예측기 (TAGE 등):
  최근 N개 분기 방향 패턴으로 다음 예측
  T T N T T N 같은 복잡한 패턴도 학습
  현대 CPU: ~98~99% 적중률

BTB (Branch Target Buffer):
  분기 명령어 PC → 목적지 주소 캐시
  간접 분기(가상 함수, 함수 포인터) 예측에 필수

분기 미스 비용:
  ~15 사이클 파이프라인 플러시 + 재시작
  랜덤 배열 50% 미스 → 정렬 배열 대비 5배 느림

Spectre:
  투기 실행이 캐시를 오염 → ROB 롤백 후에도 잔적
  → 타이밍 공격으로 비밀 데이터 추론 가능
  완화: Retpoline, IBRS, lfence (성능 비용 수반)

진단:
  perf stat -e branches,branch-misses ./a.out
  branch-misses / branches > 5% → 분기 예측이 성능 병목
  해결: 정렬, branchless, 알고리즘 변경
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 두 코드 중 어느 것이 더 높은 `branch-misses`를 보이며 왜 그런가? `perf stat -e branch-misses`로 측정할 때 예상 수치를 추정하라.

```c
#define N 10000000

// 코드 A: 루프 카운터 기반 분기
long loop_counter(void) {
    long sum = 0;
    for (int i = 0; i < N; i++) sum += i;
    return sum;
}

// 코드 B: 데이터 기반 분기 (랜덤 입력)
long data_dependent(int *arr) {
    long sum = 0;
    for (int i = 0; i < N; i++) {
        if (arr[i] & 1) sum += arr[i];  // 홀수만 합산
    }
    return sum;
}
```

<details>
<summary>해설 보기</summary>

**코드 A (루프 카운터)**가 압도적으로 낮은 branch-misses를 가집니다.

**코드 A 분석**:
- 루프 분기 `i < N`: 99,9999개는 taken(루프 계속), 마지막 1번만 not-taken(루프 탈출)
- 2-bit 카운터가 ST(Strongly Taken) 상태로 고정 → 적중률 ≈ 100%
- branch-misses ≈ 1 (루프 마지막 1번만 틀림)

**코드 B 분석 (랜덤 배열)**:
- `arr[i] & 1`이 랜덤 → 50% 확률로 홀수/짝수
- 분기 방향이 무작위 → 예측기 학습 불가
- 적중률 ≈ 50%
- branch-misses ≈ N/2 = 5,000,000 misses

**예상 perf 수치**:
```
코드 A:
  branches:      10,000,001 (루프 분기)
  branch-misses:          1  (0.00001%)

코드 B:
  branches:      10,000,000
  branch-misses:  ~5,000,000  (~50%)
```

**코드 B 개선**:
- 정렬 후 처리: 홀수 구간이 연속되어 예측기가 패턴 학습 가능
- Branchless: `sum += arr[i] & -(arr[i] & 1)` (분기 자체 제거)

</details>

---

**Q2.** 2-bit 포화 카운터가 다음 분기 패턴을 처리할 때 미스가 몇 번 발생하는가? 초기 상태는 WN(01)이다.

```
분기 패턴: T T T N T T T N T T T N (반복)
```

<details>
<summary>해설 보기</summary>

**상태 전이 추적**:

| 순서 | 실제 | 예측 | 결과 | 다음 상태 |
|------|------|------|------|-----------|
| 1 (T) | T | N (WN) | 미스 | WT(10) |
| 2 (T) | T | T (WT) | 맞음 | ST(11) |
| 3 (T) | T | T (ST) | 맞음 | ST(11) |
| 4 (N) | N | T (ST) | 미스 | WT(10) |
| 5 (T) | T | T (WT) | 맞음 | ST(11) |
| 6 (T) | T | T (ST) | 맞음 | ST(11) |
| 7 (T) | T | T (ST) | 맞음 | ST(11) |
| 8 (N) | N | T (ST) | 미스 | WT(10) |
| 9 (T) | T | T (WT) | 맞음 | ST(11) |
| ... | | | | |

**패턴이 정착되면** (WT → ST → ST → ST → N → WT 반복):
- 4개 분기마다 1번 미스 (N이 올 때)
- 패턴 T T T N에서 미스율 = 1/4 = 25%

**초기 수렴 과정**:
- 처음에는 WN에서 시작해 1번(T)에서 미스
- 이후 패턴이 정착되면 25% 미스율 유지

**역사 기반 예측기(TAGE)라면**: GHR이 "TTT" 패턴을 학습 → 다음은 N을 예측 → 미스율 ≈ 0%. 이것이 2-bit 단순 카운터 대비 역사 기반 예측기의 우위입니다.

</details>

---

**Q3.** `perf stat`에서 다음 수치를 얻었다. 이 프로그램의 병목은 분기 예측인가, 메모리 접근인가? 어떻게 판단하며 다음 조치는 무엇인가?

```
Performance counter stats:
  5,000,000,000  cycles
  4,500,000,000  instructions          # 0.90 insn per cycle
      100,000,000  branches
          500,000  branch-misses        # 0.50%
      200,000,000  cache-references
       50,000,000  cache-misses         # 25.00% cache miss rate
```

<details>
<summary>해설 보기</summary>

**판단: 메모리 접근이 병목, 분기 예측은 양호**

**분기 예측 분석**:
- branch-miss율: 0.50% → 매우 낮음 (정상 범위 < 1~2%)
- 분기로 인한 낭비 사이클: 500,000 × 15 = 7,500,000 사이클
- 전체 5,000,000,000 사이클의 0.15% → 무시 가능

**메모리 접근 분석**:
- cache-miss율: 25% → 심각하게 높음 (정상 1~5%)
- 캐시 미스 비용: 50,000,000 × ~40~200 사이클 = 2~10,000,000,000 사이클 잠재 비용
- IPC = 0.90 → 1.0 미만, 스톨이 많다는 신호
- `stalled-cycles-backend`를 추가로 확인하면 메모리 대기 확인 가능

**다음 조치**:
```bash
# 1. 어떤 캐시 레벨에서 미스가 발생하는지 확인
perf stat -e L1-dcache-load-misses,L2-load-misses,LLC-load-misses ./program

# 2. cachegrind로 어느 코드에서 미스 발생인지
valgrind --tool=cachegrind ./program
cg_annotate cachegrind.out.<pid>

# 3. 접근 패턴 개선 방향
# - 랜덤 접근 → 선형 접근으로 변경
# - AoS → SoA 변환 (Ch2-05)
# - 워킹 셋 크기 줄이기
# - 프리페치 힌트 삽입
```

**결론**: 분기 예측 최적화가 아니라 Ch2 캐시/메모리 계층 최적화가 우선입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 슈퍼스칼라와 OoO](./04-superscalar-out-of-order.md)** | **[홈으로 🏠](../README.md)** | **[다음: ILP의 한계 ➡️](./06-ilp-limits.md)**

</div>
