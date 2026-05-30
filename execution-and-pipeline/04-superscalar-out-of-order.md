# 슈퍼스칼라와 비순차 실행(Out-of-Order Execution)

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 슈퍼스칼라(Superscalar)가 IPC를 1 이상으로 만드는 메커니즘은 무엇인가?
- ROB(Reorder Buffer)와 Reservation Station은 각각 어떤 역할을 하는가?
- 명령어가 비순차 실행(Out-of-Order, OoO)되더라도 결과가 올바른 이유는 무엇인가?
- Tomasulo 알고리즘은 의존성 그래프를 어떻게 동적으로 풀어내는가?
- 레지스터 이름 변경(Register Renaming)이 WAR/WAW 해저드를 제거하는 원리는?
- `perf stat`에서 IPC가 4를 넘을 때 이것은 무엇을 의미하는가?

---

## 🔍 왜 이 개념이 중요한가

### "IPC 3.0을 보고 '이미 충분히 빠르다'고 생각하면 두 배를 놓친다"

```
Ch1-02에서 배운 것: 파이프라인이 꽉 차면 IPC = 1
Ch1-03에서 배운 것: 해저드가 파이프라인에 구멍을 만든다

그렇다면 IPC > 1은 어떻게 달성되는가?

답: 슈퍼스칼라 + 비순차 실행 (OoO)

  ┌──────────────────────────────────────────────┐
  │  perf stat -e cycles,instructions ./a.out    │
  │                                              │
  │  6,300,000,000  instructions  # 3.00 IPC    │
  │                                              │
  │  → 매 사이클마다 3개 명령어가 완료된다         │
  │  → 이것은 어떻게 가능한가?                    │
  └──────────────────────────────────────────────┘

이해하지 못하면:
  "코드를 더 최적화해봤자 이미 빠르다"고 착각
  의존성 체인이 IPC를 2.0에서 0.5로 낮추는 사태를 놓침
  누산기 분할(accumulator splitting)의 필요성을 모름

이해하면:
  "이 루프의 의존성 체인이 OoO 엔진을 직렬화한다"를 어셈블리로 읽을 수 있음
  godbolt에서 생성된 코드가 IPC를 몇으로 만들지 예측 가능
  Ch1-06 ILP 한계로 자연스럽게 이어짐
```

---

## 😱 잘못된 이해

### Before: "CPU는 명령어를 작성한 순서 그대로 실행한다"

```
잘못된 모델:
  소스 코드의 순서 = CPU 실행 순서

  int a = arr[0];    // 1번: 로드 (200 사이클 DRAM 대기 중...)
  int b = arr[1];    // 2번: 1번 완료 후 로드
  int c = b + 1;     // 3번: 2번 완료 후 계산
  int d = arr[2];    // 4번: 3번 완료 후 로드

  → 모두 순서대로 실행 → 1번의 DRAM 대기가 전체를 막음

현실 — OoO CPU는 이렇게 처리한다:
  int a = arr[0];    // 1번: 로드 시작 (대기 중...)
  int b = arr[1];    // 2번: 1번과 독립! → 즉시 실행
  int d = arr[2];    // 4번: 2번, 3번과 독립! → 즉시 실행
  int c = b + 1;     // 3번: 2번 완료 후 실행
  (1번은 백그라운드에서 계속 로드 중)

  → 세 로드가 동시에 진행
  → DRAM 레이턴시가 겹쳐서 숨겨짐
  → 전체 소요시간 ≈ 로드 1개 시간 (200 사이클이 아니라!)

"순서대로"는 프로그래머가 보는 의미적 순서일 뿐,
실행 순서는 CPU가 의존성 그래프를 보고 결정한다
```

---

## ✨ 올바른 이해

### After: OoO CPU는 의존성 그래프 기반 스케줄러다

