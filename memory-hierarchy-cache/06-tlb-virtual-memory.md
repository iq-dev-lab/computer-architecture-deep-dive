# TLB와 가상 메모리 — 페이지 테이블 워크

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 프로세스가 보는 "가상 주소"가 어떻게 물리 메모리 주소로 변환되는가?
- x86-64의 4단계 페이지 테이블 워크(PML4→PDPT→PD→PT)는 한 번에 몇 번의 I/O를 일으키는가?
- TLB(Translation Lookaside Buffer) 미스가 L1 캐시 미스보다 비싼 이유는 무엇인가?
- 4KB 일반 페이지 vs 2MB Huge Page가 TLB 적중률에 어떤 차이를 만드는가?
- 데이터베이스·JVM이 Huge Page를 적극 활용하는 이유는?
- `perf stat -e dTLB-load-misses,iTLB-load-misses`로 무엇을 진단할 수 있는가?

---

## 🔍 왜 이 개념이 중요한가

### "캐시 미스는 잡았는데 여전히 느린 코드 — TLB 미스가 숨어 있다"

```
보통 성능 분석의 기본 셋:
  perf stat → IPC, cache-misses, branch-misses
  → 여기서 끝나는 경우가 많다

하지만 다음과 같은 증상이 있다면 TLB가 범인:
  ┌─────────────────────────────────────────┐
  │ • 큰 데이터(수 GB)를 랜덤 접근           │
  │ • L3 캐시는 잘 들어가는데 여전히 느림    │
  │ • 워킹셋이 수십 MB 이상                  │
  │ • 대용량 hash table / B+Tree 탐색        │
  └─────────────────────────────────────────┘

이유:
  메모리 접근 = "가상 주소 → 물리 주소 변환" + "물리 메모리 읽기"
  변환에도 캐시가 필요 → TLB
  TLB miss → 페이지 테이블을 직접 워크 → 추가 4번의 메모리 접근

TLB 미스 한 번의 비용:
  최선: 페이지 테이블 항목들이 L1/L2 캐시에 있음 → ~수십 사이클
  최악: 모든 항목이 DRAM → 4 × 200cy = ~800 cycles
  → "캐시 미스 한 번 = 명령 수십 개" 보다 훨씬 더 큰 손실
```

---

## 😱 잘못된 이해

### Before: "메모리 주소는 그냥 주소다"

```
잘못된 모델:
  포인터 p가 0x7ffe_1234_5678을 가리킨다
  → CPU가 그 주소의 DRAM 칸을 직접 읽는다
  → 끝.

실제:
  0x7ffe_1234_5678은 가상 주소(Virtual Address)
  → 프로세스마다 독립된 주소 공간
  → 실제 물리 메모리 어디에 매핑됐는지는 OS가 관리

  CPU가 메모리에 접근할 때마다:
    ① 가상 주소를 물리 주소로 변환 (MMU + TLB)
    ② 변환된 물리 주소로 캐시/메모리 접근

  ① 자체가 비용이다 — TLB hit이면 ~1cy, miss면 ~수십~수백cy

따라서:
  "캐시 친화적으로 짜자" → 부분적으로 맞음
  하지만 워킹셋이 수십 MB 이상이면
  "TLB 친화적" 까지 고려해야 진짜 빠른 코드가 된다
```

---

## ✨ 올바른 이해

### After: 모든 메모리 접근은 두 단계 — 주소 변환 + 데이터 페치

