# 멀티코어 캐시 일관성 — MESI 프로토콜

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 코어1이 변수를 수정했을 때 코어2가 가진 같은 캐시라인은 어떻게 되는가?
- MESI의 4가지 상태(Modified/Exclusive/Shared/Invalid)가 각각 무엇을 의미하는가?
- 버스 스누핑(Bus Snooping)과 디렉토리 기반(Directory-based) 일관성은 어떻게 다른가?
- 캐시라인 하나를 두 코어가 동시에 쓰려 하면 무슨 일이 일어나는가?
- MESIF(Intel)과 MOESI(AMD)는 MESI에 무엇을 추가했는가?
- Java의 `volatile`과 `synchronized`는 결국 어떤 하드웨어 메커니즘에 의존하는가?

---

## 🔍 왜 이 개념이 중요한가

### "멀쩡한 코드가 멀티코어에서 깨지는 이유"

```
싱글 코어 시절의 상식:
  코어가 하나 → 한 번에 한 명령 실행
  → 내가 쓴 값은 즉시 메모리에 반영
  → 다른 누군가가 읽을 때 항상 최신 값

멀티코어의 현실:
  Core0  Core1  Core2  Core3
   L1     L1     L1     L1   ← 코어별 개인 캐시
    \      \    /      /
          L2/L3             ← 공유 캐시 (또는 개별)
            |
          DRAM

  Core0이 x = 42로 쓰면:
    Core0의 L1에만 42가 있음
    Core1의 L1에는 아직 0이 있을 수 있음!

  이 상태에서 Core1이 x를 읽으면?
    → 오래된 값(stale value) 반환
    → 프로그래머가 보기엔 "말도 안 되는" 동작

이 문제를 해결하는 하드웨어 메커니즘 = 캐시 일관성 프로토콜
```

캐시 일관성(Cache Coherence)은 java-concurrency-deep-dive 레포에서 다루는 `volatile`, `synchronized`, `AtomicLong`의 **하드웨어 기반**입니다. 왜 `volatile` 읽기가 항상 최신 값을 보는지, 왜 `lock` 프리픽스가 필요한지는 전부 이 프로토콜이 설명합니다.

---

## 😱 잘못된 이해

### Before: 캐시는 단순한 "빠른 메모리"

```
흔한 오해:
  "캐시는 그냥 DRAM보다 빠른 복사본이다"
  "코어가 쓰면 자동으로 다른 코어에도 전파된다"
  "volatile이면 캐시를 우회해서 DRAM에 직접 쓴다"

이 오해에서 나오는 코드:
  // 틀린 가정: flag가 바뀌면 다른 스레드가 즉시 본다
  boolean flag = false;

  // Thread A (Core0에서 실행):
  flag = true;

  // Thread B (Core1에서 실행):
  while (!flag) { /* spin */ }  // 무한루프 가능!

왜 무한루프가 되는가:
  Core1의 L1 캐시에 flag=false가 캐싱됨
  Core0이 flag=true로 써도 Core1의 캐시는 갱신 안 됨
  Core1은 자신의 L1에서 flag를 읽으므로 계속 false
  → 컴파일러 최적화까지 겹치면 더 심각해짐

"volatile이 DRAM에 직접 쓴다"는 오해:
  volatile은 캐시를 우회하지 않는다
  volatile은 캐시 일관성 프로토콜을 통해 다른 코어의
  캐시라인을 Invalid 상태로 만들고, 메모리 순서를 보장한다
```

---

## ✨ 올바른 이해

### After: MESI 프로토콜이 캐시라인마다 상태를 관리한다

```
MESI의 핵심 아이디어:
  캐시라인 하나하나에 4가지 상태 태그를 붙인다
  코어 간의 모든 읽기/쓰기가 이 상태를 바꾼다
  상태 천이(State Transition)를 통해 일관성 보장

4가지 상태:
  M (Modified)  : 이 코어만 가지고 있고, DRAM과 다른 값
  E (Exclusive) : 이 코어만 가지고 있고, DRAM과 같은 값
  S (Shared)    : 여러 코어가 같은 값을 공유 중
  I (Invalid)   : 유효하지 않음 (없는 것과 마찬가지)

핵심 규칙:
  M 상태 라인은 오직 하나의 코어만 가질 수 있다
  한 코어가 쓰기를 시도하면 다른 모든 코어의 해당 라인을
  I(Invalid) 상태로 만든다 → "Invalidation"
```

---

## 🔬 내부 동작 원리