```
핵심 개념:

  프로그래머가 작성한 순서 = 프로그램 순서 (Program Order, PO)
  CPU가 실제 실행하는 순서 = 실행 순서 (Execution Order, EO)
  커밋되는 순서 = 항상 프로그램 순서 (Commit Order = PO)

  PO ≠ EO, 하지만 결과는 PO를 따른 것과 동일

OoO CPU의 4단계 핵심 구조:

  ┌─────────────────────────────────────────────────────┐
  │  1. Rename(이름 변경): WAR/WAW 해저드 제거            │
  │     아키텍처 레지스터(rax,rbx,...) →                  │
  │     물리 레지스터(p0,p1,p2,...,p168개)로 매핑         │
  ├─────────────────────────────────────────────────────┤
  │  2. Dispatch(발행): Reservation Station(RS)에 큐잉   │
  │     RS = 실행 대기열 (Intel Skylake: 97 entry)       │
  │     모든 소스 피연산자가 준비되면 즉시 실행 유닛 발송  │
  ├─────────────────────────────────────────────────────┤
  │  3. Execute(실행): 여러 실행 유닛이 동시에 처리        │
  │     ALU × 4, Load × 2, Store × 1, FPU × 2 등       │
  │     준비된 명령어를 순서와 무관하게 실행               │
  ├─────────────────────────────────────────────────────┤
  │  4. Commit(커밋): ROB에서 프로그램 순서대로 결과 확정  │
  │     ROB = 실행 완료 후 순서 보장을 위한 버퍼           │
  │     (Intel Skylake: 224 entry)                      │
  └─────────────────────────────────────────────────────┘
```

---

## 🔬 내부 동작 원리

### 1. 슈퍼스칼라(Superscalar): 넓은 발행 폭

```
스칼라 파이프라인 (Scalar):
  매 사이클 최대 1개 명령어 발행(issue)
  이론적 IPC 상한 = 1.0

슈퍼스칼라 (Superscalar, Wide Issue):
  매 사이클 최대 N개 명령어 발행
  현대 Intel: 6-wide issue (한 사이클에 6개 µop 발행)
  현대 AMD Zen4: 6-wide issue

  사이클:   1      2      3      4      5
  명령1:  [EXEC]
  명령2:  [EXEC]   ← 같은 사이클에 동시 실행
  명령3:  [EXEC]   ← (서로 독립적이라면)
  명령4:           [EXEC]
  명령5:           [EXEC]
  명령6:           [EXEC]

  → 이론적 IPC = 발행 폭 (6)
  → 실제: 의존성·자원 충돌로 IPC 3~5 수준

현대 CPU의 실행 포트 (Intel Skylake 기준):
  Port 0: ALU, 분기, SIMD 정수, FP 곱셈, AES
  Port 1: ALU, 정수 곱셈, SIMD 정수, FP 덧셈
  Port 2: 로드, 스토어 주소
  Port 3: 로드, 스토어 주소
  Port 4: 스토어 데이터
  Port 5: ALU, 셔플, 분기
  Port 6: ALU, 분기
  Port 7: 스토어 주소
  → 8개 포트가 각자 독립적으로 실행 가능
  → 같은 사이클에 8개 연산이 동시 진행 가능 (자원 허용 시)
```

### 2. Reservation Station (RS): 의존성 감시 대기열

```
Reservation Station의 역할:
  발행된 µop을 보관
  각 µop의 소스 피연산자가 "준비됨" 상태인지 감시
  준비된 µop을 적절한 실행 유닛으로 즉시 발송

RS 엔트리 구조:
  ┌──────────────────────────────────────────────────────┐
  │  µop 종류   │ 목적 레지스터 │ 소스1(값/태그) │ 소스2 │
  ├──────────────────────────────────────────────────────┤
  │  ADD        │  p42          │  p37(준비됨)  │  p41  │
  │             │               │               │  (대기 중)│
  └──────────────────────────────────────────────────────┘
  → p41이 준비되는 순간 즉시 실행 유닛으로 발송

예시: 의존성 그래프 처리

  코드:
    add  rax, rbx   ; ①  rax = rax + rbx
    add  rcx, rdx   ; ②  rcx = rcx + rdx  (①과 독립)
    add  rsi, rax   ; ③  rsi = rsi + rax  (①에 의존)
    add  rdi, rcx   ; ④  rdi = rdi + rcx  (②에 의존)

  의존성 그래프:
    ①  ②
    ↓  ↓
    ③  ④

  RS 처리:
    사이클 1: ①, ②를 RS에 등록 → 소스 모두 준비됨 → 즉시 실행 발송
    사이클 2: ①, ② 결과 생성 → ③, ④의 소스 준비됨 신호
    사이클 3: ③, ④ 동시 실행 (이때 ①, ②는 이미 완료됨)

  → 비순차 실행! (①,②가 먼저 실행된 후 ③,④)
  → 하지만 프로그래머가 의도한 것과 동일한 결과
```

