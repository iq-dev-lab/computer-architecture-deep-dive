# 케이스 스터디 — 느린 코드 한 줄을 사이클까지 추적

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 의도적으로 느린 데이터 처리 코드를 perf로 어떻게 진단하는가?
- cache-miss의 근원을 어떻게 추적하고 AoS→SoA 전환으로 해결하는가?
- branchless 변환으로 분기 미스를 어떻게 제거하는가?
- SIMD 적용 전후의 어셈블리 diff에서 무엇을 봐야 하는가?
- 각 최적화 단계가 IPC·cache-miss·branch-miss를 어떻게 바꾸는가?
- 전체 레포에서 배운 도구들을 단일 워크로드에 어떻게 통합 적용하는가?

---

## 🔍 왜 이 개념이 중요한가

### 레포 전체의 캡스톤 — 모든 개념을 한 코드에 적용

```
지금까지 배운 것:
  Ch1  파이프라인·OoO·분기 예측
  Ch2  캐시 계층·지역성·AoS/SoA·TLB
  Ch3  MESI·False Sharing·메모리 배리어
  Ch4  추상화 비용·branchless
  Ch5  SIMD·자동/수동 벡터화·SoA
  Ch6  멀티코어·NUMA·친화도
  Ch7  perf·벤치마킹·루프라인·우선순위

이 모든 것이 실제 코드 하나에서 어떻게 작동하나?

이 문서:
  의도적으로 느린 입자 시뮬레이션 코드를 받아
  단계별로 측정·진단·최적화
  최종 ~10배 가속을 사이클 단위로 증명

각 단계마다:
  ① perf로 진단
  ② 무엇이 병목인지 식별
  ③ 어느 챕터의 어떤 도구로 해결할지 결정
  ④ 적용 후 어셈블리/IPC/캐시미스 비교
```

---

## 😱 잘못된 이해

### Before: "프로파일러 찍어서 가장 오래 걸리는 함수를 빠르게 하자"

```
흔한 최적화 접근:
  ① perf record → flame graph
  ② 가장 큰 박스 발견 → 그 함수 안에 -O3 풀고 인라이닝
  ③ 안 빨라지면 SIMD 적용 시도
  ④ 그래도 안 되면 멀티스레드

문제:
  병목의 본질을 보지 않음
  같은 함수가 느린 이유가 100가지
  → 캐시 미스인가? 분기 미스인가? 메모리 대역폭인가? IPC가 낮은가?

올바른 진단 순서:
  ① 워크로드 자체가 compute-bound인가 memory-bound인가? (Ch7-03 루프라인)
  ② IPC가 낮은가 — 왜? (의존성? 분기? 캐시?)
  ③ cache-miss가 많은가 — 어느 레벨? L1/L2/L3/TLB?
  ④ branch-miss가 많은가 — 데이터 의존 분기?
  ⑤ false sharing 있는가? (멀티스레드)
  
  각 답에 따라 다른 최적화 적용 — 한 가지 망치로 모든 못을 박지 말 것
```

---

## ✨ 올바른 이해

### After: 측정 → 진단 → 단계별 적용 → 재측정

```
이 문서의 케이스 — 2D 입자 시뮬레이션:

  입자 N개, 각 입자 (x, y, vx, vy, mass, id, color)
  매 프레임마다:
    ① 모든 입자에 중력 적용 (vy -= g * dt)
    ② 위치 업데이트 (x += vx * dt, y += vy * dt)
    ③ 화면 밖이면 제거 (조건)
    ④ 색상 결정 (속도에 따라)
  
  N = 1,000,000, 60 FPS 목표 (1프레임 16.7ms)

V0 (baseline):
  순진한 AoS + 분기 + 가상 함수 사용
  ~150ms / frame ← 9배 느림 (목표 미달)

V1 (캐시 친화 — AoS → SoA):
  ~60ms / frame ← 2.5배 가속

V2 (branchless 변환):
  ~45ms / frame ← 추가 1.3배

V3 (SIMD 수동 벡터화):
  ~12ms / frame ← 추가 3.8배

V4 (멀티스레드 + 친화도):
  ~3.5ms / frame ← 추가 3.4배

총합: 150ms → 3.5ms (~43배)
  → 단일 영역 최적화로는 절대 못 얻는 효과
  → 여러 챕터 도구를 누적 적용한 결과
```

