# 자동 벡터화 — 컴파일러가 루프를 SIMD로 바꾸는 조건

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- GCC/Clang이 루프를 자동으로 벡터화하려면 어떤 조건 4가지를 충족해야 하는가?
- 같은 루프인데 `-O2`에서는 벡터화되고 `-O1`에서는 안 되는 이유는?
- `__restrict__` 키워드가 aliasing 문제를 해결하는 메커니즘은 정확히 무엇인가?
- `-fopt-info-vec`와 `-fopt-info-vec-missed`로 컴파일러의 벡터화 판단 이유를 읽는 방법은?
- godbolt에서 어셈블리를 보고 벡터화 성공/실패를 판단하는 기준은?
- 루프 안에 `if`문 하나가 왜 벡터화를 막는가, 그리고 우회 방법은?

---

## 🔍 왜 이 개념이 중요한가

### "컴파일러가 알아서 벡터화하는 거 아닌가요?"

```
흔한 오해:
  gcc -O2 를 쓰면 → 자동으로 최적 SIMD 코드
  → "인트린식 같은 걸 직접 쓸 필요 없다"

현실:

  // 이 루프: 벡터화됨 ✅
  void add(float* c, const float* a, const float* b, int n) {
      for (int i = 0; i < n; i++)
          c[i] = a[i] + b[i];
  }

  // 이 루프: 벡터화 실패 ❌
  void add_aliased(float* c, float* a, float* b, int n) {
      for (int i = 0; i < n; i++)
          c[i] = a[i] + b[i];  // c가 a와 겹칠 수 있다!
  }

차이: const 제거 하나로 컴파일러가 aliasing을 의심
  → 벡터화 포기 → 스칼라 코드
  → 8배 처리량 손실

알아야 하는 이유:
  수동 인트린식 없이도 SIMD를 활용하려면
  컴파일러가 "벡터화해도 안전하다"고 판단할 수 있는
  코드를 작성해야 함
  → 루프 구조 + 타입 + aliasing + 분기 = 4개 조건
```

---

## 😱 잘못된 이해

### Before: "벡터화 여부는 컴파일러가 알아서 판단하므로 내가 신경 쓸 필요 없다"

```
잘못된 이해:
  컴파일러가 "가능하면 항상" 벡터화한다
  → 내 코드에 문제가 없으면 자동 처리됨

놓친 것들:
  1. 컴파일러는 "안전성"을 증명해야만 벡터화
     안전성 = 순서를 바꿔도 결과가 같은가
     → 의심스러우면 보수적으로 포기

  2. 포인터 인자가 있으면 aliasing 의심
     void f(int* a, int* b, int n)
       → a와 b가 같은 메모리를 가리킬 수 있음
       → a[0] = a[0] + b[0] 실행 후
          a[1] = a[1] + b[1] 에서 b[1]이 바뀔 수 있음
       → 순서 바꾸면 결과 달라짐 → 벡터화 불가

  3. 루프 반복 횟수가 동적이면
     for (int i = 0; i < n; i++)  // n이 런타임 값
       → n을 모르니 나머지 처리(epilogue) 코드 필요
       → 컴파일러가 벡터 + 스칼라 두 가지 코드 생성

  4. 실제로 벡터화됐는지 확인하는 방법을 모름
     → 실패를 인지 못하고 "최적화됐겠지"라고 가정
```

---

## ✨ 올바른 이해

### After: 4개 조건을 이해하면 벡터화를 예측하고 제어할 수 있다

