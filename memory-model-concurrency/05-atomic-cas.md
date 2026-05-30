# 원자적 연산과 CAS — `lock` 프리픽스의 정체

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- x86의 `lock` 프리픽스는 캐시 일관성 프로토콜을 어떻게 가로채는가?
- `lock cmpxchg` 명령어를 어셈블리 수준에서 분해하면 어떤 과정이 일어나는가?
- Compare-And-Swap은 하드웨어에서 어떻게 원자성을 보장하는가?
- ABA 문제란 무엇이고, 태그된 포인터(tagged pointer)로 어떻게 회피하는가?
- 락프리 스택의 기본 구조는 어떻게 CAS로 구현되는가?
- CAS 기반 알고리즘의 성능 한계는 무엇인가?

---

## 🔍 왜 이 개념이 중요한가

### "뮤텍스 없이 원자적 연산을 구현하는 메커니즘"

```
전통적 동기화:
  mutex.lock();
  counter++;    // 임계 구역: 한 번에 한 스레드만
  mutex.unlock();

  문제:
    mutex는 비쌈: OS 커널 진입, 컨텍스트 스위치, 스케줄링 대기
    경합이 심할 때: 대기 스레드가 CPU를 낭비하거나 잠재운다
    우선순위 역전, 데드락 위험

락프리(Lock-Free) 대안:
  counter.fetch_add(1, std::memory_order_relaxed);
  // 또는 Java: counter.incrementAndGet();

  내부에서 일어나는 일:
    1. 현재 값을 읽는다
    2. +1한 값을 준비한다
    3. "현재 값이 여전히 같으면 새 값으로 교체" → CAS
    4. 실패하면 1로 돌아가 재시도

  이 CAS(Compare-And-Swap)가 어떻게 원자적으로 실행되는가?
  → x86에서는 lock cmpxchg 명령어 하나로 구현된다

실무 연결:
  AtomicLong.compareAndSet()     → lock cmpxchg
  ConcurrentHashMap의 버킷 갱신  → lock cmpxchg
  LongAdder의 Cell 업데이트      → lock cmpxchg
  AQS(AbstractQueuedSynchronizer) state 변경 → lock cmpxchg
```

---

## 😱 잘못된 이해

### Before: 원자적 연산은 "작은 뮤텍스"다

```
잘못된 생각 1: lock cmpxchg가 버스를 완전히 멈춘다

  이전 시대 인식 (멀티소켓, 전용 버스):
    LOCK 프리픽스 → 버스 잠금(Bus Lock) → 다른 CPU 전부 대기
    → 모든 메모리 접근이 직렬화 → 매우 비쌈

  현대 CPU의 실제 (캐시 기반 일관성):
    LOCK 프리픽스 → 해당 캐시라인의 독점적 소유(M 상태) 획득 → 라인 단위 잠금
    버스 전체를 잠그지 않음!
    → 다른 라인에 대한 접근은 동시에 진행 가능

잘못된 생각 2: CAS 한 번이면 항상 성공한다

  고경합 환경에서:
    Thread A: CAS(old=0, new=1) 성도
    Thread B: CAS(old=0, new=1) 실패! (old가 이미 1)
    Thread B: 재시도 → CAS(old=1, new=2) 성공
    Thread C: 재시도 × 3 ...

  "CAS 폭풍(CAS storm)":
    스레드 수 N → 평균 N/2 CAS 재시도 발생
    성능이 O(N²)으로 저하될 수 있음
    → LongAdder가 AtomicLong보다 고경합에서 빠른 이유

잘못된 생각 3: CAS는 ABA 문제에서 안전하다
  → CAS는 "값이 같으면 교체"만 확인
  → 값이 A→B→A로 바뀌었다가 원래대로 돌아오면 CAS는 변경을 감지 못함
```

---

## ✨ 올바른 이해

### After: `lock` 프리픽스는 캐시라인 독점 획득이다

