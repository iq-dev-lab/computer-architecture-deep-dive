# CPU 파이프라인 — Fetch/Decode/Execute/Memory/Writeback

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CPU 파이프라인의 5단계는 무엇이며 각 단계는 무엇을 하는가?
- IPC(Instructions Per Cycle)가 1 미만이면 무엇이 잘못된 것인가?
- 파이프라인이 없는 CPU 대비 처리량이 몇 배 향상되는가?
- 파이프라인 깊이(단계 수)를 늘리면 왜 항상 빨라지지 않는가?
- `perf stat`의 IPC 수치는 무엇을 의미하는가?
- 현대 CPU의 파이프라인은 5단계보다 훨씬 깊은데, 그 이유는 무엇인가?

---

## 🔍 왜 이 개념이 중요한가

### "내 코드의 IPC가 1.0이면 성능을 2배 더 낼 수 있다는 뜻"

```
perf stat 출력을 처음 보는 순간:
  1,234,567,890   instructions   #    1.23  insn per cycle

"1.23 IPC가 뭐지? 좋은 건가 나쁜 건가?"

이것을 이해하지 못하면:
  IPC 1.0 → 파이프라인이 절반쯤 비어 있다는 신호
  IPC 3.0 → 파이프라인이 잘 채워지고 있다는 신호
  IPC 0.5 → 메모리 레이턴시나 분기 미스로 파이프라인이
             자주 비워지고 있다는 강한 신호

파이프라인을 이해하면:
  "왜 IPC가 낮은가"를 체계적으로 추적 가능
    → 메모리 스톨? (Ch2에서 상세)
    → 분기 미스? (Ch1-05에서 상세)
    → 데이터 의존성? (Ch1-03에서 상세)

핵심 흐름:
  파이프라인 이해
    └── 해저드 이해 (Ch1-03)
          └── OoO 실행 이해 (Ch1-04)
                └── 분기 예측 이해 (Ch1-05)
                      └── ILP 한계 이해 (Ch1-06)
```

---

## 😱 잘못된 이해

### Before: "CPU는 명령어를 하나씩 순서대로 처리한다"

```
잘못된 모델:
  ┌─────────────────────────────────────────────────┐
  │  사이클 1: MOV eax, [rbp-4] 전부 처리 후 완료    │
  │  사이클 2: ADD eax, ecx    전부 처리 후 완료    │
  │  사이클 3: MOV [rbp-8], eax 전부 처리 후 완료  │
  └─────────────────────────────────────────────────┘
  → 1 사이클 = 1 명령어 완료
  → IPC = 1이 최대

현실:
  "명령어를 처리한다"는 것 자체가
  Fetch / Decode / Execute / Memory / Writeback
  5개의 독립적인 하위 단계로 구성됨
  
  각 단계는 서로 다른 하드웨어 유닛이 처리
  → 동시에 5개 명령어의 서로 다른 단계를 처리 가능
  → 이론적 IPC = 단계 수
  → 현대 슈퍼스칼라 CPU는 IPC 4~6 달성
```

---

## ✨ 올바른 이해

### After: 파이프라인 = 명령어 처리를 단계로 쪼개 동시에 여러 명령 처리

```
파이프라인 = 공장 조립 라인의 CPU 버전

조립 라인 비유:
  자동차 공장에 4개 스테이션이 있다고 가정
  차1: [스테이션1] → [스테이션2] → [스테이션3] → [스테이션4] → 완성
  차2:              [스테이션1] → [스테이션2] → [스테이션3] → [스테이션4]
  차3:                           [스테이션1] → [스테이션2] → [스테이션3]
  
  → 4개 차가 동시에 서로 다른 스테이션에서 처리됨
  → 생산량 = 4배 향상 (각 차 완성 시간은 같지만)

CPU 파이프라인:
  명령1: [Fetch] → [Decode] → [Execute] → [Memory] → [Writeback]
  명령2:          [Fetch]  → [Decode]  → [Execute] → [Memory]
  명령3:                    [Fetch]   → [Decode]  → [Execute]
  명령4:                               [Fetch]   → [Decode]
  명령5:                                          [Fetch]
  
  → 5사이클 후: 매 사이클마다 1명령어 완료 (IPC = 1)
  → vs 비파이프라인: 5사이클마다 1명령어 완료 (IPC = 0.2)
```