```
x86-64 메모리 접근의 전체 흐름:

가상 주소 (48bit 사용, 상위 16bit는 부호 확장):
  ┌──────┬──────┬──────┬──────┬──────────────┐
  │ PML4 │ PDPT │  PD  │  PT  │  Page Offset │
  │ 9bit │ 9bit │ 9bit │ 9bit │     12 bit   │
  └──────┴──────┴──────┴──────┴──────────────┘
  
  → 9bit × 4 = 36bit (페이지 테이블 인덱스)
  → 12bit = 4KB 페이지 내 오프셋
  → 총 48bit 가상 주소

변환 흐름 (TLB miss 시 page walk):

  ① CR3 레지스터 → PML4 테이블 시작 주소
  ② PML4[PML4 index] → PDPT 시작 주소        (메모리 접근 1)
  ③ PDPT[PDPT index] → PD 시작 주소           (메모리 접근 2)
  ④ PD[PD index]     → PT 시작 주소            (메모리 접근 3)
  ⑤ PT[PT index]     → 물리 페이지 시작 주소    (메모리 접근 4)
  ⑥ 물리 페이지 + Page Offset → 실제 데이터 (메모리 접근 5)

  → TLB miss 한 번 = 페이지 테이블 4번 + 실제 데이터 1번 = 최대 5번
  → 이중 페이지 테이블 항목들도 캐시에 들어가므로 보통 빠르게 끝남
  → 하지만 워킹셋이 크면 페이지 테이블 항목 자체가 캐시에서 빠짐

TLB는 이 변환 결과를 캐시:
  TLB hit  → 변환 ~1cy (캐시처럼 빠름)
  TLB miss → 위의 page walk 발생

TLB는 보통 두 단계:
  L1 dTLB:  ~64 entries  (데이터)
  L1 iTLB:  ~128 entries (명령어)
  L2 TLB:   ~1500 entries (공유)

  → 4KB 페이지로는 L1 dTLB가 커버 가능한 영역 = 64 × 4KB = 256KB
  → 워킹셋이 256KB 넘으면 TLB 미스 가능성 급증
```

---

## 🔬 내부 동작 원리

### 1. x86-64 4단계 페이지 테이블 구조

```
CR3 레지스터 (각 프로세스마다 다른 값, context switch 시 변경)
  │
  ▼
┌───────────────────────────────────────┐
│ PML4 Table (4KB = 512 entries × 8B)   │
│ entry[0] → PDPT 시작 주소              │
│ entry[1] → PDPT 시작 주소              │
│ ...                                    │
└───────────────────┬───────────────────┘
                    │ 가상주소 47:39 (9bit)로 인덱싱
                    ▼
┌───────────────────────────────────────┐
│ PDPT Table (4KB)                       │
│ entry[k] → PD 시작 주소                 │
└───────────────────┬───────────────────┘
                    │ 가상주소 38:30 (9bit)
                    ▼
┌───────────────────────────────────────┐
│ PD Table (4KB)                          │
│ entry[k] → PT 시작 주소                  │
│ 또는 entry[k] = 2MB Huge Page 직접 매핑 │ ◀── Huge Page는 여기서 종료!
└───────────────────┬───────────────────┘
                    │ 가상주소 29:21 (9bit)
                    ▼
┌───────────────────────────────────────┐
│ PT Table (4KB)                          │
│ entry[k] → 물리 페이지 시작 주소         │
└───────────────────┬───────────────────┘
                    │ 가상주소 20:12 (9bit) + 11:0 (offset)
                    ▼
                물리 데이터

각 entry는 64bit:
  ┌───────────────┬─────────┬───┬───┬───┬───┬───┐
  │ Physical Addr │ Reserved│ A │ D │ U │ W │ P │
  │   bits 51:12  │   ...   │   │   │   │   │   │
  └───────────────┴─────────┴───┴───┴───┴───┴───┘
  P = Present, W = Writable, U = User-mode, D = Dirty, A = Accessed
```

### 2. TLB 미스 시 페이지 워크 비용 계산

```
TLB hit:
  변환 ~1 cycle (TLB는 CPU 내부 캐시)

TLB miss 비용 (모든 페이지 테이블 항목이 캐시에서):
  Best case (페이지 테이블 항목들이 L1 캐시에):
    4 × 4cy(L1) = ~16cy
  
  Worst case (페이지 테이블 항목들이 DRAM에):
    4 × 200cy(DRAM) = ~800cy
  
  Typical (혼합):
    PML4, PDPT 자주 쓰임 → L1 캐시
    PD, PT는 워크로드에 따라 → L2/L3 캐시
    → 평균 ~50~100cy

비교 — 일반 L1 캐시 미스:
  L1 miss → L2 hit: ~12cy
  L1 miss → L3 hit: ~40cy
  L1 miss → DRAM:   ~200cy

  TLB miss는 캐시 미스보다 4배 비싸다 (페이지 워크 4단계)
```

