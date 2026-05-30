# 데이터 구조와 캐시 — AoS vs SoA, 정렬, 패딩

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Array-of-Structs(AoS)와 Struct-of-Arrays(SoA)의 캐시 효율 차이는 어디서 오는가?
- SoA가 SIMD 자동 벡터화(Ch5)에 유리한 이유는 무엇인가?
- `alignas(64)`로 캐시라인에 정렬하는 것이 왜 성능에 영향을 주는가?
- 구조체 멤버 순서를 바꾸면 `sizeof`가 달라지는 패딩 규칙은 무엇인가?
- Hot/Cold 필드 분리 전략이란 무엇이고 언제 적용해야 하는가?
- False Sharing(Ch3-02)과 데이터 레이아웃의 관계는 무엇인가?

---

## 🔍 왜 이 개념이 중요한가

### "같은 루프인데 구조체를 어떻게 정의하느냐에 따라 3배 차이?"

```
게임 엔진 물리 시뮬레이션 — N개 입자의 위치 업데이트:

// 방법 A: AoS (Array of Structs)
struct Particle {
    float x, y, z;       // 위치 (12B)
    float vx, vy, vz;    // 속도 (12B)
    float mass;          // 질량 (4B)
    int   id;            // 식별자 (4B)
    char  name[32];      // 이름 (32B)  ← 업데이트 시 불필요!
};                       // sizeof = 64B (캐시라인 1개)

Particle particles[100000];

for (int i = 0; i < 100000; i++) {
    particles[i].x  += particles[i].vx;
    particles[i].y  += particles[i].vy;
    particles[i].z  += particles[i].vz;
}
// 각 Particle = 64B = 캐시라인 1개
// 필요한 데이터: x,y,z,vx,vy,vz = 24B
// 불필요한 로드: mass,id,name = 40B
// 캐시 효율: 24/64 = 37.5%  ← 60% 낭비

// 방법 B: SoA (Struct of Arrays)
struct ParticlePool {
    float x[100000], y[100000], z[100000];
    float vx[100000], vy[100000], vz[100000];
    float mass[100000];
    int   id[100000];
    char  name[100000][32];   // 업데이트 루프에서 아예 접근 안 함
};

ParticlePool pool;

for (int i = 0; i < 100000; i++) {
    pool.x[i]  += pool.vx[i];
    pool.y[i]  += pool.vy[i];
    pool.z[i]  += pool.vz[i];
}
// x[0..15] = 캐시라인 1개, vx[0..15] = 캐시라인 1개
// name 배열은 아예 접근 안 함 → 캐시 오염 없음
// 캐시 효율: 64/64 = 100%
// SIMD 자동 벡터화 가능: x[i]~x[i+7]을 AVX2로 한 번에 처리

실측: N=100,000 기준
  AoS: ~2.1ms
  SoA: ~0.7ms
  → 3배 차이 (같은 알고리즘, 데이터 레이아웃만 다름)
```

---

## 😱 잘못된 이해

### Before: "구조체는 논리적으로 관련된 데이터를 묶는 것 — 성능은 고려 대상이 아니다"

```
일반적인 사고:
  struct Player {
      float x, y, z;
      int   health;
      char  name[64];
      int   score;
      bool  alive;
  };
  // "플레이어 데이터를 하나의 단위로 관리" = 좋은 설계

  이 사고가 놓치는 것:
    ① 업데이트 루프: x, y, z, vx, vy, vz만 접근
       → health, name, score, alive가 같은 캐시라인에 끌려 올라옴
       → 사용하지 않는 데이터로 캐시 오염
    ② SIMD 벡터화 불가:
       x[0],x[1],...,x[7]이 연속 메모리가 아님
       → AVX2로 8개 float을 한 번에 처리 불가
    ③ 멀티스레드 False Sharing:
       스레드 A가 player[0].x를 씀
       스레드 B가 player[1].health를 씀
       → 같은 캐시라인 → MESI 무효화 폭풍 (Ch3-02)

  "구조체 = 논리적 단위"는 맞지만
  "구조체 = 항상 좋은 캐시 설계"는 틀렸다
```

---

## ✨ 올바른 이해

### After: 접근 패턴에 따라 AoS vs SoA를 선택하고, 정렬과 패딩을 의식적으로 설계한다