```
자동 벡터화의 4가지 전제 조건:

① 반복 횟수가 고정되거나 추론 가능
   for (int i = 0; i < 1024; i++)  ✅ 상수 → 바로 벡터화
   for (int i = 0; i < n; i++)     ✅ 런타임 n → 벡터+스칼라 epilogue

② Aliasing(포인터 겹침)이 없음
   void f(float* __restrict__ a, float* __restrict__ b, int n)  ✅
   void f(float* a, float* b, int n)  ❌ 기본적으로 의심

③ 루프 내 분기(if)가 없음 — 또는 마스킹으로 변환 가능
   for (...) { if (cond) c[i] = ...; }  ❌ 분기 있음
   for (...) { c[i] = cond ? a[i] : b[i]; }  ✅ 삼항 → blendps/cmov

④ 루프 반복 간 의존성 없음
   c[i] = a[i] + b[i-1];  ❌ b[i-1]이 이전 반복과 의존 가능
   c[i] = a[i] + b[i];    ✅ 각 i 독립

컴파일러 흐름:
  루프 발견 → 의존성 분석 → aliasing 분석 → 비용 모델 평가
    → 벡터화 이득 > 비용 → YMM/ZMM 코드 생성
    → 아니면 → 스칼라 코드 유지 (or 경고 없이 조용히 포기)
```

---

## 🔬 내부 동작 원리

### 1. 컴파일러 벡터화 파이프라인

```
루프 소스 코드
    │
    ▼
① 중간 표현(IR) 변환 (GIMPLE / LLVM IR)
    │  루프 규범화(Loop Normalization): 증가 방향, 단일 출구 확인
    ▼
② 의존성 분석 (Dependence Analysis)
    │  각 배열 접근 a[f(i)], b[g(i)] 사이의 의존성 벡터 계산
    │  RAW / WAW / WAR 의존성이 루프 내에 있는가?
    ▼
③ Aliasing 분석
    │  restrict / const / 타입 기반 TBAA(Type-Based Alias Analysis)
    │  컴파일러가 "겹치지 않음"을 증명 못하면 → 포기
    ▼
④ 비용 모델 (Cost Model)
    │  벡터화 후 사이클 수 < 스칼라 사이클 수?
    │  예외 처리(epilogue) 비용 포함
    ▼
⑤ 코드 생성
    벡터 루프 (메인) + 스칼라 나머지 (epilogue 0~7 요소)
    필요 시 런타임 aliasing 체크 코드 삽입
```

### 2. Aliasing이 벡터화를 막는 메커니즘

```c
// Case 1: aliasing 의심 → 스칼라
void add_v1(int* c, int* a, int* b, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}
// 컴파일러의 우려:
//   c == a+1 인 경우:
//     c[0] = a[0] + b[0]  → a[1]이 바뀜 (c[0] == a[1])
//     c[1] = a[1] + b[1]  → 이미 변경된 a[1] 사용 = 순서 중요
//   → 8개를 한번에 실행하면 결과가 달라짐 → 벡터화 불가

// Case 2: __restrict__ 선언 → 벡터화 ✅
void add_v2(int* __restrict__ c, int* __restrict__ a, int* __restrict__ b, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}
// __restrict__: "이 포인터들은 서로 겹치지 않는다" — 프로그래머 보증
// 컴파일러: aliasing 없음 확인 → 즉시 벡터 코드 생성

// Case 3: const로 읽기 전용 선언 → 벡터화 ✅ (더 명확)
void add_v3(int* __restrict__ c, const int* a, const int* b, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}
// a, b는 const → 쓰기 없음 → c와 같은 메모리라도 a, b는 변경 안됨
// → 더 안전한 추론 가능

// Case 4: 런타임 aliasing 체크 코드 삽입 (GCC의 선택)
// 일부 경우 컴파일러는:
//   if (c+n <= a || a+n <= c) {  ← aliasing 없음 확인
//       /* 벡터 루프 */
//   } else {
//       /* 스칼라 루프 */
//   }
// → 런타임 오버헤드 있지만 안전
```

### 3. 루프 내 분기(if)가 벡터화를 막는 메커니즘

