# 수동 SIMD — 인트린식으로 합·내적 가속

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `_mm256_loadu_ps` vs `_mm256_load_ps`의 차이와 비정렬 로드의 실제 비용은?
- `_mm256_fmadd_ps`가 두 개의 명령(`vmulps` + `vaddps`)보다 빠른 이유는?
- 수평 합(horizontal sum)을 구현하는 올바른 방법과 흔한 실수는?
- 내적(dot product) 연산에서 스칼라 대비 4~8배 가속을 실측하는 방법은?
- 인트린식 코드를 작성할 때 자주 발생하는 `#include` 누락·타입 오류는?
- AVX2 코드와 AVX-512 코드를 함께 쓸 때 주의할 `VZEROUPPER` 이슈는?

---

## 🔍 왜 이 개념이 중요한가

### "자동 벡터화가 실패하거나 부족할 때"

```
자동 벡터화가 실패하는 대표적 상황:
  1. 수평 연산(horizontal reduction): sum, max, dot product
     → 컴파일러가 어떤 수평 명령을 써야 할지 모름
     → 또는 pairwise 방식으로 비효율적으로 생성

  2. 특수 셔플/펼치기: AoS → SoA 전환
     → vunpcklps, vperm2f128 같은 특수 명령
     → 컴파일러가 패턴 인식 실패

  3. 성능 임계 코드:
     딥러닝 추론 커널, 코덱, 암호화
     → 마지막 5%의 성능을 위해 수동 제어 필요

  4. 컴파일러 버전 의존성 제거:
     "이 코드는 어떤 컴파일러에서든 반드시 SIMD 생성"

내적 연산 예시:
  float dot(float* a, float* b, int n) {
      float sum = 0;
      for (int i = 0; i < n; i++) sum += a[i] * b[i];
      return sum;
  }
  → 자동 벡터화: 부분합 누산 → vmulps + vaddps → 수평합 → 가능
  → 수동 인트린식 + FMA: vfmadd231ps 한 명령으로 곱+합 처리
  → 추가 성능: ~20~30% (FMA 처리량 활용)
```

---

## 😱 잘못된 이해

### Before: "인트린식은 어렵고 위험해서 전문가만 쓰는 것이다"

```
잘못된 이해:
  인트린식은 어셈블리나 다름없다
  → 일반 개발자는 쓰면 안 됨
  → 컴파일러가 더 잘 한다

놓친 것:
  인트린식(intrinsic)은:
    C 함수 호출 형태 = 타입 체크, 디버거 가능
    어셈블리 명령과 1:1 대응 = 불확실성 없음
    헤더 한 줄 = #include <immintrin.h>

  자동 벡터화가 못하는 핵심 연산:
    수평 합:   _mm256_hadd_ps → 인접 쌍 합산
    셔플:     _mm256_permute_ps → 요소 재배치
    FMA:      _mm256_fmadd_ps → 정밀도 향상 + 처리량 최대화
    마스킹:   _mm256_blendv_ps → 조건부 선택

  실제 사용처:
    Intel MKL, OpenBLAS, libjpeg-turbo, FFmpeg, TensorFlow 커널
    → 수동 인트린식으로 수천 줄 작성
```

---

## ✨ 올바른 이해

### After: 인트린식은 어셈블리보다 안전한 SIMD 직접 제어 수단

