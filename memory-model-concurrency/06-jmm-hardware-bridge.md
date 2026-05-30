# JMM과 하드웨어 연결 — happens-before가 하드웨어로 내려가는 길

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Java 메모리 모델(JMM)의 happens-before 규칙은 구체적으로 무엇을 보장하는가?
- happens-before가 x86과 ARM 각각에서 어떤 배리어 명령으로 매핑되는가?
- HotSpot JIT가 `volatile` 읽기/쓰기를 어떤 어셈블리로 컴파일하는가?
- `synchronized` 블록의 진입(`monitorenter`)과 탈출(`monitorexit`) 시 어떤 배리어가 삽입되는가?
- `java.util.concurrent` 패키지의 동기화 도구들이 하드웨어 수준에서 어떻게 동작하는가?
- java-concurrency-deep-dive 레포에서 배운 규칙들이 하드웨어와 어떻게 연결되는가?

---

## 🔍 왜 이 개념이 중요한가

### "JMM은 배리어의 명세서이고, JIT는 번역기다"

```
이 레포의 Chapter 3 전체를 관통하는 흐름:

  MESI 프로토콜 (01)
    ↓ 코어 간 캐시 일관성 보장
  False Sharing (02)
    ↓ 잘못된 레이아웃이 만드는 핑퐁 비용
  메모리 재정렬 (03)
    ↓ Store Buffer + 약한 모델이 순서를 바꿈
  메모리 배리어 (04)
    ↓ mfence/dmb로 순서를 강제
  원자적 연산과 CAS (05)
    ↓ lock cmpxchg가 MESI를 가로챔

  이 모든 것의 최종 목적지: Java 메모리 모델 (06)
    JMM = "Java 코드가 멀티스레드 환경에서 어떤 보장을 받는가"의 명세
    JIT = "그 명세를 현재 플랫폼(x86/ARM)에 맞는 배리어로 번역"

실무 질문:
  "volatile 변수를 읽으면 항상 최신 값을 보는가?"
  "synchronized 블록을 나가면 쓴 내용이 모든 스레드에 보이는가?"
  "AtomicLong.compareAndSet()은 x86과 ARM에서 같은 효과인가?"

  이 질문들의 답이 JMM이고,
  "어떤 명령어로 그 답이 구현되는가"가 이 문서의 내용이다.
```

---

## 😱 잘못된 이해

### Before: JMM은 Java 언어 사양이고 하드웨어와 무관하다

```
잘못된 생각 1:
  "volatile은 캐시를 우회해 메인 메모리에서 직접 읽는다"

  실제:
    volatile은 캐시를 우회하지 않는다!
    MESI 프로토콜 위에서 동작: 다른 코어가 수정했으면 I→S 전환으로 최신값 획득
    volatile의 역할: 배리어를 통해 Store Buffer에 있는 값이 캐시에 반영됨을 보장
    → "캐시를 건너뜀"이 아니라 "배리어를 통해 일관성을 강제"

잘못된 생각 2:
  "synchronized는 언제나 OS 뮤텍스를 사용한다"

  실제:
    HotSpot은 3단계 잠금을 구현:
    1. Biased Locking (편향 잠금): 동일 스레드 재진입 시 CAS조차 불필요
    2. Thin Lock (경량 잠금): 경합 없을 때 CAS (lock cmpxchg) + 스핀
    3. Inflated Lock (팽창 잠금): 경합 심할 때 OS futex → 컨텍스트 스위치
    → 경합 없는 synchronized는 뮤텍스가 아니라 CAS + 배리어

잘못된 생각 3:
  "happens-before는 실행 시간 순서를 보장한다"

  실제:
    happens-before는 "가시성(visibility)" 보장이지 "실행 순서" 보장이 아님
    Thread A의 동작 X가 Thread B의 동작 Y보다 happens-before이면:
    → X가 Y보다 먼저 실행된다는 의미가 아님
    → Y가 실행될 때 X의 부수 효과가 반드시 가시화됨을 의미
    순서는 동기화 도구(잠금, volatile)로 별도 보장 필요
```

---

## ✨ 올바른 이해

### After: JMM = 배리어 명세, JIT = 플랫폼별 번역

