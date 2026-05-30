# 캐시 미스의 종류 — Compulsory·Capacity·Conflict

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 캐시 미스를 Compulsory·Capacity·Conflict 세 가지로 분류하는 3C 모델이란 무엇인가?
- cachegrind 출력에서 각 미스 유형을 어떻게 분리해 읽는가?
- 2의 거듭제곱 크기 배열이 Conflict miss를 집중시키는 이유는?
- 배열 크기 조정과 패딩(padding)으로 Conflict miss를 회피하는 기법은?
- 워킹셋(Working Set)이 L1 → L2 → L3 → DRAM 경계를 넘을 때 성능이 절벽처럼 떨어지는 이유는?
- 같은 알고리즘을 데이터 크기만 바꾸며 실험하면 성능 절벽을 어떻게 가시화할 수 있는가?

---

## 🔍 왜 이 개념이 중요한가

### "캐시 미스가 많다 — 그래서 왜인지는 모른다"

```
흔한 상황:
  perf stat을 실행했더니 L1-dcache-load-misses가 15%
  "캐시 미스가 많다"는 것을 알았지만 해결책을 모름

  3C 모델 없이는:
    ① 배열이 너무 커서 (Capacity) → 캐시 크기를 늘릴 수 없으니 포기?
    ② 처음 접근이라서 (Compulsory) → 어쩔 수 없는 미스?
    ③ 배열 크기가 나빠서 (Conflict) → 배열 크기만 살짝 바꾸면 해결?

  3가지 원인에 따라 해결책이 완전히 다르다:
    Compulsory → 프리페치로 비용 숨기기
    Capacity   → 워킹셋 줄이기, 캐시 블로킹
    Conflict   → 배열 크기 패딩, 배치 변경

실무 예시:
  벡터 내적 코드 — 두 배열을 N = 262144(= 2^18)개 float으로 실험
  vs N = 262145(= 2^18 + 1)개 float으로 실험

  N = 2^18: Conflict miss 폭발 → 예상보다 3배 느림
  N = 2^18 + 1: Conflict miss 거의 없음 → 정상 속도

  "배열 원소 하나 차이가 3배 속도 차이" → 원인은 Conflict miss
  3C 모델을 모르면 이 현상을 이해할 수 없다
```

---

## 😱 잘못된 이해

### Before: "캐시 미스 = 데이터가 캐시에 없는 것 — 캐시가 작거나 데이터가 크면 발생한다"

```
단순화된 모델:
  캐시 미스 원인 = 캐시가 작다 또는 데이터가 많다

  이 모델로는 설명 안 되는 현상:
    ① 배열이 캐시 크기의 절반인데도 미스율이 매우 높음
       → Conflict miss (같은 Set에 몰림)

    ② 코드의 첫 실행만 느리고 두 번째 실행은 빠름
       → Compulsory miss (초기 로드 비용)

    ③ 배열 크기를 딱 1개만 늘렸더니 갑자기 빨라짐
       → Conflict miss 패턴이 깨짐

  잘못된 대응:
    "미스가 많으면 캐시를 더 큰 레벨에 올려라"
    → Conflict miss는 캐시를 크게 해도 해결 안 됨
       (직접 매핑이나 작은 연관도에서는 같은 집합 충돌)
    → 배열 크기 또는 배치만 바꾸면 해결됨
```

---

## ✨ 올바른 이해

### After: 3C 모델로 미스 원인을 분류하면 처방이 보인다

```
3C 모델 (Three C's of Cache Misses):

┌─────────────────────────────────────────────────────────────┐
│  Compulsory Miss (강제 미스, Cold Miss)                      │
│    정의: 해당 주소에 처음 접근할 때 반드시 발생하는 미스       │
│    특징: 캐시가 아무리 커도, 연관도가 아무리 높아도 피할 수 없음│
│    예시: 프로그램 시작 시 코드 로드, 배열 첫 번째 순회         │
│    해결: 프리페치(prefetch)로 지연 숨기기                     │
├─────────────────────────────────────────────────────────────┤
│  Capacity Miss (용량 미스)                                   │
│    정의: 워킹셋이 캐시 크기를 초과해 재접근 시 이미 퇴출된 미스 │
│    특징: 완전 연관(Fully Associative) 캐시에서도 발생          │
│    예시: 8MB 배열을 L3(8MB) 이상으로 순회 → 재접근 시 미스     │
│    해결: 워킹셋 줄이기 (캐시 블로킹, SoA, 알고리즘 개선)       │
├─────────────────────────────────────────────────────────────┤
│  Conflict Miss (충돌 미스)                                   │
│    정의: 같은 캐시 Set에 매핑된 주소가 Way 수를 초과해 발생    │
│    특징: 캐시 전체 용량이 충분해도 발생 (완전 연관이면 없음)    │
│    예시: 2의 거듭제곱 크기 배열 2개를 동시에 접근              │
│    해결: 배열 크기 패딩, 배치 변경                            │
└─────────────────────────────────────────────────────────────┘

직관적 비유:
  호텔(캐시)에 손님(데이터) 체크인 비유
  Compulsory: 처음 방문한 손님 — 체크인 절차 불가피
  Capacity:   방이 꽉 찼는데 새 손님 → 기존 손님 퇴실
  Conflict:   방이 남아있는데 특정 층(Set)만 꽉 참 → 그 층에 배정된 손님만 퇴실
```