```
인트린식 명명 규칙 이해:
  _mm256_loadu_ps(ptr)
   ──┬─── ──┬── ─┬─ ─┬─
     │      │    │   └── 타입: ps=packed single(float), pd=double,
     │      │    │        epi32=32bit int, epi8=8bit int ...
     │      │    └─────── 연산: loadu=비정렬 로드, add=덧셈, mul=곱셈,
     │      │              fmadd=FMA, hadd=수평합, blend=혼합 ...
     │      └──────────── 레지스터 폭: mm=64bit, mm128=128bit,
     │                    mm256=256bit(YMM), mm512=512bit(ZMM)
     └─────────────────── 접두사 (항상 _mm)

주요 타입:
  __m128  : XMM, 4× float
  __m128i : XMM, 정수 (8/16/32/64 bit 해석은 연산에 따름)
  __m128d : XMM, 2× double
  __m256  : YMM, 8× float
  __m256i : YMM, 정수
  __m256d : YMM, 4× double
  __m512  : ZMM, 16× float (AVX-512)

필수 헤더:
  #include <immintrin.h>   ← AVX/AVX2/AVX-512 전부 포함
  (내부적으로 xmmintrin.h, emmintrin.h 등 순차 포함)
```

---

## 🔬 내부 동작 원리

### 1. 로드 명령 — 정렬 vs 비정렬, 처리량 차이

```c
#include <immintrin.h>

// 정렬된 로드 (32-byte aligned 주소 전용)
__m256 va = _mm256_load_ps(ptr);      // vmovaps ymm, [ptr]
// 주소가 32-byte 경계: 0x...00, 0x...20, 0x...40
// 조건 위반 시: SIGSEGV(segmentation fault) 또는 General Protection Fault

// 비정렬 로드 (임의 주소 OK)
__m256 vb = _mm256_loadu_ps(ptr);     // vmovups ymm, [ptr]
// 어떤 주소라도 동작

// 성능 차이 (Haswell 이후):
//   캐시 히트 시: 정렬 = 비정렬 = 동일 처리량·지연시간
//   캐시라인 분리(split) 발생 시:
//     정렬 주소라면 분리 없음 (확정)
//     비정렬 주소: 캐시라인 경계 걸치면 ~2배 지연
//
// 예시: ptr = 0x1018 (비정렬)
//   캐시라인: 0x1000~0x103F, 0x1040~0x107F
//   _mm256_loadu_ps(0x1018): 0x1018~0x1037 = 32 bytes
//   → 전부 첫 번째 캐시라인(~0x103F) 안 → 분리 없음 → 정렬과 동일
//
// ptr = 0x1030 (비정렬, 경계에 걸침)
//   _mm256_loadu_ps(0x1030): 0x1030~0x104F = 32 bytes
//   → 0x1030~0x103F: 첫 번째 캐시라인
//   → 0x1040~0x104F: 두 번째 캐시라인
//   → 두 캐시라인 필요 → 추가 지연 (약 2배)

// 정렬 메모리 할당:
float* buf = (float*)_mm_malloc(n * sizeof(float), 32);  // 32-byte 정렬
// 또는
#include <stdlib.h>
float* buf2;
posix_memalign((void**)&buf2, 32, n * sizeof(float));
// 또는 C++ alignas
alignas(32) float buf3[N];
```

### 2. FMA (Fused Multiply-Add) — 곱셈+덧셈을 한 명령으로

```c
// 일반 방법: 곱셈 후 덧셈 (2 명령)
__m256 vmul = _mm256_mul_ps(va, vb);      // vmulps ymm
__m256 vacc = _mm256_add_ps(vacc, vmul);  // vaddps ymm
// → 2 명령 × Throughput 0.5 cy = 1 cy 최소

// FMA: 1 명령으로 처리
__m256 vacc = _mm256_fmadd_ps(va, vb, vacc);  // vfmadd213ps / vfmadd231ps
// acc = va * vb + acc
// → 1 명령 × Throughput 0.5 cy = 0.5 cy 최소
// → 2배 처리량

// FMA 변형:
// _mm256_fmadd_ps (a,b,c)  → a * b + c
// _mm256_fnmadd_ps(a,b,c)  → -(a * b) + c   (negated multiply-add)
// _mm256_fmsub_ps (a,b,c)  → a * b - c
// _mm256_fmaddsub_ps(a,b,c) → 홀수 요소: a*b+c, 짝수 요소: a*b-c

// 정밀도 이점:
// vmulps + vaddps: 2번 반올림 (곱셈 후, 덧셈 후)
// vfmadd:          1번 반올림 (최종 결과만)
// → 수치적으로 더 정확 (특히 행렬 곱, 내적에서 중요)

// Intel 용어:
// FMA3 = 3-operand FMA (AVX2, 2013년~)
// FMA4 = 4-operand (AMD 전용, 거의 사용 안 함)
```

