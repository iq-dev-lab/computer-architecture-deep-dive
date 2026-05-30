# 함수 호출 비용 — 스택 프레임과 호출 규약

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- System V AMD64 호출 규약에서 인수는 어떤 레지스터 순서로 전달되는가?
- `call` 명령 한 번이 실제로 몇 사이클을 소비하는가?
- 스택 프레임 생성(prologue)과 해제(epilogue)는 어떤 명령어 시퀀스로 구성되는가?
- `__attribute__((always_inline))`과 LTO(Link Time Optimization)는 호출 비용을 어떻게 제거하는가?
- 인라이닝이 항상 유리한가? 코드 크기와 i-cache 미스의 트레이드오프는 무엇인가?
- JVM의 JIT가 인라이닝 임계값을 결정하는 방식은 C++ 컴파일러와 어떻게 다른가?

---

## 🔍 왜 이 개념이 중요한가

### "함수 호출은 그냥 점프 아닌가요?"

```
흔한 착각:
  foo() 호출 = JMP 명령 하나
  → "비용이 없다"

실제로 일어나는 일:
  ① 인수를 레지스터/스택에 배치          (1~N 사이클)
  ② RIP(현재 명령 포인터)를 스택에 push  (1 사이클 + 메모리 쓰기)
  ③ 목적지로 JMP                        (파이프라인 플러시 가능)
  ④ 스택 프레임 생성 (push rbp / mov rbp, rsp / sub rsp, N)
  ⑤ 로컬 변수를 레지스터에 재배치
  ⑥ 함수 본체 실행
  ⑦ 스택 프레임 해제 (leave 또는 pop rbp)
  ⑧ ret — 스택에서 복귀 주소 pop → JMP

작은 함수(본체 3줄)의 경우:
  호출 오버헤드 ≈ 본체 비용의 3~5배
  → 본체보다 "전화 거는 비용"이 더 비싼 상황

실제 사이클:
  call + ret + 프레임 생성/해제: ~5~15 사이클
  비교: L1 캐시 히트 로드: ~4 사이클
  → 함수 호출 한 번 = 캐시 미스 없이도 L1 로드 수 번에 해당하는 비용
```

핫 루프 안에서 수십만 번 호출되는 작은 헬퍼 함수를 인라이닝하면 수십~수백%의 성능 향상을 볼 수 있습니다. 이 원리를 이해해야 컴파일러가 `-O2`에서 자동으로 인라이닝을 결정하는 이유를 납득할 수 있습니다.

---

## 😱 잘못된 이해

### Before: "함수로 잘 쪼개면 코드도 깔끔하고 성능도 상관없다"

```
잘못된 믿음:
  "추상화는 공짜다. 컴파일러가 어차피 최적화한다."

실제 상황 1 — 별도 컴파일 단위:
  // math_utils.cpp
  int add(int a, int b) { return a + b; }

  // main.cpp
  for (int i = 0; i < 1'000'000; i++)
      total += add(i, i);

  → 링크 전에는 main.cpp가 add()의 본체를 볼 수 없음
  → 컴파일러가 인라이닝 불가
  → 100만 번 × 실제 call/ret 발생

실제 상황 2 — 가상 함수:
  class Base { virtual int compute(int x) = 0; };
  for (auto& obj : objects)
      total += obj.compute(x);

  → vtable 역참조 + 간접 call
  → 분기 예측기가 예측 못 함 (Ch4-02에서 상세)
  → 호출 비용이 더 커짐

실제 상황 3 — 디버그 빌드(-O0):
  모든 함수가 실제 call/ret로 번역됨
  → 릴리즈 빌드 대비 5~10배 느린 주요 원인
  → "내 코드 느린 이유"가 단순히 최적화 미적용인 경우가 많음
```

---

## ✨ 올바른 이해

### After: 호출 규약을 알면 비용이 보인다

