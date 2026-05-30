# 처리량 vs 지연시간 — SIMD가 빛나는 워크로드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- SIMD는 처리량(throughput)을 늘리는가, 지연시간(latency)을 줄이는가?
- 작은 데이터 단발 처리에 SIMD를 적용해도 효과가 없는 이유는?
- 어떤 워크로드가 SIMD에 적합하고, 어떤 워크로드는 부적합한가?
- 같은 SIMD 명령이라도 응답성과 배치 처리에서 의미가 다른 이유는?
- GPU와 SIMD의 관계 — 둘은 같은 SIMD 가속인가 다른 것인가?
- 마이크로서비스의 작은 요청에 AVX-512가 도움이 안 되는 이유는?

---

## 🔍 왜 이 개념이 중요한가

### "SIMD 코드로 바꿨는데 응답시간은 그대로?"

```
흔한 기대:
  AVX2로 함수를 SIMD로 바꿨다
  → 한 번에 8개 처리하니까 8배 빨라지겠지?

실제로 일어나는 일:
  ┌──────────────────────────────────────────┐
  │ 단일 함수 호출 응답시간                    │
  │   스칼라: 50 ns                           │
  │   SIMD:   45 ns (오히려 거의 같음)         │
  │ 이유:                                     │
  │   호출 오버헤드 + 콜드 캐시 + 작은 데이터  │
  │   → SIMD의 처리량 이득이 드러날 시간 없음  │
  └──────────────────────────────────────────┘

같은 함수, 같은 SIMD 코드인데 배치로 호출하면:
  ┌──────────────────────────────────────────┐
  │ 1M 호출 총 처리 시간                       │
  │   스칼라: 50 ms                           │
  │   SIMD:    7 ms (~7배 빠름)                │
  │ 이유:                                     │
  │   호출 오버헤드는 분산, 데이터가 충분       │
  │   → SIMD의 처리량 이득이 누적됨            │
  └──────────────────────────────────────────┘

핵심:
  SIMD는 **처리량**을 늘린다
  **지연시간**은 거의 그대로 (또는 약간 증가할 수도)
  → 워크로드 특성을 먼저 봐야 한다
```

---

## 😱 잘못된 이해

### Before: "SIMD를 쓰면 무조건 빨라진다"

```
이 사고방식의 함정:

함정 1: 단발 호출에 SIMD 적용
  요청 1건당 4~8개 데이터 처리하는 마이크로서비스 핸들러
  → AVX-512로 바꿨지만 응답시간 그대로
  이유: 데이터가 적어 한 SIMD 명령으로 끝남 → 처리량 이득 0

함정 2: 분기가 있는 코드 SIMD화
  for (i=0; i<n; i++) if (a[i] > 0) b[i] = sqrt(a[i]);
  → 컴파일러가 못 벡터화하거나, mask를 써서 양쪽 다 계산
  결과: 항상 양쪽 다 계산 → 스칼라보다 느려질 수 있음

함정 3: 데이터 의존성이 있는 누적
  for (i=0; i<n; i++) state = update(state, a[i]);
  → 각 반복이 이전 결과에 의존
  → SIMD로 동시 처리 불가능 (Ch1-06 ILP의 한계와 같은 문제)

함정 4: AVX-512가 무조건 좋다는 믿음
  AVX-512 명령이 들어가면 CPU가 클럭을 일시적으로 낮춤(Intel)
  → 짧은 시간만 SIMD 쓸 경우 클럭 감속 손해 > SIMD 이득
  → "AVX-512 frequency penalty" 이슈

올바른 접근:
  데이터 양 × 데이터 병렬성 × 워크로드 패턴 분석 후 결정
```

---

## ✨ 올바른 이해

### After: SIMD는 처리량 도구다 — 워크로드를 분류하고 적용

