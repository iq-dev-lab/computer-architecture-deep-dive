# 데이터 레이아웃과 벡터화 — SoA가 유리한 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- AoS(Array of Structs)가 SIMD 로드를 막는 구조적 이유는 무엇인가?
- SoA(Struct of Arrays)가 한 번의 `vmovups ymm`으로 8요소를 로드할 수 있는 이유는?
- 입자 시뮬레이션과 픽셀 처리에서 SoA가 사실상 표준이 된 역사적 배경은?
- AoS에서 SoA로 전환 없이 SIMD 효과를 얻으려면 어떤 명령어를 써야 하는가?
- 분기 한 줄이 SIMD 벡터화를 깨는 메커니즘은 Ch2-05의 어떤 원리와 연결되는가?
- AoSoA (Array of Structs of Arrays) 패턴은 언제 사용하는가?

---

## 🔍 왜 이 개념이 중요한가

### "자료구조 설계가 SIMD 성능을 결정한다"

```
같은 입자 시뮬레이션, 다른 레이아웃:

AoS 방식 (직관적, OOP 스타일):
  struct Particle {
      float x, y, z;    // 위치 (12B)
      float vx, vy, vz; // 속도 (12B)
      float mass;        // 질량 (4B)
      float pad;         // 패딩 (4B)
  };                     // 합계: 32B
  Particle particles[N];

  Update 루프:
  for (int i = 0; i < N; i++) {
      particles[i].x += particles[i].vx * dt;
      particles[i].y += particles[i].vy * dt;
  }

  메모리 접근 패턴:
  [x y z vx vy vz m p] [x y z vx vy vz m p] ...
   ↑이것을 YMM로 로드하면 x, y, z, vx, vy, vz, m, p 섞임
   → x 8개를 YMM으로 로드 불가! (비연속)

SoA 방식 (SIMD 친화적):
  struct ParticlePool {
      float x[N], y[N], z[N];
      float vx[N], vy[N], vz[N];
      float mass[N];
  };

  Update 루프:
  for (int i = 0; i < N; i++) {
      pool.x[i]  += pool.vx[i] * dt;
      pool.y[i]  += pool.vy[i] * dt;
  }

  메모리 접근 패턴:
  x[]: [x0 x1 x2 x3 x4 x5 x6 x7] [x8 x9...]
        → vmovups ymm ← 8개 x를 한 번에 로드! ✅

성능 차이:
  N=1M 입자 업데이트:
  AoS: ~12ms  (stride 접근으로 캐시 낭비 + SIMD 불가)
  SoA:  ~1.5ms (순차 접근 + AVX2 8× 처리량)
  → 8× 차이, 전부 레이아웃에서 발생
```

---

## 😱 잘못된 이해

### Before: "OOP 구조체 = 데이터 응집 = 항상 캐시 친화적"

```
잘못된 이해:
  struct Particle { float x, y, z, vx, vy, vz; }
  → x, vx가 같은 구조체 = 항상 인접 메모리 = 캐시 히트

  "OOP는 관련 데이터를 묶으므로 캐시 효율적이다"

놓친 것 1 — SIMD 로드의 연속성 요건:
  SIMD는 "같은 필드 N개"를 한 번에 처리
  x 8개: x[0]~x[7] → 이것이 연속 메모리여야 함
  AoS: x[0] = 0번지, x[1] = 32번지 (다음 구조체의 x)
       → 8 × 32byte = 256 bytes stride
       → 한 YMM 레지스터에 x 8개 = 불연속 gather 필요
       → scatter-gather: 성능 손실 10~20배

놓친 것 2 — 실제로 접근하는 필드:
  Update 루프: x, vx 만 접근 (2개 필드)
  AoS 캐시라인에는: x, y, z, vx, vy, vz, mass, pad (8개 필드)
  → 불필요한 y, z, vy, vz, mass도 캐시라인에 올라옴
  → 캐시 효율: 2/8 = 25%만 사용 (75% 낭비)

놓친 것 3 — "응집"의 오해:
  응집(cohesion)이 유익한 패턴:
    하나의 struct를 통째로 처리할 때 (객체 하나씩 업데이트)
  응집이 해로운 패턴:
    N개 struct의 같은 필드를 일괄 처리할 때 (SIMD, 벡터화)
  → 패턴에 따라 AoS/SoA 선택
```