```
현대 x86의 LOCK 구현:

  LOCK 프리픽스 없음:
    명령 실행 → Store Buffer에 쌓임 → 나중에 캐시에 반영
    다른 코어가 중간에 끼어들 수 있음 → 원자성 없음

  LOCK 프리픽스 있음:
    1. 해당 메모리 주소의 캐시라인 획득
    2. MESI에서 M(Modified) 상태로 독점
       → 다른 코어가 해당 라인을 S→I로 무효화됨
    3. 원자적 읽기-수정-쓰기 실행
    4. 라인 M 상태 유지 (다른 코어의 접근 차단)
    5. 완료 후 일반 MESI 프로토콜로 복귀

  핵심: "버스 잠금"이 아니라 "캐시라인 독점 잠금"
    → 같은 CPU 내 다른 코어가 다른 캐시라인에 접근하는 것은 허용
    → 같은 라인에 대한 다른 코어의 접근만 차단

  x86에서 LOCK 가능한 명령어:
    ADD, SUB, AND, OR, XOR, INC, DEC, NOT, NEG (메모리 대상)
    XCHG (항상 LOCK implicit)
    CMPXCHG (Compare and Exchange)
    CMPXCHG8B / CMPXCHG16B (64/128비트)
    BTS, BTR, BTC (비트 조작)
    XADD (Exchange and Add → AtomicLong.getAndAdd)
```

---

## 🔬 내부 동작 원리

### 1. `lock cmpxchg` 어셈블리 해부

```asm
; CAS (Compare-And-Swap) 의사 코드:
;   if (*addr == expected) { *addr = desired; return true; }
;   else { expected = *addr; return false; }

; x86-64 어셈블리 (Intel syntax):
; 가정: rdi = addr (메모리 주소), rax = expected, rdx = desired

lock cmpxchg DWORD PTR [rdi], edx
; 해석:
;   1. [rdi]의 캐시라인을 M 상태로 독점 획득 (LOCK)
;   2. EAX (expected)와 [rdi] (현재값) 비교 (CMP)
;   3. 같으면: [rdi] = EDX (desired) 기록, ZF=1 (성공)
;              EAX 값은 변경 없음
;   4. 다르면: EAX = [rdi] (현재값으로 업데이트), ZF=0 (실패)
;              [rdi] 값은 변경 없음
; 결과 확인:
je  success_label     ; ZF=1 → 성공
; ZF=0 → 실패, EAX에 현재값 들어있음

; ─────────────────────────────────────────────────────
; C++ std::atomic<int> a{0};
; bool result = a.compare_exchange_strong(expected, desired);
; ↓ godbolt (x86-64 gcc -O2) ↓
;
; compare_exchange_strong(rdi, esi, edx):
;   mov eax, esi               ; expected를 EAX에 로드 (cmpxchg 규약)
;   lock cmpxchg [rdi], edx    ; 원자적 비교 교환
;   sete al                    ; ZF → AL (true/false 반환)
;   movzx eax, al
;   ret

; ─────────────────────────────────────────────────────
; Java AtomicInteger.compareAndSet(int expect, int update)
; JIT 산출물 (x86-64, HotSpot C2):
;
;   mov eax, [expect_addr]     ; expected
;   lock cmpxchg [obj+offset], [update_reg]
;   sete al
;   movzx eax, al
;   ret
; → Java와 C++의 JIT/컴파일 산출물이 동일한 명령어!
```

### 2. CAS 루프: `fetch_add`의 실제 구현

```cpp
// 설명용 CAS 루프 (실제 하드웨어 동작 원리)
// 실제 fetch_add는 LOCK XADD 하나로 처리되지만
// CAS 루프가 원자성이 어떻게 유지되는지 보여줌

// godbolt에서 확인:
// https://godbolt.org/
// Compiler: x86-64 gcc 13.2, -O2

#include <atomic>

std::atomic<int> counter{0};

// 방법 1: fetch_add → LOCK XADD 하나로 번역
int increment_v1() {
    return counter.fetch_add(1, std::memory_order_seq_cst);
}
// 어셈블리:
//   mov eax, 1
//   lock xadd DWORD PTR counter[rip], eax  ; 원자적으로 현재값 읽고 +1
//   ret

// 방법 2: CAS 루프 (compare_exchange_weak)
int increment_v2_cas_loop() {
    int old = counter.load(std::memory_order_relaxed);
    while (!counter.compare_exchange_weak(
        old, old + 1,
        std::memory_order_seq_cst,    // 성공 시
        std::memory_order_relaxed))   // 실패 시
    { /* retry */ }
    return old;
}
// 어셈블리 (루프):
//   .loop:
//     mov  eax, DWORD PTR counter[rip]   ; load (relaxed)
//     lea  ecx, [rax+1]                  ; old+1 계산
//     lock cmpxchg DWORD PTR counter[rip], ecx  ; CAS 시도
//     jne  .loop                          ; ZF=0 → 실패, 재시도
//   ret

// compare_exchange_weak vs compare_exchange_strong:
//   weak:   Spurious failure 허용 (아키텍처에 따라 LL/SC 실패 시)
//           루프 안에서 사용 → 루프가 어차피 재시도
//   strong: Spurious failure 없음 (내부에서 재시도)
//           단독 if 조건에서 사용
//
// ARM의 LL/SC (Load-Linked / Store-Conditional):
//   LDXR (Load Exclusive Register): 캐시라인을 "예약"
//   STXR (Store Exclusive Register): 예약이 유효하면 쓰고 성공, 아니면 실패
//   → x86 cmpxchg보다 더 세밀한 제어 가능
//   → weak CAS가 ARM에서 가장 효율적

// ARM64 어셈블리 (increment_v2_cas_loop):
// .loop:
//   ldxr w0, [x19]          ; Load Exclusive
//   add  w1, w0, #1
//   stxr w2, w1, [x19]      ; Store Exclusive (w2에 성공/실패)
//   cbnz w2, .loop           ; w2 != 0 → 실패, 재시도
//   ret
```