```c
// Case 1: 분기 있음 → 벡터화 실패 ❌
void threshold_v1(float* out, const float* in, int n) {
    for (int i = 0; i < n; i++) {
        if (in[i] > 0.0f)        // ← 분기!
            out[i] = in[i] * 2.0f;
        else
            out[i] = 0.0f;
    }
}
// 이유: 루프 내 분기 = 제어 흐름 의존성
//   8개 요소가 서로 다른 경로 → 한번에 실행 불가?
// 사실 현대 컴파일러는 이 경우를 마스킹으로 변환 가능하지만
// 복잡한 분기는 포기

// Case 2: 삼항 연산자로 변환 → 벡터화 ✅
void threshold_v2(float* out, const float* in, int n) {
    for (int i = 0; i < n; i++)
        out[i] = (in[i] > 0.0f) ? in[i] * 2.0f : 0.0f;
}
// → blendps (마스킹 셀렉트) 명령으로 변환:
//   vcmpgtps ymm_mask, ymm_in, ymm_zero  ; 비교 결과 마스크
//   vmulps   ymm_mul,  ymm_in, ymm_two   ; in * 2.0
//   vblendvps ymm_out, ymm_zero, ymm_mul, ymm_mask  ; 마스크 선택

// Case 3: break가 있으면 반드시 실패
void find_first(const int* a, int n, int target) {
    for (int i = 0; i < n; i++)
        if (a[i] == target) break;  // ← 조기 종료 → 반복 횟수 불확정
}
// break: 조기 종료 → 몇 번 반복할지 모름 → 벡터화 불가
```

### 4. godbolt에서 벡터화 성공/실패 판단법

```
벡터화 성공의 어셈블리 특징:
  → ymm/xmm/zmm 레지스터 등장
  → vmovups / vmovaps (벡터 로드/저장)
  → vaddps / vaddpd / vpaddd 등 벡터 연산
  → 루프 스텝이 스칼라보다 큼 (add rax, 32 = 8 float 전진)

벡터화 실패의 어셈블리 특징:
  → xmm도 없이 eax/rax만 사용
  → 루프 스텝 = add rax, 4 (int 1개씩)
  → addss (스칼라 float 덧셈) 또는 add eax

예시:
  벡터화 성공:
    .loop:
      vmovups ymm0, [rsi+rax]     ; 8 float 로드
      vaddps  ymm0, ymm0, [rdx+rax] ; 8 float 덧셈
      vmovups [rdi+rax], ymm0     ; 8 float 저장
      add     rax, 32             ; 32 bytes = 8 float 전진
      cmp     rax, rcx
      jne     .loop

  벡터화 실패:
    .loop:
      movss   xmm0, [rsi+rax]     ; 1 float 로드
      addss   xmm0, [rdx+rax]     ; 1 float 덧셈
      movss   [rdi+rax], xmm0     ; 1 float 저장
      add     rax, 4              ; 4 bytes = 1 float 전진
      cmp     rax, rcx
      jne     .loop
```

### 5. `-fopt-info-vec` / `-fopt-info-vec-missed` 출력 읽기

```bash
# 벡터화 성공 메시지 보기
gcc -O2 -mavx2 -fopt-info-vec my_loop.c -o my_loop

# 예상 출력:
# my_loop.c:5:5: optimized: loop vectorized using 32-byte vectors
#   ↑ 5번째 줄, 5번째 컬럼의 루프가 32byte(YMM) 벡터화 성공

# 벡터화 실패 이유 보기
gcc -O2 -mavx2 -fopt-info-vec-missed my_loop.c -o my_loop

# 예상 출력 예시:
# my_loop.c:15:5: missed: couldn't vectorize loop
# my_loop.c:15:5: missed: not vectorized: control flow in loop.
#   ↑ 15번째 줄 루프: "제어 흐름(if/break)이 있어서 벡터화 불가"

# my_loop.c:22:5: missed: not vectorized: possible aliasing between ...
#   ↑ 22번째 줄 루프: "aliasing 가능성으로 벡터화 불가"

# my_loop.c:30:5: missed: not vectorized: bad loop form.
#   ↑ 30번째 줄 루프: "루프 형식 불량 (복잡한 증감식, 포인터 산술 등)"

# Clang 버전:
clang -O2 -mavx2 -Rpass=loop-vectorize \
      -Rpass-missed=loop-vectorize \
      -Rpass-analysis=loop-vectorize my_loop.c -o my_loop
# clang이 더 상세한 이유 설명 (예: "loop not vectorized: unsafe dependent memory operations in loop")
```

---

## 💻 실전 실험

### 실험 1: aliasing 있음/없음 어셈블리 비교