---

## 🔬 내부 동작 원리

### 1. 5단계 파이프라인 각 단계 상세

```
┌────────────────────────────────────────────────────────────────┐
│  STAGE 1: Fetch (IF — Instruction Fetch)                        │
│                                                                  │
│  rip(PC)가 가리키는 주소에서 명령어 바이트를 L1 명령어 캐시에서 읽음 │
│  64비트 주소의 메모리에서 16~32 바이트 청크를 한 번에 인출          │
│  다음 PC 계산 (기본: PC + 명령어 길이, 분기 예측기가 수정 가능)     │
│                                                                  │
│  병목 지점: L1i 캐시 미스 → 수십 사이클 대기 (코드 지역성 중요)    │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  STAGE 2: Decode (ID — Instruction Decode)                      │
│                                                                  │
│  바이트 열 → 명령어 종류, 오퍼랜드(소스/목적), 제어 신호 해석       │
│  x86-64: 가변 길이 명령어(1~15 바이트) → 복잡한 디코더 필요        │
│  현대 CPU: µop(마이크로 오퍼레이션)으로 분해                        │
│    LOCK CMPXCHG → 수십 개의 µop으로 분해될 수 있음                │
│  레지스터 이름 변경(Renaming)도 이 단계에서 수행 (Ch1-04)           │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  STAGE 3: Execute (EX — Execute)                                │
│                                                                  │
│  ALU(정수 연산), FPU(부동소수점), AGU(주소 계산) 등에서 실행        │
│  ADD, SUB, AND, OR, XOR, shift 등: 1 사이클                      │
│  MUL(정수): 3~4 사이클 레이턴시                                    │
│  DIV(나눗셈): 20~90 사이클 레이턴시! (피해야 할 연산)              │
│  현대 CPU: 여러 실행 유닛 병렬 존재 → 슈퍼스칼라 (Ch1-04)          │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  STAGE 4: Memory (MEM — Memory Access)                          │
│                                                                  │
│  로드(LDR) 명령어: 계산된 주소에서 L1 데이터 캐시 접근             │
│  스토어(STR) 명령어: 스토어 버퍼에 데이터 기록                     │
│  L1d 캐시 히트: ~4 사이클                                          │
│  L1d 캐시 미스: L2→L3→DRAM 폴백 (12~200 사이클, Ch2에서 상세)     │
│  계산 명령어(ADD 등): 이 단계 스킵                                  │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  STAGE 5: Writeback (WB — Write Back)                           │
│                                                                  │
│  실행 결과를 아키텍처 레지스터 파일(rax, rbx, ...)에 기록          │
│  OoO CPU: ROB(Reorder Buffer)에서 순서대로 커밋 (Ch1-04 참조)     │
│  예외/인터럽트 처리도 이 단계에서 감지 후 파이프라인 플러시         │
└────────────────────────────────────────────────────────────────┘
```

### 2. 사이클 단위 타이밍 다이어그램

```
비파이프라인 CPU (5-사이클 명령어 3개 처리):
사이클:  1    2    3    4    5    6    7    8    9   10   11   12   13   14   15
명령1:  [IF] [ID] [EX] [MEM][WB]
명령2:                          [IF] [ID] [EX] [MEM][WB]
명령3:                                               [IF] [ID] [EX] [MEM][WB]
→ 3개 완료에 15 사이클 필요, IPC = 3/15 = 0.2

5단계 파이프라인 CPU (같은 3개 명령어):
사이클:  1    2    3    4    5    6    7
명령1:  [IF] [ID] [EX] [MEM][WB]
명령2:       [IF] [ID] [EX] [MEM][WB]
명령3:            [IF] [ID] [EX] [MEM][WB]
→ 3개 완료에 7 사이클 필요, IPC = 3/7 ≈ 0.43
→ 파이프라인 채워지면: 매 사이클 1개 완료, IPC = 1

파이프라인 완전히 채워진 상태 (10개 명령어):
사이클:  1    2    3    4    5    6    7    8    9   10   11   12   13   14
명령1:  [IF] [ID] [EX] [MEM][WB]
명령2:       [IF] [ID] [EX] [MEM][WB]
명령3:            [IF] [ID] [EX] [MEM][WB]
명령4:                 [IF] [ID] [EX] [MEM][WB]
명령5:                      [IF] [ID] [EX] [MEM][WB]
...
→ 사이클 5부터: 매 사이클 1개 완료
→ IPC = 1 (파이프라인 해저드 없을 때 상한)
```

