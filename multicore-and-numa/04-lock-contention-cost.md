# 락 경합의 하드웨어 비용 — 캐시라인 핑퐁

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 뮤텍스 변수 자체가 왜 캐시라인 핑퐁의 진원지가 되는가?
- 스핀락과 뮤텍스는 하드웨어 관점에서 어떻게 다르고, 각각 어떤 상황에서 유리한가?
- MCS 락과 Ticket 락이 일반 스핀락의 핑퐁을 어떻게 줄이는가?
- `perf c2c`로 어느 캐시라인이 경합 진원지인지 어떻게 찾는가?
- False Sharing(Ch3-02)과 락 경합이 만드는 핑퐁의 차이는 무엇인가?
- 락 경합을 줄이는 설계 원칙 — 세분화, 락프리, 분산 카운터 — 의 트레이드오프는?

---

## 🔍 왜 이 개념이 중요한가

### "synchronized 한 줄이 전체 서버를 직렬화한다"

```
실제 사례:
  Java 웹 서버, 동시 요청 처리 중
  모든 요청이 공통 캐시 레이어를 통과:

  class Cache {
      private final Map<String, Object> map = new HashMap<>();

      public synchronized Object get(String key) {  // ← 전역 락!
          return map.get(key);
      }
      public synchronized void put(String key, Object v) {
          map.put(key, v);
      }
  }

  측정 결과:
    스레드 1개:    100K req/sec
    스레드 4개:    180K req/sec (기대: 400K)
    스레드 16개:   220K req/sec (기대: 1.6M)
    스레드 32개:   200K req/sec (더 느려짐!)

  원인 분석:
    synchronized가 Object 헤더의 monitor lock 획득
    → monitor 객체의 캐시라인이 모든 코어 간 핑퐁
    → 16코어가 같은 캐시라인을 번갈아 M 상태로 올리고 내림
    → 락 대기가 총 실행 시간의 70% 이상

  lock.object의 캐시라인이 초당 수백만 번 이동:
    Core0: 락 변수 M 상태 (보유 중)
    Core1~15: 락 변수 I 상태 → BusRdX 발행 → 대기
    Core0 해제: 락 변수 → Core1으로 이동 (DRAM 경유)
    Core1 획득: M 상태 → Core2~15 대기
    → N코어가 있으면 순서 결정에만 수천 사이클 낭비
```

---

## 😱 잘못된 이해

### Before: 락은 논리적 보호 장치일 뿐 하드웨어 비용은 없다

```
잘못된 가정:
  "synchronized / mutex는 논리적으로 동시 접근을 막는 것"
  "락을 잡지 않은 코드는 하드웨어 비용이 없다"
  "경합이 없으면 락은 공짜다"

실제로 놓치는 것:

  1. 락 변수 자체의 캐시라인 이동 비용
     // mutex_t lock 변수 = 4~8 바이트
     // 이 변수가 속한 64바이트 캐시라인이 핑퐁의 본체
     // 락을 잡는 코어가 바뀔 때마다 캐시라인이 DRAM 거쳐 이동

  2. 스핀락의 busy-wait 비용
     while (!try_lock()) { /* 아무것도 안 함 */ }
     → "아무것도 안 함"처럼 보이지만 실제로는:
       · 계속 lock 변수를 L1 캐시에서 로드
       · lock 변수가 다른 코어에서 M 상태이면 무효화 대기
       · CPU 코어 전력 소모
       · L1/L2 캐시라인이 lock 변수로 가득 차 실제 작업 데이터 밀어냄

  3. 경합이 없는 락도 비용 있음
     // 단일 스레드에서 mutex lock/unlock:
     pthread_mutex_lock(&m);   // CAS + 메모리 배리어 = ~10~30 사이클
     pthread_mutex_unlock(&m); // store + 메모리 배리어 = ~10~30 사이클
     // 아무도 경합 안 해도 배리어 비용은 항상 발생
```

---

## ✨ 올바른 이해

### After: 락은 캐시라인 이동을 유발하는 하드웨어 이벤트다

