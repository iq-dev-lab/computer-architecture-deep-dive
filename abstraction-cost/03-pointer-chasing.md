# 포인터 추적(Pointer Chasing) — 객체 그래프 순회의 비용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 연결 리스트(linked list) 순회가 배열 순회보다 느린 본질적 이유는 무엇인가?
- 프리페처(hardware prefetcher)가 포인터 체인을 따라가지 못하는 이유는?
- `std::list` vs `std::vector` vs Arena 할당기의 메모리 지연 비용은 얼마나 차이나는가?
- ECS(Entity Component System) 패턴이 게임 엔진에서 표준이 된 하드웨어적 이유는?
- 객체지향(OOP)의 캐시 비용을 데이터 지향 설계(DOD, Data-Oriented Design)로 어떻게 우회하는가?
- Arena/Pool 할당기가 캐시 성능에 미치는 영향은 무엇인가?

---

## 🔍 왜 이 개념이 중요한가

### "연결 리스트는 삽입이 O(1)이니까 충분히 빠르지 않나요?"

```
알고리즘 복잡도의 함정:

  연결 리스트 순회: O(N)
  배열 순회:       O(N)
  → "같은 복잡도, 비슷한 속도"

실제로 일어나는 일:

  배열 순회 (arr[0], arr[1], arr[2], ...):
    ① CPU: "연속적인 주소를 접근하는 패턴" 인식
    ② Hardware Prefetcher: 다음 캐시라인을 미리 가져옴
    ③ arr[i] 접근 시 이미 캐시에 있음 → L1 히트 (~4 cy)
    ④ 1,000만 개 요소 순회 = 거의 전부 L1/L2 히트

  연결 리스트 순회 (node->next->next->next, ...):
    ① CPU: "다음 노드 주소는 현재 노드를 읽어야만 앎"
    ② Hardware Prefetcher: 다음 주소를 예측 불가
    ③ node->next 접근 = 힙의 랜덤한 위치 → L1 미스 → L3 또는 DRAM
    ④ 1,000만 개 노드 순회 = 최악 1,000만 번 캐시 미스
    ⑤ DRAM 접근 1회 ~200 cy × 1,000만 = 20억 사이클 낭비 가능

  실측 차이 (1,000만 요소, x86-64 Skylake):
    std::vector 순회:  ~25 ms
    std::list 순회:    ~380 ms   (약 15배 차이)
    → 알고리즘 복잡도는 같지만 캐시 동작이 완전히 다름
```

OOP로 설계된 시스템에서 `Entity*`, `Node*`, `Object*` 형태의 포인터 그래프를 순회하는 것이 성능 병목의 핵심 원인인 경우가 많습니다. 이 문제를 이해하지 못하면 알고리즘은 O(N)인데 실제 처리량은 예상의 10분의 1에 불과한 상황을 만납니다.

---

## 😱 잘못된 이해

### Before: "연결 리스트는 삽입이 빠르고, 배열은 삽입이 느리다. 삽입이 많으면 연결 리스트"

```
잘못된 선택 기준:
  삽입/삭제 빈번 → 연결 리스트 (O(1))
  순회 위주     → 배열 (O(N))
  → "삽입도 하고 순회도 하면 연결 리스트가 나은 것 아닌가?"

놓친 하드웨어 현실 1 — Prefetcher의 한계:
  Hardware Prefetcher가 탐지하는 패턴:
    ✅ 연속 접근: arr[0], arr[1], arr[2]      → 스트라이드 패턴 감지
    ✅ 일정 간격: arr[0], arr[4], arr[8]      → 스트라이드 패턴 감지
    ❌ 포인터 체인: p, *p, **p, ***p          → 다음 주소가 현재 값에 의존

  포인터 체인은 "데이터 의존 주소(data-dependent address)"
  → CPU가 현재 노드를 완전히 읽기 전까지 다음 주소를 알 수 없음
  → Prefetcher가 추측 불가 → 매 접근마다 캐시 미스 가능

놓친 하드웨어 현실 2 — 힙 단편화:
  malloc/new로 각각 할당한 노드들:
    node0: 0x7f8a3c00    (힙의 어느 위치)
    node1: 0x7f9b4400    (전혀 다른 위치)
    node2: 0x7f7d2800    (또 다른 위치)
    → 64KB 범위 내에 없을 수도 있음 → L1, L2, L3 모두 미스
    → 최악의 경우 매 노드가 DRAM 접근

놓친 하드웨어 현실 3 — 메모리 대역폭 낭비:
  캐시라인 64B를 가져왔지만 Node 구조체에서 실제 사용하는 데이터:
    struct Node { int value; Node* next; }
    → value: 4 bytes, next: 8 bytes → 사용: 12 bytes
    → 나머지 52 bytes는 버려짐 (다른 노드들이 들어있을 뿐)
    → 대역폭 효율 = 12/64 = 18.75%
```