### 3. ROB (Reorder Buffer): 순차 커밋 보장

```
ROB의 역할:
  비순차로 완료된 명령어들을 프로그램 순서대로 커밋(commit)
  예외(exception)와 인터럽트의 정확한 처리
  투기적 실행(speculative execution)의 취소 지원

ROB 동작:
  프로그램 순서     실행 완료 순서    커밋 순서
     명령1   ──→    명령3 (먼저 완료)  명령1 (대기)
     명령2   ──→    명령1 (그 다음)    명령2 (대기)
     명령3   ──→    명령2 (마지막)     명령3 (1번이 먼저 완료됐으나
                                             1번 커밋 후 2번 커밋
                                             그 후 3번 커밋)

  → 비순차 실행이지만 커밋은 프로그램 순서 유지!

ROB와 예외 처리:
  명령 5번이 실행 중 예외 발생 (예: 페이지 폴트)
  명령 6, 7, 8번이 이미 비순차로 완료됨
  → 6, 7, 8번의 결과를 ROB에서 폐기
  → 프로그램 상태는 명령 5번 직전까지만 반영됨
  → 정확한 예외(Precise Exception) 보장

ROB 크기의 의미:
  ROB = 240 entries (Intel Golden Cove 기준)
  → CPU가 최대 240개 명령어를 동시에 "비행 중(in-flight)"으로 유지
  → 긴 메모리 레이턴시(200 사이클)를 숨기려면 ROB가 200개 이상 있어야 함
  → ROB가 작으면 DRAM 대기 중에 할 일이 없어 스톨
```

### 4. Tomasulo 알고리즘: 동적 스케줄링의 원형

```
Tomasulo 알고리즘 (1967년 IBM 360/91에서 처음 구현):
  하드웨어가 런타임에 의존성을 분석하여 독립적인 명령어를 먼저 실행

핵심 아이디어:
  1. 태그(Tag) 기반 의존성 추적
     각 물리 레지스터에 "어느 명령어가 결과를 생산하는가" 태그 부여
     소비자 명령어는 "생산자의 태그가 완료되기를 기다린다"

  2. 공통 데이터 버스(CDB, Common Data Bus)
     실행 완료 시 결과를 CDB에 브로드캐스트
     RS에서 이 태그를 기다리던 모든 명령어에 즉시 전달

  3. 준비되면 즉시 실행
     소스 피연산자가 모두 준비된 순간 실행 유닛으로 발송
     프로그램 순서와 무관

현대 CPU의 Tomasulo 구현:
  RS + ROB + Register File의 조합이 Tomasulo의 현대적 구현
  "Unified Reservation Station" (Intel P6 이후)

  고전 Tomasulo:
    Reservation Station이 연산 유닛 별로 분산
    공통 데이터 버스가 브로드캐스트

  현대 Intel:
    중앙집중식 RS (Unified RS)
    실행 포트별 스케줄러가 준비된 µop을 적절한 포트로 발송
    Forward network가 결과를 RS 전체에 전파
```

### 5. 레지스터 이름 변경(Register Renaming): WAR/WAW 제거