### 3. 수평 합 (Horizontal Sum) — SIMD 결과를 스칼라로 환원

```c
// 내적 연산 후 8개 float을 1개로 합산하는 과정

// 방법 1: 메모리 경유 (간단, 느림)
float buf[8];
_mm256_storeu_ps(buf, vsum);
float total = buf[0]+buf[1]+buf[2]+buf[3]+buf[4]+buf[5]+buf[6]+buf[7];

// 방법 2: hadd 사용 (2쌍씩 합산) - 하드웨어 지원
__m256 hadd_result = _mm256_hadd_ps(vsum, vsum);
// 입력:  [a0,a1,a2,a3, a4,a5,a6,a7]
// 출력:  [a0+a1, a2+a3, a0+a1, a2+a3, a4+a5, a6+a7, a4+a5, a6+a7]
// 주의: hadd는 인접 쌍을 합산, 128bit 레인 내에서만 동작!

hadd_result = _mm256_hadd_ps(hadd_result, hadd_result);
// 출력: [(a0+a1)+(a2+a3), ...] = 4개 부분합이 2개가 됨

// 방법 3: 고효율 수평 합 (권장 패턴)
static inline float hsum_avx2(__m256 v) {
    // 상위 128bit와 하위 128bit를 합산
    __m128 lo = _mm256_castps256_ps128(v);    // 하위 4 float
    __m128 hi = _mm256_extractf128_ps(v, 1); // 상위 4 float
    lo = _mm_add_ps(lo, hi);
    // 이제 lo = [a0+a4, a1+a5, a2+a6, a3+a7]

    // pairwise 합산 (SSE3)
    __m128 shuf = _mm_movehdup_ps(lo);       // [a1+a5, a1+a5, a3+a7, a3+a7]
    __m128 sums = _mm_add_ps(lo, shuf);      // [a01+a45, ?, a23+a67, ?]
    shuf = _mm_movehl_ps(shuf, sums);        // 상위 64bit를 하위로
    sums = _mm_add_ss(sums, shuf);           // 최종 합
    return _mm_cvtss_f32(sums);             // 스칼라 float로 추출
}
// → 총 5개 명령으로 8개 float 합산
// 방법 1(메모리) 대비 약 3배 빠름 (메모리 왕복 없음)

// 다이어그램:
// 초기 vsum: [a0, a1, a2, a3, a4, a5, a6, a7]
// lo:       [a0, a1, a2, a3]
// hi:       [a4, a5, a6, a7]
// lo+hi:    [a0+a4, a1+a5, a2+a6, a3+a7]   (4개)
// shuf:     [a1+a5, a1+a5, a3+a7, a3+a7]
// sums:     [a01+a45, ?, a23+a67, ?]        (2개 유효)
// shuf2:    [a23+a67, ?, ?, ?]
// final:    [a01+a45 + a23+a67]             (1개) ✅
```

### 4. 내적 연산 완전 구현 — 스칼라 대비 가속 측정