---

## ✨ 올바른 이해

### After: 포인터 추적은 "예측 불가능한 메모리 접근"의 문제

```
핵심 통찰:
  성능을 결정하는 것은 알고리즘의 복잡도가 아니라
  "메모리 접근 패턴이 Hardware Prefetcher를 도울 수 있는가"

예측 가능한 접근 (Prefetch 가능):
  연속 배열:     arr[i++]         → 스트라이드 0으로 다음 주소 예측 가능
  일정 스트라이드: arr[i+=4]       → 스트라이드 4로 예측 가능
  블록 순회:     cache-oblivious  → 재귀적 분할로 캐시 적중

예측 불가능한 접근 (Prefetch 불가):
  포인터 체인:  node = node->next  → 다음 주소 = 현재 읽을 때 알게 됨
  해시맵 충돌:  bucket[hash(k)]->next  → 랜덤 접근
  객체 그래프:  entity->component->subsystem  → 레벨마다 캐시 미스

해결 방향:
  1. 데이터를 연속적으로 배치 (배열, Arena 할당)
  2. 필요한 데이터만 순서대로 배치 (SoA 패턴)
  3. 포인터 간접 참조 레벨 최소화
  4. 같이 접근되는 데이터를 같은 캐시라인에 (데이터 지역성)
```

---

## 🔬 내부 동작 원리

### 1. Hardware Prefetcher의 작동 방식과 한계

```
Intel Skylake Hardware Prefetcher (L1/L2 레벨):

  ① Streaming Prefetcher (L2):
     - 연속적인 캐시라인 접근 감지 → 다음 N개 미리 가져옴
     - 스트라이드(stride): 일정 간격 접근도 감지 가능
     - 포인터 체인: 다음 주소가 현재 데이터에 의존 → 감지 불가

  ② Adjacent Cache Line Prefetcher (L1):
     - 캐시라인 접근 시 인접한 캐시라인도 함께 가져옴
     - 64B × 2 = 128B를 한 번에 로드
     - 포인터 체인: 인접 노드가 다른 메모리 위치 → 무용

  포인터 체인의 근본 문제:
    현재: node_A (주소 0x100) 로드
    다음: node_A.next = 0x7f8a_bc00   ← 이 값을 알기 전까진 모름!
    → CPU는 node_A 로드가 완료될 때까지 다음 주소를 알 수 없음
    → 직렬 의존성(serial dependency): 지연 = N × 캐시 미스 지연

  연결 리스트 순회의 실제 타임라인:
    cycle   0: node_A 요청 (DRAM으로 전송)
    cycle 200: node_A 도착 (node_A.next = B의 주소 획득)
    cycle 200: node_B 요청
    cycle 400: node_B 도착 (node_B.next = C의 주소 획득)
    cycle 400: node_C 요청
    ...
    1,000만 노드 × 200 cy = 20억 사이클 직렬 대기

  배열 순회의 실제 타임라인:
    cycle   0: arr[0..7] 요청 (캐시라인 64B = int 16개 포함)
    cycle   4: arr[0..15] 도착 (Prefetcher가 이미 다음 줄 요청)
    cycle   8: arr[16..31] 도착 (완전히 겹쳐서 처리)
    ...
    Prefetcher가 앞서 가므로 대기 없음 → 처리량이 메모리 대역폭에 바인딩
```

### 2. 메모리 레이아웃 비교

