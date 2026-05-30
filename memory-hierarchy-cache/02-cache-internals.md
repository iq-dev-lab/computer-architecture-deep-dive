# 캐시 동작 원리 — 라인·태그·집합 연관

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 캐시가 "바이트 단위"가 아닌 "캐시라인(64B) 단위"로 동작하는 이유는 무엇인가?
- CPU가 메모리 주소를 태그(Tag)/인덱스(Index)/오프셋(Offset)으로 분해하는 방법은?
- 직접 매핑(Direct-Mapped), 집합 연관(Set-Associative), 완전 연관(Fully Associative)의 트레이드오프는?
- 4-way 집합 연관 캐시에서 "4-way"가 정확히 무엇을 의미하는가?
- LRU와 PLRU 교체 정책은 어떻게 동작하며 왜 하드웨어에서 PLRU를 선호하는가?
- Conflict miss가 왜 2의 거듭제곱 크기 배열에서 집중적으로 발생하는가?

---

## 🔍 왜 이 개념이 중요한가

### "캐시 히트율 92%인데 왜 여전히 느리지?"

```
흔히 놓치는 사실:
  캐시 히트율이 높아도 접근 패턴에 따라 성능이 크게 다름
  그 이유는 캐시의 내부 구조에 있다

예시 — 2MB 배열 두 개를 동시에 순회:
  double A[32768];  // 256KB
  double B[32768];  // 256KB

  for (int i = 0; i < N; i++) {
      result += A[i] * B[i];
  }

  L2 캐시 = 256KB, 8-way set-associative
  A와 B가 정확히 같은 캐시 인덱스에 매핑될 경우:
    → A[i] 로드 → B[i] 로드 → A[i+1] 로드 시 A[i]가 퇴출!
    → Conflict miss 발생 → 히트율이 이론보다 낮아짐

  원인을 이해하려면:
    주소가 캐시에 어떻게 매핑되는지 알아야 한다
    "태그/인덱스/오프셋" 분해를 이해해야 한다
    이것이 Ch2-04 Conflict miss의 기반 지식

캐시 내부 구조를 알면:
  왜 배열 크기를 2의 거듭제곱에서 살짝 피해야 하는지
  왜 캐시 미스 종류가 다른지 (Ch2-04 3C 모델)
  False Sharing이 왜 발생하는지 (Ch3-02)
  — 모두 이 구조에서 파생된다
```

---

## 😱 잘못된 이해

### Before: "캐시는 자주 쓴 데이터를 저장해두는 버퍼다"

```
단순화된 모델:
  캐시 = 해시맵처럼 임의의 주소를 저장 가능
  어떤 주소든 빈 슬롯에 들어갈 수 있음

실제:
  캐시는 집합(Set)으로 나뉘어져 있고
  각 주소는 오직 특정 집합에만 들어갈 수 있다

  이 제약 때문에:
    ① 특정 집합이 꽉 차면 그 집합에만 미스가 집중됨
    ② 전체 캐시의 대부분이 비어 있어도 특정 패턴에서 미스 발생
    ③ 2의 거듭제곱 크기 배열이 같은 집합으로 몰리는 이유

  잘못 이해하면 놓치는 것:
    "캐시 미스가 많다 → 워킹셋이 캐시보다 크다"
    → 반드시 그렇지 않다
    → 워킹셋이 캐시의 절반이어도 접근 패턴에 따라
       Conflict miss로 미스율이 높아질 수 있다
```

---

## ✨ 올바른 이해

### After: 캐시는 집합 연관 구조로 설계된 하드웨어

