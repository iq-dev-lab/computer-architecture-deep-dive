# 루프라인 모델 — compute-bound vs memory-bound

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 루프라인(Roofline) 모델의 두 축은 무엇이고, 각각 무엇을 나타내는가?
- 연산 강도(Arithmetic Intensity, FLOPs/byte)를 직접 계산하는 방법은?
- "내 코드가 memory-bound인지 compute-bound인지"를 숫자로 어떻게 판정하는가?
- 머신의 메모리 대역폭 한계와 연산 한계는 어떻게 측정하는가?
- `likwid-bench`와 `likwid-perfctr`을 활용해 연산 강도를 측정하는 방법은?
- 최적화 후 성능이 올랐지만 루프라인 상한에 못 미친다면 다음 병목은 어디인가?
- 루프라인 모델이 단일 스레드 SIMD 성능 분석에 어떻게 사용되는가?

---

## 🔍 왜 이 개념이 중요한가

### "캐시를 다 고쳤는데 왜 아직도 느리지?"

```
최적화 후에도 성능이 기대에 못 미치는 상황:

Before:
  행렬 연산 코드, 지역성 최악 (열 우선 순회)
  실행 시간: 8,000 ms
  cache-miss: 95%

After 1 (지역성 개선):
  AoS→SoA, 캐시 블로킹 적용
  실행 시간: 1,200 ms
  cache-miss: 8% ← 크게 개선

"다 고쳤는데 왜 1,200ms지? 이론적으로 더 빨라야 하지 않나?"

루프라인 모델이 답을 준다:
  이 코드의 연산 강도 = 2 FLOPs/byte
  머신의 메모리 대역폭 = 40 GB/s
  이론적 성능 상한 = 2 × 40 = 80 GFLOPs/s

  실제 성능 = 66 GFLOPs/s (상한의 82.5%)
  → 메모리 대역폭 한계에 이미 거의 다 붙어있다!
  → 메모리 접근이 있는 한 여기서 더 빠를 수 없다

  "더 최적화하려면? 캐시 블로킹으로 재사용률 높여서
   연산 강도를 2 → 8 FLOPs/byte로 올려야 한다"

루프라인 모델 없이는 "어디까지 빠르게 할 수 있는가"를
  정량적으로 알 수 없다.
```

---

## 😱 잘못된 이해

### Before: "더 최적화하면 반드시 빨라진다"

```
잘못된 사고:
  성능 목표 = 현재 성능의 2배
  → 계속 최적화하면 언젠가 달성할 수 있다

  또는:
  "이 코드가 DRAM을 많이 쓰는 것 같으니 알고리즘을 바꾸면 빨라진다"
  → 실제로는 알고리즘 변경 없이 DRAM 접근 자체를 줄이지 않으면 한계 존재

놓치는 것:
  하드웨어 자체의 물리적 한계가 있다.

  메모리 대역폭 한계 (예: DDR4-3200 dual-channel = 51 GB/s):
    byte당 1 FLOP 연산 → 최대 51 GFLOPs/s
    여기서 더 빠르려면? → DRAM을 덜 읽어야 함 (캐시 재사용)

  연산 한계 (예: AVX2 FMA 피크 = 512 GFLOPs/s):
    연산 강도가 충분히 높으면 메모리가 아닌 연산 유닛이 병목

  루프라인 모델은 이 두 한계를 한 그래프에 그려서
  "현재 내 코드가 어느 한계에 붙어있는가"를 보여준다

  한계에 붙어있으면: 지금이 (이 알고리즘 구조에서) 최적
  한계보다 훨씬 아래면: 아직 최적화 여지가 있음
```

---

## ✨ 올바른 이해

### After: 루프라인은 "성능의 천장"을 수치로 보여준다

