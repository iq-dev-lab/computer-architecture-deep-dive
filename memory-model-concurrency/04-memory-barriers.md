# 메모리 배리어 — Load/Store 배리어의 하드웨어 해석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `mfence` / `lfence` / `sfence`는 각각 무엇을 비우고, 언제 사용하는가?
- C의 `volatile`, C++ `std::atomic`, Java `volatile`은 어떤 배리어 명령으로 번역되는가?
- acquire semantics와 release semantics는 어떤 배리어와 대응되는가?
- LoadLoad / StoreStore / LoadStore / StoreLoad 네 가지 배리어는 어떻게 다른가?
- ARM의 `dmb` / `dsb` / `isb`는 x86 배리어와 어떻게 대응되는가?
- 배리어 없이도 일관성이 보장되는 경우가 있는가?

---

## 🔍 왜 이 개념이 중요한가

### "메모리 배리어는 CPU에게 보내는 명령서다"

```
03-memory-reordering.md의 결론:
  Store Buffer가 StoreLoad 재정렬을 만들고
  ARM의 약한 모델은 모든 종류의 재정렬을 허용한다

이 문제의 해결책이 메모리 배리어(Memory Barrier / Memory Fence)다

배리어 없이 쓴 코드:
  thread A: data = 42; flag = 1;
  thread B: while (!flag); use(data);  // data가 42가 아닐 수 있음!

배리어 있는 코드 (acquire/release 시멘틱):
  thread A: data = 42;
            RELEASE 배리어;    // 이전 Store들이 모두 완료된 후 flag 공개
            flag = 1;
  thread B: while (!flag);
            ACQUIRE 배리어;    // flag 확인 이후에만 data 읽기 허용
            use(data);        // 항상 42 보장

메모리 배리어 = "이 지점을 넘어 재정렬하지 말라"는 하드웨어 명령
  → CPU에게 Store Buffer를 비우거나
  → Load 큐를 직렬화하거나
  → 두 가지 모두를 강제함
```

---

## 😱 잘못된 이해

### Before: C의 `volatile`이 멀티스레드 동기화를 보장한다

```c
// 잘못된 코드 (C/C++ volatile은 하드웨어 재정렬을 막지 않음)
volatile int data = 0;
volatile int flag = 0;

// Thread A:
void producer() {
    data = 42;   // volatile → 컴파일러 재정렬은 막음
    flag = 1;    // 그러나 ARM에서는 하드웨어가 이 순서를 바꿀 수 있음!
}

// Thread B:
void consumer() {
    while (!flag) {}
    int v = data;  // ARM에서 data가 아직 0일 수 있음!
    use(v);
}

// C volatile의 실제 보장:
//   - 컴파일러가 해당 변수 읽기/쓰기를 최적화로 제거하지 않음
//   - 컴파일러 수준의 재정렬 방지
//   - 하드웨어 재정렬은 전혀 막지 않음!
//   - x86에서 우연히 돌아가도 ARM, POWER에서 깨짐

// Java volatile과의 혼동:
//   Java volatile → JMM이 acquire/release 시멘틱 보장 → 하드웨어 배리어 삽입
//   C volatile    → 컴파일러 최적화 억제만 → 배리어 없음
//   이름은 같지만 보장 범위가 전혀 다르다!
```

---

## ✨ 올바른 이해

### After: 배리어의 종류와 목적을 층위별로 파악한다

```
배리어 분류:

  계층 1: 컴파일러 배리어 (Software Barrier)
    → 어셈블리 명령어 미삽입, 런타임 비용 0
    → 컴파일러에게 이 지점 전후로 재정렬 금지 지시
    → asm volatile("" ::: "memory") in C/C++

  계층 2: 하드웨어 배리어 (Hardware Fence)
    → 실제 CPU 명령어 삽입, 런타임 비용 있음
    → CPU의 Store Buffer·Load 큐를 제어
    x86:  mfence / lfence / sfence
    ARM:  dmb / dsb / isb

  계층 3: 고수준 추상화
    → 플랫폼 독립적 배리어 의미론
    → C++11 std::atomic::memory_order_*
    → Java volatile / synchronized

4가지 배리어 유형:
  LoadLoad  (LL):  Load1 → 배리어 → Load2
    "Load1의 결과가 확정된 후에만 Load2 실행"
  StoreStore (SS): Store1 → 배리어 → Store2
    "Store1이 가시화된 후에만 Store2 실행"
  LoadStore (LS):  Load1 → 배리어 → Store2
    "Load1 완료 후 Store2 실행 (드문 패턴)"
  StoreLoad (SL):  Store1 → 배리어 → Load2
    "Store1이 모든 코어에 가시화된 후 Load2 실행"
    → 가장 비싸고 가장 강력한 배리어
    → MFENCE = SL 배리어 (+ LL + SS + LS 모두 포함)
```