```
캐시의 물리적 구조 (32KB L1, 8-way 기준):

  총 캐시라인 수 = 32KB / 64B = 512개
  Way 수 = 8
  집합(Set) 수 = 512 / 8 = 64개

  시각화:
       Way0     Way1     Way2  ... Way7
  Set0  [라인]  [라인]  [라인] ... [라인]   8개 슬롯
  Set1  [라인]  [라인]  [라인] ... [라인]
  Set2  [라인]  [라인]  [라인] ... [라인]
  ...
  Set63 [라인]  [라인]  [라인] ... [라인]

주소 분해 (64비트 주소):
  [63     12][11    6][5      0]
      태그     인덱스    오프셋

  오프셋 (bit 5~0):  6비트 → 64B 캐시라인 내 위치
  인덱스 (bit 11~6): 6비트 → 64개 집합 중 어느 집합
  태그 (bit 63~12):  나머지 → "이 집합 안에서 어느 주소"

  주소 → 집합 결정 → Way 중 하나에 저장
  같은 인덱스를 가진 주소들은 같은 집합에만 들어갈 수 있음
```

---

## 🔬 내부 동작 원리

### 1. 캐시라인(64B)이 단위인 이유

```
단순한 질문: 왜 1바이트 단위로 캐시하지 않는가?

이유 1 — DRAM은 burst 전송이 훨씬 빠르다:
  DRAM 접근 = 행(Row) 활성화 + 열(Column) 읽기
  단일 바이트 읽기: 행 활성화 비용 + 1바이트 = 낭비
  64바이트 burst:   행 활성화 비용 + 64바이트 = 효율적
  → 어차피 비용이 비슷하다면 64B를 한 번에 가져오는 게 이득

이유 2 — 공간 지역성(Spatial Locality) 활용:
  int a[16];  // 64B (캐시라인 1개에 딱 맞음!)
  for (int i = 0; i < 16; i++) sum += a[i];

  a[0] 캐시 미스 → 64B 로드 → a[1]~a[15] 동시에 캐시에 올라옴
  → a[1]~a[15]는 모두 캐시 히트

  만약 1바이트씩 캐시했다면:
    a[0]~a[15] 모두 별도 미스 → 16번 DRAM 접근
    → 64배의 미스 비용

이유 3 — 하드웨어 단순화:
  64B 정렬된 블록 단위로 관리 → 주소 비교가 단순
  오프셋 6비트로 64B 내 위치 특정 → 회로 간단

캐시라인 64B의 실용적 의미:
  int (4B) 16개가 하나의 캐시라인
  double (8B) 8개가 하나의 캐시라인
  char (1B) 64개가 하나의 캐시라인
  → 인접한 요소를 순차 접근하면 캐시 미스 1번에 16개 int 확보
```

### 2. 주소 분해: 태그/인덱스/오프셋

```c
/*
 * 32KB, 8-way set-associative, 캐시라인 64B인 L1 캐시 예시
 *
 * 64B 오프셋 = 6비트 (2^6 = 64)
 * 64 집합  = 6비트 (2^6 = 64)
 * 태그 = 나머지 52비트
 *
 * 주소 레이아웃:
 * [63      12][11      6][5       0]
 *    태그(52)   인덱스(6)  오프셋(6)
 */

// 주소 분해 계산 예시
void decode_address(uintptr_t addr) {
    uintptr_t offset = addr & 0x3F;        // bit[5:0]   = 6비트
    uintptr_t index  = (addr >> 6) & 0x3F; // bit[11:6]  = 6비트
    uintptr_t tag    = addr >> 12;          // bit[63:12] = 52비트

    printf("주소: 0x%016lx\n", addr);
    printf("  오프셋: %2lu (캐시라인 내 %lu번째 바이트)\n", offset, offset);
    printf("  인덱스: %2lu (Set %lu 에 저장)\n", index, index);
    printf("  태그:   0x%lx (동일 Set 내 구별자)\n", tag);
}

// 예시:
// addr = 0x00007fff12345678
//   bit[5:0]  = 0x38 = 56 → 캐시라인 내 56번째 바이트
//   bit[11:6] = 0x15 = 21 → Set 21에 매핑
//   bit[63:12] = 태그값

// 핵심: 두 주소의 bit[11:6]이 같으면 같은 Set에 경쟁
// 같은 Set에 여러 주소가 몰리면 Conflict miss 발생
```