---

## ✨ 올바른 이해

### After: 데이터 레이아웃은 접근 패턴에 맞춰 설계해야 한다

```
핵심 원칙:
  같은 연산을 N개 요소에 일괄 적용 → SoA가 유리
  한 객체의 여러 필드를 함께 처리 → AoS가 유리

AoS 메모리 레이아웃:
  [ x0 y0 z0 vx0 vy0 vz0 ] [ x1 y1 z1 vx1 vy1 vz1 ] ...
    ↑─────── 32B ──────────┘  ↑─────── 32B ──────────┘

  x 값들의 주소: 0, 32, 64, 96, ... (stride = 32B = struct 크기)
  → 8개 x를 YMM 로드하려면 8개 주소 gather: 비효율

SoA 메모리 레이아웃:
  x[]: [ x0 x1 x2 x3 x4 x5 x6 x7 ] [ x8 x9 ... ]
  y[]: [ y0 y1 y2 y3 y4 y5 y6 y7 ] [ y8 y9 ... ]

  x 값들의 주소: 0, 4, 8, 12, ... (stride = 4B = float 크기)
  → 연속 메모리 → vmovups ymm 단일 명령으로 8개 로드 ✅
  → + 하드웨어 프리페처가 stride-1 패턴 인식 → 자동 프리페치

SIMD 처리량 비교:
  AoS: gather 명령 + 재배치 → ~8~16 cy/8요소
  SoA: vmovups + 바로 연산 → ~1~2 cy/8요소
```

---

## 🔬 내부 동작 원리

### 1. AoS에서 SIMD 로드가 실패하는 구조

```c
// AoS 구조체
struct Particle {
    float x, y, z;    // 오프셋: 0, 4, 8
    float vx, vy, vz; // 오프셋: 12, 16, 20
};
// sizeof(Particle) = 24 bytes

Particle particles[8];

// x 값 8개를 YMM으로 로드하려면?
// particles[0].x = 주소 base + 0
// particles[1].x = 주소 base + 24
// particles[2].x = 주소 base + 48
// particles[3].x = 주소 base + 72
// ...

// vmovups ymm, [base]: 이 명령은 base에서 연속 32 bytes 로드
// → [x0, y0, z0, vx0, vy0, vz0, x1, y1] 이 로드됨 ← x만 아님!

// 올바르게 x 8개를 모으려면 gather:
__m256i vindex = _mm256_set_epi32(
    7*6, 6*6, 5*6, 4*6, 3*6, 2*6, 1*6, 0   // 각 particle x의 float 오프셋
);
__m256 vx = _mm256_i32gather_ps(&particles[0].x, vindex, 4);
// _mm256_i32gather_ps: 8개 다른 주소에서 float 수집 (scatter-gather)
// 인자: base_addr, vindex(각 요소 오프셋), scale(byte)

// gather 명령의 비용 (Skylake):
//   _mm256_i32gather_ps: Latency ~13 cy, Throughput ~8 cy
//   vs vmovups:          Latency  4 cy, Throughput 0.5 cy
//   → 16~26배 더 느림!
```

### 2. SoA에서 SIMD 로드가 최적인 이유

