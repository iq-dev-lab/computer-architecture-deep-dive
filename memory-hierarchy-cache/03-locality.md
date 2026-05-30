# 지역성(Locality) — 시간/공간이 성능을 만든다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 시간 지역성(Temporal Locality)과 공간 지역성(Spatial Locality)의 정확한 차이는?
- `int[1024][1024]` 행 우선(row-major) 순회가 열 우선(column-major) 순회보다 10배 빠른 이유는?
- 연결 리스트가 배열 대비 느린 이유가 단순히 "메모리가 비연속적이어서"가 아닌 정확한 원인은?
- 캐시라인 64B 안에 int 16개가 들어간다는 사실이 루프 성능에 어떻게 직결되는가?
- prefetch(프리페치)는 어떤 접근 패턴에서 효과적이고 어떤 패턴에서 무력화되는가?
- 지역성이 나쁜 코드를 리팩토링할 때 어떤 순서로 접근해야 하는가?

---

## 🔍 왜 이 개념이 중요한가

### "O(N²)인데 왜 이 구현이 다른 것보다 빠르지?"

```
행렬 전치(Transpose) — 같은 알고리즘, 다른 지역성:

// 방법 A: 나이브 전치 (열 우선 쓰기)
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        B[j][i] = A[i][j];  // B에 열 우선으로 씀

// 방법 B: 블록 전치 (지역성 개선)
#define BLOCK 32
for (int ii = 0; ii < N; ii += BLOCK)
    for (int jj = 0; jj < N; jj += BLOCK)
        for (int i = ii; i < ii+BLOCK; i++)
            for (int j = jj; j < jj+BLOCK; j++)
                B[j][i] = A[i][j];

N=1024, double 기준 (8MB 행렬):
  방법 A: ~800ms   ← B 배열에 캐시 미스 폭발
  방법 B: ~100ms   ← 32×32 블록이 L1에 올라와서 재사용

8배 차이의 원인은 알고리즘 차이가 아님
전부 지역성(Locality)의 차이

지역성을 이해하면:
  왜 행렬 순회 순서가 중요한지
  왜 ECS(Entity Component System)이 게임 엔진 표준인지
  왜 데이터베이스가 row 대신 columnar 저장을 선택하는지
  — 모두 지역성 최적화의 구체적인 적용이다
```

---

## 😱 잘못된 이해

### Before: "캐시는 자동으로 관리되니 접근 순서는 성능에 관계없다"

```
잘못된 이해:
  캐시는 OS/하드웨어가 알아서 관리한다
  내가 어떤 순서로 접근해도 자주 쓴 데이터는 캐시에 남는다
  → 접근 순서를 신경 쓸 필요 없다

실제로 놓치는 것:
  캐시는 하드웨어가 관리하지만, 히트 여부는 접근 패턴이 결정한다

  예:
  int A[1024][1024];  // 4MB 배열

  // 순서 1: 행 우선 접근 (C/C++ 배열 레이아웃과 일치)
  for (int i = 0; i < 1024; i++)
      for (int j = 0; j < 1024; j++)
          sum += A[i][j];

  // 순서 2: 열 우선 접근
  for (int j = 0; j < 1024; j++)
      for (int i = 0; i < 1024; i++)
          sum += A[i][j];

  두 코드 모두 같은 1M번 접근
  결과도 동일
  하지만 실행 시간: 10배 이상 차이

  캐시는 "자동으로" 히트율을 올려주지 않는다
  접근 패턴이 캐시와 맞지 않으면 캐시가 있어도 소용없다
```

---

## ✨ 올바른 이해

### After: 지역성은 캐시 히트율을 결정하는 핵심 변수