### 3. Direct-Mapped vs Set-Associative vs Fully Associative

```
Direct-Mapped (1-way):
  각 주소가 딱 하나의 슬롯에만 들어갈 수 있음

       슬롯0  슬롯1  슬롯2  ...  슬롯N-1
  Set0  [라인]
  Set1         [라인]
  ...
  SetN                      ...  [라인]

  장점: ✅ 하드웨어 단순 (비교기 1개만 필요)
        ✅ 검색 속도 빠름 (1회 비교)
  단점: ❌ 같은 인덱스에 매핑된 2개 주소가 번갈아 접근 시
           → Thrashing: 매번 퇴출-재로드 반복
           → 8-way보다 Conflict miss 압도적으로 많음

8-way Set-Associative (L1에 일반적):
  각 Set에 8개 슬롯 — 같은 인덱스 주소 8개까지 공존 가능

       Way0  Way1  Way2  Way3  Way4  Way5  Way6  Way7
  Set0 [라인][라인][라인][라인][라인][라인][라인][라인]
  Set1 [라인]...
  ...

  장점: ✅ Conflict miss 크게 감소 (8개까지 공존)
        ✅ Direct-Mapped 대비 실질적 히트율 향상
  단점: ❌ 8개 Way 비교기 병렬 필요 → 면적/전력 증가
        ❌ 지연 소폭 증가 (비교기 8개 동시)

Fully Associative (TLB에 사용):
  어떤 슬롯에도 들어갈 수 있음 — 인덱스 제약 없음

  장점: ✅ Conflict miss 완전 제거
        ✅ 교체 정책의 자유도 최대
  단점: ❌ 모든 슬롯과 동시 비교 필요 → 하드웨어 비용 폭증
        ❌ 수십 개 이상은 현실적으로 구현 불가
        → L1/L2/L3은 Set-Associative 사용
        → TLB는 small fully-associative (32~64 엔트리)

현실 CPU 캐시 구성:
  L1d:  32KB,  8-way,  64 sets,  ~4 사이클
  L2:   256KB, 4-way,  1024 sets, ~12 사이클
  L3:   8MB,   16-way, 8192 sets, ~40 사이클
  (Intel Skylake 기준, CPU 모델마다 다름)
```

### 4. LRU와 PLRU 교체 정책

```
교체 정책: 새 라인 로드 시 어느 Way를 퇴출할 것인가?

LRU (Least Recently Used):
  N-way 캐시에서 가장 오래 전에 사용된 Way를 퇴출
  
  8-way 예시:
    Way 사용 시간:  W0(100) W1(50) W2(80) W3(30) ...
    새 로드 시: W3(30)이 가장 오래됨 → W3 퇴출
  
  장점: ✅ 최적에 가까운 히트율 (이론적 최선)
  단점: ❌ 정확한 LRU를 위해 N개 Way의 사용 시간 추적 필요
        ❌ 8-way LRU = 8! = 40320 순서 조합 → 3비트×8 = 24비트 오버헤드
        ❌ 대형 캐시(L3)에서 하드웨어 구현 비용 과다

PLRU (Pseudo-LRU, 실제 CPU가 사용):
  완전 LRU 대신 비트 트리로 근사

  8-way PLRU 비트 트리:
            [0]        ← 루트 비트
           /   \
         [0]   [1]     ← 레벨 2 비트
        / \   / \
      [0][0][1][0]     ← 리프 → 각 Way 쌍

  비트 = 0: 왼쪽이 더 최근, 1: 오른쪽이 더 최근
  퇴출 후보: 비트 0→왼쪽→다시 왼쪽/오른쪽 탐색
  
  예: 루트=0 → 왼쪽 서브트리 선택
      레벨2[0]=0 → 왼쪽 선택 → Way0 퇴출
  
  장점: ✅ 8-way에 7비트만 필요 (vs LRU의 24비트)
        ✅ 히트율은 LRU와 95% 이상 동일
  단점: ❌ 정확한 LRU가 아님 → 최악의 경우 퇴출 선택 차선

실제 동작 비교:
  LRU:  [W0, W1, W2, W3, W4, W5, W6, W7] 접근 후 새 로드
        → W0이 가장 오래됨 → W0 퇴출 (완벽한 결정)
  PLRU: 비트 트리 기반 → 대부분 동일한 결정
        특정 접근 패턴에서 비최적 선택 가능 (드물게 발생)
```