```
System V AMD64 ABI (Linux/macOS x86-64 표준):

인수 전달 순서 (정수/포인터):
  1번째: RDI
  2번째: RSI
  3번째: RDX
  4번째: RCX
  5번째: R8
  6번째: R9
  7번째 이상: 스택 (오른쪽에서 왼쪽 순서로 push)

부동소수점 인수:
  XMM0 ~ XMM7 (최대 8개)

반환값:
  정수/포인터: RAX (크면 RAX + RDX)
  부동소수점: XMM0

Caller-saved (호출자가 보존): RCX, RDX, RSI, RDI, R8~R11, XMM0~XMM15
Callee-saved (피호출자가 보존): RBX, RBP, R12~R15

→ 인수 6개 이하: 레지스터만으로 전달 = 스택 쓰기 없음 (빠름)
→ 인수 7개 이상: 초과분을 스택에 push = 추가 비용 발생
→ 함수 설계 시 인수를 6개 이하로 유지하는 것이 성능상 유리
```

---

## 🔬 내부 동작 원리

### 1. call/ret의 실제 동작과 스택 레이아웃

```
호출 전 스택 상태:
  ┌─────────────────┐  ← RSP (스택 포인터)
  │    ...          │
  └─────────────────┘

call foo 실행 시:
  ① RSP -= 8
  ② [RSP] = 복귀 주소 (call 다음 명령의 RIP)
  ③ RIP = foo의 주소

foo의 prologue:
  push rbp          ; 호출자의 RBP 저장
  mov  rbp, rsp     ; 새 프레임 포인터 설정
  sub  rsp, 32      ; 로컬 변수를 위한 스택 공간 확보

  ┌─────────────────┐  ← RSP (새 프레임 바닥)
  │  로컬 변수들    │  32 bytes
  ├─────────────────┤  ← RBP
  │  저장된 RBP     │  8 bytes
  ├─────────────────┤
  │  복귀 주소      │  8 bytes
  ├─────────────────┤
  │  호출자 스택... │
  └─────────────────┘

foo의 epilogue:
  leave             ; = mov rsp, rbp; pop rbp (2개를 1개로 압축)
  ret               ; [RSP]를 RIP에 로드하고 RSP += 8

→ call: ~1 사이클 (복귀 주소 push + JMP)
→ prologue 3줄: ~3 사이클 (push + mov + sub)
→ epilogue leave: ~2 사이클
→ ret: ~1~5 사이클 (복귀 주소 예측 가능 여부에 따라)
→ 합계: ~7~11 사이클 (본체 제외)
```

### 2. godbolt로 보는 실제 어셈블리

```cpp
// C++ 코드
int add(int a, int b) {
    return a + b;
}

int caller(int x) {
    return add(x, x * 2);
}
```

```asm
; -O0 (최적화 없음) — 실제 호출 발생
add(int, int):
    push   rbp
    mov    rbp, rsp
    mov    DWORD PTR [rbp-4], edi   ; a를 스택에 저장
    mov    DWORD PTR [rbp-8], esi   ; b를 스택에 저장
    mov    edx, DWORD PTR [rbp-4]
    mov    eax, DWORD PTR [rbp-8]
    add    eax, edx                  ; a + b
    pop    rbp
    ret

caller(int):
    push   rbp
    mov    rbp, rsp
    sub    rsp, 8
    mov    DWORD PTR [rbp-4], edi   ; x를 스택에 저장
    mov    eax, DWORD PTR [rbp-4]
    lea    edx, [rax+rax]           ; x*2 계산
    mov    eax, DWORD PTR [rbp-4]
    mov    edi, eax
    call   add(int, int)            ; ← 실제 call 명령
    leave
    ret

; -O2 (최적화) — 인라이닝으로 call 완전 제거
caller(int):
    lea    eax, [rdi+rdi*2]         ; x + x*2 = 3x, 딱 1개 명령
    ret

; → -O0: 20+ 명령   vs   -O2: 2개 명령
; → 호출 오버헤드가 완전히 사라짐
```

### 3. 스택 프레임 없는 leaf 함수 최적화

```asm
; 스택을 사용하지 않는 leaf 함수 (다른 함수를 호출하지 않음):
; 컴파일러는 push rbp / pop rbp를 생략할 수 있음 (omit-frame-pointer)

; gcc -O2 -fomit-frame-pointer
add(int, int):
    lea    eax, [rdi+rsi]    ; 단 1개 명령 (add가 실제 호출된다면)
    ret

; RBP 저장 생략 → 레지스터 1개 절약 + 2 사이클 절약
; 단점: 디버거에서 스택 역추적(backtrace) 어려워짐
; 프로덕션 릴리즈에서는 대부분 -fomit-frame-pointer 적용됨
```

