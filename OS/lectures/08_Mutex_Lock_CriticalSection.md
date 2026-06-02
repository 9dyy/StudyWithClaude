# 8강 — Mutex, Lock, Critical Section

> **목표:** Race Condition을 막는 가장 기본 도구인 Mutex를 이해한다. 언리얼의 `FCriticalSection`, `FScopeLock` 사용법과 함정.
> **선수 지식:** 7강 (Race Condition, 임계영역).

---

## 1. 도입 — 임계영역에 자물쇠를 채우자

7강 결론: "임계영역은 한 번에 한 스레드만 들어가야 한다."

가장 직관적인 해결책: **자물쇠.**

```
공유 데이터 ─→ [열쇠가 있어야 들어갈 수 있는 방]
                    │
                    ↓
              ┌─────────────┐
              │   임계영역    │ ← 한 명만 들어감
              │  공유데이터+= 1│
              └─────────────┘
                    │
                    ↓
              나오면서 열쇠 반납
```

이 "자물쇠" 가 **Mutex (Mutual Exclusion)** 다.

---

## 2. Mutex의 동작

```
스레드 A: Lock()       ← 열쇠 받음 (성공). 임계영역 진입.
           Counter++;
           Unlock()     ← 열쇠 반납.

스레드 B: Lock()       ← A가 열쇠 가지고 있음. B는 Waiting 상태로 잠듦.
           ...           ← A가 Unlock() 하는 순간 OS가 B를 깨움
           Counter++;
           Unlock()
```

핵심:
- `Lock()` 은 **블로킹 호출.** 락이 이미 잡혀있으면 풀릴 때까지 잠든다.
- 잠든 스레드는 OS의 Waiting 상태로 들어가고, CPU를 다른 스레드에 양보한다.
- `Unlock()` 하면 기다리던 스레드 중 하나가 깨어난다.

---

## 3. 언리얼에서: `FCriticalSection`

표준 C++의 `std::mutex` 와 같은 역할.

```cpp
#include "HAL/CriticalSection.h"

class FInventory
{
public:
    void AddItem(int32 ItemId)
    {
        Lock.Lock();
        Items.Add(ItemId);
        Lock.Unlock();
    }

    int32 GetCount()
    {
        Lock.Lock();
        int32 Count = Items.Num();
        Lock.Unlock();
        return Count;
    }

private:
    TArray<int32>      Items;
    FCriticalSection   Lock;
};
```

이제 여러 스레드가 `AddItem` 을 동시에 불러도 안전하다.

---

## 4. 거의 항상 함정 — `Lock()` / `Unlock()` 직접 호출

위 코드의 문제:

```cpp
void Foo()
{
    Lock.Lock();

    if (SomeCondition)
        return;          // ❌ Unlock() 안 부르고 빠져나감 → 락 영구 보유

    DoStuff();           // 여기서 예외 throw 가능 → 마찬가지

    Lock.Unlock();
}
```

이런 식으로 락을 안 풀고 빠져나가면 **다른 스레드가 영원히 잠든다.** 게임 멈춤.

해결책: **RAII 패턴.** 객체 수명에 락 수명을 묶어버린다. 언리얼은 `FScopeLock`을 제공.

---

## 5. `FScopeLock` — 무조건 이걸 써라

```cpp
void AddItem(int32 ItemId)
{
    FScopeLock LockGuard(&Lock);   // 생성자에서 Lock(), 소멸자에서 Unlock()
    Items.Add(ItemId);
}   // ← 여기서 LockGuard 소멸 → 자동 Unlock
```

- `return`, 예외, 어떤 경로로 빠져나가든 소멸자가 호출되니까 락이 풀린다.
- C++ 표준의 `std::lock_guard`, `std::unique_lock` 과 같은 패턴.

> **실무 원칙: `FCriticalSection`의 `Lock/Unlock` 을 직접 부르지 마라. 무조건 `FScopeLock`.**

---

## 6. 락 잡는 범위는 최소로

```cpp
// ❌ 락 너무 넓게
void Heavy()
{
    FScopeLock G(&Lock);

    int32 Item = Items.Last();
    HeavyComputation(Item);    // 1초 걸리는 작업을 락 잡은 채로 → 다른 스레드 1초 대기
    NetworkCall();
}

// ✅ 락은 데이터 만질 때만
void Heavy()
{
    int32 Item;
    {
        FScopeLock G(&Lock);
        Item = Items.Last();
    }
    HeavyComputation(Item);   // 락 없이
    NetworkCall();             // 락 없이
}
```

락을 잡고 있는 동안은 **다른 스레드가 멈춰 있다.** 락 안에서 무거운 일, I/O, 시스템 콜은 피하라.

---

## 7. Mutex의 종류

### (1) 기본 Mutex (`std::mutex`, `FCriticalSection`)

같은 스레드가 두 번 Lock 하면 **데드락.**

```cpp
FCriticalSection Lock;

void A() { FScopeLock G(&Lock); B(); }
void B() { FScopeLock G(&Lock); ... }   // ❌ 같은 스레드가 또 락 → 영원히 멈춤
```