```
JMM happens-before 규칙 요약:

  규칙 1: 프로그램 순서 규칙 (Program Order Rule)
    단일 스레드 내에서 모든 동작은 이전 동작보다 happens-before
    → 단일 스레드에서 재정렬이 있어도 결과는 코드 순서와 같음

  규칙 2: 모니터 잠금 규칙 (Monitor Lock Rule)
    synchronized 블록의 unlock(monitorexit)은
    이후 같은 잠금의 lock(monitorenter)보다 happens-before
    → 임계 구역에서의 쓰기가 다음 진입자에게 가시화됨

  규칙 3: volatile 변수 규칙 (Volatile Variable Rule)
    volatile 변수의 쓰기는 이후 같은 변수의 읽기보다 happens-before
    → volatile 쓰기 이전의 모든 동작이 읽기 이후에 가시화됨

  규칙 4: 스레드 시작 규칙 (Thread Start Rule)
    Thread.start() 호출은 시작된 스레드의 모든 동작보다 happens-before
    → start() 이전에 설정한 값들이 새 스레드에서 보임

  규칙 5: 스레드 종료 규칙 (Thread Termination Rule)
    스레드의 모든 동작은 Thread.join() 반환보다 happens-before
    → join() 후에는 종료된 스레드의 모든 쓰기가 보임

  규칙 6: 전이 규칙 (Transitivity)
    A hb B이고 B hb C이면 A hb C

하드웨어 번역 원칙:
  happens-before 규칙 → 최소 비용 배리어 선택
  플랫폼 강도에 따라 배리어 양이 달라짐:
    x86 (TSO): 강한 모델 → 적은 배리어
    ARM (weak): 약한 모델 → 많은 배리어
```

---

## 🔬 내부 동작 원리

### 1. JMM happens-before → x86/ARM 배리어 매핑표

```
JMM 동작                    x86-64 어셈블리          ARM64 어셈블리
────────────────────────────────────────────────────────────────────────
volatile 쓰기              MOV + LOCK ADD [RSP],0   STLR (Store-Release)
                           (또는 XCHG)

volatile 읽기              MOV                      LDAR (Load-Acquire)
                           (추가 배리어 없음)

synchronized 진입          LOCK CMPXCHG (CAS)       LDAXR (Load-Acq Exclusive)
(monitorenter)             → acquire 배리어 내장      → acquire 배리어 내장

synchronized 탈출          LOCK CMPXCHG 또는         STLXR (Store-Rel Exclusive)
(monitorexit)              LOCK OR [RSP], 0          → release 배리어 내장
                           → release 배리어

Thread.start()             mfence 또는              dmb ish
                           LOCK ADD                  (스레드 생성 전 모든 쓰기 가시화)

Thread.join()              mfence 또는              dmb ish
                           LOCK ADD                  (종료 스레드의 쓰기 가시화)

AtomicInteger.             LOCK CMPXCHG             LDAXR / STLXR 루프
compareAndSet()            (= 전체 배리어)            (또는 CAS ARMv8.1+ LSE)

AtomicInteger.             LOCK XADD                LDADDAL (ARMv8.1 LSE)
getAndAdd()                                         또는 LDAXR/STAXR 루프

LockSupport.park()         OS futex (syscall)        OS 수준 대기
LockSupport.unpark()       OS futex + 배리어          OS 수준 + 배리어

─────────────────────────────────────────────────────────────────────────
배리어 강도:
  x86 release store  ≈ 일반 MOV   (TSO가 StoreStore 자동 보장)
  x86 acquire load   ≈ 일반 MOV   (TSO가 LoadLoad 자동 보장)
  ARM release store  = STLR       (~4배 비용)
  ARM acquire load   = LDAR       (~3배 비용)
```

### 2. HotSpot JIT가 `volatile`을 컴파일하는 방식

```java
// 분석 대상 Java 코드:
class Publisher {
    private int value = 0;
    private volatile boolean ready = false;

    void publish(int v) {
        value = v;      // 일반 필드 쓰기
        ready = true;   // volatile 쓰기 → happens-before 포인트
    }

    int consume() {
        if (ready) {    // volatile 읽기 → acquire 포인트
            return value; // value가 반드시 최신값
        }
        return -1;
    }
}
```