---

## 🔬 내부 동작 원리

### 1. Compulsory Miss — 피할 수 없는 초기 비용

```c
/*
 * Compulsory Miss 발생 시점:
 *  - 처음 접근하는 모든 캐시라인 = 무조건 1번 미스
 *  - 캐시 크기나 연관도와 무관
 *  - 프리페치로 지연을 숨길 수 있지만 미스 자체는 발생
 */

// 예시: 10MB 배열을 처음 순회
int arr[2621440];  // 10MB = 2.5M int

// 첫 번째 순회: Compulsory miss 폭발
for (int i = 0; i < 2621440; i++)
    sum += arr[i];
// 미스 수 = 10MB / 64B = 163,840번 (모두 Compulsory)

// 두 번째 순회 (워킹셋이 캐시에 들어간 경우):
for (int i = 0; i < 2621440; i++)
    sum += arr[i];
// L3(8MB) 초과 → 일부 재미스 (Capacity miss)
// L3(8MB) 이하라면 대부분 히트

/*
 * Compulsory miss 최소화 전략:
 * 1. 소프트웨어 프리페치: 접근 직전에 미리 로드 요청
 * 2. 하드웨어 프리페치: 순차/스트라이드 패턴에서 자동 동작
 */
for (int i = 0; i < N; i++) {
    __builtin_prefetch(&arr[i + 64], 0, 1);  // 64 요소 앞 미리 요청
    sum += arr[i];  // 이미 로드 진행 중 → 지연 숨김
}
// Compulsory miss 지연이 계산과 겹쳐서 숨겨짐
```

### 2. Capacity Miss — 워킹셋이 캐시 크기를 넘을 때

```
Capacity Miss 원리:

  워킹셋 W > 캐시 크기 C일 때 발생
  완전 연관 캐시에서도 피할 수 없는 미스

  예: L1(32KB) vs 배열 크기

  워킹셋 16KB (L1의 절반):
    전체 배열이 L1에 상주 → 재접근 시 히트 → Capacity miss 없음

  워킹셋 64KB (L1의 2배):
    L1에 들어가지 않음 → 재접근 시 L2에서 가져옴
    이 경우 Capacity miss = L1 기준에서 발생

  워킹셋 300KB (L2의 약간 초과):
    L2(256KB)에도 들어가지 않음 → L3에서 가져옴

성능 절벽(Performance Cliff) 발생 위치:
  워킹셋 < L1(32KB):   ~ 4 사이클/접근
  워킹셋 ≈ L1~L2:      ~ 12 사이클/접근  ← 첫 번째 절벽
  워킹셋 ≈ L2~L3:      ~ 40 사이클/접근  ← 두 번째 절벽
  워킹셋 > L3(8MB):    ~ 200 사이클/접근 ← 세 번째 절벽 (DRAM)

  이 절벽은 워킹셋을 1바이트씩 늘릴 때 계단식으로 나타남
  → "성능 절벽 실험"으로 측정 가능 (실험 섹션 참고)
```

### 3. Conflict Miss — 2의 거듭제곱이 만드는 함정