```c
// alias_test.c
void add_no_restrict(float* c, float* a, float* b, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}

void add_restrict(float* __restrict__ c,
                  float* __restrict__ a,
                  float* __restrict__ b, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}
```

```bash
gcc -O2 -mavx2 -S -masm=intel alias_test.c -o alias_test.s

# add_no_restrict 어셈블리 확인:
grep -A 30 "add_no_restrict:" alias_test.s
# → 런타임 aliasing 체크 코드가 삽입되거나 스칼라 유지
# → "vpcmpeqd" / 주소 비교 코드 등장 가능

# add_restrict 어셈블리 확인:
grep -A 30 "add_restrict:" alias_test.s
# → 바로 vmovups + vaddps + vmovups 벡터 루프
# → aliasing 체크 없음, 깔끔한 벡터 코드

# 또는 godbolt에서 직접 확인:
# https://godbolt.org — 두 함수 나란히 비교
```

### 실험 2: `-fopt-info-vec-missed`로 실패 이유 체계적으로 확인

```c
// vec_test.c — 다양한 벡터화 성공/실패 케이스
#include <math.h>

// 케이스 1: 성공 (기본 조건 충족)
void case1(float* __restrict__ c, const float* a, const float* b, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}

// 케이스 2: 실패 — 분기
void case2(float* __restrict__ c, const float* a, int n) {
    for (int i = 0; i < n; i++) {
        if (a[i] > 0) c[i] = a[i];    // if 분기
        else c[i] = -a[i];
    }
}

// 케이스 3: 성공 — 삼항 (case2 개선)
void case3(float* __restrict__ c, const float* a, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] > 0 ? a[i] : -a[i];   // 삼항 → 마스킹
}

// 케이스 4: 실패 — 의존성 사슬
void case4(float* a, int n) {
    for (int i = 1; i < n; i++)
        a[i] = a[i] + a[i-1];  // a[i]가 a[i-1]에 의존
}

// 케이스 5: 실패 — 함수 호출
void case5(float* __restrict__ c, const float* a, int n) {
    for (int i = 0; i < n; i++)
        c[i] = sqrtf(a[i]);    // sqrtf: 벡터 수학 함수 없으면 실패
}

// 케이스 6: 성공 — vsqrtps 사용 (-ffast-math)
// gcc -O2 -mavx2 -ffast-math 로 컴파일 시 vsqrtps 생성
```

```bash
gcc -O2 -mavx2 -fopt-info-vec -fopt-info-vec-missed vec_test.c -o vec_test 2>&1 | head -40

# 예상 출력:
# vec_test.c:6:5: optimized: loop vectorized using 32-byte vectors
#   → case1 성공

# vec_test.c:13:5: missed: not vectorized: control flow in loop.
#   → case2 실패 (if 분기)

# vec_test.c:20:5: optimized: loop vectorized using 32-byte vectors
#   → case3 성공 (삼항 → blendvps)

# vec_test.c:26:5: missed: not vectorized: unsafe dependent memory operations
#   → case4 실패 (a[i-1] 의존)

# vec_test.c:32:5: missed: not vectorized: statement clobbers memory
#   → case5 실패 (라이브러리 함수)

# -ffast-math 추가 시 sqrtf → vsqrtps 변환:
gcc -O2 -mavx2 -ffast-math -fopt-info-vec vec_test.c -o vec_test_fast 2>&1
# vec_test.c:32:5: optimized: loop vectorized using 32-byte vectors
```

### 실험 3: perf로 벡터화 전후 처리량 비교

```bash
# 벡터화 비활성화 vs 활성화 비교
gcc -O2 -fno-tree-vectorize -o no_vec alias_test.c bench_main.c
gcc -O2 -mavx2 -o with_vec alias_test.c bench_main.c

perf stat -e cycles,instructions,\
fp_arith_inst_retired.256b_packed_single,\
fp_arith_inst_retired.scalar_single \
./no_vec

perf stat -e cycles,instructions,\
fp_arith_inst_retired.256b_packed_single,\
fp_arith_inst_retired.scalar_single \
./with_vec

# no_vec 예상:
#   fp_arith_inst_retired.256b_packed_single:  0        ← SIMD 전혀 없음
#   fp_arith_inst_retired.scalar_single:       N        ← 전부 스칼라

# with_vec 예상:
#   fp_arith_inst_retired.256b_packed_single:  N/8      ← YMM 사용
#   fp_arith_inst_retired.scalar_single:       N%8      ← epilogue만
# → 명령 수 8배 감소 = 처리량 8배 향상 확인
```