### 4. `__attribute__((always_inline))`과 `inline` 키워드의 차이

```cpp
// C++의 inline 키워드 — 컴파일러에 대한 "힌트" (강제 아님)
inline int add(int a, int b) { return a + b; }
// 컴파일러는 크기가 크거나 재귀적이면 무시할 수 있음

// GCC/Clang 확장 — 컴파일러에 강제 인라이닝 지시
__attribute__((always_inline))
inline int add(int a, int b) { return a + b; }

// MSVC (Windows)
__forceinline int add(int a, int b) { return a + b; }

// 주의: 큰 함수에 always_inline → i-cache 오염 가능
// 10줄짜리 함수가 100곳에서 인라이닝 → 코드 크기 10배 → i-cache 미스 증가

// 반대: 절대 인라이닝 금지
__attribute__((noinline))
int never_inline(int a, int b) { return a + b; }
// 용도: 측정/벤치마크 시 최적화 방지, 에러 핸들링 경로 분리
```

### 5. LTO(Link Time Optimization)가 호출 비용을 제거하는 방법

```
일반 컴파일 모델의 한계:

  math.cpp  →  math.o   ←  add()의 본체가 여기 있음
  main.cpp  →  main.o   ←  add()의 선언만 보임 → 인라이닝 불가

  링크 단계: math.o + main.o → a.out
    → 이미 기계어로 번역된 상태 → 인라이닝 불가

LTO 적용 시:
  gcc -O2 -flto math.cpp main.cpp -o a.out
  또는
  clang -O2 -flto=thin math.cpp main.cpp -o a.out

  컴파일 단계: .o 파일에 IR(Intermediate Representation) 저장
  링크 단계: 모든 .o의 IR을 합쳐서 전체를 하나처럼 최적화
    → main.cpp에서 add()의 본체가 보임
    → 크로스-파일 인라이닝 가능!
    → 크로스-파일 상수 전파, 데드코드 제거도 가능

실제 효과 (대규모 C++ 프로젝트):
  LTO 없음 → 있음: 5~15% 성능 향상, 바이너리 크기 감소
  Mozilla, Chromium, LLVM 자체 모두 LTO 기본 적용

JVM 연결:
  Java JIT(C2 컴파일러)가 하는 인라이닝이 LTO와 유사한 효과
  → jvm-deep-dive의 JIT 최적화 챕터 참조
```

### 6. 호출 깊이와 스택 오버플로우

```
스택 크기 기본값:
  Linux: ulimit -s → 보통 8MB
  Windows: 1MB (기본)
  각 스레드는 독립적인 스택

재귀 호출 깊이 제한:
  각 스택 프레임 크기가 1KB라면:
    8MB / 1KB = 8,192 단계까지 가능
    → 피보나치 재귀로 n=100,000 → 스택 오버플로우

TCO(Tail Call Optimization):
  // 꼬리 재귀: 함수의 마지막 동작이 자기 자신 호출
  int sum(int n, int acc) {
      if (n == 0) return acc;
      return sum(n - 1, acc + n);   // ← 꼬리 재귀
  }

  // 컴파일러가 TCO를 적용하면:
  sum:
      test  edi, edi
      je    .return_acc
      add   esi, edi                ; acc += n
      dec   edi                     ; n -= 1
      jne   sum                     ; ← JMP (call이 아님)
  .return_acc:
      mov   eax, esi
      ret

  → 스택 프레임을 새로 만들지 않고 같은 프레임 재사용
  → O(N) 스택 → O(1) 스택 (무한 재귀 가능)

  주의: C++은 TCO를 보장하지 않음 (컴파일러 재량)
        Scheme, Haskell은 명세로 보장
        Java JIT: 부분적으로 TCO 적용 (조건 있음)
```

---

## 💻 실전 실험

### 실험 1: godbolt에서 인라이닝 전후 어셈블리 비교

