# SIMD 기본 — SSE/AVX 레지스터와 한 명령 다중 데이터

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- XMM(128bit) / YMM(256bit) / ZMM(512bit) 레지스터가 각각 몇 개 요소를 동시에 처리하는가?
- `paddd`와 `vmulps` 같은 SIMD 명령이 스칼라 명령과 비교해 왜 N배 빠른가?
- SIMD "폭"이 처리량에 미치는 영향을 공식으로 계산하는 방법은?
- 부동소수점 SIMD(SSE2)가 정수 SIMD(MMX)보다 늦게 등장한 이유는?
- 같은 AVX2 명령어라도 정수 연산과 부동소수점 연산의 지연시간이 다른 이유는?
- SIMD가 처리량(throughput)을 늘리지만 단발 지연시간(latency)을 줄이지 않는 근본 이유는?

---

## 🔍 왜 이 개념이 중요한가

### "컴파일러가 알아서 최적화한다고?"

```
스칼라 덧셈 루프:
  for (int i = 0; i < 8; i++)
      c[i] = a[i] + b[i];

  CPU가 실행하는 방식 (스칼라):
    add eax, [a+0]  → 1 요소
    add eax, [a+4]  → 1 요소
    ...              8번 반복

SIMD 덧셈 명령 1개:
  vpaddd ymm0, ymm1, ymm2   ← AVX2, 256bit
    → 8개 int를 한 명령에 처리
    → 명령 수: 8개 → 1개
    → 이론 처리량: 8배 향상

왜 중요한가:
  이미지 처리 (1920×1080 = 2,073,600 픽셀 × RGBA = 8,294,400 byte)
    → 스칼라: 8M 번 연산
    → SIMD (AVX2, byte): 8M / 32 = 262,144 번 연산
    → 이론 32배 가속

  딥러닝 추론 (행렬 곱, 합성곱)
  오디오 DSP (필터, FFT)
  물리 시뮬레이션 (파티클 업데이트)
  — 전부 SIMD가 성능의 핵심
```

---

## 😱 잘못된 이해

### Before: "SIMD는 특수 명령어라서 내 코드에는 자동으로 사용된다"

```
잘못된 이해:
  컴파일러가 -O2 이상이면 모든 루프에 SIMD 사용
  → "내 코드는 이미 최적화됨"

실제로 놓치는 것:
  1. 벡터화 조건이 엄격함
     - 루프 반복 횟수가 컴파일 타임에 알려져야 함
     - 포인터가 서로 겹치지 않아야 함 (aliasing 없음)
     - 루프 안에 분기(if/break)가 없어야 함
     - 루프 반복이 서로 독립적이어야 함 (의존성 사슬 없음)

  2. 레지스터 크기를 모르면 처리량 계산 불가
     - int 배열을 SSE2로: 4개/명령 (128bit / 32bit)
     - int 배열을 AVX2로: 8개/명령 (256bit / 32bit)
     - float 배열을 AVX-512로: 16개/명령 (512bit / 32bit)
     - 차이: 2배, 4배

  3. 데이터 레이아웃이 SIMD를 막는 경우가 많음
     struct { float x, y, z; } particles[N];   ← AoS: SIMD 로드 불편
     float x[N], y[N], z[N];                   ← SoA: SIMD 로드 완벽
```

---

## ✨ 올바른 이해

### After: SIMD는 레지스터 폭을 데이터에 매핑하는 정밀한 도구

```
레지스터 계층:

  XMM (128bit = 16 bytes):
    4× float(32bit)     / 4× int(32bit)
    2× double(64bit)    / 2× int64(64bit)
    16× int8(8bit)      / 8× int16(16bit)
    ↑ SSE / SSE2 / SSE4.2 (1999~2007년 등장)

  YMM (256bit = 32 bytes):
    8× float(32bit)     / 8× int(32bit)
    4× double(64bit)    / 4× int64(64bit)
    32× int8(8bit)      / 16× int16(16bit)
    ↑ AVX / AVX2 (2011~2013년 등장)

  ZMM (512bit = 64 bytes):
    16× float(32bit)    / 16× int(32bit)
    8× double(64bit)    / 8× int64(64bit)
    64× int8(8bit)      / 32× int16(16bit)
    ↑ AVX-512 (2016년~, Xeon/서버급 우선)

처리량 공식:
  이론 처리량 향상 = 레지스터 폭 / 요소 크기
  float AVX2:  256 / 32 = 8배
  int8 AVX2:   256 / 8  = 32배
  double AVX2: 256 / 64 = 4배

  단, 실제 처리량은 메모리 대역폭·명령 처리량(throughput)·의존성에 제한됨
```