### 5. 캐시 조회 과정 — 파이프라인

```
주소 → 캐시 히트/미스 결정 과정:

Step 1: 인덱스 추출 (bit[11:6])
  → 해당 Set의 모든 Way 태그를 병렬로 읽기

Step 2: 태그 비교 (병렬)
  Way0 태그 == 주소의 태그?
  Way1 태그 == 주소의 태그?  ← 8개 동시 비교
  ...
  Way7 태그 == 주소의 태그?

Step 3: Valid 비트 확인
  태그 일치 Way의 Valid 비트가 1인가?

Step 4: 히트/미스 결정
  ✅ 일치하는 Way 있음 → 히트 → 오프셋으로 데이터 추출
  ❌ 일치하는 Way 없음 → 미스 → 하위 계층에 요청

Step 5: 미스 시 교체
  어느 Way를 퇴출할지 PLRU로 결정
  새 캐시라인(64B) 로드
  태그/Valid 비트 갱신
  PLRU 비트 갱신

총 L1 캐시 히트 지연: ~4 사이클
  (태그 읽기 + 비교 + 데이터 반환을 파이프라인으로 처리)

VIPT (Virtually Indexed Physically Tagged):
  대부분의 현대 CPU L1이 사용
  가상 주소의 인덱스 비트를 사용해 조회 시작 → 병렬로 TLB
  물리 주소의 태그로 최종 확인 → TLB 지연 숨김
```

---

## 💻 실전 실험

### 실험 1: getconf로 실제 캐시 파라미터 확인

```bash
# Linux: 캐시 크기/연관도/라인 크기 확인
getconf -a | grep -i cache
# 출력 예시:
# LEVEL1_ICACHE_SIZE                 32768      (32KB L1 명령어)
# LEVEL1_ICACHE_ASSOC                8          (8-way)
# LEVEL1_ICACHE_LINESIZE             64         (64B 라인)
# LEVEL1_DCACHE_SIZE                 32768      (32KB L1 데이터)
# LEVEL1_DCACHE_ASSOC                8          (8-way)
# LEVEL1_DCACHE_LINESIZE             64         (64B 라인)
# LEVEL2_CACHE_SIZE                  262144     (256KB L2)
# LEVEL2_CACHE_ASSOC                 4          (4-way)
# LEVEL3_CACHE_SIZE                  8388608    (8MB L3)
# LEVEL3_CACHE_ASSOC                 16         (16-way)

# 또는 sysfs에서 직접 읽기
cat /sys/devices/system/cpu/cpu0/cache/index0/size
cat /sys/devices/system/cpu/cpu0/cache/index0/ways_of_associativity
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
cat /sys/devices/system/cpu/cpu0/cache/index0/type  # Data / Instruction / Unified

# CPUID로 자세한 정보 (x86-64)
cpuid -1 | grep -A 4 "cache"
```

### 실험 2: Conflict miss 재현 — 같은 Set에 매핑되는 배열