```asm
; ===== x86-64 JIT (HotSpot C2, -server 모드) =====

; publish(int v):
;   [rdi+offset_value] = 필드 오프셋, [rdi+offset_ready] = ready 오프셋
;
  mov DWORD PTR [rdi+0x10], esi      ; value = v  (일반 store)
  mov BYTE PTR [rdi+0x14], 0x1       ; ready = true  (volatile store 본체)
  lock add DWORD PTR [rsp+0x0], 0x0  ; StoreLoad 배리어 (MFENCE 동등)
;                                    ; ↑ 이후 load가 최신 값을 보도록 강제
;   ret

; consume():
;   volatile 읽기: x86 TSO에서 LoadLoad 자동 보장 → 단순 MOV
  movzx eax, BYTE PTR [rdi+0x14]     ; ready 읽기 (acquire: 단순 MOV)
  test eax, eax
  je not_ready
  mov eax, DWORD PTR [rdi+0x10]      ; value 읽기 (추가 배리어 없음)
  ret
not_ready:
  mov eax, -1
  ret

; ===== ARM64 JIT (HotSpot C2) =====

; publish(int v):
  str w1, [x0, #offset_value]        ; value = v (일반 STR)
  mov w2, #1
  stlr w2, [x0, #offset_ready]       ; ready = true (STLR = Store-Release)
;                                    ; ↑ 이전 store가 모두 가시화된 후 ready 공개
  ret

; consume():
  ldar w0, [x0, #offset_ready]       ; ready 읽기 (LDAR = Load-Acquire)
;                                    ; ↑ 이후 load가 ready 이후에 실행됨을 보장
  cbz w0, not_ready
  ldr w0, [x0, #offset_value]        ; value 읽기 (이후 일반 LDR 가능)
  ret

; 핵심 차이:
;   x86: volatile 읽기 = MOV (배리어 없음), 쓰기 = MOV + LOCK ADD
;   ARM: volatile 읽기 = LDAR (배리어 내장), 쓰기 = STLR (배리어 내장)
;   → ARM에서 volatile 읽기도 비용이 있음!
```

### 3. `synchronized` 블록의 배리어 메커니즘

```java
// synchronized 블록 분석
class Counter {
    private int count = 0;
    private final Object lock = new Object();

    void increment() {
        synchronized (lock) {  // monitorenter
            count++;
        }  // monitorexit
    }

    int get() {
        synchronized (lock) {  // monitorenter
            return count;      // 최신 값 보장
        }  // monitorexit
    }
}
```

```
synchronized의 3단계 잠금 (HotSpot):

[경합 없음, 첫 진입: Biased Locking]
  monitorenter:
    스레드 ID를 객체 헤더(mark word)에 CAS로 기록
    이미 같은 스레드: 추가 동기화 없음 (재진입 공짜!)
    → 단일 스레드 반복 진입에 최적
  monitorexit:
    배리어 없음 (재진입 해제만)

  x86 어셈블리 (biased 모드):
    mov rax, QWORD PTR [obj+0]    ; mark word 읽기
    cmp rax, [thread_id]          ; 내 스레드인지 확인
    je  biased_ok                 ; 그렇다면 진입 완료

[경합 없음: Thin Lock (Lightweight Locking)]
  monitorenter:
    스택에 displaced mark word 저장
    lock cmpxchg QWORD PTR [obj], rax  ; 헤더에 스택 포인터 CAS
    → acquire 배리어 내장
  monitorexit:
    lock cmpxchg 또는 lock or → release 배리어

  x86 어셈블리 (thin lock 진입):
    lea  rbx, [rsp-0x20]              ; displaced header 위치
    mov  rax, QWORD PTR [obj+0]       ; 현재 mark word
    lock cmpxchg QWORD PTR [obj+0], rbx  ; CAS로 잠금
    jne  contended                    ; 실패 → 경합 처리

  monitorexit x86:
    lock or DWORD PTR [obj], 0x0      ; release 배리어
    (또는 lock cmpxchg로 mark word 복원)

[경합 있음: Inflated Lock]
  monitorenter:
    ObjectMonitor::enter() 호출 → OS futex (syscall)
    → park() → 스레드 대기 (FUTEX_WAIT)
    → 깨어날 때: 배리어 포함
  monitorexit:
    ObjectMonitor::exit() → OS futex (FUTEX_WAKE)
    → 배리어 + 다음 스레드 깨움

synchronized의 배리어 보장:
  진입(monitorenter): acquire 배리어
    → 진입 이후의 읽기가 잠금 획득 이전 값을 보지 않음
    → "이전 소유자의 모든 쓰기가 나에게 가시화됨"
  탈출(monitorexit): release 배리어
    → 탈출 이전의 모든 쓰기가 완료됨을 보장
    → "임계 구역에서 쓴 내용이 다음 진입자에게 가시화됨"
```

### 4. `java.util.concurrent` 도구들의 배리어 매핑