### 3. 4KB vs 2MB Huge Page 비교

```
같은 1GB 데이터를 다룬다고 했을 때:

4KB 페이지:
  1GB / 4KB = 262,144 개의 페이지 = 262,144 개의 TLB 항목 필요
  L1 dTLB(64 entries)로는 256KB만 커버 → 압도적 미스율
  L2 TLB(1500 entries) 포함해도 6MB만 커버
  → 1GB 워킹셋 → TLB 미스 폭증

2MB Huge Page:
  1GB / 2MB = 512 개의 페이지 = 512 개의 TLB 항목 필요
  L2 TLB 1500개로 3GB 커버 가능
  → 1GB 워킹셋 → TLB 미스 거의 없음

추가 이점:
  Huge Page는 PD에서 매핑 종료 → 페이지 워크가 3단계로 줄어듦
  → TLB miss 시에도 비용이 25% 절감

단점:
  메모리 단편화 — 2MB 단위로만 할당
  fork 시 COW(Copy-on-Write)의 단위가 2MB → 더 비싼 COW
  Anonymous Huge Pages(THP)가 의도치 않게 동작하면 latency spike

활용:
  데이터베이스 (InnoDB Buffer Pool, PostgreSQL shared_buffers)
  JVM (-XX:+UseLargePages, -XX:LargePageSizeInBytes=2m)
  Redis (대용량 인메모리 캐시)
  → 큰 워킹셋을 일관되게 접근하는 워크로드
```

### 4. TLB와 context switch

```
프로세스 전환 시:
  CR3 값이 새 프로세스의 PML4로 교체
  → TLB 전체 무효화 (예전엔 무조건)
  → 새 프로세스는 콜드 TLB로 시작

PCID (Process Context ID) 도입 후:
  각 TLB 항목에 12bit PCID 태그
  → 프로세스 전환 시 TLB 보존
  → 다시 돌아오면 warm TLB 사용 가능
  Linux: `nopcid` 부팅 파라미터로 비활성화 가능 (디버깅용)

Meltdown 완화 이후 (KPTI: Kernel Page Table Isolation):
  커널 진입/탈출 시 CR3 교체 발생
  → PCID 없으면 syscall마다 TLB flush
  → syscall 비용 폭증
  → Spectre/Meltdown 패치 이후 syscall 비용 증가의 주요 원인
```

### 5. iTLB vs dTLB

```
iTLB (Instruction TLB):
  명령어 페치용 주소 변환
  주로 코드 크기에 비례
  큰 바이너리 (점프 테이블이 많은 인터프리터, 거대 JIT 코드) → iTLB 미스

dTLB (Data TLB):
  데이터 로드/스토어용 주소 변환
  워킹셋 크기 + 접근 패턴에 따라 결정
  큰 hash table 랜덤 접근, 큰 행렬 비순차 접근 → dTLB 미스

진단:
  perf stat -e dTLB-load-misses,dTLB-store-misses ./a.out
  perf stat -e iTLB-load-misses ./a.out
  
  미스율 (dTLB-load-misses / dTLB-loads) > 0.5% 면 의심
  > 1% 면 거의 확실히 TLB가 병목
```

---

## 💻 실전 실험

### 실험 1: 워킹셋 크기에 따른 dTLB 미스율 변화

```c
// tlb_demo.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char** argv) {
    size_t mb = atoi(argv[1]);
    size_t n  = mb * 1024 * 1024 / sizeof(int);
    int* arr = malloc(n * sizeof(int));
    memset(arr, 0, n * sizeof(int));
    
    // 랜덤 접근 — 캐시도 TLB도 망가짐
    long sum = 0;
    for (int iter = 0; iter < 100; iter++) {
        for (size_t i = 0; i < n; i += 1024) {  // 4KB stride
            sum += arr[i];
        }
    }
    
    printf("sum=%ld\n", sum);
    free(arr);
}
```

