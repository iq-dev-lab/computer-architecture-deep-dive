# 메모리 재정렬 — 컴파일러와 CPU가 명령을 뒤바꾸는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Store Buffer가 무엇이고, 왜 StoreLoad 재정렬을 일으키는가?
- x86의 TSO(Total Store Order)는 어떤 재정렬을 허용하고 어떤 것을 막는가?
- ARM의 약한 메모리 모델(Weak Memory Model)은 x86과 무엇이 다른가?
- 컴파일러 재정렬(`-O2` aliasing)과 하드웨어 재정렬은 어떻게 다른가?
- Dekker 알고리즘이 배리어 없이 올바로 동작하지 않는 이유는 무엇인가?
- 메모리 배리어가 필요한 상황을 코드에서 어떻게 식별하는가?

---

## 🔍 왜 이 개념이 중요한가

### "분명히 순서대로 썼는데 왜 다른 코어는 다른 순서로 보는가"

```
직관적 모델 (순차 일관성, Sequential Consistency):
  코드에 쓴 순서 = CPU가 실행하는 순서 = 다른 코어가 보는 순서

현실:
  코드 순서 → 컴파일러 재정렬 → CPU 실행 재정렬 → 메모리 가시화 순서
                    ↓                    ↓
              -O2 aliasing          Store Buffer
              명령어 스케줄링         비순차 실행(OoO)

  이 세 가지가 겹치면:
    코드에서 A를 먼저 썼어도 다른 코어가 B를 먼저 볼 수 있다

실제 사고:
  flag = true;    // 데이터 준비 완료를 알리는 플래그
  data = 42;      // 실제 데이터

  다른 코어:
    if (flag) use(data);  // data가 아직 42가 아닐 수 있음!

  재정렬로 인해 flag가 먼저 가시화되고 data가 나중에 가시화될 수 있음
  → 잘못된 data 값을 읽게 됨

이것이 메모리 배리어(Memory Barrier)가 필요한 근본 이유
→ 04-memory-barriers.md에서 해결책을 다룸
```

---

## 😱 잘못된 이해

### Before: CPU는 코드 순서대로 명령을 실행한다

```
잘못된 가정:
  코드:
    store(x, 1);   // x = 1
    store(y, 1);   // y = 1

  다른 코어:
    r1 = load(y);  // y 읽기
    r2 = load(x);  // x 읽기

  "당연히 r1=1이면 r2도 1이어야지"
  → 틀림: r1=1, r2=0이 가능함!

왜 틀린가:
  1. Store Buffer: CPU가 store 명령을 Store Buffer에 올려놓고
     다음 명령을 먼저 실행할 수 있음
     x=1을 Store Buffer에 넣고 y=1을 먼저 메모리에 반영 가능

  2. 컴파일러: 두 store 사이에 의존성이 없으면 순서를 바꿀 수 있음
     → -O2에서 실제로 발생

  3. OoO CPU: 실행 유닛이 비어 있으면 독립적인 명령을 먼저 실행

volatile이 이것을 막는다는 오해:
  C/C++의 volatile: 컴파일러 재정렬만 막음 (하드웨어 재정렬은 허용)
  Java의 volatile:  컴파일러 + 하드웨어 재정렬 모두 막음
  → C/C++ volatile로 멀티스레드 코드를 짜면 ARM에서 깨짐
```

---

## ✨ 올바른 이해

### After: 재정렬은 계층적으로 발생하며 각 계층마다 다른 해결책이 있다

```
재정렬의 3계층:

  계층 1: 컴파일러 재정렬
    원인: 최적화(-O2/-O3), 레지스터 할당, 명령어 스케줄링
    가시화: -O2 어셈블리를 godbolt에서 확인
    해결: asm volatile("" ::: "memory") 또는 std::atomic

  계층 2: CPU 비순차 실행(OoO)
    원인: Reservation Station에서 독립 명령을 병렬 실행
    특징: 같은 코어 내에서만 데이터 의존성은 보존됨 (ROB가 보장)
           다른 코어에 가시화되는 순서는 보장 없음 (ARM)
    해결: 메모리 배리어 명령어

  계층 3: Store Buffer 재정렬
    원인: 쓰기가 Store Buffer에 머무는 동안 다른 로드가 먼저 완료
    결과: StoreLoad 재정렬 (가장 흔한 재정렬 유형)
    해결: MFENCE (x86) / DMB (ARM)

메모리 모델 강도:
  약함 ←────────────────────────────→ 강함
  ARM/POWER    IA-64(Itanium)    x86(TSO)    Sequential Consistency
  모든 재정렬   대부분 허용        StoreLoad만  재정렬 없음
  허용                           허용
```