### 3. ABA 문제 시나리오

```
ABA 문제 (락프리 스택 예시):

초기 상태:
  스택 top → [A] → [B] → [C] → null

Thread 1이 pop 시도 (선점 직전):
  old_top = A  (읽음)
  // ← 여기서 Thread 1이 선점됨

Thread 2가 pop 두 번, push 한 번 실행:
  pop() → A 제거: top → [B] → [C] → null
  pop() → B 제거: top → [C] → null
  push(A) → A를 메모리에서 재사용: top → [A] → [C] → null
             (A의 next 포인터는 이제 C를 가리킴!)

Thread 1이 재개:
  CAS(top, A, B) 실행
  → top이 여전히 A처럼 보임 (Thread 2가 A를 재사용했기 때문)
  → CAS 성공! top = B가 됨
  → 그런데 B는 이미 팝된 노드! (해제된 메모리)
  → top → [B] → ??? (B의 next는 이미 해제됨)
  → 메모리 손상 또는 잘못된 스택 상태!

ABA 문제의 본질:
  CAS는 "값이 같은가"만 확인
  "중간에 바뀌었다가 돌아온 것"은 감지 못함
  포인터가 같은 주소를 가리키면 같은 노드처럼 보임

발생 조건:
  1. 포인터/주소를 CAS로 비교
  2. 메모리 재사용 (malloc/free 패턴에서 같은 주소 재할당)
  3. 오래된 참조(stale reference)로 CAS 시도
```

### 4. 태그된 포인터 (Tagged Pointer)로 ABA 회피

```cpp
// ABA 해결책: 포인터에 버전 번호(stamp/tag) 결합
// x86-64에서 CMPXCHG16B (128비트 CAS)를 활용

#include <atomic>
#include <cstdint>
#include <cstddef>

// 방법 1: 16B CAS (포인터 8B + 태그 8B)
struct TaggedPtr {
    void* ptr;
    uint64_t tag;  // 버전 번호
};

// __int128 또는 pair로 표현
std::atomic<TaggedPtr> top;  // 128비트 원자적 연산 지원 필요

// CMPXCHG16B 어셈블리:
// RDX:RAX = expected (128비트)
// RCX:RBX = desired (128비트)
// lock cmpxchg16b [top]
// → 16바이트 단위로 원자적 비교-교환

// 방법 2: 포인터 하위 비트 활용 (32비트 주소 공간에서)
// 64비트 가상 주소의 상위 비트 활용 (AMD64 규약: 상위 16비트 부호 확장)
// 실제로 사용 가능한 주소 비트: 48비트
// 남은 16비트를 태그로 사용

struct TaggedPointer {
    static constexpr uint64_t TAG_BITS = 16;
    static constexpr uint64_t PTR_BITS = 48;
    static constexpr uint64_t TAG_MASK = 0xFFFF000000000000ULL;
    static constexpr uint64_t PTR_MASK = 0x0000FFFFFFFFFFFFULL;

    uint64_t value;

    TaggedPointer(void* ptr, uint16_t tag) {
        value = ((uint64_t)tag << PTR_BITS) | ((uint64_t)ptr & PTR_MASK);
    }

    void* get_ptr() const {
        uint64_t p = value & PTR_MASK;
        // 부호 확장 복원 (canonical form)
        if (p & (1ULL << 47)) p |= TAG_MASK;
        return (void*)p;
    }

    uint16_t get_tag() const {
        return (uint16_t)(value >> PTR_BITS);
    }
};

// 방법 3: Java AtomicStampedReference / AtomicMarkableReference
// Java의 표준 ABA 해결책

import java.util.concurrent.atomic.AtomicStampedReference;

// stamp(버전 번호)와 참조를 함께 CAS
AtomicStampedReference<Node> top = new AtomicStampedReference<>(null, 0);

void push(Node newNode) {
    int[] stampHolder = new int[1];
    while (true) {
        Node oldTop = top.get(stampHolder);  // 참조 + stamp 동시 읽기
        int oldStamp = stampHolder[0];
        newNode.next = oldTop;
        if (top.compareAndSet(oldTop, newNode, oldStamp, oldStamp + 1)) {
            // stamp를 +1 → A→B→A라도 stamp가 달라 CAS 실패
            break;
        }
    }
}

// AtomicStampedReference 내부:
// Pair<V> { V reference; int stamp; }
// → 두 값을 원자적으로 비교 (Java 객체 참조 + int)
// → 내부적으로 참조 비교(reference equality)로 구현
// → stamp 덕분에 ABA 감지 가능
```

