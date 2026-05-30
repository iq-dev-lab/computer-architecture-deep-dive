# 파이프라인 해저드 — 데이터·제어·구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- RAW 해저드(Read-After-Write)가 왜 파이프라인을 멈추게 하는가?
- 포워딩(Forwarding/Bypass)은 스톨 없이 RAW를 어떻게 해결하는가?
- 제어 해저드에서 파이프라인 플러시(flush)가 발생하면 몇 사이클이 낭비되는가?
- 구조 해저드는 무엇이며, 현대 CPU에서 어떻게 최소화되어 있는가?
- 컴파일러의 명령어 스케줄링(instruction scheduling)이란 무엇인가?
- NOP(No-Operation) 삽입이 성능에 미치는 영향은?

---

## 🔍 왜 이 개념이 중요한가

### "파이프라인이 꽉 차 있어야 빠른데, 해저드가 구멍을 만든다"

```
Ch1-02에서 배운 것:
  파이프라인이 채워지면 매 사이클 1 명령어 완료 (IPC = 1)
  슈퍼스칼라 CPU: IPC = 4~6도 가능

현실의 문제:
  명령어들이 서로 독립적이지 않다
  
  예:
    mov eax, [rbp-4]    ; 1: 메모리 로드
    add eax, 1          ; 2: eax += 1  ← 1번 결과가 필요!
    mov [rbp-4], eax    ; 3: 결과 저장 ← 2번 결과가 필요!

  2번 명령어가 Execute 단계에 들어갔을 때
  1번 명령어는 아직 Memory/Writeback 단계 중
  → eax 값이 아직 준비 안 됨
  → 2번은 기다려야 함 → 파이프라인 스톨(stall) = 버블(bubble)

해저드를 이해하면:
  왜 특정 코드가 느린지 어셈블리 수준에서 설명 가능
  컴파일러의 명령어 재배치 의도를 읽을 수 있음
  OoO CPU가 해저드를 우회하는 원리(Ch1-04)의 기초
```

---

## 😱 잘못된 이해

### Before: "의존성 있는 명령어는 그냥 기다린다"

```
잘못된 모델:
  add eax, ebx   ; 1번: eax = eax + ebx
  add ecx, eax   ; 2번: ecx = ecx + eax  ← eax 필요
  
  "2번이 eax를 쓰므로 1번이 Writeback 끝날 때까지
   2번은 Fetch도 못 한다"

현실 — 두 가지 해결책이 존재:

1. 포워딩(Forwarding/Bypass):
   1번이 Execute 완료하면 → Writeback 전에 2번의 Execute로 직접 전달
   Writeback까지 기다리지 않고 ALU 출력을 다음 ALU 입력으로 바로 연결
   → 스톨 없이 연속 실행 가능 (많은 경우)

2. 스케줄링:
   컴파일러가 의존성 없는 다른 명령어를 사이에 끼워넣어
   자연스럽게 기다리는 시간을 채움
   → 명시적 NOP 없이 파이프라인 유지

"무조건 기다린다"는 틀림.
"어떤 상황에서 무엇이 몇 사이클 기다리는가"가 핵심
```

---

## ✨ 올바른 이해

### After: 세 종류의 해저드와 각각의 해결 메커니즘

```
해저드(Hazard) 분류:

┌───────────────────────────────────────────────────────────────┐
│  1. 데이터 해저드 (Data Hazard)                                 │
│     명령어 간 데이터 의존성이 파이프라인 단계와 충돌             │
│     종류: RAW (진짜 의존성) / WAR / WAW (이름 의존성)           │
│     해결: 포워딩(RAW 대부분) / 레지스터 이름 변경(WAR, WAW)     │
│                                                                 │
│  2. 제어 해저드 (Control Hazard)                                │
│     분기 명령어 처리 중 다음 PC를 알 수 없음                    │
│     결과: 파이프라인에 버블 삽입 또는 플러시                    │
│     해결: 분기 예측 (Ch1-05에서 상세)                           │
│                                                                 │
│  3. 구조 해저드 (Structural Hazard)                             │
│     두 명령어가 같은 하드웨어 자원을 같은 사이클에 요구          │
│     해결: 자원 복제 (현대 CPU: 대부분 미리 해결됨)              │
└───────────────────────────────────────────────────────────────┘
```