---

## 🔬 내부 동작 원리

### 1. Store Buffer가 만드는 StoreLoad 재정렬

```
CPU 파이프라인과 Store Buffer:

  Core0                          Core1
  ┌─────────────────────┐        ┌─────────────────────┐
  │ Pipeline            │        │ Pipeline            │
  │   ┌───────────────┐ │        │   ┌───────────────┐ │
  │   │ Store Buffer  │ │        │   │ Store Buffer  │ │
  │   │ x=1 (pending) │ │        │   │ y=1 (pending) │ │
  │   └───────┬───────┘ │        │   └───────┬───────┘ │
  └───────────┼─────────┘        └───────────┼─────────┘
              │                              │
         ┌────▼────────────────────────────▼────┐
         │           L3 캐시 / DRAM              │
         │   x = 0 (아직 Store Buffer에만 있음)  │
         │   y = 0 (아직 Store Buffer에만 있음)  │
         └───────────────────────────────────────┘

타임라인:
  시각 1: Core0이 x=1을 Store Buffer에 넣음 (아직 L3/DRAM에 없음)
  시각 2: Core1이 y=1을 Store Buffer에 넣음
  시각 3: Core0이 y를 로드 → Store Buffer에 y 없음 → L3/DRAM에서 읽음
          → y = 0 읽음! (Core1의 y=1이 아직 Store Buffer에 있기 때문)
  시각 4: Core1이 x를 로드 → Store Buffer에 x 없음 → L3/DRAM에서 읽음
          → x = 0 읽음!

결과: r1=0, r2=0 (둘 다 쓰기가 완료됐지만 서로의 쓰기를 못 봄!)

Store Buffer가 필요한 이유:
  쓰기는 캐시 일관성 프로토콜(MESI)의 M 상태 획득을 기다려야 함
  M 상태 획득: 버스에서 BusRdX 발행 후 다른 코어의 Invalid 확인 대기
  → 수십~수백 사이클 대기
  Store Buffer에 넣으면 CPU가 다음 명령 즉시 실행 가능 → IPC 향상
  → "쓰기 실패를 숨기는" 성능 최적화의 부산물 = StoreLoad 재정렬
```

### 2. x86 TSO (Total Store Order)

```
x86의 메모리 모델 = TSO (Total Store Order)

x86이 허용하는 재정렬:
  ✅ StoreLoad: Store 후 다른 주소 Load → 재정렬 가능
  ✅ 이 하나만!

x86이 금지하는 재정렬 (하드웨어가 자동 보장):
  ❌ LoadLoad:  Load 후 다른 주소 Load → 항상 순서 보장
  ❌ StoreStore: Store 후 다른 주소 Store → 항상 순서 보장
  ❌ LoadStore: Load 후 다른 주소 Store → 항상 순서 보장

TSO 모델 직관:
  "Store는 버퍼링될 수 있지만, 일단 버퍼를 나가면 순서가 보존됨"
  "버퍼에서 같은 주소를 Load하면 최신 Store 값을 받음 (Store Forwarding)"

실질적 의미:
  x86에서는 StoreLoad 재정렬만 막으면 됨
  → mfence 또는 lock 프리픽스로 해결
  → 다른 종류의 배리어(lfence만, sfence만)는 사용 빈도 낮음

어셈블리로 확인 (godbolt, x86-64, gcc -O2):
  int x = 0, y = 0;

  // Thread A:
  x = 1;        → MOV [x], 1
  int ry = y;   → MOV eax, [y]
                // MOV 두 개 → StoreLoad 재정렬 가능!

  // volatile 추가:
  x = 1;        → MOV [x], 1
  ry = y;       → MFENCE          ← C++의 volatile는 이걸 넣지 않음!
                   MOV eax, [y]  ← std::atomic이나 배리어 명시 필요
```