```
Conflict Miss 원리 (Ch2-02 복습):

  Set-Associative 캐시에서 주소는 특정 Set에만 들어갈 수 있음
  같은 Set에 Way 수 이상의 주소가 몰리면 Conflict miss 발생

  32KB, 8-way, 64B 캐시라인 기준:
    Set 수 = 32KB / (8 × 64B) = 64개
    인덱스 비트 = bit[11:6] = 6비트

  두 주소가 같은 Set에 매핑되는 조건:
    bit[11:6]이 동일 ← 주소 차이가 4096B(2^12)의 배수

  2의 거듭제곱 크기 배열에서의 충돌:
    float A[4096];  // 4096 × 4B = 16KB = 2^14 B
    float B[4096];  // 같은 크기

    A[0]과 B[0]의 주소 차이가 정확히 16KB = 2^14 B라면:
      A[0] bit[11:6] = 0
      B[0] bit[11:6] = 0  (2^14이 4096의 배수)
      → 둘 다 Set 0에 매핑

    A[1]과 B[1]:
      A[1] bit[11:6] = 0 (오프셋 비트만 다름)
      B[1] bit[11:6] = 0
      → 둘 다 Set 0에 매핑

  결과: A 전체와 B 전체가 같은 Set들에 매핑
        8-way에서 A와 B가 각각 1 Way씩 점유 → 공존 가능
        8-way를 초과하는 배열 9개가 같은 Set에 매핑되면 → Conflict!

  실제 문제:
    행렬 A[N][N]과 B[N][N]에서 N = 2^k일 때
    A[i][j]와 B[i][j]가 같은 Set에 몰려 서로를 퇴출
    → 교대 접근 시 Thrashing 발생
```

### 4. Conflict Miss 회피 — 배열 크기 패딩

```c
/*
 * Conflict Miss 회피: 배열 크기를 2^k에서 벗어나게 패딩
 *
 * 핵심 원리:
 *  주소 차이가 cache_size/ways의 배수가 되지 않도록
 *  배열 사이에 "이상한" 간격을 끼워 넣는다
 */

// Bad: N이 2의 거듭제곱 — A와 B가 같은 Set에 매핑
#define N 1024
float A_bad[N][N];   // 4MB
float B_bad[N][N];   // 4MB — A와 완전히 같은 Set 패턴

// Good: N+1 또는 N+소수로 패딩하여 인덱스 어긋남
#define N_PAD (1024 + 1)   // 1025 = 2의 거듭제곱 아님
float A_good[N][N_PAD];    // 열 방향 1열 여유
float B_good[N][N_PAD];    // A와 배열 보폭(stride)이 달라짐

// 또는 배열 사이에 패딩 삽입
#define CACHE_LINE 64
float A_padded[N][N];
char  _pad[CACHE_LINE * 7];  // 7 캐시라인 = 448B 간격
float B_padded[N][N];
// A_padded[0]과 B_padded[0]의 주소 차이 = 4MB + 448B
// 448 = 7 × 64 → bit[11:6]이 어긋남 → 다른 Set에 매핑

/*
 * 실용 규칙:
 *  행렬 크기 N이 2^k 형태이면 N+8 또는 N+소수(3,5,7)로 대체
 *  valgrind --tool=cachegrind로 Conflict miss가 줄었는지 확인
 */

// 행렬 내적 성능 비교 예시
void matmul_bad(float C[N][N], float A[N][N], float B[N][N]) {
    // N=1024, 2의 거듭제곱 → Conflict miss 폭발
    for (int i = 0; i < N; i++)
        for (int k = 0; k < N; k++)
            for (int j = 0; j < N; j++)
                C[i][j] += A[i][k] * B[k][j];
}

void matmul_good(float C[N][N_PAD], float A[N][N_PAD], float B[N][N_PAD]) {
    // N_PAD=1025 → Conflict miss 감소
    for (int i = 0; i < N; i++)
        for (int k = 0; k < N; k++)
            for (int j = 0; j < N; j++)
                C[i][j] += A[i][k] * B[k][j];
}
// gcc -O2 -o matmul matmul.c && time ./matmul
// Bad: ~3.2초, Good: ~1.8초 (같은 알고리즘, 크기만 다름)
```

### 5. 성능 절벽(Performance Cliff) — 계층 경계를 넘는 순간