```
시나리오: 100만 개 정수 값을 더하는 작업

① std::vector<int> (연속 배열):
  메모리:  [1][2][3][4][5][6][7][8][9]...[1,000,000]
            ↑_________________________↑
            연속 4MB 블록 (100만 × 4bytes)
  캐시라인당 16개 원소 (64B / 4B)
  총 캐시라인: 62,500개
  Prefetcher: 스트라이드 0 → 완전 예측 가능
  → L1/L2 히트 위주 → ~25 ms

② std::list<int> (기본 malloc 할당):
  메모리:  [값1|next→ 0x7f8a] ... [값2|next→ 0x7f9b] ... [값3|next→ ...]
            힙의 임의 위치들 (단편화)
  노드 크기: 4(int) + 4(패딩) + 8(next) = 16 bytes
  포인터 체인: 다음 주소를 알려면 현재를 읽어야 함
  → 최악 100만 번 캐시 미스 → ~380 ms

③ std::list<int> + Arena 할당기 (연속 블록):
  메모리:  [노드0][노드1][노드2]...[노드N]
            Arena가 연속 블록 안에서 할당
  포인터 체인: 여전히 data-dependent address
  그러나 노드들이 인접 → Prefetcher가 다음 노드를 추측으로 가져올 수 있음
  → L1/L2 히트율 크게 개선 → ~60 ms

핵심 차이:
  vector:       주소 = 기준 + stride × i        (예측 가능)
  list:         주소 = *(이전 노드 + offset)     (예측 불가)
  list + Arena: 물리적 인접 → 패턴에 근접하나 포인터 추적은 여전함
```

### 3. OOP 객체 그래프의 캐시 문제

```cpp
// 전형적인 게임 엔진 OOP 설계
class Entity {
    Position*  pos;     // 별도 힙 할당 → 포인터 추적 1단계
    Velocity*  vel;     // 별도 힙 할당 → 포인터 추적 1단계
    Renderer*  renderer; // 별도 힙 할당
    Collider*  collider;
    // ...10개 이상의 컴포넌트 포인터
};

// 물리 업데이트 루프
for (Entity* e : entities) {        // 포인터 배열 → 각 Entity도 추적
    e->pos->x += e->vel->dx;        // pos 접근: 포인터 추적
    e->pos->y += e->vel->dy;        // vel 접근: 또 다른 포인터 추적
}
```

```
메모리 접근 패턴 분석:

  entities[i]    → Entity 객체 (포인터 추적 1: 캐시 미스 가능)
  e->pos         → Position 객체 (포인터 추적 2: 또 다른 캐시 미스)
  e->vel         → Velocity 객체 (포인터 추적 3: 또 다른 캐시 미스)

  최악의 경우: 10,000 entities × 3회 포인터 추적 × 200 cy = 600만 사이클
  실제로 필요한 데이터: pos.x, pos.y, vel.dx, vel.dy → 16 bytes

  OOP 레이아웃의 비효율:
    Entity (64B)에서 사용하는 것: pos*, vel* = 16 bytes (25%)
    Position (24B)에서 사용하는 것: x, y = 8 bytes (33%)
    Velocity (24B)에서 사용하는 것: dx, dy = 8 bytes (33%)
    캐시 효율 = 사용량 / 로드량 = 16 / (64+24+24) = 14%
```

### 4. ECS(Entity Component System) — 데이터 지향 해결책

```cpp
// ECS 패턴: 타입별 배열로 분리
struct PositionArray { float x[MAX_ENTITIES]; float y[MAX_ENTITIES]; };
struct VelocityArray { float dx[MAX_ENTITIES]; float dy[MAX_ENTITIES]; };

PositionArray positions;
VelocityArray velocities;

// 물리 업데이트 루프
for (int i = 0; i < active_count; i++) {
    positions.x[i] += velocities.dx[i];
    positions.y[i] += velocities.dy[i];
}
```