```
SIMD가 빛나는 워크로드 — 처리량 우위 워크로드:

✅ 적합:
  ① 대량의 동질 데이터 (수천~수백만 요소)
  ② 데이터 간 독립적 (의존성 사슬 없음)
  ③ 분기 없음 또는 예측 가능
  ④ 메모리 접근이 연속적(streaming)
  
  예: 이미지 필터, 행렬 곱셈, 음성 신호 처리, 해시, 압축,
      체크섬, 비디오 인코딩, 머신러닝 인퍼런스, 정렬

❌ 부적합:
  ① 데이터 양이 적음 (수십 개 이하)
  ② 데이터 의존성 있음 (recurrence)
  ③ 분기가 많고 예측 불가능
  ④ 메모리 접근이 무작위 (gather/scatter)
  
  예: 트리 탐색, 그래프 탐색, 작은 JSON 파싱, 트랜잭션 처리,
      해시 테이블 룩업 (직접적으로는)

응답성 vs 배치 처리:
  지연시간 우위(p99 응답시간) → SIMD 효과 낮음
  처리량 우위(QPS, 배치 시간) → SIMD 효과 높음

CPU SIMD vs GPU 관계:
  CPU SIMD: 한 코어가 한 사이클에 8~16 요소
  GPU:      수천 SIMT 코어가 동시에 → 처리량 100배+
  → 같은 "데이터 병렬성" 원리, 규모만 다름
  → 대량 행렬 곱셈은 GPU가 압도, 작은 벡터는 CPU SIMD가 효율적
```

---

## 🔬 내부 동작 원리

### 1. 처리량과 지연시간의 분리

```
파이프라인 관점 (Ch1-02):
  명령 한 개 완료까지 = 지연시간 (Latency)
  단위 시간당 완료되는 명령 수 = 처리량 (Throughput)

  예: 5단계 파이프라인, 각 단계 1cy
    한 명령 지연시간 = 5cy
    처리량 = 1 명령/cy (파이프라인이 가득 차면)

SIMD가 바꾸는 것:
  한 명령이 처리하는 요소 수 = 4~16
  지연시간은 그대로 (한 명령의 사이클 수)
  처리량은 4~16배 (사이클당 요소 수)

벡터 명령의 지연시간:
  vmulps ymm0, ymm1, ymm2   ; AVX 부동소수점 곱셈
    지연시간: ~4~5 cycles
    처리량:   throughput 0.5 = 사이클당 2개 실행 가능
    → 한 사이클에 8 elements × 2 명령 = 16 elements

이걸로 보면:
  스칼라:  1 element / cycle (이상적)
  AVX2:    16 elements / cycle (이상적)
  AVX-512: 32 elements / cycle (이상적)
  
  → "이상적" — 실제는 메모리 대역폭, 의존성, 분기로 줄어듦
```

### 2. AVX-512 frequency penalty

```
Intel CPU의 AVX 클럭 감속 (Skylake-X / Cascade Lake 등):

  Light 명령 (스칼라, SSE):       기본 클럭 (예: 3.5 GHz)
  Medium AVX2 명령:               약간 감속 (예: 3.3 GHz)
  Heavy AVX-512 명령:             더 감속 (예: 2.8 GHz)

  → AVX-512 쓰는 동안 모든 코어의 클럭이 감속될 수 있음
  → 옆 코어의 비-SIMD 워크로드도 영향

언제 손해인가:
  AVX-512 짧게 + 다시 일반 코드:
    클럭이 감속됐다가 회복하는 데 ~수 마이크로초
    → 짧은 SIMD 가속 이득 < 일반 코드 클럭 손실
  
  서버 워크로드 1개 코어 AVX-512:
    옆 코어 7개의 일반 코드도 같이 감속
    → 전체 처리량 손해

언제 이득인가:
  배치 처리, 비디오 인코딩 등 SIMD를 계속 쓰는 워크로드:
    클럭 감속해도 SIMD 처리량 이득이 훨씬 큼

Ice Lake 이후 (Sunny Cove 마이크로아키텍처):
  AVX-512 frequency penalty 거의 사라짐
  → 새 서버 CPU에서는 이 이슈가 줄어듦

진단:
  perf stat -e cpu_clk_unhalted.thread,cpu_clk_unhalted.ref_tsc ./a.out
  → thread / ref_tsc 비율로 실제 클럭 측정
```

### 3. 응답성 워크로드 vs 배치 워크로드

```
응답성 워크로드 — 한 요청의 p50/p99 응답시간:

  웹 핸들러: GET /api/user/123
    → 인증 토큰 검증 (32B HMAC)
    → DB 1행 조회
    → JSON 직렬화
  
  SIMD를 어디 쓸 수 있을까?
    HMAC 계산? 32B는 한 AVX 명령으로 끝남 → 가속 효과 거의 없음
    JSON 직렬화? 분기 많음, 데이터 적음 → SIMD 부적합
    → SIMD 도입해도 응답시간 거의 그대로

배치 워크로드 — 총 처리 시간:

  비디오 인코딩: 1080p 영상 1시간
    → 매 프레임마다 픽셀 단위 변환
    → DCT(이산 코사인 변환), 양자화, 엔트로피 인코딩
  
  SIMD를 어디 쓸 수 있을까?
    DCT: 8×8 블록 변환 → AVX-512 한 번에 처리 → 8배 가속
    양자화: 모든 픽셀 동일 연산 → 거의 완벽한 SIMD 적합
    → 전체 인코딩 시간 5~10배 단축

규칙:
  요청당 처리하는 데이터 < SIMD 폭 → SIMD 효과 미미
  요청당 처리하는 데이터 ≫ SIMD 폭 → SIMD 효과 극대
```