```java
// ─────────────────────────────────────────────
// ReentrantLock (AQS 기반)
// ─────────────────────────────────────────────
ReentrantLock lock = new ReentrantLock();

lock.lock();
// 내부: AQS.acquire() → CAS로 state 변경
//       compareAndSetState(0, 1)
//       → x86: lock cmpxchg (acquire 배리어)
//       → ARM: ldaxr/stxr 루프 (acquire)
//       실패 시: LockSupport.park() → futex

lock.unlock();
// 내부: AQS.release() → state = 0 (release store)
//       → x86: volatile store (lock add [rsp], 0)
//       → ARM: stlr
//       LockSupport.unpark(next) → futex wake

// ─────────────────────────────────────────────
// CountDownLatch
// ─────────────────────────────────────────────
CountDownLatch latch = new CountDownLatch(3);

// countDown():
//   AQS.releaseShared() → CAS(state, state-1)
//   → state=0이 되면: release 배리어 + unpark

// await():
//   state==0이면: acquire 배리어
//   아니면: park (futex wait)

// ─────────────────────────────────────────────
// ConcurrentHashMap.put()
// ─────────────────────────────────────────────
// 빈 버킷에 삽입:
//   casTabAt(tab, i, null, new Node<>(hash, key, value))
//   → Unsafe.compareAndSwapObject → lock cmpxchg (x86)
//   → acquire 배리어 내장

// 꽉 찬 버킷 (체이닝):
//   synchronized (f) { ... }  ← 버킷 헤드 노드를 잠금으로 사용
//   → thin lock CAS → inflated (경합 시)

// ─────────────────────────────────────────────
// LongAdder.increment() (Striped64)
// ─────────────────────────────────────────────
// Cell 업데이트:
//   cell.cas(v, v+1)
//   → VarHandle.compareAndSet → lock cmpxchg
//   → @Contended로 각 Cell이 독립 캐시라인
//   → 경합 없을 때 = lock cmpxchg 한 번 (캐시라인 핑퐁 없음)

// sum():
//   모든 Cell의 value + base 합산
//   → volatile 읽기 × N (각 Cell.value는 volatile)
//   → 순간 스냅샷 아님 (카운터 값이 정확하지 않을 수 있음)
//   → 최종 집계 전용, 중간 값 조회 시 부정확 가능
```

### 5. JITWatch와 PrintAssembly로 어셈블리 확인

```bash
# ── 방법 1: PrintAssembly 직접 출력 ──
# hsdis 라이브러리 필요 (어셈블러 디스어셈블러)

cat > JMMExample.java << 'EOF'
public class JMMExample {
    volatile int vFlag = 0;
    int data = 0;

    public void writer() {
        data = 42;
        vFlag = 1;  // volatile 쓰기
    }

    public int reader() {
        if (vFlag != 0) {  // volatile 읽기
            return data;
        }
        return -1;
    }

    public static void main(String[] args) {
        JMMExample ex = new JMMExample();
        // JIT 워밍업 (C2 컴파일 트리거)
        for (int i = 0; i < 200_000; i++) {
            ex.writer();
            ex.reader();
        }
        System.out.println("Done: " + ex.data);
    }
}
EOF

javac JMMExample.java

# PrintAssembly로 JIT 어셈블리 출력 (hsdis 필요)
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintAssembly \
     -XX:PrintAssemblyOptions=intel \
     -XX:CompileCommand=compileonly,JMMExample.writer \
     -XX:CompileCommand=compileonly,JMMExample.reader \
     -XX:CompileCommand=print,JMMExample.* \
     JMMExample 2>&1 | grep -A 30 "writer\|reader"

# ── 방법 2: LogCompilation + JITWatch ──
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogCompilation \
     -XX:LogFile=jit.log \
     -XX:+PrintAssembly \
     JMMExample

# JITWatch 실행 (GUI):
# https://github.com/AdoptOpenJDK/jitwatch
# mvn install && java -jar jitwatch-ui/target/*.jar
# → jit.log 파일 열기
# → 왼쪽: 소스 코드, 중간: 바이트코드, 오른쪽: 어셈블리

# ── 방법 3: -XX:+PrintCompilation (가볍게 확인) ──
java -XX:+PrintCompilation JMMExample 2>&1 | head -20
# 출력: [컴파일 ID] [레벨] 클래스::메서드 (바이트 수)
# 레벨 4 = C2 JIT (완전 최적화)

# ── 방법 4: async-profiler (배리어 핫스팟 확인) ──
# https://github.com/async-profiler/async-profiler
# ./profiler.sh -e cpu -d 10 -f out.html PID
# → 플레임 그래프에서 lock/mfence 비중 확인

# 예상 출력 (x86-64, HotSpot 21):
# writer():
#   0x... b8 2a 00 00 00    mov eax, 0x2a
#   0x... 89 87 10 00 00 00 mov [rdi+0x10], eax    ; data = 42
#   0x... c7 87 14 00 00 00 01 00 00 00
#                           mov [rdi+0x14], 0x1     ; vFlag = 1 (volatile)
#   0x... f0 83 44 24 00 00 lock add [rsp+0x0], 0x0 ; StoreLoad 배리어!
#
# reader():
#   0x... 8b 87 14 00 00 00 mov eax, [rdi+0x14]    ; vFlag 읽기 (단순 MOV)
#   0x... 85 c0              test eax, eax
#   0x... 74 06              je not_ready
#   0x... 8b 87 10 00 00 00 mov eax, [rdi+0x10]    ; data 읽기
```