```
핵심 원칙:
  함께 접근되는 데이터 → 함께 배치
  따로 접근되는 데이터 → 따로 배치

AoS가 유리한 경우:
  - 하나의 객체 전체를 한 번에 처리할 때
    (예: 단일 플레이어 정보 조회, JSON 직렬화)
  - 객체 수가 적고 랜덤 접근이 주요 패턴일 때
  - 코드 가독성이 성능보다 중요할 때

SoA가 유리한 경우:
  - 모든 객체의 일부 필드만 반복 처리할 때
    (예: 물리 시뮬레이션, 렌더링 파이프라인)
  - SIMD 자동 벡터화를 활용할 때 (Ch5)
  - 멀티스레드로 서로 다른 필드를 처리할 때 (False Sharing 방지)

정렬 설계 원칙:
  캐시라인 경계(64B)에 정렬 → 캐시라인 분할 방지
  SIMD 레지스터(32B/64B)에 정렬 → 정렬 로드 명령 사용 가능
  자연 정렬(Natural Alignment) → CPU가 요구하는 최소 정렬

패딩 설계 원칙:
  컴파일러가 자동 삽입하는 패딩을 이해하고 통제
  멤버를 크기 내림차순으로 배치 → 패딩 최소화
  캐시라인 경계에 구조체 정렬 → False Sharing 방지
```

---

## 🔬 내부 동작 원리

### 1. AoS vs SoA — 메모리 레이아웃 비교

```
AoS 메모리 레이아웃 (N=4 입자 예시):

주소:  [0x000]       [0x040]       [0x080]       [0x0C0]
       ├──────────────┤──────────────┤──────────────┤──────────────┤
       │ P0           │ P1           │ P2           │ P3           │
       │ x y z vx vy │ x y z vx vy │ x y z vx vy │ x y z vx vy │
       │ vz mass id  │ vz mass id  │ vz mass id  │ vz mass id  │
       └──────────────┘──────────────┘──────────────┘──────────────┘
       캐시라인 1       캐시라인 2       캐시라인 3       캐시라인 4

       x 필드만 처리 시: P0.x, P1.x, P2.x, P3.x
       → 4개 캐시라인 로드 (각 라인에서 4B만 사용)
       → 효율: 4×4B used / 4×64B loaded = 6.25%

SoA 메모리 레이아웃:

주소:  [0x000]                  [0x010]
       ├────────────────────────┤────────────────────────┤
       │ x[0] x[1] x[2] x[3]  │ y[0] y[1] y[2] y[3]  │
       │ x[4] x[5] x[6] x[7]  │ ...                    │
       └────────────────────────┘
       x 배열 캐시라인 1          y 배열 캐시라인 1

       x 필드만 처리 시: x[0]~x[15] → 캐시라인 1개로 16개 처리!
       → 효율: 64B used / 64B loaded = 100%

SIMD 처리 비교:
  AoS + AVX2: x[0],x[1],...x[7]이 64B 간격 → gather 명령 필요 (느림)
  SoA + AVX2: x[0..7]이 연속 → vmovaps 명령 1개로 8 float 로드 (빠름)

  // AoS에서 SIMD 강제 시도 (비효율)
  // x[i]를 모으려면: _mm256_i32gather_ps(&particles[0].x, ...)
  // gather: 불연속 주소에서 모아오기 — 직접 로드보다 3~5배 느림

  // SoA에서 SIMD 자연스러운 활용
  __m256 vx = _mm256_load_ps(&pool.vx[i]);   // vx[i]~vx[i+7] 8개 한번에
  __m256 px = _mm256_load_ps(&pool.x[i]);    // x[i]~x[i+7] 8개 한번에
  px = _mm256_add_ps(px, vx);                // 8개 동시 덧셈
  _mm256_store_ps(&pool.x[i], px);           // 결과 저장
```

### 2. 구조체 패딩 규칙 — sizeof가 달라지는 이유