### 5. 락프리 스택 기본 구조

```cpp
// Treiber Stack: 가장 단순한 락프리 스택
// 1986년 R. Kent Treiber가 제안

#include <atomic>
#include <optional>

template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(T val) : data(val), next(nullptr) {}
    };

    std::atomic<Node*> top_{nullptr};

public:
    void push(T val) {
        Node* new_node = new Node(val);
        // CAS 루프: 다른 스레드가 top을 바꿀 수 있으므로 재시도
        Node* old_top = top_.load(std::memory_order_relaxed);
        do {
            new_node->next = old_top;
        } while (!top_.compare_exchange_weak(
            old_top,                           // expected (실패 시 자동 갱신)
            new_node,                          // desired
            std::memory_order_release,         // 성공 시: new_node의 data 쓰기가 완료됨 보장
            std::memory_order_relaxed));       // 실패 시: 재시도만
    }

    std::optional<T> pop() {
        Node* old_top = top_.load(std::memory_order_relaxed);
        while (old_top != nullptr) {
            Node* new_top = old_top->next;
            if (top_.compare_exchange_weak(
                old_top,                       // expected
                new_top,                       // desired (top을 다음 노드로)
                std::memory_order_acquire,     // 성공 시: 이후 data 읽기가 최신값 보장
                std::memory_order_relaxed)) {
                T val = old_top->data;
                // 주의: delete old_top → ABA 문제 발생 가능!
                // 해결: Hazard Pointer, RCU, 메모리 풀 등 사용
                delete old_top;                // 단순화를 위해 여기서는 delete
                return val;
            }
            // compare_exchange_weak 실패 시 old_top이 자동으로 최신값으로 갱신됨
        }
        return std::nullopt;
    }
};

// godbolt에서 push()의 핵심 어셈블리 (x86-64, -O2):
// push(T val):
//   ...
// .retry:
//   mov rax, QWORD PTR [top_]        ; old_top = top_.load()
//   mov [new_node+8], rax             ; new_node->next = old_top
//   lock cmpxchg QWORD PTR [top_], rdx ; top_ = new_node if top_ == old_top
//   jne .retry                        ; 실패 시 재시도
//   ret

// 성능 특성:
//   경합 없을 때: push/pop = 1 LOCK CMPXCHG 성공
//   N개 스레드 고경합: 평균 N/2 CAS 재시도 → O(N) 재시도
//   → 고경합에서 성능 저하 (contented CAS)
//   실용적 대안: 각 스레드에 로컬 큐 → 최종 합산 (like LongAdder)

// Java 버전 (java-concurrency-deep-dive 연결):
import java.util.concurrent.atomic.AtomicReference;

public class LockFreeStack<T> {
    private static class Node<T> {
        final T data;
        Node<T> next;
        Node(T data) { this.data = data; }
    }

    private final AtomicReference<Node<T>> top = new AtomicReference<>(null);

    public void push(T val) {
        Node<T> newNode = new Node<>(val);
        Node<T> oldTop;
        do {
            oldTop = top.get();
            newNode.next = oldTop;
        } while (!top.compareAndSet(oldTop, newNode));
        // compareAndSet → lock cmpxchg (x86) 또는 LDXR/STXR 루프 (ARM)
    }

    public T pop() {
        Node<T> oldTop, newTop;
        do {
            oldTop = top.get();
            if (oldTop == null) return null;
            newTop = oldTop.next;
        } while (!top.compareAndSet(oldTop, newTop));
        return oldTop.data;
        // Java GC가 oldTop 회수 → ABA 문제 완화
        // (GC가 같은 주소 재사용을 방지: 동일 epoch 내에서는 재할당 안 됨)
    }
}
```

---

## 💻 실전 실험