---

## 🔬 내부 동작 원리

### 1. 레지스터 구조 — 128 / 256 / 512 bit 물리적 실체

```
ZMM0 (512bit):
┌────────────────────────────────────────────────────────────────┐
│                         ZMM0 (64 bytes)                        │
├────────────────────────────┬───────────────────────────────────┤
│       YMM0 (32 bytes)      │         (상위 32 bytes)           │
├─────────────┬──────────────┤                                   │
│ XMM0(16B)   │ (상위 16B)   │                                   │
└─────────────┴──────────────┴───────────────────────────────────┘

레지스터 수:
  XMM0~XMM15: 16개 (x86-64)
  YMM0~YMM15: 16개 (AVX, XMM과 물리적으로 겹침)
  ZMM0~ZMM31: 32개 (AVX-512)

AVX 전환 비용 (VZEROUPPER):
  SSE 명령(XMM) → AVX 명령(YMM) 전환 시 상위 128bit 오염 방지 필요
  _mm256_zeroupper() / VZEROUPPER 명령으로 YMM 상위 비트 초기화
  → 이를 누락하면 False Dependency로 성능 저하
  → 컴파일러가 자동 삽입하지만 인트린식 혼용 시 주의
```

### 2. 대표 명령어 해부 — `paddd` vs `vpaddd` vs `vpaddd ymm`

```asm
; 스칼라 정수 덧셈
add    eax, ebx         ; 32bit 1개 처리

; SSE2: XMM (128bit, int32 × 4)
paddd  xmm0, xmm1       ; 4개 int32 동시 덧셈
; xmm0[0]+xmm1[0], xmm0[1]+xmm1[1], xmm0[2]+xmm1[2], xmm0[3]+xmm1[3]

; AVX2: YMM (256bit, int32 × 8) — VEX 프리픽스 붙음
vpaddd ymm0, ymm1, ymm2  ; 8개 int32 동시 덧셈
; 3-operand 형식: ymm0 = ymm1 + ymm2

; AVX2: YMM (256bit, float32 × 8)
vmulps ymm0, ymm1, ymm2  ; 8개 float32 동시 곱셈

; AVX-512: ZMM (512bit, float32 × 16)
vmulps zmm0, zmm1, zmm2  ; 16개 float32 동시 곱셈

; FMA (Fused Multiply-Add): a * b + c 를 1 명령으로
vfmadd231ps ymm0, ymm1, ymm2  ; ymm0 += ymm1 * ymm2
; 8개 float의 a*b+c를 1 명령에, 오차도 더 작음 (반올림 1회)
```

### 3. 부동소수점 SIMD가 정수 SIMD보다 늦게 등장한 배경

```
역사:
  1997년 — MMX (64bit, int8/16/32): 최초의 SIMD
    → 부동소수점 없음, 정수 전용
    → 실제론 x87 FPU 레지스터 재사용 → 상태 전환 비용 큼

  1999년 — SSE (128bit): 처음으로 부동소수점 SIMD
    → 전용 XMM 레지스터 128bit × 8개
    → float32만 지원 (double 없음)
    → 이유: 트랜지스터 예산 부족, 부동소수점 하드웨어는 더 복잡함

  2001년 — SSE2: double(64bit) SIMD 추가
    → int8~int64 모든 정수 타입도 SIMD

왜 부동소수점이 더 복잡했나:
  ① 하드웨어 면적:
     정수 덧셈 = 단순 가산기 회로
     부동소수점 덧셈 = 지수 정렬 + 가수 덧셈 + 정규화 + 반올림
     → 4배 이상 복잡한 회로 × 4 lane = 넓은 면적

  ② 정밀도 표준 (IEEE 754):
     반올림 모드(4가지), NaN/Inf/Denormal 처리
     SIMD로도 IEEE 754 완전 준수를 요구
     → 예외 처리 로직을 lane 개수만큼 복제

  ③ 실무 필요성 타임라인:
     MMX 시대: 멀티미디어(음악/이미지)는 정수 연산 우선
     SSE 시대: 3D 게임 폭발 → float 행렬·벡터 연산 필수
```