```c
// SoA 구조체
struct ParticlePool {
    float* x;   // x[0], x[1], ..., x[N-1] 연속
    float* y;
    float* z;
    float* vx;
    float* vy;
    float* vz;
};

// Update: 8개 입자를 AVX2로 한 번에
void update_avx2(ParticlePool* p, float dt, int n) {
    __m256 vdt = _mm256_set1_ps(dt);   // dt를 8개로 복제

    for (int i = 0; i <= n - 8; i += 8) {
        // x 8개 연속 로드 (vmovups ymm)
        __m256 vx  = _mm256_loadu_ps(p->x  + i);
        __m256 vvx = _mm256_loadu_ps(p->vx + i);

        // x += vx * dt (8개 동시)
        __m256 new_x = _mm256_fmadd_ps(vvx, vdt, vx);
        _mm256_storeu_ps(p->x + i, new_x);

        // y도 동일
        __m256 vy  = _mm256_loadu_ps(p->y  + i);
        __m256 vvy = _mm256_loadu_ps(p->vy + i);
        _mm256_storeu_ps(p->y + i, _mm256_fmadd_ps(vvy, vdt, vy));
    }
    // 나머지 처리 (epilogue)
    for (int i = (n & ~7); i < n; i++) {
        p->x[i] += p->vx[i] * dt;
        p->y[i] += p->vy[i] * dt;
    }
}

// 메모리 접근 패턴:
//  x[]: ──────────────────────→ 순차, 프리페치 가능
//  vx[]: ─────────────────────→ 순차, 프리페치 가능
//  → L1 스트림 프리페처 완전 활용
//  → 불필요한 y, z, vy, vz 필드 캐시 오염 없음
```

### 3. AoS → SoA 전환 패턴 (게임 엔진의 ECS)

```c
// 기존 AoS (OOP 스타일):
std::vector<Particle> particles;  // 각 Particle에 모든 데이터

// 신규 SoA (ECS 스타일, 컴포넌트 분리):
struct PositionComponent { float* x; float* y; float* z; };
struct VelocityComponent { float* vx; float* vy; float* vz; };
struct MassComponent     { float* mass; };

// Entity = 단순 ID
typedef int Entity;

// 시스템(System): 같은 컴포넌트를 N개 일괄 처리
void MovementSystem(PositionComponent& pos, VelocityComponent& vel,
                    float dt, int n) {
    // AVX2로 벡터화 가능
    for (int i = 0; i < n; i += 8) {
        __m256 vx = _mm256_loadu_ps(pos.x + i);
        // ...
    }
}

// 렌더링 시스템은 Position + 색상만 읽음 → Mass 접근 없음
// 물리 시스템은 Position + Velocity + Mass → 전부 접근

// ECS의 캐시 이점:
// 렌더링 루프: pos.x[], pos.y[], pos.z[] 만 접근
//   → 3 × N × 4B = 3 × 1M × 4B = 12MB → L3 내
//   AoS였다면: 전체 struct 로드 → 불필요한 vel, mass도 캐시에 올라옴
//   → 12MB vs 32MB+ = 캐시 점유 2.6배 차이
```

### 4. 분기 한 줄이 벡터화를 깨는 메커니즘 (Ch2-05 연결)

```c
// AoS → 분기가 더 자주 발생하는 패턴

// 예: 활성 입자만 업데이트
struct Particle {
    float x, y, z;
    bool  active;     // ← 상태 플래그가 같은 구조체에
};

// AoS 업데이트 루프
for (int i = 0; i < N; i++) {
    if (particles[i].active)   // ← 분기! 벡터화 즉시 실패
        particles[i].x += particles[i].vx * dt;
}
// → 자동 벡터화 실패
// → 수동으로 gather + masked store 필요 (복잡)

// SoA + 마스크 배열로 해결:
struct ParticlePool {
    float x[N], y[N], vx[N], vy[N];
    uint8_t active[N];  // 별도 배열로 분리
};

// SoA 마스킹 업데이트 (벡터화 가능):
for (int i = 0; i <= n - 8; i += 8) {
    // 8개 활성 플래그를 비교하여 마스크 생성
    // (uint8 → int 확장 후 비교)
    __m256i vactive = _mm256_cvtepu8_epi32(
        _mm_loadl_epi64((__m128i*)(pool.active + i))
    );
    __m256i vmask = _mm256_cmpgt_epi32(vactive, _mm256_setzero_si256());

    __m256 vx  = _mm256_loadu_ps(pool.x  + i);
    __m256 vvx = _mm256_loadu_ps(pool.vx + i);
    __m256 new_x = _mm256_fmadd_ps(vvx, _mm256_set1_ps(dt), vx);

    // 마스크된 저장: active인 것만 업데이트
    _mm256_maskstore_ps(pool.x + i, vmask, new_x);
}

// Ch2-05 연결:
// AoS에서 active 필드가 x, y, z와 같은 캐시라인에 있으면:
//   → active 필드를 체크하기 위해 전체 구조체 캐시라인 로드
//   → x 업데이트 없이도 캐시 오염 발생
// SoA에서 active[]를 별도 배열로:
//   → active[] 순차 접근 (캐시라인 1개에 64개 bool)
//   → 필요할 때만 x[], y[] 접근
```