```c
#include <immintrin.h>
#include <stddef.h>

// 스칼라 버전
float dot_scalar(const float* a, const float* b, size_t n) {
    float sum = 0.0f;
    for (size_t i = 0; i < n; i++)
        sum += a[i] * b[i];
    return sum;
}

// AVX2 + FMA 버전 (8 float/명령)
float dot_avx2(const float* a, const float* b, size_t n) {
    __m256 vsum0 = _mm256_setzero_ps();
    __m256 vsum1 = _mm256_setzero_ps();   // 누산기 2개로 ILP 활용
    __m256 vsum2 = _mm256_setzero_ps();
    __m256 vsum3 = _mm256_setzero_ps();

    size_t i = 0;
    // 32개씩 (4 × 8 float = 32 float = 128 bytes) 처리
    for (; i + 32 <= n; i += 32) {
        __m256 va0 = _mm256_loadu_ps(a + i);
        __m256 vb0 = _mm256_loadu_ps(b + i);
        vsum0 = _mm256_fmadd_ps(va0, vb0, vsum0);  // sum += a*b

        __m256 va1 = _mm256_loadu_ps(a + i + 8);
        __m256 vb1 = _mm256_loadu_ps(b + i + 8);
        vsum1 = _mm256_fmadd_ps(va1, vb1, vsum1);

        __m256 va2 = _mm256_loadu_ps(a + i + 16);
        __m256 vb2 = _mm256_loadu_ps(b + i + 16);
        vsum2 = _mm256_fmadd_ps(va2, vb2, vsum2);

        __m256 va3 = _mm256_loadu_ps(a + i + 24);
        __m256 vb3 = _mm256_loadu_ps(b + i + 24);
        vsum3 = _mm256_fmadd_ps(va3, vb3, vsum3);
    }
    // 나머지 8개 단위 처리
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);
        __m256 vb = _mm256_loadu_ps(b + i);
        vsum0 = _mm256_fmadd_ps(va, vb, vsum0);
    }
    // 4개 누산기 합산
    vsum0 = _mm256_add_ps(vsum0, vsum1);
    vsum2 = _mm256_add_ps(vsum2, vsum3);
    vsum0 = _mm256_add_ps(vsum0, vsum2);

    // 나머지 1개 단위 (epilogue)
    float tail = 0.0f;
    for (; i < n; i++) tail += a[i] * b[i];

    return hsum_avx2(vsum0) + tail;
}
```

### 5. VZEROUPPER — AVX/SSE 혼용의 함정

```c
// 문제 시나리오
void sse_func() {
    __m128 xmm0 = _mm_loadu_ps(data);
    // ... XMM 연산 ...
}

void avx_func() {
    __m256 ymm0 = _mm256_loadu_ps(data);  // YMM 사용
    // ... YMM 연산 ...
    // ← 여기서 YMM의 상위 128bit가 오염된 상태
}

// avx_func() 호출 후 sse_func() 호출:
//   XMM0 = YMM0의 하위 128bit (이건 OK)
//   그런데 CPU가 YMM 전체를 추적 중 = False Dependency
//   → SSE 명령이 YMM 전체를 읽으려 함 → 지연 발생 (AVX-SSE 전환 패널티)

// 해결: VZEROUPPER (AVX 코드 블록 끝에 삽입)
void avx_func_safe() {
    __m256 ymm0 = _mm256_loadu_ps(data);
    // ... YMM 연산 ...
    _mm256_zeroupper();  // VZEROUPPER: YMM 상위 128bit 0으로 초기화
}
// 컴파일러(-O2 이상)가 자동 삽입하지만
// 인라인 어셈블리나 복잡한 함수 경계에서는 수동 삽입 필요

// godbolt에서 확인:
// avx_func가 리턴하기 직전 "vzeroupper" 명령이 있는지 체크
```

---

## 💻 실전 실험

### 실험 1: 내적 벤치마크 — 스칼라 vs AVX2+FMA