### 4. 지연시간(Latency) vs 처리량(Throughput) — SIMD의 진짜 의미

```
Intel Skylake (Agner Fog 테이블 기준):

명령어              Latency   Throughput(역수)  해석
─────────────────────────────────────────────────────
add eax, ebx           1         0.25          스칼라 정수 덧셈
paddd xmm0, xmm1       1         0.33          SSE 정수 덧셈 (4 lane)
vpaddd ymm0,ymm1,ymm2  1         0.33          AVX2 정수 덧셈 (8 lane)
addss xmm0, xmm1       4         0.5           스칼라 float 덧셈
addps xmm0, xmm1       4         0.5           SSE float 덧셈 (4 lane)
vaddps ymm0,ymm1,ymm2  4         0.5           AVX2 float 덧셈 (8 lane)
vmulps ymm0,ymm1,ymm2  4         0.5           AVX2 float 곱셈 (8 lane)
vdivps ymm0,ymm1,ymm2  11~14     5~14          AVX2 float 나눗셈 (8 lane)

핵심 관찰:
  addps(4 lane) vs vaddps(8 lane): Latency 동일 (4 cy), Throughput 동일!
  → 레지스터가 2배 넓어졌지만 1 명령에 같은 시간 소요
  → 처리량(data/cycle): 정확히 2배
  → 단발 지연시간: 변화 없음 (1개 덧셈도 4cy, 8개 덧셈도 4cy)

결론:
  SIMD는 "더 빠른" 연산이 아님
  SIMD는 "같은 시간에 더 많은" 연산 = 처리량 향상
  단발 지연시간에 민감한 코드 (짧은 배열, 의존성 사슬)에는 이점 없음
```

### 5. 레지스터 용량과 실제 처리량 한계 계산

```
예제: N=1024 개 float 배열의 덧셈

스칼라 처리:
  1024 회 × addss (Throughput 0.5 cy/명령)
  = 1024 × 0.5 = 512 cycle (메모리 무시)

SSE (addps, 4 float/명령):
  1024 / 4 = 256 회 명령
  256 × 0.5 = 128 cycle → 4배

AVX2 (vaddps, 8 float/명령):
  1024 / 8 = 128 회 명령
  128 × 0.5 = 64 cycle → 8배

AVX-512 (vaddps zmm, 16 float/명령):
  1024 / 16 = 64 회 명령
  64 × 0.5 = 32 cycle → 16배

현실 제약:
  ① 메모리 로드 처리량: L1 캐시 대역폭 한계
     Skylake: 32 bytes/cycle (256bit 로드 2개/cycle)
     AVX2 float 8개 = 32 bytes → 로드 1 사이클, 연산 0.5 사이클
     → 메모리 바운드! 연산 처리량이 아닌 로드 처리량이 병목

  ② AVX-512의 Frequency Throttling:
     AVX-512 명령 실행 시 CPU 클럭 다운클럭 (~100~300MHz 감소)
     → 이론 2배 처리량이 실제 1.2~1.5배에 그치는 경우 있음
     → 서버 워크로드에서는 혼용 코드의 성능 저하에 주의
```

---

## 💻 실전 실험

### 실험 1: godbolt에서 SIMD 레지스터 사용 확인

```c
// simd_demo.c — godbolt.org에서 GCC 13.2 / -O2 -mavx2 로 확인
#include <immintrin.h>

// 스칼라 버전
void add_scalar(float* c, const float* a, const float* b, int n) {
    for (int i = 0; i < n; i++)
        c[i] = a[i] + b[i];
}

// 수동 AVX2 버전
void add_avx2(float* c, const float* a, const float* b, int n) {
    int i;
    for (i = 0; i <= n - 8; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);
        __m256 vb = _mm256_loadu_ps(b + i);
        __m256 vc = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(c + i, vc);
    }
    for (; i < n; i++)  // 나머지 처리
        c[i] = a[i] + b[i];
}
```