```
핵심 그림:

  주소 0x100: pthread_mutex_t lock;  // 64바이트 캐시라인 안에 위치

  Core0이 락 획득:
    lock 변수의 캐시라인 → MESI M 상태 (Core0 L1 전용)
    CAS(lock, 0, 1) 성공

  Core1이 락 획득 시도:
    lock 변수의 캐시라인이 Core0 L1에서 M 상태
    → Core1의 BusRdX → Core0의 라인 M→I (Writeback)
    → Core1이 라인 M 상태 획득 → CAS 성공 or 실패
    
    실패(락이 여전히 1): 다시 BusRdX 반복
    성공(락이 0): 1로 바꾸고 진입

  N개 코어가 경합 시:
    모든 코어가 BusRdX를 발행 → 버스 포화
    초당 수백만 번의 캐시라인 Writeback + 이동
    → 락 획득 자체에 수천 사이클 소요

락 경합 하드웨어 비용 모델:
  경합 비용 ≈ (대기 코어 수) × (캐시라인 이동 지연)
           ≈ (N-1) × 40~200 사이클
  N=16:  15 × 150 사이클 = 2,250 사이클 ← 락 획득에!
  (임계 구역이 100 사이클이면 락 오버헤드가 22배)
```

---

## 🔬 내부 동작 원리

### 1. 스핀락 vs 뮤텍스 — 하드웨어 관점

```c
// ========== 스핀락 (busy-wait) ==========
typedef struct { volatile int locked; } spinlock_t;

void spin_lock(spinlock_t* l) {
    // Test-and-Set 방식:
    while (__sync_lock_test_and_set(&l->locked, 1)) {
        // 락이 잡혀 있는 동안 계속 루프
        // x86에서 PAUSE 명령 삽입 권장:
        //   __asm__ volatile("pause" ::: "memory");
        // PAUSE: 파이프라인 힌트 → 스핀 루프 감지, 전력 절감
    }
}

void spin_unlock(spinlock_t* l) {
    __sync_lock_release(&l->locked);  // store + 배리어
}

// 스핀락의 하드웨어 동작:
// locked=1인 동안 모든 대기 코어가 l->locked 캐시라인 반복 로드
// → 캐시라인이 각 코어의 L1에서 S(Shared) 상태 유지
// → 해제 시 l->locked를 0으로 쓰는 코어가 BusRdX
//    → 다른 코어의 S 상태 모두 I로 무효화
//    → 무효화된 코어들이 다시 BusRdX → "thundering herd"
// → N개 코어 동시 대기 시 해제 후 N-1개의 BusRdX 폭발

// PAUSE를 이용한 개선 (x86 MONITOR/MWAIT 없을 때):
void spin_lock_improved(spinlock_t* l) {
    while (__sync_lock_test_and_set(&l->locked, 1)) {
        while (l->locked) {            // 먼저 읽기만 (S 상태 유지)
            __asm__ volatile("pause"); // 힌트: 스핀 중임을 CPU에 알림
        }
        // locked가 0이 됐을 때만 다시 TAS 시도
    }
}
// 개선 효과: 읽기 루프는 S 상태 → BusRdX 없음
//            0이 됐을 때만 M 상태 요청 → 핑퐁 주기 감소

// ========== pthread 뮤텍스 (blocking) ==========
// 뮤텍스의 하드웨어 동작:
// 락 획득 실패 → futex(FUTEX_WAIT) 시스템 콜 → 커널이 스레드 Sleep
// → 락 해제 시 futex(FUTEX_WAKE) → 커널이 스레드 깨움
// 핵심 비용:
//   syscall 오버헤드: ~1~5 μs (컨텍스트 스위치 없어도)
//   스레드 Sleep + Wake: ~10~50 μs
// → 락 보유 시간이 길면 뮤텍스가 유리 (CPU 낭비 없음)
// → 락 보유 시간이 짧으면 스핀락이 유리 (syscall 비용 없음)
```

### 2. MCS 락 — 핑퐁을 원천 차단하는 설계

