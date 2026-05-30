# perf 완전 활용 — 하드웨어 카운터로 진단하기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `perf stat` 출력에서 IPC·cache-misses·branch-misses를 같이 보는 이유는 무엇인가?
- IPC가 낮을 때 원인이 메모리 병목인지 분기 병목인지 어떻게 구분하는가?
- `perf record -g`로 수집한 호출 그래프를 FlameGraph로 시각화하는 정확한 순서는?
- PMU 이벤트를 직접 지정해야 하는 경우는 언제이고, `perf list`를 어떻게 활용하는가?
- `kernel.perf_event_paranoid` 설정이 무엇이고, 어떤 값이 필요한가?
- 컨테이너(Docker) 환경에서 perf를 사용할 때 반드시 알아야 할 제약은 무엇인가?
- perf와 cachegrind를 각각 언제 선택해야 하는가?

---

## 🔍 왜 이 개념이 중요한가

### "느리다는 건 알겠는데, 어디서 무엇 때문에 느린지 모른다"

```
전통적인 디버깅 방식의 한계:

1. "느린 것 같다" → printf로 구간 시간 출력
   → 문제: 측정 자체가 캐시를 오염, JIT/컴파일러 최적화가 측정 대상을 변형

2. gprof / sampling profiler
   → 문제: 100ns 이하 병목을 놓침, I/O 대기와 CPU 대기를 구분 못함

3. 코드 리뷰로 "이게 느릴 것 같다" 추측
   → 문제: 최적화 전에 측정하지 않으면 잘못된 곳을 최적화

perf의 접근:
  하드웨어 내 PMU(Performance Monitoring Unit)가
  CPU가 실제로 무엇을 하고 있는지 사이클 단위로 집계

  측정 대상:
    cycles           — 소비한 총 사이클
    instructions     — 완료된 명령 수 (이 둘의 비율 = IPC)
    cache-misses     — LLC(Last Level Cache) 미스 수
    branch-misses    — 분기 예측 실패 수
    + 수백 가지 PMU 이벤트

  이것이 왜 강력한가:
    코드 변경 없음 — 실행 중인 바이너리에 attach 가능
    오버헤드 최소 — 샘플링 방식, CPU 내부 카운터 사용
    라인 단위 귀속 — 어떤 함수/라인에서 발생했는지 추적
```

---

## 😱 잘못된 이해

### Before: "perf stat 숫자 하나만 보면 된다"

```
흔한 오해:

1. "cache-misses가 높으면 캐시 문제다"
   → 실제: cache-misses는 LLC 미스를 나타냄
     L1/L2 미스가 폭증해도 LLC에서 히트하면 cache-misses는 낮게 나옴
     정확한 진단: L1-dcache-load-misses 이벤트 필요

2. "IPC가 2.0이면 좋은 것이다"
   → 실제: IPC가 높아도 Stall 이 없다는 뜻이 아님
     SIMD 명령 1개가 스칼라 8개와 동일한 IPC를 보이면서 8배 느릴 수 있음
     명령 수 자체가 무의미할 때가 있음

3. "perf report를 보면 가장 위에 있는 함수가 문제다"
   → 실제: 함수 X가 자기 자신은 빠르지만 느린 함수 Y를 수백만 번 호출할 수 있음
     calling overhead 포함 여부, call graph 없이는 전체 그림 안 보임

4. "측정하면 성능이 그대로 나온다"
   → 실제: perf 없이 실행과 perf record로 실행은 약 3~10% 오버헤드 차이
     프로파일링이 측정 대상을 변형하지 않으면서 정보를 얻는 균형 필요
```

---

## ✨ 올바른 이해

### After: IPC·캐시·분기를 삼각형으로 묶어 읽는다