---

## 🔬 내부 동작 원리

### 1. 데이터 해저드: RAW/WAR/WAW

```
RAW (Read-After-Write): 진짜 의존성 (True Dependency)
  
  mov eax, [rbp-8]   ; 1: eax ← 메모리 로드
  add ecx, eax       ; 2: ecx += eax  ← 1번의 eax 필요
  
  타이밍 문제:
  사이클:    1    2    3    4    5    6
  명령1(mov): [F] [D] [E] [MEM][WB]
  명령2(add):      [F] [D] [??] [E] ...
                        ↑
                   Decode: eax 값 아직 없음 (MEM 단계 중)
  
  포워딩 없이: 명령2가 명령1의 WB까지 기다림 → 2 사이클 스톨
  포워딩 있음: 명령1의 MEM 완료 → 명령2의 E로 직접 전달 → 1 사이클 스톨
  (로드-사용 해저드: 메모리 로드는 1 사이클 스톨이 불가피)

WAR (Write-After-Read): 이름 의존성 (Anti-Dependency)
  
  add eax, ebx       ; 1: eax += ebx  (eax 읽기)
  mov eax, ecx       ; 2: eax = ecx   (eax 쓰기)
  
  1번이 Read 전에 2번이 Write하면 안 됨
  in-order 파이프라인에서는 발생 안 함 (항상 1번이 먼저)
  OoO CPU에서 발생 → 레지스터 이름 변경으로 해결 (Ch1-04)

WAW (Write-After-Write): 이름 의존성 (Output Dependency)
  
  mov eax, 1         ; 1: eax = 1
  mov eax, ecx       ; 2: eax = ecx
  
  2번의 결과가 최종이어야 함
  1번의 Writeback이 2번 Writeback보다 늦으면 안 됨
  OoO CPU에서 발생 → 레지스터 이름 변경으로 해결 (Ch1-04)
```

### 2. 포워딩(Forwarding/Bypass) 메커니즘

```
포워딩이란:
  ALU의 출력이 레지스터 파일에 기록(Writeback)되기 전에
  직접 다음 ALU의 입력으로 전달하는 하드웨어 경로

포워딩 경로:
  
  명령1 [Fetch][Decode][Execute] ─────────────→ [Memory][Writeback]
                                     ↓ 포워딩
  명령2 [Fetch][Decode][  ???  ] ────↑──[Execute][Memory][Writeback]
                                Forward 
                                경로로
                                즉시 전달

포워딩 성공 사례 (ALU-to-ALU):
  add eax, ebx    ; 1: Execute 완료 → eax 결과 포워딩 가능
  add ecx, eax    ; 2: 1번 Execute 완료 즉시 eax 수신 → 스톨 없음!

포워딩 실패 사례 (Load-Use Hazard):
  mov eax, [rbp-8]  ; 1: Memory 단계에서야 eax 값 확정
  add ecx, eax      ; 2: Execute 시 eax 필요 → Memory 단계 이전
  
  사이클:    1    2    3    4    5    6    7
  명령1(mov):[F]  [D]  [E] [MEM][WB]
  명령2(add):     [F]  [D]  ↓   [E] [MEM][WB]
                        스톨 1사이클 (1번 MEM 완료 기다림)
  
  → 로드-사용 해저드는 포워딩으로도 1 사이클 스톨이 남음
  → 컴파일러가 사이에 다른 명령 삽입해서 해결 (명령어 스케줄링)

포워딩 경로의 종류:
  EX/EX: ALU 출력 → 다음 ALU 입력 (0 사이클 오버헤드)
  MEM/EX: 메모리 로드 완료 → 다음 ALU 입력 (1 사이클 스톨, 로드-사용)
  EX/MEM: ALU 출력 → 스토어 데이터 (0 사이클)
```