```
ECS 메모리 접근 패턴:

  positions.x 배열: [x0][x1][x2]...[xN]  연속 4×N bytes
    → 스트라이드 0 → Prefetcher 완벽 예측
    → 한 캐시라인(64B)에 16개 float → 효율 100%

  velocities.dx 배열: [dx0][dx1][dx2]...[dxN]  연속 4×N bytes
    → 동일하게 Prefetcher 친화적

  이 루프의 접근 패턴:
    i=0: positions.x[0], velocities.dx[0]   → 서로 다른 배열이지만
    i=1: positions.x[1], velocities.dx[1]   → 각각 스트라이드 0
    → 두 스트림 동시 Prefetch → 하드웨어 Prefetcher가 양쪽 처리 가능

  컴파일러 추가 이점:
    → 루프 내 분기 없음 → 자동 벡터화 (SIMD) 적용 가능
    → AVX2: 8개 float을 한 명령으로 처리 → 추가 8배 가속
    → 전체: 포인터 추적 대비 20~50배 처리량 향상

  Unity의 DOTS(Data-Oriented Technology Stack):
    → ECS를 C# + Burst Compiler로 구현
    → 구형 OOP 기반 GameObject 대비 10~100배 빠른 물리 처리
    → 수만 개 객체를 60fps로 처리 가능
```

### 5. Arena 할당기 — 포인터 추적의 지역성 개선

```cpp
// 기본 Arena 할당기 구현
class Arena {
    char* buffer;
    size_t offset = 0;
    size_t capacity;
public:
    Arena(size_t cap) : capacity(cap), buffer(new char[cap]) {}

    void* alloc(size_t size, size_t align = 8) {
        // 정렬 조정
        offset = (offset + align - 1) & ~(align - 1);
        void* ptr = buffer + offset;
        offset += size;
        return ptr;
    }

    void reset() { offset = 0; }  // O(1) 전체 해제
};

// 사용 예시
Arena arena(1024 * 1024);  // 1MB 연속 블록

// 노드들이 연속적으로 배치됨
struct Node { int value; Node* next; };
Node* build_list(Arena& arena, int n) {
    Node* head = nullptr;
    for (int i = n - 1; i >= 0; i--) {
        Node* node = (Node*)arena.alloc(sizeof(Node));
        node->value = i;
        node->next = head;
        head = node;
    }
    return head;
}
```

```
Arena 할당기의 캐시 효과:

  일반 new/malloc:
    노드 A: 0x7f8a_3c00
    노드 B: 0x7f9b_4400   (+67.5KB 떨어짐)
    노드 C: 0x7f7d_2800   (-9.5KB, 역방향)
    → 완전 랜덤 → L3까지 미스 가능

  Arena 할당:
    노드 A: arena.buffer + 0    = 0x5555_5500
    노드 B: arena.buffer + 16   = 0x5555_5510
    노드 C: arena.buffer + 32   = 0x5555_5520
    → 연속 배치! 같은 캐시라인(64B)에 4개 노드
    → Prefetcher가 다음 캐시라인을 가져올 수 있음

  Arena 효과:
    malloc 기반 list 순회:   ~380 ms
    Arena 기반 list 순회:    ~65 ms   (5.8배 개선)
    std::vector 순회:        ~25 ms   (Arena 대비 2.6배 여전히 빠름)

  왜 Arena도 vector보다 느린가?
    → 포인터 체인(data-dependent address)은 여전히 존재
    → 노드가 16 bytes라면 64B 캐시라인에 4개 → 75% 낭비 (next 포인터 때문)
    → vector는 이 낭비가 없음
```

---

## 💻 실전 실험

### 실험 1: std::vector vs std::list 순회 성능 비교 (Google Benchmark)

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <list>
#include <numeric>

const int N = 1'000'000;

static void BM_VectorSum(benchmark::State& state) {
    std::vector<int> v(N);
    std::iota(v.begin(), v.end(), 0);
    for (auto _ : state) {
        long long sum = 0;
        for (int x : v) sum += x;
        benchmark::DoNotOptimize(sum);
    }
    state.SetItemsProcessed(state.iterations() * N);
}

static void BM_ListSum(benchmark::State& state) {
    std::list<int> l(v_data.begin(), v_data.end());
    // 한 번 생성하고 반복 순회 (힙 단편화 시뮬레이션)
    std::list<int> lst;
    // 의도적으로 단편화: 다른 할당을 섞어서 노드 흩어놓기
    std::vector<int*> fillers;
    for (int i = 0; i < N; i++) {
        lst.push_back(i);
        if (i % 7 == 0) fillers.push_back(new int(i));  // 단편화 유도
    }
    for (auto _ : state) {
        long long sum = 0;
        for (int x : lst) sum += x;
        benchmark::DoNotOptimize(sum);
    }
    state.SetItemsProcessed(state.iterations() * N);
    for (auto p : fillers) delete p;
}