```
세 수치의 관계:

          IPC (Instructions Per Cycle)
               ↗           ↖
     낮은 원인           낮은 원인
  cache-misses        branch-misses
  (메모리 대기)          (파이프라인 플러시)

진단 공식:

케이스 1: IPC 낮음 + cache-misses 높음 + branch-misses 정상
  → 메모리 병목 (Ch2의 지역성 문제 또는 워킹셋 초과)
  → 처방: AoS→SoA, 루프 재구성, prefetch, 데이터 구조 변경

케이스 2: IPC 낮음 + cache-misses 정상 + branch-misses 높음
  → 분기 예측 병목 (Ch1-05의 BTB 오염 또는 데이터 의존 분기)
  → 처방: branchless (Ch4-04), 데이터 정렬로 예측 가능성 향상

케이스 3: IPC 낮음 + cache-misses 낮음 + branch-misses 낮음
  → 의존성 체인 (Ch1-06 ILP 한계) 또는 포트 경합
  → 처방: 누산기 분할, 루프 언롤링, ILP 개선

케이스 4: IPC 높음 + 처리량이 기대 이하
  → 알고리즘 복잡도 문제 (Ch7-04 최적화 우선순위 참조)
  → 처방: 알고리즘 교체가 먼저

이렇게 삼각형으로 묶어야 진단이 완성된다.
```

---

## 🔬 내부 동작 원리

### 1. PMU (Performance Monitoring Unit)의 구조

```
CPU 칩 내부의 PMU:

  ┌──────────────────────────────────────────────┐
  │ CPU Core                                     │
  │                                              │
  │  Pipeline ──→ PMU                            │
  │               ├── Fixed Counters (항상 사용 가능)
  │               │     instructions (INST_RETIRED)
  │               │     cycles (CPU_CLK_UNHALTED)
  │               │     ref-cycles (참조 클럭)
  │               │                              │
  │               └── Programmable Counters (4~8개)
  │                     L1-dcache-load-misses     │
  │                     branch-misses             │
  │                     LLC-load-misses           │
  │                     ... 수백 가지 선택 가능   │
  └──────────────────────────────────────────────┘

perf의 작동 방식:
  1. perf stat:
     카운터를 전체 실행 시간 동안 누적 → 요약 출력
     오버헤드: 거의 없음 (CPU 내부 카운터, 소프트웨어 개입 최소)

  2. perf record (샘플링):
     N번 이벤트마다 인터럽트 발생 (기본: 4000 cycles마다)
     그 순간의 PC(Program Counter) + 콜스택 저장 → perf.data
     오버헤드: ~3~10% (샘플링 빈도에 따라)

  3. perf annotate:
     어셈블리 명령어에 샘플 수 귀속
     어떤 명령에서 CPU 시간이 머무는지 확인
```

### 2. `perf stat` 출력 해석 가이드

```bash
# 전체 측정 명령어
perf stat -e cycles,instructions,cache-references,cache-misses,\
branches,branch-misses,L1-dcache-loads,L1-dcache-load-misses,\
dTLB-load-misses \
./target_program

# 예시 출력:
#  Performance counter stats for './target_program':
#
#   10,234,567,890   cycles                    #    3.21 GHz
#    4,089,123,456   instructions              #    0.40  insn per cycle ← IPC = 0.40 (매우 낮음!)
#      234,567,890   cache-references
#       89,012,345   cache-misses              #   37.95% of all cache refs ← 38% LLC 미스
#      567,890,123   branches
#        3,456,789   branch-misses             #    0.61% of all branches ← 분기는 정상
#    1,234,567,890   L1-dcache-loads
#      345,678,901   L1-dcache-load-misses     #   28.00% of all L1-dcache hits ← 높음!
#          234,567   dTLB-load-misses
#
#        3.186924625 seconds time elapsed

# 이 출력 읽는 법:
#   IPC = 0.40 → 매우 낮음 (이상적: 2.0+)
#   cache-misses 38% + L1-dcache-load-misses 28% → 메모리 병목 확정
#   branch-misses 0.61% → 분기는 문제 없음
#   dTLB-load-misses 낮음 → TLB는 문제 없음
#   결론: Ch2 지역성 문제. AoS→SoA 또는 루프 재구성 필요
```

### 3. `perf record -g` + FlameGraph 파이프라인