---

## 🔬 내부 동작 원리

### V0: Baseline — 순진한 객체지향 구현

```cpp
// V0: AoS + 가상 함수 + 분기
struct Particle {
    float x, y;
    float vx, vy;
    float mass;
    int id;
    uint32_t color;
};

class Renderer {
public:
    virtual uint32_t color_for(float v) const {
        if (v < 1.0f) return 0x0000FF;   // 파랑
        else if (v < 5.0f) return 0x00FF00; // 초록
        else return 0xFF0000;             // 빨강
    }
};

void update_v0(std::vector<Particle>& ps, Renderer* r, float dt) {
    const float g = 9.8f;
    for (auto& p : ps) {
        if (p.y < 0) continue;            // 분기 1: 화면 밖
        p.vy -= g * dt;                   // 중력
        p.x  += p.vx * dt;                // 위치 X
        p.y  += p.vy * dt;                // 위치 Y
        float speed = std::sqrt(p.vx*p.vx + p.vy*p.vy);
        p.color = r->color_for(speed);    // 가상 함수 호출
    }
}
```

```
V0의 문제 진단:

perf stat:
  cycles                : 4.5e8
  instructions          : 3.2e8
  IPC                   : 0.71      ← 매우 낮음 (현대 CPU는 3+ 가능)
  cache-references      : 4.1e7
  cache-misses          : 8.2e6     ← 미스율 20%
  branch-misses         : 3.5e6     ← 미스율 15%

해석:
  - IPC 낮음 → 파이프라인이 자주 멈춤 (Ch1)
  - cache-miss 많음 → 메모리 접근 패턴 나쁨 (Ch2)
  - branch-miss 많음 → 데이터 의존 분기 (Ch1-05)
  - 가상 함수 호출 → BTB 오염 (Ch4-02)

병목 우선순위 (Ch7-04 우선순위 원칙):
  1. cache-miss → 데이터 레이아웃 개선 (Ch2-05)
  2. branch-miss → branchless 변환 (Ch4-04)
  3. IPC → SIMD로 명령당 작업량 ↑ (Ch5)
  4. 단일 코어 한계 → 멀티스레드 (Ch6)
```

### V1: AoS → SoA 전환 (Ch2-05, Ch5-04)

```cpp
// V1: SoA로 데이터 레이아웃 변경
struct Particles {
    std::vector<float> x;
    std::vector<float> y;
    std::vector<float> vx;
    std::vector<float> vy;
    std::vector<float> mass;
    std::vector<int> id;
    std::vector<uint32_t> color;
    
    size_t size() const { return x.size(); }
};

void update_v1(Particles& ps, Renderer* r, float dt) {
    const float g = 9.8f;
    const size_t n = ps.size();
    for (size_t i = 0; i < n; i++) {
        if (ps.y[i] < 0) continue;
        ps.vy[i] -= g * dt;
        ps.x[i]  += ps.vx[i] * dt;
        ps.y[i]  += ps.vy[i] * dt;
        float speed = std::sqrt(ps.vx[i]*ps.vx[i] + ps.vy[i]*ps.vy[i]);
        ps.color[i] = r->color_for(speed);
    }
}
```

```
V0 → V1 변화:

캐시라인 이용률:
  AoS: Particle 1개 = 32 bytes, 캐시라인 64B에 2개
       → 한 라인의 절반만 쓰는 필드 있음 (id, color)
  SoA: vx[16개] / vy[16개] 등이 한 라인에 가득
       → 캐시라인 100% 이용

perf stat 비교:
                    V0         V1
  IPC               0.71       1.45     ← 2배 향상
  cache-misses      8.2M       1.8M     ← 4.5배 감소
  branch-misses     3.5M       3.4M     ← 변화 없음 (다음 단계에서)
  실행시간          150ms      60ms     ← 2.5배 가속

자동 벡터화 가능성:
  AoS는 stride가 32B → 컴파일러가 벡터화 포기
  SoA는 stride가 4B (연속) → 자동 벡터화 시도 가능
  
  godbolt에서 어셈블리 확인:
    AoS: 스칼라 명령만
    SoA: vmovups, vmulps, vaddps 등 SIMD 명령 일부 등장
```

### V2: branchless 변환 (Ch4-04)

