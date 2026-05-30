# 명령어 수준 병렬성(ILP)의 한계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 의존성 사슬(dependency chain)이 OoO CPU의 병렬 실행을 막는 원리는 무엇인가?
- 누산기 분할(accumulator splitting)로 ILP를 높이는 방법은 구체적으로 어떻게 작동하는가?
- 단일 스레드 클럭이 2000년대 후반부터 정체된 근본 원인은 무엇인가?
- 왜 2010년대부터 코어 수를 늘리는 방향으로 방향이 전환됐는가?
- 암달의 법칙(Amdahl's Law)이 코어 수 증가에 제동을 걸 수 있는 이유는?
- ILP 벽을 만났을 때 다음 단계로 어떤 최적화 전략을 취해야 하는가?

---

## 🔍 왜 이 개념이 중요한가

### "OoO CPU가 있어도 빠르지 않은 코드 — 의존성 체인이 원인이다"

```
Ch1-04에서 배운 것:
  OoO CPU + 슈퍼스칼라 → IPC 4~6 가능
  레지스터 이름 변경 → WAR/WAW 제거
  RS + ROB → 독립 명령어를 동시에 실행

그런데 실제로 측정하면:
  많은 코드의 IPC가 여전히 1.0 이하

perf stat 출력:
  500,000,000   cycles
  400,000,000   instructions   # 0.80 insn per cycle

  "OoO가 있는데 왜 IPC가 1도 안 되나?"

  답: 의존성 체인 (Dependency Chain)
  
  // 예시:
  long x = 0;
  for (int i = 0; i < N; i++) {
      x = x * M + c;  // 매 반복이 이전 반복의 x에 의존
  }
  
  이 루프는 OoO로도 병렬화 불가
  곱셈 레이턴시 3 사이클 → IPC = 1/3 ≈ 0.33
  슈퍼스칼라 6-wide이지만 실제 처리량: 6분의 1만 사용

알아야 하는 이유:
  이 지점에 도달하면 단일 코어 최적화의 벽에 부딪힌 것
  다음 선택은: 알고리즘 변경, 누산기 분할, 또는 멀티코어(Ch6)
```

---

## 😱 잘못된 이해

### Before: "현대 CPU는 OoO가 있으니 의존성은 CPU가 알아서 처리한다"

```
잘못된 모델:
  "컴파일러 -O2로 최적화하면 OoO CPU가 알아서 병렬화한다"
  "코드 구조는 성능에 영향이 없다, CPU가 최적화한다"

반례:

  // 이 두 함수는 같은 결과를 냄
  // 버전 A: 단일 누산기
  long sum_a(long *a, int n) {
      long s = 0;
      for (int i = 0; i < n; i++) s += a[i];  // s에 의존성 체인
      return s;
  }

  // 버전 B: 4개 누산기
  long sum_b(long *a, int n) {
      long s0=0, s1=0, s2=0, s3=0;
      for (int i = 0; i < n-3; i += 4) {
          s0 += a[i]; s1 += a[i+1];   // s0, s1, s2, s3 서로 독립
          s2 += a[i+2]; s3 += a[i+3];
      }
      return s0+s1+s2+s3;
  }

  perf 측정 결과:
    sum_a: IPC ≈ 1.0, 시간 X
    sum_b: IPC ≈ 4.0, 시간 X/4

  OoO CPU는 "데이터 의존성이 없는 명령어만" 병렬화한다
  의존성 체인은 OoO CPU도 어쩔 수 없다
```

---

## ✨ 올바른 이해

### After: ILP는 의존성 그래프의 구조에 의해 결정된다

```
ILP(Instruction-Level Parallelism) 정의:
  동시에 실행 가능한 명령어의 수
  = 의존성 그래프에서 병렬 경로의 수

ILP를 결정하는 두 요소:

  1. 하드웨어 ILP 상한 (Structural ILP Limit):
     슈퍼스칼라 발행 폭: 현대 CPU 6-wide → 이론적 IPC 6
     실행 포트 수: ALU, FPU, Load, Store 등
     ROB 크기, RS 크기

  2. 소프트웨어 ILP (Algorithmic ILP Limit):
     코드의 의존성 그래프가 얼마나 병렬한가
     의존성 체인 길이 = 임계 경로(Critical Path) 길이

  IPC 상한 = min(하드웨어 상한, 소프트웨어 상한)

  소프트웨어 상한 = N개 명령어 / 임계 경로 길이

  예시:
    10개 명령어, 의존성 체인 길이 = 10 → IPC 상한 = 1.0
    10개 명령어, 의존성 체인 길이 = 2  → IPC 상한 = 5.0
    10개 명령어, 의존성 체인 없음      → IPC 상한 = 10 (하드웨어 상한에 의해 제한)

ILP의 한계(ILP Wall):
  1990~2000년대: 클럭 올리기 + OoO 발행 폭 늘리기로 성능 증가
  2000년대 중반: 의존성 체인이 발행 폭 확장의 이득을 상쇄
  발열 한계와 ILP Wall이 동시에 등장 → 단일 코어 성능 정체
  → 해결책: 코어 수를 늘리는 방향으로 전환 (Ch6으로 이어짐)
```

---

## 🔬 내부 동작 원리

### 1. 의존성 사슬과 임계 경로

```
의존성 사슬이란:
  각 명령어의 결과가 다음 명령어의 입력으로 쓰이는 체인
  → 순서대로만 실행 가능, OoO 병렬화 불가

임계 경로(Critical Path):
  의존성 그래프에서 가장 긴 경로
  임계 경로의 총 레이턴시 = 루프 반복당 최소 소요 사이클

예시: FMA(Fused Multiply-Add) 의존성 체인

  // dot product (단순 버전)
  float dot = 0.0f;
  for (int i = 0; i < N; i++) {
      dot += a[i] * b[i];  // dot에 의존성 체인
  }

  어셈블리 (SSE):
    movss  xmm0, [rsi + rax*4]   ; a[i] 로드
    mulss  xmm0, [rdx + rax*4]   ; a[i] * b[i]  (4 사이클 레이턴시)
    addss  xmm1, xmm0            ; dot += ...   (4 사이클 레이턴시)
    add    rax, 1
    cmp    rax, rcx
    jl     .loop

  임계 경로: addss의 의존성 체인
    반복 1의 addss 완료 (4 사이클)
    → 반복 2의 addss 시작
    → 반복 2의 addss 완료 (4 사이클)
    ...
  → 루프당 최소 4 사이클 (addss 레이턴시)
  → IPC ≈ 6 명령어 / 4 사이클 = 1.5

  하지만 실제 관측:
    mulss 레이턴시도 4 사이클
    mulss 결과가 addss 입력 → 직렬화
    → 루프당 최소 4~5 사이클
    → IPC ≈ 6 / 5 = 1.2

  이 코드에서 IPC를 4.0으로 올리려면?
  → 누산기 분할 (다음 섹션)

레이턴시 vs 처리량의 구분:
  레이턴시(Latency):   명령어 결과가 나올 때까지 걸리는 사이클
  처리량(Throughput): 매 사이클 몇 개 명령어를 시작할 수 있는가

  addss (FP 덧셈): 레이턴시 4 사이클, 처리량 0.5 사이클 (2/사이클)
  의미:
    의존성 없으면: 매 0.5 사이클마다 새 addss 시작 가능
    의존성 있으면: 4 사이클 기다린 후에야 다음 addss 시작
    → 의존성 체인이 처리량이 아닌 레이턴시로 속도 제한
```

### 2. 누산기 분할(Accumulator Splitting): 의존성 체인 끊기

```
핵심 아이디어:
  단일 누산기 → 여러 독립 누산기
  독립 누산기들은 서로 다른 실행 유닛에서 동시에 실행 가능

단계별 예시 (부동소수점 합산):

  // 버전 1: 단일 누산기 (IPC 낮음)
  float sum = 0.0f;
  for (int i = 0; i < N; i++) {
      sum += a[i];   // sum에 의존성 체인, 레이턴시 4 사이클
  }
  // 루프당 최소 4 사이클, IPC ≈ 1.0

  // 버전 2: 2-way 분할
  float s0 = 0.0f, s1 = 0.0f;
  for (int i = 0; i < N; i += 2) {
      s0 += a[i];    // s0 체인, 4 사이클 레이턴시
      s1 += a[i+1];  // s1 체인, 4 사이클 레이턴시 (s0과 독립!)
  }
  float sum = s0 + s1;
  // 루프당 최소 4 사이클 (두 체인이 겹쳐서 실행)
  // 하지만 2배 많은 덧셈을 같은 시간에 → IPC ≈ 2.0

  // 버전 4: 4-way 분할 (권장)
  float s0=0, s1=0, s2=0, s3=0;
  for (int i = 0; i < N; i += 4) {
      s0 += a[i];
      s1 += a[i+1];   // s0, s1, s2, s3 모두 독립
      s2 += a[i+2];
      s3 += a[i+3];
  }
  float sum = (s0+s1) + (s2+s3);
  // 루프당 최소 4 사이클 (4개 체인이 모두 동시 진행)
  // 4배 많은 덧셈을 같은 시간에 → IPC ≈ 4.0
  // 3GHz × 4.0 IPC ≈ 12,000,000,000 FLOPS = 12 GFLOPS

적절한 분할 수:
  목표: 분할 수 × 레이턴시 ≤ 처리량 분모의 역수
  addss: 레이턴시 4, 처리량 2/사이클
  최적 분할 수 = 레이턴시 × (사이클당 처리량) = 4 × 2 = 8
  실용적: 4~8개 분할이 대부분의 경우 최적

컴파일러의 자동 분할:
  GCC/Clang -O3 -ffast-math: 자동으로 누산기 분할 시도
  주의: -ffast-math는 FP 결합 법칙 적용 허용
        → 부동소수점 덧셈의 결합 법칙 적용 = 결과가 약간 달라질 수 있음
        → 보수적인 코드에서는 수동 분할 또는 명시적 SIMD 필요

어셈블리로 확인 (godbolt):
  누산기 분할 버전을 -O2로 컴파일하면
  4개의 독립적인 addss/vaddss 명령어가 연속으로 출현
  각각 다른 XMM/YMM 레지스터 사용 확인
```

### 3. ILP Wall: 단일 코어의 한계

```
1980~2005년: 클럭 + ILP로 성능 향상

  클럭 향상: 프로세스 노드 미세화 → 게이트 지연 감소 → 클럭 올리기
  ILP 향상: 파이프라인 깊게, 슈퍼스칼라 넓게, OoO 윈도우 크게

  Intel Pentium (1993): 0.06 GIPS, 66 MHz
  Intel Pentium III (2000): ~1 GIPS, 1 GHz
  Intel Pentium 4 (2003): ~3 GIPS, 3 GHz
  
  매 2년마다 성능 2배 (무어의 법칙 + 데나드 스케일링)

2005년 이후: 두 개의 벽에 동시에 부딪힘

  ① 열 설계 전력(TDP) 벽 (Power Wall):
     클럭을 올리면 전력은 f³으로 증가 (동적 전력 ∝ C×V²×f)
     4 GHz 이상에서 단위 면적당 열 방출이 원자로 수준
     → 클럭 주파수 정체 (~3~5 GHz에서 멈춤)

  ② ILP 벽 (ILP Wall):
     발행 폭을 4→6→8로 늘려도 의존성 체인이 병렬화를 막음
     대부분의 일반 코드: 진짜 의존성으로 인해 IPC ≈ 2~3
     발행 폭 8-wide로 IPC 8을 달성하기 위해선
     매 사이클 8개 독립 명령어가 있어야 함 → 일반 코드에는 없음
     → 더 넓은 슈퍼스칼라는 면적·전력 대비 수확 체감

  ③ 메모리 벽 (Memory Wall):
     CPU 클럭은 2배로 빨라지는데 DRAM 레이턴시는 개선 느림
     IPC가 높아도 메모리 대기로 파이프라인이 비워짐
     (Ch2에서 상세)

세 개의 벽이 동시에 등장 → 단일 코어 성능 정체

2005년 이후 대안: 코어 수 증가 (Many-Core):
  전력 예산 고정 시: 코어 1개를 고클럭으로 vs 코어 N개를 저클럭으로
  저클럭 N코어가 동일 전력에서 더 높은 총 처리량 달성 (전력 ∝ f³이므로)
  → 2005년 Intel Core Duo, 2006년 AMD Phenom 등장
  → 병렬 소프트웨어 요구 증가 (Ch6으로의 다리)

단일 스레드 성능 정체의 증거:
  SPEC CPU 2006 점수 (단일 스레드):
    2005년: ~100점
    2015년: ~400점 (10년에 4배 = 연 15% 성장)
  
  반면 1995~2005년:
    10년에 약 30배 성장 (연 40% 성장)
  
  → 단일 스레드 성능 향상 속도가 크게 둔화됨
```

### 4. 암달의 법칙(Amdahl's Law): 코어 증가의 한계

```
암달의 법칙 (1967):
  P = 병렬화 가능한 비율 (0~1)
  1-P = 순차 실행만 가능한 비율
  N = 코어 수

  최대 속도향상 = 1 / ((1-P) + P/N)
  N → ∞ 시: 최대 속도향상 = 1 / (1-P)

  예시:
    P = 0.95 (95% 병렬화 가능)
    N = 10 코어: 속도향상 = 1 / (0.05 + 0.10) = 6.7배
    N = 100 코어: 속도향상 = 1 / (0.05 + 0.01) = 16.7배
    N = ∞ 코어:  속도향상 = 1 / 0.05 = 20배 (상한!)

  순차 부분 5%가 전체 속도 향상의 절대 상한을 20배로 제한

  ┌────────────────────────────────────────────────────┐
  │  암달의 법칙 그래프                                 │
  │                                                    │
  │  속도향상                                           │
  │   ^                                                │
  │   │  P=0.99: ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ (상한 100배)  │
  │   │  P=0.95: ─ ─ ─ ─ ─ ─ ─ ─  (상한 20배)        │
  │   │  P=0.90: ─ ─ ─ ─ ─ ─  (상한 10배)            │
  │   │  P=0.75: ─ ─ ─ ─  (상한 4배)                 │
  │   │                                               │
  │   └───────────────────────────→ 코어 수            │
  └────────────────────────────────────────────────────┘

ILP 벽과의 연결:
  단일 코어 내에서도 암달의 법칙이 적용됨
  루프 내 의존성 체인 = "순차 부분"
  병렬화 가능한 독립 명령어 = "병렬 부분"
  
  // 예시: 10개 명령어, 4개가 의존성 체인
  의존성 체인(순차): 4개 → 1-P = 4/10 = 0.4
  독립 명령어(병렬): 6개 → P = 0.6
  최대 IPC 상한 = 1 / ((0.4/1) + (0.6/∞)) = 2.5
  → 슈퍼스칼라가 아무리 넓어도 IPC 2.5가 상한

구스타프슨의 반론 (Gustafson's Law, 1988):
  암달: 문제 크기 고정, 코어 수 늘리기
  구스타프슨: 코어 수 늘리면 더 큰 문제 해결 가능

  코어가 N배 늘면 같은 시간에 N배 큰 문제 가능
  큰 문제일수록 병렬 부분이 커지는 경향
  → 코어 수에 선형 비례하는 경우도 있음 (대규모 시뮬레이션, 렌더링 등)
  
  실제:
    "같은 소요 시간에 얼마나 많은 일을 하는가"가 목표라면
    코어 증가가 효과적
    "특정 작업을 얼마나 빨리 끝낼 수 있는가"가 목표라면
    암달의 법칙이 한계를 설정
```

### 5. ILP 한계를 넘는 전략들

```
전략 1: 알고리즘 변경 (가장 효과적)
  의존성 체인 자체를 알고리즘으로 끊기
  
  예: prefix sum (포함 스캔)
    // 순차 알고리즘 O(N): 의존성 체인 그대로
    for (int i = 1; i < n; i++) a[i] += a[i-1];
    
    // Kogge-Stone 병렬 알고리즘 O(N log N):
    for (int stride = 1; stride < n; stride *= 2)
        for (int i = stride; i < n; i++)
            a[i] += a[i - stride];  // 각 반복 내 명령어가 독립적으로 증가
    → 의존성 트리를 균형 잡아 임계 경로 = O(log N)으로 단축

전략 2: 누산기 분할 (섹션 2에서 상세)
  의존성 체인을 여러 독립 체인으로 분할
  마지막에 합산 (최적 분할 수: 레이턴시 × 처리량)

전략 3: SIMD 벡터화 (Ch5)
  단일 명령어로 여러 요소를 동시에 처리
  의존성 체인을 SIMD 너비만큼 묶어 단축
  
  // 스칼라: N번 반복, 각 반복이 이전에 의존
  // AVX2: 8개를 한 번에 처리 → 의존성 체인 길이 N/8

전략 4: 멀티코어 (Ch6)
  완전히 독립적인 작업을 별도 코어에서 병렬 실행
  단일 스레드 ILP 한계를 우회
  암달의 법칙: 직렬 부분을 최소화해야 효과적

전략 5: 비동기 / 메모리 레이턴시 은닉
  메모리 로드를 미리 발행 (prefetch)
  로드 완료를 기다리는 동안 다른 계산 수행
  소프트웨어 파이프라이닝(Software Pipelining)
```

---

## 💻 실전 실험

### 실험 1: 의존성 체인 vs 누산기 분할 IPC 비교

```c
// ilp_demo.c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N (1 << 26)  // 64M 요소

// 버전 1: 의존성 체인 (단일 누산기)
long sum_chain(long *a, long n) {
    long s = 0;
    for (long i = 0; i < n; i++) {
        s += a[i];   // 매 반복 s에 의존
    }
    return s;
}

// 버전 2: 누산기 분할 (4-way)
long sum_split4(long *a, long n) {
    long s0=0, s1=0, s2=0, s3=0;
    long i;
    for (i = 0; i < n - 3; i += 4) {
        s0 += a[i];
        s1 += a[i+1];
        s2 += a[i+2];
        s3 += a[i+3];
    }
    for (; i < n; i++) s0 += a[i];  // 나머지 처리
    return s0+s1+s2+s3;
}

// 버전 3: 누산기 분할 (8-way) — 더 많은 ILP 추출
long sum_split8(long *a, long n) {
    long s0=0, s1=0, s2=0, s3=0, s4=0, s5=0, s6=0, s7=0;
    long i;
    for (i = 0; i < n - 7; i += 8) {
        s0 += a[i];   s1 += a[i+1];
        s2 += a[i+2]; s3 += a[i+3];
        s4 += a[i+4]; s5 += a[i+5];
        s6 += a[i+6]; s7 += a[i+7];
    }
    for (; i < n; i++) s0 += a[i];
    return s0+s1+s2+s3+s4+s5+s6+s7;
}

int main(void) {
    long *a = (long*)malloc(N * sizeof(long));
    for (long i = 0; i < N; i++) a[i] = i & 0xFF;

    struct timespec t1, t2;

    // 워밍업
    volatile long dummy;
    dummy = sum_chain(a, N);
    dummy = sum_split4(a, N);
    dummy = sum_split8(a, N);

#define BENCH(fn, name) \
    clock_gettime(CLOCK_MONOTONIC, &t1); \
    dummy = fn(a, N); \
    clock_gettime(CLOCK_MONOTONIC, &t2); \
    printf("%-20s: %.1f ms  (결과: %ld)\n", name, \
           ((t2.tv_sec-t1.tv_sec)*1e3+(t2.tv_nsec-t1.tv_nsec)*1e-6), dummy);

    BENCH(sum_chain, "의존성 체인")
    BENCH(sum_split4, "4-way 분할")
    BENCH(sum_split8, "8-way 분할")

    free(a);
    return 0;
}
```

```bash
gcc -O2 -o ilp_demo ilp_demo.c
./ilp_demo

# 예상 출력 (L1/L2 캐시에 맞는 크기 기준):
# 의존성 체인:          100.0 ms
# 4-way 분할:            35.0 ms  (~2.9x)
# 8-way 분할:            28.0 ms  (~3.6x)

# IPC 측정 (각 버전을 별도 바이너리로):
perf stat -e cycles,instructions ./ilp_demo_chain
perf stat -e cycles,instructions ./ilp_demo_split4
# chain: IPC ≈ 1.0~1.5
# split4: IPC ≈ 3.0~4.0

# 어셈블리 확인 (godbolt 또는):
gcc -O2 -S -masm=intel ilp_demo.c -o ilp_demo.s
grep -A 30 "sum_chain:" ilp_demo.s
grep -A 40 "sum_split4:" ilp_demo.s
# split4: 4개의 add 명령어가 4개 다른 레지스터에 실행
```

### 실험 2: 의존성 체인 레이턴시와 처리량 측정

```c
// latency_vs_throughput.c: 레이턴시와 처리량의 차이를 직접 측정
#include <stdio.h>
#include <time.h>
#include <stdint.h>

#define REPS 1000000000LL

// 레이턴시 측정: 의존성 체인
// 각 곱셈이 이전 결과에 의존 → 곱셈 레이턴시 직렬
double measure_latency(void) {
    uint64_t x = 1;
    struct timespec t1, t2;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    for (long i = 0; i < REPS; i++) {
        x = x * 6364136223846793005ULL + 1;
    }
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ns = ((t2.tv_sec-t1.tv_sec)*1e9 + (t2.tv_nsec-t1.tv_nsec)) / (double)REPS;
    printf("(x=%lu 최적화 방지)\n", x);
    return ns;
}

// 처리량 측정: 독립 연산
// 4개 독립 누산기 → 곱셈 처리량(reciprocal throughput) 측정
double measure_throughput(void) {
    uint64_t a=1, b=2, c=3, d=4;
    const uint64_t M = 6364136223846793005ULL;
    struct timespec t1, t2;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    for (long i = 0; i < REPS/4; i++) {
        a = a * M + 1; b = b * M + 2;
        c = c * M + 3; d = d * M + 4;
    }
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ns_total = ((t2.tv_sec-t1.tv_sec)*1e9 + (t2.tv_nsec-t1.tv_nsec));
    double ns_per_mul = ns_total / (double)(REPS);  // REPS개 곱셈 총 시간
    printf("(a+b+c+d=%lu 최적화 방지)\n", a+b+c+d);
    return ns_per_mul;
}

int main(void) {
    printf("=== 64비트 곱셈 레이턴시 vs 처리량 ===\n\n");

    double lat_ns = measure_latency();
    printf("레이턴시:   %.2f ns/곱셈  ≈ %.1f 사이클 (@3GHz)\n\n",
           lat_ns, lat_ns * 3.0);

    double thr_ns = measure_throughput();
    printf("처리량:     %.2f ns/곱셈  ≈ %.1f 사이클 (@3GHz)\n\n",
           thr_ns, thr_ns * 3.0);

    printf("레이턴시/처리량 비 = %.1fx\n", lat_ns / thr_ns);
    printf("→ 이 비율만큼 누산기 분할로 성능 향상 가능\n");

    return 0;
}
```

```bash
gcc -O2 -o lat_thr latency_vs_throughput.c
./lat_thr

# 예상 출력:
# 레이턴시:   ~1.0 ns/곱셈  ≈ 3.0 사이클 (imulq 레이턴시 3 사이클)
# 처리량:     ~0.33 ns/곱셈 ≈ 1.0 사이클 (처리량 1/사이클)
# 비율: ~3x → 3-way 분할이 최적

perf stat -e cycles,instructions ./lat_thr
# 레이턴시 함수: IPC ≈ 0.33 (3 사이클마다 1 명령어)
# 처리량 함수: IPC ≈ 1.3 (4개를 3 사이클에)
```

### 실험 3: 단일 코어 클럭 정체 확인과 멀티코어 스케일링 측정

```c
// scaling_demo.c: 암달의 법칙 재현
// 직렬 부분과 병렬 부분을 명시적으로 분리
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <pthread.h>

#define N (1 << 28)    // 256M
#define MAX_THREADS 8

typedef struct {
    long *data;
    long start, end;
    long result;
} ThreadArg;

void *partial_sum(void *arg) {
    ThreadArg *a = (ThreadArg*)arg;
    long s = 0;
    // 병렬 부분: 각 스레드가 독립 구간 합산
    for (long i = a->start; i < a->end; i++) s += a->data[i];
    a->result = s;
    return NULL;
}

double time_parallel_sum(long *data, int nthreads) {
    pthread_t threads[MAX_THREADS];
    ThreadArg args[MAX_THREADS];
    long chunk = N / nthreads;

    struct timespec t1, t2;
    clock_gettime(CLOCK_MONOTONIC, &t1);

    // 병렬 합산
    for (int t = 0; t < nthreads; t++) {
        args[t] = (ThreadArg){data, t*chunk, (t+1)*chunk, 0};
        pthread_create(&threads[t], NULL, partial_sum, &args[t]);
    }

    // 직렬 부분: 결과 합산 (암달의 법칙의 "순차 부분")
    long total = 0;
    for (int t = 0; t < nthreads; t++) {
        pthread_join(threads[t], NULL);
        total += args[t].result;
    }

    clock_gettime(CLOCK_MONOTONIC, &t2);
    volatile long dummy = total;  // 최적화 방지
    (void)dummy;
    return ((t2.tv_sec-t1.tv_sec)*1e3 + (t2.tv_nsec-t1.tv_nsec)*1e-6);
}

int main(void) {
    long *data = (long*)malloc(N * sizeof(long));
    for (long i = 0; i < N; i++) data[i] = i & 0xFF;

    printf("스레드 수  |  시간(ms)  |  속도향상  |  암달 예측(P=0.98)\n");
    printf("─────────────────────────────────────────────────────────\n");

    double base_ms = time_parallel_sum(data, 1);
    printf("    1      |  %7.1f  |   1.00x    |   1.00x\n", base_ms);

    for (int t = 2; t <= MAX_THREADS; t *= 2) {
        double ms = time_parallel_sum(data, t);
        double speedup = base_ms / ms;
        // 암달 예측: P=0.98 (2% 직렬)
        double P = 0.98;
        double amdahl = 1.0 / ((1.0 - P) + P / t);
        printf("    %d      |  %7.1f  |   %.2fx    |   %.2fx\n",
               t, ms, speedup, amdahl);
    }

    free(data);
    return 0;
}
```

```bash
gcc -O2 -pthread -o scaling_demo scaling_demo.c
./scaling_demo

# 예상 출력:
# 스레드 수  |  시간(ms)  |  속도향상  |  암달 예측(P=0.98)
# ─────────────────────────────────────────────────────────
#     1      |   2000.0   |   1.00x    |   1.00x
#     2      |   1050.0   |   1.90x    |   1.96x
#     4      |    550.0   |   3.64x    |   3.77x
#     8      |    320.0   |   6.25x    |   6.89x
# → 실측 스케일링이 암달 예측에 근접
# → 8코어에서 이론 최대 8배 대비 6.25배 달성 (동기화 오버헤드, NUMA 등)

perf stat -e cycles,instructions ./scaling_demo
# 코어 수가 늘수록 총 instructions가 거의 같고 총 cycles가 줄어드는 것 확인
```

---

## 📊 성능 비교

```
ILP 한계의 실측 요약:

코드 패턴                    IPC      임계 경로     설명
──────────────────────────────────────────────────────────────────────
단일 누산기 (정수 덧셈)       1.0      N 사이클      덧셈 1 사이클 레이턴시
단일 누산기 (FP 덧셈)         0.25     4N 사이클     addss 4 사이클 레이턴시
4-way 분할 (FP 덧셈)          1.0      N 사이클      4개 체인이 겹침
8-way 분할 (FP 덧셈)          2.0      N/2 사이클    8개 체인 겹침
AVX2 수동 SIMD (8-wide FP)    4.0+     N/8 사이클    Ch5에서 상세
완전 독립 명령어 스트림        4~6      최소           발행 폭에 의해 제한

단일 코어 성능 역사 (SPEC CPU 점수 기준 추정):

  연도      단일코어 성능     클럭    주목할 점
  ──────────────────────────────────────────────────────────────────
  1993      1.0x             66 MHz   Pentium (기준)
  2000      30x              1 GHz    P6, 파이프라인 + OoO
  2003      80x              3 GHz    Pentium 4, 깊은 파이프라인
  2006      100x             2.4 GHz  Core 2 (클럭 낮지만 IPC 높음)
  2010      180x             3.3 GHz  Core i7, 단일코어 성능 둔화 시작
  2015      300x             4.0 GHz  Broadwell
  2020      450x             5.0 GHz  10년에 2.5배만 증가
  2024      600x             6.0 GHz  20년에 2배도 안 됨

암달의 법칙 스케일링 상한 (N 코어, P=병렬화 비율):

  P=0.50: 2코어→1.33x, 4코어→1.60x, 무한→2.0x  (매우 비효율적)
  P=0.90: 2코어→1.82x, 4코어→3.08x, 무한→10x
  P=0.95: 2코어→1.90x, 4코어→3.48x, 무한→20x
  P=0.99: 2코어→1.98x, 4코어→3.88x, 무한→100x
  → 직렬 부분 1%도 무한 코어의 상한을 100배로 제한
```

---

## ⚖️ 트레이드오프

```
단일 코어 ILP 최적화의 트레이드오프:

누산기 분할:
  ✅ 코드 변경 최소화, 컴파일러 힌트 없이도 효과
  ✅ 레이턴시 × 처리량만큼 성능 향상 가능 (최대 8x)
  ❌ 레지스터 사용량 증가 (레지스터 프레셔)
  ❌ 코드 가독성 저하 (반복 변수 여러 개)
  ❌ 분할 수가 너무 많으면 레지스터 스필(spill) 발생

SIMD (Ch5):
  ✅ 가장 강력한 단일 코어 ILP 향상 (8~16배)
  ✅ 누산기 분할 + 벡터화를 동시에 적용
  ❌ 코드 복잡도 급증 (인트린식 사용)
  ❌ 데이터 레이아웃 요구 (SoA 필요)
  ❌ 이식성 감소 (AVX2가 없는 CPU에서 못 씀)

멀티코어 (Ch6):
  ✅ 단일 코어 ILP 한계를 우회
  ✅ 완전 독립적인 작업에 선형 스케일링
  ❌ 동기화 오버헤드 (락, 채널, 배리어)
  ❌ 암달의 법칙: 직렬 부분이 상한 결정
  ❌ False Sharing (Ch3-02), NUMA 접근 비용 (Ch6-02)
  ❌ 소프트웨어 설계 복잡도 (레이스 컨디션, 데드락)

최적화 우선순위 (가장 효과적 순서):
  1. 알고리즘 복잡도 개선 (O(N²) → O(N log N))
  2. 메모리 접근 패턴 개선 (캐시 친화적, AoS→SoA)
  3. 의존성 체인 끊기 (누산기 분할, 알고리즘 재구성)
  4. SIMD 벡터화 (자동 또는 수동)
  5. 멀티코어 병렬화 (독립 작업 분리)
  6. 마이크로 최적화 (브랜치리스, 인라이닝)

언제 멀티코어로 가야 하는가:
  perf stat으로 측정했을 때:
    cache-misses 낮음 (데이터 잘 들어감)
    branch-misses 낮음 (예측 잘 됨)
    IPC가 발행 폭의 60~80% 수준
    의존성 체인이 IPC 상한 결정 (perf c2c, stall 분석)
  → 단일 코어에서 할 수 있는 것 다 했음 → 멀티코어로

Ch6 멀티코어와 NUMA로의 다리:
  단일 코어 ILP 한계에 부딪히면 두 가지 방향:
  ① 데이터 수준 병렬성: SIMD (Ch5)
  ② 작업 수준 병렬성: 멀티코어 (Ch6)
  
  Ch6에서 다루는 것:
    코어를 늘릴수록 성능이 선형으로 안 오르는 이유
    캐시 일관성 (MESI, Ch3)이 멀티코어 성능에 미치는 영향
    NUMA: 원격 메모리 접근 비용
    락 경합이 암달의 "직렬 부분"을 만드는 방식
```

---

## 📌 핵심 정리

```
ILP의 한계 핵심:

의존성 체인:
  진짜 RAW 의존성은 OoO도 병렬화 불가
  임계 경로 길이 = 루프당 최소 사이클
  IPC 상한 = 총 명령어 수 / 임계 경로 사이클

누산기 분할:
  단일 누산기 → N개 독립 누산기
  최적 분할 수 = 레이턴시(사이클) × 처리량(명령어/사이클)
  FP 덧셈: 레이턴시 4, 처리량 2 → 최적 분할 수 8
  컴파일러: -ffast-math로 자동 분할 (FP 결합 법칙 허용)

단일 코어 성능 정체의 원인:
  전력 벽: 클럭 주파수 ~3~5 GHz에서 정체
  ILP 벽: 일반 코드의 의존성이 발행 폭 확장 이득 상쇄
  메모리 벽: DRAM 레이턴시 개선이 클럭 속도를 못 따라감
  세 벽이 동시에 등장 → 2005년 이후 단일 코어 정체

코어 증가로의 전환:
  동일 전력 예산에서 저클럭 N코어 > 고클럭 1코어
  병렬 소프트웨어 요구 증가

암달의 법칙:
  직렬 부분 비율이 최대 속도 향상의 절대 상한 결정
  직렬 부분 5% → 최대 20배 상한 (코어가 아무리 많아도)
  락, 직렬 초기화, 순차 의존성이 "직렬 부분"을 만듦

이 챕터의 결론:
  단일 코어: 파이프라인 → 해저드 → OoO → 분기 예측 → ILP 한계
  모든 최적화를 해도 의존성 체인이 벽
  → 다음 장벽을 넘으려면 메모리 계층 (Ch2) 또는
    멀티코어 병렬성 (Ch6)으로 이동
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 루프의 임계 경로 길이를 계산하고, 현대 x86-64 CPU(Intel Skylake 기준)에서 이론적 IPC 상한을 구하라. 명령어 레이턴시: `imulq` 3 사이클, `addq` 1 사이클, 루프 분기 1 사이클.

```c
// 루프 내 명령어:
// imulq (의존성 체인에 포함)
// addq  (imulq 결과에 의존)
// 로드 (독립)
// 루프 카운터 증가 (독립)
// 분기 (루프 카운터에 의존)

long compute(long *a, int n) {
    long x = 1;
    for (int i = 0; i < n; i++) {
        x = x * 3 + a[i];
    }
    return x;
}
```

<details>
<summary>해설 보기</summary>

**어셈블리 분석** (`-O2` 기준):
```asm
; 루프 내 (대략):
imulq  rax, rax, 3        ; x = x * 3       (3 사이클 레이턴시, x에 의존)
addq   rax, [rdi+rcx*8]   ; x += a[i]       (1 사이클 레이턴시, imulq에 의존)
inc    rcx                 ; i++             (1 사이클, 독립)
cmp    rcx, rsi            ; i < n           (1 사이클, inc에 의존)
jl     .loop               ; 분기            (1 사이클, cmp에 의존)
```

**의존성 그래프**:
```
의존성 체인 (임계 경로):
  imulq (3cy) → addq (1cy) → 다음 반복 imulq (3cy) → ...

독립 명령어:
  로드 (a[i]): imulq와 독립, addq 전에 완료되면 됨
  inc, cmp, jl: 별도 의존성 체인 (루프 제어)
```

**임계 경로 계산**:
- 임계 경로: `imulq → addq` 체인이 반복
- 루프당 임계 경로 = 3(imulq) + 1(addq) = 4 사이클
- (로드는 imulq와 겹쳐서 실행되므로 임계 경로에 포함 안 됨)

**루프당 명령어 수** (대략):
- imulq 1 + addq 1 + 로드 1 + inc 1 + cmp 1 + jl 1 = 약 6개 명령어

**이론적 IPC 상한**:
- IPC 상한 = 명령어 수 / 임계 경로 = 6 / 4 = **1.5**

**측정 예상**:
```bash
perf stat -e cycles,instructions ./compute
# instructions / cycles ≈ 1.3~1.5 (메모리 접근과 루프 오버헤드 포함 시 약간 낮아질 수 있음)
```

**개선 방법**: x에 대한 의존성 체인을 끊을 수 없음 (`x = x * 3 + a[i]`는 진짜 RAW 의존성). 알고리즘 변경 없이는 이 한계를 넘기 어렵다. Horner 방법 등 수치 알고리즘 재구성이 필요한 경우.

</details>

---

**Q2.** 프로파일러로 어떤 함수가 전체 실행 시간의 80%를 차지하며 그 안에서 `branch-misses` 0%, `cache-misses` 1%, `IPC = 0.4`임을 확인했다. 이 숫자들로 무엇을 진단할 수 있고, 가장 가능성 높은 원인과 해결책은 무엇인가?

<details>
<summary>해설 보기</summary>

**진단**:

- `branch-misses 0%`: 분기 예측은 완벽 → 분기는 병목 아님
- `cache-misses 1%`: 메모리 접근도 양호 → 메모리 벽 아님
- `IPC = 0.4`: 슈퍼스칼라 6-wide에서 0.4는 매우 낮음

**가능성 높은 원인: 긴 레이턴시 의존성 체인**

IPC 0.4의 의미:
- 6-wide 슈퍼스칼라의 7% 활용 (0.4/6 × 100)
- 매 2.5 사이클에 명령어 1개 → 레이턴시가 2.5 사이클인 연산의 의존성 체인
- 부동소수점 연산 (레이턴시 4~5 사이클)이나 곱셈 체인이 의심됨

**확인 방법**:
```bash
# stalled-cycles-backend가 높으면 실행 유닛 대기 (의존성 체인 확인)
perf stat -e stalled-cycles-frontend,stalled-cycles-backend ./program

# 어셈블리 검사 (godbolt 또는)
gcc -O2 -S -masm=intel program.c | grep -E "mul|add|imul|vmulps|vaddps"
```

**해결책 우선순위**:
1. **어셈블리 확인**: FP 곱셈/덧셈 체인을 찾아 의존성 구조 파악
2. **누산기 분할**: 의존성 체인을 4~8개로 분할
3. **컴파일러 힌트**: `-ffast-math` 활성화 (FP 결합 법칙 허용)
4. **SIMD**: 8개 요소를 한 번에 처리 (Ch5)
5. **알고리즘 재설계**: 병렬화 가능한 형태로 변환

목표 IPC: 누산기 분할 후 IPC 1.5~2.5, SIMD 적용 시 IPC 4+ 달성 가능.

</details>

---

**Q3.** 다음 두 주장 중 어느 것이 더 정확한가? 암달의 법칙을 사용해 수치로 근거를 제시하라.

- **주장 A**: "코어를 100개로 늘리면 병렬화 가능한 부분이 99%인 코드는 100배 빨라진다"
- **주장 B**: "코어를 100개로 늘려도 직렬 부분이 1%만 있으면 최대 50배밖에 빨라지지 않는다"

<details>
<summary>해설 보기</summary>

**주장 B가 더 정확합니다.**

**암달의 법칙 계산**:

P = 0.99 (병렬화 가능 99%), N = 100 코어:
```
속도향상 = 1 / ((1 - P) + P/N)
         = 1 / (0.01 + 0.99/100)
         = 1 / (0.01 + 0.0099)
         = 1 / 0.0199
         ≈ 50.3배
```

**주장 A의 오류**:
100배 속도향상을 얻으려면:
```
100 = 1 / ((1-P) + P/100)
(1-P) + P/100 = 0.01
(1-P) + (1-(1-P))/100 = 0.01
100(1-P) + 1 - (1-P) = 1
99(1-P) = 0
1-P = 0
P = 1.0 (100% 병렬화, 직렬 부분 0%)
```
→ 직렬 부분이 단 1%라도 있으면 100배는 절대 불가능

**실무 시사점**:
- N=100 코어에서 50배 향상 = 효율 50% (100코어의 절반만 활용됨)
- 직렬 부분을 줄이는 것이 코어 수를 늘리는 것보다 중요
- 락 경합, 순차 초기화, I/O 대기가 "직렬 부분"에 해당

**무한 코어의 상한**:
- P=0.99, N→∞: 최대 속도향상 = 1/(1-0.99) = 100배
- 100코어로는 이 상한의 50%밖에 도달 못 함
- 1000코어로는 91% (91배): 수익 체감이 급격히 나타남

이것이 Ch6에서 코어 수를 늘리는 것 외에 **직렬 병목을 줄이고** 캐시 일관성 오버헤드를 최소화하는 것이 왜 중요한지의 기초입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 분기 예측](./05-branch-prediction.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 메모리 계층 ➡️](../memory-hierarchy-cache/01-memory-hierarchy-latency.md)**

</div>