### 3. 제어 해저드: 분기와 파이프라인 플러시

```
제어 해저드의 발생:
  
  cmp  eax, 0       ; 1: 비교
  je   .loop_end    ; 2: 제로이면 점프 ← 어디로 점프할지 모름
  add  ecx, 1       ; 3: Fetch됨 (점프 안 할 수도 있으니)
  mov  [rbp-4], ecx ; 4: Fetch됨
  ; .loop_end:

  명령 2번이 Execute 단계에 들어가야 점프 여부와 목적지 주소 확정
  그 사이 3번, 4번은 이미 Fetch/Decode 완료됨
  → 잘못된 명령어 → 파이프라인 플러시 (flush)

파이프라인 플러시 비용:
  플러시된 명령어 수 = 파이프라인 스테이지 수 - 분기 감지 스테이지
  일반적으로 3~5개 명령어 폐기
  현대 CPU (깊은 파이프라인): ~15 사이클 낭비

플러시 vs 예측:
  초기 CPU 전략: 분기 감지까지 파이프라인 멈춤 (또는 NOP으로 채움)
    → 분기마다 3 사이클 낭비 → 코드 20%가 분기라면 IPC 크게 저하
  
  현대 CPU 전략: 분기 예측기가 다음 주소 추측 후 투기적 실행
    → 예측 성공: 낭비 없음, 성능 유지
    → 예측 실패: 투기적으로 실행한 명령어 폐기 + 재시작 (~15 사이클)
    → Ch1-05에서 예측기 상세 메커니즘 설명

분기 없는 코드 (branchless):
  조건부 이동 명령 cmov가 분기 대신 사용됨
  (Ch4-04에서 상세)
  
  // 분기 있는 버전 → 제어 해저드 위험
  if (x > 0) result = x;
  else result = -x;
  
  // 어셈블리 (분기 없는 cmov 사용):
  neg   eax         ; eax = -x (임시)
  test  edi, edi    ; x > 0?
  cmovg eax, edi    ; x > 0이면 eax = x (조건부 이동, 분기 없음)
```

### 4. 구조 해저드: 자원 충돌

```
구조 해저드의 정의:
  두 명령어가 같은 하드웨어 유닛을 같은 사이클에 요구

고전 MIPS에서의 구조 해저드:
  메모리 유닛이 하나일 때:
    Fetch도 메모리 접근 (명령어 캐시)
    Load/Store도 메모리 접근 (데이터 캐시)
    → 같은 사이클에 Fetch + Load = 충돌!
  해결: 명령어 캐시(L1i)와 데이터 캐시(L1d) 분리
         → Harvard 아키텍처 개념

현대 CPU에서의 구조 해저드 최소화:
  분리 캐시: L1i(명령어)와 L1d(데이터) 별도
  중복 유닛: ALU 4개, 로드/스토어 유닛 2개 등
  실행 포트 분리:
    예: Intel Skylake 8개 실행 포트
      포트 0,1,5,6: ALU
      포트 2,3:     로드
      포트 4:       스토어
      포트 7:       스토어 주소
  → 대부분의 구조 해저드 제거됨

구조 해저드가 여전히 발생하는 경우:
  분기 예측기가 한 사이클에 예측할 수 있는 분기 수 초과
  로드/스토어 유닛이 모두 사용 중인 메모리 집약적 코드
  → perf stat의 stalled-cycles-backend로 측정 가능
```

### 5. 컴파일러의 명령어 스케줄링