---

## 🔬 내부 동작 원리

### 1. x86 배리어 명령어 해부: `mfence` / `lfence` / `sfence`

```
x86 배리어 명령어의 정확한 의미:

MFENCE (Memory Fence):
  - Store Buffer를 완전히 비울 때까지 대기 (모든 pending Store가 캐시/메모리에 반영)
  - 이후의 모든 Load가 최신 값을 보도록 직렬화
  - 효과: LoadLoad + StoreStore + LoadStore + StoreLoad 모두 막음
  - 용도: volatile 쓰기 이후, CAS, seq_cst atomic store
  - 비용: ~30~100 사이클 (Store Buffer flush 대기)

SFENCE (Store Fence):
  - Store Buffer에 있는 모든 Store가 캐시에 반영될 때까지 대기
  - Load에는 영향 없음
  - 효과: StoreStore 배리어
  - 용도: Non-Temporal Store (MOVNTQ 등) 이후 일관성 보장
  - 비용: ~5~15 사이클
  - 주의: x86 TSO에서 일반 Store는 이미 StoreStore 순서가 보장됨
          → 일반 코드에서 SFENCE를 직접 쓸 일은 드묾

LFENCE (Load Fence):
  - 이전의 모든 명령이 로컬 코어에서 완료될 때까지 대기
  - Load 직렬화 (Load들이 프로그램 순서대로 완료됨을 보장)
  - 효과: LoadLoad 배리어 + 약한 직렬화
  - 용도: Spectre 완화 (투기적 실행 차단), RDTSC 정밀 측정 전
  - 비용: ~5 사이클
  - 주의: x86 TSO에서 LoadLoad는 이미 보장 → 일반 동기화보다 Spectre 용도

x86에서 배리어가 필요한 유일한 상황:
  StoreLoad 재정렬을 막아야 할 때 → MFENCE (또는 LOCK 프리픽스)
  그 외 LoadLoad/StoreStore/LoadStore → TSO가 하드웨어에서 자동 보장

메모리 레이아웃 관점:
  MFENCE 실행 시:
    ┌──────────────────────────────────┐
    │ Store Buffer (pending writes)    │
    │  [x=1] [y=2] [z=3] ...          │
    │     ↓↓↓ flush (모두 비움) ↓↓↓    │
    └──────────────────────────────────┘
    ↓
    ┌──────────────────────────────────┐
    │ L1/L2/L3 캐시 (전파 완료)        │
    │  x=1, y=2, z=3 가시화됨          │
    └──────────────────────────────────┘
```

### 2. acquire / release semantics와 배리어 대응

```
Acquire Semantics (획득 시멘틱):
  "이 읽기 이후의 모든 읽기·쓰기는 이 읽기 완료 후에만 실행"
  = LoadLoad + LoadStore 배리어
  = "잠금을 획득한 후 임계 구역에 진입"과 같은 개념

Release Semantics (해제 시멘틱):
  "이 쓰기 이전의 모든 읽기·쓰기가 먼저 완료됨을 보장"
  = LoadStore + StoreStore 배리어
  = "임계 구역을 벗어나며 잠금을 해제"와 같은 개념

Acquire-Release 쌍 = happens-before 관계 형성:
  Thread A: data = 42;            // 일반 store
            flag.store(1, release); // release: data=42가 먼저 가시화됨 보장
  Thread B: while (!flag.load(acquire)) {} // acquire: flag 확인 후 data 접근
            use(data);              // 항상 42 (happens-before 보장)

x86에서 acquire/release의 어셈블리:
  release store:
    MOV [flag], 1        // 일반 MOV로 충분 (TSO가 StoreStore 보장)
    → SFENCE 불필요 (x86이 자동 보장)
    → 단, seq_cst가 필요하면 LOCK 또는 MFENCE 추가

  acquire load:
    MOV eax, [flag]      // 일반 MOV로 충분 (TSO가 LoadLoad 보장)
    → LFENCE 불필요 (x86이 자동 보장)

  seq_cst store (가장 강한 보장):
    XCHG [flag], eax     // LOCK implicit인 XCHG
    또는
    MOV [flag], 1
    MFENCE               // StoreLoad 차단

ARM64에서 acquire/release 어셈블리:
  release store:
    STLR W0, [X1]        // Store-Release → StoreStore + LoadStore 배리어 내장

  acquire load:
    LDAR W0, [X1]        // Load-Acquire → LoadLoad + LoadStore 배리어 내장

  full barrier:
    DMB ISH              // Data Memory Barrier, Inner Shareable

배리어 강도 비교:
  가장 약함 ←──────────────────────────────→ 가장 강함
  relaxed       acquire/release       seq_cst
  배리어 없음    한 방향 배리어         양방향 완전 배리어
```