```c
/*
 * 자연 정렬(Natural Alignment) 규칙:
 *  각 멤버는 자신의 크기의 배수 주소에 위치해야 한다
 *
 *  char   (1B): 모든 주소 OK
 *  short  (2B): 짝수 주소 필요
 *  int    (4B): 4의 배수 주소 필요
 *  double (8B): 8의 배수 주소 필요
 *  pointer(8B): 8의 배수 주소 필요 (64-bit)
 *
 *  구조체 전체 크기 = 가장 큰 멤버의 배수로 정렬
 */

// 패딩 예시 1 — 나쁜 멤버 순서 (16B 낭비)
struct BadLayout {
    char   a;      // 1B  @ offset 0
                   // ··· 7B 패딩 (double 정렬을 위해)
    double b;      // 8B  @ offset 8
    char   c;      // 1B  @ offset 16
                   // ··· 7B 패딩 (구조체 크기를 8의 배수로)
};                 // sizeof = 24B

// 패딩 예시 2 — 좋은 멤버 순서 (패딩 최소화)
struct GoodLayout {
    double b;      // 8B  @ offset 0
    char   a;      // 1B  @ offset 8
    char   c;      // 1B  @ offset 9
                   // ··· 6B 패딩 (구조체 크기를 8의 배수로)
};                 // sizeof = 16B  ← BadLayout 대비 8B 절약

// 패딩 예시 3 — 전형적인 혼합 구조체
struct Mixed {
    char   flag;    // 1B  @ offset 0
                    // 3B 패딩
    int    id;      // 4B  @ offset 4
    char   type;    // 1B  @ offset 8
                    // 7B 패딩
    double value;   // 8B  @ offset 16
    short  count;   // 2B  @ offset 24
                    // 6B 패딩
};                  // sizeof = 32B (실제 데이터 16B, 패딩 16B = 50% 낭비)

// 최적화 버전
struct MixedOpt {
    double value;   // 8B  @ offset 0
    int    id;      // 4B  @ offset 8
    short  count;   // 2B  @ offset 12
    char   flag;    // 1B  @ offset 14
    char   type;    // 1B  @ offset 15
};                  // sizeof = 16B (패딩 0B!)

// 확인 방법
#include <stddef.h>
printf("BadLayout   = %zu\n", sizeof(struct BadLayout));    // 24
printf("GoodLayout  = %zu\n", sizeof(struct GoodLayout));   // 16
printf("Mixed       = %zu\n", sizeof(struct Mixed));        // 32
printf("MixedOpt    = %zu\n", sizeof(struct MixedOpt));     // 16

// 멤버 오프셋 확인 (패딩 위치 파악)
printf("Mixed.flag  offset = %zu\n", offsetof(struct Mixed, flag));
printf("Mixed.value offset = %zu\n", offsetof(struct Mixed, value));
```

### 3. `alignas(64)` — 캐시라인 경계 정렬

```c
#include <stdalign.h>  // C11: alignas
// 또는 #include <immintrin.h>  // x86 SIMD: __attribute__((aligned(64)))

/*
 * 캐시라인 정렬이 필요한 이유:
 *
 * 미정렬 구조체의 문제:
 *  struct Foo { int a[16]; };  // 64B
 *  Foo foo;  // foo가 0x38에 배치된다면:
 *
 *  캐시라인 경계: 0x00, 0x40, 0x80, ...
 *  0x38: 캐시라인 1의 마지막 8B + 캐시라인 2의 처음 56B에 걸침
 *  → foo 접근 시 캐시라인 2개 로드 필요
 *  → 캐시 공간 2배 점유, 로드 1회 추가
 *
 * 정렬된 구조체:
 *  Foo가 0x40에 배치된다면:
 *  → 캐시라인 1개에 완전히 수용
 *  → 캐시 공간 1배 점유
 */

// 방법 1: alignas (C11/C++11)
struct alignas(64) CacheAligned {
    float data[16];   // 64B 정확히
};
CacheAligned aligned_array[1000];
// 첫 원소가 64B 경계에 정렬 → 각 원소가 독립 캐시라인
// False Sharing 완전 방지 (Ch3-02 연결)

// 방법 2: __attribute__((aligned(64))) — GCC/Clang 확장
struct HotData {
    float x, y, z;    // 핵심 처리 데이터
    float vx, vy, vz;
    float pad[10];    // 64B 채움 패딩
} __attribute__((aligned(64)));

// 방법 3: 동적 할당 시 정렬
float* simd_buf = (float*)aligned_alloc(64, N * sizeof(float));
// 또는
float* simd_buf2 = NULL;
posix_memalign((void**)&simd_buf2, 64, N * sizeof(float));

// SIMD 정렬 로드 vs 미정렬 로드 (성능 차이)
// 정렬된 경우: _mm256_load_ps(ptr)   — ptr이 32B 경계일 때만 사용 가능
// 미정렬:     _mm256_loadu_ps(ptr)  — 항상 사용 가능하지만 구형 CPU에서 느림

// 정렬 확인
printf("HotData 정렬: %zu\n", __alignof__(struct HotData));   // 64
printf("포인터 정렬:  %zu\n", (uintptr_t)simd_buf % 64);     // 0 이면 OK
```

### 4. Hot/Cold 필드 분리 전략