---

## 💻 실전 실험

### 실험 1: volatile happens-before 검증

```java
// happens-before 위반 시 발생하는 가시성 문제 재현
public class VisibilityTest {
    static int data = 0;
    static boolean ready = false;  // volatile이 아닌 버전

    // volatile 버전:
    // static volatile boolean readyV = false;

    public static void main(String[] args) throws InterruptedException {
        Thread writer = new Thread(() -> {
            data = 42;
            ready = true;  // volatile 아님 → 가시성 미보장
        });

        Thread reader = new Thread(() -> {
            long spin = 0;
            while (!ready) { spin++; }  // volatile 아님 → 무한 루프 가능!
            System.out.println("data=" + data + " (spin=" + spin + ")");
            // data가 0일 수 있음 (재정렬 + 가시성 미보장)
        });

        reader.start();
        Thread.sleep(100);  // reader가 루프에 들어가게 대기
        writer.start();

        writer.join();
        reader.join();
    }
}
// 실행:
// javac VisibilityTest.java
// java VisibilityTest
//
// 관찰 포인트:
// 1. -server JVM에서 ready가 volatile 아닐 때:
//    reader 스레드가 JIT 최적화로 while(!ready)를 무한루프로 컴파일 가능
//    (JIT가 ready를 레지스터에 캐시해 루프 안에서 재로드 안 함)
// 2. ready를 volatile로 변경하면: 항상 종료, data 값 보장

// JVM 플래그로 확인:
// java -XX:-TieredCompilation VisibilityTest    ← 인터프리터 모드 (재현 어려움)
// java -XX:+TieredCompilation VisibilityTest    ← JIT 활성 (무한루프 재현 용이)
```

### 실험 2: synchronized 배리어 비용 측정

```java
import java.util.concurrent.atomic.*;

public class SyncBarrierBench {
    static final int THREADS = 4;
    static final long ITERS = 10_000_000L;

    // 버전 1: synchronized (thin lock 경로)
    static long syncCount = 0;
    static final Object syncLock = new Object();

    // 버전 2: AtomicLong (lock cmpxchg 직접)
    static AtomicLong atomicCount = new AtomicLong(0);

    // 버전 3: volatile (배리어만, 원자성 없음 → 부정확하지만 비용 측정용)
    static volatile long volCount = 0;

    static long bench(Runnable task) throws InterruptedException {
        Thread[] threads = new Thread[THREADS];
        long start = System.nanoTime();
        for (int i = 0; i < THREADS; i++) {
            threads[i] = new Thread(task);
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        return System.nanoTime() - start;
    }

    public static void main(String[] args) throws Exception {
        // 워밍업
        for (int w = 0; w < 3; w++) {
            bench(() -> { for (long i = 0; i < ITERS/10; i++)
                synchronized (syncLock) { syncCount++; } });
        }
        syncCount = 0;

        long t1 = bench(() -> {
            for (long i = 0; i < ITERS; i++)
                synchronized (syncLock) { syncCount++; }
        });

        long t2 = bench(() -> {
            for (long i = 0; i < ITERS; i++)
                atomicCount.incrementAndGet();
        });

        long t3 = bench(() -> {
            for (long i = 0; i < ITERS; i++)
                volCount++;  // 비원자적 (측정 목적만)
        });

        System.out.printf("synchronized:  %,d ms  count=%d%n", t1/1_000_000, syncCount);
        System.out.printf("AtomicLong:    %,d ms  count=%d%n", t2/1_000_000, atomicCount.get());
        System.out.printf("volatile(비정확): %,d ms%n", t3/1_000_000);
    }
}
// javac SyncBarrierBench.java
// java -server SyncBarrierBench

// 예상 출력 (4코어 x86-64):
// synchronized:    2,800 ms  (thin lock CAS + 경합 시 inflation)
// AtomicLong:      3,200 ms  (lock cmpxchg + 핑퐁, 4스레드 경합)
// volatile(비정확):  350 ms  (배리어만, 결과 부정확)
```

### 실험 3: JMM 배리어 체인 확인 (Java 9+ VarHandle)