```
두 가지 지역성:

시간 지역성 (Temporal Locality):
  최근에 접근한 주소를 곧 다시 접근할 가능성이 높다
  
  예: 루프 변수, 자주 호출되는 함수, 핫 데이터
  for (int i = 0; i < N; i++)
      sum += a[i];  // sum과 i는 반복 접근 → 레지스터/L1에 상주

  캐시가 도움이 되려면: 재접근 전에 캐시에서 퇴출되지 않아야 함
  → 워킹셋이 캐시 크기 안에 들어와야 함

공간 지역성 (Spatial Locality):
  특정 주소 접근 후 인접한 주소를 곧 접근할 가능성이 높다

  예: 배열 순차 접근, 구조체 멤버 순차 읽기
  sum += a[0];  // a[0] 미스 → 캐시라인 64B 로드
                // a[1]~a[15]가 자동으로 캐시에 올라옴
  sum += a[1];  // 캐시 히트 (이미 로드됨)
  sum += a[2];  // 캐시 히트
  ...
  sum += a[15]; // 캐시 히트 (총 1번 미스에 16번 접근!)

캐시라인 64B의 위력:
  int (4B) × 16 = 64B = 캐시라인 1개
  순차 접근 시 미스 1번당 int 16개 확보
  미스율 = 1/16 = 6.25% (나머지 93.75%는 공간 지역성 덕분에 히트)
```

---

## 🔬 내부 동작 원리

### 1. C/C++ 2차원 배열의 메모리 레이아웃

```c
// int A[4][4] 선언
// C는 행 우선(Row-Major) 저장
//
// 메모리 레이아웃 (주소 증가 방향 →):
//
//  A[0][0] A[0][1] A[0][2] A[0][3]  A[1][0] A[1][1] ...  A[3][3]
//  |────── 행 0 (캐시라인 1) ──────|  |────── 행 1 ──────|
//
// 주소 공식: &A[i][j] = base + (i * 4 + j) * sizeof(int)
//
// 행 우선 접근 A[i][j] (j 먼저 변화):
//   A[0][0] → A[0][1] → A[0][2] → A[0][3] ← 같은 캐시라인 64B
//   → 미스 1번에 4개 int (A[0][0]~A[0][3]) 로드
//   (64B 라인이면 4 × 4 = 16개 int, 여기선 4열이니 4개)
//
// 열 우선 접근 A[i][j] (i 먼저 변화):
//   A[0][0] → A[1][0] → A[2][0] → A[3][0]
//   각각 서로 다른 행 = 서로 다른 캐시라인 = 매번 미스!

#include <stdio.h>
void show_layout() {
    int A[4][4];
    int *base = &A[0][0];
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            printf("A[%d][%d] = base+%td\n", i, j, &A[i][j] - base);
}
// 출력:
// A[0][0] = base+0   ─┐
// A[0][1] = base+1    │ 연속! 캐시라인 1개
// A[0][2] = base+2    │
// A[0][3] = base+3   ─┘
// A[1][0] = base+4   ─┐ 다음 캐시라인 (1024열 배열이면 64B 떨어짐)
// ...
```

### 2. 행 우선 vs 열 우선 미스 카운팅

```
int A[1024][1024];  // 4MB 배열

캐시라인 = 64B = int 16개
1024열이므로 각 행 = 1024 * 4B = 4096B = 64개 캐시라인

행 우선 순회 (j가 내부 루프):
  A[0][0] ~ A[0][15]: 미스 1번 (64B 로드)
  A[0][16] ~ A[0][31]: 미스 1번
  ...
  A[0][0]~A[0][1023]: 64번 미스 (1024/16)
  A[1][0]~A[1][1023]: 64번 미스
  ...
  전체 미스 = 1024 × 64 = 65,536번

열 우선 순회 (i가 내부 루프):
  A[0][0]: 미스 1번 (64B 로드, A[0][0]~A[0][15] 로드)
  A[1][0]: 미스 1번 (다른 행 → 다른 캐시라인, 4096B 떨어짐)
  A[2][0]: 미스 1번 (또 다른 행)
  ...
  A[1023][0]: 미스 1번
  → 첫 번째 j=0에서 이미 1024번 미스
  A[0][1]: L1에서 퇴출 가능성 (1024번 접근 동안 캐시 통과)
  ...
  전체 미스 = 1024 × 1024 = 1,048,576번

비율:
  행 우선: 65,536 미스
  열 우선: 1,048,576 미스
  16배 차이 (캐시라인 안 int 16개 = 이론적 16배)

실제 측정 (N=1024, int 배열):
  행 우선: ~5ms
  열 우선: ~80ms
  → ~16배 차이 (이론과 일치)
```