```bash
gcc -O2 tlb_demo.c -o tlb_demo

# 워킹셋 1MB (L1 dTLB 커버 범위 내)
perf stat -e dTLB-loads,dTLB-load-misses ./tlb_demo 1

# 워킹셋 16MB (L2 TLB 커버 범위 초과)
perf stat -e dTLB-loads,dTLB-load-misses ./tlb_demo 16

# 워킹셋 1024MB
perf stat -e dTLB-loads,dTLB-load-misses ./tlb_demo 1024

# 예상 출력 (대략):
# 1MB    → dTLB-load-misses: 0.01%
# 16MB   → dTLB-load-misses: 0.5%
# 1024MB → dTLB-load-misses: 5~15%
```

### 실험 2: Huge Page 효과 측정

```bash
# Huge Page 활성화 (Linux)
echo 1024 | sudo tee /proc/sys/vm/nr_hugepages   # 1024 × 2MB = 2GB

# 프로그램에서 madvise로 THP 요청
# (코드 수정: malloc 후 madvise(p, size, MADV_HUGEPAGE);)

# 또는 명시적 hugetlbfs 마운트
sudo mkdir /mnt/huge
sudo mount -t hugetlbfs nodev /mnt/huge

# 측정
perf stat -e dTLB-load-misses,iTLB-load-misses ./tlb_demo 1024
# Huge Page 적용 후 dTLB-load-misses가 10배 이상 감소하는 것이 정상

# JVM 활용
java -XX:+UseLargePages -XX:LargePageSizeInBytes=2m -Xms4g -Xmx4g MyApp
```

### 실험 3: 페이지 워크 사이클을 perf로 직접 측정

```bash
# 페이지 워크에 소비된 사이클 직접 측정 (Intel)
perf stat -e \
  dtlb_load_misses.walk_active,\
  dtlb_load_misses.walk_completed,\
  dtlb_load_misses.walk_pending \
  ./tlb_demo 1024

# 출력 해석:
#   walk_active   : 페이지 워크 중인 사이클 수
#   walk_completed: 완료된 페이지 워크 횟수
#   → 평균 워크 비용 = walk_active / walk_completed

# 예: 1억 사이클 / 100만 워크 = 평균 100cy per walk
# → TLB 미스 1번에 100사이클 소비 → 캐시 미스보다 더 큰 손실
```

### 실험 4: random vs sequential 접근의 TLB 영향

```c
// 같은 크기, 같은 데이터, 접근 패턴만 다름
void seq(int* arr, size_t n) {
    long s = 0;
    for (size_t i = 0; i < n; i++) s += arr[i];
    /* prefetcher가 다음 페이지를 미리 가져옴 → TLB miss 거의 없음 */
}

void rnd(int* arr, size_t n, int* idx) {
    long s = 0;
    for (size_t i = 0; i < n; i++) s += arr[idx[i]];
    /* 매번 다른 페이지 접근 → TLB miss 폭증 */
}
```

```bash
# perf로 비교 — 같은 메모리 양인데 TLB 미스 차이 극명
perf stat -e dTLB-load-misses,cache-misses ./seq_vs_rnd seq
perf stat -e dTLB-load-misses,cache-misses ./seq_vs_rnd rnd
```

---

## 📊 성능 비교

```
1GB 데이터 랜덤 접근 — 4KB 페이지 vs 2MB Huge Page:

                          4KB 페이지     2MB Huge Page
─────────────────────────────────────────────────────
필요 TLB 항목             262,144        512
L2 TLB 커버 가능 영역      6MB            3GB
dTLB-load-miss 비율        ~8%            ~0.05%
페이지 워크 단계            4              3
평균 워크 비용              ~80cy          ~30cy
실측 실행 시간             1.4초           0.6초 (~2.3배)

특히 큰 워크로드:
  PostgreSQL 100GB DB 스캔:
    4KB:  TLB 미스로 인한 CPU 25% 사용
    2MB:  TLB 미스 거의 0% → 같은 시간에 25% 더 많은 작업
  
  JVM heap 16GB:
    4KB:  GC pause 동안 TLB 미스가 STW를 늘림
    2MB:  GC 사이클이 ~10% 빠름 (사례별)
```

---

## ⚖️ 트레이드오프