```bash
# 1단계: 호출 그래프 포함 녹화
#   -g: 콜스택 포함 (frame pointer 기반)
#   --call-graph=dwarf: DWARF 디버그 정보 기반 (더 정확, 더 무거움)
#   -F 99: 초당 99번 샘플 (1000이면 오버헤드 증가)

perf record -g -F 99 ./target_program
# 또는 DWARF 방식 (인라이닝, 테일콜 등도 정확히 추적)
perf record --call-graph=dwarf -F 99 ./target_program

# 2단계: perf report로 텍스트 확인
perf report --stdio | head -50

# 출력 예시:
# # Overhead  Command        Shared Object      Symbol
# # ........  .............  .................  ................................
# #
#     42.15%  target_program  target_program    [.] process_data
#     28.34%  target_program  target_program    [.] inner_loop
#     15.22%  target_program  libc.so.6         [.] malloc
#      8.11%  target_program  target_program    [.] compute_result

# 3단계: FlameGraph로 시각화
# Brendan Gregg의 FlameGraph 도구 설치
git clone https://github.com/brendangregg/FlameGraph /opt/FlameGraph

# perf.data → 접힌 스택 형식 변환
perf script | /opt/FlameGraph/stackcollapse-perf.pl > out.folded

# SVG FlameGraph 생성
/opt/FlameGraph/flamegraph.pl out.folded > flamegraph.svg

# 브라우저에서 열기 (인터랙티브: 클릭으로 확대/축소 가능)
xdg-open flamegraph.svg
```

### 4. PMU 이벤트 직접 지정 (`perf list` 활용)

```bash
# 사용 가능한 PMU 이벤트 목록 조회
perf list

# 출력 예시:
# List of pre-defined events (to be used in -e):
#
#   branch-instructions OR branches               [Hardware event]
#   branch-misses                                 [Hardware event]
#   cache-misses                                  [Hardware event]
#   cache-references                              [Hardware event]
#   cpu-cycles OR cycles                          [Hardware event]
#   instructions                                  [Hardware event]
#   ...
#   L1-dcache-load-misses                         [Hardware cache event]
#   L1-dcache-loads                               [Hardware cache event]
#   L1-dcache-stores                              [Hardware cache event]
#   L1-icache-load-misses                         [Hardware cache event]
#   LLC-load-misses                               [Hardware cache event]
#   LLC-loads                                     [Hardware cache event]
#   ...
# mem_load_retired.l1_miss                        [Kernel PMU event]
# mem_load_retired.l2_miss                        [Kernel PMU event]
# mem_load_retired.l3_miss                        [Kernel PMU event]
# offcore_response.demand_rfo.l3_miss.remote_hitm [Kernel PMU event]
#   ↑ False Sharing 진단에 핵심 (Ch3-02 참조)

# Intel 원시 PMU 이벤트 직접 지정 (Intel SDM에서 이벤트 코드 확인)
# 예: L2_RQSTS.MISS (event=0x24, umask=0x3f)
perf stat -e cpu/event=0x24,umask=0x3f,name=L2_miss/ ./target_program

# DRAM 대역폭 관련 이벤트
perf stat -e \
  uncore_imc/cas_count_read/,\
  uncore_imc/cas_count_write/ \
  ./target_program
# → 실제 DRAM 읽기/쓰기 횟수 (루프라인 모델의 메모리 트래픽 측정에 사용)
```

### 5. 권한 설정과 `kernel.perf_event_paranoid`

```bash
# 현재 설정 확인
cat /proc/sys/kernel/perf_event_paranoid
# 기본값: 2 또는 3 (Ubuntu 20.04+)

# 값의 의미:
#  -1: 모든 이벤트 허용 (최대 권한)
#   0: 커널·사용자 공간 모두 허용 (CPU 카운터 포함)
#   1: 커널 프로파일링 제한 (사용자 공간 이벤트는 OK)
#   2: 사용자 공간만, 커널 이벤트 차단 (기본)
#   3+: 완전 차단 (perf stat도 안 됨 — Ubuntu 22.04+ 기본)

# 임시 변경 (재부팅 시 초기화)
sudo sysctl -w kernel.perf_event_paranoid=1

# 영구 변경
echo 'kernel.perf_event_paranoid=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 권한 없이 사용 가능한 우회법
sudo perf stat ./target_program
# 또는 cap_sys_admin 캐퍼빌리티 부여
sudo setcap cap_sys_admin+ep ./perf_binary

# kptr_restrict: 커널 심볼 주소 노출 제어
cat /proc/sys/kernel/kptr_restrict
sudo sysctl -w kernel.kptr_restrict=0  # 커널 심볼 이름 표시 허용
```