### 3. 파이프라인이 IPC를 높이는 원리

```
클럭 주파수(GHz)와 IPC의 관계:
  CPU 성능 = 클럭 주파수 × IPC × 코어 수
  (단일 스레드 성능 기준: 클럭 × IPC)

예: 3 GHz CPU, IPC = 1.0
  초당 명령어 수 = 3 × 10^9 × 1.0 = 30억 명령어/초

예: 3 GHz CPU, IPC = 3.0 (슈퍼스칼라, 잘 채워진 파이프라인)
  초당 명령어 수 = 3 × 10^9 × 3.0 = 90억 명령어/초

파이프라인이 IPC ≥ 1을 가능하게 하는 이유:
  각 단계는 독립적인 하드웨어를 사용
    Fetch Unit ← → Decode Unit ← → ALU ← → L1 캐시 ← → 레지스터 파일
  동시에 서로 다른 명령어가 각 유닛을 점유 가능
  → 1 사이클에 1 명령어 완료 (IPC = 1)

현대 슈퍼스칼라(Out-of-Order) CPU가 IPC > 1인 이유:
  각 파이프라인 단계에 복수의 유닛
    예: ALU 4개, 로드/스토어 유닛 2개
  1 사이클에 최대 4~6개 명령어 동시 실행 (Wide Execution)
  → 이론적 IPC = 4~6 (실제: 2~5 수준)
  → Ch1-04에서 ROB, Reservation Station과 함께 상세 설명
```

### 4. 파이프라인 깊이 트레이드오프

```
파이프라인 깊이 = 단계 수:
  얕은 파이프라인 (5단계, 예: MIPS 고전 설계):
    ✅ 각 단계 작업 많음 → 단계당 논리 복잡
    ✅ 분기 미스 비용 낮음 (5 사이클 낭비)
    ❌ 단계당 시간이 길어 클럭 주파수 한계
    ❌ 각 단계가 가장 느린 서브연산의 속도로 제한됨

  깊은 파이프라인 (20~31단계, 예: Intel NetBurst/Pentium 4):
    ✅ 각 단계가 매우 단순 → 단계당 시간 짧음 → 높은 클럭 가능
    ✅ Pentium 4: 3.8 GHz 달성
    ❌ 분기 미스 비용 폭증 (20 사이클 낭비 → 성능 급락)
    ❌ 실제 IPC가 떨어져 클럭 이점 상쇄
    ❌ 열 설계 전력(TDP) 폭증 → Pentium 4의 실패 이유

  중간 깊이 (14~19단계, 예: Intel Sandy Bridge 이후):
    ✅ 클럭과 IPC의 균형점
    ✅ 분기 미스 비용 ~15 사이클로 관리 가능
    현대 Intel: 14~19단계, AMD Zen: 19~24단계

파이프라인 깊이 추정:
  분기 미스 패널티(사이클) ≈ 파이프라인 단계 수
  perf stat의 branch-misses + 사이클 손실로 역산 가능
```

### 5. x86-64 파이프라인 현실: µop과 프론트엔드/백엔드

