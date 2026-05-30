# C 한 줄이 기계어가 되기까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- C 소스코드가 CPU에서 실행되기까지 거치는 4단계는 무엇인가?
- `int x = a + b;` 한 줄이 x86-64에서 어떤 명령어들로 변환되는가?
- `objdump -d`로 ELF 바이너리를 역어셈블하면 무엇이 보이는가?
- `-O0`(최적화 없음)과 `-O2`(최적화)의 어셈블리 차이는 어느 정도인가?
- godbolt.org에서 실시간으로 어셈블리를 확인하는 방법은?
- MOV/ADD/CMP/Jcc 명령어는 각각 내부에서 어떤 연산을 하는가?

---

## 🔍 왜 이 개념이 중요한가

### "내 코드가 CPU에서 실제로 무슨 명령인지 모른다"

```
흔한 상황:
  C/C++ 코드를 짜고 "컴파일해서 실행"하면 끝
  내부에서 무슨 명령어가 도는지는 블랙박스

이것이 문제인 이유:
  int sum = 0;
  for (int i = 0; i < N; i++) sum += arr[i];
  
  이 루프가 -O0에서는 로드/스토어 폭탄이고
  -O2에서는 벡터 명령어(SIMD) + 루프 언롤링으로 바뀐다
  
  차이를 모르면:
    "왜 릴리즈 빌드가 10배 빠른지" 설명 못함
    "컴파일러가 내 의도대로 최적화했는지" 검증 못함
    프로파일러가 "이 어셈블리 라인이 핫스팟"이라고 가리켜도 해석 불가

이것을 알면:
  컴파일러의 최적화를 측정으로 검증
  어셈블리를 읽어 "이 연산이 왜 느린지" 진단
  Ch1-03~04에서 다루는 파이프라인 해저드를 어셈블리 수준에서 추적
```

---

## 😱 잘못된 이해

### Before: "고수준 코드가 그대로 CPU에서 돈다"

```
잘못된 모델:
  C 코드 한 줄 = CPU가 실행하는 "한 동작"
  컴파일러는 코드를 기계어로 '번역'하는 단순한 도구
  최적화 옵션은 "약간 빠르게 해주는 수준"

예:
  int z = x + y;
  → "덧셈 한 번 실행"

실제 -O0 어셈블리 (Intel syntax):
  mov   eax, DWORD PTR [rbp-8]    ; x를 레지스터로 로드
  mov   edx, DWORD PTR [rbp-12]   ; y를 레지스터로 로드
  add   eax, edx                  ; 덧셈
  mov   DWORD PTR [rbp-4], eax    ; z에 결과 스토어

실제 -O2 어셈블리:
  lea   eax, [rdi+rsi]            ; 메모리 스토어 없이 바로 계산
  → 로드/스토어 제거됨

간단한 덧셈 한 줄도 최적화 수준에 따라
명령어 수, 레지스터 활용, 메모리 접근 패턴이 완전히 다르다
```

---

## ✨ 올바른 이해

### After: 4단계 변환 파이프라인과 어셈블리 읽기

```
C 소스 → 전처리 → 컴파일 → 어셈블리 → 링크 → 실행 파일

각 단계가 만드는 것:
  전처리 (cpp):     #include/#define 치환 → .i 파일
  컴파일 (cc1):     C → 어셈블리(.s) 생성, 최적화 수행
  어셈블 (as):      어셈블리 → 오브젝트(.o, 기계어 바이너리)
  링크 (ld):        여러 .o + 라이브러리 → 실행 파일(ELF)

CPU가 실제로 보는 것:
  바이트 열 (기계어)
  예: 01 c6        →  add esi, eax
      48 8b 45 f8  →  mov rax, QWORD PTR [rbp-8]

어셈블리 = 기계어의 사람이 읽을 수 있는 표현
  니모닉 + 오퍼랜드
  MOV dst, src   → src를 dst로 복사
  ADD dst, src   → dst += src
  CMP a, b       → a - b를 계산하고 플래그만 설정 (결과 버림)
  JE  label      → ZF(제로 플래그)가 1이면 label로 점프

컴파일러 최적화는:
  "의미가 동일한 더 빠른 명령어 시퀀스 선택"
  불필요한 로드/스토어 제거
  루프 언롤링, 인라이닝, SIMD 자동 변환
```

---

## 🔬 내부 동작 원리

### 1. 전처리 → 컴파일 → 어셈블 → 링크 단계별 확인