---

## 💻 실전 실험

### 실험 1: IPC로 병목 유형 판별

```c
// bottleneck_demo.c — 세 가지 병목 유형을 의도적으로 만드는 코드
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define N (1 << 24)  // 16M 요소

// 시나리오 A: 메모리 병목 (랜덤 접근)
long memory_bound(int* arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        int idx = arr[rand() % n];  // 랜덤 접근 → cache miss 폭발
        sum += idx;
    }
    return sum;
}

// 시나리오 B: 연산 병목 (순차 접근, 연산 많음)
double compute_bound(double* arr, int n) {
    double sum = 0.0;
    for (int i = 0; i < n; i++) {
        sum += arr[i] * arr[i];  // 순차 접근, 연산 집중
    }
    return sum;
}

// 시나리오 C: 분기 병목 (예측 불가 분기)
long branch_bound(int* arr, int n) {
    long sum = 0;
    for (int i = 0; i < n; i++) {
        if (arr[i] > 128)  // 랜덤 데이터 → 50% 미스
            sum += arr[i];
    }
    return sum;
}
```

```bash
gcc -O2 -o bottleneck bottleneck_demo.c

# 각 시나리오별 측정
perf stat -e cycles,instructions,cache-misses,branch-misses ./bottleneck A
# 기대: IPC 낮음, cache-misses 높음, branch-misses 낮음

perf stat -e cycles,instructions,cache-misses,branch-misses ./bottleneck B
# 기대: IPC 높음 (연산 파이프라인 포화), cache-misses 낮음

perf stat -e cycles,instructions,cache-misses,branch-misses ./bottleneck C
# 기대: IPC 낮음, cache-misses 낮음, branch-misses 높음

# 결과 비교표 (예상치):
# 시나리오  IPC    cache-misses  branch-misses
# A (메모리) 0.3    35%           0.2%
# B (연산)   2.8    2%            0.1%
# C (분기)   0.6    3%            48%
```

### 실험 2: `perf record` + FlameGraph 전체 워크플로우

```bash
# 1. 컴파일 (디버그 심볼 포함, 최적화 유지)
gcc -O2 -g -fno-omit-frame-pointer -o myapp myapp.c

# -fno-omit-frame-pointer: 정확한 콜스택을 위해 필수
#   기본 -O2에서는 frame pointer를 레지스터로 재사용해 콜스택이 깨짐

# 2. 녹화 (5초 또는 프로그램 전체 실행)
perf record -g -F 99 ./myapp
# 또는 이미 실행 중인 프로세스에 attach
perf record -g -F 99 -p $(pgrep myapp)

# 3. 빠른 확인 (TUI)
perf report
# 화살표로 함수 선택 → Enter로 호출 그래프 펼치기
# 'a': 어노테이션 보기 (어셈블리 수준)

# 4. FlameGraph 생성
perf script | \
  /opt/FlameGraph/stackcollapse-perf.pl | \
  /opt/FlameGraph/flamegraph.pl \
    --title "My App CPU Profile" \
    --colors java \
  > profile.svg

# 5. 차이 FlameGraph (두 버전 비교)
# before.folded, after.folded 생성 후:
/opt/FlameGraph/difffolded.pl before.folded after.folded | \
  /opt/FlameGraph/flamegraph.pl --negate > diff.svg
# 빨강: 후에 더 느려진 부분, 파랑: 후에 빨라진 부분
```

### 실험 3: 컨테이너 환경에서 perf 사용