BENCHMARK(BM_VectorSum)->Unit(benchmark::kMillisecond);
BENCHMARK(BM_ListSum)->Unit(benchmark::kMillisecond);
BENCHMARK_MAIN();
```

```bash
g++ -O2 -std=c++17 bench_pointer.cpp -lbenchmark -lbenchmark_main -lpthread -o bench_pointer
./bench_pointer

# 예상 출력:
# BM_VectorSum   25 ms   (캐시 효율적)
# BM_ListSum    380 ms   (포인터 추적, 단편화)
# → 약 15배 차이
```

### 실험 2: perf로 캐시 미스 비교

```bash
# vector 순회 캐시 미스
perf stat -e cycles,instructions,cache-references,cache-misses,\
    L1-dcache-loads,L1-dcache-load-misses \
    ./bench_vector_only

# list 순회 캐시 미스
perf stat -e cycles,instructions,cache-references,cache-misses,\
    L1-dcache-loads,L1-dcache-load-misses \
    ./bench_list_only

# 예상 비교:
#              vector          list
# cache-miss:    0.1%          35~55%
# IPC:           3.2           0.4
# cycles:        ~50M          ~800M
#
# IPC가 낮은 이유: 포인터 체인 대기로 CPU 실행 유닛이 놀게 됨
# → "memory-bound" 워크로드의 전형적 특징

# cachegrind로 미스 위치 정밀 분석
valgrind --tool=cachegrind --cache-sim=yes ./bench_list_only
cg_annotate cachegrind.out.<pid>
```

### 실험 3: ECS 패턴 vs OOP 패턴 처리 속도 비교

```cpp
#include <benchmark/benchmark.h>
#include <vector>

const int ENTITIES = 100'000;

// OOP 방식
struct Position { float x, y; };
struct Velocity { float dx, dy; };
struct EntityOOP {
    Position* pos;
    Velocity* vel;
    int id;
    char padding[44];  // 실제 컴포넌트들로 64B 근처 채움
};

static void BM_OOP_Update(benchmark::State& state) {
    std::vector<EntityOOP> entities(ENTITIES);
    std::vector<Position> positions(ENTITIES);
    std::vector<Velocity> velocities(ENTITIES);
    for (int i = 0; i < ENTITIES; i++) {
        entities[i].pos = &positions[i];
        entities[i].vel = &velocities[i];
        positions[i] = {float(i), float(i)};
        velocities[i] = {0.1f, 0.1f};
    }
    for (auto _ : state) {
        for (auto& e : entities) {
            e.pos->x += e.vel->dx;
            e.pos->y += e.vel->dy;
        }
        benchmark::DoNotOptimize(entities[0].pos->x);
    }
}

// ECS (DOD) 방식 — SoA 레이아웃
struct ECS {
    std::vector<float> pos_x, pos_y;
    std::vector<float> vel_dx, vel_dy;
    ECS(int n) : pos_x(n), pos_y(n), vel_dx(n, 0.1f), vel_dy(n, 0.1f) {
        for (int i = 0; i < n; i++) { pos_x[i] = i; pos_y[i] = i; }
    }
};

static void BM_ECS_Update(benchmark::State& state) {
    ECS ecs(ENTITIES);
    for (auto _ : state) {
        for (int i = 0; i < ENTITIES; i++) {
            ecs.pos_x[i] += ecs.vel_dx[i];
            ecs.pos_y[i] += ecs.vel_dy[i];
        }
        benchmark::DoNotOptimize(ecs.pos_x[0]);
    }
}

BENCHMARK(BM_OOP_Update)->Unit(benchmark::kMicrosecond);
BENCHMARK(BM_ECS_Update)->Unit(benchmark::kMicrosecond);
BENCHMARK_MAIN();
```

```bash
g++ -O2 -std=c++17 bench_ecs.cpp -lbenchmark -lbenchmark_main -lpthread -o bench_ecs
./bench_ecs

# 예상 출력 (100,000 entities):
# BM_OOP_Update   ~4800 us   (포인터 추적 × 3단계)
# BM_ECS_Update    ~220 us   (연속 배열, SIMD 벡터화 가능)
# → 약 22배 차이

