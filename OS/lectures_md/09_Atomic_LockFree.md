# 9강 — Atomic과 Lock-Free의 맛보기

> **목표:** 락 없이 멀티스레드 안전한 코드가 가능한 이유를 안다. `std::atomic`, CAS, 메모리 배리어의 개념을 잡는다. lock-free가 항상 빠르다는 환상을 깬다.
> **선수 지식:** 7~8강 (Race Condition, Mutex).

> **이 강의는 일부러 깊게 안 판다.** Lock-free의 정확한 구현은 박사 논문 주제다. 면접에서 "atomic이 뭔지, 왜 빠른지, 한계가 뭔지" 답하는 정도가 목표.

---

## 1. 도입 — 락은 비싸다

8강 끝에서 본 카운터 코드.

```cpp
int32 g_Shared = 0;
FCriticalSection g_Lock;

for (int i = 0; i < 1000000; ++i)
{
    FScopeLock G(&g_Lock);   // 락 잡고
    g_Shared++;              // ++
}                            // 락 풀고
```

이거 잘 돈다. 정확히 2,000,000 나옴. 그런데 **느리다.**
매 ++마다 락/언락 비용이 들고, 두 스레드가 거의 직렬로 도는 셈이다.

`++` 한 줄을 위해 락을 잡는 게 과한 거 아닌가? 더 가벼운 방법이 있다.

---

## 2. CPU가 직접 보장하는 원자 명령

현대 CPU는 "이 한 명령은 무조건 원자적으로 처리" 라는 **특수 명령어**들을 제공한다.

### x86의 LOCK 접두사

```asm
lock inc dword ptr [g_Shared]    ; g_Shared++ 를 원자적으로
```

`lock` 접두사가 붙으면 CPU가 그 명령 동안 **메모리 버스를 잠가서** 다른 코어가 해당 캐시 라인에 접근 못 하게 한다. 한 명령이 통째로 원자적으로 끝남.

### CAS (Compare-And-Swap) — 원자성의 왕

```
CAS(addr, expected, new):
    if (*addr == expected):
        *addr = new
        return true
    else:
        return false
    (전체가 한 덩어리, 끊김 없음)
```

x86의 `cmpxchg` 명령이 이걸 한다. **대부분의 lock-free 자료구조가 CAS로 만들어진다.**

---

## 3. `std::atomic` — 위 CPU 명령의 C++ 래퍼

```cpp
#include <atomic>

std::atomic<int32> Counter{0};

void Worker()
{
    for (int i = 0; i < 1000000; ++i)
    {
        Counter.fetch_add(1);   // 내부적으로 lock inc 같은 명령
    }
}
```

- `Counter++` 도 가능. 연산자 오버로딩으로 atomic 연산이 호출됨.
- 락 없이 정확. 그리고 락보다 훨씬 빠름.

### 언리얼의 대응

| C++ 표준 | 언리얼 |
|---------|--------|
| `std::atomic<int32>` | `TAtomic<int32>` 또는 `FThreadSafeCounter` |
| `std::atomic<bool>` | `FThreadSafeBool` |

```cpp
FThreadSafeCounter Counter;
Counter.Increment();   // 원자적 ++
Counter.Add(5);        // 원자적 += 5
int32 V = Counter.GetValue();
```

`FThreadSafeCounter` 가 단순 카운터엔 제일 무난.

---

## 4. atomic이 mutex보다 빠른 이유

| | Mutex | Atomic |
|---|-------|--------|
| 락 못 받으면 | OS Waiting 상태로 잠 → 컨텍스트 스위칭 발생 | CAS 재시도 또는 한 명령으로 끝 |
| 비용 | 수십~수백 ns + 잠들면 수 μs | 보통 한 자릿수 ns ~ 수십 ns |
| 보호 범위 | 임의 코드 구간 | 한 변수 하나 (보통 4~8바이트) |
| 복잡도 | 쉬움 | 어려움 (메모리 순서, ABA 문제 등) |