```bash
# godbolt 확인 URL:
# https://godbolt.org/z/... (GCC 13.2, -O2 -mavx2)

# 로컬 컴파일:
gcc -O2 -mavx2 -S -masm=intel simd_demo.c -o simd_demo.s

# add_scalar의 어셈블리 (-O2 활성화 시 자동 벡터화):
# .L3:
#   vmovups  ymm1, YMMWORD PTR [rsi+rax]     ← 8 float 로드
#   vaddps   ymm0, ymm1, YMMWORD PTR [rdx+rax] ← 8 float 덧셈
#   vmovups  YMMWORD PTR [rdi+rax], ymm0     ← 8 float 저장
#   add      rax, 32
#   cmp      rax, rcx
#   jne      .L3

# 자동 벡터화 성공! vaddps ymm (256bit = 8 float)
```

### 실험 2: perf로 벡터 명령어 실행 횟수 측정

```bash
# avx2_bench.c 컴파일
gcc -O2 -mavx2 -o avx2_bench avx2_bench.c

# 벡터 명령어 실행 횟수 측정 (PMU 이벤트)
perf stat -e cycles,instructions,\
fp_arith_inst_retired.256b_packed_single,\
fp_arith_inst_retired.scalar_single \
./avx2_bench

# 예상 출력 (N=1024 float, 1M 반복):
# 1,280,000,000   cycles
# 2,100,000,000   instructions
#   128,000,000   fp_arith_inst_retired.256b_packed_single   ← YMM 사용!
#             0   fp_arith_inst_retired.scalar_single        ← 스칼라 0
# → 256bit 명령만 실행됨 = 자동 벡터화 성공
```

### 실험 3: 스칼라 vs SSE vs AVX2 처리량 직접 비교

```c
// throughput_bench.c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <immintrin.h>

#define N (1 << 22)   // 4M float = 16MB

float a[N], b[N], c[N];

void run_scalar() {
    for (int i = 0; i < N; i++)
        c[i] = a[i] + b[i];
}

void run_sse() {   // SSE: 4 float/명령
    for (int i = 0; i < N; i += 4) {
        __m128 va = _mm_loadu_ps(a + i);
        __m128 vb = _mm_loadu_ps(b + i);
        _mm_storeu_ps(c + i, _mm_add_ps(va, vb));
    }
}

void run_avx2() {  // AVX2: 8 float/명령
    for (int i = 0; i < N; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);
        __m256 vb = _mm256_loadu_ps(b + i);
        _mm256_storeu_ps(c + i, _mm256_add_ps(va, vb));
    }
}

static long now_ns() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000000000L + ts.tv_nsec;
}

int main() {
    for (int i = 0; i < N; i++) { a[i] = (float)i; b[i] = (float)(N-i); }

    const int REPS = 100;
    long t0, t1;

    t0 = now_ns(); for (int r = 0; r < REPS; r++) run_scalar(); t1 = now_ns();
    printf("Scalar: %.1f ms\n", (t1-t0)/1e6/REPS);

    t0 = now_ns(); for (int r = 0; r < REPS; r++) run_sse();    t1 = now_ns();
    printf("SSE:    %.1f ms\n", (t1-t0)/1e6/REPS);

    t0 = now_ns(); for (int r = 0; r < REPS; r++) run_avx2();   t1 = now_ns();
    printf("AVX2:   %.1f ms\n", (t1-t0)/1e6/REPS);
    return 0;
}
```

```bash
gcc -O1 -mno-avx -msse2 -o bench_sse throughput_bench.c   # SSE 강제
gcc -O1 -mavx2 -o bench_avx2 throughput_bench.c            # AVX2 사용
# -O1: 자동 벡터화 비활성화 (수동 인트린식만 측정)

# 예상 결과 (3.5GHz CPU, L3 히트 기준):
# Scalar: ~8.0 ms   (기준)
# SSE:    ~2.1 ms   (3.8배, 이론 4배에 근접)
# AVX2:   ~1.1 ms   (7.3배, 이론 8배에 근접)
# → 메모리 대역폭 제한으로 이론값에 완전히 도달하지 못함
```

---

## 📊 성능 비교

