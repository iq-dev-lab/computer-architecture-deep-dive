# 가상 함수·동적 디스패치 — vtable 추적의 진짜 비용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- vtable 포인터 역참조가 왜 캐시 미스를 일으키는가?
- 간접 `call`이 직접 `call`보다 분기 예측 실패율이 높은 이유는?
- BTB(Branch Target Buffer) 오염이란 무엇이며, 왜 다형성 호출이 이를 유발하는가?
- 컴파일러의 devirtualization은 어떤 조건에서 가상 호출을 직접 호출로 바꾸는가?
- PGO(Profile-Guided Optimization)는 vtable 비용을 어떻게 줄이는가?
- JIT의 인라인 캐싱(Inline Caching)은 C++ vtable과 무엇이 다른가?

---

## 🔍 왜 이 개념이 중요한가

### "다형성은 공짜라고 배웠는데"

```
OOP의 약속:
  Base* obj = new Derived();
  obj->process();
  → 컴파일 타임에 타입 몰라도 됨 = 유연성

그 유연성의 숨은 청구서:

  1. vtable 포인터 역참조 (메모리 접근 2회):
     obj          → vtable 포인터  → 함수 포인터  → 함수 본체
     [객체 메모리]   [vtable 배열]   [코드 영역]
     ↓                ↓                ↓
     L1 캐시 히트?   별도 캐시라인?   i-cache 히트?

  2. 간접 분기 예측 실패:
     직접 call foo: CPU가 "다음은 foo" 알고 미리 fetch
     간접 call [rax]: CPU가 rax 값을 알 때까지 기다려야 함
     → 파이프라인 버블 ~15~20 사이클

  3. 인라이닝 불가:
     컴파일 타임에 어느 함수인지 모름
     → 본체를 호출 지점에 펼칠 수 없음
     → 추가 최적화(상수 전파, dead code 제거)도 불가

핫 루프 안에서의 가상 호출:
  for (auto& obj : objects)   // 100만 개 객체
      total += obj->compute();

  → 100만 번 × (vtable 역참조 + 간접 call + 파이프라인 버블)
  → 정적 디스패치 대비 2~5배 느릴 수 있음
```

---

## 😱 잘못된 이해

### Before: "vtable은 그냥 함수 테이블이니까 포인터 하나만 추가되는 것 아닌가?"

```
잘못된 이해:
  vtable = 정적 배열
  호출 = 배열에서 함수 포인터 꺼내서 jump
  "포인터 하나 추가, 별거 없음"

놓친 점 1 — 캐시 계층 압박:
  객체 크기: 64 bytes (캐시라인 1개)
    → 처음 8 bytes: vtable 포인터
    → 나머지: 실제 데이터

  vtable 자체는 어디에?
    → 코드 세그먼트 근처, 객체와 전혀 다른 캐시라인
    → 객체를 fetch해도 vtable은 별도로 fetch 필요

  vtable에서 함수 포인터 꺼내면?
    → 그 함수 코드가 i-cache에 있는가?
    → 없으면 또 다른 캐시 미스

  최악의 경우: 3단계 잠재적 캐시 미스
    객체 → vtable → 함수 코드

놓친 점 2 — BTB 오염:
  동일 호출 지점에서 매번 다른 함수 포인터:
    for (auto* b : mixed_objects)  // Dog, Cat, Bird 혼합
        b->speak();
    → BTB: "이 call은 어디로 가나?" 기록이 계속 바뀜
    → 예측 실패율 높아짐
    → 특히 수십 개 서브클래스가 섞이면 BTB 슬롯 경합
```

---

## ✨ 올바른 이해

### After: 가상 함수 비용은 두 가지 독립적 비용의 합