```java
import java.lang.invoke.*;

// VarHandle: Java 9+에서 메모리 접근의 세밀한 제어
// C++의 std::atomic::memory_order와 동일한 수준의 제어 가능
public class VarHandleBarrier {
    static int data = 0;
    static int flag = 0;

    static final VarHandle DATA, FLAG;
    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            DATA = l.findStaticVarHandle(VarHandleBarrier.class, "data", int.class);
            FLAG = l.findStaticVarHandle(VarHandleBarrier.class, "flag", int.class);
        } catch (Exception e) { throw new Error(e); }
    }

    void publisher() {
        DATA.setOpaque(42);           // 컴파일러 재정렬 방지 (하드웨어 배리어 없음)
        FLAG.setRelease(1);           // release store: StoreStore+LoadStore 배리어
        // x86: MOV + (묵시적 TSO)
        // ARM: STR data + STLR flag
    }

    int subscriber() {
        int f = (int) FLAG.getAcquire();  // acquire load: LoadLoad+LoadStore 배리어
        // x86: MOV (TSO 자동 보장)
        // ARM: LDAR flag
        if (f != 0) {
            return (int) DATA.get();   // 일반 load (happens-before 이미 성립)
        }
        return -1;
    }

    // VarHandle의 memory_order 종류 (C++ 대응):
    // set()          / get()          ← volatile (seq_cst 수준)
    // setOpaque()    / getOpaque()    ← relaxed (컴파일러 재정렬만 차단)
    // setRelease()   / getAcquire()   ← release/acquire
    // getAndSet()    / compareAndSet()← CAS (seq_cst)

    // godbolt 동등 실험 (C++ memory_order와 1:1 비교):
    // setRelease → store(memory_order_release)
    // getAcquire → load(memory_order_acquire)
    // set        → store(memory_order_seq_cst)
}
```

---

## 📊 성능 비교

```
JMM 동기화 도구별 단일 연산 비용 (x86-64, HotSpot 21, 경합 없음):

도구                                비용(ns/op)   어셈블리
──────────────────────────────────────────────────────────────────
일반 필드 읽기/쓰기                    0.25 ns     MOV
volatile 읽기 (x86)                   0.3 ns      MOV (추가 배리어 없음)
volatile 쓰기 (x86)                  18 ns        MOV + LOCK ADD
VarHandle.getAcquire (x86)            0.3 ns      MOV
VarHandle.setRelease (x86)            0.3 ns      MOV
synchronized (biased, 재진입)          0.5 ns      헤더 비교만
synchronized (thin, 경합 없음)         20 ns       LOCK CMPXCHG
synchronized (inflated, 경합 있음)     ~500 ns     OS futex + 스케줄링
AtomicLong.get (relaxed)              0.3 ns      MOV
AtomicLong.incrementAndGet            18 ns       LOCK XADD
AtomicLong.compareAndSet              18 ns       LOCK CMPXCHG
LongAdder.increment (경합 없음)        18 ns       LOCK XADD (Cell)
LongAdder.increment (8스레드 경합)      2 ns        경합 없는 Cell 접근

ARM64 대응 비용 (Apple M1):
volatile 읽기 (ARM)                   0.8 ns      LDAR (~3배)
volatile 쓰기 (ARM)                    1.1 ns      STLR (~4배)
synchronized (thin, ARM)               1.5 ns      LDAXR/STXR

x86 vs ARM volatile 비교:
  x86: 읽기 무료, 쓰기 비쌈 (LOCK ADD ~18ns)
  ARM: 읽기도 비용 있음 (LDAR ~0.8ns), 쓰기도 비용 (STLR ~1.1ns)
  → x86: 읽기가 많은 패턴에서 volatile 비용 낮음
  → ARM: 읽기도 약간 느리지만, 쓰기는 x86보다 훨씬 저렴
```

---

## ⚖️ 트레이드오프