### 1. MESI 4가지 상태와 천이 다이어그램

```
상태 정의:

┌──────────────────────────────────────────────────────────┐
│ 상태        의미                                 DRAM 동기 │
├──────────────────────────────────────────────────────────┤
│ Modified(M) 이 코어만 보유, 변경됨 (dirty)       아니오   │
│ Exclusive(E)이 코어만 보유, 미변경              예       │
│ Shared(S)  여러 코어가 보유, 미변경              예       │
│ Invalid(I) 유효하지 않음 (miss처럼 취급)         해당없음  │
└──────────────────────────────────────────────────────────┘

MESI 상태 천이 다이어그램:

         PrWr (자신이 쓰기)
         ┌───┐
         │   ▼
  ┌──────────────┐
  │   Modified   │◄──────────────────────────────────┐
  │     (M)      │                                   │
  └──────┬───────┘                                   │
         │ BusRd (다른 코어 읽기)                     │ PrWr (다른 코어 Invalid 후 쓰기)
         │ → DRAM으로 Writeback                      │
         ▼                                           │
  ┌──────────────┐   BusRd (다른 코어도 읽음)  ┌──────────────┐
  │  Exclusive   │──────────────────────────►│   Shared     │
  │     (E)      │                           │     (S)      │
  └──────┬───────┘                           └──────┬───────┘
         │ PrWr (자신이 쓰기)                       │ BusRdX (다른 코어 쓰기 요청)
         │ → E→M (BusRdX 불필요)                   │ → 모든 S 상태 → I로
         ▼                                          ▼
  ┌──────────────┐                           ┌──────────────┐
  │   Modified   │                           │   Invalid    │
  │     (M)      │◄──────────────────────────│     (I)      │
  └──────────────┘  PrRd (캐시 미스 후 로드)  └──────────────┘

약어:
  PrRd  = Processor Read  (이 코어의 읽기)
  PrWr  = Processor Write (이 코어의 쓰기)
  BusRd = Bus Read        (버스에서 읽기 요청 감지)
  BusRdX= Bus Read Exclusive (버스에서 쓰기 의도 읽기 요청)
```

### 2. 코어1이 쓸 때 코어2의 캐시라인이 Invalid가 되는 과정

```
초기 상태: x = 0 이 DRAM과 Core0, Core1 L1에 모두 캐싱

Core0 L1: [주소 A | 값 0 | 상태: S]
Core1 L1: [주소 A | 값 0 | 상태: S]
DRAM:     [주소 A | 값 0]

Step 1: Core0이 x = 42 쓰기 시도
  Core0이 BusRdX (Write Intent) 메시지를 버스에 발행
  "나 주소 A 쓸 건데, 다른 코어들 무효화해"

Step 2: Core1이 BusRdX를 스누핑(Snooping)
  Core1은 자신의 L1에 주소 A 라인이 있음을 확인
  해당 라인을 S → I 상태로 변경
  "나는 이 라인 버립게"

Step 3: Core0이 쓰기 완료
  Core0 L1: [주소 A | 값 42 | 상태: M]
  Core1 L1: [주소 A | ??? | 상태: I]  ← Invalid!
  DRAM:     [주소 A | 값 0]           ← 아직 갱신 안 됨

Step 4: Core1이 x를 읽으려 함
  Core1 L1: 상태가 I → Cache Miss 발생
  Core1이 BusRd 요청을 버스에 발행

Step 5: Core0이 BusRd를 스누핑
  Core0은 자신의 라인이 M 상태임을 확인
  DRAM에 먼저 Writeback (42 기록)
  Core0 라인: M → S 상태로 변경

Step 6: Core1이 DRAM(또는 Core0)에서 42를 읽어옴
  Core1 L1: [주소 A | 값 42 | 상태: S]
  Core0 L1: [주소 A | 값 42 | 상태: S]
  DRAM:     [주소 A | 값 42]

→ 이제 양쪽 코어가 42를 본다!
```

### 3. 버스 스누핑 (Bus Snooping) vs 디렉토리 기반