```bash
# Docker 컨테이너 내에서 perf 사용 시 주의사항

# ---- 잘못된 방법 ----
# 일반 컨테이너 실행 후 perf 시도
docker run -it ubuntu:24.04 bash
# 내부에서: perf stat ls
# 오류: "Permission denied" 또는 이벤트 카운터 미지원

# ---- 올바른 방법 1: --privileged ----
docker run --privileged -it ubuntu:24.04 bash
# --privileged: 거의 모든 권한 부여 (보안 주의!)
# 내부에서 perf stat 사용 가능

# ---- 올바른 방법 2: 필요 권한만 부여 ----
docker run \
  --cap-add SYS_ADMIN \
  --cap-add SYS_PTRACE \
  --security-opt seccomp=unconfined \
  -it ubuntu:24.04 bash
# SYS_ADMIN: perf_event_open 시스템 콜 허용
# SYS_PTRACE: 프로세스 tracing 허용
# seccomp=unconfined: perf 관련 syscall 차단 해제

# ---- 올바른 방법 3: 호스트에서 프로파일 (권장) ----
# 컨테이너 PID를 호스트에서 찾아 attach
docker run -d --name mycontainer myapp
docker inspect mycontainer | grep '"Pid"'
# → 호스트 PID 12345
sudo perf record -g -p 12345  # 호스트에서 컨테이너 프로세스 프로파일

# perf.data가 컨테이너 내부 바이너리를 참조하므로:
# 심볼 경로가 다를 수 있음 → --symfs 옵션으로 컨테이너 루트 지정
sudo perf report --symfs /proc/12345/root

# ---- 컨테이너 내 perf 사용 핵심 제약 ----
# 1. PMU 이벤트: 컨테이너가 호스트 커널 PMU를 공유 → 다른 컨테이너의 이벤트와 혼재
# 2. 커널 심볼: 컨테이너 내부에서 /proc/kallsyms 없음
# 3. 하드웨어 카운터 멀티플렉싱: 컨테이너가 많을수록 카운터 공유로 정확도 저하
# 4. cgroup 경계: --pid=host 또는 호스트 PID 직접 지정 필요
```

---

## 📊 성능 비교

```
perf stat 출력 패턴으로 보는 병목 유형 요약:

시나리오          IPC    cache-miss%  branch-miss%  처방
───────────────────────────────────────────────────────────────────
메모리 병목        <0.5   >20%         <2%           Ch2 지역성 최적화
  (열 우선 순회, 랜덤 접근, 포인터 추적)               AoS→SoA, 캐시 블로킹

분기 예측 병목     <1.0   <5%          >10%          Ch4-04 branchless
  (랜덤 데이터, 불예측성 높은 조건)                    데이터 정렬로 예측성 향상

의존성 체인 병목   <1.0   <5%          <2%           Ch1-06 누산기 분할
  (직렬 연쇄, ILP 차단)                               루프 언롤 + 독립 누산기

이상적 실행        >2.5   <3%          <1%           병목 없음
  (순차 접근, 예측 가능, 독립 명령)

False Sharing      N/A    매우 높음     낮음          Ch3-02 64B 패딩
  (멀티스레드, HITM 이벤트 폭발)                      @Contended, 로컬 누산

TLB 병목           <0.5   낮음         낮음          Ch2-06 Huge Page
  (dTLB-load-misses 높음)                             4KB→2MB 페이지

───────────────────────────────────────────────────────────────────

perf 도구 선택 가이드:

도구              용도                                  오버헤드
───────────────────────────────────────────────────────────────────
perf stat         전체 실행의 카운터 요약               거의 없음
perf record -g    핫스팟 함수 + 호출 그래프             ~5%
perf annotate     어셈블리 수준 핫스팟 라인             ~5%
perf c2c          False Sharing 진단 (HITM)             ~10%
cachegrind        정확한 미스 수 (시뮬레이션)            ~20~50x 느림
```

---

## ⚖️ 트레이드오프