```c
/*
 * Hot/Cold 분리:
 *  자주 접근되는 필드(Hot) = 캐시라인 앞부분에
 *  드물게 접근되는 필드(Cold) = 캐시라인 뒤 또는 별도 구조체에
 *
 *  이유: 캐시라인 1개(64B)가 로드될 때
 *        Hot 필드가 앞에 있으면 1개의 캐시라인만으로 작업 완료
 *        Cold 필드가 섞이면 동일 캐시라인에 불필요한 데이터 로드
 */

// 예: 해시맵 노드
// 조회 시 key와 hash 값만 빠르게 비교 → Hot
// 값 전체 반환은 조회 성공 후에만 필요 → Cold

// Bad: Hot/Cold 혼합
struct BadMapNode {
    uint64_t hash;      // 8B  Hot: 빠른 비교에 필요
    char     key[64];   // 64B Cold: 실제 키 비교 (드물게)
    void*    value;     // 8B  Cold: 매치 성공 시에만 반환
    uint32_t flags;     // 4B  Cold
};  // sizeof = 88B → 캐시라인 2개에 걸침
    // hash 비교 시: 88B 전체 로드 (key, value 불필요)

// Good: Hot/Cold 분리
struct GoodMapNode {
    // === 첫 캐시라인 (Hot) ===
    uint64_t hash;       // 8B  빠른 비교
    uint64_t key_len;    // 8B  키 길이 (비교 조건)
    char     short_key[48]; // 48B 짧은 키 인라인 저장
    // total: 64B = 캐시라인 1개
    // === 두 번째 캐시라인 (Cold) ===
    char*    long_key;   // 8B  긴 키는 포인터로 (드물게 접근)
    void*    value;      // 8B  매치 성공 후에만 접근
    uint32_t flags;      // 4B
    uint32_t _pad;       // 4B
} __attribute__((aligned(64)));

// 조회 루프에서의 효과:
// - hash 비교: 첫 캐시라인만 로드 (64B)
// - 매치 실패: 두 번째 캐시라인 불필요 (Cold 데이터 캐시 미오염)
// - 매치 성공 (드물게): 두 번째 캐시라인 로드 (허용)

// Linux 커널의 Hot/Cold 분리 패턴
// (task_struct가 대표적 예시 — hot 필드를 앞 64B에 집중)
struct task_struct {
    // Hot — 스케줄러가 매 컨텍스트 스위치마다 접근
    volatile long state;        // 실행 상태
    void* stack;                // 커널 스택
    atomic_t usage;
    // ...
    // Cold — 드물게 접근 (디버깅, 통계 등)
    // 파일 핸들, 신호 핸들러 등 (수백 바이트 뒤에 배치)
};
```

### 5. 패딩 확인 도구 — `pahole`

```bash
# pahole: 구조체 레이아웃 시각화 도구
# 설치: apt-get install dwarves  (Ubuntu)

# 바이너리의 구조체 패딩 분석
gcc -g -O0 -o myprogram myprogram.c
pahole myprogram

# 예시 출력:
# struct BadLayout {
#         char               a;       /*    0     1 */
#
#         /* XXX 7 bytes hole, try to pack */
#
#         double             b;       /*    8     8 */
#         char               c;       /*   16     1 */
#
#         /* XXX 7 bytes hole, try to pack */
#
# }; /* size: 24, cachelines: 1, members: 3 */
#    /* sum members: 10, holes: 2, sum holes: 14 */
#    /* padding: 7, data size: 17 */

# 특정 구조체만 분석
pahole --class_name=MyStruct myprogram

# 전체 프로그램의 패딩 통계
pahole --packable myprogram  # 패딩 낭비가 많은 구조체 목록
```

---

## 💻 실전 실험

### 실험 1: AoS vs SoA 성능 비교