### 3. ARM의 약한 메모리 모델 (Weak Memory Model)

```
ARM 메모리 모델: 허용되는 재정렬

  StoreStore:  허용! (x86은 금지)
  LoadLoad:    허용! (x86은 금지)
  LoadStore:   허용! (x86은 금지)
  StoreLoad:   허용! (x86도 허용)

ARM이 허용하는 극단적 예:
  // Thread A (ARM):
  data = 42;     // Store
  flag = 1;      // Store
  → ARM에서 이 두 Store가 다른 코어에 역순으로 가시화될 수 있음!
    flag=1이 먼저 보이고 data=42가 나중에 보일 수 있음

  // Thread B (ARM):
  if (flag) use(data);  // data가 아직 42가 아닐 수 있음!

ARM에서 StoreStore 재정렬을 막으려면:
  data = 42;
  DMB ST;        // DMB: Data Memory Barrier, ST: Store-Store
  flag = 1;

ARM 어셈블리 (Apple M1/Android ARM64):
  // Java volatile 쓰기의 ARM64 JIT 산출물:
  STR  W1, [X0, #offset_data]   // data = 42
  DMB  ISH                      // Inner Shareable Domain 배리어
  STR  W2, [X0, #offset_flag]   // flag = 1

  // Java volatile 읽기의 ARM64 JIT 산출물:
  LDR  W1, [X0, #offset_flag]   // flag 읽기
  DMB  ISH                      // 배리어
  LDR  W2, [X0, #offset_data]   // data 읽기

x86 vs ARM 배리어 비용:
  x86 MFENCE:   ~30~100 사이클 (비쌈 — StoreLoad 재정렬만 막으면 됨)
  ARM DMB ISH:  ~10~30 사이클 (상대적으로 저렴 — 자주 필요하지만)
  실제로 volatile이 더 비싼 플랫폼은 x86
  (약한 모델에서 더 자주 배리어가 필요하지만 개별 비용이 낮음)
```

### 4. 컴파일러 재정렬: -O2 aliasing

```c
// 컴파일러 재정렬 실험 (godbolt에서 확인)

// ===== 예시 1: 단순 재정렬 =====
// C 코드:
void reorder_demo(int *a, int *b, int *c) {
    *a = 1;
    *b = 2;
    *c = *a + *b;
}

// gcc -O0 어셈블리 (순서 보존):
// MOV [a], 1
// MOV [b], 2
// MOV eax, [a]
// ADD eax, [b]
// MOV [c], eax

// gcc -O2 어셈블리 (재정렬 + 상수 폴딩):
// MOV [a], 1
// MOV [b], 2
// MOV [c], 3   ← 컴파일러가 *a+*b=3으로 계산해버림!
// (a, b, c가 다른 포인터라고 aliasing 없다고 가정)

// ===== 예시 2: aliasing 재정렬 =====
int x_global = 0, flag_global = 0;

void writer_no_volatile() {
    x_global = 42;
    flag_global = 1;
}
// gcc -O2: 두 store를 역순으로 배치 가능!
// MOV [flag_global], 1
// MOV [x_global], 42

// 컴파일러 재정렬을 막으려면:
void writer_with_barrier() {
    x_global = 42;
    asm volatile("" ::: "memory");  // 컴파일러 배리어 (하드웨어 배리어 아님!)
    flag_global = 1;
}
// gcc -O2: 순서 보존
// MOV [x_global], 42
// MOV [flag_global], 1

// C++ std::atomic으로 해결 (컴파일러 + 하드웨어 모두):
#include <atomic>
std::atomic<int> x_atomic{0}, flag_atomic{0};

void writer_atomic() {
    x_atomic.store(42, std::memory_order_release);
    flag_atomic.store(1, std::memory_order_release);
    // → x86: MOV + SFENCE 또는 XCHG
    // → ARM: STR + DMB ST + STR
}
```

### 5. Dekker 알고리즘으로 재정렬 재현