### 5. AoSoA — 중간 타협 패턴

```c
// AoSoA: Array of Struct of Arrays
// 8개 또는 16개 단위로 묶어서 SoA 이점을 유지하면서 그룹 지역성 확보

// 8개 float 블록 단위 SoA 구조
#define BLOCK 8

struct ParticleBlock {
    float x[BLOCK];    // 8개 x 연속
    float y[BLOCK];
    float z[BLOCK];
    float vx[BLOCK];
    float vy[BLOCK];
    float vz[BLOCK];
};   // sizeof = 6 × 8 × 4 = 192 bytes = 3 캐시라인

ParticleBlock blocks[N / BLOCK];

// 이점:
//   블록 내: SoA → x[BLOCK] 연속 → vmovups ymm ✅
//   블록 단위 처리: 3 캐시라인(192B)에 8입자 전부 → 시간 지역성 ✅
//   BLOCK=16이면 AVX-512에 최적, BLOCK=8이면 AVX2에 최적

// 단점:
//   코드 복잡도 증가
//   N이 BLOCK 배수가 아니면 나머지 처리 필요

// 사용처:
//   Bullet Physics, Intel Embree(광선 추적), ISPC 자동 생성 코드
//   → 캐시 + SIMD 동시 최적화가 필요한 최고 성능 코드
```

---

## 💻 실전 실험

### 실험 1: AoS vs SoA 업데이트 루프 성능 비교

```c
// layout_bench.c
#include <stdlib.h>
#include <stdio.h>
#include <time.h>
#include <immintrin.h>

#define N (1 << 20)  // 1M 입자

// AoS
typedef struct { float x, y, vx, vy; } ParticleAoS;
static ParticleAoS aos[N];

// SoA
static float soa_x[N], soa_y[N], soa_vx[N], soa_vy[N];

void update_aos(float dt) {
    for (int i = 0; i < N; i++) {
        aos[i].x += aos[i].vx * dt;
        aos[i].y += aos[i].vy * dt;
    }
}

void update_soa_scalar(float dt) {
    for (int i = 0; i < N; i++) {
        soa_x[i] += soa_vx[i] * dt;
        soa_y[i] += soa_vy[i] * dt;
    }
}

void update_soa_avx2(float dt) {
    __m256 vdt = _mm256_set1_ps(dt);
    for (int i = 0; i <= N - 8; i += 8) {
        __m256 vx = _mm256_loadu_ps(soa_x + i);
        vx = _mm256_fmadd_ps(_mm256_loadu_ps(soa_vx + i), vdt, vx);
        _mm256_storeu_ps(soa_x + i, vx);

        __m256 vy = _mm256_loadu_ps(soa_y + i);
        vy = _mm256_fmadd_ps(_mm256_loadu_ps(soa_vy + i), vdt, vy);
        _mm256_storeu_ps(soa_y + i, vy);
    }
}

static long now_ns() {
    struct timespec ts; clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000000000L + ts.tv_nsec;
}

int main() {
    for (int i = 0; i < N; i++) {
        aos[i].x = soa_x[i] = (float)i * 0.001f;
        aos[i].y = soa_y[i] = (float)i * 0.002f;
        aos[i].vx = soa_vx[i] = 1.0f;
        aos[i].vy = soa_vy[i] = 0.5f;
    }
    const int REPS = 100;
    long t0, t1;

    t0 = now_ns();
    for (int r = 0; r < REPS; r++) update_aos(0.016f);
    t1 = now_ns();
    printf("AoS (scalar):     %.2f ms\n", (t1-t0)/1e6/REPS);

    t0 = now_ns();
    for (int r = 0; r < REPS; r++) update_soa_scalar(0.016f);
    t1 = now_ns();
    printf("SoA (scalar):     %.2f ms\n", (t1-t0)/1e6/REPS);

    t0 = now_ns();
    for (int r = 0; r < REPS; r++) update_soa_avx2(0.016f);
    t1 = now_ns();
    printf("SoA (AVX2+FMA):   %.2f ms\n", (t1-t0)/1e6/REPS);
    return 0;
}
```