```c
// aos_soa_bench.c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#define N 1000000  // 1M 입자

// AoS
typedef struct {
    float x, y, z;
    float vx, vy, vz;
    float mass;
    int   id;
    char  _cold[32];  // 업데이트 시 불필요한 콜드 데이터
} ParticleAoS;

// SoA
typedef struct {
    float  x[N], y[N], z[N];
    float  vx[N], vy[N], vz[N];
    float  mass[N];
    int    id[N];
    char   _cold[N][32];
} ParticleSoA;

ParticleAoS particles_aos[N];
ParticleSoA particles_soa;

long bench_aos(int reps) {
    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int r = 0; r < reps; r++)
        for (int i = 0; i < N; i++) {
            particles_aos[i].x += particles_aos[i].vx;
            particles_aos[i].y += particles_aos[i].vy;
            particles_aos[i].z += particles_aos[i].vz;
        }
    clock_gettime(CLOCK_MONOTONIC, &t1);
    return (t1.tv_sec - t0.tv_sec) * 1000 +
           (t1.tv_nsec - t0.tv_nsec) / 1000000;
}

long bench_soa(int reps) {
    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int r = 0; r < reps; r++)
        for (int i = 0; i < N; i++) {
            particles_soa.x[i] += particles_soa.vx[i];
            particles_soa.y[i] += particles_soa.vy[i];
            particles_soa.z[i] += particles_soa.vz[i];
        }
    clock_gettime(CLOCK_MONOTONIC, &t1);
    return (t1.tv_sec - t0.tv_sec) * 1000 +
           (t1.tv_nsec - t0.tv_nsec) / 1000000;
}

int main() {
    // 초기화
    for (int i = 0; i < N; i++) {
        particles_aos[i].x = particles_soa.x[i] = (float)i;
        particles_aos[i].vx = particles_soa.vx[i] = 1.0f;
        particles_aos[i].y = particles_soa.y[i] = 0.0f;
        particles_aos[i].vy = particles_soa.vy[i] = 0.5f;
        particles_aos[i].z = particles_soa.z[i] = 0.0f;
        particles_aos[i].vz = particles_soa.vz[i] = 0.1f;
    }

    printf("sizeof(ParticleAoS) = %zu\n", sizeof(ParticleAoS));

    // 워밍업
    bench_aos(1); bench_soa(1);

    printf("AoS: %ld ms\n", bench_aos(5));
    printf("SoA: %ld ms\n", bench_soa(5));
    // 예상: AoS ~80ms, SoA ~25ms (약 3배 차이)
    return 0;
}
// gcc -O2 -o aos_soa aos_soa_bench.c && ./aos_soa
// gcc -O3 -mavx2 -o aos_soa_vec aos_soa_bench.c && ./aos_soa_vec
// -O3 + -mavx2: SoA는 SIMD 자동 벡터화 → SoA가 더욱 빨라짐
```

```bash
# perf로 캐시 미스 비교
perf stat -e cycles,L1-dcache-load-misses,LLC-load-misses ./aos_soa

# 자동 벡터화 확인 (godbolt 또는 로컬)
gcc -O3 -mavx2 -S -masm=intel aos_soa_bench.c -o aos_soa.s
grep -E "ymm|vmovaps|vaddps" aos_soa.s
# SoA 루프에서 ymm 레지스터(AVX2) 사용 확인
# AoS 루프에서는 스칼라 또는 gather 명령 사용 → 효율 낮음
```

### 실험 2: 구조체 패딩 실험 — 멤버 순서 변경 전후 sizeof

```c
// padding_demo.c
#include <stdio.h>
#include <stddef.h>

struct S1 { char a; int b; char c; };         // 12B (패딩 6B)
struct S2 { int b; char a; char c; };         // 8B  (패딩 2B)
struct S3 { char a; char c; int b; };         // 8B  (패딩 2B)

struct S4 { char a; double b; int c; char d; };  // 24B
struct S5 { double b; int c; char a; char d; };  // 16B

int main() {
    printf("S1 { char, int, char }      = %2zu B  (holes: %zu)\n",
           sizeof(struct S1),
           sizeof(struct S1) - (1+4+1));
    printf("S2 { int, char, char }      = %2zu B  (holes: %zu)\n",
           sizeof(struct S2),
           sizeof(struct S2) - (4+1+1));
    printf("S3 { char, char, int }      = %2zu B  (holes: %zu)\n",
           sizeof(struct S3),
           sizeof(struct S3) - (1+1+4));
    printf("\n");
    printf("S4 { char, double, int, char } = %2zu B\n", sizeof(struct S4));
    printf("S5 { double, int, char, char } = %2zu B\n", sizeof(struct S5));
    printf("\n");

    // 각 멤버의 오프셋 출력
    printf("S4 멤버 오프셋:\n");
    printf("  .a = %zu\n", offsetof(struct S4, a));
    printf("  .b = %zu\n", offsetof(struct S4, b));
    printf("  .c = %zu\n", offsetof(struct S4, c));
    printf("  .d = %zu\n", offsetof(struct S4, d));
    // S4 출력:
    // .a = 0 (1B 사용 + 7B 패딩)
    // .b = 8 (8B 사용)
    // .c = 16 (4B 사용)
    // .d = 20 (1B 사용 + 3B 패딩)
    // sizeof = 24
    return 0;
}
// gcc -o padding padding_demo.c && ./padding
```

### 실험 3: `alignas(64)` 정렬 효과 — False Sharing 방지