```c
// dot_bench.c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <immintrin.h>
#include <math.h>

#define N (1 << 20)   // 1M float

static float a[N], b[N];

// [위에서 정의한 dot_scalar, dot_avx2, hsum_avx2 포함]

static long now_ns() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000000000L + ts.tv_nsec;
}

int main() {
    for (int i = 0; i < N; i++) {
        a[i] = (float)(i % 100) / 100.0f;
        b[i] = (float)((N-i) % 100) / 100.0f;
    }

    const int REPS = 200;
    volatile float result;
    long t0, t1;

    // 워밍업
    for (int r = 0; r < 10; r++) result = dot_scalar(a, b, N);

    t0 = now_ns();
    for (int r = 0; r < REPS; r++) result = dot_scalar(a, b, N);
    t1 = now_ns();
    double scalar_ms = (t1 - t0) / 1e6 / REPS;

    t0 = now_ns();
    for (int r = 0; r < REPS; r++) result = dot_avx2(a, b, N);
    t1 = now_ns();
    double avx2_ms = (t1 - t0) / 1e6 / REPS;

    printf("Scalar:    %.3f ms (result=%.4f)\n", scalar_ms, (float)result);
    printf("AVX2+FMA:  %.3f ms (%.1fx faster)\n",
           avx2_ms, scalar_ms / avx2_ms);
    return 0;
}
```

```bash
# 컴파일 (FMA 활성화 필수)
gcc -O2 -mavx2 -mfma -o dot_bench dot_bench.c

# 예상 출력 (N=1M float, L3 캐시 내):
# Scalar:    3.20 ms
# AVX2+FMA:  0.42 ms (7.6x faster)

# perf로 FMA 명령 실행 확인
perf stat -e cycles,instructions,\
fp_arith_inst_retired.256b_packed_single \
./dot_bench
# fp_arith_inst_retired.256b_packed_single = N/8 × 2(mul+add = FMA 1개) × REPS
# 실제로는 FMA가 "2 FLOPs in 1 instruction"으로 계산됨
```

### 실험 2: godbolt에서 인트린식 어셈블리 검증

```c
// intrinsic_verify.c — godbolt.org GCC 13.2 / -O2 -mavx2 -mfma
#include <immintrin.h>

// 이 함수의 어셈블리를 확인
float dot8(const float* a, const float* b) {
    __m256 va   = _mm256_loadu_ps(a);
    __m256 vb   = _mm256_loadu_ps(b);
    __m256 vprod = _mm256_mul_ps(va, vb);
    // 수평 합 (간단 버전)
    __m128 lo = _mm256_castps256_ps128(vprod);
    __m128 hi = _mm256_extractf128_ps(vprod, 1);
    __m128 s  = _mm_add_ps(lo, hi);
    s = _mm_hadd_ps(s, s);
    s = _mm_hadd_ps(s, s);
    return _mm_cvtss_f32(s);
}
```

```
godbolt 예상 어셈블리:
  dot8(float const*, float const*):
    vmovups   ymm0, YMMWORD PTR [rdi]    ; a 8개 로드
    vmovups   ymm1, YMMWORD PTR [rsi]    ; b 8개 로드
    vmulps    ymm0, ymm0, ymm1           ; 8개 곱셈
    vextractf128  xmm1, ymm0, 0x1        ; 상위 128bit 추출
    vaddps    xmm0, xmm0, xmm1          ; lo + hi
    vhaddps   xmm0, xmm0, xmm0          ; pairwise 합산
    vhaddps   xmm0, xmm0, xmm0          ; pairwise 합산 2번째
    vzeroupper                            ; YMM 상위 클리어
    ret

FMA 버전 (vfmadd231ps로 변환):
  vfmadd231ps ymm0, ymm1, ymm2   ← vmulps + vaddps → 1 명령으로 합쳐짐
  → 컴파일러가 FMA 기회 발견 시 자동 치환 (-mfma 필요)
```

### 실험 3: 정렬 vs 비정렬 로드 성능 비교