```bash
gcc -O2 -mavx2 -mfma -fno-tree-vectorize -o layout_bench layout_bench.c
# -fno-tree-vectorize: AoS/SoA scalar 비교를 위해 자동 벡터화 비활성화

./layout_bench
# 예상 출력:
# AoS (scalar):     ~8.5 ms   (stride 접근 → 캐시 미스 많음)
# SoA (scalar):     ~5.5 ms   (순차 접근 → 캐시 히트율 높음)
# SoA (AVX2+FMA):   ~0.9 ms   (AVX2 8×처리량)
# → AoS vs SoA AVX2: 9.4배 차이

# cachegrind로 캐시 미스 비교
valgrind --tool=cachegrind --cache-sim=yes ./layout_bench
cg_annotate cachegrind.out.<pid>
# → update_aos의 D1miss: N × 2필드접근 / (64B/16B) = 매우 높음
# → update_soa_avx2의 D1miss: N/16 × 2배열 = 매우 낮음
```

### 실험 2: perf로 캐시 미스율 비교

```bash
# AoS vs SoA AVX2 캐시 미스 차이
perf stat -e cycles,instructions,L1-dcache-load-misses,L1-dcache-loads \
    ./layout_bench_aos   # AoS만 실행

perf stat -e cycles,instructions,L1-dcache-load-misses,L1-dcache-loads \
    ./layout_bench_soa   # SoA AVX2만 실행

# 예상:
# AoS:  L1-dcache-load-misses / L1-dcache-loads ≈ 25~35% (stride 접근)
# SoA:  L1-dcache-load-misses / L1-dcache-loads ≈  6~8%  (순차 접근)
```

### 실험 3: godbolt에서 자동 벡터화 비교

```c
// auto_vec_compare.c — godbolt GCC 13.2 -O2 -mavx2 -mfma
struct ParticleAoS { float x, y, vx, vy; };

// AoS 업데이트 — 자동 벡터화 실패 여부 확인
void update_aos_auto(struct ParticleAoS* p, float dt, int n) {
    for (int i = 0; i < n; i++) {
        p[i].x += p[i].vx * dt;
        p[i].y += p[i].vy * dt;
    }
}

// SoA 업데이트 — 자동 벡터화 성공 여부 확인
void update_soa_auto(float* __restrict__ x, float* __restrict__ y,
                     float* __restrict__ vx, float* __restrict__ vy,
                     float dt, int n) {
    for (int i = 0; i < n; i++) {
        x[i] += vx[i] * dt;
        y[i] += vy[i] * dt;
    }
}
```

```bash
# 자동 벡터화 진단
gcc -O2 -mavx2 -mfma -fopt-info-vec -fopt-info-vec-missed auto_vec_compare.c

# 예상 출력:
# auto_vec_compare.c:7:5: missed: not vectorized: ...
#   → AoS: stride 접근 + struct 경계 → 자동 벡터화 실패

# auto_vec_compare.c:15:5: optimized: loop vectorized using 32-byte vectors
#   → SoA + restrict: 바로 YMM 벡터화 성공
```