```
버스 스누핑 방식:
  ┌────────────────────────────────────────────┐
  │           공유 버스 (Shared Bus)            │
  └──┬───────────┬───────────┬───────────┬────┘
     │           │           │           │
  Core0       Core1       Core2       Core3
  (L1+L2)     (L1+L2)     (L1+L2)     (L1+L2)

  동작:
    모든 캐시가 버스를 항상 감청(Snoop)
    한 코어의 메모리 요청이 버스에 브로드캐스트
    모든 코어가 자신의 캐시를 확인하고 응답

  장점:
    단순하고 빠름 (직접 코어 간 통신)
    지연시간 낮음

  단점:
    버스 대역폭이 병목 → 코어 수 증가에 비례해 트래픽 폭증
    일반적으로 2~8 소켓 이하의 소규모 시스템에 적합
    현대 서버급 CPU(수십~수백 코어)에는 부적합

디렉토리 기반 방식:
  ┌─────────┐    ┌─────────┐    ┌─────────┐
  │ Core0   │    │ Core1   │    │ Core2   │
  │ (L1+L2) │    │ (L1+L2) │    │ (L1+L2) │
  └────┬────┘    └────┬────┘    └────┬────┘
       │              │              │
  ┌────▼──────────────▼──────────────▼────┐
  │            인터커넥트 네트워크          │
  └────┬──────────────────────────────────┘
       │
  ┌────▼────────────────────────┐
  │   Directory (중앙 디렉토리)  │
  │ 주소 A: Core0(M), Core2(S)  │
  │ 주소 B: Core1(E)            │
  │ 주소 C: 없음                │
  └─────────────────────────────┘

  동작:
    각 캐시라인마다 "어느 코어가 이 라인을 갖고 있는지" 기록
    쓰기 요청 → 디렉토리 조회 → 해당 코어에만 Invalidation 전송
    브로드캐스트 없음 → 포인트-투-포인트 통신

  장점:
    수십~수천 코어 규모의 확장성 (Scalable)
    네트워크 트래픽이 필요한 코어에만 집중

  단점:
    디렉토리 자체가 메모리/지연 오버헤드
    구현 복잡도 높음

실제 사용:
  x86 데스크톱/노트북 (4~16 코어): 링 버스 + 스누핑
  Intel Xeon 서버 (수십~수백 코어): 메시 네트워크 + 디렉토리 혼합
  AMD EPYC (Infinity Fabric): 링/메시 + 분산 디렉토리
  ARM 대형 시스템: AMBA CHI 프로토콜 (디렉토리 기반)
```

### 4. MESIF (Intel) / MOESI (AMD) — 확장 프로토콜

```
MESIF (Intel, LLC 캐시 활용):
  기존 MESI에 F(Forward) 상태 추가

  Forward(F) 상태:
    여러 코어가 공유 중(Shared)이지만,
    이 라인에 대한 요청은 DRAM 대신 F 상태 코어가 공급
    → DRAM 대역폭 절약

  예시:
    Core0: 주소 A = 100 (S)
    Core1: 주소 A = 100 (S)
    Core2: 주소 A = 100 (F)  ← "나는 공급 담당"

    Core3가 주소 A 요청 → Core2가 직접 공급 (DRAM 불필요)
    → 멀티소켓 시스템에서 DRAM 트래픽 감소

MOESI (AMD, 직접 코어 간 전달):
  기존 MESI에 O(Owned) 상태 추가

  Owned(O) 상태:
    이 코어가 최신 값을 갖고 있으며 DRAM은 오래된 상태
    하지만 다른 코어들도 읽기 가능 (Shared처럼)
    → Writeback 없이 직접 다른 코어에 전달 가능

  MESI vs MOESI 비교:
    MESI에서 M 상태 라인을 다른 코어가 읽으면:
      M → S (Writeback to DRAM 필요)
      느림: Writeback I/O 발생

    MOESI에서 M 상태 라인을 다른 코어가 읽으면:
      M → O (Writeback 없이 공급)
      빠름: DRAM I/O 생략, 캐시 간 직접 전달

실질적 성능 차이:
  MOESI의 O 상태는 코어 간 데이터 공유가 잦은 워크로드에서
  DRAM 대역폭 사용량을 크게 줄여줌
  AMD의 멀티코어 서버(EPYC)가 NUMA 환경에서 효율적인 이유 중 하나
```

### 5. MESI와 Java volatile의 연결

```java
// java-concurrency-deep-dive 레포와의 연결 지점

// volatile 변수 x를 가진 클래스
class Shared {
    volatile int x = 0;
}

// Thread A (Core0에서 실행):
shared.x = 42;
// JIT 산출물 (x86-64):
//   MOV [addr_x], 42
//   MFENCE          ← Store-Load 배리어
// → MFENCE 이후에 Core1의 x 캐시라인이 Invalid 처리됨

// Thread B (Core1에서 실행):
int val = shared.x;
// JIT 산출물 (x86-64):
//   MOV eax, [addr_x]  ← 캐시 miss → MESI 프로토콜로 최신 값 취득
// → Invalid 상태였으므로 BusRd 발행 → Core0에서 42를 받아옴

// 결과: val = 42 보장
// 근거: MESI 프로토콜의 Invalidation + 배리어의 조합
```