### 3. 연결 리스트 vs 배열 — 포인터 추적의 비용

```c
// 연결 리스트 순회
struct Node {
    int value;
    struct Node* next;
};

long long list_sum(struct Node* head) {
    long long sum = 0;
    for (struct Node* n = head; n != NULL; n = n->next)
        sum += n->value;
    return sum;
}

// 배열 순회
long long array_sum(int* a, int n) {
    long long sum = 0;
    for (int i = 0; i < n; i++)
        sum += a[i];
    return sum;
}

// 왜 연결 리스트가 느린가?

// 배열의 메모리:
// [4B][4B][4B]...[4B]   ← 연속 배치
// 캐시라인 1번 미스에 16개 int 로드
// 다음 접근: 캐시 히트 (이미 로드됨)

// 연결 리스트의 메모리 (heap 할당 후):
// Node0: 주소 0x1000 (value=1, next=0x3A80)  ← 64B 캐시라인
// Node1: 주소 0x3A80 (value=2, next=0x21C0)  ← 다른 캐시라인!
// Node2: 주소 0x21C0 (value=3, next=0x4F00)  ← 또 다른 캐시라인!
// 각 Node가 임의의 heap 주소에 할당됨
// → 순회 = 포인터 추적 = 매번 새 캐시라인 접근 = 매번 미스 가능

// 핵심: Node 크기 = sizeof(int) + sizeof(pointer) = 4 + 8 = 12B
// 캐시라인 64B 안에 최대 5개 Node가 연속으로 들어갈 수 있지만
// heap 할당자가 인접하게 배치한다는 보장 없음
// → 대부분의 경우 Node 1개 = 캐시미스 1번

// N=1,000,000 요소:
// 배열: ~65,536번 미스 (캐시라인 1번에 16개)
// 연결 리스트: ~1,000,000번 미스 (각 Node마다 미스 가능)
// → ~15배 미스 차이 → ~10~15배 속도 차이
```

### 4. 하드웨어 프리페치와 지역성

```
프리페치(Prefetch): CPU가 미스가 발생하기 전에 미리 데이터를 로드하는 기능

하드웨어 프리페치 동작 원리:
  순차 패턴 감지:
    a[0] 미스 → a[0] 로드 시작
    a[64] 미스 → a[64] 로드 시작
    → 패턴 인식: 64B씩 전진
    → a[128], a[192] 미리 로드 시작 (프리페치)
    → a[128] 접근 시: 이미 로드 완료 → 히트!

  스트라이드(일정 간격) 패턴:
    a[0], a[8], a[16] 접근 → 8 간격 인식 → 미리 로드
    (stride prefetcher가 지원하는 정규 패턴)

  랜덤 패턴에서 무력화:
    a[rand() % N] → 패턴 없음 → 프리페치 불가
    → 연결 리스트: 각 next 포인터를 읽기 전까지 다음 주소 모름
    → 하드웨어 프리페치로 이점 없음

소프트웨어 프리페치:
  // N번 뒤를 미리 요청 (GCC 내장)
  __builtin_prefetch(&a[i + 16], 0, 1);
  // 인자: 주소, rw(0=읽기), locality(1=L2에 올림)

  규칙적 패턴에서 유효:
  for (int i = 0; i < N; i++) {
      __builtin_prefetch(&a[i + 16], 0, 1);  // 16 요소 앞 미리 요청
      sum += a[i];
  }
  → 미스 비용이 숨겨짐 → 처리량 증가

  연결 리스트에는 사실상 무효:
  for (Node* n = head; n; n = n->next) {
      if (n->next) __builtin_prefetch(n->next, 0, 1);  // 1칸만 봄
      sum += n->value;
  }
  → 1 hop 미리 요청은 메모리 지연 숨기기에 불충분
  → 5~10 hop 미리 요청이 이상적이지만 구현 복잡
```