### 3. C++ `std::atomic` → 어셈블리 변환 (godbolt 확인)

```cpp
// godbolt.org: x86-64 gcc 13.2, -O2
// URL 예시: https://godbolt.org/z/atomics

#include <atomic>

std::atomic<int> data{0}, flag{0};

// ===== relaxed: 배리어 없음 =====
void store_relaxed() {
    data.store(42, std::memory_order_relaxed);
    flag.store(1,  std::memory_order_relaxed);
}
// x86-64 어셈블리:
//   mov DWORD PTR data[rip], 42    ; 단순 MOV
//   mov DWORD PTR flag[rip], 1     ; 단순 MOV
//   ret
// → 배리어 없음, 재정렬 가능

// ===== release/acquire: 단방향 배리어 =====
void store_release() {
    data.store(42, std::memory_order_relaxed);
    flag.store(1,  std::memory_order_release);
}
// x86-64 어셈블리:
//   mov DWORD PTR data[rip], 42    ; 먼저
//   mov DWORD PTR flag[rip], 1     ; TSO가 StoreStore 보장 → 추가 명령 없음
//   ret
// → x86 TSO에서 release store는 단순 MOV로 충분!

int load_acquire() {
    return flag.load(std::memory_order_acquire);
}
// x86-64 어셈블리:
//   mov eax, DWORD PTR flag[rip]   ; 단순 MOV
//   ret
// → x86 TSO에서 acquire load도 단순 MOV로 충분!

// ===== seq_cst: 완전 배리어 =====
void store_seqcst() {
    data.store(42, std::memory_order_relaxed);
    flag.store(1,  std::memory_order_seq_cst);
}
// x86-64 어셈블리:
//   mov DWORD PTR data[rip], 42
//   xchg DWORD PTR flag[rip], edi  ; XCHG = 암묵적 LOCK (MFENCE와 동등)
//   ret
// → seq_cst store: XCHG로 StoreLoad 차단

// ===== ARM64 동일 코드 비교 =====
// store_release() ARM64 어셈블리:
//   mov w0, 42
//   str w0, [data주소]              ; 일반 STR (data)
//   mov w0, 1
//   stlr w0, [flag주소]             ; STLR: Store-Release (StoreStore 배리어 내장)
//   ret
// → ARM은 x86보다 더 명시적 배리어 필요

// load_acquire() ARM64 어셈블리:
//   ldar w0, [flag주소]             ; LDAR: Load-Acquire (LoadLoad 배리어 내장)
//   ret
```

### 4. Java `volatile`이 어떤 배리어로 번역되는가

```java
// JVM: HotSpot JIT (C2 컴파일러)
// 확인 방법: -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly
//           또는 JITWatch (https://github.com/AdoptOpenJDK/jitwatch)

class VolatileExample {
    volatile int data = 0;
    volatile int flag = 0;

    void producer() {
        data = 42;   // volatile 쓰기
        flag = 1;    // volatile 쓰기
    }

    int consumer() {
        int f = flag;   // volatile 읽기
        int d = data;   // volatile 읽기
        return f + d;
    }
}

// ===== x86-64 JIT 어셈블리 (HotSpot C2, -server) =====
// producer():
//   mov DWORD PTR [rdx+0x10], 0x2a    ; data = 42 (일반 MOV)
//   lock add DWORD PTR [rsp], 0x0     ; volatile StoreStore 배리어
//                                     ; = MFENCE와 동등 효과
//   mov DWORD PTR [rdx+0x14], 0x1     ; flag = 1
//   lock add DWORD PTR [rsp], 0x0     ; volatile 후 배리어
//
// consumer():
//   mov eax, DWORD PTR [rdx+0x14]     ; flag 읽기 (acquire: 단순 MOV)
//   mov ecx, DWORD PTR [rdx+0x10]     ; data 읽기 (acquire: 단순 MOV)
//   add eax, ecx
//   ret
// → x86에서 volatile 읽기는 배리어 없음 (TSO가 LoadLoad 보장)
// → volatile 쓰기는 LOCK ADD 로 StoreLoad 막음

// ===== ARM64 JIT 어셈블리 (HotSpot C2) =====
// producer():
//   mov w2, #42
//   stlr w2, [x1, #16]    ; STLR (Store-Release) = data volatile 쓰기
//   mov w2, #1
//   stlr w2, [x1, #20]    ; STLR (Store-Release) = flag volatile 쓰기
//
// consumer():
//   ldar w0, [x1, #20]    ; LDAR (Load-Acquire) = flag volatile 읽기
//   ldar w1, [x1, #16]    ; LDAR (Load-Acquire) = data volatile 읽기
//   add w0, w0, w1
//   ret
// → ARM에서는 volatile 읽기도 LDAR (Load-Acquire) 사용
// → 읽기도 배리어 명령 필요 (x86과 다름!)

// JITWatch 사용법:
//   1. java -XX:+UnlockDiagnosticVMOptions \
//           -XX:+LogCompilation \
//           -XX:+PrintAssembly \
//           -XX:PrintAssemblyOptions=intel \
//           VolatileExample > jit.log 2>&1
//   2. JITWatch에서 jit.log 열기
//   3. Source → Bytecode → Assembly 3단 비교 가능
```