### 실험 4: `#pragma GCC ivdep`로 컴파일러 aliasing 경고 억제

```c
// pragma_test.c
void add_pragma(float* c, float* a, float* b, int n) {
    #pragma GCC ivdep          // "내가 aliasing 없다고 보장, 무시하고 벡터화"
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}

// Clang 버전:
void add_pragma_clang(float* c, float* a, float* b, int n) {
    #pragma clang loop vectorize(enable)
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}
```

```bash
gcc -O2 -mavx2 -S -masm=intel pragma_test.c -o pragma_test.s
# → vaddps ymm 생성 확인 (__restrict__ 없이도 벡터화)

# 주의: #pragma ivdep는 프로그래머 책임
# aliasing이 실제로 있는데 pragma 쓰면 → 조용한 데이터 손상
```

---

## 📊 성능 비교

```
N=4M float 배열 덧셈 (L3 캐시 내 기준):

코드 변형                     벡터화     시간(예상)   배율
──────────────────────────────────────────────────────────
float* c, float* a (인자)     ❌ 스칼라   ~8.0ms      1×
const float* a + restrict     ✅ AVX2    ~1.1ms      7.3×
__restrict__ 양쪽              ✅ AVX2    ~1.1ms      7.3×
#pragma GCC ivdep              ✅ AVX2    ~1.1ms      7.3×
-fopt-info-vec 확인 후 수정    ✅ AVX2    ~1.1ms      7.3×

분기 처리 차이:
  if/else 형식          ❌ 스칼라   ~6.0ms      1×
  삼항 연산자 형식       ✅ AVX2    ~1.3ms      4.6×  (blendvps 포함)
  branchless (abs 등)   ✅ AVX2    ~1.0ms      6×

의존성 해소 기법:
  누산기 1개 (의존성 사슬) ❌ 스칼라  ~5.0ms      1×
  누산기 4개 (Ch1-06 ILP)  ✅+ILP   ~1.5ms      3.3×
  누산기 4개 + AVX2         ✅AVX2   ~0.4ms      12.5×

컴파일러별 차이 (동일 코드, -O2 -mavx2):
  GCC 13.2:   add_restrict → YMM 벡터화 ✅
  Clang 17:   add_restrict → YMM 벡터화 ✅ (더 공격적)
  GCC 13.2:   add_no_restrict → 런타임 aliasing 체크 + YMM (느림)
  Clang 17:   add_no_restrict → 스칼라 (보수적)
```

---

## ⚖️ 트레이드오프

```
자동 벡터화 vs 수동 인트린식:

자동 벡터화:
  ✅ 코드 가독성 유지
  ✅ 플랫폼 독립성 (컴파일러가 최적 SIMD 선택)
  ✅ 유지보수 편의 (비즈니스 로직 변경 시 자동 재벡터화)
  ❌ 컴파일러가 못 하는 경우 있음 (수평 연산, 특수 셔플)
  ❌ 컴파일러 버전별 결과 다를 수 있음

수동 인트린식 (Ch5-03):
  ✅ 컴파일러 판단 불필요 → 항상 원하는 SIMD 명령 생성
  ✅ 특수 명령(_mm256_hadd_ps, _mm256_permute_ps) 사용 가능
  ❌ 코드 가독성 극히 낮음
  ❌ 플랫폼 종속 (SSE / AVX2 / AVX-512 각각 작성)
  ❌ 유지보수 비용 높음

권장 전략:
  1. 먼저 자동 벡터화 시도 (const + __restrict__ + 삼항)
  2. -fopt-info-vec-missed로 실패 이유 확인
  3. 루프 리팩토링으로 해결 가능하면 해결
  4. 그래도 안 되면 → 수동 인트린식 (Ch5-03)

__restrict__의 위험성:
  void buggy(int* __restrict__ a, int* __restrict__ b, int n) {
      for (int i = 0; i < n; i++)
          a[i] += b[i];
  }
  buggy(arr+1, arr, 10);  // 실제로 겹침! 하지만 restrict 선언
  → 벡터화 후 결과 undefined (UB)
  → restrict는 "실제로 겹치지 않음을 내가 보증"할 때만 사용

-ffast-math 주의:
  sqrtf → vsqrtps 변환 등 벡터 수학 함수 활성화
  그러나 NaN/Inf 처리 변경, 결합법칙 재배열 등 허용
  → 과학 계산에서 정밀도 문제 가능
  → 게임/DSP 등 정밀도 덜 중요한 곳에서만 권장
```