### (2) 재귀 Mutex (`std::recursive_mutex`)

같은 스레드가 여러 번 Lock 해도 OK. 풀 때도 같은 횟수 Unlock 해야 풀림.
편하긴 한데 코드 흐름이 복잡해지면 의도치 않은 동작 위험. **남용 X.**

### (3) Read-Write Lock (`FRWLock`)

- 여러 스레드의 **읽기는 동시 허용.**
- **쓰기는 단독.**

읽기가 압도적으로 많은 데이터(설정값, 캐시)에 유리.

```cpp
FRWLock RWLock;

// 읽기
RWLock.ReadLock();
int32 V = Cache[Key];
RWLock.ReadUnlock();

// 쓰기
RWLock.WriteLock();
Cache[Key] = NewValue;
RWLock.WriteUnlock();
```

### (4) Spinlock

기다릴 때 자지 않고 **빙빙 돌면서(spin) 대기**. CPU를 낭비하지만 컨텍스트 스위칭 비용이 없음.
**락을 아주 짧게만 잡을 거고, 잠드는 비용이 더 클 때** 유리.

```cpp
FSpinLock Spin;

Spin.Lock();
SmallOperation();    // 1μs 이하
Spin.Unlock();
```

쓰는 경우가 드물고, 잘못 쓰면 코어 하나가 100% 됨.

---

## 8. 락은 공짜가 아니다 — 잠재 비용