```
4KB 페이지 (기본):
  ✅ 작은 단위 → 메모리 단편화 최소
  ✅ fork 시 COW 단위가 작아 효율적
  ❌ 큰 워킹셋에서 TLB 미스 폭증
  ❌ 페이지 테이블 자체가 메모리 차지 (1GB 매핑 = 2MB 페이지 테이블)

2MB Huge Page:
  ✅ TLB 적중률 대폭 상승 (수십~수백 배)
  ✅ 페이지 워크 1단계 감소 (PD에서 종료)
  ✅ 페이지 테이블 크기 1/512
  ❌ 메모리 단편화 — 2MB 미만은 통째로 점유
  ❌ THP가 의도치 않게 동작하면 page fault 시 long pause
  ❌ NUMA 환경에서 한 노드에 큰 페이지가 몰리면 부담

1GB Huge Page (Gigantic):
  ✅ 거대 DB / HPC 워크로드에서 TLB 거의 무력화
  ❌ 부팅 시점에 예약 필요 → 동적 할당 불가
  ❌ 단편화 위험 극대

THP (Transparent Huge Pages):
  ✅ 애플리케이션 수정 없이 자동 활용
  ❌ khugepaged 백그라운드 스레드의 latency 영향
  ❌ 데이터베이스 종류에 따라 비활성화 권장 (MongoDB, Redis)
  Linux 권장 설정:
    /sys/kernel/mm/transparent_hugepage/enabled = madvise (기본)
    명시적 madvise(MADV_HUGEPAGE)한 영역만 huge page 사용
```

---

## 📌 핵심 정리

```
TLB와 가상 메모리 핵심:

모든 메모리 접근은 두 단계:
  ① 가상→물리 주소 변환 (MMU + TLB)
  ② 변환된 주소로 캐시/메모리 접근
  → ①이 캐시 미스 못지않은 비용

x86-64 페이지 테이블:
  4단계 (PML4→PDPT→PD→PT) × 9bit
  4KB 페이지 + 12bit 오프셋
  CR3 레지스터에서 시작

TLB:
  주소 변환 결과 캐시
  L1 dTLB ~64, L1 iTLB ~128, L2 ~1500 entries
  miss 시 4단계 워크 → ~50~100cy 평균, 최악 800cy

워킹셋과 TLB 커버 범위:
  4KB × 1500 = 6MB까지만 L2 TLB 커버
  넘어가면 TLB 미스 폭증
  → 큰 데이터 = TLB 친화성도 고려

Huge Page:
  2MB 페이지 → 같은 메모리를 1/512 항목으로 매핑
  TLB 적중률 수십~수백 배
  데이터베이스·JVM·Redis 권장
  -XX:+UseLargePages, vm.nr_hugepages, MADV_HUGEPAGE

진단:
  perf stat -e dTLB-load-misses,iTLB-load-misses
  미스율 0.5% 이상이면 의심, 1% 이상이면 TLB 병목

다른 챕터 연결:
  Ch2-03 지역성 → 페이지 단위 지역성도 중요
  Ch2-05 데이터 레이아웃 → hot/cold 분리는 TLB에도 효과
  Ch7-01 perf 활용 → dTLB-load-misses는 표준 진단 지표
  linux-for-backend-deep-dive → THP 설정, page fault 처리
```

---

## 🤔 생각해볼 문제

**Q1.** 1GB 데이터를 순차 접근하는 코드와 랜덤 접근하는 코드의 캐시 미스율이 비슷한데도 랜덤 접근이 훨씬 느리다. 무엇 때문일 가능성이 높은가?

<details>
<summary>해설 보기</summary>

가장 유력한 원인은 **TLB 미스**입니다.

순차 접근은 다음 두 가지 이점이 있습니다:
- 같은 페이지 안에서 여러 데이터를 읽으므로 한 페이지에 대한 TLB 항목으로 1024개의 int(4KB/4B)를 모두 처리
- 다음 페이지로 넘어갈 때 hardware prefetcher가 다음 페이지의 TLB 항목까지 미리 가져올 수 있음