```bash
# 예제 소스: demo.c
cat > demo.c << 'EOF'
#include <stdio.h>

int add(int a, int b) {
    return a + b;
}

int main(void) {
    int x = 10, y = 20;
    int z = add(x, y);
    printf("%d\n", z);
    return 0;
}
EOF

# 1단계: 전처리만
gcc -E demo.c -o demo.i
# demo.i: #include 치환, 매크로 확장됨 (수백 줄)
head -5 demo.i

# 2단계: 어셈블리 생성 (Intel syntax, 최적화 없음)
gcc -O0 -S -masm=intel demo.c -o demo_O0.s
cat demo_O0.s

# 3단계: 오브젝트 파일 생성
gcc -O0 -c demo.c -o demo.o
# ELF 형식의 바이너리, 링크 전 상태

# 4단계: 링크 (실행 파일)
gcc -O0 demo.c -o demo_O0
```

### 2. x86-64 핵심 명령어 해부

```
x86-64 기본 레지스터 (64비트):
  rax, rbx, rcx, rdx, rsi, rdi    범용 레지스터
  rbp                              베이스 포인터 (스택 프레임)
  rsp                              스택 포인터
  rip                              명령어 포인터 (PC)

32비트 부분 사용:
  eax = rax의 하위 32비트
  (int 연산은 eax/ecx/edx/... 사용)

핵심 명령어:

  MOV dst, src
    메모리↔레지스터, 레지스터↔레지스터 복사
    mov eax, 42          ; 상수를 레지스터에
    mov eax, [rbp-8]     ; 스택 메모리에서 로드
    mov [rbp-4], eax     ; 스택 메모리로 스토어

  ADD dst, src
    dst = dst + src, 플래그(CF/ZF/SF/OF) 설정
    add eax, ecx         ; eax += ecx
    add eax, 1           ; eax += 1 (inc eax와 동일)

  SUB dst, src
    dst = dst - src
    sub rsp, 32          ; 스택 포인터 이동 (스택 프레임 할당)

  CMP a, b
    a - b 계산, 결과는 버리고 플래그만 설정
    cmp eax, 0           ; eax가 0인지 확인용

  조건 점프 (Jcc):
    JE  / JZ   → ZF=1 (같음/0)
    JNE / JNZ  → ZF=0
    JL  / JNGE → SF≠OF (부호 있는 미만)
    JG  / JNLE → ZF=0 AND SF=OF (부호 있는 초과)
    JB  / JNAE → CF=1 (부호 없는 미만)

  LEA dst, [base + offset]
    실제 메모리 접근 없이 주소 계산
    lea eax, [rdi+rsi]   ; eax = rdi + rsi (덧셈 트릭으로 활용)
    lea rax, [rdi*4+rdi] ; rax = rdi*5 (곱셈 대체)

  CALL / RET
    call <label>  → 리턴 주소를 스택에 push + 점프
    ret           → 스택에서 리턴 주소 pop + 점프
```

### 3. -O0 vs -O2 어셈블리 비교: add 함수

```c
// 소스 코드
int add(int a, int b) {
    return a + b;
}
```

```asm
; -O0 (최적화 없음) — Intel syntax
; gcc -O0 -S -masm=intel demo.c
add(int, int):
    push   rbp              ; 스택 프레임 시작
    mov    rbp, rsp
    mov    DWORD PTR [rbp-4], edi   ; 인자 a를 스택에 스토어
    mov    DWORD PTR [rbp-8], esi   ; 인자 b를 스택에 스토어
    mov    edx, DWORD PTR [rbp-4]   ; a를 다시 레지스터로 로드
    mov    eax, DWORD PTR [rbp-8]   ; b를 다시 레지스터로 로드
    add    eax, edx                 ; 덧셈
    pop    rbp              ; 스택 프레임 해제
    ret                     ; 결과는 eax에

; 명령어 수: 8개, 불필요한 스택 왕복 4번
```

```asm
; -O2 (최적화) — Intel syntax
; gcc -O2 -S -masm=intel demo.c
add(int, int):
    lea    eax, [rdi+rsi]   ; 단 1 명령어! 주소 계산 트릭으로 덧셈
    ret

; 명령어 수: 2개
; 스택 프레임 불필요 → push/pop 없음
; 인자는 레지스터(rdi/rsi)에서 직접 사용 (System V AMD64 호출 규약)
; → 8배 적은 명령어 수, 스택 메모리 접근 0회
```