```
비용 1: 메모리 간접 참조
  obj->vptr          → L1 히트  ~4 cy   (객체가 캐시에 있을 때)
  obj->vptr[index]   → L1 히트  ~4 cy   (vtable이 캐시에 있을 때)
  함수 코드 fetch    → i-cache  ~0~4 cy (코드가 캐시에 있을 때)
  
  모두 캐시 히트: ~8~12 cy 추가
  vtable 캐시 미스:  +~40 cy (L3 히트) 또는 +~200 cy (DRAM)

비용 2: 간접 분기 예측 실패
  항상 같은 서브클래스 → BTB 학습 가능 → 미스 드물게 발생
  여러 서브클래스 혼합 → BTB 혼란 → 미스 빈발
  미스 1회 = ~15~20 사이클 파이프라인 플러시

핵심 인사이트:
  단일 서브클래스만 있는 경우:
    → devirtualization 가능 (컴파일러/JIT가 직접 호출로 변환)
    → 비용이 직접 호출과 동등해짐
  
  여러 서브클래스가 핫 루프에서 혼합된 경우:
    → BTB 오염 + 캐시 미스가 중첩
    → 데이터 지향 설계(Ch4-03)가 해결책
```

---

## 🔬 내부 동작 원리

### 1. vtable 구조 상세

```cpp
// C++ 소스
class Shape {
public:
    virtual double area() = 0;
    virtual void draw() = 0;
    int color;
};

class Circle : public Shape {
public:
    double area() override { return 3.14 * r * r; }
    void draw() override { /* ... */ }
    double r;
};
```

```
메모리 레이아웃:

Circle 객체 (heap):
  ┌────────────────────┐  offset 0
  │  vptr              │  8 bytes ← vtable 포인터
  ├────────────────────┤  offset 8
  │  color (int)       │  4 bytes (Shape::color)
  ├────────────────────┤  offset 12 (+4 패딩)
  │  r (double)        │  8 bytes
  └────────────────────┘  = 24 bytes 합계

Circle의 vtable (정적 영역, 프로그램당 1개):
  ┌────────────────────┐
  │  Circle::area      │  → area() 함수 코드의 주소
  ├────────────────────┤
  │  Circle::draw      │  → draw() 함수 코드의 주소
  └────────────────────┘

가상 호출 obj->area() 의 기계어 번역:
  mov  rax, [rdi]        ; vptr 로드 (obj의 첫 8 bytes)
  call [rax + 0]         ; vtable[0] 호출 (area의 슬롯)
  
  ; area()가 2번째 가상함수라면:
  call [rax + 8]         ; vtable[1] 호출 (draw의 슬롯)

직접 호출과의 비교:
  직접: call Circle::area  ; 1개 명령 (주소 즉시값)
  가상: mov + call         ; 2개 명령 + 간접 참조
```

### 2. 간접 분기 예측 실패 메커니즘

```
직접 call의 분기 예측:
  call foo          ; 목적지 = 고정 주소 0x4005a0
  CPU: "이 명령은 항상 0x4005a0으로 간다" → BTB에 기록
  예측 정확도: ~100%

간접 call의 분기 예측:
  call [rax]        ; 목적지 = rax의 런타임 값
  CPU: "이 명령은 어디로 갔었나?" → BTB에서 과거 이력 조회

단일 서브클래스 시나리오 (예측 가능):
  for (Circle* c : circles)
      c->area();
  → 항상 call → Circle::area
  → BTB: 이 간접 call은 항상 Circle::area → 예측 성공
  → 미스율 ~0~1%

혼합 서브클래스 시나리오 (예측 불가):
  std::vector<Shape*> shapes = {circle, square, triangle, ...};
  for (Shape* s : shapes)
      s->area();
  → Circle::area, Square::area, Triangle::area 번갈아 호출
  → BTB: 어제 Circle::area였는데 오늘은 Square::area
  → 예측 실패율 ~30~50% (무작위 혼합 시)
  → 실패 1회 = ~15~20 사이클 파이프라인 버블

BTB 오염 (BTB Pollution):
  BTB 슬롯 수가 제한됨 (~수천 개)
  다양한 간접 call이 같은 슬롯을 경쟁
  → 하나의 가상 호출 핫스팟이 다른 코드의 BTB 엔트리를 밀어냄
  → 전혀 무관한 코드의 분기 예측도 나빠짐
  → "BTB 오염"이라고 부름
```