```
성능 절벽 현상:

  워킹셋 크기를 1KB씩 늘리면서 접근 지연 측정:

  워킹셋   → 평균 접근 지연
  ─────────────────────────────────────────────
   4KB     →  ~4 cy   (L1에 완전 수용)
   8KB     →  ~4 cy   (L1에 완전 수용)
  16KB     →  ~4 cy   (L1 32KB 절반)
  32KB     →  ~4 cy   (L1 32KB 한계)
  40KB     → ~10 cy   ← 첫 번째 절벽! (L1 초과 → L2)
  64KB     → ~12 cy   (L2에 수용)
  128KB    → ~12 cy   (L2에 수용)
  256KB    → ~12 cy   (L2 256KB 한계)
  300KB    → ~30 cy   ← 두 번째 절벽! (L2 초과 → L3)
  1MB      → ~40 cy   (L3에 수용)
  4MB      → ~40 cy   (L3에 수용)
  8MB      → ~40 cy   (L3 8MB 한계)
  10MB     →~150 cy   ← 세 번째 절벽! (L3 초과 → DRAM)
  100MB    →~200 cy   (DRAM 직접 접근)
  ─────────────────────────────────────────────

  절벽이 계단식으로 나타나는 이유:
    각 캐시 계층은 크기가 고정 → 워킹셋이 그 크기를 넘는 순간
    재접근 시 더 느린 계층으로 폴백
    경계를 1바이트 넘는 것만으로도 다음 계층 지연으로 점프

ASCII 성능 그래프:
  지연
  (cy)
  200│                                              ████
     │                                              ████
     │                                              ████
  40 │                              ████████████████
     │                              ████████████████
  12 │          ████████████████████
     │          ████████████████████
   4 │██████████
     └──────────────────────────────────────────────────▶
     1KB  8KB 32KB 64KB 256KB 1MB  4MB  8MB  16MB  64MB
           L1   L1   L2   L2   L3   L3   L3
          절벽      절벽          절벽
```

---

## 💻 실전 실험

### 실험 1: cachegrind로 3C 미스 분리 측정

```bash
# cachegrind는 D1(L1 데이터)과 LL(Last-Level) 미스를 분리 출력
# D1  miss = Compulsory + Capacity + Conflict (L1 기준 전체 미스)
# LL  miss = 주로 Capacity miss (L3를 뚫고 DRAM 접근)

# 행 우선 순회 (지역성 좋은 경우)
valgrind --tool=cachegrind ./row_col row
cg_annotate cachegrind.out.<pid>

# 예상 출력 (요약):
# ==12345== D   refs:      1,048,576  (1M 접근)
# ==12345== D1  misses:       65,536  (6.25% — 공간 지역성 효과)
# ==12345== LLd misses:        4,096  (0.4% — L3 히트율 높음)
#
# D1  miss - LLd miss ≈ Conflict + Capacity at L1
# LLd miss ≈ DRAM까지 간 미스 (주로 Capacity)

# Conflict miss 집중 재현 (2의 거듭제곱 배열 2개 교대 접근)
cat > conflict_test.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#define N (1 << 22)  // 4M int = 16MB (L3 초과)

int A[N], B[N];

int main() {
    volatile long long sum = 0;
    // 같은 인덱스를 교대 접근 — A와 B가 같은 Set에 매핑될 경우
    for (int i = 0; i < N; i++) {
        sum += A[i];
        sum += B[i];
    }
    printf("sum=%lld\n", sum);
    return 0;
}
EOF
gcc -O2 -o conflict_test conflict_test.c

valgrind --tool=cachegrind \
         --D1=32768,8,64 \
         --LL=8388608,16,64 \
         ./conflict_test

# D1 miss가 높을 때 → 주소 배치 확인
# nm ./conflict_test | grep -E "^[0-9a-f]+ [Bb] [AB]$"
# A와 B 사이 주소 차이가 4096의 배수인지 확인
```

### 실험 2: 워킹셋 크기별 성능 절벽 실험