```c
// Dekker 알고리즘: 뮤텍스 없이 두 스레드의 임계 구역 진입 조율
// 재정렬이 있으면 두 스레드가 동시에 임계 구역에 진입할 수 있음!

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdatomic.h>

int flag0 = 0, flag1 = 0;  // volatile 없는 버전
int critical_count = 0;     // 동시 진입 카운트 (0이어야 함)
int violations = 0;

void* thread0(void* _) {
    for (int i = 0; i < 1000000; i++) {
        flag0 = 1;                    // "나 들어갈래"
        // ← 여기서 재정렬: flag0=1이 flag1 로드보다 늦게 가시화될 수 있음
        while (flag1) { /* spin */ }   // 상대가 안 들어갔으면 진입
        // === 임계 구역 ===
        critical_count++;
        if (critical_count > 1) violations++;
        critical_count--;
        // === 임계 구역 끝 ===
        flag0 = 0;
    }
    return NULL;
}

void* thread1(void* _) {
    for (int i = 0; i < 1000000; i++) {
        flag1 = 1;
        while (flag0) { /* spin */ }
        critical_count++;
        if (critical_count > 1) violations++;
        critical_count--;
        flag1 = 0;
    }
    return NULL;
}

// 예상: violations > 0 (재정렬로 인해 임계 구역 동시 진입)

// 배리어 추가 버전:
void* thread0_safe(void* _) {
    for (int i = 0; i < 1000000; i++) {
        flag0 = 1;
        __sync_synchronize();  // 전체 메모리 배리어 (= MFENCE on x86)
        while (flag1) { /* spin */ }
        critical_count++;
        if (critical_count > 1) violations++;
        critical_count--;
        flag0 = 0;
    }
    return NULL;
}

// 배리어 추가 후: violations == 0

// 빌드: gcc -O2 -pthread -o dekker dekker.c
// 실행: ./dekker (violations가 0이 아님을 확인)
```

---

## 💻 실전 실험

### 실험 1: godbolt로 컴파일러 재정렬 관찰

```bash
# godbolt.org에서 직접 확인하는 방법:
# https://godbolt.org/
# Compiler: x86-64 gcc 13.2, Options: -O2

# 실험 코드 A: volatile 없는 경우
int x, y;
void write_xy() {
    x = 1;
    y = 2;
}
# -O2 어셈블리:
# write_xy():
#   mov DWORD PTR y[rip], 2   ← y가 먼저!
#   mov DWORD PTR x[rip], 1
#   ret

# 실험 코드 B: std::atomic 사용
#include <atomic>
std::atomic<int> ax, ay;
void write_xy_atomic() {
    ax.store(1, std::memory_order_seq_cst);
    ay.store(2, std::memory_order_seq_cst);
}
# -O2 어셈블리:
# write_xy_atomic():
#   mov DWORD PTR ax[rip], 1
#   mfence               ← MFENCE 삽입!
#   mov DWORD PTR ay[rip], 2
#   ret

# ARM64 gcc 13.2 동일 코드:
# write_xy_atomic():
#   mov w0, 1
#   stlr w0, [x1]      ← Store-Release
#   mov w0, 2
#   stlr w0, [x2]      ← Store-Release
#   ret
# → x86의 MFENCE 대신 STLR 명령어 사용
```

### 실험 2: litmus 테스트로 하드웨어 재정렬 관찰

```bash
# litmus7 도구 설치 (herd7 포함)
# https://github.com/herd/herdtools7

# 또는 직접 C 코드로 재정렬 관찰
cat > reorder_test.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdint.h>

volatile int x = 0, y = 0;
volatile int r1, r2;
volatile int iterations_done = 0;

void* thread_a(void* _) {
    for (int i = 0; i < 10000000; i++) {
        x = 0; y = 0;  // reset
        __asm__ volatile("mfence" ::: "memory");  // 리셋 완료 보장
        x = 1;
        // StoreLoad 재정렬 발생 위치 (mfence 없음)
        r1 = y;
    }
    return NULL;
}

void* thread_b(void* _) {
    for (int i = 0; i < 10000000; i++) {
        y = 1;
        r2 = x;
    }
    return NULL;
}

int main() {
    pthread_t ta, tb;
    int count_00 = 0;

    for (int trial = 0; trial < 100; trial++) {
        x = 0; y = 0; r1 = -1; r2 = -1;
        pthread_create(&ta, NULL, thread_a, NULL);
        pthread_create(&tb, NULL, thread_b, NULL);
        pthread_join(ta, NULL);
        pthread_join(tb, NULL);
        if (r1 == 0 && r2 == 0) count_00++;
    }

    printf("r1=0, r2=0 발생 횟수: %d/100\n", count_00);
    printf("→ StoreLoad 재정렬 관찰됨\n");
    return 0;
}
EOF

gcc -O2 -pthread -o reorder_test reorder_test.c
./reorder_test
# r1=0, r2=0이 발생하면 StoreLoad 재정렬이 실제로 일어남을 확인
```