---

## 📊 성능 비교

```
N=1M 입자 업데이트 (x, y += vx, vy * dt):

레이아웃 + 구현                캐시 미스율   시간(예상)   배율
──────────────────────────────────────────────────────────────
AoS 스칼라                      30%          ~8.5ms       1×
SoA 스칼라                       8%          ~5.5ms       1.5×
SoA 자동 벡터화(-O2)             8%          ~1.2ms       7.1×
SoA 수동 AVX2+FMA               6%          ~0.9ms       9.4×
AoS + gather(수동 SIMD)         25%          ~6.0ms       1.4×

AoSoA (BLOCK=8) + AVX2          5%          ~0.8ms       10.6×
AoSoA (BLOCK=16) + AVX-512      4%          ~0.5ms       17×

픽셀 처리 (RGBA 포맷 비교, 4K 이미지):
레이아웃                         연산 속도     배율
─────────────────────────────────────────────────
RGBA (AoS, 인터리브)             ~45ms         1×
  → R, G, B, A 섞여서 SIMD 어려움
R[], G[], B[], A[] (SoA)         ~8ms          5.6×
  → 채널별 연속 → vmovups + 연산
R+G+B+A 인터리브 + shuffle        ~12ms         3.7×
  → shuffle로 분리 후 SIMD (타협안)
```

---

## ⚖️ 트레이드오프

```
AoS vs SoA 선택 기준:

AoS가 유리한 경우:
  ✅ 단일 객체를 여러 필드에 걸쳐 처리 (하나씩 직렬 처리)
  ✅ 객체 생성/소멸이 빈번한 경우 (메모리 할당 단위 = 객체)
  ✅ 코드 가독성 중요 (팀 전체가 OOP에 익숙)
  ✅ 객체 수가 적어 캐시 이슈가 문제되지 않는 경우 (<수천 개)
  예: 네트워크 세션, DB 레코드, UI 컴포넌트

SoA가 유리한 경우:
  ✅ 같은 연산을 수만~수백만 개에 일괄 처리 (SIMD 핵심 조건)
  ✅ 루프 안에서 일부 필드만 접근 (다른 필드 캐시 오염 방지)
  ✅ GPU/벡터 프로세서와 데이터 공유
  ✅ 자동 벡터화 성공 조건 충족이 목표
  예: 물리 시뮬레이션, 이미지/오디오 처리, ML 추론 배치

AoSoA가 유리한 경우:
  ✅ SIMD 폭(8/16)에 맞는 블록으로 묶어 공간 지역성도 확보
  ✅ Intel Embree, ISPC, Bullet Physics에서 채택
  ✅ BLOCK = SIMD 폭 × 데이터 타입 = AVX2: 8, AVX-512: 16
  ❌ 코드 복잡도가 AoS/SoA보다 높음

AoS에서 SIMD를 강제하는 방법:
  1. gather 명령: 성능 손실 크므로 최후 수단
  2. deinterleave: 루프 진입 전 SoA로 변환 후 처리 후 AoS로 변환
     → 변환 비용 < SIMD 이득이면 채택 가능
  3. AoSoA로 구조 변경 (가장 깔끔)

분기와 데이터 레이아웃의 상호작용:
  AoS + 조건 필드: if (p.active) { update } → 벡터화 불가 + 캐시 낭비
  SoA + 별도 active[]: 마스킹으로 벡터화 가능 + active[] 캐시라인 분리
  → 조건 처리도 SoA가 더 유리

  Ch2-05 연결:
    Hot/Cold 필드 분리 = SoA의 원리와 동일
    "자주 쓰는 필드만 함께" = SoA의 핵심
    → 캐시 활용 + SIMD 벡터화 동시 최적화
```