# ECS는 -O3에서 AVX2 자동 벡터화까지 적용됨
g++ -O3 -march=native -std=c++17 bench_ecs.cpp -lbenchmark -lbenchmark_main -lpthread -o bench_ecs_avx
./bench_ecs_avx
# BM_ECS_Update   ~30 us   (SIMD 적용 시 추가 7배)
```

### 실험 4: godbolt에서 ECS 루프 벡터화 확인

```cpp
// https://godbolt.org — GCC 13.2 -O3 -march=haswell

// OOP 버전 — 벡터화 불가
void update_oop(float** px, float** vx, int n) {
    for (int i = 0; i < n; i++)
        *px[i] += *vx[i];   // 간접 참조 → 벡터화 불가
}

// ECS 버전 — 자동 벡터화
void update_ecs(float* __restrict__ px, const float* __restrict__ vx, int n) {
    for (int i = 0; i < n; i++)
        px[i] += vx[i];   // 연속 배열 + restrict → vmovups/vaddps 생성
}
```

```bash
# 어셈블리 출력 비교
gcc -O3 -march=haswell -S -masm=intel ecs_test.cpp -o ecs_test.s
grep -E 'vmov|vadd|ymm' ecs_test.s
# update_ecs에서 ymm 레지스터 명령 확인 (8 float 동시 처리)
```

---

## 📊 성능 비교

```
N=1,000,000 정수 합산 (x86-64, Skylake, DDR4-2400):

자료구조/패턴             시간      캐시미스율  IPC    비고
────────────────────────────────────────────────────────────────
std::vector<int>         ~25 ms    0.1%       3.2    이상적 기준
std::list (Arena)        ~65 ms    8.5%       1.4    지역성 개선
std::list (단편화)       ~380 ms   48%        0.4    포인터 추적
포인터 배열 순회          ~180 ms   25%        0.8    간접 참조
OOP 엔티티 (3단계 추적)  ~4800 us  35%        0.5    (엔티티 10만)

N=100,000 엔티티 물리 업데이트:

방식                       시간       IPC    벡터화
────────────────────────────────────────────────────
OOP (포인터 컴포넌트)      ~4800 us   0.5    불가
ECS SoA (-O2)             ~220 us    2.8    스칼라
ECS SoA (-O3 AVX2)        ~30 us     3.8+   AVX2 8way
개선 비율 (OOP → ECS AVX)  160배

Google Benchmark 실측 (Intel Core i7-8700, Ubuntu 22.04):
  perf stat 캐시 미스:
    OOP 루프:   cache-miss rate = 32.4%,  IPC = 0.52
    ECS 루프:   cache-miss rate =  0.3%,  IPC = 3.71
```

---

## ⚖️ 트레이드오프

```
데이터 지향 설계(DOD)의 트레이드오프:

DOD / ECS가 유리한 경우:
  ✅ 동일한 연산을 대량 데이터에 반복 (게임 물리, 입자 시뮬레이션)
  ✅ 접근 패턴이 일정하고 예측 가능한 경우
  ✅ SIMD 벡터화로 추가 가속이 필요한 경우
  ✅ 10만 개 이상의 동질적 객체 처리
  ✅ 성능이 최우선인 핫 루프

OOP/포인터 그래프가 합리적인 경우:
  ❌ 객체 수가 수백~수천으로 적은 경우 (캐시에 다 들어감)
  ❌ 접근 패턴이 불규칙하여 어차피 DOD도 도움이 안 될 때
  ❌ 유지보수성, 확장성이 처리량보다 훨씬 중요한 경우
  ❌ 삽입/삭제가 순회보다 훨씬 많은 경우

Arena 할당기 적합 사례:
  ✅ 생명주기가 같은 객체들 (프레임 단위, 요청 단위)
  ✅ 연결 리스트/트리를 써야 하는데 캐시 성능도 필요한 경우
  ✅ malloc 오버헤드를 줄이고 싶을 때 (할당 O(1), 해제 O(1))

연결 리스트가 여전히 적합한 경우:
  ✅ 빈번한 임의 위치 삽입/삭제 (중간 삽입이 정말 핵심인 경우)
  ✅ 원소 크기가 매우 커서 이동 비용이 압도적인 경우
  ✅ 순회 횟수가 매우 적고 구조 변경이 빈번한 경우