### 5. 지역성 개선 전략 — 루프 재구성

```c
// 예: 행렬-벡터 곱 (N=1024)
// A[N][N], x[N], y[N]
// y[i] = sum_j A[i][j] * x[j]

// 나이브 구현 (A는 행 우선 OK, x는 반복 접근 OK)
for (int i = 0; i < N; i++) {
    y[i] = 0;
    for (int j = 0; j < N; j++) {
        y[i] += A[i][j] * x[j];  // A: 순차 ✅, x: 반복 ✅
    }
}
// A는 행 우선 접근 → 공간 지역성 ✅
// x[0..N-1]은 N번 재사용 → 시간 지역성 ✅ (x가 캐시에 남을 만큼 작으면)

// 전치 행렬 버전 (공간 지역성 명시적 개선)
// AT[j][i] = A[i][j] — 미리 전치한 버전
for (int i = 0; i < N; i++) {
    y[i] = 0;
    for (int j = 0; j < N; j++) {
        y[i] += AT[j][i] * x[j];  // AT에 열 우선 접근 → 미스 증가 ❌
    }
}
// 실제로는 더 나쁨: 전치 후 열 접근이 원래보다 나쁠 수 있음

// 캐시 블로킹 (Tiling) — 지역성 최대화
#define B 64  // 블록 크기 (L1 맞춤: 64*64*4 = 16KB < 32KB)
for (int ii = 0; ii < N; ii += B)
    for (int jj = 0; jj < N; jj += B)
        for (int i = ii; i < ii+B && i < N; i++)
            for (int j = jj; j < jj+B && j < N; j++)
                y[i] += A[i][j] * x[j];
// B×B 블록이 L1에 올라와 있는 동안 완전 재사용
// 시간 지역성 극대화
```

---

## 💻 실전 실험

### 실험 1: 행 우선 vs 열 우선 순회 — perf로 미스율 측정

```bash
# row_col.c 컴파일
gcc -O2 -o row_col row_col.c

# 행 우선 순회 미스율
perf stat -e cycles,instructions,L1-dcache-loads,L1-dcache-load-misses,\
LLC-loads,LLC-load-misses ./row_col row

# 예상 출력 (행 우선):
#   300,000,000   cycles
#   280,000,000   instructions       #  0.93  insn per cycle
#  65,600,000     L1-dcache-loads
#     4,096,000   L1-dcache-load-misses  # 6.2% ← 이론값 1/16 = 6.25%
#     4,096,000   LLC-loads
#        10,000   LLC-load-misses    # 0.24% ← L3에서 대부분 히트

# 열 우선 순회 미스율
perf stat -e cycles,instructions,L1-dcache-loads,L1-dcache-load-misses,\
LLC-loads,LLC-load-misses ./row_col col

# 예상 출력 (열 우선):
#  5,000,000,000  cycles
#   280,000,000   instructions       #  0.06  insn per cycle ← IPC 폭락!
#  65,600,000     L1-dcache-loads
#  65,000,000     L1-dcache-load-misses  # 99.1% ← 거의 모든 접근이 미스!
#  65,000,000     LLC-loads
#  64,000,000     LLC-load-misses    # 98.4% ← DRAM 직행
```