### 5. ARM `dmb` / `dsb` / `isb` 비교

```
ARM 배리어 명령어 계층:

ISB (Instruction Synchronization Barrier):
  - CPU 파이프라인을 완전히 비움 (Flush)
  - 이후 명령이 업데이트된 시스템 레지스터를 반영함을 보장
  - 용도: 컨텍스트 스위치, 캐시 초기화, 코드 자기수정 후
  - 메모리 접근 순서화와는 무관
  - 비용: 매우 비쌈

DSB (Data Synchronization Barrier):
  - 이전의 모든 메모리 접근이 완료될 때까지 대기
  - DSB 이후의 명령은 메모리 접근이 완전히 완료된 후에만 실행
  - 용도: TLB flush 후 확인, 캐시 유지보수 완료 확인
  - DMB보다 강력 (메모리 뿐 아니라 시스템 수준 직렬화)
  - 비용: 매우 비쌈

DMB (Data Memory Barrier):
  - 이전의 메모리 접근이 이후 메모리 접근보다 먼저 완료됨을 보장
  - 파이프라인은 멈추지 않음 (DSB보다 가벼움)
  - 범위 파라미터:
    SY    (System):              전체 시스템 (소켓 간)
    ISH   (Inner Shareable):     같은 Inner Shareable 도메인 (일반적 멀티코어 동기화)
    OSH   (Outer Shareable):     outer shareable 도메인 포함
    ISHST (Inner Sh. Store):     Store-to-Store만
    ISHLD (Inner Sh. Load):      Load-to-Load만
  - 용도: 일반 멀티스레드 동기화
  - 비용: ~10~30 사이클 (ISH 기준)

x86 ↔ ARM 배리어 대응표:
  x86           ARM              보장 내용
  ─────────────────────────────────────────────────────
  mfence        dmb ish          LoadLoad+StoreStore+LoadStore+StoreLoad
  sfence        dmb ishst        StoreStore (Non-temporal store 후)
  lfence        dmb ishld        LoadLoad (Spectre 완화에서는 csdb)
  lock 프리픽스  dmb ish (full)   원자적 읽기-수정-쓰기 + 전체 배리어
  xchg          ldxr/stlxr 루프  Compare-and-Swap과 유사
  (없음)        isb              파이프라인 플러시 (ARM 특유)

실용 예 (ARM64 C 코드):
  // producer (data-before-flag 보장):
  __asm__ volatile("dmb ishst" ::: "memory");  // StoreStore 배리어
  또는
  __atomic_store_n(&flag, 1, __ATOMIC_RELEASE); // GCC 내장

  // consumer:
  while (!__atomic_load_n(&flag, __ATOMIC_ACQUIRE)) {}
  // LDAR 명령으로 변환됨 → LoadLoad 배리어 내장
```

---

## 💻 실전 실험

### 실험 1: godbolt에서 배리어 어셈블리 확인

```cpp
// godbolt.org: https://godbolt.org/
// Compiler: x86-64 gcc 13.2, -O2
// 같은 코드를 ARM64 gcc 13.2로 전환하면 ARM 어셈블리 확인 가능

#include <atomic>
#include <cstdint>

std::atomic<int> g_flag{0};
int g_data = 0;

// ── 실험 A: 각 memory_order가 생성하는 어셈블리 비교 ──

void test_relaxed() {
    g_data = 42;
    g_flag.store(1, std::memory_order_relaxed);
}
// x86: mov + mov (배리어 없음)
// ARM: str + str (배리어 없음)

void test_release() {
    g_data = 42;
    g_flag.store(1, std::memory_order_release);
}
// x86: mov + mov (TSO가 StoreStore 보장 → 추가 명령 없음)
// ARM: str + stlr (STLR = Store-Release)

void test_seqcst() {
    g_data = 42;
    g_flag.store(1, std::memory_order_seq_cst);
}
// x86: mov + xchg (XCHG = 암묵적 LOCK → StoreLoad 차단)
// ARM: str + stlr + dmb ish (또는 stlr만으로 충분한 경우도 있음)

int test_load_acquire() {
    return g_flag.load(std::memory_order_acquire);
}
// x86: mov (단순 MOV, 추가 배리어 없음)
// ARM: ldar (Load-Acquire → LoadLoad 배리어 내장)

// ── 실험 B: C volatile vs std::atomic 차이 ──
volatile int cv_flag = 0;

void cvolatile_store() {
    g_data = 42;
    cv_flag = 1;
}
// x86: mov + mov (컴파일러 재정렬은 막음, 하드웨어 배리어 없음!)
// ARM64: str + str (배리어 없음! → ARM에서 unsafe)
```