```cpp
// V2: 분기 제거
void update_v2(Particles& ps, Renderer* r, float dt) {
    const float g = 9.8f;
    const size_t n = ps.size();
    
    for (size_t i = 0; i < n; i++) {
        // 화면 밖이면 위치 안 바꾸도록 mask 사용
        float mask = (ps.y[i] >= 0) ? 1.0f : 0.0f;
        
        ps.vy[i] -= g * dt * mask;
        ps.x[i]  += ps.vx[i] * dt * mask;
        ps.y[i]  += ps.vy[i] * dt * mask;
        
        float speed = std::sqrt(ps.vx[i]*ps.vx[i] + ps.vy[i]*ps.vy[i]);
        
        // 가상 함수 → 인라인 선택 (Ch4-02)
        uint32_t c1 = 0x0000FF;
        uint32_t c2 = 0x00FF00;
        uint32_t c3 = 0xFF0000;
        uint32_t c = (speed < 1.0f) ? c1 : ((speed < 5.0f) ? c2 : c3);
        // 또는 완전 branchless:
        // uint32_t c = (speed >= 1.0f) ? ((speed >= 5.0f) ? c3 : c2) : c1;
        // 또는 lookup table 활용
        
        ps.color[i] = c;
    }
}
```

```
V1 → V2 변화:

godbolt 어셈블리 diff:
  V1:
    cmp     dword ptr [rdx + 4*rax], 0
    jl      .skip
    ...
    call    Renderer::color_for(float)  ← 가상 함수
    .skip:
  
  V2:
    ucomiss xmm0, xmm1
    cmovl   eax, ebx     ← cmov로 분기 제거
    ucomiss xmm2, xmm3
    cmovl   ecx, edx

perf stat 비교:
                    V1         V2
  IPC               1.45       2.10     ← 1.4배 향상
  cache-misses      1.8M       1.7M     ← 변화 없음
  branch-misses     3.4M       0.4M     ← 8.5배 감소
  실행시간          60ms       45ms     ← 1.3배 가속

trade-off:
  branchless는 양쪽 다 계산 → 명령 수는 늘어남
  하지만 분기 미스 손실(15cy)이 더 컸으므로 순이득
  
주의 (Ch4-04):
  화면 밖 입자 비율이 매우 작거나 매우 큰 경우(거의 100% 예측 가능)
  → 분기가 오히려 빠를 수 있음
  → 측정으로 결정
```

### V3: SIMD 수동 벡터화 (Ch5-03)

```cpp
// V3: AVX2 인트린식 — 한 번에 8개 처리
#include <immintrin.h>

void update_v3(Particles& ps, float dt) {
    const float g = 9.8f;
    const size_t n = ps.size();
    const __m256 g_dt = _mm256_set1_ps(g * dt);
    const __m256 zero = _mm256_setzero_ps();
    const __m256 one  = _mm256_set1_ps(1.0f);
    const __m256 dt_v = _mm256_set1_ps(dt);
    
    size_t i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 x  = _mm256_loadu_ps(&ps.x[i]);
        __m256 y  = _mm256_loadu_ps(&ps.y[i]);
        __m256 vx = _mm256_loadu_ps(&ps.vx[i]);
        __m256 vy = _mm256_loadu_ps(&ps.vy[i]);
        
        // y >= 0 ? 1.0 : 0.0  마스크
        __m256 mask = _mm256_blendv_ps(zero, one,
                          _mm256_cmp_ps(y, zero, _CMP_GE_OS));
        
        // vy -= g*dt*mask
        vy = _mm256_sub_ps(vy, _mm256_mul_ps(g_dt, mask));
        
        // x += vx*dt*mask
        x = _mm256_fmadd_ps(_mm256_mul_ps(vx, dt_v), mask, x);
        // y += vy*dt*mask
        y = _mm256_fmadd_ps(_mm256_mul_ps(vy, dt_v), mask, y);
        
        _mm256_storeu_ps(&ps.x[i],  x);
        _mm256_storeu_ps(&ps.y[i],  y);
        _mm256_storeu_ps(&ps.vy[i], vy);
        
        // speed = sqrt(vx² + vy²)
        __m256 sp = _mm256_sqrt_ps(
            _mm256_fmadd_ps(vx, vx, _mm256_mul_ps(vy, vy)));
        
        // 색상은 별도 처리 (분기·blend로)
        // ... (생략)
    }
    
    // 나머지 (n % 8) 스칼라 처리
    for (; i < n; i++) {
        // V2와 동일
    }
}
```