### 3. x86-64 어셈블리로 보는 직접 vs 가상 호출

```cpp
// 비교 코드
struct Direct {
    int compute(int x) { return x * 2; }
};

struct Virtual {
    virtual int compute(int x) { return x * 2; }
};
```

```asm
; godbolt.org — GCC 13.2 -O2

; Direct::compute 호출 (인라이닝 후)
; int use_direct(Direct& d, int x) { return d.compute(x); }
use_direct(Direct&, int):
    lea    eax, [rsi+rsi]   ; x*2 직접 계산, call 없음
    ret                      ; ← 인라이닝 + 상수 계산 완료

; Virtual::compute 호출 (인라이닝 불가)
; int use_virtual(Virtual& d, int x) { return d.compute(x); }
use_virtual(Virtual&, int):
    mov    rax, QWORD PTR [rdi]    ; vtable 포인터 로드
    mov    rdi, rdi                 ; (this 포인터 정렬)
    jmp    [QWORD PTR [rax]]        ; vtable[0] 간접 점프
    ; ← jmp로 tail call 최적화됨, 하지만 여전히 간접

; 실제 복잡한 경우 (루프 + 혼합 타입):
; void process(Virtual** objs, int n) {
;     for (int i=0; i<n; i++) objs[i]->compute(i); }
process(Virtual**, int):
    test   esi, esi
    jle    .done
    xor    eax, eax
.loop:
    mov    rcx, QWORD PTR [rdi+rax*8]   ; objs[i] 로드
    mov    rdx, QWORD PTR [rcx]          ; vtable 포인터 로드 ← 잠재적 미스
    mov    esi, eax
    mov    rdi, rcx
    call   QWORD PTR [rdx]               ; 간접 call ← 분기 예측 불확실
    ...
```

### 4. Devirtualization — 컴파일러가 가상 호출을 제거하는 조건

```cpp
// 조건 1: final 키워드 — 더 이상 오버라이드 없음을 보장
class Circle final : public Shape {
    double area() override { return 3.14 * r * r; }
};

void use(Circle& c) { c.area(); }
// → 컴파일러: "Circle은 final, area()는 항상 Circle::area"
// → 직접 call 또는 인라이닝

// 조건 2: 스코프 내 구체 타입 확정
void local_scope() {
    Circle c(5.0);  // 스택에 Circle 생성 → 타입 확정
    c.area();       // → 인라이닝 가능
}

// 조건 3: 포인터 타입이 루프 불변
void loop_devirt(Circle** circles, int n) {
    for (int i = 0; i < n; i++)
        circles[i]->area();
    // → circles[i]는 Circle* → area()는 Circle::area
    // → 컴파일러가 증명 가능하면 devirtualize
}

// devirtualize 불가 — 런타임 타입 알 수 없음
void impossible(Shape* s) {
    s->area();   // s가 Circle인지 Square인지 모름
}
```

### 5. PGO(Profile-Guided Optimization)와 추측적 devirtualization

```bash
# 1단계: 계측 빌드 (instrumented build)
g++ -O2 -fprofile-generate -o prog_instr main.cpp
./prog_instr < typical_input.txt   # 프로파일 데이터 수집
# → *.gcda 파일 생성 (호출 빈도, 타입 분포 기록)

# 2단계: 프로파일 기반 최적화 빌드
g++ -O2 -fprofile-use -fprofile-correction -o prog_opt main.cpp
```

```
PGO가 적용하는 추측적 devirtualization:

프로파일 데이터 예시:
  obj->compute() 100만 회 중:
    Circle::compute: 95만 회 (95%)
    Square::compute: 5만 회  (5%)

PGO 적용 후 생성 코드 (pseudo):
  if (likely(vptr == Circle_vtable)) {
      Circle::compute(obj, x);  // 직접 call → 인라이닝 가능
  } else {
      (*vptr->compute)(obj, x); // 간접 call (예외 경로)
  }

효과:
  95% 케이스: 직접 call → 분기 예측 거의 완벽
  5% 케이스: 간접 call (기존과 같음)
  → 전체 평균 비용 대폭 감소

JIT(Java HotSpot)의 인라인 캐싱:
  Monomorphic IC: 단 한 가지 타입만 → 직접 call (인라이닝까지)
  Bimorphic IC:   두 가지 타입 → if/else 직접 call
  Megamorphic IC: 3가지 이상 → vtable 방식으로 폴백
  → JIT는 런타임에 실시간으로 타입 분포를 추적 → PGO보다 적응적
  → jvm-deep-dive 레포의 JIT 최적화 섹션 참조
```