```
명령어 스케줄링 = 컴파일러가 해저드를 회피하도록 명령어 순서를 변경

목표:
  로드-사용 해저드의 1 사이클 스톨을 독립적인 명령어로 채움
  의존성 체인을 끊어 파이프라인 활용률 높임

예시: 로드-사용 해저드 회피

// 최적화 전 (의존성 바로 발생)
int a = arr[0];
int b = a + 1;         // a 로드 직후 사용 → 1 사이클 스톨

// 컴파일러 스케줄링 후
mov eax, DWORD PTR [rdi]    ; arr[0] 로드 시작
mov ecx, DWORD PTR [rdi+4]  ; arr[1] 로드 (독립적, 스톨 채우기)
add eax, 1                  ; arr[0]+1 (로드 완료 후 포워딩으로 수신)
add ecx, 1                  ; arr[1]+1 (로드 완료 후)

→ 두 로드를 먼저 발행하고, 계산을 나중에 → 스톨 없음

컴파일러 스케줄링 확인 (godbolt):
  -O2 이상에서 스케줄링 활성화
  -O2 -masm=intel 으로 어셈블리 확인
  인접 로드 명령어가 USE 명령어보다 앞에 나오면 스케줄링 적용됨

CPU 자체 스케줄링 (OoO 실행):
  컴파일러가 못 한 부분을 OoO CPU가 런타임에 처리
  Reservation Station에서 실행 준비된 명령어 선택 (Ch1-04)
  → 컴파일러 + CPU 양쪽에서 이중으로 해저드 회피

NOP 삽입 (구식 방법):
  컴파일러가 스케줄링할 독립 명령어가 없으면
  명시적 NOP(아무것도 안 하는 명령어)을 삽입하여 스톨 시간 채움
  → 현대 OoO CPU에서는 불필요 (CPU가 알아서 처리)
  → 임베디드/VLIW(Very Long Instruction Word) 아키텍처에서 여전히 사용
```

### 6. 어셈블리에서 해저드 패턴 읽기

```asm
; 해저드 패턴 예시: godbolt에서 확인 가능

; 패턴 1: 로드-사용 해저드 (1 사이클 스톨 가능)
mov  eax, DWORD PTR [rdi]  ; 로드
add  eax, 1                ; 바로 사용 → 스톨 (포워딩으로 1 사이클만)

; 패턴 2: 컴파일러 스케줄링으로 채워진 예시
mov  eax, DWORD PTR [rdi]   ; 로드 A
mov  ecx, DWORD PTR [rsi]   ; 로드 B (A와 독립, 스톨 채우기)
add  eax, ecx               ; A+B (두 로드 모두 완료됨, 스톨 없음)

; 패턴 3: ALU-to-ALU 포워딩 (스톨 없음)
add  eax, ebx               ; 실행 완료 즉시 eax 포워딩 가능
add  ecx, eax               ; 포워딩 수신 → 스톨 없음

; 패턴 4: 제어 해저드 (분기 예측 미스 시 ~15 사이클)
cmp  eax, ebx
jne  .far_target             ; 예측 미스 → 플러시 + 재시작
add  ecx, 1                  ; 잘못 인출된 명령어 (플러시됨)
```

---

## 💻 실전 실험

### 실험 1: 로드-사용 해저드 vs 스케줄링 비교

```c
// hazard_test.c
#include <stdio.h>
#include <time.h>
#include <stdlib.h>

#define N 100000000L

// 버전 A: 로드 직후 사용 (해저드 유발)
long test_hazard(int *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n - 1; i++) {
        int a = arr[i];
        sum += a;         // 바로 사용 → 로드-사용 해저드
    }
    return sum;
}

// 버전 B: 독립 로드 먼저 (스케줄링)
long test_scheduled(int *arr, int n) {
    long sum = 0;
    for (int i = 0; i < n - 2; i++) {
        int a = arr[i];
        int b = arr[i+1];  // 다음 로드를 미리 발행
        sum += a;           // a는 이미 완료됨
        (void)b;
    }
    return sum;
}

int main(void) {
    int *arr = malloc(N * sizeof(int));
    for (int i = 0; i < N; i++) arr[i] = i & 0xFF;

    struct timespec t1, t2;

    clock_gettime(CLOCK_MONOTONIC, &t1);
    long r1 = test_hazard(arr, N);
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_hazard = ((t2.tv_sec-t1.tv_sec)*1e9+(t2.tv_nsec-t1.tv_nsec))/1e6;

    clock_gettime(CLOCK_MONOTONIC, &t1);
    long r2 = test_scheduled(arr, N);
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ms_sched = ((t2.tv_sec-t1.tv_sec)*1e9+(t2.tv_nsec-t1.tv_nsec))/1e6;

    printf("결과: %ld %ld\n", r1, r2);
    printf("해저드 버전:   %.1f ms\n", ms_hazard);
    printf("스케줄링 버전: %.1f ms\n", ms_sched);
    printf("OoO CPU는 런타임에도 재스케줄링하므로 차이가 작을 수 있음\n");

    free(arr);
    return 0;
}
```