```
루프라인 모델의 두 축:

  Y축 (세로): 성능 [GFLOPs/s]
    - 초당 수행한 부동소수점 연산 수
    - 높을수록 좋음

  X축 (가로): 연산 강도 [FLOPs/byte]
    - 메모리에서 1 byte 읽을 때마다 수행하는 부동소수점 연산 수
    - 낮은 값: 데이터를 많이 읽고 연산은 적음 (memory-bound)
    - 높은 값: 데이터를 적게 읽고 연산이 많음 (compute-bound)

두 개의 한계선:

  1. 메모리 대역폭 선: slope = 메모리 대역폭 [GB/s]
     y = bandwidth × x  (FLOPs/s = bytes/s × FLOPs/byte)
     이 선 아래는 메모리에서 데이터를 못 받아서 연산 유닛이 놀고 있음

  2. 연산 한계 선: y = peak_GFLOP (수평선)
     연산 유닛의 이론적 최대치
     이 선 이상은 연산 유닛이 포화상태 — 더 빠를 수 없음

  루프라인 = 두 한계선 중 낮은 쪽
     min(bandwidth × AI, peak_GFLOP)

  내 코드의 점을 그렸을 때:
  ├── 메모리 대역폭 선 근처: memory-bound → 연산 강도 높이기
  ├── 연산 한계 선 근처:     compute-bound → 알고리즘/SIMD 최적화
  └── 두 선 훨씬 아래:       다른 병목 존재 (분기? 의존성 체인?)
```

---

## 🔬 내부 동작 원리

### 1. 연산 강도 (Arithmetic Intensity) 계산

```
연산 강도 = 총 FLOPs / 총 메모리 트래픽 (bytes)

예제 1: 벡터 합산 (SAXPY)
  코드: y[i] = a * x[i] + y[i]
  N개 요소, float(4B)

  FLOPs:
    곱셈 1번 + 덧셈 1번 = 2 FLOPs/요소
    총 FLOPs = 2N

  메모리 트래픽:
    x[i] 읽기:  N × 4 bytes
    y[i] 읽기:  N × 4 bytes
    y[i] 쓰기:  N × 4 bytes
    총 트래픽 = 12N bytes

  연산 강도 = 2N / 12N = 1/6 ≈ 0.17 FLOPs/byte
  → 매우 낮음! 메모리에서 1 byte 올 때마다 0.17 연산만 함
  → 전형적인 memory-bound 연산

예제 2: 행렬 곱 (Naive, N×N float)
  코드: C[i][j] += A[i][k] * B[k][j]  (삼중 루프)
  N=512, float(4B)

  FLOPs:
    각 (i,j) 쌍에 N번의 (곱셈+덧셈) = 2 FLOPs
    총 FLOPs = 2 × N³

  메모리 트래픽 (캐시 블로킹 없는 경우):
    A: N² × 4B (각 행을 N번 읽음, 하지만 L3에 남으면 한 번)
    B: N³ × 4B (열 접근 → 매번 미스)
    C: N² × 4B
    캐시 없다고 가정 시: 트래픽 ≈ N³ × 4B (B가 지배)

  단순 연산 강도 ≈ 2N³ / (4N³) = 0.5 FLOPs/byte
  → 여전히 낮음

  캐시 블로킹 적용 후:
    블록 B×B가 L1에 올라오면 해당 블록 재사용
    A 블록: B² × 4 bytes 읽어서 B² × 2B FLOPs 수행
    연산 강도 ≈ B × 0.5 FLOPs/byte (B=32이면 ≈ 16 FLOPs/byte)
    → compute-bound 영역으로 이동!

예제 3: 내적 (Dot Product, N float)
  FLOPs = 2N (곱+합)
  트래픽 = 2N × 4B = 8N bytes (x, y 각각 한 번씩)
  연산 강도 = 2N / 8N = 0.25 FLOPs/byte → memory-bound
```

### 2. 머신 파라미터 측정

```bash
# 방법 1: likwid-bench로 메모리 대역폭 측정
# apt install likwid

# Triad 벤치마크: a[i] = b[i] + scalar * c[i]
# 3개 스트림 읽기/쓰기 → 메모리 대역폭 측정의 표준
likwid-bench -t triad -w S0:1GB

# 출력 예시:
# MFlops/s:                 11234.56
# MBytes/s:                 44938.24  ← 메모리 대역폭 = 44.9 GB/s
# (triad: FLOPs = 2, bytes = 4×3 = 12 → 대역폭 = MFlops/s × 12/2 = MBytes/s)

# 방법 2: 직접 스트리밍 쓰기로 측정
likwid-bench -t copy -w S0:1GB
# copy: a[i] = b[i] (읽기+쓰기만, FLOPs 없음)
# MBytes/s = 메모리 대역폭 직접 측정

# 방법 3: Intel MLC (Memory Latency Checker)
# https://www.intel.com/content/www/us/en/download/736633/intel-memory-latency-checker.html
mlc --bandwidth_matrix
# 각 NUMA 노드 간 대역폭 행렬 출력
# 로컬: 45 GB/s, 원격: 25 GB/s (Ch6-02 NUMA 참조)

# 방법 4: 이론적 계산
# DDR4-3200 × 2채널: 3200 × 64bit × 2 / 8 = 51.2 GB/s
# DDR5-4800 × 2채널: 4800 × 64bit × 2 / 8 = 76.8 GB/s
# (실제는 이론의 80~90%)
```