```
JMM 동기화 도구 선택 가이드:

volatile:
  ✅ 단순 플래그, 상태 공개에 최적
  ✅ 읽기 비용 거의 없음 (x86)
  ✅ happens-before 추론이 쉬움
  ❌ 복합 연산(read-modify-write) 원자성 없음
  ❌ 쓰기가 비쌈 (x86: LOCK ADD ~18ns)
  ❌ check-then-act 패턴에 사용 불가 (경쟁 조건)

synchronized:
  ✅ 복합 연산 원자성 보장
  ✅ 데드락 분석이 비교적 쉬움 (Java Monitor)
  ✅ biased → thin → inflated 자동 조정
  ❌ inflated 상태: OS 개입 → ~500ns
  ❌ 잠금 범위가 넓으면 처리량 제한

AtomicLong/AtomicReference:
  ✅ CAS 기반 → OS 없이 원자적 연산
  ✅ 경합 없을 때 synchronized보다 빠름
  ❌ 고경합에서 CAS 폭풍 → synchronized보다 느릴 수 있음
  ❌ 복잡한 다변수 원자성은 불가 (CAS는 단일 변수)

LongAdder (고경합 카운터):
  ✅ @Contended + 분산 Cell → 고경합에서 최강 처리량
  ❌ 순간 정확한 값 조회 불가 (sum()은 스냅샷 아님)
  ❌ 작은 값이나 저경합에서는 AtomicLong과 비슷하거나 느림

VarHandle (Java 9+):
  ✅ C++ atomic과 동일한 수준의 세밀한 memory_order 제어
  ✅ getAcquire/setRelease로 불필요한 seq_cst 회피
  ❌ API 복잡도 높음 → 일반 개발자에게 부담
  ❌ 잘못 쓰면 volatile보다 약한 보장으로 버그 발생

실무 결정 트리:
  단순 카운터 (최고 성능)?       → LongAdder
  단순 참조 교환?                → AtomicReference
  플래그 공개 (값 하나)?         → volatile
  임계 구역 (여러 변수 일관성)?   → synchronized
  성능 중요 + 세밀한 제어?       → VarHandle.setRelease/getAcquire
  검증된 컬렉션?                 → ConcurrentHashMap, CopyOnWriteArrayList
```

---

## 📌 핵심 정리