### 실험 1: godbolt로 CAS 어셈블리 확인

```cpp
// godbolt.org: x86-64 gcc 13.2, Options: -O2
// ARM64: ARM64 gcc 13.2, -O2

#include <atomic>
#include <cstdint>

std::atomic<int> g_counter{0};
std::atomic<int*> g_ptr{nullptr};

// ── 실험 A: fetch_add vs CAS 루프 ──
int experiment_xadd() {
    return g_counter.fetch_add(1, std::memory_order_seq_cst);
}
// x86: lock xadd DWORD PTR g_counter[rip], eax
// ARM: ldaddal w0, w1, [x19]   (ARMv8.1 LSE 원자 명령)
//   또는 LDXR/STXR 루프 (구형 ARM)

int experiment_cas_loop() {
    int old = g_counter.load(std::memory_order_relaxed);
    while (!g_counter.compare_exchange_weak(old, old + 1,
        std::memory_order_seq_cst, std::memory_order_relaxed));
    return old;
}
// x86:
//   .loop:
//     mov eax, g_counter[rip]
//     lea ecx, [rax+1]
//     lock cmpxchg g_counter[rip], ecx
//     jne .loop

// ── 실험 B: CMPXCHG vs XCHG ──
int experiment_xchg() {
    return g_counter.exchange(42, std::memory_order_seq_cst);
}
// x86: xchg eax, DWORD PTR g_counter[rip]
//   (XCHG는 항상 암묵적 LOCK → 배리어 효과 내장)

// ── 실험 C: 128비트 CAS (포인터 + 태그) ──
struct alignas(16) TaggedPtr {
    int* ptr;
    uint64_t tag;
};
std::atomic<TaggedPtr> g_tagged;

bool experiment_cas128(TaggedPtr expected, TaggedPtr desired) {
    return g_tagged.compare_exchange_strong(expected, desired,
        std::memory_order_seq_cst);
}
// x86: lock cmpxchg16b XMMWORD PTR g_tagged[rip]
// (CMPXCHG16B: RDX:RAX = expected, RCX:RBX = desired)
```

### 실험 2: CAS 경합 비용 측정

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <chrono>
#include <cstdio>

const int N_THREADS = 8;
const long ITERS    = 10'000'000L;

std::atomic<long> shared_counter{0};

// 버전 A: 모든 스레드가 하나의 카운터에 CAS
void bench_contended() {
    auto work = [&]() {
        for (long i = 0; i < ITERS; i++)
            shared_counter.fetch_add(1, std::memory_order_relaxed);
    };
    std::vector<std::thread> threads;
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N_THREADS; i++) threads.emplace_back(work);
    for (auto& t : threads) t.join();
    auto t1 = std::chrono::high_resolution_clock::now();
    printf("경합 있음 (LOCK XADD): %ld ms, 총 %ld\n",
        std::chrono::duration_cast<std::chrono::milliseconds>(t1-t0).count(),
        shared_counter.load());
}

// 버전 B: 스레드별 로컬 카운터 + 최종 합산
alignas(64) std::atomic<long> per_thread[N_THREADS];

void bench_local_then_merge() {
    auto work = [&](int id) {
        for (long i = 0; i < ITERS; i++)
            per_thread[id].fetch_add(1, std::memory_order_relaxed);
    };
    std::vector<std::thread> threads;
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N_THREADS; i++) threads.emplace_back(work, i);
    for (auto& t : threads) t.join();
    long total = 0;
    for (int i = 0; i < N_THREADS; i++) total += per_thread[i].load();
    auto t1 = std::chrono::high_resolution_clock::now();
    printf("로컬 합산 (경합 없음): %ld ms, 총 %ld\n",
        std::chrono::duration_cast<std::chrono::milliseconds>(t1-t0).count(),
        total);
}

int main() {
    bench_contended();
    for (int i = 0; i < N_THREADS; i++) per_thread[i] = 0;
    bench_local_then_merge();
    return 0;
}
// g++ -O2 -std=c++17 -pthread -o cas_bench cas_bench.cpp
// ./cas_bench

// 예상 출력 (8코어, Intel Core i7):
//   경합 있음:    ~3,200 ms  (CAS 경합 + 캐시라인 핑퐁)
//   로컬 합산:      ~120 ms  (~27배 빠름)
```

### 실험 3: ABA 문제 재현

```cpp
#include <atomic>
#include <thread>
#include <cstdio>
#include <memory>

// 단순 락프리 스택 (ABA 취약 버전)
struct Node {
    int val;
    Node* next;
};

std::atomic<Node*> stack_top{nullptr};