### 실험 2: 배리어 비용 측정

```bash
# 배리어 유형별 처리량 비교 실험
cat > barrier_cost.cpp << 'EOF'
#include <atomic>
#include <chrono>
#include <cstdio>

const long N = 100'000'000L;

std::atomic<long> counter{0};
long plain_var = 0;

auto now() { return std::chrono::high_resolution_clock::now(); }
double ms(auto s, auto e) {
    return std::chrono::duration_cast<std::chrono::microseconds>(e-s).count() / 1000.0;
}

int main() {
    // 1. 배리어 없음 (relaxed)
    auto t0 = now();
    for (long i = 0; i < N; i++)
        counter.store(i, std::memory_order_relaxed);
    auto t1 = now();

    // 2. release
    for (long i = 0; i < N; i++)
        counter.store(i, std::memory_order_release);
    auto t2 = now();

    // 3. seq_cst (XCHG / MFENCE)
    for (long i = 0; i < N; i++)
        counter.store(i, std::memory_order_seq_cst);
    auto t3 = now();

    // 4. 수동 MFENCE
    for (long i = 0; i < N; i++) {
        plain_var = i;
        __asm__ volatile("mfence" ::: "memory");
    }
    auto t4 = now();

    printf("relaxed:      %8.1f ms\n", ms(t0, t1));
    printf("release:      %8.1f ms\n", ms(t1, t2));
    printf("seq_cst:      %8.1f ms\n", ms(t2, t3));
    printf("수동 mfence:  %8.1f ms\n", ms(t3, t4));
    return 0;
}
EOF

g++ -O2 -std=c++17 -o barrier_cost barrier_cost.cpp
./barrier_cost

# 예상 출력 (Intel Core i7-12700, x86-64):
# relaxed:          85.3 ms
# release:          86.1 ms  ← x86에서는 relaxed와 거의 동일!
# seq_cst:        1840.7 ms  ← ~21배 느림 (XCHG/MFENCE 비용)
# 수동 mfence:    1920.4 ms  ← seq_cst와 비슷

# perf로 사이클 단위 확인
perf stat -e cycles,instructions,mem_uops_retired.all_stores \
  ./barrier_cost
```

### 실험 3: MFENCE가 필요한 시나리오 재현

```c
// StoreLoad 재정렬 → MFENCE로 해결하는 실험
#include <pthread.h>
#include <stdio.h>
#include <stdint.h>

volatile int x = 0, y = 0;
volatile int r1, r2;
long violations = 0;
const int TRIALS = 1000000;

// 버전 A: mfence 없음 (StoreLoad 재정렬 발생 가능)
void* thread_a_no_fence(void* _) {
    for (int i = 0; i < TRIALS; i++) {
        x = 1;
        // StoreLoad 재정렬: y 읽기가 x=1 쓰기보다 먼저 가시화될 수 있음
        r1 = y;
    }
    return NULL;
}

void* thread_b_no_fence(void* _) {
    for (int i = 0; i < TRIALS; i++) {
        y = 1;
        r2 = x;
    }
    return NULL;
}

// 버전 B: mfence 있음
void* thread_a_fence(void* _) {
    for (int i = 0; i < TRIALS; i++) {
        x = 1;
        __asm__ volatile("mfence" ::: "memory");  // Store Buffer flush
        r1 = y;
    }
    return NULL;
}

void* thread_b_fence(void* _) {
    for (int i = 0; i < TRIALS; i++) {
        y = 1;
        __asm__ volatile("mfence" ::: "memory");
        r2 = x;
    }
    return NULL;
}

int test_reorder(void*(*fa)(void*), void*(*fb)(void*)) {
    int count = 0;
    for (int trial = 0; trial < 100; trial++) {
        x = 0; y = 0; r1 = -1; r2 = -1;
        pthread_t ta, tb;
        pthread_create(&ta, NULL, fa, NULL);
        pthread_create(&tb, NULL, fb, NULL);
        pthread_join(ta, NULL);
        pthread_join(tb, NULL);
        if (r1 == 0 && r2 == 0) count++;  // 재정렬 발생
    }
    return count;
}

int main() {
    printf("mfence 없음: r1=0,r2=0 발생 %d/100회 (재정렬 관찰)\n",
           test_reorder(thread_a_no_fence, thread_b_no_fence));
    printf("mfence 있음: r1=0,r2=0 발생 %d/100회 (재정렬 없음)\n",
           test_reorder(thread_a_fence, thread_b_fence));
    return 0;
}

// gcc -O2 -pthread -o fence_test fence_test.c
// ./fence_test
// → 첫 번째 줄: 수십~수백회 발생 (재정렬 실제로 일어남)
// → 두 번째 줄: 0회 (mfence가 StoreLoad 재정렬 차단)
```