```
N=4M float 배열 덧셈 (L3 캐시 내 데이터 기준):

방식                레지스터    요소/명령   시간(예상)   이론 배율   실측 배율
──────────────────────────────────────────────────────────────────────────
스칼라 (addss)      32bit       1           ~8.0ms       1×          1×
SSE2   (addps)      128bit      4           ~2.1ms       4×          3.8×
AVX2   (vaddps)     256bit      8           ~1.1ms       8×          7.3×
AVX-512 (vaddps)    512bit      16          ~0.6ms       16×         13.3×

메모리 대역폭이 병목일 때:
  Skylake L3: ~40 GB/s, float 4B × 4M × 3(로드2+저장1) = 48MB → ~1.2ms 한계
  → AVX-512도 메모리 바운드에서는 AVX2와 큰 차이 없음

연산 집약적 워크로드 (L1 재사용):
  행렬 곱: flops/byte 비율 높음 → AVX-512가 AVX2 대비 ~1.8배 실측
  → compute-bound → SIMD 폭이 직접 처리량에 기여

부동소수점 vs 정수 처리량:
  vaddps  ymm (8 float):    Throughput 0.5 cy
  vpaddd  ymm (8 int32):    Throughput 0.33 cy
  → 정수 덧셈이 부동소수점보다 처리량 높음 (1.5배)
  → 이유: 정수 가산기가 더 단순 → 실행 포트 더 많이 배정
```

---

## ⚖️ 트레이드오프

```
SIMD 선택 기준:

레지스터 폭 결정:
  ✅ 타겟 하드웨어가 AVX2 지원하면 YMM (대부분의 2013년 이후 PC/서버)
  ✅ AVX-512는 Skylake-X, Ice Lake 이후 서버급; 소비자 CPU는 제한적
  ❌ AVX-512 전용 코드를 구형 서버에 배포하면 SIGILL(불법 명령) 폭사
  → 런타임 CPU 감지(CPUID) + 분기 dispatch 필요

데이터 타입 선택:
  int8/uint8 연산: AVX2에서 32개/명령 → 이미지 처리에 강력
  float32:         8개/명령 → 딥러닝 추론, 물리 시뮬레이션
  float64:         4개/명령 → 과학 계산 (정밀도 필요 시)
  → 정밀도가 허용되면 float32를 float64 대신 선택 → 2배 처리량

자동 vs 수동 벡터화:
  ✅ 자동 벡터화(-O2/-O3): 조건 충족 시 컴파일러가 처리 → 유지보수 쉬움
  ✅ 수동 인트린식: 자동 실패 시, 특수 명령(FMA·수평합) 필요 시
  ❌ 인트린식 코드는 가독성 낮고 플랫폼 종속
  → 먼저 자동 벡터화 시도, 실패하면 수동 (Ch5-02/03 참조)

AVX-512 전환 비용:
  짧은 AVX-512 코드 블록 → 클럭 다운 발생 → 주변 코드도 느려짐
  → 전체 핫 루프가 AVX-512일 때만 이득
  → 짧은 루프에 AVX-512: 손해일 수 있음
```

---

## 📌 핵심 정리

```
SIMD 핵심 개념:

레지스터 폭 계층:
  XMM (128bit) → SSE/SSE2/SSE4.2: float 4개, int32 4개
  YMM (256bit) → AVX/AVX2:         float 8개, int32 8개
  ZMM (512bit) → AVX-512:          float 16개, int32 16개

처리량 공식:
  이론 처리량 향상 = 레지스터 폭 / 요소 크기 (비트)
  AVX2 float32: 256/32 = 8배  ← 실측 6~7배 (메모리·실행 포트 한계)
  AVX2 int8:    256/8  = 32배 ← 실측 20~25배

대표 명령:
  vpaddd ymm  : int32 × 8 동시 덧셈
  vmulps ymm  : float32 × 8 동시 곱셈
  vfmadd231ps : float32 × 8 동시 FMA (a*b+c)

부동소수점이 늦게 등장한 이유:
  회로 복잡도 (IEEE 754 완전 준수 = 정수 4배 트랜지스터)
  실용 수요 (3D 게임 폭발 전까지 FP SIMD 필요 없었음)

SIMD의 한계:
  처리량(throughput) 증가 ← SIMD가 개선
  단발 지연시간(latency) 무변화 ← SIMD로 줄지 않음
  메모리 대역폭 병목 시 이론 배수에 미달
```