```c
// alignment_bench.c
// 캐시라인 정렬이 멀티스레드 False Sharing을 방지하는 효과
#include <stdio.h>
#include <pthread.h>
#include <stdint.h>
#include <time.h>
#include <stdalign.h>

#define REPS 100000000LL

// 미정렬: 두 카운터가 같은 캐시라인에 있을 수 있음
typedef struct {
    volatile long counter_a;  // 8B
    volatile long counter_b;  // 8B — counter_a와 같은 캐시라인!
} UnalignedCounters;

// 정렬: 각 카운터가 독립적인 캐시라인
typedef struct {
    alignas(64) volatile long counter_a;  // 64B 경계 정렬
    alignas(64) volatile long counter_b;  // 다음 64B 경계
} AlignedCounters;

UnalignedCounters uc = {0, 0};
AlignedCounters   ac = {0, 0};

void* inc_a_unaligned(void* arg) {
    for (long long i = 0; i < REPS; i++) uc.counter_a++;
    return NULL;
}
void* inc_b_unaligned(void* arg) {
    for (long long i = 0; i < REPS; i++) uc.counter_b++;
    return NULL;
}
void* inc_a_aligned(void* arg) {
    for (long long i = 0; i < REPS; i++) ac.counter_a++;
    return NULL;
}
void* inc_b_aligned(void* arg) {
    for (long long i = 0; i < REPS; i++) ac.counter_b++;
    return NULL;
}

long bench(void* (*fa)(void*), void* (*fb)(void*)) {
    pthread_t ta, tb;
    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    pthread_create(&ta, NULL, fa, NULL);
    pthread_create(&tb, NULL, fb, NULL);
    pthread_join(ta, NULL);
    pthread_join(tb, NULL);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    return (t1.tv_sec - t0.tv_sec) * 1000 +
           (t1.tv_nsec - t0.tv_nsec) / 1000000;
}

int main() {
    printf("UnalignedCounters: a=%p b=%p 거리=%td\n",
           &uc.counter_a, &uc.counter_b,
           (char*)&uc.counter_b - (char*)&uc.counter_a);
    printf("AlignedCounters:   a=%p b=%p 거리=%td\n",
           &ac.counter_a, &ac.counter_b,
           (char*)&ac.counter_b - (char*)&ac.counter_a);

    printf("미정렬 (False Sharing 가능): %ld ms\n",
           bench(inc_a_unaligned, inc_b_unaligned));
    printf("정렬 (False Sharing 없음):  %ld ms\n",
           bench(inc_a_aligned, inc_b_aligned));
    // 예상: 미정렬 ~2500ms, 정렬 ~800ms (약 3배 차이)
    // Ch3-02 False Sharing 문서의 측정과 연결
    return 0;
}
// gcc -O2 -pthread -o alignment alignment_bench.c && ./alignment
// perf stat -e cache-misses,cache-references ./alignment
```

---

## 📊 성능 비교

```
AoS vs SoA — N=1,000,000 입자 위치 업데이트 (x+=vx, y+=vy, z+=vz):

  레이아웃   sizeof(단위)  캐시 효율   L1-miss    시간     SIMD
  ─────────────────────────────────────────────────────────────
  AoS        64B           37.5%       높음        ~80ms   불가
  AoS-opt    32B (cold 제거) 75%       중간        ~45ms   제한적
  SoA        4B × N        100%        낮음        ~25ms   자동
  SoA+AVX2   위 + 벡터화   100%        낮음        ~8ms    수동
  ─────────────────────────────────────────────────────────────

구조체 패딩 절약 효과 (N=1,000,000 객체):

  레이아웃    sizeof   메모리 사용   캐시라인 수
  ─────────────────────────────────────────────
  { char, int, char }  12B  12MB   187,500개
  { int, char, char }   8B   8MB   125,000개  ← 33% 절약
  { char, double, int, char } 24B 24MB → { double, int, char, char } 16B 16MB (33% 절약)
  ─────────────────────────────────────────────

캐시라인 정렬 효과 (2스레드 독립 카운터):

  레이아웃           캐시라인 공유   처리량   speedup
  ─────────────────────────────────────────────
  미정렬 카운터      Yes (False Share) ~400M/s   1x
  64B 정렬 카운터    No               ~1200M/s   3x
  ─────────────────────────────────────────────
```

---

## ⚖️ 트레이드오프