```c
// row_col.c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#define N 1024
int A[N][N];

int main(int argc, char* argv[]) {
    // 초기화
    for (int i = 0; i < N; i++)
        for (int j = 0; j < N; j++)
            A[i][j] = i * N + j;

    struct timespec t0, t1;
    volatile long long sum = 0;

    if (argc > 1 && argv[1][0] == 'r') {
        // 행 우선
        clock_gettime(CLOCK_MONOTONIC, &t0);
        for (int rep = 0; rep < 64; rep++)
            for (int i = 0; i < N; i++)
                for (int j = 0; j < N; j++)
                    sum += A[i][j];
        clock_gettime(CLOCK_MONOTONIC, &t1);
    } else {
        // 열 우선
        clock_gettime(CLOCK_MONOTONIC, &t0);
        for (int rep = 0; rep < 64; rep++)
            for (int j = 0; j < N; j++)
                for (int i = 0; i < N; i++)
                    sum += A[i][j];
        clock_gettime(CLOCK_MONOTONIC, &t1);
    }

    long ms = (t1.tv_sec - t0.tv_sec) * 1000 +
              (t1.tv_nsec - t0.tv_nsec) / 1000000;
    printf("%s: %ld ms (sum=%lld)\n",
           argc>1 ? argv[1] : "col", ms, sum);
    return 0;
}
```

### 실험 2: 연결 리스트 vs 배열 미스율 비교

```c
// list_vs_array.c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define N (1 << 20)  // 1M 요소

typedef struct Node {
    int value;
    struct Node* next;
} Node;

int main() {
    // 배열 초기화
    int* arr = (int*)malloc(N * sizeof(int));
    for (int i = 0; i < N; i++) arr[i] = i;

    // 연결 리스트 초기화 (순차 할당 — 최선의 경우)
    Node* nodes = (Node*)malloc(N * sizeof(Node));
    for (int i = 0; i < N; i++) {
        nodes[i].value = i;
        nodes[i].next = (i + 1 < N) ? &nodes[i+1] : NULL;
    }
    Node* head_seq = &nodes[0];

    // 연결 리스트 초기화 (랜덤 할당 — 현실적 경우)
    // 실제로는 각각 malloc으로 흩어진 위치에 할당된다고 가정
    // 여기서는 셔플로 순서를 랜덤화
    int* order = (int*)malloc(N * sizeof(int));
    for (int i = 0; i < N; i++) order[i] = i;
    for (int i = N-1; i > 0; i--) {
        int j = rand() % (i+1);
        int tmp = order[i]; order[i] = order[j]; order[j] = tmp;
    }
    Node* nodes_rand = (Node*)malloc(N * sizeof(Node));
    for (int i = 0; i < N; i++) nodes_rand[i].value = order[i];
    for (int i = 0; i < N-1; i++) nodes_rand[i].next = &nodes_rand[order[i+1]];
    nodes_rand[N-1].next = NULL;

    struct timespec t0, t1;
    volatile long long sum;

    // 배열 합산
    clock_gettime(CLOCK_MONOTONIC, &t0);
    sum = 0;
    for (int i = 0; i < N; i++) sum += arr[i];
    clock_gettime(CLOCK_MONOTONIC, &t1);
    printf("배열:              %5ld ms\n",
        (t1.tv_sec-t0.tv_sec)*1000 + (t1.tv_nsec-t0.tv_nsec)/1000000);

    // 연결 리스트 (순차 할당)
    clock_gettime(CLOCK_MONOTONIC, &t0);
    sum = 0;
    for (Node* n = head_seq; n; n = n->next) sum += n->value;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    printf("연결 리스트 (순차): %5ld ms\n",
        (t1.tv_sec-t0.tv_sec)*1000 + (t1.tv_nsec-t0.tv_nsec)/1000000);

    // 예상 결과:
    // 배열:              ~2 ms  (L3 히트 위주)
    // 연결 리스트 (순차): ~5 ms  (Node 사이 stride로 L2/L3 미스 증가)
    // 연결 리스트 (랜덤): ~50ms  (실제 사용 시 heap 단편화로 이 정도)

    free(arr); free(nodes); free(nodes_rand); free(order);
    return 0;
}
```