```
perf vs 다른 프로파일링 도구 비교:

perf (Linux PMU 기반):
  ✅ 실제 하드웨어 카운터 → 현실적 측정
  ✅ 거의 모든 프로그램에 적용 (attach 방식)
  ✅ 커널 공간 이벤트 포함
  ✅ FlameGraph 등 시각화 도구와 연동 용이
  ❌ Linux 전용 (macOS: Instruments, Windows: VTune 사용)
  ❌ 컨테이너/권한 설정 번거로움
  ❌ 커널 버전, CPU 종류마다 지원 이벤트 다름

cachegrind (Valgrind):
  ✅ 정확한 캐시 시뮬레이션 (100% 재현 가능)
  ✅ 권한 불필요 (사용자 공간만)
  ✅ 소스 라인 단위 귀속이 정확
  ❌ 20~50배 느림 → 실제 캐시 동작과 다를 수 있음 (열 효과 없음)
  ❌ SMT·멀티코어 상호작용 미반영
  ❌ PMU 카운터가 아닌 시뮬레이션

Intel VTune:
  ✅ Roofline 모델, 메모리 대역폭, 벡터화 분석 통합
  ✅ GUI 강력
  ❌ Intel CPU 최적화 (AMD에서 일부 기능 제한)
  ❌ 설치 무거움

async-profiler (JVM):
  ✅ JIT 컴파일 코드 포함 정확한 스택
  ✅ JVM 내부 이벤트 (GC, 락 경합)와 통합
  ❌ JVM 전용

선택 기준:
  빠른 병목 확인 → perf stat
  핫스팟 함수 → perf record + FlameGraph
  정확한 캐시 미스 라인 → cachegrind
  False Sharing → perf c2c
  메모리 대역폭/루프라인 → likwid-perfctr (Ch7-03 참조)
```

---

## 📌 핵심 정리

```
perf 완전 활용 핵심:

세 수치를 묶어 읽어라:
  IPC + cache-misses + branch-misses
  세 개를 같이 봐야 병목 유형이 특정됨

진단 공식:
  IPC 낮음 + 캐시 높음 → 메모리 병목 (Ch2 지역성)
  IPC 낮음 + 분기 높음 → 분기 예측 병목 (Ch1-05)
  IPC 낮음 + 둘 다 낮음 → 의존성 체인 (Ch1-06)

FlameGraph 파이프라인:
  perf record -g -fno-omit-frame-pointer
  → perf script → stackcollapse-perf.pl → flamegraph.pl

권한 설정:
  kernel.perf_event_paranoid=1 이면 대부분 측정 가능
  컨테이너: --cap-add SYS_ADMIN 또는 호스트에서 직접 프로파일

PMU 이벤트 직접 지정:
  perf list 로 사용 가능 이벤트 확인
  L1-dcache-load-misses, dTLB-load-misses, mem_load_retired.l3_miss 등

핵심 원칙:
  측정 없이 최적화하지 말 것
  perf stat으로 5분 진단이 며칠간의 잘못된 최적화를 막는다
```

---

## 🤔 생각해볼 문제

**Q1.** `perf stat`으로 어떤 프로그램을 측정했더니 다음 결과가 나왔다. 이 프로그램의 주요 병목은 무엇이고, 어떤 최적화를 먼저 시도해야 하는가?

```
4,567,890,123   cycles
1,234,567,890   instructions    #    0.27  insn per cycle
   89,012,345   cache-references
   85,234,567   cache-misses    #   95.75% of all cache refs
      234,567   branches
        2,345   branch-misses   #    1.00% of all branches
```

<details>
<summary>해설 보기</summary>

**주요 병목: 메모리 병목 (Cache-Miss 96%).**

분석:
- IPC = 0.27로 매우 낮습니다. CPU의 절반 이상이 메모리 대기 상태입니다.
- cache-misses가 **95.75%** — 캐시 참조의 거의 전부가 LLC 미스입니다. DRAM 직접 접근이 압도적입니다.
- branch-misses = 1% — 분기는 정상 범위입니다.

원인 추정:
1. 워킹셋이 L3 캐시 크기를 훨씬 초과하거나
2. 랜덤 접근 패턴(포인터 추적, 연결 리스트)이거나
3. 열 우선 순회처럼 캐시 비친화적 접근 패턴

최적화 순서:
1. `perf record -g`로 cache-miss가 발생하는 함수 특정
2. 해당 함수의 데이터 접근 패턴 확인 (Ch2-03 지역성)
3. 연결 리스트 → 배열, AoS → SoA (Ch2-05) 전환 검토
4. 워킹셋 크기 확인 후 캐시 블로킹 적용