**핵심: atomic은 "한 변수"에 대한 짧은 연산에만 쓴다.** 여러 변수를 일관성 있게 바꿔야 하면 mutex가 답.

---

## 5. 메모리 배리어 / 메모리 순서

여기서부터 진짜 어렵다. 핵심만 짚고 간다.

### 문제: 컴파일러와 CPU가 명령 순서를 바꾼다

```cpp
Data = 42;
bReady = true;
```

단일 스레드 관점에선 순서가 바뀌어도 결과 같으므로, 컴파일러나 CPU가 둘을 바꿔서 실행할 수 있다. **다른 스레드가 보면:**

```cpp
if (bReady)        // true 보임
    Use(Data);     // 그런데 Data는 아직 0!
```

### `std::atomic` 의 memory_order 인자

```cpp
std::atomic<int> Data{0};
std::atomic<bool> bReady{false};

// 쓰는 쪽
Data.store(42, std::memory_order_relaxed);
bReady.store(true, std::memory_order_release);   // 이 store 이전의 쓰기는 다 보이게

// 읽는 쪽
if (bReady.load(std::memory_order_acquire))      // 이 load 이후의 읽기는 그 이후로
{
    int V = Data.load(std::memory_order_relaxed);  // 안전하게 42
}
```

종류:
- `memory_order_relaxed` — 원자성만 보장. 순서 보장 X. 카운터 같은 데.
- `memory_order_acquire` / `release` — 락 잡기/풀기 같은 효과. **가장 흔히 씀.**
- `memory_order_seq_cst` — 가장 강함, 기본값. 다 순서 보장. 느림.

> 면접에서 "memory order 다 설명해보세요" 들으면 깊이 빠진다. **"순서 재배치를 막는 도구이고, acquire/release를 가장 많이 쓴다"** 정도로 충분.

---

## 6. Lock-Free 자료구조

CAS를 활용해서 락 없이 동시 접근 가능한 자료구조들. 예:

- Lock-free queue (생산자-소비자 패턴)
- Lock-free stack
- Lock-free hash map

```cpp
// 의사 코드: lock-free stack의 push
void Push(Node* NewNode)
{
    Node* OldHead;
    do {
        OldHead = Head.load();
        NewNode->Next = OldHead;
    } while (!Head.compare_exchange_weak(OldHead, NewNode));
    // OldHead가 그동안 안 바뀌었으면 Head를 NewNode로 바꾸고 끝.
    // 누가 끼어들었으면 다시 시도.
}
```

이게 핵심 패턴: **"비교해서 같으면 바꿔, 다르면 다시 시도."**

언리얼이 내부적으로 `TLockFreePointerListUnordered` 같은 걸 제공한다.

---

## 7. "Lock-Free = 무조건 빠름" 은 거짓이다

흔한 환상: "lock-free라니까 lock보다 빠르겠지."

현실:

### (1) 경쟁이 적으면 둘 다 빠름, 차이 작음

락도 경쟁 없으면 거의 공짜.

### (2) 경쟁이 심하면 lock-free가 더 나빠질 수도

CAS 실패하면 **다시 시도(retry).** 경쟁이 심하면 retry 폭주 → CPU 다 쓰면서 진척은 더딤. 이걸 **livelock** 이라 부르기도 함.

### (3) 캐시 라인 핑퐁

여러 코어가 같은 atomic 변수를 두드리면, 그 변수의 캐시 라인이 코어 간에 계속 왔다갔다 한다(**False Sharing의 사촌**). 매번 코어 캐시가 무효화돼서 느려진다.

```cpp
// 코어 0, 1, 2, 3 이 다 두드림
std::atomic<int64> Counter{0};   // 캐시 라인 하나에 모든 코어 경쟁 → 핑퐁
```