```
AoS:
  ✅ 코드 가독성 (단일 객체를 하나의 단위로 다룸)
  ✅ 랜덤 객체 접근 패턴에서 효율적 (객체 전체를 1~2 캐시라인에)
  ✅ 객체 복사, 직렬화, 삭제 단순
  ❌ 배치 처리(batch processing) 시 캐시 낭비 (사용 안 하는 필드 로드)
  ❌ SIMD 자동 벡터화 어려움 (비연속 필드)
  ❌ 멀티스레드 False Sharing 발생 가능

SoA:
  ✅ 배치 처리 캐시 효율 100% (필요한 필드만 로드)
  ✅ SIMD 자동/수동 벡터화 용이 (Ch5-04 연결)
  ✅ 필드별 병렬 처리 시 False Sharing 없음
  ❌ 코드 복잡도 증가 (pool.x[i] vs particle.x)
  ❌ 단일 객체 처리 비효율 (여러 배열을 따로 접근)
  ❌ 객체 삽입/삭제 시 여러 배열을 동기화해야 함

Hot/Cold 분리:
  ✅ 핫 경로의 캐시 효율 극대화 (핫 데이터만 캐시 점유)
  ✅ AoS 유지하면서 부분적 성능 개선 가능
  ❌ 구조체 분리 또는 멤버 순서 재정렬 필요 (리팩토링 비용)
  ❌ 콜드 데이터 접근 시 추가 포인터 역참조 비용

alignas(64):
  ✅ 캐시라인 분할 방지 → 안정적 접근 비용
  ✅ False Sharing 방지 (각 객체가 독립 캐시라인)
  ✅ SIMD 정렬 로드 명령 사용 가능
  ❌ 메모리 낭비: 구조체가 64B 미만이면 패딩 필요
  ❌ 배열 원소가 64B 미만 구조체이면 배열 내 패딩 누적

실무 선택 가이드:
  배치 루프(for i: update all[i]) → SoA
  랜덤 단일 조회(get entity X) → AoS
  혼합 워크로드 → Hot/Cold AoS 또는 AoSoA(하이브리드)
  멀티스레드 공유 카운터 → alignas(64) 필수
```

---

## 📌 핵심 정리