```c
// working_set_cliff.c
// 워킹셋을 2배씩 늘리며 평균 접근 지연을 측정
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <time.h>

#define MAX_SIZE (64 * 1024 * 1024)  // 64MB
#define REPS 16

static uint8_t buffer[MAX_SIZE];

// 포인터 추적(pointer chasing)으로 하드웨어 프리페치 무력화
// 랜덤 순서로 접근 → 패턴 없음 → 순수 캐시 지연 측정
void init_random_walk(uint8_t* buf, size_t size, size_t stride) {
    // 각 요소가 stride만큼 떨어진 다음 위치를 가리키게 구성
    size_t count = size / stride;
    size_t* ptr = (size_t*)buf;
    // 셔플된 순서로 포인터 연결
    size_t* order = malloc(count * sizeof(size_t));
    for (size_t i = 0; i < count; i++) order[i] = i;
    // Fisher-Yates 셔플
    for (size_t i = count - 1; i > 0; i--) {
        size_t j = rand() % (i + 1);
        size_t tmp = order[i]; order[i] = order[j]; order[j] = tmp;
    }
    for (size_t i = 0; i < count - 1; i++)
        ptr[order[i] * (stride / sizeof(size_t))] =
            (uintptr_t)&ptr[order[i+1] * (stride / sizeof(size_t))];
    ptr[order[count-1] * (stride / sizeof(size_t))] = (uintptr_t)&ptr[order[0] * (stride / sizeof(size_t))];
    free(order);
}

int main() {
    printf("%-12s  %-10s  %-12s\n", "워킹셋", "ns/접근", "캐시 계층");

    for (size_t size = 4096; size <= MAX_SIZE; size *= 2) {
        init_random_walk(buffer, size, 64);  // 캐시라인 단위 stride
        volatile size_t* ptr = (size_t*)buffer;

        // 워밍업
        for (int w = 0; w < 100; w++) ptr = (size_t*)*ptr;

        struct timespec t0, t1;
        clock_gettime(CLOCK_MONOTONIC, &t0);
        size_t accesses = (size_t)(1e7 / (size / 4096.0)) + 1000;
        for (size_t i = 0; i < accesses; i++)
            ptr = (size_t*)*ptr;
        clock_gettime(CLOCK_MONOTONIC, &t1);

        double ns = ((t1.tv_sec - t0.tv_sec) * 1e9 +
                     (t1.tv_nsec - t0.tv_nsec)) / accesses;

        const char* tier;
        if (size <= 32*1024)          tier = "L1";
        else if (size <= 256*1024)    tier = "L2";
        else if (size <= 8*1024*1024) tier = "L3";
        else                           tier = "DRAM";

        printf("%-12zu  %-10.1f  %-12s\n", size, ns, tier);
        (void)ptr;  // 최적화 방지
    }
    return 0;
}
// gcc -O2 -o cliff working_set_cliff.c && ./cliff
```

```bash
# 실행 결과 예시 (Intel Core i7, 이 머신의 실제 값과 다를 수 있음):
# 워킹셋        ns/접근     캐시 계층
# 4096          1.3         L1
# 8192          1.3         L1
# 16384         1.3         L1
# 32768         1.4         L1
# 65536         3.8         L2       ← 첫 번째 절벽 (~3배 증가)
# 131072        4.2         L2
# 262144        4.3         L2
# 524288        13.2        L3       ← 두 번째 절벽 (~3배 증가)
# 1048576       14.8        L3
# 4194304       15.2        L3
# 8388608       16.0        L3
# 16777216      62.1        DRAM     ← 세 번째 절벽 (~4배 증가)
# 33554432      63.5        DRAM
# 67108864      64.0        DRAM
```

### 실험 3: Conflict miss — 배열 크기 패딩 전후 비교

```c
// conflict_padding.c
// 2의 거듭제곱 크기 vs 패딩된 크기의 행렬 연산 비교
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#define N 1024
#define N_GOOD (N + 7)  // 2의 거듭제곱 아님 → Conflict miss 감소

float A_bad[N][N],    B_bad[N][N];
float A_good[N][N_GOOD], B_good[N][N_GOOD];

long run_bad(int reps) {
    volatile float sum = 0;
    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int r = 0; r < reps; r++)
        for (int i = 0; i < N; i++)
            for (int j = 0; j < N; j++)
                sum += A_bad[i][j] * B_bad[j][i];  // B에 열 접근
    clock_gettime(CLOCK_MONOTONIC, &t1);
    return (t1.tv_sec - t0.tv_sec) * 1000 +
           (t1.tv_nsec - t0.tv_nsec) / 1000000;
}

long run_good(int reps) {
    volatile float sum = 0;
    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int r = 0; r < reps; r++)
        for (int i = 0; i < N; i++)
            for (int j = 0; j < N; j++)
                sum += A_good[i][j] * B_good[j][i];  // B에 열 접근
    clock_gettime(CLOCK_MONOTONIC, &t1);
    return (t1.tv_sec - t0.tv_sec) * 1000 +
           (t1.tv_nsec - t0.tv_nsec) / 1000000;
}

int main() {
    // 초기화
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < N; j++) {
            A_bad[i][j]    = A_good[i][j]    = (float)(i * N + j);
            B_bad[i][j]    = B_good[i][j]    = (float)(j * N + i);
        }
    }
    int reps = 3;
    printf("N=%d (2의 거듭제곱): %ld ms\n",   N,      run_bad(reps));
    printf("N=%d (패딩 +7):     %ld ms\n",   N_GOOD, run_good(reps));
    // 예상: Bad ~1800ms, Good ~1100ms (같은 알고리즘, 크기만 다름)
    return 0;
}
// gcc -O2 -o conflict_padding conflict_padding.c
// perf stat -e L1-dcache-load-misses,L2-load-misses ./conflict_padding
```