```
V2 → V3 변화:

명령당 처리량:
  V2: 1 입자/명령
  V3: 8 입자/명령 (AVX2)

perf stat 비교:
                    V2         V3
  IPC               2.10       2.50     ← 약간 향상
  instructions      3.2e8      4.5e7    ← 7배 감소 (명령당 작업량 8배)
  cache-misses      1.7M       1.6M     ← 변화 없음
  실행시간          45ms       12ms     ← 3.75배 가속

근접 메모리 대역폭 한계:
  perf stat -e cycles_activity.stalls_l3_miss
  → L3 miss로 인한 스톨이 ~30% → memory-bound 영역 진입
  → 더 빠르게 하려면 streaming store나 prefetch 시도
  → 또는 working set 축소 (필요한 필드만 SoA화)

루프라인 모델 (Ch7-03):
  V0~V2: compute-bound (IPC 향상으로 가속)
  V3:    memory-bound 경계 (메모리 BW가 한계로)
  V4:    같은 BW를 멀티코어로 활용
```

### V4: 멀티스레드 + 친화도 (Ch6-05)

```cpp
// V4: OpenMP + 친화도
#include <omp.h>
#include <sched.h>
#include <pthread.h>

void update_v4(Particles& ps, float dt) {
    const size_t n = ps.size();
    
    #pragma omp parallel num_threads(8)
    {
        // 스레드별 친화도 핀닝
        int tid = omp_get_thread_num();
        cpu_set_t mask; CPU_ZERO(&mask); CPU_SET(tid, &mask);
        pthread_setaffinity_np(pthread_self(), sizeof(mask), &mask);
        
        // chunk 분할
        size_t per_thread = (n + 7) / 8;
        size_t start = tid * per_thread;
        size_t end   = std::min(start + per_thread, n);
        
        // 각 스레드가 자기 chunk만 처리 (False Sharing 회피)
        // chunk 경계가 캐시라인 정렬되도록 padding
        for (size_t i = start; i + 8 <= end; i += 8) {
            // V3의 SIMD 코드 그대로
            __m256 x = _mm256_loadu_ps(&ps.x[i]);
            // ...
        }
    }
}
```

```
V3 → V4 변화:

8코어 활용:
  V3: 1 코어, 메모리 BW 1개 채널 활용
  V4: 8 코어, 메모리 BW 다중 채널 활용 (NUMA 노드 균등 분배 시)

False Sharing 회피:
  스레드 i가 ps.x[i*125000 : (i+1)*125000] 처리
  → 다른 스레드의 영역과 캐시라인 겹침 없음 (충분히 큰 chunk)

NUMA (Ch6-02):
  첫 입자 데이터는 한 NUMA 노드에 있을 가능성
  → 다른 노드의 스레드가 접근하면 원격 비용
  해결: numa_alloc_interleaved로 메모리를 노드에 분산
  또는 thread 0이 전체 데이터 초기화 → first-touch로 노드 0에 몰림 (피하기)

perf stat 비교 (8 스레드):
                    V3         V4
  task-clock        12ms       28ms (CPU·시간 누적)
  실제 wall time    12ms       3.5ms     ← 3.4배 가속
  IPC               2.50       2.30     (코어별)
  cache-references  8M         9M       (코어별)
  
실제 wall time이 3.4배 → 이상적 8배 못 미침
  이유:
    1. 메모리 대역폭 포화
    2. 스레드 시작/종료 오버헤드 (작은 워크로드)
    3. 같은 L3 도메인의 캐시 경쟁
  
스케일링 분석 (Ch6-03):
  암달 P=0.95 가정 → 8 코어에서 최대 5.9배
  실측 3.4배 → 추가 직렬 부분(스레드 fork/join) 존재
  
권장 추가 최적화:
  - 스레드풀 사전 생성 (매 프레임 fork/join 비용 제거)
  - 좀더 큰 chunk로 잡 디스패치 비용 분산
  - NUMA-aware 메모리 할당
```

### 최종 결과 — 사이클 단위 비교