### 4. 실제 워크로드별 SIMD 효과 사례

```
ML 인퍼런스 (행렬 곱셈):
  AVX-512 FMA로 8개 float 동시에 곱셈+덧셈
  실제: BLAS 라이브러리(MKL, OpenBLAS) 내부 핵심
  효과: 5~10배 (메모리 대역폭이 결국 병목)

이미지 처리 (블러, 채도 등):
  각 픽셀에 동일 연산 → 거의 완벽한 SIMD 적합
  효과: 4~8배

문자열 처리 (UTF-8 검증, JSON 파싱):
  ⚠ 분기가 많아 보이지만 SIMD 친화적 알고리즘 존재
  simdjson 라이브러리: JSON 파싱 4 GB/s 달성
  효과: 2~4배 (전용 알고리즘 필요)

암호화 (AES, SHA):
  ⚠ AES-NI는 SIMD가 아닌 전용 명령
  SHA-256: SHA Extensions로 가속
  효과: 5~20배

체크섬 (CRC32, Adler32):
  CRC32 명령 자체는 스칼라
  병렬 CRC32 (PCLMULQDQ) 사용 시 → 10배+

배열 합/내적:
  ✅ 거의 완벽한 SIMD 적합
  효과: 4~8배 (메모리 대역폭이 한계)

트리 탐색 (B-Tree):
  ⚠ 노드 내부 키 비교는 SIMD 가능
  하지만 노드 간 이동은 포인터 추적 → SIMD 불가
  실제 효과: 1.5~2배 (Ch4-03 포인터 추적과 연결)
```

### 5. CPU SIMD vs GPU SIMT

```
같은 "데이터 병렬성" 원리, 규모/모델 차이:

CPU SIMD:
  코어 1개에 SIMD 유닛 → 사이클당 8~16 elements
  레이턴시: 매우 낮음 (수십 ns)
  적합: 작은~중간 데이터, 빠른 응답이 필요한 경우

GPU SIMT (Single Instruction Multiple Thread):
  수천 개의 SIMT 코어 → 사이클당 수천 elements
  레이턴시: 높음 (수십~수백 µs 전송 비용 포함)
  적합: 거대 행렬, 딥러닝 학습/인퍼런스, 그래픽

선택 기준:
  데이터가 < ~1MB이고 자주 호출:    CPU SIMD
  데이터가 > ~10MB이고 단발 처리:     GPU
  → 데이터 전송 비용이 GPU 손익분기점을 결정

실제 사례:
  LLM 인퍼런스 토큰 1개 생성:
    KV cache가 GPU 메모리에 있어야 효율적
    → 한 번 GPU에 올린 모델은 CPU로 안 내려감
  
  웹 서버의 작은 텍스트 분류:
    GPU 호출 비용 > 모델 계산 비용
    → CPU AVX로 처리하는 게 더 빠름 (DistilBERT 등 작은 모델)
```

---

## 💻 실전 실험

### 실험 1: 데이터 크기에 따른 SIMD 효과 측정

```cpp
// vec_size_demo.cpp
#include <benchmark/benchmark.h>
#include <immintrin.h>
#include <vector>

float scalar_sum(const float* a, size_t n) {
    float s = 0;
    for (size_t i = 0; i < n; i++) s += a[i];
    return s;
}

float simd_sum(const float* a, size_t n) {
    __m256 acc = _mm256_setzero_ps();
    size_t i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 v = _mm256_loadu_ps(a + i);
        acc = _mm256_add_ps(acc, v);
    }
    float buf[8];
    _mm256_storeu_ps(buf, acc);
    float s = buf[0]+buf[1]+buf[2]+buf[3]+buf[4]+buf[5]+buf[6]+buf[7];
    for (; i < n; i++) s += a[i];
    return s;
}

static void BM_Scalar(benchmark::State& s) {
    size_t n = s.range(0);
    std::vector<float> v(n, 1.0f);
    for (auto _ : s) benchmark::DoNotOptimize(scalar_sum(v.data(), n));
}
static void BM_SIMD(benchmark::State& s) {
    size_t n = s.range(0);
    std::vector<float> v(n, 1.0f);
    for (auto _ : s) benchmark::DoNotOptimize(simd_sum(v.data(), n));
}

BENCHMARK(BM_Scalar)->Arg(8)->Arg(64)->Arg(1024)->Arg(1<<20);
BENCHMARK(BM_SIMD)->Arg(8)->Arg(64)->Arg(1024)->Arg(1<<20);
BENCHMARK_MAIN();
```