---

## 💻 실전 실험

### 실험 1: perf로 캐시 일관성 트래픽 측정

```bash
# LLC(Last Level Cache) 관련 이벤트 확인
perf list | grep -i "cache\|coherence\|snoop"

# 두 코어가 같은 캐시라인을 놓고 경쟁하는 워크로드 생성
cat > mesi_demo.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdint.h>

// 두 스레드가 같은 라인을 반복해서 읽고 씀
volatile int64_t shared_var = 0;

void* writer(void* arg) {
    for (long i = 0; i < 100000000L; i++)
        shared_var = i;
    return NULL;
}

void* reader(void* arg) {
    long sum = 0;
    for (long i = 0; i < 100000000L; i++)
        sum += shared_var;
    return (void*)sum;
}

int main() {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, writer, NULL);
    pthread_create(&t2, NULL, reader, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    return 0;
}
EOF

gcc -O2 -pthread -o mesi_demo mesi_demo.c

# 캐시 미스와 LLC 이벤트 측정
perf stat -e cycles,instructions,cache-misses,cache-references,\
LLC-loads,LLC-load-misses,LLC-stores,LLC-store-misses \
./mesi_demo

# 예상: cache-misses가 매우 높음 (코어 간 핑퐁으로 인한 Invalid → miss 반복)
```

### 실험 2: taskset으로 코어 지정 후 비교

```bash
# 같은 물리 코어의 두 HyperThread (L1 공유, MESI 불필요)
taskset -c 0,1 ./mesi_demo   # HT pair (fast)

# 다른 물리 코어 (L1 분리, MESI 풀가동)
taskset -c 0,4 ./mesi_demo   # Different physical cores (slow)

# NUMA 노드를 넘어가는 경우
numactl --cpunodebind=0 taskset -c 0 ./mesi_demo &
numactl --cpunodebind=1 taskset -c 8 ./mesi_demo &

# lscpu로 코어 토폴로지 확인
lscpu -e | head -20
# CORE 컬럼이 같으면 HyperThread pair
```

### 실험 3: MESI 상태 천이를 litmus test로 관찰

```c
// 두 코어의 쓰기 순서를 관찰하는 litmus test
// (실제로는 메모리 재정렬과 함께 나타남 — 03-memory-reordering.md에서 상세)

#include <pthread.h>
#include <stdio.h>
#include <stdatomic.h>

atomic_int x = 0, y = 0;
int r1, r2;

void* thread_a(void* _) {
    atomic_store_explicit(&x, 1, memory_order_relaxed);
    r1 = atomic_load_explicit(&y, memory_order_relaxed);
    return NULL;
}

void* thread_b(void* _) {
    atomic_store_explicit(&y, 1, memory_order_relaxed);
    r2 = atomic_load_explicit(&x, memory_order_relaxed);
    return NULL;
}

// r1==0 && r2==0 이 가능한가?
// x86-64에서는 TSO 때문에 이론상 가능 (StoreLoad 재정렬)
// MESI만으로는 이 경우를 막지 못함 → 메모리 배리어가 필요한 이유
// → 03-memory-reordering.md, 04-memory-barriers.md에서 상세 분석
```

---

## 📊 성능 비교

```
캐시 일관성 프로토콜 오버헤드 측정 (Intel Core i9, 8코어):

시나리오                          처리량 (ops/sec)   지연시간
─────────────────────────────────────────────────────────────
단일 코어 쓰기 (E→M 반복)           ~1,000M           1 cy
두 코어 쓰기 경쟁 (M→I→M 핑퐁)       ~15M            ~100 cy
두 코어 읽기 공유 (E→S)             ~500M             4 cy
한 코어 쓰기 + 나머지 읽기 (M→I)     ~20M            ~80 cy

캐시라인 상태별 접근 비용:
  M (Modified, 이 코어 소유):    L1 Hit ~4 cy
  E (Exclusive, 이 코어 소유):   L1 Hit ~4 cy
  S (Shared, 여러 코어 공유):    L1 Hit ~4 cy
  I (Invalid, 캐시 미스):        ~40~200 cy (L3 또는 DRAM 접근 + 프로토콜 오버헤드)

코어 간 캐시 직접 전달 (MOESI O 상태):
  같은 소켓, 인접 코어:   ~40 cy
  같은 소켓, 먼 코어:     ~60 cy
  다른 소켓 (NUMA):       ~100~200 cy

실제 영향:
  핑퐁이 없는 경우 대비 핑퐁이 있는 경우: 처리량 5~50배 저하
  → False Sharing 문제의 근본 원인 (02-false-sharing.md에서 측정)
```