```c
// MCS 락 (Mellor-Crummey and Scott, 1991)
// 핵심 아이디어: 각 스레드가 자신만의 노드를 스핀
// → 공유 캐시라인이 아닌 스레드 전용 캐시라인에서 대기
// → 핑퐁 원천 제거

typedef struct mcs_node {
    volatile struct mcs_node* next;
    volatile int              locked;  // ← 스레드 전용 캐시라인!
} mcs_node_t;

typedef volatile mcs_node_t* mcs_lock_t;

void mcs_lock(mcs_lock_t* lock, mcs_node_t* node) {
    node->next   = NULL;
    node->locked = 1;

    // 자신을 큐의 끝에 원자적으로 추가
    mcs_node_t* prev = __sync_lock_test_and_set(lock, node);

    if (prev != NULL) {
        // 이전 노드가 있음 → 대기 큐에 추가
        prev->next = node;
        // ▶ 핵심: node->locked가 0이 될 때까지 대기
        //         이 캐시라인은 오직 이 스레드와
        //         이전 락 보유자만 접근!
        while (node->locked) {
            __asm__ volatile("pause");
        }
    }
    // 락 획득 완료
}

void mcs_unlock(mcs_lock_t* lock, mcs_node_t* node) {
    if (node->next == NULL) {
        // 큐에 대기자 없음 → 락 해제
        if (__sync_bool_compare_and_swap(lock, node, NULL))
            return;
        // CAS 실패 → 방금 누군가 큐에 추가됨 → 기다림
        while (node->next == NULL)
            __asm__ volatile("pause");
    }
    // 다음 대기자의 locked를 0으로 → 그 스레드의 캐시라인만 무효화
    node->next->locked = 0;
}

// MCS의 핵심 특성:
// ① 각 스레드가 자신만의 mcs_node_t.locked를 스핀
//    → 서로 다른 캐시라인 → 핑퐁 없음
// ② FIFO 순서 보장 (기아 없음)
// ③ 해제 시 단 하나의 캐시라인(다음 대기자의 node.locked)만 무효화
//    → 일반 스핀락 해제 시 N-1개 BusRdX 대신 딱 1개
```

### 3. Ticket 락 — FIFO 공정성 보장

```c
// Ticket 락: 은행 대기번호 시스템
// 핵심 아이디어: 순서를 미리 부여 → 자기 번호 때만 진입
// 단점: 여전히 공유 변수 스핀 (MCS보다 더 많은 핑퐁)
// 장점: 구현 단순, FIFO 보장

typedef struct {
    volatile uint32_t next_ticket;   // 다음 발급 번호
    volatile uint32_t now_serving;   // 현재 서비스 번호
} ticket_lock_t;

void ticket_lock(ticket_lock_t* l) {
    // 자신의 번호 원자적으로 발급
    uint32_t my_ticket = __atomic_fetch_add(
        &l->next_ticket, 1, __ATOMIC_SEQ_CST);

    // 자기 번호가 올 때까지 대기
    while (l->now_serving != my_ticket)
        __asm__ volatile("pause");
}

void ticket_unlock(ticket_lock_t* l) {
    // 다음 번호로 넘김 → 해당 스레드가 스핀 종료
    __atomic_fetch_add(&l->now_serving, 1, __ATOMIC_SEQ_CST);
}

// Ticket 락의 핑퐁 문제:
// now_serving이 변경될 때마다 모든 대기 스레드가 캐시 무효화
// → N개 대기 시 N번의 캐시라인 무효화 전파
// MCS와 달리 공유 캐시라인(now_serving) 하나를 모두 스핀

// 비교:
// 일반 스핀락: 무질서한 핑퐁 + 불공정 (기아 가능)
// Ticket 락:  공정(FIFO) + 여전히 공유 변수 핑퐁
// MCS 락:     공정(FIFO) + 전용 캐시라인 → 핑퐁 최소화
```

### 4. perf c2c로 락 경합 위치 추적

```bash
# perf c2c: Cache-to-Cache 전송 추적
# 어느 캐시라인이 코어 간 핑퐁을 일으키는지 정확히 보여줌
# Linux 4.10+, Intel PT 지원 CPU 필요

# 1단계: 레코드 수집
perf c2c record -g -- ./lock_contention_bench

# 2단계: 보고서 출력
perf c2c report --stdio

# 예상 출력:
#  =================================================
#  Shared Cache Line Event Information
#  =================================================
#
#  Total records                     :     15243400
#  Locked Load/Store Operations      :      1233450
#  Load Operations                   :      8234567
#  Loads - src is HiT                :         1234  (로컬 HIT)
#  Loads - src is HITm               :      1122334  (← 핑퐁 핵심 지표!)
#
#  HITM = Hit In Modified State
#  다른 코어의 M 상태 라인에서 데이터 가져옴 = 핑퐁

# 3단계: 경합 라인 상세 보기
perf c2c report --stdio -N 10  # 상위 10개 라인

# 출력 예시:
# ─────────────────────────────────────────────────────────────────
# Index   0
# Cacheline: 0x7ffff7a1c3c0   ← 이 주소가 핑퐁 진원지
# Total HITMs: 1122334
# ─────────────────────────────────────────────────────────────────
# Rmt/Lcl Peer Node           RmtHitm  LclHitm
#    0  /    0  :    0          123456   1000000  ← 로컬 코어 간 핑퐁
# ─────────────────────────────────────────────────────────────────
# Shared Data Type: Contended
#
# Stores Pareto: (이 라인에 쓰는 코드 위치)
# 3.45%  lock_contention_bench  lock_fn         lock.c:42
# 3.45%  lock_contention_bench  lock_fn         lock.c:42
# ...

# 4단계: 주소에서 변수 역추적
addr2line -e ./lock_contention_bench -a 0x7ffff7a1c3c0
# → 어떤 변수가 이 주소에 있는지 확인
objdump -d ./lock_contention_bench | grep -A5 "1c3c0"
```