```bash
gcc -O1 -o hazard_test hazard_test.c  # -O1: 일부 최적화, 스케줄링 제한
./hazard_test

# perf로 스톨 사이클 확인
perf stat -e cycles,instructions,stalled-cycles-backend ./hazard_test

# 어셈블리 확인 (로드-사용 패턴 찾기)
gcc -O1 -S -masm=intel hazard_test.c -o hazard_O1.s
cat hazard_O1.s | grep -A5 "test_hazard:"
```

### 실험 2: 구조 해저드 측정 (나눗셈 연산)

```c
// struct_hazard.c: 나눗셈 vs 곱셈 포트 경합
#include <stdio.h>
#include <time.h>

#define REPS 100000000L

int main(void) {
    struct timespec t1, t2;
    volatile long r;

    // 나눗셈 (포트 경합 심함, 20~90 사이클 레이턴시)
    clock_gettime(CLOCK_MONOTONIC, &t1);
    long a = 123456789;
    for (long i = 1; i <= REPS; i++) {
        a = a / 7;          // 7로 나눔 (정수 나눗셈)
        a += i;             // 의존성 유지
    }
    r = a;
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ns_div = ((t2.tv_sec-t1.tv_sec)*1e9+(t2.tv_nsec-t1.tv_nsec)) / REPS;

    // 곱셈 (1~3 사이클 레이턴시)
    clock_gettime(CLOCK_MONOTONIC, &t1);
    long b = 1;
    for (long i = 1; i <= REPS; i++) {
        b = b * 3;
        b -= i;
    }
    r = b;
    clock_gettime(CLOCK_MONOTONIC, &t2);
    double ns_mul = ((t2.tv_sec-t1.tv_sec)*1e9+(t2.tv_nsec-t1.tv_nsec)) / REPS;

    printf("나눗셈: %.2f ns/iter\n", ns_div);
    printf("곱셈:   %.2f ns/iter\n", ns_mul);
    printf("비율:   %.1fx\n", ns_div / ns_mul);
    printf("(r=%ld 최적화 방지)\n", r);

    // 예상: 나눗셈이 20~40배 느림
    // → 컴파일러가 나눗셈을 곱셈+shift로 바꾸는 이유
    // gcc -O2로 컴파일하면 a/7 → 상수 나눗셈 최적화 적용됨
    // 상수 제수 나눗셈: 곱셈 reciprocal 기법으로 변환
    return 0;
}
```

```bash
gcc -O1 -o struct_hazard struct_hazard.c  # O2면 나눗셈을 곱셈으로 최적화함
./struct_hazard

# 어셈블리 확인 (idiv 명령어 있는지)
gcc -O1 -S -masm=intel struct_hazard.c | grep -E "div|mul|imul"

# O2에서 어떻게 바뀌는지 확인 (나눗셈 → 곱셈 최적화)
gcc -O2 -S -masm=intel struct_hazard.c | grep -B2 -A5 "imul"
# imulq + sarq 패턴으로 a/7을 표현 (컴파일러의 구조 해저드 회피)
```

### 실험 3: perf로 파이프라인 스톨 측정