```
현대 Intel/AMD CPU의 실제 구조:
  클래식 5단계를 추상화로만 이해하고 실제는 더 복잡

  ┌──────────────────────────────────┐
  │  프론트엔드 (Frontend)             │
  │    Fetch (L1i 캐시 접근)           │
  │    Predecode (명령어 길이 파악)     │
  │    Decode (x86→µop 변환, 1~4 µop)│
  │    µop Cache (Decoded ICache)     │← 자주 쓰는 µop 시퀀스 캐시
  │    Branch Prediction              │
  └──────────────────┬───────────────┘
                     ↓ µop 스트림
  ┌──────────────────────────────────┐
  │  백엔드 (Backend, OoO 엔진)        │
  │    Rename / Allocate             │← RAT(Register Alias Table)
  │    Reservation Station (RS)      │← 실행 준비된 µop 대기
  │    Execution Units (ALU×4, etc.) │← 실제 연산
  │    Reorder Buffer (ROB)          │← 순차 커밋 보장
  │    Load / Store Buffer           │
  └──────────────────────────────────┘

핵심 용어:
  µop(마이크로 오퍼레이션): x86 명령어를 내부 RISC 유사 연산으로 분해
    PUSH rax → sub rsp, 8 + mov [rsp], rax (2 µop)
    단순 ADD → 1 µop
    복잡한 LOCK CMPXCHG → 수십 µop
  
  µop Cache: 이미 디코딩된 µop 시퀀스를 캐시 → Fetch→Decode 스킵
    L1i 캐시 히트보다도 빠름 (디코드 오버헤드 없음)
    루프 내 자주 실행되는 코드에서 큰 이득
```

---

## 💻 실전 실험

### 실험 1: perf stat으로 IPC 측정

```bash
# 배열 합산 벤치마크
cat > pipeline_bench.c << 'EOF'
#include <stdlib.h>
#include <stdio.h>

#define N (1 << 24)   // 16M 요소

int main(void) {
    int *arr = malloc(N * sizeof(int));
    for (int i = 0; i < N; i++) arr[i] = i & 0xFF;

    long sum = 0;
    // 단순 순차 합산 — 메모리 접근 패턴 좋음
    for (int i = 0; i < N; i++)
        sum += arr[i];

    printf("%ld\n", sum);  // 최적화 제거 방지
    free(arr);
    return 0;
}
EOF

gcc -O2 -o pipeline_bench pipeline_bench.c

# IPC 측정
perf stat -e cycles,instructions,cache-misses,branch-misses \
    ./pipeline_bench

# 예상 출력 (대략):
#    2,100,000,000   cycles
#    6,300,000,000   instructions   #  3.00  insn per cycle
#           12,000   cache-misses
#            5,000   branch-misses
#
# IPC 3.0 → 슈퍼스칼라가 잘 작동 중
# cache-misses 적음 → L1/L2 캐시에 잘 들어맞는 데이터

# 비교: 최적화 없음
gcc -O0 -o pipeline_bench_O0 pipeline_bench.c
perf stat -e cycles,instructions ./pipeline_bench_O0

# 예상: IPC ~1.2 (-O0는 불필요한 스택 접근으로 파이프라인 스톨)
```

### 실험 2: 파이프라인 단계를 직접 '느끼는' 지연 체인 실험

```c
// latency_chain.c: 의존성 체인으로 파이프라인 레이턴시 측정
// 각 명령어가 이전 결과에 의존 → 파이프라인이 직렬화됨
#include <stdio.h>
#include <time.h>
#include <stdint.h>

#define REPS 1000000000LL

int main(void) {
    struct timespec start, end;
    uint64_t x = 1;

    clock_gettime(CLOCK_MONOTONIC, &start);

    // 의존성 체인: 각 반복이 이전 반복의 결과에 의존
    // → 파이프라인이 앞 명령 완료를 기다려야 함
    for (long i = 0; i < REPS; i++) {
        x = x * 6364136223846793005ULL + 1;  // LCG (곱셈 3사이클 레이턴시)
    }

    clock_gettime(CLOCK_MONOTONIC, &end);

    double ns = (end.tv_sec - start.tv_sec) * 1e9
               + (end.tv_nsec - start.tv_nsec);
    printf("x=%lu\n", x);
    printf("평균 %.2f ns/iter (곱셈 레이턴시 ≈ %.1f 사이클 @ 3GHz)\n",
           ns / REPS, ns / REPS * 3.0);
    return 0;
}
```

```bash
gcc -O2 -o latency_chain latency_chain.c
./latency_chain

# 예상: ~1 ns/iter ≈ 3 사이클
# → 64비트 곱셈의 레이턴시가 3 사이클임을 확인
# → IPC는 높지 않음: 의존성 체인이 병렬 실행을 막음
# (이것이 Ch1-06에서 다루는 의존성 체인 = ILP 병목)

perf stat -e cycles,instructions ./latency_chain
# instructions / cycles ≈ 1/3 = 0.33 IPC
# → 매 3 사이클마다 1 명령어만 완료됨 (나머지는 대기)
```