---

## 💻 실전 실험

### 실험 1: 락 경합 벤치마크 — 경합 강도 변화

```c
// lock_contention_bench.c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <time.h>

#define NUM_THREADS    8
#define TOTAL_OPS      10000000L

// 단일 전역 락 (경합 최대)
pthread_mutex_t global_mutex = PTHREAD_MUTEX_INITIALIZER;
volatile long   global_counter = 0;

// 세분화된 락 (경합 감소)
#define NUM_SHARDS     16
typedef struct {
    pthread_mutex_t mu;
    volatile long   counter;
    char            _pad[64 - sizeof(pthread_mutex_t) - sizeof(long)];
} __attribute__((aligned(64))) shard_t;
shard_t shards[NUM_SHARDS];

// 1. 단일 전역 락
void* global_lock_worker(void* arg) {
    long ops = TOTAL_OPS / NUM_THREADS;
    for (long i = 0; i < ops; i++) {
        pthread_mutex_lock(&global_mutex);
        global_counter++;
        pthread_mutex_unlock(&global_mutex);
    }
    return NULL;
}

// 2. 세분화된 락 (Sharding)
void* sharded_lock_worker(void* arg) {
    int id = *(int*)arg;
    int shard = id % NUM_SHARDS;
    long ops = TOTAL_OPS / NUM_THREADS;
    for (long i = 0; i < ops; i++) {
        pthread_mutex_lock(&shards[shard].mu);
        shards[shard].counter++;
        pthread_mutex_unlock(&shards[shard].mu);
    }
    return NULL;
}

// 3. 원자적 연산 (락 없음)
volatile long atomic_counter = 0;
void* atomic_worker(void* arg) {
    long ops = TOTAL_OPS / NUM_THREADS;
    for (long i = 0; i < ops; i++)
        __atomic_fetch_add(&atomic_counter, 1, __ATOMIC_SEQ_CST);
    return NULL;
}

// 4. 스레드 로컬 + 최종 합산
__thread long local_count = 0;
volatile long final_sum = 0;
void* local_worker(void* arg) {
    long ops = TOTAL_OPS / NUM_THREADS;
    for (long i = 0; i < ops; i++)
        local_count++;
    // 최종 합산 시에만 원자적 연산
    __atomic_fetch_add(&final_sum, local_count, __ATOMIC_SEQ_CST);
    return NULL;
}

double run(void*(*fn)(void*)) {
    pthread_t threads[NUM_THREADS];
    int ids[NUM_THREADS];
    struct timespec s, e;

    clock_gettime(CLOCK_MONOTONIC, &s);
    for (int i = 0; i < NUM_THREADS; i++) {
        ids[i] = i;
        pthread_create(&threads[i], NULL, fn, &ids[i]);
    }
    for (int i = 0; i < NUM_THREADS; i++)
        pthread_join(threads[i], NULL);
    clock_gettime(CLOCK_MONOTONIC, &e);

    return (e.tv_sec - s.tv_sec) * 1e3 +
           (e.tv_nsec - s.tv_nsec) / 1e6;
}

int main() {
    for (int i = 0; i < NUM_SHARDS; i++)
        pthread_mutex_init(&shards[i].mu, NULL);

    printf("방법                     시간(ms)   상대 처리량\n");
    printf("─────────────────────────────────────────────\n");

    double t_global = run(global_lock_worker);
    double t_shard  = run(sharded_lock_worker);
    double t_atomic = run(atomic_worker);
    double t_local  = run(local_worker);

    printf("전역 뮤텍스 (단일락):   %8.1f ms   1.0x\n", t_global);
    printf("세분화 뮤텍스 (16샤드): %8.1f ms   %.1fx\n",
           t_shard, t_global / t_shard);
    printf("원자적 연산 (CAS):      %8.1f ms   %.1fx\n",
           t_atomic, t_global / t_atomic);
    printf("스레드 로컬 (최적):     %8.1f ms   %.1fx\n",
           t_local, t_global / t_local);

    return 0;
}
// gcc -O2 -pthread -o lock_contention lock_contention_bench.c
```