```c
// alignment_bench.c
#include <immintrin.h>
#include <stdlib.h>
#include <time.h>
#include <stdio.h>
#include <string.h>

#define N (1 << 22)  // 4M float

int main() {
    // 정렬 버퍼 (32-byte 정렬)
    float* aligned   = (float*)_mm_malloc((N+1) * sizeof(float), 32);
    // 비정렬 버퍼 (1 float = 4 byte 오프셋)
    float* unaligned = aligned + 1;  // 4 bytes 오프셋 = 비정렬

    for (int i = 0; i < N; i++) aligned[i] = (float)i;

    __m256 vsum = _mm256_setzero_ps();
    const int REPS = 50;
    struct timespec t0, t1;

    // 정렬 로드
    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int r = 0; r < REPS; r++)
        for (int i = 0; i < N; i += 8)
            vsum = _mm256_add_ps(vsum, _mm256_load_ps(aligned + i));
    clock_gettime(CLOCK_MONOTONIC, &t1);
    printf("Aligned:   %.2f ms\n",
        ((t1.tv_sec-t0.tv_sec)*1e9 + t1.tv_nsec-t0.tv_nsec) / 1e6 / REPS);

    // 비정렬 로드 (4 bytes off = 캐시라인 분리 없음)
    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int r = 0; r < REPS; r++)
        for (int i = 0; i < N; i += 8)
            vsum = _mm256_add_ps(vsum, _mm256_loadu_ps(unaligned + i));
    clock_gettime(CLOCK_MONOTONIC, &t1);
    printf("Unaligned: %.2f ms\n",
        ((t1.tv_sec-t0.tv_sec)*1e9 + t1.tv_nsec-t0.tv_nsec) / 1e6 / REPS);

    // 결과:
    // Aligned:   ~5.0 ms
    // Unaligned: ~5.1 ms  ← 캐시라인 분리 없으면 거의 동일!
    // (캐시라인 경계 걸치는 경우는 훨씬 느림: ~8~10 ms)

    _mm_free(aligned);
    return 0;
}
```

---

## 📊 성능 비교

```
내적 연산 N=1M float (L3 캐시 내 기준):

구현 방식                명령 수(/iter)  시간(예상)   배율
──────────────────────────────────────────────────────────
스칼라 (addss+mulss)      2M              ~3.2ms      1×
SSE (mulps+addps, 4개)    250K×2          ~0.85ms     3.8×
AVX2 (vmulps+vaddps)      125K×2          ~0.45ms     7.1×
AVX2+FMA (vfmadd, 4 acc)  32K×1           ~0.38ms     8.4×
AVX-512+FMA (16개/명령)   16K×1           ~0.22ms     14.5×

누산기 수와 처리량:
  누산기 1개 (의존성 사슬, FMA latency 4cy):
    처리량 병목 = 4cy / (FMA Throughput 0.5cy) = 8× 손실
    실제: 이론 최대의 12% 활용

  누산기 4개 이상 (의존성 끊음):
    FMA latency 4cy, 4개면 파이프라인 완전 채움
    → Throughput 0.5cy 가능 = 최대 처리량 도달

수평 합 방법 비교:
  메모리 경유 (storeu + scalar loop):  ~15 cy
  hadd 2회 + movehdup:                ~8 cy
  extractf128 + addps + hadd×2:      ~6 cy (권장)

로드 비용 (Skylake, L1 히트 기준):
  _mm256_load_ps (정렬):     4 cy 지연, 0.5 cy 처리량
  _mm256_loadu_ps (비정렬):  4 cy 지연, 0.5 cy 처리량 (캐라인 분리 없을 때)
  비정렬 + 캐라인 분리:       ~8 cy 지연
```

---

## ⚖️ 트레이드오프