### 4. 루프 코드의 -O0 vs -O2 비교

```c
// 배열 합산 루프
long sum_array(int *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        sum += arr[i];
    }
    return sum;
}
```

```asm
; -O0: 매 반복마다 스택에서 i와 sum을 로드/스토어
sum_array:
    push    rbp
    mov     rbp, rsp
    mov     QWORD PTR [rbp-24], rdi  ; arr 스택 저장
    mov     DWORD PTR [rbp-28], esi  ; n 스택 저장
    mov     QWORD PTR [rbp-8], 0     ; sum = 0 (스택)
    mov     DWORD PTR [rbp-12], 0    ; i = 0 (스택)
.L3:
    mov     eax, DWORD PTR [rbp-12]  ; i 로드
    cmp     eax, DWORD PTR [rbp-28]  ; i < n 비교
    jge     .L4                      ; i >= n이면 탈출
    mov     eax, DWORD PTR [rbp-12]  ; i 로드 (또!)
    cdqe                             ; eax → rax (부호 확장)
    lea     rdx, [rax*4]             ; 바이트 오프셋
    mov     rax, QWORD PTR [rbp-24]  ; arr 포인터 로드
    add     rdx, rax                 ; &arr[i]
    mov     eax, DWORD PTR [rdx]     ; arr[i] 로드
    cdqe
    add     QWORD PTR [rbp-8], rax   ; sum += arr[i] (스택에!)
    add     DWORD PTR [rbp-12], 1    ; i++ (스택에!)
    jmp     .L3
.L4:
    mov     rax, QWORD PTR [rbp-8]   ; sum 로드해서 반환
    pop     rbp
    ret
; 루프 바디: 약 12 명령어, 매 반복마다 4~5회 메모리 접근
```

```asm
; -O2: 레지스터만 사용, 루프 오버헤드 제거
sum_array:
    test    esi, esi              ; n == 0?
    jle     .L4                   ; n <= 0이면 바로 0 반환
    lea     eax, [rsi-1]          ; n-1
    lea     rax, [rdi+rax*4+4]    ; arr 끝 포인터
    xor     edx, edx              ; sum = 0 (레지스터!)
.L3:
    mov     ecx, DWORD PTR [rdi]  ; arr[i] 로드 (rdi가 포인터 직접 전진)
    add     rdi, 4                ; 포인터 증가
    movsx   rcx, ecx              ; 부호 확장
    add     rdx, rcx              ; sum += arr[i] (레지스터!)
    cmp     rdi, rax              ; 끝 도달?
    jne     .L3
    mov     rax, rdx
    ret
.L4:
    xor     eax, eax
    ret
; 루프 바디: 5 명령어, 메모리 접근 1회 (로드만)
; sum은 레지스터(rdx)에 유지 → 스택 왕복 없음
```

### 5. objdump로 ELF 바이너리 역추적

```bash
# 컴파일된 바이너리를 역어셈블
gcc -O2 demo.c -o demo
objdump -d -M intel demo | grep -A 20 "<add>"

# 출력 예시:
# 0000000000001140 <add>:
#     1140:  8d 04 37    lea    eax,[rdi+rsi*1]
#     1143:  c3          ret
#
# 왼쪽부터: 가상 주소 | 기계어 바이트 | 어셈블리 니모닉

# 더 상세한 분석: 소스와 어셈블리 함께 보기 (디버그 정보 필요)
gcc -O2 -g demo.c -o demo_dbg
objdump -d -M intel -S demo_dbg | head -60

# 특정 섹션만 확인
objdump -d -M intel demo | grep -A 5 "<main>"

# readelf로 ELF 헤더 확인
readelf -h demo
readelf -S demo  # 섹션 목록 (.text, .data, .bss 등)

# nm으로 심볼 테이블
nm demo | grep -E " (T|t) "  # 코드 심볼 목록
```

```
ELF 바이너리 구조:
  ┌────────────────────────────────────┐
  │  ELF Header                        │  파일 타입, ISA, 엔트리 포인트
  ├────────────────────────────────────┤
  │  .text  (코드 섹션)                 │  실제 기계어 바이트들
  │    add():   8d 04 37 c3             │
  │    main():  55 48 89 e5 ...         │
  ├────────────────────────────────────┤
  │  .rodata (읽기전용 데이터)           │  문자열 리터럴 등
  │    "%d\n": 25 64 0a 00              │
  ├────────────────────────────────────┤
  │  .data  (초기화된 전역변수)          │
  ├────────────────────────────────────┤
  │  .bss   (초기화 안 된 전역변수)      │  파일에는 크기만, 로드 시 0으로 채워짐
  └────────────────────────────────────┘
```