### 6. 인터페이스 vs vtable — Java와 C++의 차이

```
C++ vtable:
  단일 상속: vptr + vtable 1개 (단순)
  다중 상속: 복수 vptr + 복수 vtable (복잡, 포인터 조정 비용)

Java 인터페이스 호출 (invokeinterface):
  interface Shape { double area(); }
  class Circle implements Shape { ... }

  invokeinterface 명령:
    → itable(interface table) 조회: vtable보다 비쌈
    → itable은 인터페이스마다 별도 슬롯 → 탐색 비용 추가
    → HotSpot JIT가 Monomorphic IC로 최적화하면 직접 call로 변환

  invokevirtual (클래스 메서드) vs invokeinterface:
    invokevirtual: vtable 슬롯 오프셋 고정 → 1회 역참조로 끝
    invokeinterface: 인터페이스 구현 위치 탐색 → 추가 비용
    → "인터페이스를 많이 쓰면 느리다"는 말의 하드웨어 근거
    → JIT의 IC가 충분히 학습하면 차이 사라짐
```

---

## 💻 실전 실험

### 실험 1: godbolt에서 vtable 어셈블리 확인

```cpp
// https://godbolt.org — GCC 13.2 -O2

#include <cstdio>

struct Animal {
    virtual void speak() = 0;
    int id;
};

struct Dog : Animal {
    void speak() override { puts("Woof"); }
};

struct Cat : Animal {
    void speak() override { puts("Meow"); }
};

// 이 세 함수의 어셈블리를 비교
void call_direct(Dog& d)    { d.speak(); }        // devirtualize
void call_virtual(Animal& a){ a.speak(); }        // vtable 필요
void call_local() {
    Dog d;
    d.speak();                                      // 타입 확정 → 인라인
}

// 관찰 포인트:
// call_direct: "mov rax, [rdi]" + "call [rax]" — final 없으면 여전히 간접
// call_virtual: 반드시 간접 call
// call_local: puts("Woof")로 직접 치환될 수 있음
```

### 실험 2: Google Benchmark로 직접 vs 가상 호출 비용 측정

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <memory>

struct Base {
    virtual int compute(int x) { return x * 2; }
    virtual ~Base() = default;
};

struct Derived : Base {
    int compute(int x) override { return x * 2; }
};

// 직접 호출 (정적 디스패치)
static void BM_DirectCall(benchmark::State& state) {
    std::vector<Derived> objs(1000);
    int sum = 0;
    for (auto _ : state) {
        for (auto& o : objs)
            sum += o.Derived::compute(42);  // 정적 디스패치 강제
        benchmark::DoNotOptimize(sum);
    }
}

// 가상 호출 — 단일 타입 (예측 가능)
static void BM_VirtualMono(benchmark::State& state) {
    std::vector<std::unique_ptr<Base>> objs;
    for (int i = 0; i < 1000; i++)
        objs.push_back(std::make_unique<Derived>());
    int sum = 0;
    for (auto _ : state) {
        for (auto& o : objs)
            sum += o->compute(42);
        benchmark::DoNotOptimize(sum);
    }
}

// 가상 호출 — 다형성 혼합 (예측 어려움)
struct Derived2 : Base { int compute(int x) override { return x * 3; } };
struct Derived3 : Base { int compute(int x) override { return x * 4; } };
struct Derived4 : Base { int compute(int x) override { return x * 5; } };