### 실험 2: perf c2c로 경합 라인 위치 추적

```bash
# 위 벤치마크로 경합 진원지 찾기

# 전역 락 버전 실행 중 perf c2c 레코드
perf c2c record -g -- ./lock_contention global

# 보고서
perf c2c report --stdio 2>&1 | head -60

# HITM 비율이 높은 캐시라인 상위 5개 확인
perf c2c report --stdio -N 5

# 경합 라인의 코드 위치 확인
perf c2c report --stdio --call-graph dwarf 2>&1 | grep -A 20 "Cacheline:"

# 세분화 락 버전과 비교
perf c2c record -g -- ./lock_contention sharded
perf c2c report --stdio 2>&1 | grep "Total HITMs"
# → HITM 수가 전역 락의 1/16 수준이면 이상적

# perf stat으로 전반적 캐시 미스 비교
perf stat -e cycles,instructions,cache-misses,LLC-load-misses \
    ./lock_contention global 2>&1

perf stat -e cycles,instructions,cache-misses,LLC-load-misses \
    ./lock_contention sharded 2>&1
```

### 실험 3: MCS 락 vs 스핀락 성능 비교

```c
// mcs_vs_spin_bench.c
// 간단한 MCS 락 구현과 비교
// 앞서 정의한 mcs_lock_t, spinlock_t 코드 포함

#define NUM_THREADS  16
#define OPS         5000000L

// ... (spinlock_t, mcs_lock_t 코드 위에서 복사) ...

spinlock_t slock = {0};
mcs_lock_t mlock = NULL;

__thread mcs_node_t my_node;  // 스레드 전용 노드 (스택에 배치)
volatile long spin_counter = 0;
volatile long mcs_counter  = 0;

void* spin_worker(void* arg) {
    for (long i = 0; i < OPS / NUM_THREADS; i++) {
        spin_lock(&slock);
        spin_counter++;
        spin_unlock(&slock);
    }
    return NULL;
}

void* mcs_worker(void* arg) {
    for (long i = 0; i < OPS / NUM_THREADS; i++) {
        mcs_lock(&mlock, &my_node);
        mcs_counter++;
        mcs_unlock(&mlock, &my_node);
    }
    return NULL;
}

// perf c2c로 두 버전 비교:
// 스핀락: slock 변수 캐시라인에 HITM 집중
// MCS 락: my_node.locked 캐시라인에 HITM → 하지만 스레드별로 분산
```

### 실험 4: Java synchronized vs ReentrantLock vs LongAdder