---

## 📌 핵심 정리

```
자동 벡터화 4가지 조건:

① 반복 횟수 추론 가능
   for (int i = 0; i < n; i++) → OK (epilogue 생성)
   while (ptr != end) → 불확실한 경우 실패

② Aliasing 없음
   __restrict__ 또는 const + 읽기 전용 = 가장 확실한 해결
   포인터 인자 그대로 = 컴파일러 의심 → 실패 or 런타임 체크

③ 분기 없음 (또는 마스킹으로 변환 가능)
   if/else → 삼항 연산자로 리팩토링
   break/continue → 제거 불가 → 벡터화 포기

④ 반복 간 의존성 없음
   a[i] = a[i-1] + x → 의존성 존재 → 실패
   a[i] = a[i] + b[i] → 독립 → 성공

진단 도구:
  GCC:   -fopt-info-vec / -fopt-info-vec-missed
  Clang: -Rpass=loop-vectorize / -Rpass-missed=loop-vectorize
  어셈블리: ymm 레지스터 + vaddps/vmulps 등장 여부

실패 시 수동 수정 우선순위:
  1. const 추가 / __restrict__ 추가
  2. if → 삼항 연산자
  3. 의존성 사슬 → 누산기 분할 (Ch1-06)
  4. 여전히 실패 → Ch5-03 수동 인트린식
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 루프 중 GCC `-O2 -mavx2`에서 자동 벡터화될 가능성이 높은 것과 낮은 것을 분류하고, 각 이유를 설명하라.

```c
// A
for (int i = 0; i < n; i++)
    a[i] = b[i] * 2.0f + c[i];

// B
for (int i = 0; i < n; i++)
    if (b[i] != 0) a[i] = 1.0f / b[i];

// C
for (int i = 0; i < n; i++)
    a[i] = a[i-1] * 0.9f + b[i];

// D
for (int i = 0; i < n; i++)
    a[i] = (b[i] > 0.0f) ? b[i] : -b[i];  // fabs