### 3. 실제 연산 강도 측정 (`likwid-perfctr`)

```bash
# likwid-perfctr로 실제 FLOPs와 메모리 트래픽 측정
# 권한 필요: sudo 또는 kernel.perf_event_paranoid=0

# 사용 가능한 그룹 확인
likwid-perfctr -a | grep -i flop

# FLOPS_DP 그룹: 배정밀도 FLOPs
likwid-perfctr -C 0 -g FLOPS_DP ./matrix_mult

# 출력 예시:
# +-----------------------+------------+
# |         Metric        |   Core 0   |
# +-----------------------+------------+
# |  DP [MFLOP/s] STAT    |   89234.56 |  ← 실제 89.2 GFLOPs/s
# |  MFLOPS                |  89234.56  |
# +-----------------------+------------+

# 메모리 대역폭 측정
likwid-perfctr -C 0 -g MEM_DP ./matrix_mult

# 출력 예시:
# +----------------------------------+------------+
# |             Metric               |   Core 0   |
# +----------------------------------+------------+
# |  Memory bandwidth [MBytes/s]     |  12456.78  |  ← 12.5 GB/s 실제 사용
# |  Memory data volume [GBytes]     |     15.23  |
# +----------------------------------+------------+

# 두 측정값으로 연산 강도 계산:
# AI = 89234.56 MFLOPs/s ÷ 12456.78 MB/s
#    = 7.16 FLOPs/byte

# Intel Advisor를 사용하는 경우 (GUI):
# advisor --collect=roofline --project-dir=./advisor_results -- ./matrix_mult
# advisor --report=roofline --project-dir=./advisor_results
# → 자동으로 루프라인 그래프 생성 + 코드의 점 표시
```

### 4. 루프라인 그래프 직접 그리기

```python
# roofline.py — 루프라인 그래프 생성 (Python + matplotlib)
import numpy as np
import matplotlib.pyplot as plt

# 머신 파라미터 (예: Intel Core i7-10700, DDR4-3200 × 2ch)
MEM_BW   = 45.0    # GB/s (likwid-bench triad 측정값)
PEAK_DP  = 384.0   # GFLOPs/s (Core × 4.8GHz × 2 FMAs × 8 DP = AVX2 DP 피크)
PEAK_SP  = 768.0   # GFLOPs/s (단정밀도는 2배)
L3_BW    = 200.0   # GB/s (L3 내 접근, 메모리보다 빠름)

# X축: 연산 강도
AI = np.logspace(-2, 3, 1000)  # 0.01 ~ 1000 FLOPs/byte

# 루프라인 (DRAM 기준)
roofline_dram = np.minimum(MEM_BW * AI, PEAK_DP)

# 루프라인 (L3 기준)
roofline_l3 = np.minimum(L3_BW * AI, PEAK_DP)

fig, ax = plt.subplots(figsize=(10, 7))
ax.loglog(AI, roofline_dram, 'b-', linewidth=2, label=f'DRAM BW ({MEM_BW} GB/s)')
ax.loglog(AI, roofline_l3,  'g--', linewidth=2, label=f'L3 BW ({L3_BW} GB/s)')
ax.axhline(PEAK_DP, color='r', linewidth=2, label=f'Peak DP ({PEAK_DP} GFLOPs/s)')

# 내 코드의 측정 결과 점 찍기
workloads = [
    ("SAXPY (naive)",          0.17,  3.2,  'rs', 'red'),    # AI=0.17, 3.2 GFLOPs/s
    ("MatMul (naive)",         0.5,   18.0, 'b^', 'blue'),   # AI=0.5, 18 GFLOPs/s
    ("MatMul (블로킹)",         16.0,  280.0,'go', 'green'),  # AI=16, 280 GFLOPs/s
    ("FFT",                    1.2,   42.0, 'm*', 'purple'),  # AI=1.2, 42 GFLOPs/s
]

for name, ai, perf, marker, color in workloads:
    ax.plot(ai, perf, marker, markersize=10, color=color, label=name)
    ax.annotate(name, (ai, perf), textcoords="offset points",
                xytext=(10, 5), fontsize=9)

ax.set_xlabel('Arithmetic Intensity [FLOPs/byte]', fontsize=12)
ax.set_ylabel('Performance [GFLOPs/s]', fontsize=12)
ax.set_title('Roofline Model — Intel Core i7-10700', fontsize=14)
ax.legend(loc='lower right', fontsize=9)
ax.grid(True, which='both', alpha=0.3)
plt.tight_layout()
plt.savefig('roofline.png', dpi=150)
print("roofline.png 저장 완료")
```