### 실험 4: perf로 성능 절벽 이벤트 확인

```bash
# 워킹셋별 캐시 미스 이벤트 측정
# L1 미스, L2 미스, LLC(L3) 미스를 각각 측정

# 작은 워킹셋 (L1 내 수용)
perf stat -e \
  L1-dcache-loads,L1-dcache-load-misses,\
  l2_rqsts.demand_data_rd_hit,l2_rqsts.demand_data_rd_miss,\
  LLC-loads,LLC-load-misses \
  ./cliff 32768    # 32KB 워킹셋

# 예상 출력 (32KB):
# L1-dcache-loads:            10,000,000
# L1-dcache-load-misses:           1,000   ← 0.01% (L1에서 해결)
# LLC-loads:                           0   ← L3 접근 거의 없음

# 큰 워킹셋 (DRAM)
perf stat -e \
  L1-dcache-loads,L1-dcache-load-misses,\
  LLC-loads,LLC-load-misses \
  ./cliff 33554432  # 32MB 워킹셋

# 예상 출력 (32MB):
# L1-dcache-load-misses:       9,900,000   ← 99% (L1 미스)
# LLC-load-misses:             9,800,000   ← 98% (DRAM 직행)

# cachegrind로 미스 유형 분리 (L1 vs LL 구분)
valgrind --tool=cachegrind --branch-sim=yes ./conflict_padding
cg_annotate cachegrind.out.<pid>
# D1  miss: L1 기준 미스 (Compulsory + Capacity + Conflict 합계)
# LLd miss: L3까지 뚫린 미스 (주로 Capacity miss)
# D1 miss >> LLd miss 이면 → L2에서 해결되는 Conflict/Capacity 미스 많음
# D1 miss ≈ LLd miss 이면 → DRAM까지 가는 Capacity miss가 주요 원인
```

---

## 📊 성능 비교

```
3C 미스 유형별 해결 방법과 효과:

  미스 유형     발생 원인               해결책                     효과
  ─────────────────────────────────────────────────────────────────────
  Compulsory    첫 접근 (불가피)         소프트웨어 프리페치          지연 숨김 (미스 자체 제거 불가)
  Capacity      워킹셋 > 캐시 크기       캐시 블로킹, SoA, 알고리즘  히트율 직접 향상 (수 배 효과)
  Conflict      같은 Set 과부하          배열 크기 패딩, 배치 변경    히트율 직접 향상 (2~5배 효과)
  ─────────────────────────────────────────────────────────────────────

성능 절벽 실측 (Intel Core i7 기준 포인터 추적 접근):

  워킹셋       캐시 히트         평균 지연   IPC
  ─────────────────────────────────────────────────
   ≤ 32KB      L1                ~1.3 ns    ~2.5
   ≤ 256KB     L2                ~4.0 ns    ~0.9   ← 3배 증가
   ≤ 8MB       L3               ~15.0 ns    ~0.25  ← 11배 증가
   > 8MB       DRAM             ~64.0 ns    ~0.06  ← 50배 증가 (L1 대비)
  ─────────────────────────────────────────────────

N=1024 행렬, Conflict miss 패딩 전후:

  배열 크기    L1-dcache-miss   실행 시간
  ─────────────────────────────────────────────
  N=1024       28,000,000       ~1800 ms  ← Conflict miss 높음
  N=1031       11,000,000       ~1050 ms  ← Conflict miss 60% 감소
  ─────────────────────────────────────────────
  → 코드 변경 없이 크기만 바꿔 1.7배 가속

cachegrind D1/LLd 미스 비율로 원인 추론:

  D1 miss  LLd miss  비율      주요 원인
  ─────────────────────────────────────────────
  100,000   2,000    2%        Conflict (L2 내 해결)
  100,000  80,000    80%       Capacity (DRAM 직행)
  100,000  50,000    50%       혼합 (Capacity + Conflict)
  ─────────────────────────────────────────────
```

---

## ⚖️ 트레이드오프