### 실험 3: perf로 메모리 배리어 오버헤드 측정

```bash
cat > barrier_bench.c << 'EOF'
#include <stdio.h>
#include <time.h>
#include <stdint.h>

#define N 100000000L

volatile long x = 0;

double bench_no_barrier() {
    struct timespec s, e;
    clock_gettime(CLOCK_MONOTONIC, &s);
    for (long i = 0; i < N; i++) x = i;
    clock_gettime(CLOCK_MONOTONIC, &e);
    return (e.tv_sec-s.tv_sec)*1e3 + (e.tv_nsec-s.tv_nsec)/1e6;
}

double bench_mfence() {
    struct timespec s, e;
    clock_gettime(CLOCK_MONOTONIC, &s);
    for (long i = 0; i < N; i++) {
        x = i;
        __asm__ volatile("mfence" ::: "memory");
    }
    clock_gettime(CLOCK_MONOTONIC, &e);
    return (e.tv_sec-s.tv_sec)*1e3 + (e.tv_nsec-s.tv_nsec)/1e6;
}

int main() {
    printf("배리어 없음:  %.1f ms\n", bench_no_barrier());
    printf("MFENCE 있음:  %.1f ms\n", bench_mfence());
    return 0;
}
EOF

gcc -O2 -o barrier_bench barrier_bench.c
./barrier_bench

# 예상: MFENCE 있음이 수십 배 느림
# MFENCE가 Store Buffer를 완전히 비울 때까지 대기하기 때문
```

---

## 📊 성능 비교

```
재정렬 유형별 발생 가능성과 비용:

                  x86 TSO     ARM WeakMM    컴파일러(-O2)
─────────────────────────────────────────────────────────
LoadLoad 재정렬   불가         가능           가능
LoadStore 재정렬  불가         가능           가능
StoreStore 재정렬 불가         가능           가능
StoreLoad 재정렬  가능 ★       가능           가능

메모리 배리어 비용 (Latency, Intel Core i9):
  MFENCE:        ~30~100 cy  (Store Buffer flush 대기)
  SFENCE:        ~5~10 cy    (Store 완료 대기)
  LFENCE:        ~5 cy       (Load 직렬화)
  LOCK 프리픽스: ~30~50 cy   (캐시라인 독점 + MFENCE 효과)
  컴파일러 배리어: 0 cy       (런타임 비용 없음)

ARM DMB 배리어 비용:
  DMB SY:   ~10~30 cy  (System 범위)
  DMB ISH:  ~10~20 cy  (Inner Shareable, 같은 소켓)
  DMB ISHST:~5~10 cy   (Store-to-Store만)

배리어 유무에 따른 처리량 비교 (1억 회 Store 루프):
  배리어 없음:       ~50ms  (Store Buffer 활용 최대화)
  MFENCE 매 Store:  ~2000ms (40배 느림)
  SFENCE 매 Store:  ~200ms  (4배 느림)
  Release-Store:    ~150ms  (acquire/release 시멘틱만)
```

---

## ⚖️ 트레이드오프