---

## 💻 실전 실험

### 실험 1: SAXPY의 루프라인 위치 확인

```c
// saxpy.c — 전형적인 memory-bound 연산
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N (1 << 25)  // 32M float → 128MB

void saxpy(float* y, const float* x, float a, int n) {
    for (int i = 0; i < n; i++)
        y[i] = a * x[i] + y[i];
}

int main() {
    float* x = (float*)malloc(N * sizeof(float));
    float* y = (float*)malloc(N * sizeof(float));
    for (int i = 0; i < N; i++) { x[i] = 1.0f; y[i] = 2.0f; }

    struct timespec t0, t1;
    const int REPS = 10;

    // 워밍업
    saxpy(y, x, 3.14f, N);

    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int r = 0; r < REPS; r++)
        saxpy(y, x, 3.14f, N);
    clock_gettime(CLOCK_MONOTONIC, &t1);

    double ms = ((t1.tv_sec - t0.tv_sec) * 1e3
                + (t1.tv_nsec - t0.tv_nsec) / 1e6) / REPS;

    double bytes = (double)N * sizeof(float) * 3;   // x 읽기 + y 읽기 + y 쓰기
    double flops = (double)N * 2;                    // 곱 + 합
    double ai    = flops / bytes;                    // 연산 강도
    double gbps  = bytes / (ms * 1e-3) / 1e9;        // 실제 대역폭
    double gflops = flops / (ms * 1e-3) / 1e9;

    printf("시간:    %.2f ms\n", ms);
    printf("대역폭:  %.2f GB/s\n", gbps);
    printf("GFLOPs/s: %.2f\n", gflops);
    printf("연산 강도: %.3f FLOPs/byte\n", ai);

    // 출력 예시:
    // 시간:    7.23 ms
    // 대역폭:  44.38 GB/s  ← 머신 최대 45 GB/s에 98.6% 도달!
    // GFLOPs/s: 7.40
    // 연산 강도: 0.167 FLOPs/byte
    // → DRAM 대역폭 한계에 완전히 붙어있음 → 더 빠르게 하려면 AI를 올려야 함

    free(x); free(y);
    return 0;
}
```

```bash
gcc -O3 -march=native -o saxpy saxpy.c
./saxpy

# perf로 더 자세히 확인
perf stat -e cycles,instructions,\
uncore_imc/cas_count_read/,\
uncore_imc/cas_count_write/ \
./saxpy

# likwid-perfctr로 측정 (더 정확한 메모리 트래픽)
likwid-perfctr -C 0 -g MEM_SP ./saxpy
```

### 실험 2: 행렬 곱의 루프라인 위치 변화 (캐시 블로킹 전후)

```bash
# 행렬 곱 naive vs 블로킹 버전의 루프라인 위치 비교

# FLOPS_DP 그룹으로 FLOPs 측정
likwid-perfctr -C 0 -g FLOPS_DP ./matmul_naive
# 출력: DP [MFLOP/s] = 18,234 (18.2 GFLOPs/s)

likwid-perfctr -C 0 -g MEM_DP ./matmul_naive
# 출력: Memory bandwidth = 44,891 MB/s
# AI = 18,234 / 44,891 = 0.41 FLOPs/byte (DRAM 선 아래)

likwid-perfctr -C 0 -g FLOPS_DP ./matmul_blocked
# 출력: DP [MFLOP/s] = 284,567 (284.6 GFLOPs/s)

likwid-perfctr -C 0 -g MEM_DP ./matmul_blocked
# 출력: Memory bandwidth = 19,234 MB/s (데이터 재사용으로 크게 감소!)
# AI = 284,567 / 19,234 = 14.8 FLOPs/byte (DRAM 선 훨씬 위, Peak 근처)

# 결과 해석:
# Naive:   AI=0.41 → DRAM 대역폭 한계 선 아래 (memory-bound)
# Blocked: AI=14.8 → DRAM 선 훨씬 위, 연산 한계(compute-bound) 근처
# → 캐시 블로킹이 연산 강도를 36배 향상시킨 것
```