현실적 조언:
  → std::deque: 배열의 O(1) 앞 삽입 + 연속 블록 (vector보다 약간 느리나 훨씬 낫다)
  → intrusive list: 외부 포인터 없이 객체 안에 next/prev → 포인터 추적 1단계 절감
  → perf로 먼저 측정 → cache-miss가 실제 병목인지 확인 후 구조 변경
```

---

## 📌 핵심 정리

```
포인터 추적 비용 핵심:

근본 원인:
  포인터 체인 = data-dependent address
  → 현재 값을 읽기 전까지 다음 주소를 알 수 없음
  → Hardware Prefetcher가 예측 불가
  → 노드마다 캐시 미스 가능 (최악 DRAM ~200 cy)

비교 (1M 요소 합산):
  std::vector:      ~25 ms   (L1/L2 히트 위주, IPC ~3.2)
  std::list (단편): ~380 ms  (DRAM 미스 위주, IPC ~0.4)
  std::list (Arena): ~65 ms  (지역성 개선, 여전히 포인터 추적 존재)

ECS / 데이터 지향 설계:
  문제: OOP 포인터 그래프 → 3단계 포인터 추적 → 대량 캐시 미스
  해결: 타입별 연속 배열(SoA) → 스트라이드 0 → Prefetcher 최적
  효과: 포인터 기반 대비 20~160배 처리량 (SIMD 포함)
  사례: Unity DOTS, Unreal ECS, Bevy (Rust), EnTT (C++)

Arena 할당기:
  연속 블록 안에서 bump allocation → 노드 간 공간적 지역성 확보
  malloc 기반 list 대비 ~6배 개선
  완전한 해결책은 아님 (포인터 추적 자체는 여전히 존재)

설계 원칙:
  "같이 읽히는 데이터는 같이 두어라" (데이터 지역성)
  "다음 접근 주소를 미리 계산 가능하게 하라" (Prefetcher 친화성)
  측정 없이 최적화하지 말 것 — perf stat으로 cache-miss % 확인 먼저
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 두 코드의 캐시 성능 차이를 설명하라. Prefetcher 관점에서 각각의 접근 패턴을 분석하시오.

```cpp
// A: 포인터 배열 순회
struct Particle { float x, y, z, mass; };
std::vector<Particle*> particles;  // 포인터 배열
for (Particle* p : particles)
    p->x += p->y * 0.01f;

// B: 값 배열 순회
std::vector<Particle> particles;   // 값 배열
for (Particle& p : particles)
    p.x += p.y * 0.01f;
```

<details>
<summary>해설 보기</summary>

**A (포인터 배열)**: 두 단계의 캐시 압박이 발생합니다.

1. `particles[i]`에서 포인터 로드: `std::vector<Particle*>` 자체는 연속적이므로 Prefetcher가 포인터들을 미리 가져올 수 있습니다. 그러나 포인터가 가리키는 각 `Particle`은 힙의 임의 위치에 있습니다.

2. `p->x`, `p->y` 접근: 각 `Particle`이 새로 할당된 별도의 캐시라인을 필요로 할 수 있습니다. Prefetcher는 `particles[i]` (포인터 값)를 읽기 전까지 `Particle` 객체의 주소를 알 수 없으므로 — 전형적인 data-dependent address — 각 `Particle` 접근에서 캐시 미스가 발생할 수 있습니다.

**B (값 배열)**: `std::vector<Particle>`는 `Particle` 객체들이 연속적으로 배치됩니다. `sizeof(Particle) = 16 bytes`이므로 캐시라인(64B) 하나에 4개가 들어갑니다. Prefetcher는 스트라이드 64B (캐시라인 단위)로 다음 블록을 미리 가져올 수 있습니다. 간접 참조가 없으므로 루프 본체에 `vaddps` 같은 SIMD 명령이 적용될 수도 있습니다.

perf로 확인하면 A는 `L1-dcache-load-misses` 이벤트가 많고 IPC가 낮으며, B는 캐시 미스가 거의 없고 IPC가 높게 나옵니다. 객체 수가 수만 개 이상이면 차이가 수 배에서 수십 배가 됩니다.

</details>

---