### 실험 3: 독립 명령어 vs 의존성 체인 IPC 비교

```c
// independent.c: 독립적인 명령어들 → 파이프라인 완전 활용
#include <stdio.h>
#include <time.h>
#include <stdint.h>

#define REPS 1000000000LL

int main(void) {
    struct timespec start, end;
    // 서로 독립된 4개 누산기 사용
    uint64_t a = 1, b = 2, c = 3, d = 4;

    clock_gettime(CLOCK_MONOTONIC, &start);

    for (long i = 0; i < REPS; i++) {
        // 4개 연산이 서로 독립적 → 슈퍼스칼라가 동시에 실행 가능
        a = a * 6364136223846793005ULL + 1;
        b = b * 6364136223846793005ULL + 2;
        c = c * 6364136223846793005ULL + 3;
        d = d * 6364136223846793005ULL + 4;
    }

    clock_gettime(CLOCK_MONOTONIC, &end);

    double ns = (end.tv_sec - start.tv_sec) * 1e9
               + (end.tv_nsec - start.tv_nsec);
    printf("a+b+c+d=%lu\n", a+b+c+d);
    printf("평균 %.2f ns/iter (4개 독립 곱셈)\n", ns / REPS);
    return 0;
}
```

```bash
gcc -O2 -o independent independent.c
./independent

# 예상:
# 의존성 체인 버전: ~1 ns/iter (3 사이클/곱셈)
# 독립 버전:        ~1 ns/iter 이지만 4개를 동시에! (4배 처리량)
#
# 즉, 같은 시간에 4배 더 많은 곱셈 완료됨
# perf stat으로 확인:

perf stat -e cycles,instructions ./latency_chain
perf stat -e cycles,instructions ./independent

# latency_chain: IPC ≈ 0.33
# independent:   IPC ≈ 1.3 (4개 곱셈 동시 → 4/3 ≈ 1.33)
```

---

## 📊 성능 비교

```
파이프라인 유무 및 깊이별 비교:

                     비파이프라인   5단계    현대OoO(4-wide)
─────────────────────────────────────────────────────────────
이론적 IPC              0.2        1.0         4.0
실제 IPC (일반 코드)    0.2        0.8~1.0     2.0~4.0
클럭 주파수 (예)        100MHz     200MHz      3,500MHz
초당 명령어             20M        160~200M    21,000~42,000M
분기 미스 비용          5 사이클    5 사이클    15~20 사이클
캐시 미스의 영향        큼          큼          매우 큼

실제 CPU 파이프라인 깊이:
  Intel Core (Skylake~Raptor Lake): ~14단계 (프론트엔드~백엔드 합산)
  Intel NetBurst (Pentium 4):       31단계 (실패 사례)
  AMD Zen 4:                         ~23단계
  ARM Cortex-A78:                    ~13단계

perf stat IPC 해석 가이드:
  IPC < 1.0  → 메모리 스톨 또는 분기 미스 심각
  IPC 1~2    → 보통 수준, 개선 여지 있음
  IPC 2~4    → 슈퍼스칼라 잘 활용 중
  IPC > 4    → SIMD 명령어 사용 (1 SIMD = 여러 연산)

메모리 접근 패턴별 IPC 차이 (perf 측정):
  순차 배열 합산:       IPC ≈ 3.0 (L1/L2 캐시 히트)
  랜덤 포인터 추적:     IPC ≈ 0.3 (DRAM 레이턴시 대기)
  → IPC 10배 차이, 메모리 접근 패턴이 전부 (Ch2에서 상세)
```

---

## ⚖️ 트레이드오프