### 실험 3: 각종 연산의 이론적 AI 계산

```
자주 나오는 연산의 연산 강도 목록 (float, 캐시 미고려):

연산                     FLOPs   Bytes   AI        특성
─────────────────────────────────────────────────────────────────────
벡터 합산 (sum)           N       4N      0.25      memory-bound
벡터 덧셈 (a+b)           N       12N     0.08      매우 memory-bound
SAXPY (a*x+y)             2N      12N     0.17      memory-bound
내적 (dot product)        2N      8N      0.25      memory-bound
행렬-벡터 곱 (N×N)        2N²     4N²+4N  ~0.5      memory-bound
행렬 곱 (N×N, no cache)   2N³     4N³     0.5       memory-bound
행렬 곱 (N×N, L1 캐시)    2N³     4N²×3   N/3       N이 크면 compute-bound
Stencil 5-point (2D)     9N      4N×5    0.45      memory-bound
FFT (N log N)             5N lgN  8N      ~1.5      중간 영역
─────────────────────────────────────────────────────────────────────

AI 해석 기준 (DDR4-3200, Intel AVX2 기준):
  AI < 10 FLOPs/byte → DRAM 대역폭 한계 아래 → memory-bound
  AI ≈ 20 FLOPs/byte → 루프라인 모퉁이점 (ridge point) 근방
  AI > 50 FLOPs/byte → 연산 한계에 붙음 → compute-bound
```

---

## 📊 성능 비교

```
루프라인 모델 기반 최적화 효과 정리 (float, N=2048×2048):

────────────────────────────────────────────────────────────────────
구현                AI (FLOPs/B)  실제 성능    이론 한계     효율
────────────────────────────────────────────────────────────────────
행렬 곱 naive        0.41          18 GFLOPs   18.5 GFLOPs  97% ← DRAM 한계
행렬 곱 (32×32 블록) 8.2          212 GFLOPs   211 GFLOPs  100% ← DRAM 한계 돌파
행렬 곱 (64×64 블록) 14.8         284 GFLOPs   332 GFLOPs   86% ← 연산 한계 근처
행렬 곱 (OpenBLAS)   ~20          356 GFLOPs   384 GFLOPs   93% ← 연산 한계

SAXPY              0.17           7.4 GFLOPs   7.5 GFLOPs  99% ← DRAM 한계
SAXPY+L3 재사용     0.17          37 GFLOPs    34 GFLOPs  109% ← L3 대역폭
────────────────────────────────────────────────────────────────────

루프라인 한계 위반(>100%)의 의미:
  - 측정 오류 (캐시 효과를 트래픽에 반영 안 함)
  - 하드웨어 프리페치가 대역폭을 예상보다 높임
  - L3 BW를 DRAM BW로 혼동 (별도 측정 필요)

케이스 스터디 (Ch7-05)에서 이 수치들이 실제로 어떻게 달라지는지 추적함
```

---

## ⚖️ 트레이드오프