### 6. godbolt에서 실시간 어셈블리 확인

```
godbolt.org 사용법:
  1. 좌측 패널에 C/C++ 코드 입력
  2. 우측 컴파일러 선택: "x86-64 gcc 13.2"
  3. 컴파일러 옵션 입력: -O2 -masm=intel -march=native
  4. 우측 패널에서 어셈블리 실시간 확인
  5. 코드 라인을 클릭하면 대응하는 어셈블리가 하이라이트

유용한 옵션 조합:
  -O0 -masm=intel               : 최적화 없음, 이해하기 쉬운 어셈블리
  -O2 -masm=intel               : 최적화 적용, 실제 릴리즈 빌드
  -O3 -masm=intel -march=native : 고급 최적화 + SIMD 자동 벡터화
  -O2 -masm=intel -fno-inline   : 인라이닝 없이 (함수 경계 유지)

확인 포인트:
  함수 호출이 인라이닝됐는지 (call 명령어 존재 여부)
  루프가 벡터화됐는지 (vmovdqu / vpaddd 등 SIMD 명령어)
  불필요한 mov 체인이 사라졌는지 (레지스터 최적화)
```

---

## 💻 실전 실험

### 실험 1: -O0 vs -O2 어셈블리 크기 비교

```bash
# 파일 준비
cat > bench.c << 'EOF'
#include <stdlib.h>

long sum_array(int *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++)
        sum += arr[i];
    return sum;
}

int main(void) {
    int N = 10000000;
    int *arr = malloc(N * sizeof(int));
    for (int i = 0; i < N; i++) arr[i] = i;
    long s = sum_array(arr, N);
    free(arr);
    return (int)s;
}
EOF

# 컴파일 및 어셈블리 생성
gcc -O0 -S -masm=intel bench.c -o bench_O0.s
gcc -O2 -S -masm=intel bench.c -o bench_O2.s

# 어셈블리 크기 비교
echo "=== sum_array 함수 명령어 수 ==="
echo "-O0:"
awk '/^sum_array:/{p=1} p && /^\./{exit} p && /^\t[a-z]/{count++} END{print count}' bench_O0.s
echo "-O2:"
awk '/^sum_array:/{p=1} p && /^\./{exit} p && /^\t[a-z]/{count++} END{print count}' bench_O2.s

# 실제 성능 측정
gcc -O0 -o bench_O0 bench.c
gcc -O2 -o bench_O2 bench.c

echo "=== 실행 시간 비교 ==="
time ./bench_O0
time ./bench_O2

# 예상 결과:
# -O0: 명령어 수 약 15~20개, 실행 시간 ~40ms
# -O2: 명령어 수 약 5~8개, 실행 시간 ~5ms (8배 차이)
```

### 실험 2: perf stat으로 IPC 비교

```bash
# IPC(Instructions Per Cycle) 및 캐시 미스 측정
echo "=== -O0 성능 카운터 ==="
perf stat -e cycles,instructions,cache-misses,branch-misses ./bench_O0 2>&1

echo "=== -O2 성능 카운터 ==="
perf stat -e cycles,instructions,cache-misses,branch-misses ./bench_O2 2>&1

# 예상 결과 (대략):
# -O0: IPC ≈ 1.2, instructions ≈ 350M (불필요한 로드/스토어 포함)
# -O2: IPC ≈ 3.5, instructions ≈ 45M (레지스터 최적화)
# → 명령어 수가 7배 줄고, IPC는 3배 향상 → 합산 약 20배 차이

# objdump로 핫 명령어 확인
perf record -g ./bench_O2
perf report --stdio | head -40
```

### 실험 3: godbolt에서 조건 분기 어셈블리 확인

```c
// godbolt.org에 입력 (컴파일러: x86-64 gcc, 옵션: -O2 -masm=intel)
int classify(int x) {
    if (x > 0)
        return 1;
    else if (x < 0)
        return -1;
    else
        return 0;
}
```