```
문제: WAR(Write-After-Read)과 WAW(Write-After-Write)

  WAR 예시 (이름 의존성):
    add rax, rbx   ; ① rax 읽기 및 쓰기
    mov rax, [rdi] ; ② rax 쓰기 ← ①보다 먼저 실행하면?
                   ;   ①이 rax를 읽기 전에 ②가 rax를 덮어쓰면 오류!
    add rcx, rax   ; ③ rax 읽기 (①의 결과 필요)

  WAW 예시 (출력 의존성):
    mov rax, 1     ; ① rax = 1
    mov rax, 2     ; ② rax = 2 ← 둘 다 rax를 씀
                   ;   ②가 먼저 커밋되면 최종값이 1이 되어버림!

레지스터 이름 변경의 해결:
  아키텍처 레지스터 (ISA 레지스터):
    프로그래머/컴파일러가 사용하는 이름: rax, rbx, rcx, ...
    x86-64: 16개 범용 레지스터

  물리 레지스터 (Physical Registers):
    CPU 내부의 실제 레지스터
    Intel Skylake: 물리 정수 레지스터 180개
    AMD Zen4: 물리 정수 레지스터 224개

  Register Alias Table (RAT):
    아키텍처 레지스터 → 물리 레지스터 매핑 테이블
    매 명령어마다 갱신됨

예시: WAR 해저드가 이름 변경으로 사라지는 과정

  명령어       아키텍처 레지스터   물리 레지스터 할당
  add rax,rbx  rax 읽기+쓰기   →  p12 읽기+쓰기 → p50 생성
  mov rax,[rdi] rax 쓰기      →  p50 쓰기     → p51 생성 (새 물리 레지스터!)
  add rcx,rax   rax 읽기       →  p50 읽기     (①의 결과를 올바르게 읽음)

  → ②는 p51을 쓰고, ①과 ③은 p50을 사용
  → WAR 의존성이 사라짐! 두 명령어는 완전히 독립
  → OoO 엔진이 ①과 ②를 동시에 실행 가능

레지스터 이름 변경 후 의존성:
  진짜 의존성(RAW): 이름 변경으로 제거 불가 (실제 데이터 흐름)
  이름 의존성(WAR): 이름 변경으로 완전 제거
  출력 의존성(WAW): 이름 변경으로 완전 제거

  → OoO에서 성능을 제한하는 것은 RAW 의존성만 남는다
```

### 6. 전체 OoO 파이프라인 흐름

```
x86-64 명령어 → 전체 OoO 실행 경로:

Fetch (L1i 캐시에서 명령어 인출)
  ↓
Decode (x86 → µop 변환, 1~4 µop)
  ↓
Rename (RAT로 아키텍처 레지스터 → 물리 레지스터 매핑)
  ↓
Dispatch (ROB와 RS에 동시 등록)
  ├── ROB: 프로그램 순서로 슬롯 할당 (커밋 순서 보장용)
  └── RS: 실행 대기 (소스 피연산자 준비 감시)
       ↓
  [소스 피연산자 모두 준비됨]
       ↓
Schedule (RS 스케줄러가 적절한 실행 포트 선택)
  ↓
Execute (실행 유닛에서 연산, 비순차 실행 가능)
  ↓
Writeback (결과를 포워딩 네트워크로 브로드캐스트 → RS의 대기 µop에 전달)
  ↓
Commit (ROB에서 프로그램 순서대로 아키텍처 레지스터 파일 갱신)

핵심: Execute와 Commit이 분리됨
  Execute 순서 = 의존성 기반 OoO
  Commit 순서 = 항상 프로그램 순서 (in-order commit)
```

---

## 💻 실전 실험

### 실험 1: IPC > 1 측정 — 슈퍼스칼라 확인

```c
// superscalar_demo.c
#include <stdio.h>
#include <time.h>
#include <stdint.h>

#define N 1000000000LL

// 의존성 체인: 직렬화됨 → IPC 낮음
uint64_t dep_chain(void) {
    uint64_t x = 1;
    for (long i = 0; i < N; i++) {
        x = x * 6364136223846793005ULL + 1;
    }
    return x;
}

// 독립적인 연산: OoO가 병렬화 → IPC 높음
uint64_t independent(void) {
    uint64_t a = 1, b = 2, c = 3, d = 4;
    const uint64_t M = 6364136223846793005ULL;
    for (long i = 0; i < N / 4; i++) {
        a = a * M + 1;
        b = b * M + 2;  // a와 독립
        c = c * M + 3;  // a, b와 독립
        d = d * M + 4;  // a, b, c와 독립
    }
    return a ^ b ^ c ^ d;
}

int main(void) {
    struct timespec t1, t2;
    volatile uint64_t r;

    clock_gettime(CLOCK_MONOTONIC, &t1);
    r = dep_chain();
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms1 = ((t2.tv_sec-t1.tv_sec)*1e3 + (t2.tv_nsec-t1.tv_nsec)*1e-6);

    clock_gettime(CLOCK_MONOTONIC, &t1);
    r = independent();
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms2 = ((t2.tv_sec-t1.tv_sec)*1e3 + (t2.tv_nsec-t1.tv_nsec)*1e-6);

    printf("결과: %lu (최적화 방지)\n", r);
    printf("의존성 체인:    %.1f ms\n", ms1);
    printf("독립 연산(×4):  %.1f ms\n", ms2);
    printf("속도비: %.2fx\n", ms1 / ms2);
    return 0;
}
```