```
루프라인 모델의 한계와 주의사항:

모델이 잘 맞는 경우:
  ✅ 대용량 데이터 처리 (워킹셋 > L3)
  ✅ 수치 연산 (HPC, 행렬, 시뮬레이션)
  ✅ 스트리밍 패턴 (캐시 재사용 무시 가능)

모델이 맞지 않는 경우:
  ❌ 데이터가 L3에 완전히 들어오는 경우
     → L3 대역폭 기준 루프라인을 별도로 그려야 함
  ❌ 분기 집약적 코드
     → FLOPs 계산이 의미 없음, 분기 모델 필요
  ❌ 포인터 추적 (랜덤 접근)
     → 대역폭이 아닌 메모리 지연(latency)이 병목
     → DRAM 대역폭을 다 쓰지도 못함 (Ch2-03 포인터 추적 비용 참조)
  ❌ 멀티코어 경합
     → 단일 코어 루프라인이 멀티코어에 단순 확장 안 됨

정확한 트래픽 측정의 어려움:
  캐시 계층에 따라 실제 DRAM 트래픽이 이론과 다름
  Write-allocate 정책: 쓰기 전에 읽기 발생 (트래픽 증가)
  하드웨어 프리페치: 실제 필요보다 더 많이 읽을 수 있음

루프라인 vs 실제 성능 격차:
  루프라인은 "이상적 한계"
  실제로는 TLB 미스, 파이프라인 비효율, 메모리 컨트롤러 포화 등으로
  이론의 70~90%까지만 달성하는 경우가 많음

결론:
  루프라인 모델은 "방향 제시자"로 사용
  memory-bound vs compute-bound 판정과
  최적화 여지가 얼마나 남았는지 가이드
  정밀한 수치는 실제 측정(perf, likwid)으로 보완
```

---

## 📌 핵심 정리

```
루프라인 모델 핵심:

두 축:
  Y: 성능 [GFLOPs/s] — 높을수록 좋음
  X: 연산 강도 [FLOPs/byte] — 높을수록 compute-bound에 가까움

두 한계선:
  메모리 대역폭 선: y = bandwidth × AI (기울기 = 대역폭)
  연산 한계 선:     y = peak_GFLOP (수평선)

병목 판정:
  내 코드의 점이 메모리 선 근처 → memory-bound → 연산 강도 높이기
  내 코드의 점이 연산 선 근처  → compute-bound → SIMD/알고리즘 개선
  두 선 모두보다 한참 아래      → 다른 병목 (분기? 의존성 체인? TLB?)

연산 강도 높이는 방법:
  캐시 블로킹 (데이터 재사용 증가)
  루프 퓨전 (여러 패스를 한 번에)
  AoS→SoA + SIMD (메모리 대역폭 효율화, Ch5-04)

측정 도구:
  likwid-bench: 머신 대역폭 측정 (triad 표준)
  likwid-perfctr: 실제 FLOPs + 메모리 트래픽 측정
  Intel Advisor: 자동 루프라인 시각화

performance-testing-deep-dive 레포와의 연결:
  USE 방법론의 "Utilization"이 메모리 대역폭 활용률로 환원됨
  루프라인의 효율(%)이 곧 하드웨어 자원 활용도
```

---

## 🤔 생각해볼 문제

**Q1.** 어떤 코드의 연산 강도를 계산했더니 0.25 FLOPs/byte이고, 실제 성능이 3.2 GFLOPs/s였다. 머신의 DRAM 대역폭이 40 GB/s이고 연산 피크가 256 GFLOPs/s일 때, 이 코드의 루프라인 한계는 얼마이고, 현재 효율은 몇 %인가? 더 최적화하려면 무엇을 해야 하는가?

<details>
<summary>해설 보기</summary>

**루프라인 한계 계산:**

```
루프라인 = min(DRAM_BW × AI, Peak_GFLOP)
         = min(40 GB/s × 0.25 FLOPs/byte, 256 GFLOPs/s)
         = min(10 GFLOPs/s, 256 GFLOPs/s)
         = 10 GFLOPs/s
```

**현재 효율:**
```
효율 = 실제 성능 / 루프라인 한계
     = 3.2 GFLOPs/s / 10 GFLOPs/s
     = 32%
```

**해석:**
이 코드는 memory-bound (AI=0.25 < ridge point)이고, 이론적 한계의 32%밖에 달성하지 못하고 있습니다.

**최적화 방향:**

1. **효율 개선 (32% → 80~90%):** 메모리 접근 패턴이 비효율적인 것이 원인일 수 있습니다. 열 우선 순회를 행 우선으로 변경, SIMD로 한 번에 8개 요소 처리하는 등 대역폭을 더 효율적으로 사용합니다.

2. **연산 강도 증가 (0.25 → 2+ FLOPs/byte):** 데이터를 캐시에 더 오래 유지하고 재사용 횟수를 늘립니다. 루프 퓨전이나 캐시 블로킹으로 데이터를 읽어온 뒤 더 많은 연산을 수행합니다.

현실적으로 memory-bound 코드에서 대역폭 효율 80% 이상은 달성하기 어렵기 때문에, 연산 강도를 높여 ridge point를 넘기는 것이 더 큰 개선을 가져옵니다.