```cpp
// https://godbolt.org/ — GCC 13.2, -O2 로 시작

// 실험 코드
#include <numeric>

// 인라이닝 허용 버전
int sum_inline(const int* arr, int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s += arr[i];
    return s;
}

// 인라이닝 금지 버전 (실제 call 강제)
__attribute__((noinline))
int helper(int a, int b) { return a + b; }

int sum_noinline(const int* arr, int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s = helper(s, arr[i]);    // ← 루프마다 실제 call
    return s;
}

// 관찰 포인트:
// sum_inline: 루프 본체에 add 명령만 → 벡터화까지 가능
// sum_noinline: 루프마다 call helper / ret 발생
// → 생성된 명령어 수 비교, 루프 구조 비교
```

### 실험 2: Google Benchmark로 호출 비용 측정

```cpp
#include <benchmark/benchmark.h>

__attribute__((noinline))
int add_noinline(int a, int b) { return a + b; }

__attribute__((always_inline))
inline int add_inline(int a, int b) { return a + b; }

static void BM_NoinlineCall(benchmark::State& state) {
    int sum = 0;
    for (auto _ : state) {
        for (int i = 0; i < 1000; i++)
            sum = add_noinline(sum, i);
        benchmark::DoNotOptimize(sum);
    }
}

static void BM_InlineCall(benchmark::State& state) {
    int sum = 0;
    for (auto _ : state) {
        for (int i = 0; i < 1000; i++)
            sum = add_inline(sum, i);
        benchmark::DoNotOptimize(sum);
    }
}

BENCHMARK(BM_NoinlineCall);
BENCHMARK(BM_InlineCall);
BENCHMARK_MAIN();
```

```bash
# 빌드 및 실행
g++ -O2 -std=c++17 bench.cpp -lbenchmark -lbenchmark_main -lpthread -o bench
./bench --benchmark_format=console

# 예상 출력:
# BM_NoinlineCall   1200 ns    1200 ns    582583
# BM_InlineCall      180 ns     180 ns   3881234
# → 약 6~7배 차이 (루프 1000번 × 호출 오버헤드 누적)
```

### 실험 3: perf로 call 명령어 빈도 측정

```bash
# 실제 call/ret 발생 횟수 측정
perf stat -e instructions,cycles,branches,branch-misses \
    ./your_program

# 더 정밀하게: 인라이닝 전후 IPC 비교
# IPC(Instructions Per Cycle)가 높아지면 인라이닝이 효과적
perf stat -e instructions,cycles ./bench_noinline
perf stat -e instructions,cycles ./bench_inline

# 호출 그래프 기록 (어떤 함수가 몇 번 호출되는지)
perf record -g ./your_program
perf report --call-graph dwarf

# -fno-omit-frame-pointer 빌드 권장 (스택 역추적 정확도)
g++ -O2 -fno-omit-frame-pointer -g -o your_program main.cpp
```

### 실험 4: LTO 효과 측정

```bash
# 두 파일로 분리된 코드
# math.cpp: 작은 함수들 정의
# main.cpp: 루프에서 호출

# LTO 없이
g++ -O2 math.cpp main.cpp -o bench_no_lto

# LTO 있음
g++ -O2 -flto math.cpp main.cpp -o bench_lto

# 실행 시간 비교
time ./bench_no_lto
time ./bench_lto

# 바이너리 크기도 비교 (LTO가 더 작을 수 있음)
ls -lh bench_no_lto bench_lto

# objdump로 인라이닝 확인
objdump -d bench_lto | grep -A 20 "<main>"
# LTO 적용 시 main() 안에 add() 본체가 포함됨
```

---

## 📊 성능 비교