void push(int v) {
    Node* n = new Node{v, nullptr};
    Node* old = stack_top.load();
    do { n->next = old; }
    while (!stack_top.compare_exchange_weak(old, n));
}

Node* pop_node() {
    Node* old = stack_top.load();
    while (old) {
        if (stack_top.compare_exchange_weak(old, old->next))
            return old;
    }
    return nullptr;
}

// ABA 재현 시나리오
void aba_scenario() {
    push(3); push(2); push(1);  // 스택: 1→2→3
    // top = &Node{1}

    Node* a = stack_top.load();  // Thread 1: top을 읽음 (A = &Node{1})

    // Thread 2가 끼어듦:
    Node* popped1 = pop_node();  // 1 제거: 2→3
    Node* popped2 = pop_node();  // 2 제거: 3만 남음
    // Node{1}을 재사용해서 push
    popped1->next = nullptr;
    Node* old_t = stack_top.load();
    do { popped1->next = old_t; }
    while (!stack_top.compare_exchange_weak(old_t, popped1));
    // 스택: 1(재사용)→3  ← popped1의 주소가 동일!

    // Thread 1이 재개: CAS(top, A, A->next)
    Node* desired = a->next;  // a는 Node{1}, next는... 원래 Node{2}
    // CAS 성공 조건: top == a (주소 같음) → 성공!
    if (stack_top.compare_exchange_strong(a, desired)) {
        printf("ABA 발생! top이 old->next(원래 2였던 포인터)로 설정됨\n");
        printf("실제 스택 상태가 깨짐 (3이 사라짐)\n");
    }
}
// 이 실험은 ABA가 개념적으로 어떻게 발생하는지 보여줌
// 실제 멀티스레드 환경에서는 타이밍이 필요하므로 재현이 어려울 수 있음
```

---

## 📊 성능 비교

```
원자적 연산 비용 (Intel Core i7-12700, x86-64, 단일 스레드):

연산                        비용(ns/op)    비고
────────────────────────────────────────────────────────
일반 변수 ++ (레지스터)          0.25 ns    비교 기준
일반 변수 ++ (메모리)            0.3 ns     L1 캐시 적중
fetch_add (relaxed)             0.5 ns     LOCK XADD (비경합)
fetch_add (seq_cst)             18 ns      LOCK XADD + MFENCE
compareAndSet (CAS, 성공)       18 ns      LOCK CMPXCHG
compareAndSet (CAS, 실패 재시도) 35 ns     재시도 1회 추가
XCHG (exchange)                 18 ns      암묵적 LOCK

멀티스레드 경합 상황 (8스레드, 공유 카운터):
  일반 ++ (경쟁 조건 있음):  ~120 ms   ← 결과 틀림!
  LOCK XADD (경합 있음):   ~3,200 ms  ← CAS 폭풍 + 핑퐁
  스레드 로컬 + 합산:         ~120 ms  ← LongAdder 방식

Java AtomicLong vs LongAdder (8스레드, 10M 연산):
  AtomicLong.incrementAndGet():  ~4,800 ms (LOCK XADD + 핑퐁)
  LongAdder.increment():           ~350 ms (13.7배 빠름, 분산 Cell)
  → 고경합에서 LongAdder가 압도적

CAS 성공률 vs 스레드 수 (공유 변수 1개):
  스레드 수   CAS 성공률   평균 재시도
  1           100%         0
  2           ~75%         0.33
  4           ~55%         0.82
  8           ~35%         1.86
  16          ~20%         4.00
  → 스레드 수 증가 시 CAS 성공률 급감, 재시도 폭증
```

---

## ⚖️ 트레이드오프

```
CAS 기반 락프리 vs 뮤텍스:

CAS 락프리:
  ✅ OS 개입 없음 → 컨텍스트 스위치 비용 없음
  ✅ 경합 없을 때 뮤텍스보다 훨씬 빠름 (LOCK 한 번)
  ✅ 우선순위 역전 없음, 데드락 없음 (Progress 보장 약함)
  ❌ 고경합에서 CAS 폭풍 → 성능 저하
  ❌ ABA 문제 → tagged pointer 또는 GC 필요
  ❌ 코드 복잡도 높음, 디버깅 어려움
  ❌ Livelock 가능 (스레드들이 계속 실패)

뮤텍스 (pthread_mutex, synchronized):
  ✅ 고경합에서도 예측 가능한 성능 (대기 스레드가 잠든다)
  ✅ 코드 단순 → 유지보수 쉬움
  ✅ ABA, Livelock 없음
  ❌ OS 컨텍스트 스위치 → ~1~10µs 오버헤드
  ❌ 경합 없어도 최소 LOCK CMPXCHG + OS 호출 비용