```bash
# 파이프라인 스톨의 종류를 perf로 구분
cat > stall_bench.c << 'EOF'
#include <stdlib.h>
#include <stdio.h>

// 의존성 체인 → 데이터 해저드 스톨 유발
long dep_chain(long n) {
    long x = 1;
    for (long i = 0; i < n; i++) {
        x = (x * 6364136223846793005LL) + 1442695040888963407LL;
    }
    return x;
}

int main(void) {
    printf("%ld\n", dep_chain(500000000L));
    return 0;
}
EOF

gcc -O2 -o stall_bench stall_bench.c

# 백엔드 스톨 (실행 유닛 대기) vs 프론트엔드 스톨 (Fetch/Decode 병목)
perf stat -e cycles,instructions \
          -e stalled-cycles-frontend,stalled-cycles-backend \
          ./stall_bench

# 예상:
# stalled-cycles-frontend: 적음 (의존성 체인이므로 프론트엔드는 잘 흐름)
# stalled-cycles-backend: 많음 (곱셈 레이턴시 3 사이클 대기)

# 추가: 토포다운 분석 (Toplev)
# toplev --core S0-C0 ./stall_bench  (Intel VTune 또는 pmu-tools 필요)
```

---

## 📊 성능 비교

```
해저드 종류별 비용 비교:

해저드 종류           발생 조건                 비용 (사이클)  해결 방법
─────────────────────────────────────────────────────────────────────────
ALU-ALU RAW          ADD 후 즉시 ADD              0            포워딩 (무료)
로드-사용 RAW        로드 후 즉시 사용            1            포워딩+1스톨
로드-사용 (L2)       L2 캐시 히트 로드 후 사용    12+          캐시 개선 필요
분기 제어 해저드     예측 성공                    0            예측기 성공
분기 제어 해저드     예측 실패 (플러시)           15~20        예측 개선, branchless
구조 해저드          나눗셈 유닛 경합             20~90        나눗셈 대신 곱셈

컴파일러 스케줄링 효과 (단순 루프 기준):
  -O0 (스케줄링 없음): 로드-사용 스톨 매 반복 발생 → IPC ≈ 1.0~1.5
  -O2 (스케줄링 있음): 스톨 제거 + 레지스터 활용 → IPC ≈ 2.5~4.0

명령어별 레이턴시 (Intel Skylake 기준):
  ADD, SUB, AND, OR, XOR, CMP  →  1 사이클 레이턴시
  IMUL (32비트 정수 곱셈)       →  3 사이클 레이턴시
  IDIV (32비트 정수 나눗셈)     →  26~42 사이클 레이턴시
  FADD, FMUL (부동소수점)       →  4~5 사이클 레이턴시
  FDIV (부동소수점 나눗셈)      →  14~16 사이클 레이턴시
  L1 캐시 로드                  →  4 사이클 레이턴시
  L2 캐시 로드                  →  12 사이클 레이턴시
  L3 캐시 로드                  →  40 사이클 레이턴시
  DRAM 로드                     →  ~200 사이클 레이턴시
```

---

## ⚖️ 트레이드오프

```
포워딩:
  ✅ 대부분의 RAW 해저드 무료 해결
  ✅ 복잡한 재스케줄링 없이도 성능 유지
  ❌ 로드-사용 해저드는 여전히 1 사이클 스톨
  ❌ L2/L3/DRAM 로드는 포워딩으로도 해결 불가 (레이턴시 자체가 큼)

컴파일러 명령어 스케줄링:
  ✅ 정적 분석으로 로드-사용 스톨 대부분 제거
  ✅ 독립 명령어를 사이에 끼워넣어 파이프라인 채움
  ❌ 런타임 의존성(포인터 주소, 루프 반복 수)은 정적 분석 한계
  ❌ 긴 의존성 체인은 스케줄링으로도 은닉 불가

OoO CPU의 런타임 재스케줄링 (Ch1-04):
  ✅ 컴파일러가 못 한 부분을 런타임에 처리
  ✅ 여러 독립 명령어가 있으면 순서 바꿔 실행
  ❌ 하드웨어 자원(ROB 크기, RS 크기) 한계 존재
  ❌ 긴 의존성 체인은 OoO도 해결 불가 (Ch1-06 ILP 한계)

NOP 삽입 (구식 방법):
  ✅ 단순 in-order CPU(임베디드)에서 명확한 해결책
  ❌ 코드 크기 증가, i-cache 낭비
  ❌ 현대 OoO CPU에서는 의미 없음 (CPU가 직접 스케줄링)
  ❌ 컴파일러 스케줄링이 훨씬 나은 대안

branchless (제어 해저드 회피):
  ✅ cmov 등 조건부 이동으로 분기 제거 → 플러시 없음
  ❌ 매우 잘 예측되는 분기(항상 참인 루프 조건 등)는 오히려 느려질 수 있음
  ❌ cmov는 양쪽 값을 항상 계산 → 계산 비용 증가 가능
  → Ch4-04에서 trade-off 상세 분석
```