```bash
gcc -O2 -o superscalar_demo superscalar_demo.c

# IPC 측정
perf stat -e cycles,instructions \
          -e stalled-cycles-backend,stalled-cycles-frontend \
          ./superscalar_demo

# 예상 결과:
# 의존성 체인: IPC ≈ 0.33 (곱셈 레이턴시 3 사이클마다 1 명령어)
# 독립 연산:   IPC ≈ 1.3  (4개 곱셈을 OoO가 병렬화)
# 속도비: 약 3~4x

# 어셈블리 확인 (godbolt 또는)
gcc -O2 -S -masm=intel superscalar_demo.c -o superscalar_demo.s
grep -A 20 "dep_chain:" superscalar_demo.s
grep -A 30 "independent:" superscalar_demo.s
# independent: 4개 imulq가 연속 출현 → 4개 독립 곱셈 확인
```

### 실험 2: ROB 크기와 메모리 레이턴시 은닉

```c
// rob_hiding.c: OoO가 DRAM 레이턴시를 은닉하는 것을 측정
// 독립적인 로드가 여러 개일 때 vs 순차 로드 의존성 체인
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <stdint.h>

#define ARRAY_SIZE (1 << 26)  // 256MB - L3 캐시 초과

int main(void) {
    // L3를 초과하는 배열 (DRAM 강제)
    uint32_t *arr = (uint32_t*)malloc(ARRAY_SIZE * sizeof(uint32_t));
    if (!arr) { perror("malloc"); return 1; }

    // 배열 초기화 (랜덤 인덱스 체인 vs 독립 인덱스)
    // 테스트 1: 여러 독립적인 캐시 미스 (OoO가 동시에 처리)
    uint64_t sum = 0;
    struct timespec t1, t2;

    // 독립적인 스트라이드 접근 (8개 독립 스트림)
    clock_gettime(CLOCK_MONOTONIC, &t1);
    long stride = ARRAY_SIZE / 8;
    for (long i = 0; i < stride; i++) {
        sum += arr[i]
             + arr[i + stride]
             + arr[i + stride*2]
             + arr[i + stride*3]
             + arr[i + stride*4]
             + arr[i + stride*5]
             + arr[i + stride*6]
             + arr[i + stride*7];
    }
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_indep = ((t2.tv_sec-t1.tv_sec)*1e3+(t2.tv_nsec-t1.tv_nsec)*1e-6);

    // 단순 순차 접근 (프리페처가 작동 가능)
    sum = 0;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    for (long i = 0; i < ARRAY_SIZE; i++) {
        sum += arr[i];
    }
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_seq = ((t2.tv_sec-t1.tv_sec)*1e3+(t2.tv_nsec-t1.tv_nsec)*1e-6);

    printf("sum=%lu (최적화 방지)\n", sum);
    printf("독립 8스트림: %.1f ms (ROB가 8개 미스를 동시에 처리)\n", ms_indep);
    printf("순차 접근:    %.1f ms (프리페처 도움)\n", ms_seq);

    free(arr);
    return 0;
}
```

```bash
gcc -O2 -o rob_hiding rob_hiding.c

perf stat -e cycles,instructions,LLC-load-misses \
          -e stalled-cycles-backend ./rob_hiding

# ROB가 여러 미스를 동시에 처리하는 것을 확인
# stalled-cycles-backend: 독립 접근 시 더 낮아야 함 (미스가 겹쳐 처리됨)
```

### 실험 3: godbolt에서 Register Renaming 추적

```c
// renaming_demo.c: 이름 변경 효과를 godbolt에서 확인
// https://godbolt.org/ → C++ → x86-64 gcc -O2

// WAR 해저드가 있어 보이지만 이름 변경으로 제거되는 코드
long renaming_test(long a, long b, long c) {
    long t = a + b;    // rax = a + b (물리 레지스터 p50)
    a = c * 2;         // rax = c * 2 (물리 레지스터 p51, 새로 할당)
                       // WAR: t에서 rax를 읽고, a=c*2가 rax를 씀
                       // → 이름 변경으로 p50, p51 분리 → 완전 독립!
    return t + a;      // p50 + p51
}
```