```

<details>
<summary>해설 보기</summary>

**A — 높음**: `a`, `b`, `c` 포인터 인자지만 컴파일러가 런타임 aliasing 체크를 삽입하거나, `__restrict__`가 있으면 바로 벡터화. 독립 반복, 분기 없음. `__restrict__` 추가 권장.

**B — 낮음**: `if (b[i] != 0)` 분기가 있어서 컴파일러가 마스킹 변환을 시도하지만, 0으로 나누기(1.0f/0.0f = Inf)가 발생하는 경우 IEEE 754 예외 처리 문제로 포기할 수 있습니다. `-ffast-math` 추가 시 성공 가능. 삼항으로 바꿔도 동일 문제.

**C — 매우 낮음 (실패)**: `a[i] = a[i-1] * ...`는 이전 반복 결과에 의존하는 **재귀 의존성**입니다. `a[i-1]`이 현재 반복에서 계산되었으므로 8개를 동시에 실행할 수 없습니다. 자동 벡터화 거의 불가.

**D — 높음**: 삼항 연산자(`? :`)는 `vblendvps` / `vandps`로 변환 가능합니다. 실제로 `fabs` 연산은 AVX2에서 `vandps ymm, ymm, abs_mask`로 최적화됩니다. 분기 없이 마스킹으로 처리되어 높은 확률로 벡터화 성공.

</details>

---

**Q2.** `__restrict__` 없이도 `const float* a, const float* b` 선언만으로 벡터화가 성공하는 이유는? `const float* a, float* c` (a는 const, c는 non-const)인 경우는?

<details>
<summary>해설 보기</summary>

**`const float* a, const float* b` 양쪽 const**: 컴파일러는 "루프 내에서 `a`와 `b`에 쓰기가 없다"고 확신합니다. 쓰기가 없으면 `a[i]`를 미리 읽어도 나중 반복에 영향이 없으므로 순서를 바꿔도 안전 → 벡터화 허용. 단, `c[i] = a[i] + b[i]`에서 `c`가 non-const이면 `c`와 `a`/`b`가 같은 메모리일 가능성이 여전히 존재하므로 컴파일러는 런타임 aliasing 체크를 추가하거나 벡터화를 보수적으로 포기합니다.

**`const float* a, float* c` (나머지 non-const)**: `a`는 const이므로 루프 내 변경 없음 확실. `c`는 non-const → `c`가 `a`와 겹칠 가능성 있음. 그러나 `a`는 const로 선언했기 때문에 TBAA(Type-Based Alias Analysis) 관점에서 "const 포인터를 통한 쓰기 없음"을 활용해 벡터화를 시도할 수 있습니다. 결론: GCC는 이 경우 런타임 체크를 삽입한 벡터화를 선택하는 경우가 많습니다. 확실한 보장은 `__restrict__` 추가입니다.

</details>

---

**Q3.** 다음 루프를 자동 벡터화되도록 리팩토링하라. 원래 의미(결과)는 유지해야 한다.

```c
float running_max = -1e38f;
for (int i = 0; i < n; i++) {
    if (a[i] > running_max)
        running_max = a[i];
}
```

<details>
<summary>해설 보기</summary>

`running_max`가 매 반복에서 이전 값에 의존하므로 의존성 사슬이 있습니다. 8개 독립 누산기로 분리하면 벡터화 가능합니다.

```c
// 방법 1: vmaxps를 활용한 벡터화 (컴파일러가 자동 변환)
__m256 vmax = _mm256_set1_ps(-1e38f);
for (int i = 0; i <= n - 8; i += 8) {
    __m256 va = _mm256_loadu_ps(a + i);
    vmax = _mm256_max_ps(vmax, va);      // 8개 동시 max
}
// 수평 max 추출 (Ch5-03에서 상세)
float buf[8]; _mm256_storeu_ps(buf, vmax);
float result = buf[0];
for (int i = 1; i < 8; i++) result = result > buf[i] ? result : buf[i];
for (int i = (n & ~7); i < n; i++) result = result > a[i] ? result : a[i];
```

또는 컴파일러에게 힌트만 주는 방법:
```c
// 누산기 8개 → 컴파일러 자동 벡터화 유도
float max0 = -1e38f, max1 = -1e38f, max2 = -1e38f, max3 = -1e38f;
float max4 = -1e38f, max5 = -1e38f, max6 = -1e38f, max7 = -1e38f;
for (int i = 0; i <= n - 8; i += 8) {
    max0 = a[i+0] > max0 ? a[i+0] : max0;
    max1 = a[i+1] > max1 ? a[i+1] : max1;
    // ... (반복)
}
float result = max0;
for (int j = 1; j < 8; j++) { float m = (&max0)[j]; if (m > result) result = m; }
```

핵심: 의존성 사슬을 끊어 독립 누산기로 만들면 컴파일러가 `vmaxps ymm`으로 벡터화합니다. 이는 Ch1-06의 ILP 최적화와 동일한 원리입니다.

</details>

---

<div align="center">

**[⬅️ 이전: SIMD 기본](./01-simd-basics.md)** | **[홈으로 🏠](../README.md)** | **[다음: 수동 SIMD 인트린식 ➡️](./03-manual-simd-intrinsics.md)**

</div>