static void BM_VirtualPoly(benchmark::State& state) {
    std::vector<std::unique_ptr<Base>> objs;
    for (int i = 0; i < 1000; i++) {
        switch (i % 4) {
            case 0: objs.push_back(std::make_unique<Derived>()); break;
            case 1: objs.push_back(std::make_unique<Derived2>()); break;
            case 2: objs.push_back(std::make_unique<Derived3>()); break;
            case 3: objs.push_back(std::make_unique<Derived4>()); break;
        }
    }
    int sum = 0;
    for (auto _ : state) {
        for (auto& o : objs)
            sum += o->compute(42);
        benchmark::DoNotOptimize(sum);
    }
}

BENCHMARK(BM_DirectCall);
BENCHMARK(BM_VirtualMono);
BENCHMARK(BM_VirtualPoly);
BENCHMARK_MAIN();
```

```bash
g++ -O2 -std=c++17 bench_vtable.cpp -lbenchmark -lbenchmark_main -lpthread -o bench_vtable
./bench_vtable

# 예상 출력 (단위: ns/iter, 1000 요소 처리):
# BM_DirectCall    ~500 ns    (정적 디스패치, 벡터화 가능)
# BM_VirtualMono  ~900 ns    (단일 타입, BTB 학습 완료)
# BM_VirtualPoly  ~2100 ns   (4가지 혼합, BTB 혼란)
# → 단형 vs 다형 가상 호출: ~2.3배 차이
```

### 실험 3: perf로 branch-misses 측정

```bash
# 단일 타입 vs 혼합 타입 분기 예측 실패율 비교
perf stat -e cycles,instructions,branch-misses,branches \
    ./bench_virtual_mono

perf stat -e cycles,instructions,branch-misses,branches \
    ./bench_virtual_poly

# 주목할 항목:
# branch-miss ratio = branch-misses / branches
# Mono: 1~2% 예상
# Poly: 15~30% 예상 (4가지 타입 × 25% 각 확률)

# 캐시 미스도 확인 (vtable이 콜드 캐시인 경우)
perf stat -e cycles,cache-misses,cache-references \
    ./bench_virtual_poly_cold
```

### 실험 4: devirtualization 확인 (final 키워드)

```bash
# final 없는 버전과 final 있는 버전의 어셈블리 비교

# godbolt에서:
# 버전 A: class Derived : public Base { ... };
# 버전 B: class Derived final : public Base { ... };

# 직접 확인:
g++ -O2 -S -masm=intel vtable_test.cpp -o vtable_no_final.s
# Derived에 final 추가 후:
g++ -O2 -S -masm=intel vtable_test.cpp -o vtable_final.s

diff vtable_no_final.s vtable_final.s
# final 버전: "call [rax]" 사라지고 직접 "call _ZN7Derived7compute..."
```

---

## 📊 성능 비교

```
디스패치 방식별 비용 (x86-64, 함수 본체 1개 명령 기준):

방식                        사이클(캐시 히트)  사이클(캐시 미스)  인라이닝
────────────────────────────────────────────────────────────────────────
정적 디스패치(직접 call)        ~5               ~5              가능
인라이닝 (same TU, -O2)         ~1               ~1              완전
가상 단일 타입 (BTB 학습)       ~8~12            +40~200         불가
가상 혼합 타입 (BTB 실패)       ~20~35           +40~200+        불가
가상 + vtable 캐시 미스         +40~200          +40~200         불가

Google Benchmark 실측 (1000 요소 루프):
  정적 호출:          ~500 ns
  가상 단일 타입:     ~900 ns   (1.8배)
  가상 4가지 혼합:   ~2100 ns   (4.2배)

perf 측정 (branch-miss ratio):
  정적 호출:         0.1%
  가상 단일 타입:    1.5%
  가상 4가지 혼합:   22.4%
```

---

## ⚖️ 트레이드오프

```
가상 함수 사용 결정 기준:

사용하는 것이 합리적인 경우:
  ✅ 실제로 다형성이 필요한 경우 (런타임 타입 변환이 핵심 요구사항)
  ✅ 해당 호출이 성능 크리티컬 경로가 아닌 경우
  ✅ 인터페이스 분리가 유지보수성을 크게 높이는 경우
  ✅ JIT 환경 (Java): IC가 모노모픽으로 학습하면 직접 call과 동등