```java
// LockContention.java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.concurrent.locks.*;

public class LockContention {
    static final int THREADS = 8;
    static final long OPS = 10_000_000L;

    // 1. synchronized (모니터 락)
    static long syncCounter = 0;
    static synchronized void syncIncrement() { syncCounter++; }

    // 2. ReentrantLock
    static long reentrantCounter = 0;
    static final ReentrantLock rl = new ReentrantLock();
    static void rlIncrement() {
        rl.lock();
        try { reentrantCounter++; } finally { rl.unlock(); }
    }

    // 3. AtomicLong (CAS, 락 없음)
    static final AtomicLong atomicCounter = new AtomicLong();

    // 4. LongAdder (Striped64, 분산 카운터)
    static final LongAdder adder = new LongAdder();

    // 5. StampedLock (낙관적 읽기 + 비관적 쓰기)
    static long stampedCounter = 0;
    static final StampedLock sl = new StampedLock();
    static void stampedIncrement() {
        long stamp = sl.writeLock();
        try { stampedCounter++; } finally { sl.unlockWrite(stamp); }
    }

    static long benchmark(Runnable task) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(THREADS);
        long start = System.nanoTime();
        for (int i = 0; i < THREADS; i++) pool.submit(task);
        pool.shutdown();
        pool.awaitTermination(60, TimeUnit.SECONDS);
        return (System.nanoTime() - start) / 1_000_000;
    }

    public static void main(String[] args) throws Exception {
        // 워밍업
        for (int i = 0; i < 3; i++) {
            benchmark(() -> { for (long j=0;j<OPS/10;j++) adder.increment(); });
        }

        long t1 = benchmark(() -> {
            for (long i=0;i<OPS/THREADS;i++) syncIncrement(); });
        long t2 = benchmark(() -> {
            for (long i=0;i<OPS/THREADS;i++) rlIncrement(); });
        long t3 = benchmark(() -> {
            for (long i=0;i<OPS/THREADS;i++) atomicCounter.incrementAndGet(); });
        long t4 = benchmark(() -> {
            for (long i=0;i<OPS/THREADS;i++) adder.increment(); });

        System.out.printf("synchronized:    %,4d ms  (1.0x)%n", t1);
        System.out.printf("ReentrantLock:   %,4d ms  (%.1fx)%n", t2, (double)t1/t2);
        System.out.printf("AtomicLong:      %,4d ms  (%.1fx)%n", t3, (double)t1/t3);
        System.out.printf("LongAdder:       %,4d ms  (%.1fx)%n", t4, (double)t1/t4);
        // async-profiler로 락 경합 시각화:
        // java -agentpath:/path/to/libasyncProfiler.so=start,event=lock,file=lock.html LockContention
    }
}
// 컴파일: javac LockContention.java
// 실행: java -XX:-RestrictContended LockContention
```

---

## 📊 성능 비교

```
락 방식별 처리량 비교 (8스레드, 1천만 회 카운터 증가):

방법                         시간(ms)   처리량 비교   HITM/sec
──────────────────────────────────────────────────────────────────
전역 pthread_mutex             4,200     1.0x         ~8M
16-shard mutex                   650     6.5x         ~0.5M
spinlock (test-and-set)         3,800     1.1x         ~12M  ← 핑퐁 심함!
spinlock (test-test-and-set)   1,200     3.5x         ~4M
MCS 락                           480     8.75x        ~0.1M
Ticket 락                        900     4.7x         ~6M
__atomic_fetch_add             1,800     2.3x         ~15M  ← 직접 경합
스레드 로컬 누산                   80    52.5x         ~0M

Java (8스레드):
synchronized:       4,500 ms  (1.0x)
ReentrantLock:      3,800 ms  (1.2x)
AtomicLong:         2,200 ms  (2.0x)
LongAdder:            380 ms  (11.8x)

락 보유 시간에 따른 최적 선택:
  보유 시간   권장 방식            이유
  ────────────────────────────────────────────
  < 50 ns    스핀락 (개선형)       syscall 피하기
  50~500 ns  적응형 스핀락         일부 스핀 후 Sleep
  > 500 ns   뮤텍스/ReentrantLock  CPU 낭비 방지
  항상       LongAdder/로컬 누산   락 자체 제거
```

---

## ⚖️ 트레이드오프

```
락 설계 트레이드오프:

단일 전역 락:
  ✅ 구현 단순, 정확성 보장 용이
  ❌ 경합 강도 = 스레드 수에 비례
  ❌ 코어 증가에 따라 성능 저하 (Ch6-03 암달)

세분화 (Sharding):
  ✅ 경합을 샤드 수로 나눔 → 선형에 가까운 확장
  ❌ 전체 집계 시 모든 샤드 순회 필요
  ❌ 키 분배 불균형 → 특정 샤드에 경합 집중
  ConcurrentHashMap이 이 전략의 대표 사례 (기본 16 샤드)

스핀락:
  ✅ 락 보유 시간 짧을 때 syscall 비용 없음
  ❌ 실행 유닛 소모 (busy-wait)
  ❌ HT 형제 코어의 자원 소모
  ❌ 우선순위 역전 (낮은 우선순위가 락 보유 중 선점)

MCS 락:
  ✅ 핑퐁 최소화 (전용 캐시라인 스핀)
  ✅ FIFO 공정성
  ❌ 노드 메모리 관리 복잡 (스레드별 mcs_node 필요)
  ❌ 구현 복잡도 높음

락프리 (CAS 기반):
  ✅ 데드락 없음
  ✅ 우선순위 역전 없음
  ❌ ABA 문제 (Ch3-05 참조)
  ❌ 고경합 시 CAS 재시도 폭증 → 오히려 느려질 수 있음
  ❌ 복잡한 데이터 구조에 적용 어려움

락 제거 (로컬 누산):
  ✅ 가장 빠름 (L1 캐시에서만 작업)
  ✅ 완전 선형 확장
  ❌ 최종 집계까지 중간값 불가
  ❌ 집계 주기 설계 필요
  → LongAdder, striped counter가 이 패턴의 구현체

False Sharing(Ch3-02)과의 차이:
  False Sharing: 논리적으로 다른 데이터가 같은 캐시라인에 → 우연적 핑퐁
  락 경합: 락 변수를 공유하는 것 자체가 목적 → 필연적 핑퐁
  → 둘 다 캐시라인 핑퐁이지만 해결책이 다름
     False Sharing: 패딩으로 라인 분리
     락 경합: 락 자체 제거 또는 세분화
```