```bash
g++ -O2 -mavx2 vec_size_demo.cpp -lbenchmark -lpthread -o vec
./vec

# 예상 출력:
# n=8:      Scalar 3ns, SIMD 4ns         → SIMD 약간 느림 (오버헤드)
# n=64:     Scalar 25ns, SIMD 8ns        → SIMD 3배
# n=1024:   Scalar 400ns, SIMD 80ns      → SIMD 5배
# n=1M:     Scalar 400µs, SIMD 60µs      → SIMD 6배 (메모리 대역폭 한계)
```

### 실험 2: AVX-512 frequency penalty 확인

```bash
# 코어 0의 실제 클럭 추적
perf stat -e cycles,task-clock,cpu_clk_unhalted.thread,cpu_clk_unhalted.ref_tsc \
  taskset -c 0 ./avx512_workload

# AVX-512 명령이 들어가는 동안 cycles/ref_tsc 비율이 떨어지는지 확인

# 직접 모니터:
watch -n 0.5 'grep MHz /proc/cpuinfo | head -1'
# 실행 중 AVX-512 들어가면 클럭이 감속되는 게 보임 (Skylake-X 등)

# Ice Lake / Sapphire Rapids 등은 거의 감속 없음
```

### 실험 3: simdjson — SIMD가 빛나는 비전통적 워크로드

```bash
# JSON 파싱은 분기가 많아 SIMD 부적합 같지만,
# simdjson은 전용 알고리즘으로 4 GB/s 달성

git clone https://github.com/simdjson/simdjson
cd simdjson && cmake -B build && cmake --build build

# 벤치마크
./build/benchmark/parsingcompetition twitter.json
# 출력 예: simdjson: 3.5 GB/s, rapidjson: 0.6 GB/s, jsoncpp: 0.2 GB/s
# → 같은 작업을 ~6배 빠르게 (SIMD + 알고리즘 설계)
```

### 실험 4: GPU vs CPU 손익분기점

```python
# pytorch_breakeven.py
import torch, time

for n in [128, 1024, 16384, 1048576]:
    a = torch.randn(n, n)
    b = torch.randn(n, n)
    
    # CPU
    t = time.perf_counter()
    for _ in range(10): c = a @ b
    cpu_ms = (time.perf_counter() - t) * 100
    
    # GPU
    if torch.cuda.is_available():
        ag, bg = a.cuda(), b.cuda()
        torch.cuda.synchronize()
        t = time.perf_counter()
        for _ in range(10): cg = ag @ bg
        torch.cuda.synchronize()
        gpu_ms = (time.perf_counter() - t) * 100
        print(f"n={n}: CPU {cpu_ms:.2f}ms, GPU {gpu_ms:.2f}ms")
    
# 예상: n=128 → CPU가 더 빠르거나 비슷
#       n=16384 → GPU가 압도적
#       → 전송 비용 포함하면 손익분기점은 더 큰 데이터
```

---

## 📊 성능 비교

```
워크로드별 SIMD(AVX2) 효과:

워크로드                       데이터 크기      효과(스칼라 대비)
─────────────────────────────────────────────────────────────
배열 합/내적                    > 1KB           4~8배 (메모리 BW 한계)
이미지 픽셀 변환                전체 이미지      6~8배
행렬 곱셈 (BLAS)                > 256×256       8~16배
DCT/FFT                         배치 처리        5~10배
SHA-256 (SHA Extensions)        > 64B           10~20배
AES-NI 암호화                   > 16B           5~20배 (전용 명령)
JSON 파싱 (simdjson)            > 64KB          3~6배 (전용 알고리즘)
UTF-8 검증                      > 1KB           3~5배
B-Tree 노드 내 검색             16~64 키        1.5~2배
랜덤 hash 룩업                  -               거의 0배
트랜잭션 처리                    -               거의 0배
작은 응답 핸들러                 < 64B          오히려 손해 가능

규칙:
  데이터 양 ≫ SIMD 폭 → 처리량 이득 극대
  데이터 의존성 적음 → 이득 극대
  분기 없음 → 자동 벡터화도 됨
  메모리 streaming → 메모리 대역폭이 종종 진짜 병목
```