</details>

---

**Q2.** 스트리밍 워크로드(예: 대규모 로그 파싱, 영상 스트리밍 처리)의 연산 강도는 왜 대부분 낮은가? 이런 워크로드에서 루프라인 모델이 말하는 "최적화"가 현실에서 어떤 의미를 갖는가?

<details>
<summary>해설 보기</summary>

**스트리밍 워크로드의 낮은 연산 강도:**

로그 파싱이나 스트리밍 처리는 데이터를 한 번 읽고 간단한 처리(비교, 파싱, 출력)를 한 후 버립니다. 데이터 재사용이 없으므로 연산 강도가 구조적으로 낮습니다.

- 로그 파싱: 바이트 읽기 → 문자 비교 → 카운트. AI ≈ 0.05~0.1 FLOPs/byte
- 이미지 밝기 조정: 읽기 → 곱하기 → 쓰기. AI ≈ 0.08 FLOPs/byte

**루프라인 관점에서의 최적화 의미:**

이런 워크로드에서 루프라인 한계 = `DRAM_BW × AI`이므로, AI를 높이기가 구조적으로 어렵습니다. 따라서 "최적화"의 의미가 달라집니다:

1. **대역폭 효율을 높인다:** 현재 대역폭의 30%만 사용하고 있다면 prefetch, SIMD 로드(aligned), 비순차 I/O 제거로 대역폭 활용을 극대화합니다.

2. **연산 자체는 빠르게 (CPU 대기 최소화):** 메모리에서 오는 데이터를 연산 유닛이 즉각 처리할 수 있도록 파이프라인을 깨끗이 유지합니다.

3. **알고리즘 재설계:** 멀티패스를 싱글패스로 통합(루프 퓨전)하면 연산 강도가 약간 증가하고 총 메모리 트래픽도 줄어듭니다.

핵심은 memory-bound 스트리밍 워크로드의 최종 한계는 메모리 대역폭이며, CPU 성능보다 SSD I/O 대역폭이나 네트워크 대역폭이 병목인 경우가 많다는 것입니다.

</details>

---

**Q3.** 같은 행렬 곱 코드를 단일 코어와 8코어로 실행했다. 단일 코어에서 AI=14, 성능=280 GFLOPs/s이었다. 8코어에서는 AI가 유지된다고 가정할 때, 성능 기대치는 얼마인가? 실제로는 얼마가 나올 것으로 예상하며, 그 차이의 원인은 무엇인가?

<details>
<summary>해설 보기</summary>

**이론적 기대치:**
단일 코어 280 GFLOPs/s × 8코어 = 2,240 GFLOPs/s

**실제 예상치와 이유:**

단순 8배 확장은 달성하기 어렵습니다. 여러 제약이 있습니다:

1. **공유 메모리 대역폭 (Ch6-03 스케일링의 벽):**
   머신 총 메모리 대역폭은 고정 (예: 45 GB/s). 8코어가 동시에 메모리를 접근하면 총 대역폭을 나눠 씁니다. AI=14로 compute-bound에 가깝지만 메모리 트래픽이 조금이라도 있으면 경합 발생.

2. **L3 캐시 공유 경합:**
   행렬 블록이 L3 캐시를 공유합니다. 8코어가 경쟁하면 유효 L3 용량이 줄어들어 블록 크기 최적값이 달라집니다.

3. **연산 유닛 포화:**
   소켓당 연산 피크가 한계입니다. 코어 수와 클럭이 같으면 이론 피크도 8배이지만, Turbo Boost가 멀티코어 모드에서 클럭을 낮추면 실제 피크가 7~7.5배 수준입니다.

4. **NUMA 효과 (소켓 2개인 경우):**
   소켓 경계를 넘으면 메모리 지연 1.5~3배 증가 (Ch6-02).

**현실적 예상:** 약 5~6배 확장, 1,400~1,680 GFLOPs/s. 암달의 법칙 + 공유 자원 경합으로 선형 확장은 달성하기 어렵습니다 (Ch6-03).

</details>

---

<div align="center">

**[⬅️ 이전: 마이크로벤치마킹의 함정](./02-microbenchmarking-pitfalls.md)** | **[홈으로 🏠](../README.md)** | **[다음: 최적화 우선순위 ➡️](./04-optimization-priority.md)**

</div>