```bash
# godbolt.org에서 확인:
# gcc x86-64, -O2, Intel syntax

# 생성 예상 어셈블리:
# lea    rax, [rdi+rsi]   ; t = a + b (즉시)
# lea    rdi, [rdx+rdx]   ; a = c * 2 (즉시, 위와 독립)
# add    rax, rdi          ; t + a
# ret

# → 두 lea가 순서와 무관하게 동시에 실행 가능
# → 이름 변경 덕분에 rax의 WAR 의존성이 사라짐

# 실제로 IPC 측정:
gcc -O2 -o renaming_demo renaming_demo.c
perf stat -e cycles,instructions ./renaming_demo
```

---

## 📊 성능 비교

```
슈퍼스칼라 OoO 효과 측정 (Intel Core i9 기준):

명령어 패턴               IPC 측정값    설명
────────────────────────────────────────────────────────────────
의존성 체인 (곱셈)          0.33       곱셈 3 사이클 레이턴시 직렬화
의존성 체인 (덧셈)          1.00       덧셈 1 사이클, 완전 직렬
독립 덧셈 ×4              3.5~4.0     4-wide 슈퍼스칼라 활용
독립 덧셈 ×8              5.0~6.0     여러 ALU 포트 포화
SIMD 벡터 연산            8~16+       1 명령어 = 여러 데이터

OoO 윈도우 크기 (in-flight 명령어):
  Intel Skylake:      ROB 224, RS 97, 물리 레지스터(정수) 180
  Intel Golden Cove:  ROB 512, RS 160, 물리 레지스터(정수) 280
  AMD Zen4:           ROB 320, RS 128, 물리 레지스터(정수) 224
  → ROB가 클수록 긴 메모리 레이턴시를 더 잘 숨길 수 있음

레지스터 이름 변경의 효과:
  WAR, WAW 의존성 완전 제거
  아키텍처 레지스터 16개 → 물리 레지스터 180~280개로 확장
  루프 언롤링(loop unrolling)이 더 효과적인 이유

실제 perf 측정 결과 해석:
  IPC < 0.5   → 긴 레이턴시 의존성 체인 (메모리 또는 나눗셈)
  IPC 1~2     → 의존성 있지만 OoO가 부분적으로 병렬화
  IPC 2~4     → OoO 잘 작동, 독립 명령어 충분
  IPC > 4     → SIMD 명령어 사용 중 (1 명령어 = 여러 요소 처리)
```

---

## ⚖️ 트레이드오프

```
OoO 엔진의 이점:
  ✅ 컴파일러가 완벽히 스케줄링 못 한 코드도 런타임에 최적화
  ✅ DRAM 레이턴시(200 사이클) 중에 다른 명령어 실행 → 레이턴시 은닉
  ✅ 함수 호출 경계를 넘어 최적화 (ROB 윈도우 내에서)
  ✅ 코드 이식성: 아키텍처 레지스터 16개로도 물리 레지스터 200+개 활용

OoO 엔진의 비용:
  ❌ 하드웨어 복잡도 폭증: RS, ROB, RAT, CDB, 물리 레지스터 파일
  ❌ 전력 소비: OoO 엔진은 CPU 전력의 상당 부분 차지
  ❌ 칩 면적: ARM Cortex-A55(in-order)가 A77(OoO)보다 60% 작음
  ❌ 긴 의존성 체인은 OoO도 해결 불가 (Ch1-06에서 상세)

ROB 크기의 트레이드오프:
  ROB 클수록:
    ✅ 더 긴 레이턴시를 은닉 (DRAM 200 사이클도 가리기 위해 200+ ROB 필요)
    ✅ 더 많은 명령어를 "비행 중" 유지 → 높은 IPC
    ❌ 전력/면적 비용 증가
    ❌ 체크포인트 관리 복잡 (분기 예측 실패 시 롤백)

임베디드 vs 서버 CPU의 철학:
  임베디드 (ARM Cortex-M, RISC-V):
    In-order 파이프라인 사용
    OoO 없음 → 훨씬 작고 저전력
    예측 가능한 실행 시간 (실시간 시스템에 유리)

  서버/데스크톱 (Intel Core, AMD Zen):
    OoO 필수 → 높은 IPC로 단일 스레드 성능 최대화
    메모리 접근 레이턴시 은닉이 핵심 목적

Spectre 취약점과의 연결:
  OoO + 투기적 실행이 Spectre의 근본 원인
  ROB가 롤백하기 전에 투기적 실행이 캐시 상태를 변경
  → Ch1-05 분기 예측에서 상세 설명

java-concurrency-deep-dive 연결:
  volatile, synchronized의 happens-before는
  OoO 커밋이 항상 프로그램 순서를 따른다는 사실 위에 구현됨
  Ch3-04 메모리 배리어에서 OoO commit과 배리어의 관계 설명
```