---

## ⚖️ 트레이드오프

```
SIMD 도입 트레이드오프:

✅ 도입할 때:
  - 데이터 병렬성이 명확한 핫스팟 (perf로 확인)
  - 배치 처리, 미디어, ML, 압축, 암호화 등 처리량 중심
  - 데이터 양이 SIMD 폭의 10배 이상

❌ 피해야 할 때:
  - 응답시간이 ms 미만이고 처리 데이터 적은 핸들러
  - 분기 많고 데이터 패턴 불규칙
  - 다른 코어에 영향 줄 수 있는 AVX-512 (구세대 CPU)
  - 이식성이 우선인 라이브러리 (스칼라 + 자동 벡터화 의존)

코드 복잡도:
  인트린식 코드는 가독성 떨어짐
  → 핫스팟에만 적용
  → 자동 벡터화로 충분한지 먼저 확인 (Ch5-02)

이식성:
  AVX-512 → 모든 CPU에서 동작 X
  __builtin_cpu_supports로 런타임 dispatch
  또는 ifunc(GCC) / multi-versioning

GPU와의 경계:
  데이터 > 10MB 이고 단발/긴 작업 → GPU 고려
  데이터 < 1MB 또는 매우 자주 호출 → CPU SIMD
  중간 영역은 측정으로 결정
```

---

## 📌 핵심 정리

```
처리량 vs 지연시간 핵심:

SIMD의 본질:
  처리량 도구 — 단위 시간당 처리량 4~16배
  지연시간은 거의 그대로 (한 명령의 사이클 수는 비슷)

워크로드 분류:
  처리량 우위 (배치, 미디어, ML, 압축, 암호) → SIMD 큰 효과
  지연시간 우위 (작은 응답 핸들러, 트랜잭션) → SIMD 효과 미미

부적합 신호:
  데이터 의존성 (recurrence)
  예측 불가능한 분기
  무작위 메모리 접근 (gather/scatter는 느림)
  데이터 양이 SIMD 폭보다 작음

AVX-512 주의:
  Skylake-X/Cascade Lake: frequency penalty 있음
  Ice Lake 이후: 거의 사라짐
  perf stat의 cpu_clk_unhalted.thread/ref_tsc로 확인

CPU SIMD vs GPU:
  같은 "데이터 병렬성" 원리, 규모 차이
  데이터 < 1MB / 자주 호출 → CPU SIMD
  데이터 ≫ 1MB / 단발 → GPU
  손익분기점은 측정으로

연결 챕터:
  Ch1-06 ILP의 한계 → 의존성 사슬이 SIMD에도 영향
  Ch2-05 데이터 레이아웃 → SoA가 SIMD 효율 극대
  Ch4-04 branchless → SIMD 마스킹의 기초
  Ch7-03 루프라인 모델 → memory-bound vs compute-bound 판정
```

---

## 🤔 생각해볼 문제

**Q1.** 결제 API 핸들러(요청당 ~20개 정수 검증)에 AVX2를 도입했더니 p50 응답시간이 거의 그대로다. 왜이며, 어떤 워크로드에 SIMD가 효과적이었을까?

<details>
<summary>해설 보기</summary>

**왜 효과 없음**:
- 데이터 양이 SIMD 폭(AVX2 = 8개 int)과 비슷하거나 약간 더 큼
- 한 SIMD 명령 1~3개로 끝나는 작업 → 처리량 이득 누적 불가
- 함수 호출 오버헤드, 캐시 콜드, JIT 오버헤드 등이 응답시간의 대부분
- 결제 핸들러 전체 시간은 DB I/O, 네트워크, 로깅이 지배 — CPU 산술 비중이 작음

**SIMD가 효과적인 곳**:
- 같은 결제 시스템에서도 **사기 탐지 배치(Fraud Detection)** : 매 5분마다 수십만 트랜잭션을 ML 모델로 평가 → 처리량 큰 효과
- **일일 정산 리포트**: 수백만 거래에 대한 합계/평균/표준편차 계산
- **암호화/서명 검증**: 대량의 HMAC 일괄 검증