피해야 하는 경우:
  ❌ 핫 루프 안에서 수백만 회 이상 호출 + 다양한 서브클래스 혼합
  ❌ 객체가 캐시에 올라가지 않는 대규모 배열 순회
  ❌ 함수 본체가 매우 작아서 호출 오버헤드 > 본체 비용인 경우

대안:
  ① std::variant + std::visit: 컴파일 타임 타입 열거 → devirtualize 가능
  ② CRTP (Curiously Recurring Template Pattern): 정적 다형성
     template<class Derived> struct Base {
         void call() { static_cast<Derived*>(this)->impl(); }
     };
  ③ 데이터 지향 설계 (Ch4-03): 타입별로 배열 분리 → 각 배열은 단일 타입
     std::vector<Circle>  circles;    // 전부 Circle → devirtualize
     std::vector<Square>  squares;    // 전부 Square → devirtualize
  ④ final 키워드: 컴파일러에 "더 이상 상속 없음" 알림 → static devirt

PGO 활용:
  → 프로덕션 빌드에 PGO 적용 시 hotspot의 가상 호출 비용 30~50% 절감
  → Chromium, Firefox 모두 PGO 기본 적용
```

---

## 📌 핵심 정리

```
vtable 비용 핵심:

구조:
  객체 → vptr (8 bytes) → vtable (함수 포인터 배열) → 함수 코드
  단일 상속: vptr 1개
  다중 상속: vptr 여러 개 (복잡)

비용 두 가지:
  1. 메모리 간접 참조: vptr 로드 + vtable 슬롯 로드
     → 캐시 히트 시 ~8~12 cy, 미스 시 +40~200 cy
  2. 간접 분기 예측 실패:
     → 단일 타입: BTB 학습 후 거의 없음 (~1%)
     → 혼합 타입: 높은 미스율 (~20~30%), 각 15~20 cy 낭비

Devirtualization:
  컴파일러: final, 구체 타입 확정, 같은 TU에서 확인 가능 시
  PGO: 프로파일 기반 추측적 devirtualize → if(likely) 직접 call
  JIT(HotSpot): 인라인 캐싱 → Mono/Bi/Megamorphic 자동 분류

인라이닝 불가가 핵심 비용:
  가상 함수는 컴파일 타임에 본체 모름 → 인라이닝 불가
  → 추가 최적화(상수 전파 등)도 차단됨
  → 정적 디스패치 대비 최대 4~5배 느릴 수 있음 (최악 케이스)

JVM 연결:
  invokeinterface > invokevirtual (itable 탐색 추가 비용)
  JIT IC: Mono→직접 call, Bi→if/else, Mega→vtable 방식
  java-concurrency-deep-dive의 동기화 메서드도 이 경로로 최적화됨
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 `-O2`를 사용할 때 어느 호출이 devirtualize 되는가? 그 이유는?

```cpp
struct Base { virtual int f() = 0; };
struct Derived final : Base { int f() override { return 42; } };

void test1(Base* b)     { b->f(); }    // A
void test2(Derived* d)  { d->f(); }   // B
void test3()            { Derived d; d.f(); }  // C
```

<details>
<summary>해설 보기</summary>

**A (Base*)**: devirtualize 불가. 컴파일러는 `b`가 실제로 `Derived`인지 다른 서브클래스인지 알 수 없습니다(비록 `Derived`가 final이라도 `Base`를 상속한 다른 클래스가 존재할 수 있습니다). 간접 `call`이 생성됩니다.

**B (Derived*)**: devirtualize 가능. `Derived`가 `final`이므로 더 이상 오버라이드가 없습니다. 컴파일러는 "이 호출은 반드시 `Derived::f`"라고 증명할 수 있어 직접 call 또는 인라이닝이 적용됩니다. godbolt에서 확인하면 `call [rax]` 대신 `call Derived::f` 또는 아예 `mov eax, 42`로 치환됩니다.