```
수동 인트린식 사용 결정 기준:

사용이 적합한 경우:
  ✅ 자동 벡터화가 실패하거나 최적 명령을 생성하지 못하는 경우
  ✅ 수평 연산, 셔플, 특수 변환이 필요한 경우
  ✅ 성능이 가장 중요한 단 하나의 핫 루프 (전체 코드의 1~5%)
  ✅ 컴파일러 버전 무관 확정적 SIMD 생성 필요

피해야 하는 경우:
  ❌ 코드베이스 전체에 인트린식 도배 → 유지보수 불가능
  ❌ 자동 벡터화로 충분한 경우 (Ch5-02 먼저 시도)
  ❌ 크로스 플랫폼 코드 (ARM NEON 등 별도 작성 필요)
  ❌ 팀 모두가 인트린식을 이해하지 못하는 환경

대안 접근:
  ① Highway 라이브러리 (Google):
     플랫폼 추상화 SIMD C++ 라이브러리
     hwy::VFromD<float, D> = 플랫폼별 최적 레지스터
     코드 한 번 작성 → SSE/AVX2/AVX-512/NEON 자동 선택

  ② xtensor / xsimd:
     C++ 과학 계산용 SIMD 추상화
     NumPy 스타일 인터페이스 + SIMD 가속

  ③ OpenMP SIMD:
     #pragma omp simd
     GCC/Clang에서 명시적 벡터화 힌트 (플랫폼 독립)

FMA 사용 시 주의:
  -mfma 없이 컴파일하면 → _mm256_fmadd_ps가 illegal instruction (SIGILL)
  → CMakeLists.txt: target_compile_options(-mavx2 -mfma)
  → 또는 런타임 CPUID 체크 후 FMA 코드 선택

AVX-512 전환 비용:
  AVX-512 명령 첫 실행 시 CPU 주파수 다운클럭 (~100~300MHz)
  → 짧은 AVX-512 블록이 주변 코드까지 느리게 만듦
  → 전체 핫 루프를 AVX-512로 전환할 때만 효과적
```

---

## 📌 핵심 정리

```
수동 SIMD 인트린식 핵심:

명명 규칙:
  _mm{폭}_{연산}_{타입}
  _mm256_fmadd_ps(a,b,c) → YMM, FMA(a*b+c), float

로드 선택:
  _mm256_load_ps:   32-byte 정렬 전용, 위반 시 fault
  _mm256_loadu_ps:  비정렬 OK, Haswell+ 캐라인 분리 없으면 동일 성능
  → 일반적으로 loadu 사용 권장, 정렬 보장 시 load

FMA 우선:
  vmulps + vaddps(2 명령) → vfmadd(1 명령)으로 2배 처리량
  정밀도도 향상 (반올림 1회 vs 2회)
  -mavx2 -mfma 컴파일 옵션 필수

수평 합 패턴:
  extractf128 → addps → movehdup → addps → movehl → addss
  → 5~6 명령으로 8 float → 1 float 환원
  → 메모리 경유보다 2~3배 빠름

VZEROUPPER:
  YMM 사용 후 XMM(SSE) 코드 호출 전 반드시 삽입
  컴파일러가 대부분 자동 삽입 (-O2 이상)
  수동 인라인 어셈블리나 복잡한 흐름에서는 수동 호출

누산기 4개 규칙:
  FMA latency = 4cy → 4개 누산기로 파이프라인 완전 활용
  누산기 1개 = FMA 처리량의 12%만 활용
  누산기 4개 이상 = FMA 처리량 100% 도달 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `_mm256_fmadd_ps(a, b, c)`와 `_mm256_add_ps(_mm256_mul_ps(a, b), c)` 두 코드의 성능 차이를 만드는 요소 두 가지를 설명하라. 수치적으로도 결과가 다를 수 있는 이유는?

<details>
<summary>해설 보기</summary>

**성능 차이 요소**:

1. **명령 수**: `vmulps` + `vaddps` = 2 명령, `vfmadd231ps` = 1 명령. 처리량이 각각 0.5 cy/명령이라면 2명령 = 1 cy, FMA = 0.5 cy → **2배 처리량 차이**.

2. **임시 레지스터**: `vmulps + vaddps`는 중간 결과를 레지스터에 저장해야 하므로 레지스터 1개 추가 소비. 핫 루프에서 레지스터 압박 시 스필(spill) 발생 → 추가 메모리 왕복.

**수치 차이**: 부동소수점 연산은 반올림이 발생합니다. `vmulps + vaddps`는 곱셈 후 1회 반올림, 덧셈 후 1회 반올림 = **총 2회 반올림**. `vfmadd`는 내부적으로 무한 정밀도 중간 결과를 유지하다 최종 1회만 반올림 = **1회 반올림**. 결과가 IEEE 754 기준 다를 수 있고, FMA 쪽이 더 정확합니다. 이 차이가 `-ffast-math` 없이도 허용되는 이유는 FMA가 별도 IEEE 754 표준(IEEE 754-2008)으로 정의되었기 때문입니다.

</details>

---

**Q2.** `hsum_avx2` 구현에서 `_mm256_hadd_ps`만 반복해서 사용하지 않고 `extractf128 + addps`를 먼저 하는 이유는? `hadd` 2회만으로 구현하면 어떤 문제가 있는가?

<details>
<summary>해설 보기</summary>

**`hadd`의 구조적 제약**:

`_mm256_hadd_ps(a, a)`는 **128bit 레인 내에서만** 인접 쌍을 합산합니다. 즉 상위 128bit와 하위 128bit가 독립적으로 처리되어 서로 섞이지 않습니다.

```
입력 vsum: [a0, a1, a2, a3 | a4, a5, a6, a7]  (| = 128bit 경계)
hadd 1회: [a0+a1, a2+a3, a0+a1, a2+a3 | a4+a5, a6+a7, a4+a5, a6+a7]
hadd 2회: [a01+a23, ..., ... | a45+a67, ...]
```

두 번 hadd 후에도 하위 레인의 합(`a0+a1+a2+a3`)과 상위 레인의 합(`a4+a5+a6+a7`)이 분리되어 있습니다. 최종 합산을 위해서는 결국 `extractf128`나 `permute2f128`로 두 레인을 합쳐야 합니다.

`extractf128 + addps`를 먼저 쓰면 8개를 4개로 한번에 줄이므로 이후 `hadd` 횟수가 줄어들고 레인 경계 문제도 없습니다. `hadd`의 처리량도 상대적으로 낮으므로(0.5~1 cy) 사용 횟수를 최소화하는 것이 효율적입니다.

</details>

---

**Q3.** 다음 FMA 루프에서 누산기가 1개일 때와 4개일 때 실제 처리량 차이를 Skylake의 FMA 명령 latency/throughput 수치로 계산하라.

```c
// 누산기 1개
__m256 s = _mm256_setzero_ps();
for (int i = 0; i < N; i += 8)
    s = _mm256_fmadd_ps(_mm256_loadu_ps(a+i), _mm256_loadu_ps(b+i), s);