---

## 📌 핵심 정리

```
데이터 레이아웃과 SIMD 핵심:

AoS vs SoA:
  AoS: [x0,y0,vx0,vy0] [x1,y1,vx1,vy1] → 1객체씩 연속, SIMD 불편
  SoA: x[]: [x0,x1,...,x7] y[]: [y0,...] → 같은 필드 연속, SIMD 최적

왜 SoA가 SIMD에 유리한가:
  vmovups ymm, [ptr]: ptr에서 연속 32 bytes 로드
  AoS에서 x 8개: 비연속 → gather 명령 필요 (16~26× 느림)
  SoA에서 x 8개: x[0..7] 연속 → vmovups 1 명령 ✅

캐시 효율:
  AoS 업데이트: x, vx만 필요인데 y, z, vy, vz도 캐시라인에 올라옴
  SoA 업데이트: x[], vx[] 배열만 접근 → 캐시 오염 없음

분기와의 상호작용:
  AoS + 조건 필드 → 분기 + 비연속 접근 = 이중 손실
  SoA + 별도 마스크 배열 → 마스킹 SIMD 가능

ECS (Entity Component System):
  게임 엔진의 SoA 적용 = 컴포넌트별 배열 분리
  → 관련 컴포넌트만 묶어 처리 → 캐시 + SIMD 동시 최적화

AoSoA:
  BLOCK=8: struct에 8개 float 배열 → AVX2 레지스터 크기 정확히 맞춤
  → 블록 내 SoA이점 + 블록 간 공간 지역성
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드를 SoA로 전환하면 자동 벡터화가 가능해지는가? 전환 후 예상되는 SIMD 명령과 성능 향상 배율을 추정하라.

```c
struct Color { uint8_t r, g, b, a; };
Color pixels[WIDTH * HEIGHT];

// 밝기 조정: 각 채널에 factor 곱
void brighten(float factor) {
    for (int i = 0; i < WIDTH * HEIGHT; i++) {
        pixels[i].r = (uint8_t)(pixels[i].r * factor);
        pixels[i].g = (uint8_t)(pixels[i].g * factor);
        pixels[i].b = (uint8_t)(pixels[i].b * factor);
    }
}
```

<details>
<summary>해설 보기</summary>

**AoS 문제 분석**: `struct Color`는 4 bytes(r,g,b,a). `r` 채널만 접근해도 g, b, a가 같은 캐시라인에 올라옵니다. SIMD로 r 32개를 처리하려면 stride-4 gather가 필요합니다.

**SoA로 전환**:
```c
uint8_t r[WIDTH*HEIGHT], g[WIDTH*HEIGHT], b[WIDTH*HEIGHT], a[WIDTH*HEIGHT];