```
함수 호출 방식별 오버헤드 (x86-64, 인수 2개, 본체 1개 명령 기준):

호출 방식                    오버헤드(사이클)   명령어 수   특이사항
────────────────────────────────────────────────────────────────────
인라이닝 (컴파일 시간 결정)     0               0          최적
직접 호출 (같은 TU, 인라인)     0               0          -O2 기본
직접 호출 (다른 TU, LTO 없음)  ~8~12           ~10        call+ret+프레임
직접 호출 (LTO 있음)           0~2             0~2        크로스파일 인라인
가상 함수 호출                 ~12~20+         ~5         vtable역참조 (Ch4-02)
함수 포인터 호출               ~8~15           ~5         간접 분기

Google Benchmark 실측 (루프 10,000회, 본체 int add):
  always_inline:   ~30 ns  (루프 오버헤드만)
  일반 call:       ~250 ns (호출 × 10,000)
  noinline:        ~280 ns (캐시 관계로 비슷)

IPC 비교 (perf stat):
  인라이닝 버전:  IPC ~3.5 (벡터화까지 적용됨)
  noinline 버전:  IPC ~1.8 (call/ret가 파이프라인 채움)
```

---

## ⚖️ 트레이드오프

```
인라이닝 결정 기준:

인라이닝이 유리한 경우:
  ✅ 함수 본체가 매우 작음 (5줄 이하)
  ✅ 핫 루프 안에서 자주 호출됨 (10만 회 이상/초)
  ✅ 인수/반환값 타입 변환(형변환, boxing)이 없어짐
  ✅ 인라이닝 후 추가 최적화 가능 (상수 전파, dead code 제거)
  ✅ 같은 TU 내 or LTO 적용 환경

인라이닝이 불리한 경우:
  ❌ 함수 본체가 큼 (100줄 이상) → 코드 크기 급증 → i-cache 압박
  ❌ 여러 호출 지점에서 인라이닝 → 코드 중복 → 바이너리 비대화
  ❌ 에러 처리 경로 (콜드 경로) → 인라이닝 이득 없음, 크기만 증가
  ❌ 재귀 함수 → 인라이닝 불가 (또는 제한적으로만 가능)
  ❌ 디버깅 빌드 → 스택 트레이스 가독성 중요

컴파일러의 자동 인라이닝 기준 (GCC):
  --param max-inline-insns-single=400   (기본값: 명령어 400개 이하)
  --param inline-min-speedup=10         (최소 10% 속도 향상 예상 시)
  -finline-functions                    (-O3에서 기본 활성화)

JVM 연결:
  HotSpot JIT(C2)의 인라이닝 기준:
    -XX:MaxInlineSize=35 (바이트코드 35 bytes 이하)
    -XX:FreqInlineSize=325 (핫 메서드는 325 bytes까지)
    호출 횟수 임계값 초과 시 자동 인라이닝
    → jvm-deep-dive 레포의 JIT 컴파일 챕터 참조
```

---

## 📌 핵심 정리

```
함수 호출 비용 핵심:

System V AMD64 호출 규약:
  인수 6개까지: RDI/RSI/RDX/RCX/R8/R9 (스택 없음)
  7번째부터: 스택에 push (추가 비용)
  반환: RAX (정수/포인터)
  → 인수 6개 이하 설계 시 레지스터만으로 전달 가능

스택 프레임 비용:
  prologue: push rbp + mov rbp,rsp + sub rsp,N (~3 사이클)
  epilogue: leave + ret (~3~7 사이클)
  합계: ~6~15 사이클 (본체 제외)
  → 본체 1줄 함수의 경우 오버헤드가 본체보다 큼

인라이닝으로 비용 제거:
  같은 TU: -O2에서 컴파일러가 자동 판단
  다른 TU: LTO(-flto) 필요
  강제: __attribute__((always_inline))
  금지: __attribute__((noinline))

LTO:
  크로스 파일 인라이닝, 상수 전파, dead code 제거
  → 5~15% 성능 향상 (대형 프로젝트 기준)
  → 빌드 시간 증가는 감수해야 함

인라이닝 트레이드오프:
  함수가 작고 핫할수록 인라이닝 이득 큼
  함수가 크고 콜드할수록 코드 크기 증가만 유발
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 두 함수 중 `-O2`에서 어느 것이 실제 `call` 명령을 생성하는가? 그 이유는?

```cpp
// A
inline int square(int x) { return x * x; }
int use_square(int n) {
    int s = 0;
    for (int i = 0; i < n; i++) s += square(i);
    return s;
}