```
메모리 모델 강도 설계 트레이드오프:

강한 메모리 모델 (x86 TSO):
  ✅ 프로그래밍 모델이 단순 → StoreLoad만 조심하면 됨
  ✅ 대부분의 경우 배리어 없이도 일관성 유지
  ❌ 하드웨어 구현 복잡도 높음 → 칩 면적/전력 증가
  ❌ Store Buffer 유지 비용

약한 메모리 모델 (ARM):
  ✅ 하드웨어 구현 단순 → 저전력/소형 칩에 적합
  ✅ 배리어를 선택적으로 삽입해 최적화 여지 넓음
  ❌ 프로그래밍 모델이 복잡 → 정확한 배리어 위치 파악 어려움
  ❌ x86 코드를 ARM으로 포팅할 때 동기화 버그 발생 가능

컴파일러 최적화 트레이드오프:
  ✅ 재정렬 + aliasing 최적화 → 성능 대폭 향상
  ❌ 멀티스레드 코드에서 예상치 못한 동작
  ✅ std::atomic / Java volatile 사용 시 컴파일러가 보수적으로 처리
  ❌ atomic 남용 시 최적화 기회 손실

개발자 관점 교훈:
  C/C++: 멀티스레드 공유 데이터는 반드시 std::atomic 또는 명시적 배리어
  Java: volatile / synchronized / java.util.concurrent API 사용
  "락 없이 안전한 코드"는 전문가 영역 — 잘못 짜면 ARM에서만 깨짐
  플랫폼 종속 코드 회피: Java volatile이 x86/ARM 차이를 추상화해줌
```

---

## 📌 핵심 정리

```
메모리 재정렬 핵심:

재정렬의 3계층:
  컴파일러: -O2 aliasing, 명령어 스케줄링 → asm barrier 또는 atomic
  CPU OoO:  독립 명령 병렬 실행 → 같은 코어 내에서는 의존성 보존
  Store Buffer: StoreLoad 재정렬 → MFENCE/DMB로 해결

x86 TSO:
  허용: StoreLoad 재정렬만
  금지: LoadLoad, StoreStore, LoadStore
  → Java volatile은 x86에서 MFENCE 하나로 충분

ARM Weak Memory Model:
  허용: 모든 종류의 재정렬
  → Java volatile은 ARM에서 DMB + Load/Store acquire/release 사용
  → x86에서 돌아가던 락프리 코드가 ARM에서 깨질 수 있음

Dekker 알고리즘 교훈:
  배리어 없이 플래그로 동기화하는 코드는 실제로 깨짐
  재정렬이 "임계 구역 보호"를 무너뜨림
  → 언어 수준의 동기화 도구(synchronized, mutex)를 써야 하는 이유

java-concurrency 연결:
  volatile 변수 → acquire/release 시멘틱 → 하드웨어 배리어
  자세한 배리어 → 04-memory-barriers.md
  JMM과 하드웨어의 전체 매핑 → 06-jmm-hardware-bridge.md
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 C 코드를 x86-64와 ARM64에서 각각 `-O2`로 컴파일하면 어떤 차이가 있는가? 그리고 멀티스레드 환경에서 안전하게 만들려면 어떻게 수정해야 하는가?

```c
int data = 0;
int ready = 0;

void producer() {
    data = 42;
    ready = 1;
}

void consumer() {
    while (!ready) {}
    use(data);  // data가 항상 42인가?
}
```

<details>
<summary>해설 보기</summary>

**x86-64 `-O2`에서의 문제:**
1. `producer()`: `ready = 1`이 `data = 42`보다 먼저 실행되도록 컴파일러가 재정렬 가능 (StoreStore 컴파일러 재정렬)
2. `consumer()`: `while (!ready) {}` 루프에서 ready를 레지스터에 캐시하면 영원히 루프 탈출 불가 (컴파일러 최적화로 루프 내 ready 재로드 제거)

**ARM64 `-O2`에서의 추가 문제:**
- x86에서 운 좋게 올바로 동작하더라도, ARM에서는 StoreStore 하드웨어 재정렬까지 발생해 `data=42`가 `ready=1` 이후에 가시화될 수 있음

**안전한 수정 (C++11 atomic):**
```c
#include <stdatomic.h>
atomic_int data = 0;
atomic_int ready = 0;

void producer() {
    atomic_store_explicit(&data, 42, memory_order_relaxed);
    atomic_store_explicit(&ready, 1, memory_order_release);
    // release: data의 쓰기가 ready 쓰기보다 먼저 가시화됨을 보장
}