규칙: 응답시간이 짧고 처리 데이터가 적은 핸들러는 SIMD 후보가 아니다. 배치, 분석, 미디어, ML이 SIMD 후보다.

</details>

---

**Q2.** AVX-512가 도입된 서버에서 일부 마이크로서비스가 옆 마이크로서비스 처리량까지 떨어뜨린다. 무엇이 일어났으며, 어떻게 진단·해결하는가?

<details>
<summary>해설 보기</summary>

**원인 — AVX-512 frequency penalty**:
- 구세대 Skylake-X / Cascade Lake CPU는 AVX-512 명령이 들어가면 해당 코어 또는 같은 다이의 모든 코어의 클럭을 일시적으로 낮춤
- 컨테이너 환경에서 옆 컨테이너의 일반 코드 워크로드도 같이 감속
- "전체 처리량 감소"로 드러남

**진단**:
```bash
# 클럭 추적
perf stat -e cpu_clk_unhalted.thread,cpu_clk_unhalted.ref_tsc -p <PID>
# thread / ref_tsc 비율이 평소 대비 떨어지면 클럭 감속 발생

# AVX-512 사용 코드 식별
perf record -e fp_arith_inst_retired.512b_packed_single -p <PID>
perf report
```

**해결**:
1. **컴파일러 옵션 조정**: `-march=native -mno-avx512f` 또는 `-mprefer-vector-width=256`로 AVX2까지만 사용
2. **JIT 옵션**: JVM `-XX:UseAVX=2`로 AVX-512 비활성화
3. **격리**: AVX-512 워크로드를 별도 노드/코어로 격리 (taskset, cgroup)
4. **CPU 업그레이드**: Ice Lake / Sapphire Rapids 이후는 frequency penalty 거의 없음

**예방**:
- 라이브러리 의존성에서 의도치 않게 AVX-512가 활성화되는지 확인 (예: OpenBLAS, MKL 빌드 옵션)
- 운영 환경 CPU 세대 고려해서 컴파일 타깃 정하기

</details>

---

**Q3.** ML 인퍼런스 서비스를 만든다. 모델은 작고(distilBERT, ~250MB), 요청당 토큰 ~50개를 처리한다. CPU SIMD로 갈지 GPU로 갈지의 판단 기준은?

<details>
<summary>해설 보기</summary>

**결론 — 이 경우 CPU SIMD가 유리**:

**판단 기준**:

1. **요청당 데이터 양**: 토큰 50개 × hidden_size(768) × layer(6) ≈ 200KB 정도의 활성화. GPU 메모리로 전송하는 비용이 계산 비용보다 비쌀 수 있음.

2. **응답시간 SLA**: 요청당 ms 단위 응답 필요 → GPU 호출 오버헤드(수십 µs~ms) 부담

3. **요청 빈도**: 초당 100~1000 요청 → GPU는 배치하지 않으면 IDLE 시간 많음 (낮은 활용도)

4. **모델 크기**: 250MB 정도면 CPU L3 캐시(~30MB)에는 안 들어가지만 DRAM에는 무난히 적재. CPU SIMD + AVX-512로 행렬 곱셈 가속 가능.

5. **운영 비용**: GPU 인스턴스는 비싸고 IDLE 시간이 비용. CPU는 다른 워크로드와 공유 가능.

**CPU SIMD가 유리한 신호**:
- 작은 모델 (DistilBERT, MobileBERT, GPT-2 small 등)
- 응답시간 우위 워크로드
- 요청 빈도가 GPU 배치 효율을 만들 만큼 크지 않음
- ONNX Runtime, OpenVINO, llama.cpp 등 CPU 최적화 SIMD 런타임 활용

**GPU가 유리한 신호**:
- 큰 모델 (LLM 7B+)
- 배치 가능 (offline inference, 비동기 처리)
- 토큰 생성 속도가 처리량 우위 (스트리밍보다 throughput 중심)

**중간 영역 — 측정으로 결정**:
- 같은 모델로 CPU SIMD vs GPU 부하 테스트 후 p99 응답시간과 시간당 비용 비교

이 챕터의 핵심: 워크로드 특성(처리량 vs 지연시간)이 가속기 선택의 기준이지, "ML이니까 GPU"가 답이 아니다.

</details>

---

<div align="center">

**[⬅️ 이전: 데이터 레이아웃과 벡터화](./04-data-layout-for-simd.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 멀티코어와 NUMA ➡️](../multicore-and-numa/01-multicore-topology.md)**

</div>