스핀락 (busy-wait):
  ✅ OS 개입 없음
  ✅ 잠금 시간이 매우 짧을 때 효율적
  ❌ CPU 낭비: 대기 중 코어가 spin → 에너지·처리량 손실
  ❌ Hyper-Threading에서 파트너 스레드를 굶길 수 있음
  ❌ 우선순위 역전 가능

적용 가이드:
  경합 낮음 + 임계 구역 짧음:    CAS/락프리 → 뮤텍스보다 빠름
  경합 높음 + 스레드 많음:       LongAdder 방식 (분산) 또는 뮤텍스
  읽기 많음 + 쓰기 드묾:        읽기-쓰기 락 (RWLock)
  원자적 카운터/플래그:          AtomicLong/AtomicReference
  복잡한 자료구조 보호:          Mutex 또는 검증된 Concurrent 컬렉션

Java 구체적 가이드:
  카운터 (읽기 < 쓰기, 고경합):  LongAdder
  카운터 (읽기 > 쓰기, 저경합):  AtomicLong
  단일 참조 교환:                AtomicReference.compareAndSet
  ABA 방지 필요:                 AtomicStampedReference
  컬렉션:                        ConcurrentHashMap (내부에서 CAS 활용)
```

---

## 📌 핵심 정리

```
원자적 연산과 CAS 핵심:

x86 LOCK 프리픽스의 실제 동작:
  버스 전체 잠금 X → 해당 캐시라인의 M 상태 독점 획득
  MESI에서 타 코어의 같은 라인을 I 상태로 무효화
  원자적 읽기-수정-쓰기 후 MESI 일반 상태 복귀
  → 비용: 경합 없으면 ~18ns, 경합 있으면 캐시라인 핑퐁 추가

lock cmpxchg 해부:
  EAX = expected, [mem] = 현재값, new = desired
  if [mem] == EAX: [mem] = new, ZF=1 (성공)
  else: EAX = [mem], ZF=0 (실패 → 재시도)
  → C++ compare_exchange_strong/weak, Java compareAndSet 모두 이것

ARM의 LL/SC:
  LDXR (Load Exclusive): 캐시라인 예약
  STXR (Store Exclusive): 예약 유효 시 성공, 아니면 실패
  → compare_exchange_weak → LDXR/STXR 루프로 번역
  → ARMv8.1+ LSE: LDADD, CAS 명령으로 한 줄 처리

ABA 문제:
  A→B→A 변경 후 CAS는 A로 같아보여 성공 → 잘못된 결과
  해결: AtomicStampedReference (Java), 태그된 포인터 (C++), GC

CAS 성능 한계:
  경합 있을 때 O(N) 재시도 → LongAdder처럼 분산 설계가 해법

java-concurrency 연결:
  AtomicInteger.compareAndSet → lock cmpxchg
  LongAdder (Striped64) → 분산 CAS + @Contended
  ConcurrentHashMap → 버킷 단위 CAS
  AQS (ReentrantLock 등) → state 변경에 CAS 사용