---

## 📌 핵심 정리

```
락 경합의 하드웨어 비용 핵심:

왜 비싼가:
  락 변수의 캐시라인이 M 상태 소유권을 코어 간에 이동
  N개 코어 경합 시 해제 시 N-1개 BusRdX 폭발 (thundering herd)
  락 획득 자체에 수백~수천 사이클 소모
  (임계 구역이 짧을수록 락 오버헤드 비율이 높아짐)

스핀락 vs 뮤텍스:
  스핀락: 짧은 임계 구역, CPU 소비 but syscall 없음
  뮤텍스: 긴 임계 구역, Sleep/Wake but futex 비용
  MCS:    핑퐁 최소화, 전용 캐시라인 스핀

MCS/Ticket의 핑퐁 감소 원리:
  Ticket: FIFO 보장, 여전히 공유 변수 스핀
  MCS: 스레드별 전용 노드 스핀 → 공유 캐시라인 없음
  → 해제 시 딱 1개의 캐시라인만 무효화

진단 도구:
  perf c2c record/report: HITM 수가 높은 캐시라인 주소와 코드 위치
  perf lock record/report: 락 대기 시간 합산
  async-profiler (Java): wall-clock + lock 이벤트 프로파일링

해결 우선순위:
  1. 락 제거: 스레드 로컬 누산 → LongAdder 방식
  2. 락프리: CAS 기반 (경합 낮을 때)
  3. 락 세분화: Sharding으로 경합 1/N
  4. 락 개선: MCS/Ticket, 적응형 스핀락

Ch3-02 False Sharing과의 연결:
  False Sharing = 의도치 않은 캐시라인 공유
  락 경합 = 의도적 캐시라인 공유(락 변수)
  둘 다 perf c2c의 HITM으로 진단 → 해결책이 다름
```

---

## 🤔 생각해볼 문제

**Q1.** 어떤 코드에서 임계 구역이 평균 30ns이고 8개 스레드가 경합하고 있다. 뮤텍스와 스핀락 중 무엇이 더 적합한가? 그리고 이 임계 구역을 100ns로 늘리면 판단이 어떻게 달라지는가?

<details>
<summary>해설 보기</summary>

**임계 구역 30ns일 때:**
futex(FUTEX_WAIT/WAKE) 시스템 콜 왕복 비용이 약 1~5μs(1,000~5,000ns)입니다. 임계 구역(30ns)보다 락 대기 비용이 수십 배 크므로, 대기 시간이 짧다면 스핀락이 유리합니다.

8스레드 경합 상황에서 한 번에 한 스레드만 진입하므로 평균 대기 = 7 × 30ns = 210ns입니다. 스핀락으로 210ns을 스핀하는 비용은 8코어에서 8 × 210ns = 1,680ns의 CPU-사이클 낭비이지만, 뮤텍스의 futex 비용(1,000~5,000ns × 7회 = 7~35μs)보다 훨씬 작습니다.

**임계 구역 100ns일 때:**
평균 대기 = 7 × 100ns = 700ns. 이 정도면 futex 비용(~1~2μs)과 비슷해집니다. 경합이 많고 대기 시간이 길어질수록 뮤텍스가 유리해집니다. CPU를 Sleep 상태로 두어 다른 유용한 작업에 코어를 쓸 수 있기 때문입니다.

**결론:** 임계 구역이 수십 ns이면 스핀락(개선형), 수백 ns 이상이면 뮤텍스 또는 적응형 스핀락(일정 횟수 스핀 후 Sleep) 사용을 권장합니다. pthread_mutex는 Linux에서 내부적으로 적응형 전략을 사용하기도 합니다.