```asm
; 예상 -O2 어셈블리 (branchless 최적화 가능)
classify(int):
    xor    eax, eax       ; result = 0
    test   edi, edi       ; x == 0?
    setg   al             ; al = (x > 0) ? 1 : 0
    movzx  eax, al
    test   edi, edi
    jns    .done          ; x >= 0이면 완료
    mov    eax, -1        ; x < 0이면 -1
.done:
    ret

; 또는 GCC가 더 영리하게:
; sar edi, 31         ; 최상위 비트로 부호 전파 (음수 → -1, 나머지 → 0)
; 이런 패턴을 branchless라 함 (Ch1-06, Ch4-04에서 상세 다룸)
```

---

## 📊 성능 비교

```
최적화 수준별 sum_array(N=10M) 성능 비교:

                   -O0          -O1          -O2          -O3
─────────────────────────────────────────────────────────────────
명령어 수 (루프)   ~15개        ~8개         ~5개         ~4개 (SIMD)
실행 시간         ~45ms        ~18ms        ~8ms         ~2ms
IPC               ~1.2         ~2.0         ~3.5         ~5.0
스택 메모리 접근  매 반복 4회   1회          0회          0회
벡터화 여부       없음         없음         없음         ymm 레지스터

-O0 대비 속도:    1x           2.5x         5.6x         22x

핵심 차이:
  -O0: 모든 변수를 스택에서 로드/스토어 (관찰성을 위해)
  -O2: 핵심 변수를 레지스터에 유지 (register allocation)
  -O3: 루프를 SIMD 명령어로 벡터화 (Ch5에서 상세 설명)

objdump 기계어 바이트 비교 (add 함수):
  -O0:  55 48 89 e5 89 7d fc 89 75 f8 8b 55 fc 8b 45 f8 01 d0 5d c3
        (20 bytes, 8 명령어)
  -O2:  8d 04 37 c3
        (4 bytes, 2 명령어) ← 5배 작음
```

---

## ⚖️ 트레이드오프

```
최적화 수준 선택:

-O0 (최적화 없음):
  ✅ 디버거(gdb)에서 변수 값을 정확히 확인 가능
  ✅ 스택 프레임이 규칙적 → 충돌 분석 쉬움
  ✅ 컴파일 속도 가장 빠름
  ❌ 실행 성능 최악 (5~20배 느림)
  ❌ 불필요한 스택 접근 폭탄

-O1 (기본 최적화):
  ✅ 대부분의 레지스터 할당 최적화
  ✅ 인라이닝 일부 수행
  ❌ 루프 언롤링, SIMD 없음

-O2 (표준 릴리즈):
  ✅ 실무 릴리즈 빌드 표준
  ✅ 루프 최적화, 인라이닝, branchless 변환
  ✅ SIMD 자동 벡터화 없음 (안전한 범위)
  ❌ 디버깅 어려움 (인라이닝으로 스택 트레이스 변형)

-O3 (공격적 최적화):
  ✅ 자동 벡터화 (SIMD), 루프 언롤링
  ✅ 수치 계산 코드에서 큰 폭 향상
  ❌ 부동소수점 결합법칙 가정 (-ffast-math와 결합 시 결과 달라질 수 있음)
  ❌ 코드 크기 증가 (루프 언롤로 i-cache 압박)

Intel syntax vs AT&T syntax:
  AT&T (GCC 기본): movl -8(%rbp), %eax  ← 소스가 왼쪽, % 접두사 많음
  Intel (MSVC/NASM): mov eax, [rbp-8]   ← 목적지가 왼쪽, 직관적
  이 레포는 Intel syntax 사용 (-masm=intel 또는 godbolt 옵션)
```

---

## 📌 핵심 정리

```
C → 기계어 4단계:
  전처리(cpp) → 컴파일(cc1) → 어셈블(as) → 링크(ld)
  각 단계 산출물: .i → .s → .o → 실행 파일(ELF)

x86-64 핵심 명령어:
  MOV: 데이터 이동 (레지스터 ↔ 메모리)
  ADD/SUB: 산술 연산 + 플래그 설정
  CMP: 비교 (결과 버리고 플래그만)
  Jcc: 조건 점프 (JE/JNE/JL/JG/JB 등)
  LEA: 주소 계산 (덧셈/곱셈 트릭으로도 활용)
  CALL/RET: 함수 호출/반환

-O0 vs -O2 차이의 본질:
  -O0: 디버거 지원 위해 모든 변수를 스택에 저장
  -O2: 핵심 변수를 레지스터에 유지, 불필요한 명령어 제거
  결과: 명령어 수 5~10배, 속도 5~20배 차이

역어셈블 도구:
  objdump -d -M intel <binary>  : ELF 바이너리 역추적
  gcc -S -masm=intel            : 어셈블리 파일 직접 생성
  godbolt.org                   : 실시간 컴파일러 탐색기

다음 단계:
  이 명령어들이 CPU 파이프라인에서 어떻게 처리되는지 →
  Ch1-02 CPU 파이프라인에서 계속
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 C 함수를 `-O2`로 컴파일했을 때 어셈블리 명령어 수가 대략 몇 개가 될지 예상하고, godbolt에서 실제로 확인하라. 스택 프레임이 생기는가?

```c
int max(int a, int b) {
    return a > b ? a : b;
}
```

<details>
<summary>해설 보기</summary>

`-O2 -masm=intel` 기준 대략:

```asm
max(int, int):
    cmp    edi, esi       ; a와 b 비교
    mov    eax, esi       ; eax = b (기본값)
    cmovg  eax, edi       ; a > b이면 eax = a (조건부 이동)
    ret