---

## 📌 핵심 정리

```
슈퍼스칼라와 OoO 핵심:

슈퍼스칼라:
  매 사이클 N개 명령어 발행 (현대 CPU: 6-wide)
  여러 실행 포트가 동시에 서로 다른 명령어 처리
  IPC > 1의 근본 원인

OoO 4대 구성요소:
  Rename   → WAR/WAW 해저드 제거, 아키텍처→물리 레지스터 매핑
  RS       → 소스 준비 감시, 준비된 µop 즉시 실행 발송
  Execute  → 준비된 순서대로 실행 (프로그램 순서 무관)
  ROB      → 비순차 완료를 프로그램 순서로 재정렬하여 커밋

핵심 불변성:
  실행 순서 ≠ 프로그램 순서
  커밋 순서 = 항상 프로그램 순서
  → 비순차 실행이지만 프로그래머가 의도한 결과와 동일

Tomasulo 알고리즘:
  태그 기반 의존성 추적 + CDB 브로드캐스트
  현대 OoO 엔진의 원형 (1967년에 이미 핵심 아이디어 존재)

레지스터 이름 변경:
  아키텍처 레지스터 16개 → 물리 레지스터 200개+
  WAR, WAW 의존성 완전 제거
  남은 장벽은 RAW(진짜 데이터 의존성)뿐

이 문서에서 다음으로 이어지는 것:
  분기 예측 실패 → ROB 전체 플러시 비용 → Ch1-05
  의존성 체인이 OoO도 막는 이유 → Ch1-06 ILP 한계
  메모리 배리어가 OoO 커밋과 어떻게 상호작용 → Ch3-04
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 OoO CPU가 실행 순서를 어떻게 바꿀 수 있는지 설명하고, 어떤 명령어 쌍이 병렬 실행 가능한지 의존성 그래프를 그려라.

```c
long foo(long *a, long *b, long n) {
    long x = a[0] + a[1];   // ①
    long y = b[0] * b[1];   // ②
    long z = x + y;         // ③
    long w = a[2] - b[2];   // ④
    return z * w;            // ⑤
}
```

<details>
<summary>해설 보기</summary>

**의존성 그래프 분석**:

```
①  ②  ④   ← 세 명령어 모두 서로 독립 (다른 주소 로드 + 다른 연산)
↓  ↓
③           ← ①, ②에 의존 (둘 다 완료 후 실행 가능)
    ↓
  ⑤         ← ③, ④에 의존