```

---

## 🤔 생각해볼 문제

**Q1.** Java `AtomicReference.compareAndSet()`은 내부적으로 x86에서 `lock cmpxchg`로 번역된다. 그런데 `AtomicStampedReference.compareAndSet()`은 어떻게 구현되어 있으며, 왜 `AtomicReference`보다 느린가?

<details>
<summary>해설 보기</summary>

`AtomicReference`는 단일 참조(포인터)를 `lock cmpxchg`로 원자적으로 교환합니다. 참조 하나가 기계어 워드 크기(64비트)이므로 하드웨어가 직접 지원합니다.

`AtomicStampedReference`는 참조와 stamp(int) 두 값을 동시에 교환해야 합니다. 내부 구현은 다음과 같습니다:

```java
// AtomicStampedReference 내부 (대략):
private static class Pair<T> {
    final T reference;
    final int stamp;
    static <T> Pair<T> of(T ref, int stamp) { ... }
}
private volatile Pair<V> pair;  // volatile Pair 참조
```

`compareAndSet()`은 `Pair` 객체를 새로 생성하고, `pair` 필드에 대해 `AtomicReferenceFieldUpdater`(내부적으로 `UNSAFE.compareAndSwapObject`)를 사용해 포인터 교환을 합니다.

**느린 이유:**
1. 매 CAS 시도마다 새 `Pair` 객체 할당 → GC 압력 증가
2. 참조 비교만으로는 (reference, stamp) 쌍을 한 번에 비교 불가 → 내부 Pair 객체의 동일성 비교
3. x86의 `lock cmpxchg16b`(128비트)를 직접 활용하지 않음 — JVM이 플랫폼 독립적으로 구현

따라서 `AtomicStampedReference`는 ABA 문제가 실제로 발생하는 상황에서만 사용하고, 단순 참조 교환에는 `AtomicReference`를 쓰는 것이 적절합니다.

</details>

---

**Q2.** x86에서 `XCHG [mem], reg`는 `LOCK` 프리픽스 없이도 원자적 동작을 하는가? 그리고 `LOCK ADD [mem], imm`과 `XCHG`의 메모리 배리어 측면 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

**`XCHG [mem], reg`는 항상 암묵적 LOCK:**

Intel 매뉴얼에 따르면 `XCHG` 명령이 메모리 피연산자를 가질 때 `LOCK` 프리픽스를 명시하지 않아도 자동으로 LOCK semantics가 적용됩니다. 이는 역사적으로 버스 잠금 프로토콜의 일부로 설계된 것입니다. 따라서 `XCHG`는 항상 원자적이며 StoreLoad 배리어 효과를 가집니다.

**배리어 측면 차이:**

- `LOCK ADD [mem], 0` (또는 `LOCK ADD [rsp], 0`): 피연산자에 0을 더하는 것은 값을 바꾸지 않지만 `LOCK` 프리픽스 덕분에 MFENCE와 동등한 배리어 효과를 얻습니다. HotSpot JIT가 `volatile` 쓰기 후에 이 기법을 사용합니다(`LOCK ADD [RSP+0], 0`).

- `XCHG [mem], reg`: 실제 값을 교환하면서 암묵적 LOCK + 배리어. 값을 바꾸지 않으려면 레지스터에 현재 값을 먼저 로드해야 합니다.

**성능 측면:**

둘 다 StoreLoad 배리어를 삽입하므로 비용은 유사합니다 (~18ns). `LOCK ADD [RSP], 0`이 더 가볍다는 구현도 있고, `XCHG`가 더 명시적이라는 구현도 있습니다. HotSpot은 예전에 `MFENCE`를 직접 사용했지만 `LOCK ADD`가 일부 마이크로아키텍처에서 더 효율적임이 확인돼 전환했습니다.

</details>

---

**Q3.** 락프리 스택(Treiber Stack)에서 Java GC 환경이 ABA 문제를 완화하는 이유는 무엇이며, 완전히 해결하는가?

<details>
<summary>해설 보기</summary>

**Java GC가 ABA를 완화하는 이유:**

C/C++에서 ABA가 발생하는 핵심 원인은 `delete`된 노드의 메모리가 `malloc`으로 같은 주소에 재할당될 수 있다는 점입니다. 동일한 포인터 주소가 전혀 다른 노드를 가리킬 수 있습니다.

Java GC 환경에서는:
1. 참조가 닿지 않는 객체만 GC가 수집합니다. Thread 1이 오래된 참조 `a`를 보유하는 한, 그 객체는 수집되지 않습니다.
2. GC가 객체를 수집한 뒤 같은 주소에 다른 객체를 배치하더라도, 자바 코드 수준에서는 참조(reference)가 달라 `compareAndSet`이 실패합니다 (JVM이 이동 GC를 수행해도 참조 업데이트가 자동으로 일어나기 때문).
3. 따라서 같은 참조(주소)가 재사용되려면 GC가 회수하고 같은 위치에 재배치해야 하는데, Java에서는 이것이 일반적으로 발생하지 않습니다.

**완전히 해결하지는 않는다:**

이론적으로는 다음 경우에 ABA가 여전히 가능합니다:
- 객체 풀(object pool)에서 같은 객체를 재사용하는 경우: 같은 참조가 반환될 수 있음
- `Unsafe` API를 통해 원시 포인터를 조작하는 경우

따라서 완전한 보장이 필요하면 `AtomicStampedReference`를 사용하고, 일반적인 자료구조 구현에서는 GC 덕분에 실용적으로는 ABA가 거의 발생하지 않습니다. `java.util.concurrent.ConcurrentLinkedQueue`와 같은 JDK 구현이 GC의 이 특성을 전제로 설계된 이유입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 메모리 배리어](./04-memory-barriers.md)** | **[홈으로 🏠](../README.md)** | **[다음: JMM과 하드웨어 연결 ➡️](./06-jmm-hardware-bridge.md)**

</div>