```
파이프라인 깊이 설계:

얕은 파이프라인 (5~10단계):
  ✅ 분기 미스 비용 낮음 (5~10 사이클)
  ✅ 단계별 논리 간단 → 설계 단순
  ✅ 제어 흐름 집약적 코드에 유리
  ❌ 단계당 처리 시간 길어 클럭 주파수 한계
  ❌ 현재 3GHz 이상 달성 어려움

깊은 파이프라인 (20~31단계):
  ✅ 단계별 논리 단순 → 높은 클럭 주파수 가능
  ❌ 분기 미스 시 20~31 사이클 낭비 → 성능 붕괴
  ❌ 분기가 많은 코드(일반 애플리케이션)에서 역효과
  ❌ Pentium 4: 이 방향의 실패 사례

균형점 (14~19단계, 현대 주류):
  ✅ 3~5 GHz 클럭 + IPC 3~5 균형
  ✅ 분기 예측 미스 비용 ~15 사이클 (수용 가능)
  ✅ OoO + 슈퍼스칼라와 조합으로 실질 성능 극대화

IPC vs 클럭 주파수 트레이드오프:
  같은 반도체 공정:
    클럭 두 배 → 단위 시간당 사이클 두 배이지만
    파이프라인 더 깊어져 분기 미스 비용 증가
    실질 성능 증가 < 클럭 증가
  
  IPC 두 배 → 더 넓은 발행 폭 필요 → 더 많은 트랜지스터
    전력/열 제약 + 의존성 분석 복잡도 증가
    하지만 분기 미스 비용 그대로 → 균형 잡힌 개선

jvm-deep-dive 연결:
  JVM JIT가 생성하는 코드도 같은 파이프라인을 탑니다
  JIT의 인라이닝, 루프 최적화가 IPC를 높이는 원리가 동일합니다
```

---

## 📌 핵심 정리

```
파이프라인 핵심:

5단계 구조:
  Fetch   → L1i 캐시에서 명령어 바이트 인출
  Decode  → x86 명령어 → µop 변환 + 레지스터 디코드
  Execute → ALU/FPU에서 연산
  Memory  → L1d 캐시 로드/스토어
  Writeback → 레지스터 파일에 결과 기록

IPC의 의미:
  IPC = 1: 파이프라인이 꽉 차서 매 사이클 1 명령어 완료
  IPC < 1: 스톨 발생 (메모리 대기, 분기 미스, 데이터 의존성)
  IPC > 1: 슈퍼스칼라 (한 사이클에 여러 명령어 실행)
  perf stat으로 직접 측정 가능

파이프라인 깊이 트레이드오프:
  깊을수록 클럭 주파수 높아지지만 분기 미스 비용 증가
  현대 주류: 14~19단계로 균형

이 문서에서 다음으로 이어지는 것:
  스톨의 원인 → Ch1-03 파이프라인 해저드
  IPC > 1의 원리 → Ch1-04 슈퍼스칼라와 OoO
  분기 미스 비용 → Ch1-05 분기 예측
  IPC의 한계 → Ch1-06 ILP 한계
  IPC < 1 진단 → Ch7-01 perf 완전 활용
```

---

## 🤔 생각해볼 문제

**Q1.** `perf stat`으로 어떤 프로그램을 측정했더니 IPC = 0.3이 나왔다. 이 수치만으로 병목의 원인을 특정할 수 없다. 원인을 구분하기 위해 어떤 추가 카운터를 봐야 하는가?

<details>
<summary>해설 보기</summary>

IPC = 0.3이 낮은 이유는 크게 두 가지입니다:

**1. 메모리 레이턴시 스톨**:
```bash
perf stat -e cycles,instructions,cache-misses,cache-references \
          -e L1-dcache-load-misses,LLC-load-misses ./program
```
- `LLC-load-misses` (Last Level Cache miss)가 많으면 → DRAM 대기가 병목
- 이 경우: Ch2-01 메모리 계층 문서에서 해결책 탐구

**2. 분기 미스 스톨**:
```bash
perf stat -e cycles,instructions,branches,branch-misses ./program
```
- `branch-misses / branches` 비율이 10% 이상이면 → 분기 예측 실패 병목
- 이 경우: Ch1-05 분기 예측 문서에서 해결책 탐구