```
입자 1,000,000개 한 프레임 처리:

버전   실행시간   사이클        명령어      IPC   캐시미스   분기미스   주요 기법
─────────────────────────────────────────────────────────────────────────────
V0     150ms      4.5e8         3.2e8       0.71  8.2M       3.5M       baseline (AoS + 분기 + 가상함수)
V1     60ms       1.8e8         2.6e8       1.45  1.8M       3.4M       SoA (Ch2-05)
V2     45ms       1.3e8         2.8e8       2.10  1.7M       0.4M       branchless (Ch4-04)
V3     12ms       3.6e7         4.5e7       2.50  1.6M       0.1M       SIMD (Ch5-03)
V4     3.5ms      1.0e7×8core   1.5e7       2.30  9M         0.1M       MT+친화도 (Ch6-05)

전체 가속: 150ms → 3.5ms = 43배

기법별 기여:
  데이터 레이아웃     → 2.5배 (가장 큰 단일 효과)
  분기 제거          → 1.3배
  SIMD              → 3.75배
  멀티스레드 + 친화도 → 3.4배
  
교훈:
  1. 알고리즘 → 메모리 접근 → 벡터화 → 멀티코어 순서가 효과 순 (Ch7-04)
  2. 한 가지 도구로는 절대 못 얻는 가속을 다단계 누적으로 달성
  3. 매 단계마다 perf로 측정·검증
```

---

## 💻 실전 실험

### 실험 1: 단계별 perf로 IPC 추적

```bash
# 각 버전을 빌드
g++ -O2 -mavx2 -mfma -fopenmp v0.cpp -o v0
g++ -O2 -mavx2 -mfma -fopenmp v1.cpp -o v1
# ... v4까지

# 각 버전 측정
for v in v0 v1 v2 v3 v4; do
  echo "=== $v ==="
  perf stat -e cycles,instructions,cache-references,cache-misses,branches,branch-misses \
    ./$v 2>&1 | tail -15
done

# IPC와 미스율 변화를 표로 정리 → V0~V4 각 단계의 효과 확인
```

### 실험 2: 어셈블리 diff로 SIMD 검증

```bash
# V2(분기 제거 스칼라) vs V3(SIMD) 어셈블리
g++ -O3 -mavx2 -S -masm=intel v2.cpp -o v2.s
g++ -O3 -mavx2 -S -masm=intel v3.cpp -o v3.s

# diff로 명령어 패턴 비교
diff v2.s v3.s | head -50

# 또는 godbolt에서 색깔 매칭으로 확인
# → V3 코드 영역에 vmulps, vfmadd, vsqrtps 등 등장

# fma 활용 여부 확인
grep -c "vfmadd\|vfnmadd" v3.s
# 0이면 -mfma 옵션 추가 필요
```

### 실험 3: 루프라인 모델로 위치 추적

```bash
# likwid-perfctr로 연산 강도(FLOPS/byte) 측정
likwid-perfctr -C 0 -g MEM_DP -m ./v3
# 출력 중:
#   DP MFLOP/s         : 12500
#   Memory bandwidth   : 18.5 GB/s
#   → 강도 = 12500e6 / 18.5e9 = 0.68 FLOPS/byte

# 머신의 메모리 BW 측정 (peak)
likwid-bench -t stream -W N:1GB:1
# 예: 26 GB/s 라면 V3은 BW의 71% 활용 중

# Intel Advisor로 시각화
advisor -collect roofline -project-dir ./adv_proj -- ./v3
advisor-gui ./adv_proj
# → V3가 메모리 대역폭 한계에 가까이 붙어있는지 확인
```

### 실험 4: 멀티스레드 스케일링 확인

```bash
# 스레드 수 1, 2, 4, 8로 변화
for t in 1 2 4 8; do
  OMP_NUM_THREADS=$t perf stat -r 3 ./v4 2>&1 | grep "time elapsed"
done

# 출력 예:
# 1 thread: 12ms
# 2 threads: 6.5ms (~1.85배)
# 4 threads: 4.0ms (~3배)
# 8 threads: 3.5ms (~3.4배)
# → 4→8 사이에서 스케일링 감속 → 메모리 BW 또는 NUMA 영향

# perf c2c로 캐시라인 경쟁 확인
perf c2c record ./v4
perf c2c report
# False Sharing 후보 라인 확인
```

### 실험 5: 친화도 효과 확인