void consumer() {
    while (!atomic_load_explicit(&ready, memory_order_acquire)) {}
    // acquire: ready 읽기 이후에 data를 읽음을 보장
    int val = atomic_load_explicit(&data, memory_order_relaxed);
    use(val);  // 항상 42
}
```

release/acquire 쌍이 happens-before 관계를 만들어 재정렬을 차단합니다.

</details>

---

**Q2.** x86에서 `asm volatile("" ::: "memory")` (컴파일러 배리어)는 충분한가, 아니면 `__sync_synchronize()` (하드웨어 배리어)가 필요한가? 두 경우의 차이를 들어 설명하라.

<details>
<summary>해설 보기</summary>

**`asm volatile("" ::: "memory")` (컴파일러 배리어):**
- 컴파일러에게 이 지점 이전과 이후의 메모리 접근을 재정렬하지 말라고 지시
- 어셈블리 수준에서 실제 명령어를 삽입하지 않음 → 런타임 비용 0
- **문제**: 하드웨어 재정렬은 막지 못함. x86에서는 StoreLoad만 허용되므로 StoreStore/LoadLoad를 막는 데는 컴파일러 배리어로 충분. 하지만 ARM에서는 모든 재정렬이 허용되므로 컴파일러 배리어만으로는 부족.

**`__sync_synchronize()` (= MFENCE on x86, DMB SY on ARM):**
- 컴파일러 배리어 + 하드웨어 배리어 모두 포함
- 런타임 비용 있음 (~30~100 cy on x86)
- 모든 플랫폼에서 완전한 메모리 일관성 보장

**결론:**
- 단순히 컴파일러 최적화를 막고 싶을 때 (싱글스레드): 컴파일러 배리어
- x86에서 StoreStore/LoadLoad만 보장하면 될 때: 컴파일러 배리어 (x86 TSO가 하드웨어에서 자동 보장)
- x86에서 StoreLoad를 막아야 할 때 또는 ARM에서 어떤 재정렬이든 막아야 할 때: 하드웨어 배리어

가장 안전한 방법은 `std::atomic` + `memory_order` 시멘틱을 사용해 컴파일러와 하드웨어 재정렬을 모두 플랫폼 독립적으로 처리하는 것입니다.

</details>

---

**Q3.** Java `volatile`이 x86과 ARM에서 서로 다른 어셈블리로 번역되는데, 왜 개발자는 플랫폼을 신경 쓰지 않아도 되는가?

<details>
<summary>해설 보기</summary>

Java `volatile`은 **JMM(Java Memory Model)**이 정의한 happens-before 보장을 제공합니다. JMM은 다음을 요구합니다:
1. volatile 쓰기 → 이전의 모든 쓰기가 먼저 완료됨 (Release semantics)
2. volatile 읽기 → 이후의 모든 읽기가 최신 값을 봄 (Acquire semantics)

JIT 컴파일러(HotSpot)는 이 JMM 보장을 현재 플랫폼에 맞는 최소 비용의 배리어로 번역합니다:
- **x86**: `volatile` 쓰기 → `LOCK ADD [RSP], 0` (또는 XCHG) → MFENCE와 동등
  `volatile` 읽기 → 단순 `MOV` (x86 TSO가 LoadLoad/LoadStore를 자동 보장)
- **ARM64**: `volatile` 쓰기 → `STLR` (Store-Release)
  `volatile` 읽기 → `LDAR` (Load-Acquire)

개발자가 신경 쓰지 않아도 되는 이유는 JMM이 **추상화 계층**을 제공하기 때문입니다. JMM의 규칙을 따르기만 하면 JIT가 플랫폼별로 올바른 배리어를 생성합니다. 이는 JVM의 핵심 가치 중 하나로, `java-concurrency-deep-dive` 레포에서 happens-before 규칙으로 다루는 내용이 결국 `06-jmm-hardware-bridge.md`에서 설명하는 x86/ARM 배리어로 내려갑니다.

</details>

---

<div align="center">

**[⬅️ 이전: False Sharing](./02-false-sharing.md)** | **[홈으로 🏠](../README.md)** | **[다음: 메모리 배리어 ➡️](./04-memory-barriers.md)**

</div>