**3. 구분 방법**:
```bash
perf stat -e cycles,instructions,cache-misses,branch-misses,stalled-cycles-frontend,stalled-cycles-backend ./program
```
- `stalled-cycles-frontend` 높음 → Fetch/Decode 병목 (코드 캐시 미스 등)
- `stalled-cycles-backend` 높음 → 실행 유닛 대기 (메모리 레이턴시가 주 원인)

</details>

---

**Q2.** Intel Pentium 4 CPU는 31단계 파이프라인으로 3.8 GHz까지 달성했지만 시장에서 실패했다. Core 2 Duo는 14단계 파이프라인에 2.4 GHz였는데 성능이 더 좋았다. 이유를 파이프라인 관점에서 설명하라.

<details>
<summary>해설 보기</summary>

**Pentium 4의 문제**:

클럭 주파수 = 3.8 GHz × 단계당 작업량 역수
단계가 31개로 잘게 쪼개졌으므로 각 단계가 짧아 높은 클럭 달성.

하지만:
- 분기 미스 패널티 = **31 사이클** (파이프라인 플러시)
- 일반 애플리케이션 코드의 분기 미스율 ~5%
- 분기당 31 사이클 × 분기 매 20 명령어마다 = 매 20 명령어 중 1.55 사이클 낭비
- 실제 유효 IPC ≈ 1.0 (설계상 IPC 3+를 의도했지만)

**Core 2 Duo (14단계)의 우위**:
- 분기 미스 패널티 = **14 사이클** (절반 이하)
- 클럭은 낮지만 유효 IPC가 Pentium 4보다 높음
- 실제 SPEC 벤치마크: Core 2 @ 2.4 GHz > Pentium 4 @ 3.8 GHz

**교훈**:
성능 = 클럭 × IPC이고, 파이프라인을 깊게 해서 클럭을 올려도 IPC가 떨어지면 순손실입니다. Intel이 Pentium 4를 포기하고 모바일 지향 Banias(Pentium M) 계보에서 Core 아키텍처를 파생시킨 이유입니다.

</details>

---

**Q3.** 다음 두 루프 중 어느 것이 더 높은 IPC를 가질 것인가? 이유를 파이프라인 관점에서 설명하라.

```c
// 루프 A: 큰 배열 순차 합산
long sum_sequential(int *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) sum += arr[i];
    return sum;
}

// 루프 B: 연결 리스트 순회
struct Node { int val; struct Node *next; };
long sum_list(struct Node *head) {
    long sum = 0;
    for (struct Node *p = head; p; p = p->next) sum += p->val;
    return sum;
}
```

<details>
<summary>해설 보기</summary>

**루프 A(배열 합산)가 압도적으로 높은 IPC**를 가집니다.

**루프 A 분석**:
- 배열 메모리가 연속 → L1 캐시라인(64B)에 16개 int가 함께 로드
- 첫 번째 요소 로드 후 같은 캐시라인의 나머지 15개는 이미 L1에 있음
- `sum += arr[i]`는 레지스터 연산 → 빠른 실행
- CPU 프리페처가 순차 패턴 감지 → 다음 캐시라인 미리 로드
- IPC ≈ 3~4 (L2/L3 캐시 범위라면)

**루프 B 분석**:
- 각 노드가 힙의 다른 위치 → 주소를 예측 불가
- `p = p->next` → 다음 주소를 알려면 현재 메모리 로드를 완료해야 함
- 이 의존성 체인이 파이프라인을 직렬화: 로드 완료(~4~200 사이클) 후 다음 로드
- CPU 프리페처가 랜덤 주소를 예측 불가 → 프리페치 효과 없음
- IPC ≈ 0.1~0.3 (DRAM 캐시 미스 많을 경우)

**측정**:
```bash
perf stat -e cycles,instructions,LLC-load-misses ./루프A
perf stat -e cycles,instructions,LLC-load-misses ./루프B
```

이것이 Ch2-03 지역성 문서에서 "연결 리스트가 배열보다 느린 정확한 이유"로 이어집니다.

</details>

---

<div align="center">

**[⬅️ 이전: C 한 줄이 기계어가 되기까지](./01-c-to-machine-code.md)** | **[홈으로 🏠](../README.md)** | **[다음: 파이프라인 해저드 ➡️](./03-pipeline-hazards.md)**

</div>