### 실험 4: Java volatile 어셈블리 확인 (JITWatch)

```bash
# HotSpot JIT 어셈블리 출력 준비
# 1. hsdis 라이브러리 설치 (어셈블리 프린트용)
#    Ubuntu: apt-get install libhsdis
#    또는 https://github.com/liuzhengyang/hsdis 빌드

# 2. Java 코드 실행 (어셈블리 출력)
cat > VolatileBarrier.java << 'EOF'
public class VolatileBarrier {
    volatile int flag = 0;
    int data = 0;

    public void producer() {
        data = 42;
        flag = 1;  // volatile 쓰기
    }

    public int consumer() {
        int f = flag;  // volatile 읽기
        return f + data;
    }

    public static void main(String[] args) throws Exception {
        VolatileBarrier v = new VolatileBarrier();
        // JIT 컴파일 트리거를 위한 워밍업
        for (int i = 0; i < 100_000; i++) {
            v.producer();
            v.consumer();
        }
    }
}
EOF

# 어셈블리 출력 (hsdis 필요):
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintAssembly \
     -XX:PrintAssemblyOptions=intel \
     -XX:CompileCommand=print,VolatileBarrier.producer \
     -XX:CompileCommand=print,VolatileBarrier.consumer \
     VolatileBarrier 2>&1 | grep -A 20 "producer\|consumer"

# x86 예상 출력 (producer):
#   0x...  mov  DWORD PTR [rdi+0x10], 0x2a    ; data = 42
#   0x...  lock add DWORD PTR [rsp+0x0], 0x0  ; volatile 배리어
#   0x...  mov  DWORD PTR [rdi+0x14], 0x1     ; flag = 1 (volatile)

# JITWatch GUI 사용법:
# git clone https://github.com/AdoptOpenJDK/jitwatch
# cd jitwatch && mvn install
# java -jar jitwatch-ui/target/jitwatch-ui-*.jar
# → LogCompilation 파일 로드 → Source/Bytecode/Assembly 3단 뷰
```

---

## 📊 성능 비교

```
배리어 유형별 단일 스레드 처리량 (Intel Core i7-12700, x86-64):

명령 유형                처리량(ns/op)  배리어 없음 대비 비율
────────────────────────────────────────────────────────────
일반 store (MOV)             0.3 ns         1.0x
volatile store (C)           0.3 ns         1.0x  ← 하드웨어 배리어 없음!
release store (std::atomic)  0.35 ns        1.2x  ← TSO가 자동 보장
acquire load (std::atomic)   0.3 ns         1.0x  ← TSO가 자동 보장
seq_cst store (XCHG)         18 ns          60x   ← Store Buffer flush
수동 mfence                  19 ns          63x
수동 sfence                  2.1 ns         7x
수동 lfence                  1.2 ns         4x

ARM64 (Apple M1) 배리어 비용:
명령 유형                처리량(ns/op)
────────────────────────────────────────
일반 store (STR)             0.25 ns
STLR (Store-Release)         1.1 ns    ← release store, ~4배
LDAR (Load-Acquire)          0.8 ns    ← acquire load, ~3배
DMB ISH (full barrier)       3.5 ns    ← ~14배

x86 vs ARM 배리어 비용 비교:
  x86: release/acquire → 거의 무료 (TSO 덕분)
       seq_cst → 비쌈 (MFENCE/XCHG)
  ARM: release → 중간 (STLR)
       acquire → 중간 (LDAR)
       seq_cst → STLR + LDAR

Java volatile 쓰기/읽기 비용 (HotSpot on x86-64):
  volatile 쓰기: ~18 ns (LOCK ADD = MFENCE와 동등)
  volatile 읽기: ~0.3 ns (단순 MOV)
  일반 쓰기:     ~0.3 ns
  → volatile 쓰기가 읽기보다 약 60배 비쌈
  → 읽기 집약적 패턴에서 volatile 비용은 매우 낮음
```

---

## ⚖️ 트레이드오프