```
JMM-하드웨어 브릿지 핵심:

JMM happens-before 6가지 규칙:
  프로그램 순서 / 모니터 잠금 / volatile / 스레드 시작 / 스레드 종료 / 전이
  → 이 규칙이 충족되면 가시성 보장

JIT 번역 원칙:
  JMM 규칙 → 플랫폼별 최소 비용 배리어 선택
  x86 (TSO): LoadLoad/StoreStore/LoadStore 자동 보장 → 주로 LOCK ADD/XCHG
  ARM (weak): 모든 재정렬 가능 → STLR/LDAR 적극 사용

volatile:
  쓰기: acquire/release = "이전 쓰기 모두 완료 후 공개"
  x86: MOV + LOCK ADD | ARM: STLR
  읽기: acquire = "이 이후 읽기는 최신값 보장"
  x86: MOV (추가 비용 없음) | ARM: LDAR

synchronized:
  진입: acquire 배리어 (이전 소유자 쓰기 가시화)
  탈출: release 배리어 (내 쓰기 다음 진입자에 가시화)
  구현: biased(공짜) → thin CAS(fast) → inflated OS(slow)

Chapter 3 전체 연결:
  MESI (01) → 코어 간 일관성 기반
  False Sharing (02) → 레이아웃이 만드는 핑퐁
  재정렬 (03) → Store Buffer + 약한 모델
  배리어 (04) → mfence/dmb로 순서 강제
  CAS (05) → lock cmpxchg가 MESI 가로챔
  JMM (06) → 모든 것의 Java 추상화

java-concurrency-deep-dive 레포로의 다리:
  이 문서가 그 레포에서 배운 규칙들의 "왜"를 설명
  volatile happens-before → STLR/LDAR
  synchronized acquire/release → LOCK CMPXCHG
  AtomicLong.CAS → lock cmpxchg
  LongAdder → @Contended + 분산 CAS
  AQS (ReentrantLock 등) → state CAS + park/unpark
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드는 JMM 관점에서 올바른가? x86과 ARM에서 각각 실제로 안전한가?

```java
class Singleton {
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

<details>
<summary>해설 보기</summary>

**이 코드는 JMM 관점에서 안전하지 않습니다.** 이것이 유명한 "Double-Checked Locking" 문제입니다.

**문제:**

`instance = new Singleton()`은 3단계로 이루어집니다:
1. 메모리 할당 (`new`)
2. 생성자 실행 (`Singleton()`)
3. `instance`에 참조 저장

JMM에서 2와 3의 순서가 재정렬될 수 있습니다. 즉, `instance`가 완전히 초기화되지 않은 상태에서 참조가 저장될 수 있습니다.

외부의 `if (instance == null)` 체크는 `synchronized` 블록 밖에 있으며, `instance`가 `volatile`이 아니므로 happens-before 보장이 없습니다. 따라서 다른 스레드가 부분적으로 초기화된 객체를 볼 수 있습니다.

**x86에서:** TSO 덕분에 StoreStore 재정렬이 없어서 우연히 대부분의 경우 동작합니다. 하지만 JMM 보장이 없으므로 "작동하는 것처럼 보이는 버그"입니다.

**ARM에서:** StoreStore 재정렬이 허용되므로 실제로 깨질 수 있습니다.

**올바른 수정 방법:**

```java
// 방법 1: volatile (Java 5+ JMM 수정으로 안전)
private static volatile Singleton instance;
// volatile 쓰기가 release 배리어를 삽입해 생성자 완료 후 instance 공개 보장

// 방법 2: 클래스 로더 초기화 (가장 권장)
class Singleton {
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
        // 클래스 로더가 INSTANCE를 한 번만 초기화 → 스레드 안전
    }
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}

// 방법 3: enum (Item 3 in Effective Java)
enum Singleton { INSTANCE; }
```

</details>

---

**Q2.** `java.util.concurrent.CompletableFuture.complete()`가 호출된 이후에 `thenApply()` 콜백이 실행될 때, complete() 이전의 쓰기가 콜백에서 보이는 것은 JMM의 어느 규칙으로 보장되는가?

<details>
<summary>해설 보기</summary>

`CompletableFuture.complete(value)` → `thenApply(fn)` 콜백 실행까지의 happens-before 체인:

1. **`complete(value)` 내부:**
   - `CompletableFuture`의 내부 `result` 필드는 `volatile`로 선언되어 있습니다 (`volatile Object result`).
   - `result = value`는 **volatile 쓰기** → **release 배리어**
   - JMM volatile 규칙: `complete()` 이전의 모든 쓰기 hb `result` volatile 쓰기

2. **`thenApply()` 콜백 실행 시:**
   - 콜백 스레드가 `result`를 읽어 완료 여부 확인
   - `result` **volatile 읽기** → **acquire 배리어**
   - JMM volatile 규칙: `result` volatile 쓰기 hb `result` volatile 읽기
   - → `complete()` 이전 쓰기 hb 콜백 실행

3. **전이 규칙:**
   `complete() 이전 쓰기` hb `volatile 쓰기` hb `volatile 읽기` hb `콜백`
   → 전이적으로: `complete() 이전 쓰기` hb `콜백`

**하드웨어 수준:**
- `complete()`: `result` volatile 쓰기 → x86: LOCK ADD (StoreLoad 배리어), ARM: STLR
- 콜백 트리거: `result` volatile 읽기 → x86: MOV, ARM: LDAR
- 두 배리어가 쌍을 이뤄 complete() 이전 데이터가 콜백에서 완전히 가시화됨

이 보장 때문에 `CompletableFuture`를 통해 스레드 간 데이터를 안전하게 전달할 수 있으며, 추가적인 `volatile`이나 `synchronized` 없이도 완결성이 보장됩니다.

</details>

---

**Q3.** 다음 두 구현은 동일한 결과를 보장하는가? 하드웨어 배리어 관점에서 비교하라.

```java
// 구현 A
class A {
    volatile long counter = 0;
    void increment() { counter++; }
}

// 구현 B
class B {
    long counter = 0;
    void increment() {
        synchronized (this) { counter++; }
    }
}
```

<details>
<summary>해설 보기</summary>

**두 구현은 동일한 결과를 보장하지 않습니다.**

**구현 A (volatile counter++)의 문제:**

`counter++`는 읽기(load) → +1 연산 → 쓰기(store)의 3단계로, **원자적이지 않습니다.** `volatile`은 개별 읽기와 쓰기에 배리어를 삽입하지만, 읽기-수정-쓰기 전체를 원자적으로 만들지 않습니다.

타이밍:
```
Thread A: load(counter=0) ... store(1)
Thread B:              load(counter=0) ... store(1)
결과: counter=1  (정답은 2)
```

따라서 여러 스레드가 동시에 `increment()`를 호출하면 갱신이 유실될 수 있습니다.

**구현 B (synchronized):**

`synchronized`는 `counter++` 전체를 원자적 임계 구역으로 감싸므로 올바릅니다.

**하드웨어 배리어 관점:**

| 동작 | 구현 A (volatile) | 구현 B (synchronized) |
|------|-------------------|------------------------|
| 읽기 | MOV (acquire, x86) | LOCK CMPXCHG (진입) |
| 쓰기 | MOV + LOCK ADD | MOV (임계 구역 내) |
| 원자성 | 읽기·쓰기 각각만 | 읽기-수정-쓰기 전체 |

**올바른 락프리 대안:** `AtomicLong.incrementAndGet()` → 내부에서 `lock xadd` 또는 `lock cmpxchg` 루프로 원자적 read-modify-write를 보장합니다.

고경합이라면 `LongAdder.increment()`가 훨씬 효율적입니다 (`02-false-sharing.md` 참조).

</details>

---

<div align="center">

**[⬅️ 이전: 원자적 연산과 CAS](./05-atomic-cas.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 추상화의 숨은 비용 ➡️](../abstraction-cost/01-function-call-cost.md)**

</div>