해결: 코어별로 카운터를 따로 둬서 마지막에 합산(per-CPU counter).

### (4) 코드 복잡도

lock-free 자료구조는 **버그 만들기 정말 쉽다.** ABA 문제, 메모리 회수 타이밍 등. 면접에서 "lock-free 직접 짜본 적 있어요?" 들으면 신중하게 답하라.

---

## 8. ABA 문제 (들어만 두면 됨)

CAS의 함정.

1. 스레드 1이 Head를 읽음 → 값 A
2. 스레드 2가 A를 빼고, B를 넣고, 다시 A를 넣음 (메모리 재사용)
3. 스레드 1이 CAS(A → New) 시도 → "A로 보이네, 그동안 안 바뀌었군" → **사실 바뀌었음**
4. 스레드 1이 망가진 상태로 갱신

해결책: 버전 카운터 동반(`std::atomic<std::pair<T*, uint64>>`), 해저드 포인터, RCU 등. 그냥 "ABA 문제" 키워드만 알아두고, 직접 해결은 면접 범위 밖.

---

## 9. 실전 — 언제 락, 언제 atomic?

### Atomic 적당한 경우

- 단일 카운터 (`FThreadSafeCounter`)
- 플래그 (`FThreadSafeBool`)
- 단일 포인터 교체 (예: 더블 버퍼링)

### Mutex 적당한 경우

- 컬렉션 (`TArray`, `TMap`)
- 여러 변수가 일관성 있게 바뀌어야 할 때
- 임계영역이 좀 길 때

### Lock-Free 자료구조 적당한 경우

- 생산자-소비자 패턴에서 진짜로 처리량이 병목일 때
- 검증된 라이브러리를 쓸 수 있을 때

**원칙: 가장 간단한 도구부터.** mutex → atomic → lock-free. 점점 위로 가면서 성능 측정해보고 진짜 필요할 때만.

---

## 10. 코드로 — 같은 카운터, 3가지 방법

```cpp
// (1) Mutex
int32 V1 = 0;
FCriticalSection L;
void Inc1()
{
    FScopeLock G(&L);
    V1++;
}

// (2) Atomic
std::atomic<int32> V2{0};
void Inc2()
{
    V2.fetch_add(1, std::memory_order_relaxed);
}

// (3) 언리얼 FThreadSafeCounter
FThreadSafeCounter V3;
void Inc3()
{
    V3.Increment();
}
```

벤치마크 결과(대략): **(2) ≈ (3) ≪ (1)**. atomic이 mutex보다 10~50배 빠른 게 보통.
다만 카운터가 아니라 컬렉션 수정이면 atomic 단독으론 못 한다.

---

## 11. 흔한 오해

### 오해 1: "`volatile` 쓰면 atomic이다"

이거 또 나오는 오해. (8강에서도 다룸.) `volatile`은 컴파일러 최적화만 막을 뿐, 멀티스레드 안전성 보장 X. `std::atomic`이나 `TAtomic`이 따로 있다.

### 오해 2: "atomic 변수 두 개 쓰면 둘 사이도 안전"

NO. **각 변수의 원자성만 보장.** 두 atomic 변수의 관계는 별개.

```cpp
std::atomic<int> A{0}, B{0};

A.store(1);
B.store(1);
// 다른 스레드가 봤을 때 "B=1, A=0" 도 가능. 두 변수 사이 순서 보장은 별도.
```

### 오해 3: "lock-free면 wait-free 같은 거잖아"

다르다.
- **Lock-Free**: 어떤 스레드는 항상 진척한다. 다른 스레드들은 retry 할 수 있음.
- **Wait-Free**: 모든 스레드가 유한한 단계 안에 끝난다. 가장 강한 보장.

CAS 루프는 lock-free이지만 wait-free는 아님(특정 스레드가 retry만 계속 할 수 있음).

### 오해 4: "FString 같은 큰 타입도 `std::atomic`으로 감싸면 빨라짐"