// B
__attribute__((noinline)) int square(int x) { return x * x; }
int use_square_noinline(int n) {
    int s = 0;
    for (int i = 0; i < n; i++) s += square(i);
    return s;
}
```

<details>
<summary>해설 보기</summary>

**A (inline)**: `-O2`에서 `square`는 인라이닝됩니다. `inline` 키워드가 힌트를 줬고, 함수가 매우 작으므로 컴파일러는 실제 `call` 대신 루프 안에 `imul edi, edi` 한 줄을 집어넣습니다. godbolt에서 확인하면 `use_square`의 어셈블리에 `call square`가 없고 자동 벡터화까지 적용된 SIMD 명령어가 나타날 수 있습니다.

**B (noinline)**: `__attribute__((noinline))`은 컴파일러에 인라이닝 금지를 **강제**합니다. 루프마다 `call square` 명령이 발생합니다. 이 경우 루프 1,000만 회라면 1,000만 번의 실제 call/ret가 발생합니다.

핵심: `inline`은 힌트일 뿐이지만 `noinline`과 `always_inline`은 강제 지시어입니다. 측정 코드를 작성할 때 컴파일러가 최적화로 측정 대상을 없애버리지 않도록 `noinline`을 적극 활용해야 합니다.

</details>

---

**Q2.** System V AMD64에서 인수를 7개 받는 함수 `int f(int a, int b, int c, int d, int e, int f, int g)`를 호출하면 7번째 인수는 어떻게 전달되는가? 그리고 이것이 성능에 미치는 영향은?

<details>
<summary>해설 보기</summary>

6번째까지(`a~f`)는 RDI, RSI, RDX, RCX, R8, R9에 배치됩니다. 7번째 `g`는 **스택에 push**됩니다. 구체적으로 호출 전에 `push g_value`가 실행되고, 피호출자는 `[rbp+16]` 또는 `[rsp+8]`로 스택에서 읽습니다.

성능 영향:
1. 스택 쓰기 1회 추가: L1 캐시 히트라면 ~4 사이클이지만 스택 메모리 접근 자체가 추가됩니다.
2. 7번째 이후 인수는 모두 스택 경유이므로 인수가 10개인 함수는 스택 push 4개 + 스택 읽기 4개 추가됩니다.
3. 더 큰 문제: 스택의 인수는 인라이닝 후 `caller-saved` 레지스터 재배치와 충돌을 일으킬 수 있어 컴파일러의 레지스터 할당 최적화를 방해합니다.

실무 팁: 인수가 많은 함수는 구조체(또는 포인터)로 묶으면 레지스터 1개로 전달 가능합니다. `void f(const Options& opts)` 패턴이 이 이유에서도 유리합니다.

</details>

---

**Q3.** 재귀 피보나치 `fib(n)`을 `-O3`로 컴파일하면 스택 오버플로우를 피할 수 있는가? Tail Call Optimization이 적용되는 조건은 무엇인가?

<details>
<summary>해설 보기</summary>

일반 피보나치 `return fib(n-1) + fib(n-2)`는 **꼬리 재귀가 아닙니다**. `fib(n-1)`을 호출한 후 반환값을 `fib(n-2)` 결과와 더해야 하므로, 호출 직후 추가 연산이 필요합니다. 컴파일러는 TCO를 적용할 수 없고 `-O3`도 스택 오버플로우를 막지 못합니다.

TCO가 적용되는 조건:
1. 함수의 **마지막 동작**이 자기 자신 호출이어야 함
2. 반환값을 변형 없이 그대로 반환해야 함

꼬리 재귀 형태로 변환하면 TCO 가능:
```cpp
int fib_tail(int n, int a, int b) {
    if (n == 0) return a;
    return fib_tail(n-1, b, a+b);  // ← 꼬리 재귀
}
```
이 형태는 `-O2/-O3`에서 GCC/Clang이 루프로 변환합니다. godbolt에서 확인하면 `jmp fib_tail` (call 아님)가 보입니다. JVM은 TCO를 명세에서 보장하지 않아 Scala의 `@tailrec` 어노테이션이나 trampoline 패턴이 필요합니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: JMM과 하드웨어 연결](../memory-model-concurrency/06-jmm-hardware-bridge.md)** | **[홈으로 🏠](../README.md)** | **[다음: 가상 함수·동적 디스패치 ➡️](./02-virtual-functions.md)**

</div>