랜덤 접근은:
- 매번 다른 페이지에 접근 → TLB 항목이 매번 다름 → L1/L2 TLB 용량 초과 → 페이지 워크 필요
- 페이지 워크 한 번 = 4번의 추가 메모리 접근 = ~50~100cy

캐시 미스율이 비슷해 보여도 perf로 `dTLB-load-misses`를 확인하면 차이가 10~100배 나는 경우가 많습니다.

해결: `madvise(addr, len, MADV_HUGEPAGE)`로 Huge Page 활용 또는 `transparent_hugepage` 설정 조정.

</details>

---

**Q2.** JVM 운영팀이 `-XX:+UseLargePages`를 적용했더니 평균 응답 시간은 줄었지만 가끔 GC pause가 1초 넘게 튄다. 원인이 뭘까?

<details>
<summary>해설 보기</summary>

가장 흔한 원인은 **Huge Page 메모리 단편화 + THP의 백그라운드 압축(khugepaged)** 입니다:

1. **단편화**: 2MB 단위 페이지가 안정적으로 할당되려면 물리 메모리에 연속된 2MB 영역이 필요. 시스템이 오래 동작하면 단편화로 인해 새 2MB 페이지 확보가 어려워짐 → 페이지 회수(compaction) 트리거 → 그 동안 프로세스가 멈춤.

2. **khugepaged**: THP를 자동으로 4KB → 2MB로 승격하는 백그라운드 스레드. 승격 작업 중에 해당 메모리 영역의 접근을 잠시 막을 수 있어 latency spike 발생.

해결책:
- 부팅 시 `vm.nr_hugepages`로 명시적으로 예약(THP 대신 hugetlbfs 사용)
- `transparent_hugepage/enabled=madvise`로 자동 승격 끄기
- JVM에 `-XX:+UseTransparentHugePages` 대신 `-XX:+UseLargePages`로 명시적 huge page만 사용
- `/sys/kernel/mm/transparent_hugepage/defrag=never`로 백그라운드 압축 비활성화

데이터베이스·캐시 서버에서 자주 마주치는 함정으로, MongoDB 공식 문서가 THP 비활성화를 권장하는 이유입니다.

</details>

---

**Q3.** Linux의 KPTI(Kernel Page Table Isolation, Meltdown 완화) 패치 이후 syscall 비용이 30~50% 증가했다. TLB 관점에서 무엇이 일어났는가?

<details>
<summary>해설 보기</summary>

KPTI 이전:
- 한 프로세스 안에 user space와 kernel space의 페이지 테이블이 통합되어 있음
- syscall로 kernel 진입 시 CR3 변경 없음 → TLB 유지
- syscall 비용은 모드 전환과 콘텍스트 저장 정도

KPTI 이후:
- user space와 kernel space의 페이지 테이블 분리 (Meltdown 차단 위함)
- syscall마다 CR3 교체 발생 → 원칙적으로 TLB flush 발생
- 다시 user로 돌아올 때 또 CR3 교체 → 또 flush
- → syscall 1회당 TLB가 두 번 비워짐 → 직후 접근들이 모두 TLB miss

완화책 — PCID(Process Context ID):
- TLB 항목에 12bit PCID 태그 부여
- CR3 교체해도 해당 PCID의 항목은 유지
- 따라서 PCID 지원 CPU(Westmere 이후, 활용은 Haswell+)에서는 TLB flush 회피
- 그래도 일부 항목 무효화는 필요 → 여전히 ~30% 비용 증가

운영 임팩트:
- syscall 많은 워크로드(예: Nginx, Redis, DB)는 직접 영향
- 대처: io_uring 등 syscall 묶음 처리, syscall 호출 줄이기, vDSO 활용

이것이 Spectre/Meltdown 완화가 보안과 성능의 트레이드오프인 이유이고, 일부 폐쇄망 환경에서 `nopti` 부팅 파라미터로 비활성화하는 배경입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 데이터 구조와 캐시](./05-data-layout-cache.md)** | **[홈으로 🏠](../README.md)** | **[다음 챕터: 메모리 모델과 동시성 ➡️](../memory-model-concurrency/01-mesi-cache-coherence.md)**

</div>