큰 타입은 lock을 내부적으로 쓰는 fallback이 들어가서 오히려 느릴 수 있다. `std::atomic`은 보통 **포인터 크기(8바이트)나 그 미만**일 때 진짜 원자 명령으로 처리됨.

---

## 12. 면접에서 나오면

**Q1. atomic 연산이 뭔가요?**
→ "중간에 다른 스레드의 개입 없이 원자적으로 실행되는 연산입니다. CPU의 LOCK 접두사나 CMPXCHG 같은 특수 명령으로 보장하며, C++에서는 `std::atomic`, 언리얼에서는 `TAtomic`이나 `FThreadSafeCounter`로 사용합니다."

**Q2. Mutex 대신 atomic을 쓰면 좋은 점은?**
→ "락 획득/해제 비용이 없고, 락 못 받았을 때 컨텍스트 스위칭이 일어나지 않습니다. 또 OS 호출이 거의 없어 수십 배 빠릅니다. 다만 한 변수에 대한 짧은 연산만 가능하고, 여러 변수의 일관성을 지켜야 한다면 여전히 mutex가 필요합니다."

**Q3. CAS (Compare-And-Swap)가 뭔가요?**
→ "지정한 메모리 주소의 값이 예상값과 같으면 새 값으로 바꾸고, 다르면 실패를 반환하는 원자 연산입니다. 이 비교-교환 전체가 끊김 없이 일어나며, 대부분의 lock-free 자료구조의 기본 빌딩 블록입니다."

**Q4. Lock-Free가 항상 더 빠른가요?**
→ "아닙니다. 경쟁이 낮으면 mutex와 별 차이 없고, 경쟁이 심하면 CAS retry가 폭주해 오히려 느려질 수 있습니다. 또 캐시 라인 핑퐁 때문에 코어 간 경쟁이 비싸지고, 구현이 어려워 버그 가능성도 높습니다. 단순 카운터 같은 곳에는 효과적이지만 만능은 아닙니다."

**Q5. 메모리 배리어가 왜 필요한가요?**
→ "컴파일러와 CPU가 단일 스레드 관점에서 결과가 같은 한 명령어 순서를 재배치할 수 있습니다. 단일 스레드에선 문제 없지만 다른 스레드가 그걸 보면 순서가 뒤집혀 보일 수 있고, '데이터 준비 → 플래그 세움' 같은 동기화가 깨집니다. 메모리 배리어(acquire/release 등)는 이 재배치를 막아 다른 스레드가 일관된 순서로 보게 합니다."

**Q6. `FThreadSafeCounter`와 `FCriticalSection`을 둘 다 쓸 수 있을 때 뭘 고르겠어요?**
→ "보호 대상이 단일 정수 카운터라면 `FThreadSafeCounter`를 고릅니다. 락 비용 없이 같은 안전성을 제공하면서 훨씬 빠릅니다. 만약 카운터 외에 다른 변수도 함께 일관되게 갱신해야 한다면 `FCriticalSection`이 필요합니다."

---

## 13. 셀프 체크

- [ ] atomic이 CPU의 원자 명령(LOCK, CMPXCHG)에 기반한다는 걸 안다.
- [ ] atomic과 mutex의 속도 차이가 나는 이유를 말할 수 있다.
- [ ] CAS의 동작과 retry 루프를 의사 코드로 그릴 수 있다.
- [ ] 메모리 배리어가 왜 필요한지 한 문단으로 설명할 수 있다.
- [ ] Lock-Free가 항상 빠른 게 아니라는 걸 안다.
- [ ] 단일 카운터/플래그엔 atomic, 컬렉션엔 mutex라는 원칙을 안다.

---

다음 강의: **10강 — 데드락 (Deadlock)**
"락 두 개를 잘못 잡으면 게임이 멈춘다. 4가지 조건, 해결책, 실무 디버깅."