```
배리어 강도 선택의 트레이드오프:

relaxed (배리어 없음):
  ✅ 가장 빠름 (단순 MOV)
  ✅ 카운터 누산, 통계 수집 같이 정확한 순서가 불필요한 경우
  ❌ 동기화 의미 없음: 플래그/신호 전달에 사용 불가
  ❌ 컴파일러와 CPU 모두 자유롭게 재정렬

acquire/release (단방향 배리어):
  ✅ 대부분의 동기화 패턴에 충분 (플래그 전달, 잠금 구현)
  ✅ x86에서 거의 무료 (TSO가 대부분 자동 보장)
  ✅ ARM에서도 STLR/LDAR로 최소 비용
  ❌ seq_cst보다 약한 보장 (전체 순서 없음)

seq_cst (완전 배리어):
  ✅ 가장 강한 보장: 모든 스레드가 동일한 순서 관찰
  ✅ 잘못 짜기 어려움 → 기본값으로 시작 후 필요 시 완화
  ❌ x86에서 비쌈 (MFENCE/XCHG → ~60배)
  ❌ ARM에서도 비쌈

Java volatile의 특성:
  ✅ 자동으로 플랫폼 최적 배리어 선택
  ✅ JMM이 happens-before 보장 → 정확성 쉽게 추론
  ❌ 쓰기 비용이 읽기보다 월등히 높음 (x86: ~60배)
  ❌ 복합 연산(read-modify-write)은 원자성 보장 안 됨
     → AtomicInteger.incrementAndGet() 필요

배리어를 쓰지 말아야 할 경우:
  - 단일 스레드 접근만 있는 변수: 배리어 불필요
  - 읽기 전용 상수: 배리어 없음
  - 이미 외부 동기화(mutex, synchronized)로 보호된 경우:
    락 진입/탈출 자체가 배리어 역할 → 내부에서 추가 배리어 불필요
```

---

## 📌 핵심 정리

```
메모리 배리어 핵심:

x86 배리어:
  mfence: 모든 재정렬 차단 (Store Buffer flush + Load 직렬화) → ~60x 비용
  sfence: StoreStore 배리어 (Non-temporal store 용)
  lfence: LoadLoad 직렬화 (Spectre 완화 주 용도)
  → TSO 덕분에 LoadLoad/StoreStore/LoadStore는 자동 보장
  → StoreLoad만 mfence/lock으로 막으면 됨

ARM 배리어:
  dmb ish:   멀티코어 동기화의 표준 (LoadLoad+StoreStore+LoadStore+StoreLoad)
  stlr/ldar: Store-Release / Load-Acquire (Java volatile에 매핑)
  dsb:       시스템 수준 직렬화 (커널/드라이버용)
  isb:       파이프라인 플러시 (컨텍스트 스위치 후)

Acquire/Release 시멘틱:
  release store = StoreStore + LoadStore 배리어 (이전 작업 완료 후 공개)
  acquire load  = LoadLoad + LoadStore 배리어 (확인 후 접근)
  → 두 가지를 쌍으로 사용하면 happens-before 관계 성립

언어별 배리어 번역:
  C volatile:     컴파일러 재정렬만 막음 → ARM에서 unsafe
  C++ atomic:     memory_order에 따라 플랫폼 최적 배리어 생성
  Java volatile:  JMM acquire/release → x86: LOCK ADD, ARM: STLR/LDAR

java-concurrency 연결:
  volatile → happens-before → 배리어 (이 문서)
  synchronized → 모니터 진입/탈출 → 배리어 (06-jmm-hardware-bridge.md)
  AtomicLong.compareAndSet → lock cmpxchg (05-atomic-cas.md)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 Java 코드에서 `x86-64`와 `ARM64` 각각에서 `volatile` 쓰기가 생성하는 어셈블리 명령어는 무엇인가? 그리고 왜 ARM64에서 비용이 더 높은가?

```java
class Flag {
    volatile boolean ready = false;
    int value = 0;