```
3C 미스 대응 전략의 트레이드오프:

Compulsory miss 대응 — 프리페치:
  ✅ Compulsory 지연을 계산과 오버랩하여 숨김
  ✅ 순차/스트라이드 패턴에서 하드웨어 자동 처리
  ❌ 랜덤 접근에서는 프리페치 효과 없음 (next 주소 예측 불가)
  ❌ 소프트웨어 프리페치는 타이밍 조절이 어려움 (너무 빠르면 퇴출됨)

Capacity miss 대응 — 캐시 블로킹:
  ✅ 가장 확실한 Capacity miss 감소 (워킹셋을 캐시 안으로)
  ✅ 행렬 연산, 이미지 처리 등 다양한 워크로드에 적용 가능
  ❌ 코드 복잡도 증가 (중첩 루프, 블록 크기 조정)
  ❌ 블록 크기가 머신에 따라 다름 → 이식성 문제 (BLAS 라이브러리 활용 권장)

Conflict miss 대응 — 배열 크기 패딩:
  ✅ 간단한 변경 (크기에 소수 또는 비거듭제곱 수 더하기)
  ✅ 코드 로직 수정 없음
  ❌ 메모리 낭비 (패딩 열/행은 사용 안 함)
  ❌ 최적 패딩 크기는 CPU 캐시 구조에 따라 다름
  ❌ cachegrind로 확인하지 않으면 효과 불확실

실무 결론:
  1. 먼저 cachegrind로 D1/LLd 미스 비율 확인
  2. LLd miss가 많으면 → Capacity 문제 → 워킹셋 줄이기
  3. D1 miss만 많으면 → Conflict 가능성 → 배열 크기 패딩 시도
  4. 첫 실행만 느리면 → Compulsory → 프리페치 고려
  측정 없이 최적화 방향을 결정하지 말 것
```

---

## 📌 핵심 정리

```
3C 캐시 미스 모델 핵심:

Compulsory (강제 미스):
  처음 접근 시 항상 발생 — 피할 수 없음
  해결: 프리페치로 지연만 숨길 수 있음

Capacity (용량 미스):
  워킹셋 > 캐시 크기 — 재접근 시 이미 퇴출됨
  해결: 캐시 블로킹, SoA로 핫 데이터만 접근, 알고리즘 개선

Conflict (충돌 미스):
  같은 캐시 Set에 너무 많은 주소 몰림
  2의 거듭제곱 크기 배열에서 집중 발생
  해결: 배열 크기에 소수 더하기, 배열 사이 패딩 삽입

성능 절벽 3단계:
  워킹셋 > L1(32KB)   → 3배 지연 증가
  워킹셋 > L2(256KB)  → 3배 추가 증가 (L1 대비 ~10배)
  워킹셋 > L3(8MB)    → 4배 추가 증가 (L1 대비 ~50배)

측정 도구:
  cachegrind: D1 miss (Compulsory+Capacity+Conflict 합계)
              LLd miss (주로 Capacity — DRAM 직행)
  perf stat:  LLC-load-misses로 DRAM 접근 빈도 확인

다음 단계:
  Conflict miss 원인 → Ch2-02 (캐시 집합 연관 구조)
  Capacity miss 해결 → Ch2-05 (SoA, 핫/콜드 분리)
  DRAM까지 가는 비용 → Ch2-06 (TLB와 페이지 테이블 워크)
```

---

## 🤔 생각해볼 문제

**Q1.** `float A[1024][1024]`와 `float B[1024][1024]`를 선언하고 `for (i) for (j) C[i][j] = A[i][j] + B[i][j]`를 실행할 때, A와 B가 동일한 캐시 Set에 매핑되는 경우 Conflict miss가 발생한다. 이를 확인하고 해결하는 방법을 단계적으로 설명하라.

<details>
<summary>해설 보기</summary>

**확인 단계**:

1. `nm` 또는 `objdump -t`로 A와 B의 주소를 확인
2. A와 B의 주소 차이를 계산 → 차이가 `캐시 크기 / way 수`의 배수이면 Conflict 가능성

예: L2(256KB, 4-way):
- 같은 Set 매핑 조건 = 주소 차이가 256KB/4 = 64KB의 배수
- A[1024][1024] = 4MB, B[1024][1024] = 4MB → 차이가 4MB = 64 × 64KB의 배수 → 같은 Set!

**해결 방법**:

방법 1 — 열 크기 패딩:
```c
float A[1024][1024 + 8];  // 1032열로 패딩
float B[1024][1024 + 8];
// 행 보폭(stride) = 1032 × 4B = 4128B (2의 거듭제곱 아님)
// A[i]와 B[i]가 다른 Set에 매핑될 확률 높아짐
```

방법 2 — 배열 사이 패딩:
```c
float A[1024][1024];
char _pad[64 * 7];  // 448B 비정렬 간격
float B[1024][1024];
```