```c
// conflict_miss.c
// 같은 캐시 인덱스에 매핑되는 주소를 번갈아 접근하면 Conflict miss 발생
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

// L2: 256KB, 4-way → Set 수 = 256KB / (4 × 64B) = 1024 sets
// 동일 인덱스 = 256KB(=262144B) 간격의 주소들
#define L2_SIZE  (256 * 1024)
#define STRIDE   L2_SIZE  // 같은 Set에 매핑
#define N        (1 << 24)  // 16M 반복

int buffer[L2_SIZE * 5 / sizeof(int)];

int main() {
    struct timespec t0, t1;
    volatile long long sum = 0;

    // 패턴 A: L2 크기 절반 순차 접근 → 대부분 L2 히트
    clock_gettime(CLOCK_MONOTONIC, &t0);
    int half = L2_SIZE / 2 / sizeof(int);
    for (int iter = 0; iter < N / half; iter++)
        for (int i = 0; i < half; i++)
            sum += buffer[i];
    clock_gettime(CLOCK_MONOTONIC, &t1);
    printf("순차 (L2 절반): %ld ms\n",
        (t1.tv_sec - t0.tv_sec) * 1000 +
        (t1.tv_nsec - t0.tv_nsec) / 1000000);

    // 패턴 B: L2_SIZE 간격으로 5개 주소 번갈아 접근
    // → 모두 같은 Set에 매핑 → 4-way 초과 → Conflict miss!
    clock_gettime(CLOCK_MONOTONIC, &t0);
    int* ptrs[5];
    for (int i = 0; i < 5; i++)
        ptrs[i] = buffer + i * (STRIDE / sizeof(int));
    for (int iter = 0; iter < N / 5; iter++)
        for (int i = 0; i < 5; i++)
            sum += *ptrs[i];
    clock_gettime(CLOCK_MONOTONIC, &t1);
    printf("Conflict (L2 초과): %ld ms\n",
        (t1.tv_sec - t0.tv_sec) * 1000 +
        (t1.tv_nsec - t0.tv_nsec) / 1000000);

    printf("(합산: %lld, 최적화 방지)\n", sum);
    return 0;
}
// gcc -O2 -o conflict conflict_miss.c && ./conflict
// 예상: Conflict 패턴이 3~5배 느림 (동일 총 접근 수임에도)
```

### 실험 3: cachegrind로 캐시 미스 상세 분석

```bash
# cachegrind 실행 — 캐시 시뮬레이션
valgrind --tool=cachegrind \
         --I1=32768,8,64 \
         --D1=32768,8,64 \
         --LL=8388608,16,64 \
         ./conflict

# 출력:
# I   refs:      ...
# I1  misses:    ...
# LLi misses:    ...
# D   refs:      1,006,632,960  (1G 데이터 참조)
# D1  misses:         500,000  (0.05% — 순차 패턴)
# LLD misses:          10,000  (아주 적음)

# 주석 파일 생성 후 분석
cg_annotate cachegrind.out.<pid>

# 실행 중인 프로세스의 캐시 이벤트 perf로 확인
perf stat -e L1-dcache-load-misses,L2-load-misses,LLC-load-misses ./conflict
```

---

## 📊 성능 비교

```
캐시 연관도별 Conflict miss 비율 (L2 256KB 기준, 같은 인덱스 2개 주소 번갈아 접근):

  연관도           Conflict miss   히트율    하드웨어 비용
  ─────────────────────────────────────────────────────────
  Direct-Mapped    매우 높음        60%       최소 (비교기 1개)
  2-way            중간             85%       낮음 (비교기 2개)
  4-way            낮음             97%       중간 (비교기 4개)
  8-way (L1 표준)  매우 낮음        99%+      높음 (비교기 8개)
  Fully Assoc.     없음             100%      구현 불가 (대형 캐시)
  ─────────────────────────────────────────────────────────

교체 정책 비교 (8-way, 표준 접근 패턴):
  LRU 이론값:    99.5% 히트율 (기준선)
  PLRU 실제값:   99.3% 히트율 (LRU 대비 0.2% 손실, 무시 가능)
  Random:        97.8% 히트율 (교체 정책 없음 기준)
  → PLRU가 LRU와 거의 동등하면서 하드웨어 면적 3배 절감

캐시라인 크기별 영향:
  32B:  공간 지역성 절반 → 미스 2배 (int 8개만 한 번에)
  64B:  표준 — int 16개, double 8개 (현재 대부분의 CPU)
  128B: 공간 지역성 2배 → 작은 구조체도 대부분 한 라인
        단, 사용 않는 데이터도 로드 → 대역폭 낭비 가능
```