---

## ⚖️ 트레이드오프

```
캐시 일관성 프로토콜의 설계 트레이드오프:

버스 스누핑:
  ✅ 단순한 구현 → 낮은 지연시간
  ✅ 코어 수가 적을 때 매우 효율적
  ❌ 코어 수 증가 → 버스 트래픽 O(N²) 증가
  ❌ 수십 코어 이상에서 버스가 병목

디렉토리 기반:
  ✅ 수백~수천 코어로 확장 가능
  ✅ 포인트-투-포인트 통신 → 불필요한 트래픽 없음
  ❌ 디렉토리 자체의 메모리 공간과 조회 지연
  ❌ 구현 복잡도 높음

MESI vs MOESI:
  MESI:
    ✅ 단순하고 검증된 프로토콜
    ❌ Writeback 강제 → DRAM 대역폭 추가 소비
  MOESI:
    ✅ Writeback 생략 → DRAM 대역폭 절약
    ✅ 코어 간 직접 전달 → 지연시간 감소
    ❌ 구현이 더 복잡

프로그래머 관점:
  ✅ 하드웨어가 일관성을 자동으로 보장 → 개발자는 신경 안 써도 됨
  ❌ 성능은 여전히 개발자 책임:
       False Sharing → 의도치 않은 핑퐁 → 처리량 붕괴
       과도한 공유 상태 → Shared 라인이 많으면 확장성 저하
  ✅ volatile/atomic의 하드웨어 기반을 알면 언제 쓸지 판단 가능
  ❌ 프로토콜의 존재를 모르면 왜 느린지 이해 불가
```

---

## 📌 핵심 정리

```
MESI 캐시 일관성 프로토콜 핵심:

4가지 상태:
  M (Modified)  : 이 코어만 보유, dirty (DRAM과 다름)
  E (Exclusive) : 이 코어만 보유, clean (DRAM과 같음)
  S (Shared)    : 여러 코어가 동일한 값을 공유
  I (Invalid)   : 유효하지 않음 → 다음 접근 시 캐시 미스

핵심 규칙:
  쓰기 전 반드시 M 상태 획득 (독점 소유)
  M 획득 = 다른 모든 코어의 해당 라인 → I 상태 (Invalidation)
  Invalidation은 버스 메시지 또는 디렉토리를 통해 전파

버스 스누핑 vs 디렉토리:
  스누핑: 모든 코어가 버스 감청 → 소규모 고성능
  디렉토리: 중앙 기록 → 대규모 확장 가능

MESIF (Intel) / MOESI (AMD):
  MESIF: F(Forward) 상태 → Shared 중 한 코어가 공급 담당 → DRAM 절약
  MOESI: O(Owned) 상태  → Writeback 없이 코어 간 직접 전달 → 지연 감소

java-concurrency 연결:
  volatile 읽기/쓰기 → MESI Invalidation + 메모리 배리어의 조합
  synchronized → MESI + lock 프리픽스 (05-atomic-cas.md)
  AtomicLong   → lock cmpxchg → MESI M 상태 독점 획득
  자세한 매핑 → 06-jmm-hardware-bridge.md
```

---

## 🤔 생각해볼 문제

**Q1.** Core0이 주소 A의 값을 수정 중이고(M 상태), 동시에 Core1도 같은 주소 A에 쓰기를 시도한다. 두 코어 모두 M 상태를 동시에 가질 수는 없는데, 이 경쟁 상황은 어떻게 해결되는가?

<details>
<summary>해설 보기</summary>

**MESI 프로토콜의 경쟁 해결:**

두 코어가 동시에 BusRdX(쓰기 의도 요청)를 발행하면 버스(또는 인터커넥트)가 중재자 역할을 합니다. 하드웨어 수준에서 **직렬화(Serialization)**가 발생합니다.

1. 버스/디렉토리가 한 요청을 먼저 처리합니다 (예: Core0 우선)
2. Core0이 M 상태를 획득하고 Core1의 라인은 I 상태가 됩니다
3. Core0이 쓰기를 완료합니다
4. Core1의 BusRdX가 다음으로 처리됩니다 — Core0의 M 상태 라인이 Writeback되고 Core1이 새로운 M 상태를 획득합니다
5. Core1이 쓰기를 완료합니다