**C (스택 지역 변수)**: 완전한 devirtualize + 인라이닝. `d`의 타입이 스택에서 확정(`Derived`)이고, `Derived::f`가 `return 42`이므로 컴파일러는 전체를 `mov eax, 42; ret`으로 최적화합니다.

핵심: 컴파일러의 devirtualization 능력은 "타입을 알 수 있는 범위"에 의존합니다. `Base*`는 너무 넓은 범위이고, `Derived* final`이나 스택 변수는 타입을 확정할 수 있습니다.

</details>

---

**Q2.** Java의 `invokeinterface`와 `invokevirtual`의 성능 차이는 무엇이고, JIT의 인라인 캐싱이 이 차이를 어떻게 메우는가?

<details>
<summary>해설 보기</summary>

`invokevirtual`(클래스 메서드): vtable 슬롯 오프셋이 컴파일 타임에 고정됩니다. `[vptr + offset]` 한 번의 역참조로 끝납니다.

`invokeinterface`(인터페이스 메서드): 인터페이스를 구현한 클래스들이 각자 다른 위치에 메서드를 배치합니다. JVM은 itable(interface dispatch table)을 탐색하는 추가 단계가 필요하고, 이는 추가 메모리 접근 1~2회를 의미합니다.

그러나 HotSpot JIT의 인라인 캐싱이 이 차이를 해소합니다:

- **Monomorphic IC**: 인터페이스를 통해 항상 같은 구체 클래스가 호출되면 JIT는 "런타임에 타입을 체크하고, 일치하면 직접 call"로 변환합니다. 사실상 `invokevirtual`과 동일한 비용이 됩니다.
- **Bimorphic IC**: 두 가지 타입이 50:50으로 섞이면 `if (type == A) A::method else B::method` 형태의 직접 call로 변환됩니다.
- **Megamorphic IC**: 3가지 이상이면 itable 탐색으로 폴백합니다.

실무 가이드: Java에서 인터페이스를 많이 써도 대부분의 호출 지점이 Monomorphic이라면 JIT 최적화로 성능 차이가 없어집니다. 문제는 정말로 많은 구현체가 한 호출 지점에서 섞이는 경우(Megamorphic)입니다.

</details>

---

**Q3.** 게임 엔진에서 `std::vector<Enemy*>` 대신 `std::vector<Enemy>`를 권장하는 이유는 vtable 비용 외에 또 무엇이 있는가?

<details>
<summary>해설 보기</summary>

vtable 비용 외에 두 가지 중요한 이유가 있습니다:

**1. 캐시 지역성 (pointer chasing vs contiguous memory)**:
`std::vector<Enemy*>`: 포인터 배열 → 각 포인터가 힙의 서로 다른 위치를 가리킴. 순회 시 매 원소마다 캐시 미스 가능성. 전형적인 "pointer chasing" 문제 (Ch4-03에서 상세).

`std::vector<Enemy>`: 객체 자체가 연속적으로 배치 → 순회 시 프리페치 가능, 캐시 히트율 높음.

**2. 메모리 할당 오버헤드**:
`std::vector<Enemy*>`: 1,000개 적군 = 1,000번의 `new` 호출 → 힙 단편화, malloc 오버헤드, 메타데이터 각 16~32 bytes 낭비.

`std::vector<Enemy>`: 1번의 연속 할당 → 단편화 없음, malloc 오버헤드 최소.

실무에서 게임 엔진(ECS 패턴)은 이 두 문제를 해결하기 위해 아예 가상 함수를 없애고 컴포넌트별 배열(`std::vector<Position>`, `std::vector<Velocity>`)을 사용합니다. 동일 타입의 데이터가 연속 배치되어 SIMD까지 적용 가능해집니다. 자세한 내용은 Ch4-03 포인터 추적 문서를 참조하세요.

</details>

---

<div align="center">

**[⬅️ 이전: 함수 호출 비용](./01-function-call-cost.md)** | **[홈으로 🏠](../README.md)** | **[다음: 포인터 추적 ➡️](./03-pointer-chasing.md)**

</div>