방법 3 — 동적 할당으로 주소 어긋남:
```c
float* A = aligned_alloc(64, 4*1024*1024 + 512);  // 여분 공간
float* B = aligned_alloc(64, 4*1024*1024);
// A와 B의 할당 주소가 임의로 다름 → Conflict miss 감소
```

**검증**: `valgrind --tool=cachegrind` 실행 후 D1 miss가 줄었는지 확인. 패딩 전후 D1 miss 수를 비교하여 Conflict miss가 감소했음을 정량적으로 확인한다.

</details>

---

**Q2.** "성능 절벽(Performance Cliff)"을 실제 코드에서 발견했을 때, 이 절벽이 L1/L2/L3/DRAM 중 어느 계층 경계인지 판단하는 방법을 설명하라.

<details>
<summary>해설 보기</summary>

**판단 방법**:

1. **워킹셋 크기 계산**: 코드가 실제로 접근하는 고유 메모리 양을 계산
   - 예: `float A[N]`이면 워킹셋 = N × 4B

2. **캐시 크기 확인**:
   ```bash
   getconf LEVEL1_DCACHE_SIZE    # L1
   getconf LEVEL2_CACHE_SIZE     # L2
   getconf LEVEL3_CACHE_SIZE     # L3
   ```

3. **미스 이벤트 측정**:
   ```bash
   perf stat -e L1-dcache-load-misses,LLC-load-misses ./myprogram
   ```
   - `L1-dcache-load-misses` 높고 `LLC-load-misses` 낮음 → L2/L3에서 해결 (Conflict 또는 Capacity L1)
   - 두 값 모두 높음 → DRAM 접근 (Capacity L3 이상)

4. **워킹셋 크기별 스윕 실험**: 입력 크기 N을 2배씩 늘리며 성능 측정
   ```bash
   for size in 4K 8K 16K 32K 64K 128K 256K 512K 1M 4M 8M 16M; do
       echo -n "$size: "; ./working_set_test $size
   done
   ```
   성능이 갑자기 떨어지는 지점 = 절벽 위치

5. **절벽 위치와 캐시 크기 대조**: 절벽이 발생하는 워킹셋 크기가 어느 캐시 크기와 일치하는지 확인

이 분석을 통해 "L2→L3 경계에서 절벽"이라는 진단이 나오면, 해결책은 워킹셋을 256KB 이하로 줄이는 캐시 블로킹이다.

</details>

---

**Q3.** Capacity miss와 Conflict miss를 cachegrind의 D1/LLd 수치만으로 완전히 구분할 수 있는가? 구분이 어렵다면 어떤 추가 실험으로 둘을 분리할 수 있는가?

<details>
<summary>해설 보기</summary>

**완전한 구분은 어렵다**: cachegrind의 D1은 L1 기준 미스 (Compulsory + Capacity + Conflict 합계)이고, LLd는 L3를 뚫은 미스입니다. D1 - LLd가 크면 L2/L3에서 해결되는 미스가 많다는 것을 알지만 Capacity인지 Conflict인지는 알 수 없습니다.

**추가 실험으로 구분**:

**방법 1 — 배열 크기 변경으로 Conflict 확인**:
- 현재 배열 크기 N에서 N+1, N+3, N+7로 바꿔 재측정
- D1 miss가 크게 감소하면 → Conflict miss (크기가 Set 매핑 패턴을 깼음)
- D1 miss가 거의 안 변하면 → Capacity miss (크기와 무관한 용량 문제)

**방법 2 — 완전 연관 시뮬레이션**:
```bash
# cachegrind의 연관도를 인위적으로 높여서 Conflict를 제거
valgrind --tool=cachegrind --D1=32768,256,64 ./myprogram
# 256-way로 설정하면 Conflict가 사실상 0
# 이 때와 실제 8-way(--D1=32768,8,64)의 D1 miss 차이
# = Conflict miss 기여분
```

**방법 3 — 워킹셋 크기 vs 미스율 그래프**:
- 워킹셋이 캐시 크기보다 훨씬 작은데 미스율이 높으면 → Conflict
- 워킹셋이 캐시 크기를 막 초과할 때 미스율이 높으면 → Capacity

실무에서는 방법 1이 가장 간단하고 직관적입니다. 배열 크기를 +7 해서 미스가 절반으로 줄면 Conflict가 주요 원인이고, 그렇지 않으면 Capacity를 의심한다.

</details>

---

<div align="center">

**[⬅️ 이전: 지역성](./03-locality.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데이터 구조와 캐시 ➡️](./05-data-layout-cache.md)**

</div>