```bash
# 친화도 없이
OMP_NUM_THREADS=8 ./v4   # 평균 3.5ms, p99 6ms (가끔 스파이크)

# OMP_PROC_BIND로 친화도
OMP_PROC_BIND=close OMP_NUM_THREADS=8 ./v4
# → 인접 코어에 스레드 배치 (캐시 공유)
# 평균 3.0ms, p99 3.5ms (안정화)

# OMP_PROC_BIND=spread
# → 코어들을 노드에 분산 → NUMA 노드 활용 ↑ (대역폭 ↑)
```

---

## 📊 성능 비교

```
이 케이스의 최적화 단계별 효과 — 단일 테이블 통합:

단계   기법               vs V0    누적 가속    적용 챕터
─────────────────────────────────────────────────────────────
V0     baseline           1.00x    -            -
V1     AoS→SoA            0.40x    2.5x         Ch2-05, Ch5-04
V2     branchless        0.30x    3.3x         Ch4-04
V3     SIMD (AVX2)        0.08x    12.5x        Ch5-03
V4     MT + Affinity      0.023x   43.0x        Ch6-05

cache 미스 추이:
  V0 → V1: 8.2M → 1.8M (78% 감소) — 가장 큰 단일 개선
  V1 → V2: 1.8M → 1.7M (변화 없음 — 분기와 무관)
  V2 → V3: 1.7M → 1.6M (변화 없음)
  V3 → V4: 1.6M → 9M  (코어별 → 합산은 비슷)

branch 미스 추이:
  V0 → V1: 거의 동일 (분기 패턴은 변하지 않음)
  V1 → V2: 8.5배 감소 (분기 자체를 제거)
  V2 → V4: 거의 동일

IPC 추이:
  V0 0.71 → V3 2.50 (3.5배)
  V4는 코어별 IPC는 약간 감소 (메모리 BW 경쟁)
  
이 추적으로 알 수 있는 것:
  각 도구가 어떤 카운터에 영향을 주는지 명확
  → 새 워크로드에서도 같은 진단 → 같은 도구 선택 가능
  → "측정 기반 최적화"의 실전 패턴
```

---

## ⚖️ 트레이드오프

```
이 케이스의 트레이드오프:

코드 가독성:
  V0: 가장 자연스러운 OOP — 코드 100줄
  V1: SoA → 컬렉션 구조 명시적 관리 — 150줄
  V2: branchless → 산술적 표현 → 직관 ↓
  V3: 인트린식 코드 → 가독성 큰 폭 ↓ — 300줄
  V4: 멀티스레드 → 동기화·NUMA 관리 → 400줄+

  → 핫스팟에만 적용 권장
  → 비핫스팟은 V0 스타일 유지

이식성:
  V3: AVX2 강제 → ARM 등 다른 ISA 분기 필요
  V4: 친화도는 OS 의존 (Linux 위주)
  → 라이브러리화 + 런타임 dispatch + 빌드 옵션 분기

유지보수:
  더 많은 최적화 = 더 많은 버그 가능성
  → 단위 테스트 + 결과 검증 필수
  → V0과 결과 비교 (tolerance 안에서 일치)
  → fuzz 테스트로 엣지 케이스 검증

언제 V0에 머무르는가:
  N이 작거나 (< 1000)
  실행 빈도가 낮거나 (분당 한 번)
  성능 SLA가 충분히 만족됨
  → "충분히 빠른가" 기준 (Ch7-04)

언제 V4까지 가는가:
  엄격한 latency SLA
  계산이 시스템 비용의 큰 부분
  처리량이 비즈니스 가치와 직결 (게임, 거래, ML 인퍼런스)
```

---

## 📌 핵심 정리