---

## 📌 핵심 정리

```
파이프라인 해저드 핵심:

3종류 해저드:
  데이터 해저드: 명령어 간 데이터 의존성 (RAW/WAR/WAW)
  제어 해저드:  분기 명령어 처리 중 다음 PC 불확실
  구조 해저드:  동일 하드웨어 자원 동시 요구

RAW 해결:
  포워딩(EX→EX): ALU 출력을 즉시 다음 ALU로 전달 → 스톨 없음
  포워딩(MEM→EX): 로드 결과를 즉시 ALU로 → 1 사이클 스톨은 남음
  컴파일러 스케줄링: 독립 명령어를 사이에 삽입 → 스톨 채우기

제어 해저드 해결:
  분기 예측 + 투기 실행 (Ch1-05에서 상세)
  예측 실패 → 파이프라인 플러시 ~15 사이클 비용
  branchless/cmov로 분기 자체 제거 가능

구조 해저드:
  현대 CPU: 대부분 자원 복제로 사전 해결
  나눗셈(20~90 사이클) → 컴파일러가 곱셈으로 변환
  perf stat stalled-cycles-backend 로 측정

명령어 레이턴시가 핵심:
  ADD: 1 사이클, IMUL: 3 사이클, IDIV: 26~42 사이클
  L1 로드: 4 사이클, DRAM: ~200 사이클
  레이턴시 높은 연산 → 의존성 체인 → IPC 저하
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 어셈블리 시퀀스에서 데이터 해저드를 찾고, 포워딩으로 해결되는지 여부와 스톨 사이클 수를 답하라.

```asm
mov  eax, DWORD PTR [rdi]   ; ① arr[0] 로드
add  eax, 5                  ; ② eax += 5
mov  DWORD PTR [rdi], eax    ; ③ arr[0] 저장
mov  ecx, DWORD PTR [rdi+4] ; ④ arr[1] 로드
add  ecx, eax                ; ⑤ ecx += eax
```

<details>
<summary>해설 보기</summary>

**해저드 분석**:

1. **① → ②: 로드-사용 RAW 해저드**
   - ①이 Memory 단계 완료 후 eax 확정
   - ②는 Execute 단계에서 eax 필요
   - **포워딩(MEM→EX): 1 사이클 스톨** (로드-사용 해저드는 포워딩으로도 1 사이클 불가피)

2. **② → ③: ALU-to-Store RAW**
   - ②가 Execute 완료 → eax 포워딩 가능 (EX→MEM 경로)
   - **포워딩 성공: 스톨 없음**

3. **④ → ⑤: 로드-사용 RAW 해저드**
   - ④가 Memory 단계 완료 후 ecx 확정
   - ⑤는 Execute 단계에서 ecx 필요
   - **포워딩(MEM→EX): 1 사이클 스톨**

4. **② → ⑤: ALU-ALU RAW** (eax를 ②가 쓰고 ⑤가 읽음)
   - 명령어 사이에 ③, ④가 있으므로 ②가 이미 Writeback 완료됨
   - **스톨 없음**

**요약**: 총 2 사이클 스톨 (①→② 1 사이클, ④→⑤ 1 사이클)

컴파일러 스케줄링 개선안: ①과 ④ 로드를 앞에 모아서 발행하면 두 스톨을 하나로 줄이거나 제거 가능합니다.

</details>

---

**Q2.** 다음 C 코드를 `-O2`로 컴파일하면 `idiv` 명령어가 생성되지 않는다. 왜인가? godbolt에서 직접 확인하고 컴파일러가 대신 사용하는 명령어를 설명하라.

```c
long divide_by_7(long x) {
    return x / 7;
}
```

<details>
<summary>해설 보기</summary>

**이유: 상수 나눗셈 최적화 (Strength Reduction)**

컴파일러는 `x / 7`을 다음과 같은 수학적 동치로 변환합니다:
`x / 7 ≈ x × (1/7)` → `x × magic_number >> shift`

실제 `-O2` 어셈블리:
```asm
divide_by_7(long):
    mov    rax, rdi                         ; rax = x
    movabs rcx, -8480242477960039729        ; magic = 2^63/7 근사값
    imul   rcx                             ; rax:rdx = x * magic (128비트)
    add    rdx, rdi                         ; 보정
    sar    rdx, 2                           ; 오른쪽 시프트 (2비트)
    mov    rax, rdx
    sar    rax, 63                          ; 부호 처리
    sub    rdx, rax
    mov    rax, rdx
    ret