</details>

---

**Q2.** MCS 락에서 `mcs_node_t`를 힙(heap)에 할당하면 NUMA 원격 접근 문제가 생길 수 있다. 왜 그런가? 이를 피하는 올바른 방법은 무엇인가?

<details>
<summary>해설 보기</summary>

MCS 락의 핵심은 각 스레드가 자신의 `mcs_node_t.locked`를 스핀한다는 점입니다. 이 캐시라인을 해제 스레드가 `node->next->locked = 0`으로 변경할 때 MESI 프로토콜로 다음 대기 스레드에게 전달됩니다.

**힙 할당의 문제:**
`malloc`으로 `mcs_node_t`를 할당하면 first-touch 정책에 따라 할당한 스레드의 NUMA 노드 메모리에 배치됩니다. 그런데 스레드가 마이그레이션되거나 다른 소켓 코어에서 실행되면, `locked` 변수가 다른 NUMA 노드에 있게 됩니다. 해제 스레드가 `node->next->locked = 0`을 쓸 때 NUMA 원격 쓰기가 발생하고, 대기 스레드는 원격 DRAM에서 오는 캐시 무효화를 기다립니다. 이는 MCS의 핵심 장점(로컬 캐시라인 스핀)을 무력화합니다.

**올바른 방법:**
1. **스레드 스택에 배치:** `__thread mcs_node_t my_node;` — 스레드가 실행 중인 코어의 스택에 있으므로 로컬 NUMA 메모리 가능성이 높음.
2. **NUMA 로컬 할당:** `numa_alloc_local(sizeof(mcs_node_t))` — 현재 스레드의 NUMA 노드에 명시적 할당.
3. **코어 고정(Thread Affinity):** Ch6-05에서 다루는 `pthread_setaffinity_np`로 스레드를 특정 코어에 고정하면 마이그레이션 후 NUMA 불일치 방지.

이는 MCS 락이 NUMA 인식 방식으로 확장된 "NUMA-aware MCS lock"의 설계 동기이기도 합니다.

</details>

---

**Q3.** `perf c2c report`에서 어떤 캐시라인의 `RmtHitm`(Remote HITM) 값이 `LclHitm`(Local HITM)보다 10배 높다. 이것이 의미하는 바는 무엇이고, 어떤 조치를 취해야 하는가?

<details>
<summary>해설 보기</summary>

**의미:**
`RmtHitm`이 높다는 것은 이 캐시라인이 다른 소켓(NUMA 원격 노드)의 코어가 M 상태로 보유하고 있는 상태에서 데이터를 가져오는 일이 많다는 뜻입니다. 즉, 이 락 변수(또는 공유 데이터)에 접근하는 스레드들이 서로 다른 소켓에 분산되어 있고, 캐시라인 이동이 QPI/UPI를 거쳐 소켓 간으로 이루어지고 있습니다.

로컬 핑퐁(~40 사이클)과 달리 원격 핑퐁은 ~150~300 사이클이 소요됩니다. 따라서 `LclHitm`만 있는 경우보다 훨씬 비싼 경합입니다.

**조치:**
1. **NUMA 토폴로지 인식 스레드 배치(Ch6-05):** 이 캐시라인에 접근하는 스레드들을 같은 소켓의 코어에 배치합니다. `taskset` 또는 `numactl --cpunodebind`를 사용합니다.
2. **락 데이터 로컬화:** 락 변수 자체를 접근하는 스레드들이 있는 소켓의 메모리에 배치합니다. `numa_alloc_onnode()`를 사용합니다.
3. **소켓별 독립 인스턴스:** 가능하다면 소켓0과 소켓1이 완전히 독립된 락과 데이터 사본을 가지도록 설계를 변경합니다(멀티 인스턴스 샤딩). 최종 집계만 공유합니다.
4. **접근 패턴 검토:** 왜 원격 소켓에서 이 캐시라인에 접근하는지 `perf c2c`의 소스 코드 위치를 확인하고, 스레드 배치 또는 데이터 복제로 원격 접근을 줄입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 스케일링의 벽](./03-scaling-wall.md)** | **[홈으로 🏠](../README.md)** | **[다음: 코어 친화도 ➡️](./05-thread-affinity.md)**

</div>