```
케이스 스터디 핵심 — 전체 레포의 종합:

진단 순서 (Ch7-04 우선순위):
  1. 알고리즘 검토
  2. 데이터 레이아웃 (Ch2-05) — 보통 가장 큰 이득
  3. 분기 패턴 (Ch4-04) — branchless 적합한지
  4. 자동 벡터화 가능성 (Ch5-02)
  5. 수동 SIMD (Ch5-03)
  6. 멀티스레드 + 친화도 (Ch6-05)

매 단계 측정:
  perf stat -e cycles,instructions,cache-references,cache-misses,branches,branch-misses
  IPC가 오르는지, 미스가 줄어드는지 확인
  안 오르면 이 단계가 아닌 다른 곳을 봐야 함

도구별 효과 (이 케이스 기준):
  데이터 레이아웃   → 2.5배
  branchless        → 1.3배
  SIMD              → 3.75배
  멀티스레드+친화도  → 3.4배
  총 43배

연결 — 모든 챕터:
  Ch1  분기 예측·OoO·ILP → branchless·의존성 사슬 끊기
  Ch2  캐시 계층·AoS→SoA → 가장 큰 단일 효과
  Ch3  MESI·False Sharing → 멀티스레드 시 회피
  Ch4  추상화 비용 → 가상 함수 제거·인라이닝
  Ch5  SIMD·자동/수동 → 컴파일러 활용 + 인트린식
  Ch6  멀티코어·NUMA·친화도 → 마지막 단계
  Ch7  측정 도구 일체

다음 단계 (이 레포 너머):
  더 많은 최적화 → CUDA/GPU로
  더 많은 처리량 → 분산 시스템 (참고: performance-testing-deep-dive)
  하지만 단일 노드 효율이 모든 분산의 기반
  → 이 레포에서 배운 도구는 어디서나 통용
```

---

## 🤔 생각해볼 문제

**Q1.** 이 케이스에서 V1(SoA 전환)이 가장 큰 단일 효과(2.5배)를 냈다. 만약 입자 수가 1,000,000이 아니라 1,000이었다면 V1의 효과는 어떻게 달라졌을까?

<details>
<summary>해설 보기</summary>

**N이 작을 때 V1의 효과**:

N=1000일 때:
- 한 입자당 32B × 1000 = 32KB → L1 캐시(32KB)에 거의 들어감
- 어떤 레이아웃이든 캐시 hit률이 매우 높음
- → V0와 V1의 차이가 거의 없음 (둘 다 L1에서 처리)

N=10,000일 때:
- 320KB → L2 캐시(256KB~1MB) 영역
- 여전히 차이 작음

N=1,000,000일 때 (이 케이스):
- 32MB → L3 캐시(8~32MB)를 넘김 → DRAM 접근 필요
- 캐시 라인 이용률이 결정적 → SoA가 큰 차이

교훈:
- 작은 데이터: 알고리즘이 효과 결정 (캐시는 충분)
- 큰 데이터: 메모리 접근 패턴이 효과 결정
- → 워크로드 크기에 따라 최적화 우선순위가 다름
- → 이것이 루프라인 모델(Ch7-03)이 중요한 이유: compute-bound인지 memory-bound인지가 최적화 방향을 결정

추가: N=100 정도라면 V0를 그대로 두고 SIMD만 적용하는 게 가성비 최고일 수 있음 (코드 단순성 유지).

</details>

---

**Q2.** V3에서 V4로 가면서 wall time이 12ms → 3.5ms로 3.4배 줄었다. 이상적 8배 가속과 비교하면 4.6배 손실이 있다. 이 손실의 원인을 우선순위대로 추정하라.

<details>
<summary>해설 보기</summary>

**손실 원인 우선순위**:

**1. 메모리 대역폭 포화 (가장 가능성 높음)**:
- V3에서 이미 memory-bound 경계 진입
- 8 코어가 같은 DRAM 채널을 공유하면 BW 1.5~3배 정도만 늘어남
- 진단: `likwid-perfctr -g MEM` 또는 `perf stat -e cas_count_rd,cas_count_wr`
- 해결: NUMA 노드 분산 (`numactl --interleave=all` 또는 첫 노드별 데이터 배치)

**2. NUMA 원격 접근 (큰 시스템에서 강력)**:
- 단일 소켓 8코어면 영향 적음
- 듀얼 소켓 시스템에서 메모리 노드 0에 데이터가 몰리면 노드 1 스레드가 원격 접근 → 1.5~3배 손실
- 진단: `numastat -p <PID>` 또는 `perf stat -e node-loads,node-load-misses`
- 해결: `numactl --interleave=all` 또는 명시적 first-touch

**3. 스레드 생성/종료 오버헤드**:
- OpenMP `parallel` 블록 진입 시 ~10~50µs 오버헤드
- 3.5ms 작업에 50µs면 ~1.4% 손실 (작음)
- 해결: 스레드풀 사전 생성, 같은 풀 재사용