---

## ⚖️ 트레이드오프

```
캐시 설계 트레이드오프:

연관도 높임:
  ✅ Conflict miss 감소 → 히트율 향상
  ✅ 2의 거듭제곱 크기 배열에서 강건
  ❌ 비교기 수 증가 → 다이 면적/전력 증가
  ❌ 캐시 접근 지연 소폭 증가 (8-way 비교기 병렬화 필요)

캐시 크기 증가:
  ✅ Capacity miss 감소 → 더 큰 워킹셋 지원
  ❌ SRAM 면적 증가 → 단가 상승, 열 발생
  ❌ 크기가 클수록 접근 지연 증가 (물리적 거리)

캐시라인 크기 증가:
  ✅ 공간 지역성 더 잘 활용 (더 많은 인접 데이터 사전 로드)
  ✅ DRAM burst 효율 증가
  ❌ 사용하지 않는 데이터도 로드 → 대역폭 낭비
  ❌ False Sharing 발생 영역 증가 (Ch3-02 참조)

프로그래머 관점 결론:
  구조는 바꿀 수 없으므로 — 구조를 이해하고 맞춰라
  ✅ 2의 거듭제곱 크기 배열은 살짝 패딩 (Conflict 회피)
  ✅ 같은 인덱스에 매핑될 데이터를 함께 배치 금지
  ✅ 구조체를 64B 경계에 정렬 (캐시라인 하나에 맞춤)
  ✅ 핫 데이터는 구조체 앞부분에 (첫 캐시라인에 집중)
```

---

## 📌 핵심 정리

```
캐시 내부 구조 핵심:

캐시라인(64B)이 기본 단위인 이유:
  DRAM burst 전송 효율 + 공간 지역성 활용
  int 16개 = 1 캐시라인 → 순차 접근 시 미스 1/16로 감소

주소 분해 (32KB, 8-way, 64B 기준):
  bit[5:0]   = 오프셋 (64B 내 위치)
  bit[11:6]  = 인덱스 (64개 집합 중 어느 집합)
  bit[63:12] = 태그   (집합 내 Way 구별)

연관도 트레이드오프:
  Direct-Mapped: 빠르지만 Conflict miss 많음
  Set-Associative (8-way): 표준, Conflict 거의 없음
  Fully Associative: Conflict 없음, 큰 캐시엔 구현 불가

교체 정책:
  LRU: 이론적 최적, 하드웨어 비용 높음
  PLRU: LRU의 99% 효율 + 7비트만 사용 → 실제 CPU 선택

실무 연결:
  Conflict miss: 2의 거듭제곱 배열 크기 피하기 (Ch2-04)
  False Sharing: 같은 캐시라인의 다른 변수 (Ch3-02)
  TLB도 Fully Associative 캐시 (Ch2-06)
```

---

## 🤔 생각해볼 문제

**Q1.** L1 캐시가 32KB, 8-way set-associative, 64B 캐시라인이라면 집합(Set) 수는 몇 개인가? 그리고 두 주소 `0x1000`과 `0x9000`은 같은 집합에 매핑되는가?

<details>
<summary>해설 보기</summary>

**집합 수 계산**:
- 전체 캐시라인 수 = 32KB / 64B = 512개
- Way 수 = 8
- 집합 수 = 512 / 8 = **64개**

**주소 분해**:
- 오프셋 비트: 6비트 (64B = 2⁶)
- 인덱스 비트: 6비트 (64 집합 = 2⁶)
- 인덱스는 bit[11:6]을 사용