```

**핵심 포인트**:
- `idiv`(나눗셈): **26~42 사이클 레이턴시**
- `imul`(곱셈) + `sar`(시프트) 조합: **3~4 사이클 레이턴시**
- 약 10배 빠름!

이것이 "컴파일러가 나눗셈을 곱셈으로 바꾸는" 구조 해저드 회피 최적화입니다. 가변 제수(`x / y`, y가 변수)는 이 최적화가 불가능해 `idiv`가 생성됩니다.

</details>

---

**Q3.** 다음 두 함수는 동일한 동작을 하지만 해저드 프로파일이 다르다. 어떤 것이 더 높은 IPC를 가지며 이유는 무엇인가?

```c
// 버전 A: 의존성 체인
long chain(long *a, int n) {
    long x = 0;
    for (int i = 0; i < n; i++) x = x ^ a[i];  // 매 반복 x에 의존
    return x;
}

// 버전 B: 누산기 분할
long split(long *a, int n) {
    long x0 = 0, x1 = 0, x2 = 0, x3 = 0;
    for (int i = 0; i < n - 3; i += 4) {
        x0 ^= a[i];
        x1 ^= a[i+1];
        x2 ^= a[i+2];
        x3 ^= a[i+3];
    }
    return x0 ^ x1 ^ x2 ^ x3;
}
```

<details>
<summary>해설 보기</summary>

**버전 B(누산기 분할)가 훨씬 높은 IPC**를 가집니다.

**버전 A 분석**:
- `x = x ^ a[i]` → 매 반복 x에 RAW 의존성
- 현재 x 완료 전 다음 반복 시작 불가
- XOR 레이턴시 1 사이클이지만 의존성 체인으로 직렬화
- IPC ≈ 1.0 (루프당 실질 1 연산)

**버전 B 분석**:
- x0, x1, x2, x3 서로 독립적
- 4개 XOR을 동시에 발행 가능 (슈퍼스칼라 4-wide라면 1 사이클에 4개)
- 로드도 4개가 독립적 → 병렬 처리
- IPC ≈ 4.0 (이론상)

이것이 **Ch1-06에서 다루는 "누산기 분할(accumulator splitting)"**의 핵심 아이디어입니다. 의존성 체인을 끊어 ILP를 활용합니다.

실제 측정:
```bash
perf stat -e cycles,instructions ./chain_vs_split
```
버전 B가 3~4배 빠르며 IPC도 그만큼 높게 측정됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: CPU 파이프라인](./02-cpu-pipeline-stages.md)** | **[홈으로 🏠](../README.md)** | **[다음: 슈퍼스칼라와 OoO ➡️](./04-superscalar-out-of-order.md)**

</div>