---

## 🤔 생각해볼 문제

**Q1.** `double` 배열에 AVX2를 사용할 때와 `float` 배열에 AVX2를 사용할 때, 이론 처리량이 몇 배 차이나는가? 실제 측정에서 그 차이가 이론보다 작거나 클 수 있는 이유는?

<details>
<summary>해설 보기</summary>

**이론 처리량 차이**: double(64bit)은 YMM에 4개, float(32bit)은 YMM에 8개 → **2배 차이**입니다.

실제 측정이 이론과 다른 이유:

1. **메모리 바운드 워크로드**: 단순 배열 덧셈처럼 메모리 대역폭이 병목인 경우, `vaddpd` vs `vaddps`의 차이보다 메모리 로드 속도가 먼저 한계에 닿습니다. double 배열은 2배 큰 메모리를 사용하므로 메모리 대역폭 소비도 2배 → 결국 비슷한 시간이 걸릴 수 있습니다.

2. **연산 집약적 워크로드**: 행렬 곱처럼 캐시에서 데이터를 재사용하는 경우에는 이론값인 2배 차이가 실측에서도 나타납니다.

3. **실행 포트 처리량**: `vaddpd ymm`과 `vaddps ymm`의 처리량이 동일(0.5 cy)이므로 연산 처리량만 보면 정확히 2배입니다.

결론: 메모리 바운드 → 이론보다 차이 작음 / 연산 바운드 → 이론에 근접.

</details>

---

**Q2.** 다음 코드에서 `sum`을 구할 때 AVX2를 사용할 수 있는가? 있다면 방법은, 없다면 이유는?

```c
float sum = 0;
for (int i = 0; i < N; i++)
    sum += a[i];
```

<details>
<summary>해설 보기</summary>

**직접적인 SIMD 적용은 불가능**합니다. `sum += a[i]`는 이전 `sum` 값에 의존하는 **의존성 사슬(dependency chain)**이라 각 반복이 독립적이지 않기 때문입니다.

**우회 방법 — 부분합 + 수평 합산**:

```c
// AVX2 부분합: 8개 채널에 각각 누적
__m256 vsum = _mm256_setzero_ps();
for (int i = 0; i <= N - 8; i += 8) {
    __m256 va = _mm256_loadu_ps(a + i);
    vsum = _mm256_add_ps(vsum, va);
}
// 수평 합산 (horizontal sum): 8 → 1 스칼라
// (자세한 구현은 Ch5-03 참조)
float result[8];
_mm256_storeu_ps(result, vsum);
float sum = 0;
for (int i = 0; i < 8; i++) sum += result[i];
```

이 기법은 8개의 독립적인 누산기를 병렬 실행하므로 AVX2의 8× 처리량을 활용합니다. 의존성 사슬을 끊는 것은 SIMD와 무관하게 ILP(명령어 수준 병렬성) 최적화의 기본 패턴이기도 합니다 (Ch1-06 참조).

</details>

---

**Q3.** 같은 AVX2 명령어(`vaddps ymm`)로 float 8개를 처리할 때, 정렬된 메모리(32-byte aligned)와 비정렬 메모리(unaligned)의 성능 차이는 현대 CPU(Haswell 이후)에서 어느 정도인가?

<details>
<summary>해설 보기</summary>

**Haswell(2013년) 이후 대부분의 경우 차이 없음**입니다.

구분:
- `vmovaps` (aligned) vs `vmovups` (unaligned): Haswell부터 L1 히트 시 지연시간·처리량 동일
- 단, **캐시라인 경계를 넘는(split) 비정렬 접근**은 여전히 비용 발생 (약 2배 지연)

실무 지침:
- 32-byte 경계 정렬을 지키면 확실히 split 미발생 → `_mm_malloc(size, 32)` 또는 `alignas(32)`
- 정렬 보장 불가 시 `vmovups` 사용 + 성능 측정으로 확인
- 상세 비용은 Ch5-03 수동 인트린식 문서에서 다룹니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: 측정으로 증명](../abstraction-cost/05-proof-by-measurement.md)** | **[홈으로 🏠](../README.md)** | **[다음: 자동 벡터화 ➡️](./02-auto-vectorization.md)**

</div>