1. **잠금/해제 비용** — 보통 수십 나노초. 많이 잡으면 누적.
2. **대기 비용** — 락 못 받은 스레드는 잔다 → 컨텍스트 스위칭.
3. **직렬화 비용** — 락 안의 코드는 결국 **한 명만** 도므로, 그 부분이 전체 성능 상한을 만든다 (Amdahl's Law).
4. **데드락 위험** — 다음 강의 (10강)에서 다룸.

그래서 **공유를 줄이는 게 최선**, 어쩔 수 없으면 **락 범위 최소화**, 그래도 부족하면 **lock-free / atomic (9강)** 으로 간다.

---

## 9. 세마포어 (Semaphore) — Mutex의 일반화

Mutex가 "한 명만 들어가는 자물쇠" 라면, 세마포어는 "**N명까지** 들어가는 자물쇠."

```
세마포어 = 카운터
Wait()    → 카운터 0이면 잠, 아니면 카운터-1 하고 진입
Signal()  → 카운터+1, 기다리던 스레드 깨움
```

용도:
- **자원 풀:** 동시에 N개의 네트워크 커넥션만, 동시에 4개의 비디오 디코더만.
- **생산자-소비자 패턴:** 버퍼에 데이터 있는 만큼만 소비자가 깨어남.

```cpp
// 의사 코드
FSemaphore Sem(/*Count=*/4);   // 동시 4개 허용

void HandleRequest()
{
    Sem.Wait();
    DoWork();    // 동시에 최대 4명만 여기 있음
    Sem.Signal();
}
```

> 세마포어 vs Mutex 정리: Mutex는 N=1인 세마포어와 비슷하지만, "락 잡은 스레드만 풀 수 있다" 는 **소유권** 개념이 있고, 세마포어는 그게 없다(아무나 Signal 가능).

### 이진 세마포어 (Binary Semaphore)

N=1인 세마포어. Mutex와 거의 같지만 소유권이 없음. **시그널 용도**(한 스레드가 다른 스레드를 깨우는 용도)에 종종 쓰임.

언리얼의 `FEvent` 가 이 시그널/이벤트 패턴에 가깝다.

---

## 10. 코드로 — 7강의 망가진 카운터 고치기

```cpp
// Before (7강)
int32 g_Shared = 0;
void Worker()
{
    for (int i = 0; i < 1000000; ++i)
        g_Shared++;   // Race
}

// After
int32 g_Shared = 0;
FCriticalSection g_Lock;

void Worker()
{
    for (int i = 0; i < 1000000; ++i)
    {
        FScopeLock G(&g_Lock);
        g_Shared++;
    }
}
```

이제 결과는 정확히 2,000,000 이 나온다. **하지만 굉장히 느리다.** 매 ++마다 락/언락 비용 + 두 스레드가 직렬로 도는 셈이라.

→ 이런 단순 카운터엔 **atomic** 이 훨씬 빠르다. (다음 강의)

```cpp
// 더 좋은 버전 (9강에서 다룸)
std::atomic<int32> g_Shared{0};
void Worker()
{
    for (int i = 0; i < 1000000; ++i)
        g_Shared.fetch_add(1, std::memory_order_relaxed);
}
```

---

## 11. 흔한 오해

### 오해 1: "락만 잘 쓰면 어떤 상황도 안전하다"

락만 잡으면 안전하긴 한데, **데드락**, **성능 저하**, **우선순위 역전** 같은 새 문제가 생긴다.

### 오해 2: "락 잡은 줄 모르고 또 잡으면 알아서 되겠지"

기본 Mutex는 **같은 스레드도 두 번 잡으면 데드락.** 재귀 락이 필요하면 명시적으로 `recursive_mutex` 써야 함.

### 오해 3: "Mutex가 const 함수에선 안 잡혀도 됨"

읽기만 하는 const 함수라도 **다른 스레드가 동시에 쓰고 있을 수 있다.** 읽기조차도 락(혹은 RWLock의 ReadLock)이 필요.

```cpp
class FCounter
{
public:
    int32 Get() const     // const니까 안전? NO
    {
        return Value;     // 다른 스레드가 쓰는 중일 수 있음
    }
    mutable FCriticalSection Lock;
    int32 Value = 0;
};
```

### 오해 4: "`volatile` 키워드면 스레드 안전"

C++의 `volatile`은 **컴파일러 최적화 방지**용이지 **멀티스레드 동기화용이 아니다.** Race를 막아주지 못한다. 진짜 동기화는 mutex나 atomic을 써야 한다. (자바의 `volatile`은 의미가 다름, 거기랑 헷갈리지 말 것.)

---

## 12. 면접에서 나오면

**Q1. Mutex가 뭔가요?**
→ "임계영역에 한 번에 한 스레드만 진입하도록 보장하는 동기화 기본 도구입니다. Lock으로 들어가고 Unlock으로 나오며, 락이 잡혀있으면 다른 스레드는 풀릴 때까지 Waiting 상태로 잠들었다가 깨어납니다."

**Q2. Mutex와 Semaphore의 차이는?**
→ "Mutex는 동시 진입을 1명으로 제한하고 락을 잡은 스레드만 풀 수 있는 소유권 개념이 있습니다. 세마포어는 N명까지 동시 진입을 허용하는 카운터이고, 락을 잡지 않은 스레드도 Signal로 카운터를 올릴 수 있습니다. Mutex는 임계영역 보호, 세마포어는 자원 풀 제한이나 생산자-소비자 패턴에 적합합니다."

**Q3. `FScopeLock` (또는 `std::lock_guard`)을 쓰는 이유는?**
→ "RAII 패턴으로 락 수명을 객체 수명에 묶어 자동 해제를 보장합니다. `return`, 예외, 어떤 경로로든 함수가 빠져나갈 때 소멸자가 호출되어 자동으로 Unlock되므로, Lock/Unlock을 직접 부를 때 생기는 락 누수 버그를 막아줍니다."

**Q4. 락을 잡고 무거운 연산을 하면 안 되는 이유는?**
→ "락을 잡고 있는 동안 다른 스레드들이 다 멈춰서 기다리므로 동시성이 사라집니다. 또 그 시간만큼 컨텍스트 스위칭 비용도 늘고, 직렬화된 부분이 전체 성능의 상한이 됩니다(Amdahl's Law). 락은 데이터 접근하는 최소 구간에만 잡고, 무거운 계산은 락 밖에서 해야 합니다."

**Q5. Read-Write Lock이 일반 Mutex보다 좋은 경우는?**
→ "읽기 작업이 압도적으로 많고 쓰기가 드물 때 유리합니다. 일반 Mutex는 읽기조차 직렬화하지만, RWLock은 동시 읽기를 허용하고 쓰기일 때만 단독 접근하므로 처리량이 늘어납니다. 설정 캐시나 게임 상태 조회 같은 데 적합합니다."

**Q6. Spinlock은 언제 쓰나요?**
→ "락을 잡는 시간이 매우 짧고, 컨텍스트 스위칭 비용보다 잠시 도는 비용이 더 작을 때 씁니다. 멀티코어에서 짧은 임계영역에 유리합니다. 다만 락을 길게 잡으면 CPU를 낭비하고, 싱글코어에선 의미가 없습니다."

**Q7. C++의 `volatile`이 스레드 동기화 도구인가요?**
→ "아닙니다. `volatile`은 컴파일러의 최적화를 막아 매번 메모리에서 값을 읽도록 강제할 뿐, Race Condition을 막아주지 않습니다. 멀티스레드 동기화는 mutex나 `std::atomic`을 써야 합니다."

---

## 13. 셀프 체크

- [ ] Mutex의 Lock/Unlock 동작을 그림으로 설명할 수 있다.
- [ ] `FCriticalSection`을 `FScopeLock` 없이 직접 쓰면 왜 위험한지 안다.
- [ ] 락 범위를 좁게 잡아야 하는 이유를 안다.
- [ ] Mutex, RWLock, Semaphore, Spinlock의 용도를 구분할 수 있다.
- [ ] `volatile`이 동기화 도구가 아니라는 걸 안다.
- [ ] 락을 잡고 있을 때 같은 스레드가 또 잡으면 어떤 일이 생기는지 안다.

---

다음 강의: **9강 — Atomic과 Lock-Free의 맛보기**
"락 없이 멀티스레드 안전을? `std::atomic`, 메모리 배리어, 그리고 환상은 깨주기."