```

**OoO 실행 순서 (가능한 예)**:
- 사이클 1: ①의 로드, ②의 로드, ④의 로드 — **3개 동시 실행**
- 사이클 2~5: 로드 완료 대기 (L1 히트면 4 사이클)
- 사이클 5: ①의 덧셈(로드 완료), ②의 곱셈(로드 완료), ④의 뺄셈 — **동시 실행**
- 사이클 8: ③ 실행 (①의 덧셈 + ②의 곱셈 완료 필요)
- 사이클 9: ⑤ 실행 (③ + ④ 완료 필요)

**핵심**: 프로그램 순서대로 실행하면 5개 연산이 직렬화되지만, OoO는 ①②④를 동시에 실행하므로 2~3배 빠르다. 특히 ④는 ③보다 늦게 작성됐어도 ③보다 먼저 실행될 수 있다.

**레지스터 이름 변경 포인트**: 만약 컴파일러가 같은 레지스터를 재사용해 WAR를 만들어도, 이름 변경으로 독립성이 복원된다.

</details>

---

**Q2.** ROB 크기가 200 entries인 CPU와 500 entries인 CPU가 있다. DRAM 레이턴시가 200 사이클인 환경에서 각각의 성능 차이를 설명하라. 어떤 종류의 워크로드에서 차이가 가장 클 것인가?

<details>
<summary>해설 보기</summary>

**ROB 크기와 메모리 레이턴시 은닉**:

DRAM 레이턴시를 숨기려면, ROB가 200 사이클 동안 "의미 있는 다른 명령어"를 계속 채울 수 있어야 한다.

- **ROB 200인 CPU**: 200 사이클 레이턴시의 로드 명령어가 ROB에 들어오면, 그 뒤로 최대 199개의 독립 명령어만 처리 가능. 독립 명령어가 소진되면 ROB가 꽉 차서 더 이상 새 명령어를 받지 못하고 프론트엔드가 스톨.

- **ROB 500인 CPU**: 같은 200 사이클 동안 499개의 독립 명령어를 처리할 여유가 있어 더 오래 유용한 작업을 계속할 수 있다.

**차이가 가장 큰 워크로드**:
1. **불규칙한 메모리 접근 패턴** (포인터 추적, 해시 테이블, 그래프 탐색): 프리페처가 효과 없어 DRAM 레이턴시 그대로 노출. 독립 명령어가 충분히 많으면 ROB 500이 200보다 유리.
2. **데이터베이스 워크로드**: 넓은 ROB가 여러 독립적인 행 처리를 겹쳐 처리.

**차이가 작은 워크로드**:
- 순차 배열 접근: 프리페처가 작동해 DRAM 레이턴시가 거의 노출되지 않음 → ROB 크기 중요하지 않음.
- 의존성 체인: ROB 크기와 무관하게 다음 명령어를 실행할 수 없음.

</details>

---

**Q3.** Spectre 취약점이 OoO 실행의 어떤 특성을 악용하는지 설명하라. ROB의 커밋이 항상 프로그램 순서를 따름에도 불구하고 왜 정보 누출이 발생하는가?

<details>
<summary>해설 보기</summary>

**Spectre의 핵심 메커니즘**:

OoO CPU의 두 특성이 결합됩니다:
1. **투기적 실행**: 분기 예측기가 분기 결과를 예측하고, 예측 방향의 명령어를 ROB에 넣어 실행
2. **캐시 사이드 이펙트**: 투기적으로 실행된 로드 명령어가 캐시를 채움

```c
// Spectre 패턴 (단순화)
if (index < array_size) {          // 분기 예측기가 "true"로 예측
    char secret = secret_data[index]; // 투기적 실행: 비밀 데이터 로드
    temp = probe_array[secret * 64];  // 비밀 값으로 캐시 인덱스 접근
}
// 분기 예측 실패 → ROB에서 투기적 명령어 폐기
// 하지만 캐시는 이미 변경됨!
```

**정보 누출 경로**:
1. 분기 예측기가 `index < array_size`를 "true"로 예측
2. ROB에 투기적 명령어 등록, `secret_data[index]` 로드 실행
3. `probe_array[secret * 64]` 접근 → 캐시라인 채워짐
4. 실제 분기 결과: `false` → ROB에서 투기적 명령어 폐기
5. **하지만 캐시 상태는 폐기되지 않음!**
6. 공격자: `probe_array[0..255 * 64]` 접근 시간 측정 → 어느 인덱스가 캐시에 있는지 확인 → 비밀값 역추적

**ROB 커밋이 순서대로라도 정보가 새는 이유**:
- ROB 롤백은 레지스터와 메모리 쓰기를 되돌리지만
- 캐시 상태(어떤 캐시라인이 채워졌는가)는 아키텍처 상태가 아닌 **마이크로아키텍처 상태**
- 마이크로아키텍처 상태는 롤백 시 복원되지 않음
- 이것이 아키텍처-마이크로아키텍처 경계를 통한 정보 누출

이것이 Ch1-05에서 투기 실행과 Spectre의 연결로 이어집니다.

</details>

---

<div align="center">

**[⬅️ 이전: 파이프라인 해저드](./03-pipeline-hazards.md)** | **[홈으로 🏠](../README.md)** | **[다음: 분기 예측 ➡️](./05-branch-prediction.md)**

</div>