```
데이터 레이아웃과 캐시 핵심:

AoS vs SoA:
  AoS: 객체 단위 접근에 적합 (단일 객체 = 1~2 캐시라인)
  SoA: 배치 처리에 적합 (필요한 필드만 순차 접근)
  배치 루프에서 SoA가 3~8배 빠른 이유:
    → 사용하는 필드만 캐시에 올라옴 (불필요한 Cold 필드 제외)
    → SIMD 자동 벡터화 가능 (Ch5와 연결)

구조체 패딩 규칙:
  각 멤버는 자신 크기의 배수 주소에 배치 (자연 정렬)
  멤버를 크기 내림차순으로 배치 → 패딩 최소화
  pahole 도구로 현재 레이아웃 시각화
  경험 법칙: double/pointer → int → short → char 순서

캐시라인 정렬 (alignas(64)):
  구조체가 2개의 캐시라인에 걸치는 분할(split) 방지
  멀티스레드 환경에서 False Sharing 방지 (Ch3-02 연결)
  SIMD 정렬 로드(_mm256_load_ps)를 위한 32B/64B 정렬

Hot/Cold 분리:
  핫 경로의 캐시라인을 꼭 필요한 데이터로만 채움
  첫 캐시라인(64B)에 핫 필드 집중 → 나머지 접근 횟수 최소화
  Linux kernel task_struct가 대표적 패턴

연결 지점:
  SoA + SIMD: Ch5-04에서 자동 벡터화 연결
  alignas + False Sharing: Ch3-02 MESI 캐시 일관성
  Hot/Cold + 객체 그래프: Ch4-03 포인터 추적 비용
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 구조체의 `sizeof`를 계산하고, 멤버 순서를 바꿔 `sizeof`를 최소화하라. 최소화 후 `sizeof`는 얼마인가?

```c
struct Original {
    char   flag;     // 1B
    double price;    // 8B
    int    count;    // 4B
    char   type;     // 1B
    short  code;     // 2B
};
```

<details>
<summary>해설 보기</summary>

**Original의 sizeof 계산**:
- `char flag` (1B) @ offset 0 → 7B 패딩 (double 정렬 위해)
- `double price` (8B) @ offset 8
- `int count` (4B) @ offset 16
- `char type` (1B) @ offset 20 → 1B 패딩 (short 정렬 위해)
- `short code` (2B) @ offset 22
- 구조체 끝 @ offset 24 → double(8B)의 배수로 맞춤
- **sizeof(Original) = 24B** (실제 데이터 = 1+8+4+1+2 = 16B, 패딩 = 8B)

**최적화 순서 (크기 내림차순)**:
```c
struct Optimized {
    double price;    // 8B @ offset 0
    int    count;    // 4B @ offset 8
    short  code;     // 2B @ offset 12
    char   flag;     // 1B @ offset 14
    char   type;     // 1B @ offset 15
};               // sizeof = 16B (패딩 0B!)
```

최소화 결과: **sizeof(Optimized) = 16B** — 원래보다 8B(33%) 절약.

N=1,000,000개의 배열이라면: 24MB → 16MB로 줄어들고, 캐시라인 375,000개 → 250,000개로 감소합니다.

</details>

---

**Q2.** 게임 엔진에서 10,000개 엔티티를 관리하는 시스템이 있다. 매 프레임마다 물리 업데이트(위치, 속도)는 모든 엔티티에 적용되지만, 렌더링 데이터(색상, 텍스처 ID, 메시 ID)는 카메라 뷰에 들어온 엔티티(~500개)에만 접근된다. 어떤 데이터 레이아웃 전략이 최적인가?

<details>
<summary>해설 보기</summary>

**최적 전략: Hybrid AoSoA + Hot/Cold 분리**

물리 업데이트(모든 엔티티, 매 프레임) → SoA:
```c
struct PhysicsSoA {
    float x[10000], y[10000], z[10000];
    float vx[10000], vy[10000], vz[10000];
    float mass[10000];
};
// 모든 엔티티를 순차 처리 → 공간 지역성 최대
// SIMD로 8개 엔티티 동시 처리 가능
```

렌더링 데이터(500개만, 간헐적 접근) → 별도 AoS:
```c
struct RenderData {
    uint32_t  color;
    uint32_t  texture_id;
    uint32_t  mesh_id;
    uint8_t   lod_level;
    uint8_t   _pad[3];
};
RenderData render[10000];  // 또는 visible 엔티티만
```

이 전략의 이유:
1. **물리 SoA**: 10,000개 모두 매 프레임 접근 → 순차 접근의 공간 지역성 필수 → SIMD 가능
2. **렌더 AoS 분리**: 렌더 데이터를 물리 SoA에 섞으면 물리 루프에서 렌더 데이터가 캐시를 오염시킴 → 분리하여 물리 루프의 캐시 효율 보호
3. **렌더는 AoS 허용**: 500개를 랜덤 순서로 접근 → SoA보다 AoS가 단일 엔티티 조회에 더 적합

추가 최적화: 가시성(visible) 엔티티 인덱스를 별도 배열로 유지하여 렌더 루프도 간접 접근 최소화.

</details>

---

**Q3.** `alignas(64)`로 캐시라인 경계 정렬된 구조체의 배열을 만들 때, 구조체 크기가 40B라면 실제 메모리 레이아웃은 어떻게 되며, 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

**문제 분석**:

`alignas(64)`가 지정된 40B 구조체의 배열:
```c
struct alignas(64) MyStruct {
    char data[40];   // 실제 40B
    char _pad[24];   // 컴파일러가 64B로 맞추기 위해 24B 패딩 추가
};                   // sizeof = 64B
```

`sizeof(MyStruct) = 64B`가 됩니다. 따라서:
- `MyStruct arr[1000]`의 실제 메모리 = 64KB (vs 정렬 없이 40KB)
- **메모리 낭비**: 24B × 1000 = 24KB (60% 낭비)

**장점과 단점의 균형**:
- 장점: 각 원소가 독립 캐시라인 → False Sharing 완전 방지 → 멀티스레드에서 극적 성능 향상
- 단점: 메모리 사용량 60% 증가 → 캐시 수용 가능한 원소 수 감소

**실무 판단 기준**:
1. **멀티스레드에서 서로 다른 원소를 병렬로 씀** → alignas(64) 필수 (False Sharing 비용이 메모리 낭비보다 큼)
2. **단일 스레드 또는 읽기 전용 배열** → 불필요, 메모리 낭비만
3. **자연 패킹 최적화**: 40B 구조체를 32B 또는 64B 정확히 맞게 리팩토링하면 낭비 없이 정렬 달성 가능

**해결책**: 구조체 크기를 정확히 64B로 맞추도록 리팩토링 (필요한 Cold 필드를 제거하거나, Hot 필드를 늘려 64B를 꽉 채우는 설계).

</details>

---

<div align="center">

**[⬅️ 이전: 캐시 미스의 종류](./04-cache-miss-types.md)** | **[홈으로 🏠](../README.md)** | **[다음: TLB와 가상 메모리 ➡️](./06-tlb-virtual-memory.md)**

</div>