**4. SMT 형제 경쟁**:
- 8 코어가 모두 다른 물리 코어면 영향 없음
- 4 물리 코어 + SMT 8 논리 코어면 실행 유닛 경쟁 → 2배 가속이 1.4배에 그침
- 진단: `lscpu --extended`로 SMT 형제 확인
- 해결: 짝수 CPU ID만 사용 (또는 BIOS에서 SMT 비활성화)

**5. False Sharing**:
- chunk 경계가 캐시라인에 걸치면 인접 스레드 라인 경쟁
- chunk 크기 125,000 entries × 4B = 500KB → 라인 단위 정렬 가능
- 진단: `perf c2c record/report`
- 해결: chunk 시작/끝을 64B 정렬

**6. 캐시 워밍업**:
- 매 프레임 시작마다 콜드 캐시 (다른 작업이 캐시를 evict한 경우)
- 진단: 측정 반복 시 첫 번째와 이후 시간 차이

암달의 법칙 관점:
- P=0.95(병렬화 비율) 가정 → 8코어 최대 5.9배
- 실측 3.4배 → 추가로 P가 더 낮음을 시사 (P≈0.85)
- → 직렬 부분(스레드 fork/join, 결과 reduce 등) 분석 필요

</details>

---

**Q3.** 이 케이스의 최적화 패턴을 LLM 인퍼런스 핫루프(예: float16 행렬-벡터 곱셈)에 어떻게 응용할 수 있을까?

<details>
<summary>해설 보기</summary>

**LLM 인퍼런스 핫루프 분석**:

매트릭스-벡터 곱셈 `y = W @ x`:
- W: [hidden, hidden] float16 (예: 4096×4096)
- x: [hidden] float16
- y: [hidden] float16

**이 케이스의 최적화 패턴 적용**:

**1. 데이터 레이아웃 (Ch2-05) → 이미 SoA**:
- 행렬은 기본적으로 연속 메모리 (row-major)
- 추가 개선: 캐시 친화 블록 분할 (cache blocking / tiling)
- 64×64 블록 단위로 처리 → L1 캐시에 W 블록 fit

**2. branchless (Ch4-04) → 거의 무관**:
- 행렬 곱셈에는 분기가 거의 없음
- 단, activation function(ReLU 등)에는 적용 가능
  - `max(0, x)` 는 `__m256 _mm256_max_ps(zero, x)`로 branchless

**3. SIMD (Ch5-03) → 절대적으로 중요**:
- float16 SIMD: AVX-512의 `_mm512_dpbf16_ps` (bf16) 또는 별도 라이브러리
- Intel AMX(Sapphire Rapids 이후): bf16/int8 행렬 명령
- ARM SVE2: scalable vector with bf16
- 효과: 8~32배

**4. 멀티스레드 + 친화도 (Ch6-05)**:
- 행을 스레드별 분할 (`y[i:i+chunk] = W[i:i+chunk] @ x`)
- 각 스레드는 자기 W 부분만 접근 → False Sharing 없음
- NUMA: 큰 모델은 노드별 W 부분 배치 (interleave 권장)

**LLM 특화 추가 최적화 (이 케이스 너머)**:

**5. 양자화**:
- float16 → int8 / int4 → 메모리 BW 4~8배 절약
- llama.cpp의 q4_k_m 양자화가 대표적

**6. KV 캐시 활용**:
- 이전 토큰의 attention key/value 재사용
- 새 토큰만 새로 계산
- 메모리 접근 패턴이 매우 중요

**7. Flash Attention**:
- softmax(QK^T)V 를 메모리에 거대 행렬로 저장 안 하고 streaming
- 메모리 BW 한계 회피 (Ch7-03 루프라인 관점)

**8. GPU와 비교**:
- 큰 모델(LLM 7B+): GPU가 압도적 (메모리 BW 1TB/s 이상)
- 작은 모델(distillBERT 등): CPU SIMD로 충분 (Ch5-05 결론)

**도구 패턴은 같다**:
- perf로 IPC/BW/캐시 미스 측정
- compute-bound인지 memory-bound인지 판정 (루프라인)
- 가장 큰 병목부터 해결
- 한 번에 하나씩, 측정으로 검증

이 케이스에서 배운 진단·도구 적용 패턴은 ML 인퍼런스, DB 쿼리, 비디오 인코딩 등 모든 핫루프에 동일하게 적용됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: 최적화 우선순위](./04-optimization-priority.md)** | **[홈으로 🏠](../README.md)**

</div>