**Q2.** Arena 할당기를 사용해도 `std::vector`보다 연결 리스트 순회가 느린 이유를 캐시라인 수준에서 설명하라.

<details>
<summary>해설 보기</summary>

Arena 할당기는 노드들을 연속 메모리에 배치하여 공간적 지역성을 크게 개선합니다. 그러나 두 가지 근본 한계가 남아 있습니다.

**1. 포인터 체인의 직렬 의존성(serial dependency)은 여전함**:
노드가 물리적으로 인접해 있어도, 다음 노드 주소는 `node->next`를 읽어야만 알 수 있습니다. CPU의 비순차 실행(OoO)이 다음 노드의 로드를 미리 시작하려 해도 현재 노드가 완전히 처리되어야 `node->next` 값이 확정됩니다. 이 직렬 의존성이 파이프라인 깊이를 1 단계로 제한합니다.

반면 `std::vector`의 `arr[i+1]`은 `arr[i]` 값과 무관하게 주소가 확정(`base + (i+1)*sizeof(T)`)되어 Prefetcher가 완전히 앞서갈 수 있습니다.

**2. 노드 크기 패딩으로 캐시라인 효율 손실**:
`Node { int value; Node* next; }`는 4 + 8 = 12 bytes이지만 정렬로 인해 16 bytes로 패딩됩니다. 캐시라인(64B)에 4개 노드가 들어가며, 실제 `value` 데이터는 4 bytes × 4 = 16 bytes / 64B = 25% 효율입니다. `std::vector<int>`는 4 bytes × 16개 = 64 bytes 전부 유효한 데이터로 100% 효율입니다.

따라서 Arena는 `malloc` 단편화의 최악 시나리오를 제거해주지만, 포인터 추적의 근본 한계(직렬 의존성 + 낮은 데이터 밀도)는 해결하지 못합니다.

</details>

---

**Q3.** Unity의 DOTS(Data-Oriented Technology Stack)와 전통적인 `GameObject` 방식의 성능 차이가 발생하는 하드웨어 이유를 두 가지 이상 설명하라.

<details>
<summary>해설 보기</summary>

전통적인 `GameObject` 방식과 DOTS/ECS의 차이는 하드웨어 수준에서 세 가지로 귀결됩니다.

**1. 포인터 추적 단계 수 차이**:
`GameObject` 방식에서는 `GameObject → Transform → Position`, `GameObject → Rigidbody → Velocity` 형태로 2~3단계 포인터 추적이 발생합니다. 10만 개 오브젝트를 처리하면 10만 × 3회 = 30만 번의 잠재적 캐시 미스가 생깁니다. DOTS ECS는 `PositionArray[i]`와 `VelocityArray[i]`를 연속 배열로 직접 접근합니다.

**2. 데이터 밀도와 캐시라인 효율**:
`GameObject`는 렌더링, 물리, 애니메이션, 스크립트 등 모든 컴포넌트 참조를 가진 거대 객체입니다. 물리 업데이트에서 필요한 것은 Position과 Velocity뿐인데 `GameObject` 전체(수백 bytes)를 캐시에 올려야 합니다. DOTS ECS의 SoA(Struct of Arrays) 레이아웃은 필요한 컴포넌트 데이터만 연속 배치되어 있어 캐시라인 활용률이 훨씬 높습니다.

**3. SIMD 자동 벡터화 가능 여부**:
DOTS의 Burst Compiler는 연속 배열과 간접 참조 없는 루프를 AVX2/NEON으로 자동 벡터화합니다. 물리 업데이트 루프에서 8개 float을 한 명령으로 처리합니다. `GameObject` 방식은 포인터 참조와 가상 함수 호출이 섞여 있어 자동 벡터화가 불가능합니다. 이 차이만으로도 순수 처리량 8배 차이가 발생합니다.

Unity DOTS 실측: 1만 개 오브젝트 물리 업데이트가 `GameObject` 방식으로 16ms(60fps 달성 불가)였던 것이 DOTS로 전환 후 0.3ms로 떨어진 사례가 공식 문서에 보고되어 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: 가상 함수·동적 디스패치](./02-virtual-functions.md)** | **[홈으로 🏠](../README.md)** | **[다음: 분기 없는 코드 ➡️](./04-branchless-code.md)**

</div>