```

또는 더 간단하게:

```asm
max(int, int):
    mov    eax, edi       ; eax = a
    cmp    esi, edi       ; b vs a
    cmovge eax, esi       ; b >= a이면 eax = b
    ret
```

**핵심 포인트**:
- 명령어 수: 3~4개 (스택 프레임 없음!)
- `push rbp / mov rbp, rsp / pop rbp`가 전혀 없다 → 컴파일러가 프레임이 필요 없다고 판단
- 3항 연산자 `?:`가 분기(JG) 대신 `cmovg`(조건부 이동)로 변환됨 → branchless!
- 이것이 Ch1-05에서 다루는 "분기 예측 없이 실행되는 branchless code"의 기초

</details>

---

**Q2.** `gcc -O2`로 컴파일한 실행 파일을 `objdump -d`로 역어셈블하면 소스 코드의 `add` 함수가 보이지 않을 수 있다. 왜인가?

<details>
<summary>해설 보기</summary>

**이유: 인라이닝(Inlining)**

컴파일러는 `-O2`에서 작은 함수를 호출 지점에 직접 삽입(인라이닝)합니다. `add` 함수가 딱 한 번 `main`에서 호출된다면:

```c
int z = add(x, y);  →  int z = x + y;  (컴파일러 입장)
```

이렇게 되면 `add`라는 독립적인 심볼이 생성되지 않고, `main` 함수 안에 흡수됩니다.

**확인 방법**:
```bash
# 인라이닝 금지 옵션
gcc -O2 -fno-inline -masm=intel -S demo.c
# 이제 add 함수가 별도로 생성됨
```

또는 `__attribute__((noinline))`으로 개별 함수에 적용:
```c
__attribute__((noinline))
int add(int a, int b) { return a + b; }
```

이것이 Ch4-01에서 다루는 "함수 호출 비용"의 핵심 — 인라이닝은 `call/ret` 명령어 쌍과 스택 프레임 오버헤드를 완전히 제거합니다.

</details>

---

**Q3.** 다음 두 코드는 의미가 같다. `-O2`에서 컴파일하면 동일한 어셈블리가 생성되는가?

```c
// 버전 A
int triple_A(int x) {
    return x * 3;
}

// 버전 B
int triple_B(int x) {
    return x + x + x;
}
```

<details>
<summary>해설 보기</summary>

**결론: 동일한 어셈블리가 생성됩니다.**

실제 `-O2` 출력:
```asm
; 둘 다 동일하게:
triple_A(int):  / triple_B(int):
    lea    eax, [rdi+rdi*2]   ; eax = rdi + rdi*2 = rdi*3
    ret
```

**핵심 포인트**:
- 컴파일러는 "의미"를 기준으로 최적화하지 "소스 코드"를 기준으로 최적화하지 않음
- `x * 3` → 곱셈 명령(`imul`)이 비쌈 → `lea rdi+rdi*2`로 LEA 트릭 적용 (1 사이클)
- `x + x + x` → 두 번의 ADD처럼 보이지만 역시 같은 LEA로 수렴
- 이것이 "컴파일러가 사람보다 잘 아는" 영역의 첫 번째 예시

참고: `imul`(정수 곱셈)은 1사이클이지만 레이턴시가 3사이클입니다. LEA는 1사이클 레이턴시입니다. 컴파일러가 2의 거듭제곱이나 작은 상수 곱셈을 shift/lea로 교체하는 이유입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: CPU 파이프라인 ➡️](./02-cpu-pipeline-stages.md)**

</div>