### 실험 3: cachegrind로 미스 위치 정확히 파악

```bash
# 행 우선 cachegrind
valgrind --tool=cachegrind --cache-sim=yes ./row_col row
cg_annotate cachegrind.out.<pid> row_col.c

# 출력에서 각 라인별 D cache miss 수 표시:
# Ir  I1mr  ILmr  Dr  D1mr  DLmr  file:function
# ...
# 64,000,000  0  0  64,000,000  4,000,000  10,000  row_col.c:sum += A[i][j]
#                                           ^L1 미스     ^LLC 미스

# 열 우선 cachegrind
valgrind --tool=cachegrind --cache-sim=yes ./row_col col
# sum += A[i][j] 라인에서 D1mr(L1 미스)이 64,000,000으로 증가
# → 이 한 줄이 전체 성능 차이의 원인임을 확인
```

---

## 📊 성능 비교

```
N=1024 int 배열 기준 지역성 패턴별 성능:

  접근 패턴                  캐시 미스 수    시간      IPC
  ─────────────────────────────────────────────────────────
  행 우선 순회 (최적)          65,536         ~5ms      ~2.5
  열 우선 순회 (최악)        1,048,576        ~80ms     ~0.15
  연결 리스트 (순차 할당)      ~200,000        ~15ms     ~0.8
  연결 리스트 (heap 단편화)   ~1,000,000       ~60ms     ~0.2
  랜덤 접근 (캐시 크기 초과)   ~1,048,576       ~300ms    ~0.05
  ─────────────────────────────────────────────────────────

캐시라인 64B 효과 (순차 접근):
  타입     바이트  1 캐시라인  미스 감소 배율
  int      4B     16개 확보   16배
  double   8B     8개 확보    8배
  char     1B     64개 확보   64배
  ─────────────────────────────────────────────────────────

블록 전치 vs 나이브 전치 (N=1024 double):
  나이브 전치 (열 쓰기):  ~800ms
  블록 전치 (32×32):     ~100ms
  → 8배 차이 (같은 O(N²) 알고리즘, 지역성만 다름)
```

---

## ⚖️ 트레이드오프

```
지역성 최적화의 트레이드오프:

행 우선 고집:
  ✅ C/C++ 자연스러운 레이아웃과 일치 → 코드 간단
  ✅ 순차 접근 → 공간 지역성 자동 확보
  ❌ 열 접근 패턴이 필요한 알고리즘 (예: 전치 행렬)에서 불리

캐시 블로킹(Tiling):
  ✅ 행렬 연산, 전치, 행렬 곱에서 획기적 성능 향상
  ✅ 시간/공간 지역성 동시 최대화
  ❌ 코드 복잡도 증가 (4중 루프, 블록 크기 조정 필요)
  ❌ 블록 크기가 틀리면 오히려 성능 저하

연결 리스트 대신 배열:
  ✅ 공간 지역성 → 미스 1/16 이하
  ✅ 프리페치 효과
  ✅ SIMD 벡터화 가능 (Ch5-04)
  ❌ 삽입/삭제가 O(N) → 수정이 잦으면 비적합
  ❌ 동적 크기 관리 복잡 (사전 크기 모를 때)

배열 기반 연결 리스트(Array-Based Linked List, Index 방식):
  ✅ 지역성 확보 + 삽입/삭제 O(1)
  // int next[N]; int prev[N]; int data[N];
  // 인덱스로 연결 — 포인터 추적 없이 배열 인덱싱
  ❌ 코드 간결성 손실
  ❌ 크기 사전 결정 필요

결론:
  데이터 접근 패턴이 설계 최우선
  "캐시 친화적이 되려면" → 접근 순서와 메모리 레이아웃을 맞춰라
  측정 없이 최적화하지 말 것: cachegrind로 실제 미스 위치 확인 먼저
```