```

<details>
<summary>해설 보기</summary>

**Skylake FMA 수치**: `vfmadd231ps ymm` — Latency: **4 cy**, Throughput: **0.5 cy** (2 포트 실행 가능).

**누산기 1개 분석**:
- 각 반복에서 `s = fmadd(va, vb, s)` — 현재 `s`가 이전 반복의 `s`에 의존
- 의존성 사슬 길이 = FMA latency = **4 cy/반복**
- FMA 처리량 0.5 cy/반복이지만 의존성 때문에 4 cy/반복에 제한됨
- CPU 활용률: 0.5 / 4 = **12.5%**

**누산기 4개 분석**:
- `s0, s1, s2, s3`이 각각 독립 사슬 → 4개 FMA를 파이프라인에 겹쳐서 실행 가능
- 4개 × latency 4 cy = 4 반복에 걸쳐 파이프라인 채움
- 실제 처리량: **0.5 cy/반복** (이론 최대 도달)
- CPU 활용률: **100%**
- 가속비: 4 cy / 0.5 cy = **8배 차이**

실측: N=1M float 기준 누산기 1개 ~2.0ms, 누산기 4개 ~0.4ms ≈ **5배 차이** (메모리 대역폭 제한으로 이론 8배에 미달).

</details>

---

<div align="center">

**[⬅️ 이전: 자동 벡터화](./02-auto-vectorization.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데이터 레이아웃과 벡터화 ➡️](./04-data-layout-for-simd.md)**

</div>