    void publish(int v) {
        value = v;
        ready = true;   // volatile 쓰기
    }
}
```

<details>
<summary>해설 보기</summary>

**x86-64 JIT 어셈블리:**

```
mov DWORD PTR [rdx+offset_value], edi      ; value = v (일반 store)
mov BYTE PTR [rdx+offset_ready], 0x1       ; ready = true (volatile store)
lock add DWORD PTR [rsp], 0x0              ; StoreLoad 배리어 (MFENCE 효과)
```

x86의 TSO가 StoreStore 순서(`value` → `ready`)를 하드웨어에서 자동 보장합니다. `LOCK ADD`는 이후의 Load가 최신 Store를 보도록 StoreLoad 재정렬만 막습니다.

**ARM64 JIT 어셈블리:**

```
str w1, [x0, #offset_value]     ; value = v (일반 STR)
mov w2, #1
stlr w2, [x0, #offset_ready]   ; ready = true (Store-Release)
```

ARM은 TSO가 없어 StoreStore 재정렬이 가능합니다. 따라서 `STLR` (Store-Release) 명령을 사용해 `value`의 store가 `ready`의 store보다 먼저 모든 코어에 가시화됨을 보장합니다.

**비용 차이 이유:**

- x86: TSO가 하드웨어에서 이미 StoreStore를 보장 → 추가 명령이 LOCK ADD (매우 가볍지 않지만, Store 순서 자체는 공짜)
- ARM: 하드웨어가 재정렬을 자유롭게 허용 → STLR이 Store 완료까지 대기 (~4배 오버헤드)

ARM이 단순한 하드웨어로 저전력을 달성하는 대가가 바로 더 많은 명시적 배리어 명령입니다.

</details>

---

**Q2.** C++ `std::atomic<int>` 에서 `memory_order_seq_cst`가 기본값인 이유와, `memory_order_release`/`memory_order_acquire` 쌍으로 교체할 수 있는 조건은 무엇인가?

<details>
<summary>해설 보기</summary>

**seq_cst가 기본값인 이유:**

`memory_order_seq_cst`는 모든 스레드가 모든 atomic 연산을 동일한 순서로 관찰하는 **단일 전체 순서(single total order)**를 보장합니다. 이는 추론이 가장 쉬운 모델로, 기본값으로 삼아 정확성을 먼저 확보하고 성능이 문제가 될 때 완화하는 전략이 안전합니다.

**release/acquire로 교체 가능한 조건:**

release/acquire 쌍은 특정 두 스레드 사이의 happens-before 관계만 성립시킵니다. 전체 순서가 필요 없고 다음 조건이 충족되면 교체 가능합니다:

1. **생산자-소비자 패턴**: Thread A가 데이터를 쓰고 flag를 release-store, Thread B가 acquire-load로 flag를 읽은 후 데이터에 접근 → 완전히 안전
2. **단방향 의존성**: "A가 쓴 것을 B가 읽는다"는 방향이 명확한 경우
3. **3개 이상 스레드의 복잡한 관찰 순서가 불필요한 경우**

교체 **불가** 조건:
- 여러 스레드가 서로 다른 atomic 변수들의 연산 순서를 동일하게 관찰해야 하는 경우
- 잠금 구현처럼 모든 스레드가 mutex 획득 순서에 동의해야 하는 경우

실용 가이드: 먼저 seq_cst로 짜고 perf로 병목이 확인되면, acquire/release로 최소한으로 완화합니다. x86에서 release/acquire는 거의 무료이므로 x86 전용 코드라면 성능 차이가 미미할 수 있습니다.

</details>

---

**Q3.** Java `synchronized` 블록 진입과 탈출 시 어떤 배리어가 삽입되는가? 그리고 `synchronized` 내부에서 `volatile` 변수를 추가로 읽는 것이 의미 없는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

**`synchronized` 블록의 배리어:**

- **진입(monitorenter)**: 잠금 획득 → **acquire 배리어** 삽입
  - 이후의 모든 읽기/쓰기가 잠금 획득 이후에 실행됨을 보장 (LoadLoad + LoadStore)
  - x86: LOCK CMPXCHG (잠금 획득 시도) → 내재된 MFENCE 효과
  - ARM: LDAXR (Load-Acquire Exclusive) → LDAR의 독점 버전

- **탈출(monitorexit)**: 잠금 해제 → **release 배리어** 삽입
  - 이전의 모든 읽기/쓰기가 잠금 해제 이전에 완료됨을 보장 (LoadStore + StoreStore)
  - x86: LOCK 프리픽스 연산으로 Store Buffer flush
  - ARM: STLXR (Store-Release Exclusive) → STLR의 독점 버전

**`synchronized` 내부에서 `volatile` 읽기가 불필요한 이유:**

`synchronized`가 이미 acquire/release 배리어를 제공하므로, 내부의 변수는 진입 시점에 최신 값을 보고 탈출 시점에 모든 쓰기가 가시화됩니다. 그 내부에서 `volatile`을 추가로 읽어도:

1. acquire 배리어(진입)가 이미 이후 모든 읽기에 최신 값을 보장
2. release 배리어(탈출)가 이미 이전 모든 쓰기의 가시화를 보장

따라서 `synchronized` 블록 안에서의 `volatile` 읽기는 중복 배리어입니다. 오히려 `volatile` 쓰기가 추가 LOCK ADD를 발생시켜 불필요한 비용을 더할 수 있습니다. 단, `synchronized` 외부에서도 해당 변수에 접근하는 스레드가 있다면 `volatile`이 의미가 있습니다 — 잠금 없이 읽는 스레드에게 가시성을 보장하기 위해서입니다.

자세한 내용은 `06-jmm-hardware-bridge.md`에서 JMM의 전체 happens-before 규칙과 배리어 매핑표로 다룹니다.

</details>

---

<div align="center">

**[⬅️ 이전: 메모리 재정렬](./03-memory-reordering.md)** | **[홈으로 🏠](../README.md)** | **[다음: 원자적 연산과 CAS ➡️](./05-atomic-cas.md)**

</div>