void brighten_soa(float factor) {
    for (int i = 0; i < WIDTH * HEIGHT; i++) {
        r[i] = (uint8_t)(r[i] * factor);
        g[i] = (uint8_t)(g[i] * factor);
        b[i] = (uint8_t)(b[i] * factor);
    }
}
```

전환 후 자동 벡터화: `uint8_t` 배열이 연속이고 `__restrict__`가 없어도 r[], g[], b[]가 독립적이므로 컴파일러가 각 배열을 독립적으로 벡터화합니다.

AVX2로: `uint8_t` 32개 = 256bit → `vpmulhuw` 또는 float로 변환 후 `vmulps` + 다시 uint8로 클램핑. `uint8` × float는 복잡하므로 실제로는 정수 확장 후 연산이 일반적입니다.

**성능 향상 추정**: 4K 이미지(8M 픽셀)에서 채널당 uint8 처리 시 AVX2 32byte = 32개/명령 → 이론 32배이나 메모리 대역폭 한계로 실측 5~10배가 현실적입니다.

</details>

---

**Q2.** AoSoA에서 BLOCK 크기를 8로 정할 때와 16으로 정할 때의 차이를 설명하라. CPU 타겟이 AVX2만 지원할 때 BLOCK=16은 손해인가?

<details>
<summary>해설 보기</summary>

**BLOCK=8 (AVX2 최적)**: 각 블록의 float 배열이 정확히 32 bytes = YMM 레지스터 크기. 하나의 `vmovups ymm` 명령으로 블록 하나를 완전히 로드. 루프 없이 단일 명령 처리 가능.

**BLOCK=16 (AVX-512 최적)**: 각 블록의 float 배열이 64 bytes = ZMM 레지스터 크기. AVX-512가 없는 CPU에서는 2개의 YMM 명령이 필요하지만 여전히 동작합니다.

**AVX2 CPU에서 BLOCK=16**: 한 블록 처리에 `vmovups ymm × 2` = 2 명령. BLOCK=8보다 명령 수가 2배이지만 루프 반복 수가 반으로 줄고 루프 오버헤드도 반으로 감소. 실제로는 BLOCK=8이 더 최적이지만 BLOCK=16이 크게 느리지는 않습니다(5~15% 차이). 코드를 나중에 AVX-512 지원 서버에 배포할 계획이라면 BLOCK=16이 더 유리합니다.

**결론**: BLOCK = 타겟 SIMD 폭 (AVX2: 8, AVX-512: 16). 멀티 플랫폼이라면 런타임 CPUID 체크 후 BLOCK 선택이 이상적입니다.

</details>

---

**Q3.** 다음 두 코드가 같은 알고리즘을 구현한다. `-O2 -mavx2`에서 어느 쪽이 더 빠르고, `perf`로 어떤 차이가 관찰되는가?

```c
// 버전 A: AoS
struct Vec3 { float x, y, z; };
float magnitude_sum_aos(struct Vec3* v, int n) {
    float s = 0;
    for (int i = 0; i < n; i++)
        s += sqrtf(v[i].x*v[i].x + v[i].y*v[i].y + v[i].z*v[i].z);
    return s;
}

// 버전 B: SoA
float magnitude_sum_soa(float* __restrict__ x, float* __restrict__ y,
                         float* __restrict__ z, int n) {
    float s = 0;
    for (int i = 0; i < n; i++)
        s += sqrtf(x[i]*x[i] + y[i]*y[i] + z[i]*z[i]);
    return s;
}
```

<details>
<summary>해설 보기</summary>

**버전 B (SoA)가 빠릅니다**.

차이 원인:
1. **자동 벡터화**: 버전 A는 구조체 stride 접근으로 자동 벡터화 어려움. 버전 B는 `__restrict__` + 연속 배열 → `vsqrtps ymm` 포함한 완전 벡터화 가능 (단, `-ffast-math` 필요 또는 `sqrtf`를 `vsqrtps`로 변환 필요).
2. **캐시 효율**: 버전 A의 `Vec3` = 12 bytes. x, y, z를 순서대로 접근해도 struct 사이즈가 stride. 캐시라인 64B에 Vec3 5.3개 → 정확히 맞지 않아 약간의 경계 효과 발생. 버전 B: x[] = 완전 연속, 캐시라인당 16 float.

`perf` 차이:
- 버전 A: `L1-dcache-load-misses` 상대적으로 높음, `fp_arith_inst_retired.256b_packed_single` = 0 (벡터화 실패 시)
- 버전 B: `fp_arith_inst_retired.256b_packed_single` = N/8 (AVX2 벡터화 성공), 미스율 낮음

실측 차이: `-ffast-math` 없이 sqrtf가 벡터화되지 않으면 두 버전 유사. `-ffast-math -O3` 추가 시 버전 B가 2~4배 빠릅니다.

</details>

---

<div align="center">

**[⬅️ 이전: 수동 SIMD 인트린식](./03-manual-simd-intrinsics.md)** | **[홈으로 🏠](../README.md)** | **[다음: 처리량 vs 지연시간 ➡️](./05-throughput-vs-latency.md)**

</div>