---

## 📌 핵심 정리

```
지역성(Locality) 핵심:

두 가지 지역성:
  시간 지역성: 같은 주소를 반복 접근 → 워킹셋이 캐시에 남아야 효과
  공간 지역성: 인접 주소 연속 접근 → 캐시라인 64B 한 번 미스에 16개 int 확보

2차원 배열 교훈:
  C/C++: 행 우선(Row-Major) 저장
  행 우선 순회: 미스율 6.25% (1/16)
  열 우선 순회: 미스율 ~99% → 16배 느림
  → 내부 루프를 "빠르게 변하는 인덱스"가 연속 메모리 방향으로

연결 리스트 vs 배열:
  연결 리스트: 포인터 추적 → 매번 새 캐시라인 → 프리페치 불가
  배열: 순차 접근 → 공간 지역성 → 하드웨어 프리페치 효과
  → 순수 순회 성능: 배열이 10~15배 빠름

캐시라인 64B의 위력:
  int 16개 = 1 캐시라인
  순차 접근 시 미스 1번에 16개 확보 → 16배 미스 감소
  구조체가 64B를 넘으면 첫 접근 시 여러 캐시라인 → 미스 여러 번

지역성 최적화 순서:
  1. cachegrind로 미스 위치 확인
  2. 접근 패턴과 메모리 레이아웃 정렬
  3. 필요 시 캐시 블로킹
  4. Hot/Cold 필드 분리 (Ch2-05)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 두 코드가 있다. `double A[N][N], B[N][N], C[N][N]` (N=512)에서 행렬 곱 `C = A × B`를 구현한다. 어느 버전이 빠르고, 그 이유를 캐시라인 단위로 설명하라.

```c
// Version 1
for (int i=0; i<N; i++)
  for (int j=0; j<N; j++)
    for (int k=0; k<N; k++)
      C[i][j] += A[i][k] * B[k][j];

// Version 2
for (int i=0; i<N; i++)
  for (int k=0; k<N; k++)
    for (int j=0; j<N; j++)
      C[i][j] += A[i][k] * B[k][j];
```

<details>
<summary>해설 보기</summary>

**Version 2가 훨씬 빠릅니다.**

Version 1 (i-j-k 순서)의 문제:
- 내부 루프(k)에서 `B[k][j]`는 k가 변할 때 열 접근 → B를 열 방향으로 접근
- j(고정)에 대해 k가 증가할수록 `B[0][j], B[1][j], B[2][j]...` → 서로 다른 캐시라인
- B에 대한 L1 미스가 N³ 수준 발생

Version 2 (i-k-j 순서)의 장점:
- 내부 루프(j)에서 `B[k][j]`는 j가 변할 때 행 접근 → 순차 접근, 공간 지역성 ✅
- `C[i][j]`도 j가 변할 때 행 접근 → 순차 접근 ✅
- `A[i][k]`는 k, j 루프 동안 고정 → 레지스터에 상주 (시간 지역성 ✅)

실측: N=512 double 기준, Version 1 ~5초 vs Version 2 ~0.5초 (10배 차이)

이것이 컴파일러가 루프 변환(loop interchange)을 수행하는 이유이기도 합니다. `gcc -O3`은 자동으로 루프 순서를 바꿀 수 있지만, 의존성 분석이 필요해 항상 성공하지는 않습니다.

</details>

---

**Q2.** 게임 엔진에서 `Entity`를 아래 두 방식으로 관리할 때, `Update()` 루프 성능 차이를 지역성 관점에서 설명하라.

```c
// AoS: Array of Structs
struct Entity {
    float x, y, z;      // Position (12B)
    float vx, vy, vz;   // Velocity (12B)
    char name[64];       // Name (64B) ← 업데이트 시 불필요
    int  flags;          // Flags (4B)
    // Total: 96B
};
Entity entities[10000];