결과적으로 두 쓰기는 순차적으로 실행되며, 어느 것이 먼저냐는 하드웨어가 결정합니다. 이것이 x86에서 `lock` 프리픽스(`lock xchg`, `lock cmpxchg`)가 원자성을 보장하는 근본 메커니즘입니다 — 캐시라인에 대한 독점 M 상태를 잡은 다음 읽기-수정-쓰기를 한 번에 완료합니다.

</details>

---

**Q2.** 4코어 시스템에서 배열 `int arr[16]`을 각 코어가 `arr[0]`, `arr[4]`, `arr[8]`, `arr[12]`에 초당 수백만 회 씩 쓴다고 할 때, MESI 관점에서 어떤 일이 발생하는가? 그리고 각 코어가 `arr[0]`, `arr[16]`, `arr[32]`, `arr[48]`을 쓴다면 달라지는가?

<details>
<summary>해설 보기</summary>

**첫 번째 경우 (arr[0], arr[4], arr[8], arr[12]):**

`int`는 4 bytes이므로 이 네 원소는 각각 오프셋 0, 16, 32, 48 bytes에 있습니다. 캐시라인 크기가 64 bytes이므로 **arr[0]~arr[15] 전체가 단 하나의 캐시라인**에 들어갑니다.

Core0이 arr[0]을 쓰면 BusRdX 발행 → Core1, Core2, Core3의 라인이 모두 I 상태로 → Core1이 arr[4]를 쓰려면 다시 BusRdX 발행 → Core0의 라인이 I 상태로... 이 과정이 반복됩니다. 이것이 **False Sharing**이며 캐시라인이 코어 간에 핑퐁됩니다. 처리량이 단일 코어 대비 5~10배 저하될 수 있습니다 (02-false-sharing.md에서 측정).

**두 번째 경우 (arr[0], arr[16], arr[32], arr[48]):**

오프셋이 각각 0, 64, 128, 192 bytes이므로 **각 원소가 서로 다른 캐시라인**에 있습니다. 각 코어가 서로 다른 라인을 독점적으로 M 상태로 유지할 수 있어 Invalidation이 발생하지 않습니다. 처리량이 단일 코어의 4배에 가깝게 선형 확장됩니다.

</details>

---

**Q3.** `synchronized` 블록을 진입할 때와 탈출할 때 JVM은 각각 어떤 MESI 상태 전이를 유발하는가? `volatile` 쓰기와 비교하면?

<details>
<summary>해설 보기</summary>

**`volatile` 쓰기:**
- x86-64: `MOV [addr], val` + `MFENCE` (또는 `LOCK XCHG`)
- MESI: 해당 캐시라인을 M 상태로 만들고, 다른 코어의 같은 라인을 I 상태로 Invalidate
- 단 하나의 캐시라인에만 영향

**`synchronized` 진입 (`monitorenter`):**
- JVM이 모니터 객체의 헤더 word에 대해 CAS 수행 (`lock cmpxchg`)
- MESI: 모니터 객체가 있는 캐시라인을 M 상태로 독점 획득
- x86의 `lock` 프리픽스 → Acquire semantics (이전 Load/Store가 다 완료된 후 이 연산)
- 다른 코어가 같은 모니터를 잡으려 하면 I 상태 → 스핀 또는 대기

**`synchronized` 탈출 (`monitorexit`):**
- 모니터 해제 + `MFENCE` 또는 `LOCK ADD [rsp], 0` (Release semantics)
- MESI: `synchronized` 블록 안에서 수정한 모든 캐시라인이 다른 코어에 가시화됨
- 이후 어떤 코어든 최신 값 읽기 가능

핵심 차이: `volatile`은 **한 변수**의 가시성만 보장, `synchronized`는 **블록 안의 모든 변수**의 가시성을 보장합니다. 그러나 MESI 관점에서 둘 다 "해당 캐시라인(들)의 Invalidation + 배리어"라는 동일한 하드웨어 메커니즘을 사용합니다. 자세한 JIT 어셈블리 분석은 06-jmm-hardware-bridge.md를 참조하세요.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: TLB와 가상 메모리](../memory-hierarchy-cache/06-tlb-virtual-memory.md)** | **[홈으로 🏠](../README.md)** | **[다음: False Sharing ➡️](./02-false-sharing.md)**

</div>