이런 수치라면 알고리즘이 올바르더라도 데이터 레이아웃 변경만으로 5~10배 향상을 기대할 수 있습니다.

</details>

---

**Q2.** `-fno-omit-frame-pointer`를 `-O2`와 함께 지정하면 성능에 어떤 영향을 미치는가? 이 플래그 없이 perf record를 하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

**`-fno-omit-frame-pointer`의 효과:**

`-O2`는 기본적으로 RBP(frame pointer) 레지스터를 범용 레지스터로 재사용하여 성능을 높입니다. 이렇게 하면 RBP가 없어 스택 프레임을 역추적하기 어렵습니다.

`-fno-omit-frame-pointer`를 추가하면:
- RBP 레지스터를 항상 프레임 포인터로 예약 → 범용 레지스터 1개 줄어듦
- 성능 영향: 일반적으로 **1~5%** 느려짐 (레지스터 1개 손실)
- 코드 크기는 약간 증가 (함수 진입/탈출 시 `push/pop rbp` 추가)

**이 플래그 없이 `perf record -g` 시 문제:**

스택 언와인딩(stack unwinding)이 실패하여:
1. 호출 그래프가 깨짐 — "unknown" 또는 단순화된 스택만 표시
2. FlameGraph에서 각 함수가 독립 프레임으로 표시되어 호출 관계를 파악 불가

**대안:**
- `--call-graph=dwarf`: DWARF 디버그 정보로 스택 복원. 더 정확하지만 perf.data 크기가 크게 증가
- `--call-graph=lbr`: Intel LBR(Last Branch Record) 사용. 하드웨어가 최근 32~64개 분기를 기록. frame pointer 불필요지만 깊은 스택은 잘릴 수 있음

프로덕션 프로파일링에는 `-fno-omit-frame-pointer`로 빌드하는 것이 권장됩니다. 성능 손실이 1~5%에 불과하고 프로파일링 정확도가 크게 향상됩니다.

</details>

---

**Q3.** 같은 코드를 perf로 두 번 측정했는데 IPC가 0.4와 1.8로 크게 다르게 나왔다. 측정 오류인가, 아니면 정상인가? 어떤 요인이 같은 코드에서 IPC 차이를 만드는가?

<details>
<summary>해설 보기</summary>

**정상일 가능성이 높습니다.** 같은 코드라도 다음 요인들이 IPC를 크게 변화시킵니다:

1. **워밍업 상태**: 첫 번째 실행은 콜드 캐시로 시작해 cache-miss가 폭발하여 IPC가 낮습니다. 두 번째 실행은 데이터가 L3에 남아있어 IPC가 높아집니다. 이것이 Ch7-02 마이크로벤치마킹의 워밍업 문제입니다.

2. **NUMA 노드**: 첫 실행에서 메모리가 원격 NUMA 노드에 할당됐다면 DRAM 지연이 1.5~3배 증가하여 IPC가 낮아집니다 (Ch6-02).

3. **주파수 스케일링(DVFS)**: Turbo Boost가 켜진 상태에서 열이 없는 초기에는 높은 주파수로 실행됩니다. 두 번째 실행 시 CPU가 이미 뜨거우면 주파수가 낮아집니다. `perf stat`의 GHz 숫자를 확인하면 파악 가능합니다.

4. **SMT 경쟁**: 같은 물리 코어의 Hyper-Thread 파트너가 활성화되면 실행 포트를 공유하여 IPC가 낮아집니다.

**올바른 측정을 위한 대책:**
- `taskset -c 0 perf stat`으로 코어를 고정합니다 (Ch6-05 참조).
- `cpupower frequency-set -g performance`로 주파수를 고정합니다.
- 여러 번 반복 실행 후 중앙값을 사용합니다 (Ch7-02 마이크로벤치마킹 함정).
- 워밍업 실행 후 측정합니다.

</details>

---

<div align="center">

**[⬅️ 이전 챕터: 코어 친화도](../multicore-and-numa/05-thread-affinity.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이크로벤치마킹의 함정 ➡️](./02-microbenchmarking-pitfalls.md)**

</div>