// Update: 위치 += 속도
for (int i = 0; i < 10000; i++)
    entities[i].x += entities[i].vx; // 96B 로드, 24B만 사용
```

<details>
<summary>해설 보기</summary>

**AoS의 문제**: `Entity` 구조체가 96B이므로 캐시라인 2개를 차지합니다. `Update()`에서 x, y, z, vx, vy, vz(24B)만 필요한데, `name[64]`도 항상 캐시라인에 올라옵니다.

- 10,000개 엔티티 업데이트 = 10,000 × 2 캐시라인 로드 (실제 필요: ~10,000 × 0.4 캐시라인)
- L1(32KB)에 들어가는 엔티티 수: 32KB / 96B ≈ 341개 → 자주 교체됨

**SoA (Struct of Arrays)로 개선**:
```c
struct EntityPool {
    float x[10000], y[10000], z[10000];   // Position 배열
    float vx[10000], vy[10000], vz[10000]; // Velocity 배열
    char  name[10000][64];  // Name 배열 (업데이트 시 접근 안 함)
    int   flags[10000];
};
// Update:
for (int i = 0; i < 10000; i++)
    pool.x[i] += pool.vx[i];  // x와 vx 배열만 순차 접근
```

SoA에서 x[0..15]는 64B 캐시라인 1개, vx[0..15]도 64B 1개. 총 2 캐시라인으로 16개 엔티티 처리 → AoS 대비 약 4배 효율적입니다. 이것이 Ch2-05와 Ch5-04에서 다루는 SoA의 핵심 이점입니다.

</details>

---

**Q3.** 다음 코드에서 `sum1`과 `sum2` 계산의 캐시 히트율이 왜 다른가? 각 계산에서 캐시라인 미스 횟수를 추정하라. (N=1024, int[N] 배열, L1=32KB)

```c
int a[1024], b[1024];
// a: 0x1000번지, b: 0x2000번지 (64B 캐시라인 경계 정렬)

volatile int sum1 = 0, sum2 = 0;

// Loop 1: a와 b 교대 접근
for (int i = 0; i < 1024; i++) {
    sum1 += a[i];
    sum1 += b[i];
}

// Loop 2: a 먼저, b 다음
for (int i = 0; i < 1024; i++) sum2 += a[i];
for (int i = 0; i < 1024; i++) sum2 += b[i];
```

<details>
<summary>해설 보기</summary>

두 배열의 크기: 1024 × 4B = 4KB 각 → 합계 8KB < L1 32KB

**Loop 1 (교대 접근)**:
- `a[0]` 미스 → a의 캐시라인 로드 (a[0]~a[15])
- `b[0]` 미스 → b의 캐시라인 로드 (b[0]~b[15])
- `a[1]`~`a[15]`: 히트 (이미 로드됨)
- `b[1]`~`b[15]`: 히트
- 총 미스: a 64라인 + b 64라인 = **128번**

**Loop 2 (순서 분리)**:
- a[0]~a[1023]: 64번 미스
- b[0]~b[1023] 시작 시: a 캐시라인이 L1에 남아있음 (8KB << 32KB)
- b[0]~b[1023]: 64번 미스
- 총 미스: 64 + 64 = **128번**

**결론**: 이 경우 두 루프의 미스 수는 동일합니다. 둘 다 8KB 워킹셋이 L1(32KB)에 완전히 들어가기 때문입니다.

차이가 생기는 경우: 배열이 커서 L1을 초과할 때입니다. 예를 들어 L1이 16KB이고 배열이 각 8KB라면, Loop 1은 a와 b가 번갈아 L1을 차지해 `a[i]` 접근 시 이전에 올린 a 라인이 퇴출될 수 있습니다. Loop 2는 a를 완전히 처리하고 b를 처리하므로 캐시를 더 효율적으로 사용합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 캐시 동작 원리](./02-cache-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: 캐시 미스의 종류 ➡️](./04-cache-miss-types.md)**

</div>