`0x1000`의 인덱스: `0x1000 >> 6 & 0x3F = 0x40 & 0x3F = 0x00` → 인덱스 0

`0x9000`의 인덱스: `0x9000 >> 6 & 0x3F = 0x240 & 0x3F = 0x00` → 인덱스 0

두 주소 모두 인덱스 0 → **같은 집합(Set 0)에 매핑**됩니다. 태그만 다릅니다. 이 두 주소를 번갈아 접근하면 같은 집합의 Way를 경쟁합니다. 단, 8-way이므로 이 2개 주소는 충분히 공존 가능합니다. 같은 Set에 9개 이상의 주소를 동시에 접근할 때 Conflict miss가 발생합니다.

</details>

---

**Q2.** Direct-Mapped 캐시(1-way)가 집합 연관 캐시보다 같은 용량에서 접근 속도가 빠를 수 있다고 알려져 있다. 그렇다면 왜 현대 CPU의 L1 캐시는 Direct-Mapped를 사용하지 않는가?

<details>
<summary>해설 보기</summary>

Direct-Mapped가 빠른 이유는 비교기가 1개뿐이라 태그 비교가 단순하기 때문입니다. 하지만 현대 CPU가 8-way를 선택하는 이유는:

**1. Conflict miss의 치명적 영향**: Direct-Mapped에서는 같은 인덱스를 가진 두 주소가 번갈아 접근되면 무조건 미스가 발생합니다(Thrashing). 반면 8-way는 8개까지 공존 가능합니다. L1 미스 한 번에 L2 접근 비용(~8 사이클)이 발생하는데, 이는 비교기 추가 비용보다 훨씬 큽니다.

**2. 현대 CPU의 파이프라인**: 8개 비교기를 병렬 실행하므로 실질적인 지연 차이가 매우 작습니다. 1-way의 "단순한 비교기 1개"와 8-way의 "8개 병렬 비교기"의 지연 차이는 최신 공정에서 0~1 사이클 수준입니다.

**3. 컴파일러/운영체제의 임의 배치**: 컴파일러나 OS의 메모리 할당 위치를 프로그래머가 완전히 제어할 수 없으므로, Direct-Mapped에서는 우연한 인덱스 충돌이 빈번합니다. 8-way는 이 "우연한 충돌"에 훨씬 강건합니다.

</details>

---

**Q3.** 캐시라인 크기가 현재 64B에서 128B로 두 배 늘어난다면, 어떤 워크로드의 성능이 향상되고 어떤 워크로드의 성능이 저하되는가?

<details>
<summary>해설 보기</summary>

**성능 향상 워크로드**:
- 순차 접근 패턴 (배열 순회): 미스 1번에 128B 로드 → 2배 많은 데이터 사전 확보
- 구조체가 64B~128B인 경우: 전체 구조체가 캐시라인 1개에 들어감
- 공간 지역성이 높은 코드 전반

**성능 저하 워크로드**:
- **랜덤 접근**: 128B를 로드하지만 그 중 1개만 사용 → 대역폭 낭비 2배
- **False Sharing**: 서로 다른 스레드가 다른 변수를 다루지만 같은 128B 라인에 있을 확률 2배 증가 → MESI 무효화 더 빈번 (Ch3-02)
- **작은 구조체 배열**: 구조체가 8B라면 128B 라인에 16개가 들어가는데, 1개만 접근해도 15개의 데이터를 쓸모없이 로드 → 캐시 공간 낭비

**결론**: 공간 지역성이 높은 순차 접근 코드에는 유리, 랜덤 접근이나 멀티스레드 환경에서는 불리합니다. 64B는 다양한 워크로드에서 경험적으로 선택된 균형점입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 메모리 계층 지연](./01-memory-hierarchy-latency.md)** | **[홈으로 🏠](../README.md)** | **[다음: 지역성 ➡️](./03-locality.md)**

</div>
