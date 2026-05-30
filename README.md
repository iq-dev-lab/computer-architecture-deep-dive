<div align="center">

# 🪨 Computer Architecture Deep Dive

**"코드가 동작한다 vs 이 코드가 왜 이 하드웨어에서 이 속도인지 안다"**

<br/>

> *"같은 O(N²) 코드인데 한쪽이 10배 빠른 이유를 설명 못한다면, `volatile`이 왜 필요한지 모른다면, 프로파일러가 가리키는 'cache-miss'가 무슨 의미인지 모른다면 — CPU를 블랙박스로 두고 있는 것이다"*

캐시라인이 만드는 10배 격차부터 분기 예측이 정렬 하나로 코드를 빠르게 만드는 원리, False Sharing이 멀쩡한 자료구조를 무너뜨리는 메커니즘, 메모리 배리어가 하드웨어 위에서 실제로 무엇을 막는지까지
**왜 이 명령어가 이 사이클에 끝나는가** 라는 질문으로 CPU·캐시·메모리 계층을 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![CPU](https://img.shields.io/badge/CPU-x86--64-0071C5?style=flat-square&logo=intel&logoColor=white)](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
[![perf](https://img.shields.io/badge/perf-Linux_Tools-FCC624?style=flat-square&logo=linux&logoColor=black)](https://perf.wiki.kernel.org/index.php/Main_Page)
[![godbolt](https://img.shields.io/badge/Compiler-Explorer-67C52A?style=flat-square&logo=llvm&logoColor=white)](https://godbolt.org/)
[![Docs](https://img.shields.io/badge/Docs-38개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

컴퓨터 구조에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"개념 설명"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "캐시 친화적으로 짜세요" | 캐시라인 64B가 행 우선/열 우선 순회의 cache-miss를 어떻게 가르는지, `perf stat`으로 같은 알고리즘의 사이클·미스율을 before/after로 측정 |
| "CPU는 명령을 파이프라인으로 처리합니다" | 5단계 파이프라인의 데이터/제어 해저드, 비순차 실행과 ROB(Reorder Buffer), 분기 미스 한 번이 파이프라인을 비우며 만드는 실제 사이클 비용 |
| "`volatile`을 쓰면 가시성이 보장됩니다" | Store Buffer가 만드는 재정렬, x86의 TSO와 ARM의 weak ordering 차이, `lock` 프리픽스가 캐시 일관성 프로토콜(MESI)을 어떻게 잡아채는지 |
| "false sharing을 피하세요" | 서로 다른 변수가 한 캐시라인에 있을 때 코어 간 무효화로 발생하는 캐시라인 핑퐁, `@Contended`/패딩 전후의 처리량 차이를 perf로 증명 |
| "분기 예측이 있어서 빨라요" | `if (data[i] >= 128)` 한 줄이 정렬된 입력에서 빠르고 랜덤 입력에서 느려지는 그 유명한 현상을 sort-then-filter로 직접 재현 |
| "SIMD로 벡터화하면 빠릅니다" | 컴파일러 자동 벡터화 조건, 분기가 벡터화를 깨는 이유, AVX 인트린식으로 내적을 4~8배 가속하고 godbolt에서 어셈블리 확인 |
| 이론 나열 | 실행 가능한 perf 명령어 + cachegrind 출력 + godbolt 어셈블리 + before/after 사이클 측정 + JVM·동시성 레포로 이어지는 연결 지점 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Pipeline](https://img.shields.io/badge/🔹_Execution-C_한_줄에서_기계어까지-0071C5?style=for-the-badge&logo=intel&logoColor=white)](./execution-and-pipeline/01-c-to-machine-code.md)
[![Cache](https://img.shields.io/badge/🔹_Cache-메모리_계층_지연-0071C5?style=for-the-badge&logo=intel&logoColor=white)](./memory-hierarchy-cache/01-memory-hierarchy-latency.md)
[![MESI](https://img.shields.io/badge/🔹_Concurrency-MESI_캐시_일관성-0071C5?style=for-the-badge&logo=intel&logoColor=white)](./memory-model-concurrency/01-mesi-cache-coherence.md)
[![Cost](https://img.shields.io/badge/🔹_Abstraction-함수_호출_비용-0071C5?style=for-the-badge&logo=intel&logoColor=white)](./abstraction-cost/01-function-call-cost.md)
[![SIMD](https://img.shields.io/badge/🔹_SIMD-벡터_레지스터_원리-0071C5?style=for-the-badge&logo=intel&logoColor=white)](./simd-data-parallelism/01-simd-basics.md)
[![NUMA](https://img.shields.io/badge/🔹_Multicore-토폴로지_확인-0071C5?style=for-the-badge&logo=intel&logoColor=white)](./multicore-and-numa/01-multicore-topology.md)
[![Perf](https://img.shields.io/badge/🔹_Measurement-perf_완전_활용-0071C5?style=for-the-badge&logo=intel&logoColor=white)](./measurement-and-optimization/01-perf-deep-dive.md)

<br/>

👉 **레포의 모든 도구를 단일 워크로드에 적용해 ~43배 가속을 사이클 단위로 증명하는 캡스톤:**
[![Capstone](https://img.shields.io/badge/⭐_Capstone-150ms_→_3.5ms_케이스_스터디-FF6B00?style=for-the-badge)](./measurement-and-optimization/05-case-study-end-to-end.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: 실행의 실체 — 명령어와 파이프라인

> **핵심 질문:** 내가 작성한 C 한 줄은 CPU에서 정확히 무엇이 되며, 왜 한 사이클에 두 개를 실행할 수도 있고 한 분기 미스 때문에 수십 사이클을 버리기도 하는가?

<details>
<summary><b>컴파일·기계어부터 분기 예측·ILP의 한계까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. C 한 줄이 기계어가 되기까지](./execution-and-pipeline/01-c-to-machine-code.md) | 전처리→컴파일→어셈블리→링크 단계가 godbolt에서 실제 어떻게 보이는지, x86-64 기본 명령어(MOV/ADD/CMP/Jcc) 해부, `objdump -d`로 ELF 바이너리에서 기계어를 역추적하는 방법, `-O0`과 `-O2`의 생성 어셈블리 비교 |
| [02. CPU 파이프라인 — Fetch/Decode/Execute/Memory/Writeback](./execution-and-pipeline/02-cpu-pipeline-stages.md) | 5단계 파이프라인이 IPC(Instructions Per Cycle)를 1 이상으로 끌어올리는 원리, 비파이프라인 vs 파이프라인의 처리량 비교, 사이클 단위로 단계가 겹치는 타이밍 다이어그램, 파이프라인 깊이의 트레이드오프 |
| [03. 파이프라인 해저드 — 데이터·제어·구조](./execution-and-pipeline/03-pipeline-hazards.md) | RAW/WAR/WAW 데이터 해저드, 분기로 인한 제어 해저드, 자원 충돌로 인한 구조 해저드, 포워딩(Forwarding/Bypass)으로 스톨을 제거하는 메커니즘, NOP 삽입과 컴파일러의 명령어 스케줄링 |
| [04. 슈퍼스칼라와 비순차 실행(OoO)](./execution-and-pipeline/04-superscalar-out-of-order.md) | 한 사이클에 여러 명령을 발행하는 슈퍼스칼라 구조, ROB(Reorder Buffer)와 Reservation Station이 의존성 그래프를 풀어내는 방식, 명령은 비순차 실행되지만 결과는 순차 커밋되는 이유, Tomasulo 알고리즘 개요 |
| [05. 분기 예측 — 파이프라인을 비우지 않으려는 투기 실행](./execution-and-pipeline/05-branch-prediction.md) | 2-bit 포화 카운터 예측기, BTB(Branch Target Buffer), Spectre와 연결되는 투기 실행의 본질, 분기 미스 한 번에 발생하는 파이프라인 플러시 비용(10~20 사이클), 정렬된 배열과 랜덤 배열의 분기 미스율 측정 |
| [06. 명령어 수준 병렬성(ILP)의 한계](./execution-and-pipeline/06-ilp-limits.md) | 의존성 사슬(dependency chain)이 ILP를 막는 원리, 누산기 분할(accumulator splitting)로 의존성을 끊는 기법, 왜 2010년대 이후 단일 스레드 클럭이 정체되고 코어 수로 방향이 바뀌었나, 암달의 법칙과의 연결 |

</details>

<br/>

### 🔹 Chapter 2: 메모리 계층 — 성능을 지배하는 캐시

> **핵심 질문:** 같은 알고리즘이 데이터 접근 패턴 하나로 10배 차이가 나는 이유는 무엇이고, 캐시라인·지역성·TLB는 어떻게 그 차이를 만드는가?

<details>
<summary><b>메모리 계층 지연부터 TLB·Huge Page까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 메모리 계층 — 레지스터에서 디스크까지의 지연](./memory-hierarchy-cache/01-memory-hierarchy-latency.md) | L1(~4cy) / L2(~12cy) / L3(~40cy) / DRAM(~200cy)의 사이클 단위 체감, "캐시 미스 한 번 = 명령 수십 개 실행 시간"이 의미하는 바, lat_mem_rd / Intel MLC로 실제 머신의 지연 측정, 계층 깊이가 비용 곡선을 만드는 이유 |
| [02. 캐시 동작 원리 — 라인·태그·집합 연관](./memory-hierarchy-cache/02-cache-internals.md) | 캐시라인(64B) 단위 동작, 주소를 태그·인덱스·오프셋으로 분해하는 매핑 구조, 직접 매핑 vs 집합 연관(set-associative) vs 완전 연관의 트레이드오프, LRU/PLRU 교체 정책의 실제 구현 |
| [03. 지역성(Locality) — 시간/공간이 성능을 만든다](./memory-hierarchy-cache/03-locality.md) | 시간 지역성(같은 주소 재참조) vs 공간 지역성(인접 주소 참조), `int[1024][1024]`의 행 우선 vs 열 우선 순회 cache-miss 비교, 연결 리스트가 배열 대비 느린 정확한 이유, 캐시라인 64B 안에 int 16개가 들어간다는 사실의 위력 |
| [04. 캐시 미스의 종류 — Compulsory·Capacity·Conflict](./memory-hierarchy-cache/04-cache-miss-types.md) | 3C 모델로 분류하는 미스의 원인, cachegrind로 각 미스 유형을 분리해 측정하는 방법, Conflict miss를 줄이는 배열 크기 회피 기법, 워킹셋이 L1/L2/L3 경계를 넘을 때 발생하는 성능 절벽(performance cliff) |
| [05. 데이터 구조와 캐시 — AoS vs SoA, 정렬, 패딩](./memory-hierarchy-cache/05-data-layout-cache.md) | Array-of-Structs vs Struct-of-Arrays가 캐시·벡터화에 미치는 영향, `alignas(64)`로 캐시라인에 정렬하는 이유, 구조체 멤버 순서가 sizeof를 바꾸는 패딩 규칙, Hot/Cold 필드 분리 전략 |
| [06. TLB와 가상 메모리 — 페이지 테이블 워크](./memory-hierarchy-cache/06-tlb-virtual-memory.md) | 4단계 페이지 테이블 워크(PML4→PDPT→PD→PT)의 실제 비용, TLB 미스가 L1 캐시 미스 이상으로 비싼 이유, 4KB vs 2MB Huge Page가 TLB 적중률에 미치는 영향, `perf stat -e dTLB-load-misses`로 측정 |

</details>

<br/>

### 🔹 Chapter 3: 메모리 모델과 동시성의 하드웨어

> **핵심 질문:** 코어 간 캐시는 어떻게 일관성을 유지하고, `volatile`·메모리 배리어·CAS는 하드웨어 어디에서 어떤 명령으로 번역되는가?

<details>
<summary><b>MESI 프로토콜부터 JMM 연결까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 멀티코어 캐시 일관성 — MESI 프로토콜](./memory-model-concurrency/01-mesi-cache-coherence.md) | Modified/Exclusive/Shared/Invalid 4 상태의 천이 다이어그램, 코어1이 쓸 때 코어2의 캐시라인이 Invalid로 떨어지는 과정, 버스 스누핑과 디렉토리 기반 일관성 비교, MESIF/MOESI 같은 확장 프로토콜 |
| [02. False Sharing — 같은 라인이 만드는 성능 붕괴](./memory-model-concurrency/02-false-sharing.md) | 두 변수가 의도와 무관하게 같은 64B 라인에 배치될 때 발생하는 캐시라인 핑퐁, perf의 cache-misses 폭증으로 진단, 64B 패딩과 Java `@Contended`로 해결, 카운터 배열 워크로드에서 처리량 5~10배 차이 재현 |
| [03. 메모리 재정렬 — 컴파일러와 CPU가 명령을 뒤바꾸는 이유](./memory-model-concurrency/03-memory-reordering.md) | Store Buffer가 만드는 StoreLoad 재정렬, x86의 TSO(Total Store Order)와 ARM의 약한 메모리 모델 차이, 컴파일러 재정렬(`-O2` aliasing) vs 하드웨어 재정렬, Dekker 알고리즘으로 재정렬을 재현하는 실험 |
| [04. 메모리 배리어 — Load/Store 배리어의 하드웨어 해석](./memory-model-concurrency/04-memory-barriers.md) | `mfence`/`lfence`/`sfence`가 실제로 무엇을 비우는지, `volatile`·`std::atomic`·Java `volatile`이 어떤 배리어로 번역되는지(godbolt로 확인), acquire/release semantics와 LoadLoad/StoreStore 배리어의 대응 |
| [05. 원자적 연산과 CAS — `lock` 프리픽스의 정체](./memory-model-concurrency/05-atomic-cas.md) | x86 `lock cmpxchg`가 캐시 일관성 프로토콜을 가로채는 방식, Compare-And-Swap의 하드웨어 구현, ABA 문제가 발생하는 시나리오와 태그된 포인터로 회피하기, 락프리 스택의 기본 구조 |
| [06. JMM과의 연결 — happens-before가 하드웨어로 내려가는 길](./memory-model-concurrency/06-jmm-hardware-bridge.md) | Java 메모리 모델의 happens-before 규칙이 x86/ARM 배리어로 매핑되는 표, `volatile` 읽기/쓰기가 JIT에서 생성하는 어셈블리 비교(JITWatch), `synchronized`가 모니터 진입/탈출 시 발생시키는 배리어, java-concurrency 레포로의 연결 |

</details>

<br/>

### 🔹 Chapter 4: 추상화의 숨은 비용

> **핵심 질문:** 함수 호출, 가상 함수, 포인터 추적이 만드는 비용을 사이클 단위로 환산하면 얼마이고, 인라이닝과 branchless는 그 비용을 어떻게 없애는가?

<details>
<summary><b>함수 호출 비용부터 측정으로 증명까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 함수 호출 비용 — 스택 프레임과 호출 규약](./abstraction-cost/01-function-call-cost.md) | System V AMD64 호출 규약(RDI/RSI/RDX/RCX/R8/R9)이 만드는 레지스터 절약, 스택 프레임 생성/해제 비용, `__attribute__((always_inline))`과 LTO가 호출 비용을 제거하는 메커니즘, 작은 함수의 인라이닝 전후 사이클 비교 |
| [02. 가상 함수·동적 디스패치 — vtable 추적의 진짜 비용](./abstraction-cost/02-virtual-functions.md) | vtable 포인터 역참조의 cache-miss + 간접 분기로 인한 분기 예측 실패, 모놀리식 다형성 호출이 BTB를 오염시키는 방식, devirtualization(Profile-Guided Optimization)이 가상 호출을 직접 호출로 바꾸는 조건, JIT의 인라인 캐싱과의 비교 |
| [03. 포인터 추적(Pointer Chasing) — 객체 그래프 순회의 비용](./abstraction-cost/03-pointer-chasing.md) | 연결 리스트 순회가 배열 순회보다 느린 본질적 이유(prefetch 불가), Linked List vs Array vs Arena 할당 시 메모리 지연 비교, ECS(Entity Component System) 패턴이 게임 엔진에서 채택되는 이유, OOP의 캐시 비용을 데이터 지향 설계로 우회하기 |
| [04. 분기 vs 분기 없는 코드 — branchless·cmov·SIMD 마스킹](./abstraction-cost/04-branchless-code.md) | 데이터 의존 분기가 분기 예측을 무력화하는 시나리오, `cmov`(조건 이동) 명령으로 분기를 제거하는 패턴, 절댓값/min/max의 branchless 구현, branchless가 항상 빠른 것은 아닌 이유(매우 잘 예측되는 분기 vs 무조건 실행) |
| [05. 측정으로 증명 — 같은 로직 두 구현의 사이클 비교](./abstraction-cost/05-proof-by-measurement.md) | 같은 알고리즘의 추상화 수준만 다른 두 구현을 perf로 비교하는 실험 설계, 데드코드 제거·상수 폴딩을 막는 `volatile`·`benchmark::DoNotOptimize` 사용법, IPC·cache-miss·branch-miss를 한 표로 정리해 추상화 비용을 사이클로 환산 |

</details>

<br/>

### 🔹 Chapter 5: SIMD와 데이터 병렬성

> **핵심 질문:** 한 명령으로 여러 데이터를 처리하는 SIMD는 언제 빛나고 언제 깨지며, 컴파일러는 어떤 조건에서만 자동으로 벡터화하는가?

<details>
<summary><b>SIMD 기본부터 처리량 vs 지연시간 관점까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. SIMD 기본 — SSE/AVX 레지스터와 한 명령 다중 데이터](./simd-data-parallelism/01-simd-basics.md) | XMM(128bit)/YMM(256bit)/ZMM(512bit) 레지스터 구조, 하나의 명령(`paddd`, `vmulps`)이 4·8·16개 요소를 동시에 처리하는 원리, SIMD 폭과 처리량 한계 계산, 부동소수점 SIMD가 정수 SIMD보다 늦게 등장한 이유 |
| [02. 자동 벡터화 — 컴파일러가 루프를 SIMD로 바꾸는 조건](./simd-data-parallelism/02-auto-vectorization.md) | `-O2`/`-O3`에서 GCC·Clang이 벡터화하는 루프의 조건(횟수 정해짐, aliasing 없음, 분기 없음), `__restrict__` 키워드와 aliasing 방지, `-fopt-info-vec-missed`로 벡터화 실패 이유 확인, godbolt에서 SIMD 어셈블리 생성 여부 검증 |
| [03. 수동 SIMD — 인트린식으로 합·내적 가속](./simd-data-parallelism/03-manual-simd-intrinsics.md) | `_mm256_loadu_ps` / `_mm256_fmadd_ps` 같은 AVX2/AVX-512 인트린식 사용법, 정렬되지 않은 로드의 비용, 수평 합(horizontal sum)으로 SIMD 결과를 스칼라로 환원하기, 내적 연산을 스칼라 대비 4~8배 가속하는 실험 |
| [04. 데이터 레이아웃과 벡터화 — SoA가 유리한 이유](./simd-data-parallelism/04-data-layout-for-simd.md) | AoS(Array of Structs)가 SIMD 로드를 막는 구조적 이유, SoA(Struct of Arrays)가 연속 메모리로 한 번에 8요소를 로드 가능하게 만드는 원리, 입자 시뮬레이션·픽셀 처리에서 SoA가 표준이 된 배경, 분기 한 줄이 벡터화를 깨는 메커니즘 |
| [05. 처리량 vs 지연시간 관점 — SIMD가 빛나는 워크로드](./simd-data-parallelism/05-throughput-vs-latency.md) | SIMD가 처리량을 늘리지 지연시간을 줄이지 않는다는 사실, 작은 데이터 단발 처리에는 SIMD가 무의미한 이유, 이미지 필터·행렬 곱셈·해시 같은 워크로드가 SIMD에 적합한 이유, GPU와의 비교 관점 |

</details>

<br/>

### 🔹 Chapter 6: 멀티코어와 NUMA

> **핵심 질문:** 코어를 늘릴수록 성능이 선형으로 증가하지 않는 이유는 무엇이고, 메모리 위치·락 경합·코어 친화도는 그 비선형성을 어떻게 만드는가?

<details>
<summary><b>멀티코어 토폴로지부터 코어 친화도까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 멀티코어 토폴로지 — 코어·L3·소켓의 실제 구조](./multicore-and-numa/01-multicore-topology.md) | 물리 코어 vs 논리 코어(SMT/Hyper-Threading), L1/L2 코어 전용 vs L3 공유의 의미, 소켓·다이·CCX(AMD) 단위 토폴로지, `lstopo --of console`과 `lscpu`로 실제 머신의 캐시·NUMA 구조 시각화 |
| [02. NUMA — 원격 메모리 접근 비용](./multicore-and-numa/02-numa-remote-access.md) | NUMA 노드 간 메모리 접근이 로컬 대비 1.5~3배 느린 이유(QPI/UPI/Infinity Fabric), `numactl --hardware`로 노드 거리 확인, `numactl --cpunodebind=0 --membind=0` vs `--membind=1`로 로컬/원격 접근 처리량 측정, first-touch 정책의 함정 |
| [03. 스케일링의 벽 — 암달·구스타프슨과 공유 자원 경합](./multicore-and-numa/03-scaling-wall.md) | 직렬 부분이 만드는 암달의 법칙 상한, 문제 크기가 커지면 회피되는 구스타프슨의 관점, 락·캐시라인·메모리 대역폭이 모두 공유 자원이라는 사실, 코어 수를 두 배로 늘려도 처리량이 1.3배만 오르는 워크로드의 진단 |
| [04. 락 경합의 하드웨어 비용 — 캐시라인 핑퐁](./multicore-and-numa/04-lock-contention-cost.md) | 뮤텍스 변수 자체가 캐시라인 핑퐁의 진원지가 되는 구조, 스핀락 vs 뮤텍스의 하드웨어 관점 비교(busy-wait의 캐시 점유), MCS 락·Ticket 락이 핑퐁을 줄이는 방식, `perf c2c`로 라인 단위 경합 위치 추적 |
| [05. 코어 친화도(Affinity) — 핀닝이 캐시 지역성을 지킨다](./multicore-and-numa/05-thread-affinity.md) | 스케줄러가 스레드를 옮길 때 발생하는 캐시·TLB warm-up 손실, `taskset`·`sched_setaffinity`·`pthread_setaffinity_np`로 코어 고정, 같은 L2/L3를 공유하는 코어에 협력 스레드를 묶는 전략, JVM `-XX:+UseG1GC`와 친화도의 상호작용 |

</details>

<br/>

### 🔹 Chapter 7: 측정과 최적화 방법론

> **핵심 질문:** 내 코드가 compute-bound인지 memory-bound인지 어떻게 판단하고, perf 출력을 어떤 순서로 읽어야 진짜 병목을 찾는가?

<details>
<summary><b>perf 완전 활용부터 케이스 스터디까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. perf 완전 활용 — 하드웨어 카운터로 진단하기](./measurement-and-optimization/01-perf-deep-dive.md) | `perf stat`의 IPC·cache-misses·branch-misses를 묶어 읽는 법, `perf record -g`로 호출 그래프 수집 후 `perf report`/FlameGraph로 시각화, PMU 이벤트 직접 지정(`perf list`), 권한·`paranoid` 설정과 컨테이너 환경에서 perf 사용 시 주의점 |
| [02. 마이크로벤치마킹의 함정](./measurement-and-optimization/02-microbenchmarking-pitfalls.md) | 데드코드 제거·상수 폴딩으로 측정이 0ns가 되는 사고, 워밍업 부재로 콜드 캐시·콜드 BTB·JIT 미컴파일 상태가 측정되는 문제, Google Benchmark / JMH의 표준 방어 패턴, 측정 분산을 통계적으로 처리하는 방법 |
| [03. 루프라인 모델 — compute-bound vs memory-bound](./measurement-and-optimization/03-roofline-model.md) | 연산 강도(FLOPs/byte) 축과 성능(GFLOPs) 축으로 그리는 루프라인, 내 코드가 메모리 대역폭 벽에 붙어 있는지 연산 한계에 붙어 있는지 판단, `likwid-bench`로 머신의 메모리 대역폭 측정, Intel Advisor / `likwid-perfctr`로 실제 강도 측정 |
| [04. 최적화 우선순위 — 알고리즘부터 시작해야 하는 이유](./measurement-and-optimization/04-optimization-priority.md) | 알고리즘 → 메모리 접근 패턴 → 벡터화 → 마이크로 최적화 순서가 효과의 크기 순서인 이유, 잘못된 알고리즘에 마이크로 최적화를 쏟는 함정, 80/20 원칙으로 최적화 대상을 좁히기, "충분히 빠른가"를 결정하는 SLA 기반 사고 |
| [05. 케이스 스터디 — 느린 코드 한 줄을 사이클까지 추적](./measurement-and-optimization/05-case-study-end-to-end.md) | 의도적으로 느린 데이터 처리 코드를 perf로 진단 → cache-miss 근원 추적 → AoS→SoA 전환 → branchless 적용 → SIMD 적용 단계별 before/after 사이클 비교, 어셈블리 diff로 어떤 명령이 줄었는지 확인, 최종 ~10배 가속을 정량적으로 증명 |

</details>

---

## 🔬 핵심 분석 대상 — 한 명령이 실행되기까지

이 레포의 모든 챕터는 아래 흐름을 완전히 이해하는 것을 목표로 합니다.

```
소스 코드 (C / C++ / Java JIT 산출물)
  │  컴파일 / JIT
  ▼
┌─────────────────────────────────────────────┐
│  기계어 (x86-64 ISA)                         │
│    └── godbolt / objdump로 가시화 (Ch1)      │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  CPU Frontend                               │
│    ① Fetch     명령어 캐시(L1i)에서 인출        │
│    ② Decode    µop으로 변환                   │
│    ③ Branch    예측기로 다음 PC 추측 (Ch1)     │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  CPU Backend (비순차 실행)                    │
│    ④ Rename    레지스터 이름 변경               │
│    ⑤ Dispatch  Reservation Station 큐잉      │
│    ⑥ Execute   ALU / FPU / SIMD 유닛 실행    │
│    ⑦ Commit    ROB에서 순차 커밋 (Ch1)         │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  메모리 계층                                  │
│                                             │
│  레지스터    (0 cycle)                       │
│      ↓                                      │
│  L1 캐시     (~4 cy)     코어 전용  (Ch2)     │
│      ↓                                      │
│  L2 캐시     (~12 cy)    코어 전용            │
│      ↓                                      │
│  L3 캐시     (~40 cy)    소켓 공유            │
│      ↓                                      │
│  DRAM        (~200 cy)   NUMA 노드 (Ch6)     │
│      ↓                                      │
│  Disk/SSD    (수만~수십만 cy)                  │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  멀티코어 동시성                              │
│    MESI 캐시 일관성 프로토콜       (Ch3)       │
│      ├── False Sharing → 핑퐁                │
│      ├── Store Buffer → 재정렬                │
│      └── lock cmpxchg → 원자적 갱신           │
└─────────────────────────────────────────────┘

성능을 가르는 5가지 사건:
  ① 분기 미스      → 파이프라인 플러시 (~15 cy 손실)
  ② L1 miss        → L2/L3/DRAM으로 폴백 (8~200 cy)
  ③ TLB miss       → 페이지 테이블 워크 (~수십 cy)
  ④ False Sharing  → 코어 간 캐시라인 핑퐁
  ⑤ NUMA 원격 접근 → 로컬 대비 1.5~3배 지연
```

---

## 🐳 측정 환경 (Dockerfile)

이 레포는 docker-compose가 아니라 **로컬 측정 도구가 핵심**입니다. 재현성을 위해 Dockerfile을 제공합니다.

```dockerfile
# Dockerfile — 측정 도구 일괄 설치 (Linux x86-64 권장)
FROM ubuntu:24.04

RUN apt-get update && apt-get install -y \
    gcc g++ clang \
    linux-tools-generic linux-tools-common \
    valgrind \
    likwid hwloc numactl \
    google-perftools libgoogle-perftools-dev \
    python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*

# perf는 호스트 커널 권한 필요:
#   docker run --privileged --pid=host ...
#   또는 호스트에서 직접 실행 권장
```

```bash
# 핵심 측정 명령어 세트

# IPC·캐시미스·분기미스 한눈에
perf stat -e cycles,instructions,cache-references,cache-misses,branches,branch-misses ./a.out

# 캐시 미스 상세 (라인 단위)
valgrind --tool=cachegrind ./a.out
cg_annotate cachegrind.out.<pid>

# 핫스팟 프로파일
perf record -g ./a.out && perf report

# 머신 토폴로지 / NUMA
lstopo --of console
numactl --hardware
numactl --cpunodebind=0 --membind=0 ./a.out      # 로컬
numactl --cpunodebind=0 --membind=1 ./a.out      # 원격 (비교용)

# 어셈블리 확인 (또는 godbolt.org)
gcc -O2 -S -masm=intel demo.c -o demo.s

# 코어 친화도 고정
taskset -c 0 ./a.out
```

각 문서에서 해당 시나리오에 필요한 명령어와 실험 순서를 안내합니다.

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 10섹션 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 중요한가** | 실무에서 마주치는 성능 문제와 이 개념의 연결 |
| 😱 **잘못된 이해 (Before)** | 추상적으로만 사고할 때 놓치는 지점 |
| ✨ **올바른 이해 (After)** | 하드웨어를 알고 코드를 설계할 때의 변화 |
| 🔬 **내부 동작 원리** | 어셈블리·캐시 동작·파이프라인 다이어그램 |
| 💻 **실전 실험** | perf·cachegrind·godbolt로 직접 측정 |
| 📊 **성능 비교** | before/after를 사이클·미스율 수치로 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "지금 코드가 왜 느린지 당장 알아야 한다" — 실전 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch7-01  perf 완전 활용 → IPC·cache-miss·branch-miss 읽는 법
Day 2  Ch2-03  지역성 → 행 우선 vs 열 우선 직접 측정
       Ch2-04  캐시 미스 종류 → cachegrind로 원인 분리
Day 3  Ch3-02  False Sharing 진단·해결
       Ch7-05  케이스 스터디 → 한 함수의 before/after 비교
```

</details>

<details>
<summary><b>🟡 "동시성 코드의 하드웨어 근원을 이해하고 싶다" — 핵심 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch2-01  메모리 계층 지연 체감
Day 2  Ch2-02  캐시 동작 원리 (라인·태그·집합 연관)
       Ch2-03  지역성과 캐시 미스
Day 3  Ch3-01  MESI 캐시 일관성 프로토콜
Day 4  Ch3-02  False Sharing 측정·해결
       Ch3-03  메모리 재정렬 (x86 TSO vs ARM)
Day 5  Ch3-04  메모리 배리어 (mfence/lfence/sfence)
       Ch3-05  CAS와 lock 프리픽스
Day 6  Ch3-06  JMM과의 연결 (java-concurrency로의 다리)
Day 7  Ch6-04  락 경합의 하드웨어 비용
```

</details>

<details>
<summary><b>🔴 "CPU·캐시·메모리를 사이클 단위로 이해하고 최적화까지" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — 실행의 실체
        → godbolt로 -O0/-O2 어셈블리 비교, 분기 예측 실험 재현

2주차  Chapter 2 전체 — 메모리 계층
        → 행/열 순회·AoS/SoA·Huge Page를 perf로 직접 측정

3주차  Chapter 3 전체 — 메모리 모델
        → False Sharing 5~10배 격차 재현, mfence 전후 어셈블리 diff

4주차  Chapter 4 전체 — 추상화의 비용
        → 가상 함수 vs 정적 디스패치, branchless 전환 사이클 비교

5주차  Chapter 5 전체 — SIMD
        → 자동 벡터화 성공/실패 케이스, AVX 인트린식으로 4~8배 가속

6주차  Chapter 6 전체 — 멀티코어와 NUMA
        → lstopo로 토폴로지 확인, NUMA 로컬/원격 처리량 측정

7주차  Chapter 7 전체 — 측정과 최적화 방법론
        → 루프라인 모델로 워크로드 분류, 케이스 스터디로 ~10배 가속
```

</details>

---

## 🔗 연관 학습 레포지토리

| 레포 | 관계 | 연결 지점 |
|------|------|-----------|
| [linux-for-backend-deep-dive](https://github.com/dev-book-lab/linux-for-backend-deep-dive) | 🤝 시너지 | OS 스케줄러가 코어 친화도(Ch6-05)를 어떻게 다루는지, 페이지 폴트와 TLB(Ch2-06)의 커널 측 상호작용 |
| [jvm-deep-dive](https://github.com/dev-book-lab/jvm-deep-dive) | 🤝 시너지 | JIT가 생성하는 기계어가 Ch1의 어셈블리와 연결되는 지점, escape analysis와 인라이닝(Ch4-01) |
| [java-concurrency-deep-dive](https://github.com/dev-book-lab/java-concurrency-deep-dive) | 🤝 시너지 | `volatile`·`synchronized`·`AtomicLong`이 Ch3의 메모리 배리어·CAS·MESI로 내려가는 다리 |
| [memory-management-compared](https://github.com/dev-book-lab/memory-management-compared) | 🧬 수렴 | GC/ARC/수동 관리의 모든 비용이 결국 Ch2 메모리 계층 위에서 측정된다는 사실 |
| [performance-testing-deep-dive](https://github.com/dev-book-lab/performance-testing-deep-dive) | 🧬 수렴 | USE 방법론의 "U(Utilization)"가 결국 CPU 자원·메모리 대역폭(Ch7-03 루프라인)으로 환원되는 지점 |

> 💡 이 레포는 **스택의 최하단**입니다. 선행 학습 레포는 없습니다. 단, C/어셈블리를 읽을 수 있으면 실험이 훨씬 풍부해집니다. JVM/동시성 레포에서 외운 규칙들이 여기서 "왜 그렇게 정의됐는지"의 답을 만나게 됩니다.

---

## 🙏 Reference

- *Computer Systems: A Programmer's Perspective* — Bryant & O'Hallaron
- *What Every Programmer Should Know About Memory* — Ulrich Drepper ([PDF](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf))
- *Computer Architecture: A Quantitative Approach* — Hennessy & Patterson
- *Performance Analysis and Tuning on Modern CPUs* — Denis Bakhvalov ([Book](https://book.easyperf.net/perf_book))
- [Agner Fog — 최적화 매뉴얼](https://www.agner.org/optimize/)
- [Intel 64 and IA-32 Architectures Optimization Reference Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Compiler Explorer (godbolt)](https://godbolt.org/)
- [Brendan Gregg — Linux Performance](https://www.brendangregg.com/linuxperf.html)
- [perf wiki — Tutorial](https://perf.wiki.kernel.org/index.php/Tutorial)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"코드를 짜는 것과, 그 코드가 CPU·캐시·메모리에서 실제로 무슨 일을 하는지 아는 것은 다르다"*

</div>
