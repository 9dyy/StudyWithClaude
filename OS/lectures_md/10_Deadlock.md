# 10강 — 데드락 (Deadlock)

> **목표:** 데드락이 왜 생기는지, 4가지 조건, 예방/회피/탐지/복구 전략, 그리고 실무에서 데드락을 디버깅하는 법을 안다.
> **선수 지식:** 8강 (Mutex).

> **Part 3 동시성의 마지막 강의.** 면접에서 거의 항상 한 번은 묻는 주제다.

---

## 1. 도입 — 게임이 갑자기 멈춘다

상황: 새로 만든 인벤토리 시스템에서 가끔 게임이 영구히 멈춘다. 크래시도 안 나고, CPU 사용률도 0. 그냥 정지.

코드:

```cpp
FCriticalSection InventoryLock;
FCriticalSection EquipLock;

void Equip(int32 ItemId)
{
    FScopeLock G1(&InventoryLock);   // 1. 인벤토리 락
    FScopeLock G2(&EquipLock);       // 2. 장비 락
    MoveFromInventoryToEquip(ItemId);
}

void Unequip(int32 ItemId)
{
    FScopeLock G1(&EquipLock);       // 1. 장비 락 ← 순서 다름!
    FScopeLock G2(&InventoryLock);   // 2. 인벤토리 락
    MoveFromEquipToInventory(ItemId);
}
```

스레드 A가 `Equip` 호출, 스레드 B가 동시에 `Unequip` 호출:

```
시각      스레드 A                       스레드 B
────────────────────────────────────────────────────
t1       InventoryLock 잡음 ✓
t2                                       EquipLock 잡음 ✓
t3       EquipLock 기다림...              InventoryLock 기다림...
t4       (영원히)                         (영원히)
```

둘 다 상대가 가진 락을 기다린다. **아무도 안 풀어준다.** 게임 정지.

이게 **Deadlock(데드락, 교착 상태)** 다.

---

## 2. 데드락의 4가지 필요 조건 (Coffman Conditions)

**다 만족해야** 데드락이 가능. 하나라도 없으면 데드락 X.

### (1) 상호 배제 (Mutual Exclusion)

자원이 한 번에 한 스레드만 점유 가능. (락의 본질이라 이건 못 없앰)

### (2) 점유 대기 (Hold and Wait)

스레드가 자원 하나를 들고서 다른 자원을 기다린다.
위 예: A가 InventoryLock을 들고 EquipLock 기다림.

### (3) 비선점 (No Preemption)

다른 스레드가 가진 락을 강제로 뺏을 수 없다.
OS가 "A 너 락 토해" 못 함.

### (4) 순환 대기 (Circular Wait)

스레드들이 자원을 기다리는 그래프에 사이클이 있다.
A → B → A. 또는 A → B → C → A.

> 면접 단골: "데드락 발생 조건 4가지". 외워라.
> **Mutual Exclusion, Hold and Wait, No Preemption, Circular Wait.**

---

## 3. 해결 전략 4가지

데드락에 대처하는 방법은 OS 교과서 분류로 4가지.

### 전략 1: 예방 (Prevention)

4가지 조건 중 **하나를 처음부터 못 만들게** 한다.

#### (a) 점유 대기 깨기 — "한 번에 다 잡거나, 아예 안 잡거나"

```cpp
// 두 락을 한 번에 잡는 도구 (C++17)
std::lock(InventoryLock, EquipLock);   // 둘 다 잡힐 때까지 기다림. 부분 점유 없음.
std::lock_guard<std::mutex> G1(InventoryLock, std::adopt_lock);
std::lock_guard<std::mutex> G2(EquipLock, std::adopt_lock);
```

언리얼엔 직접 대응이 없어서 수동으로 구현하거나, 다음 (b) 방법을 쓴다.

#### (b) 순환 대기 깨기 — "락 잡는 순서를 전역으로 고정" ← **실무 표준**

```cpp
// 규칙: InventoryLock을 EquipLock보다 항상 먼저 잡는다.
void Equip(int32 ItemId)
{
    FScopeLock G1(&InventoryLock);   // 항상 Inventory 먼저
    FScopeLock G2(&EquipLock);
    ...
}

void Unequip(int32 ItemId)
{
    FScopeLock G1(&InventoryLock);   // 똑같이 Inventory 먼저
    FScopeLock G2(&EquipLock);
    ...
}
```

**락 순서를 전역적으로 한 방향으로 고정**하면 사이클이 생길 수 없다. 가장 단순하고 효과적.

규모가 커지면 락마다 "레벨"을 부여하고 낮은 레벨부터 잡는 규칙(Lock Hierarchy)을 둔다.

#### (c) 비선점 깨기 — `try_lock` 사용

```cpp
if (InventoryLock.TryLock())
{
    if (EquipLock.TryLock())
    {
        // 둘 다 잡음 → 작업
        EquipLock.Unlock();
        InventoryLock.Unlock();
    }
    else
    {
        InventoryLock.Unlock();   // 둘째 못 잡으면 첫째도 놔주고 재시도
        // 잠깐 쉬고 다시
    }
}
```

데드락은 안 생기지만 **livelock**(서로 양보만 계속) 위험. 잘 안 쓴다.

### 전략 2: 회피 (Avoidance)

요청이 들어오면 "이거 허락하면 데드락 날 수도 있나?" 미리 분석. **Banker's Algorithm**.

> 학교에서 많이 가르치지만 **실무에선 거의 안 씀.** 모든 스레드의 미래 자원 요청을 알아야 해서 비현실적. 면접에서 "Banker's Algorithm" 이름만 알면 됨.

### 전략 3: 탐지 후 복구 (Detection & Recovery)

일단 데드락 나게 두고, OS가 주기적으로 자원 그래프를 검사해서 사이클 찾으면:
- 한 스레드를 강제 종료
- 또는 일부 자원을 빼앗아 줌(rollback)

**DB(데이터베이스)에서 많이 쓴다.** 트랜잭션 데드락 발생 시 한 쪽을 abort.
OS 차원에서는 잘 안 함.

### 전략 4: 무시 (Ignorance) — "타조 알고리즘"

"데드락 날 일 거의 없으니까 신경 안 씀." **유닉스, 리눅스 커널의 일부 자원이 실제로 이렇게 함.** 그냥 멈추면 재부팅.

게임 클라이언트는 무시하면 안 됨. 데드락 = 게임 멈춤 = 환불.

---

## 4. 게임 개발자의 현실: "락 순서 규칙" 이 답

실무에서 99% 쓰는 건 **전략 1-(b): 락 순서 고정.**

규칙:
1. **락이 여러 개 있는 모듈에선 락 순서를 문서화한다.**
2. **모든 코드가 그 순서대로 잡는다.**
3. **새 락을 추가할 땐 기존 순서 어디에 끼울지 결정한다.**

예:
```
프로젝트 락 순서 (위→아래):
1. GameStateLock
2. PlayerLock
3. InventoryLock
4. EquipLock
5. LogLock
```

이렇게 정해놓고 위에서 아래로만 잡으면 데드락 없음.

언리얼이 내부적으로 비슷한 규칙이 있다. UObject 시스템은 GC와 GameThread 간 락 순서가 엄격히 정해져 있어서, 워커에서 함부로 락 잡으면 데드락 위험.

---

## 5. 실무 데드락 디버깅

### 증상

- 게임이 영구히 멈춤. 크래시 X.
- CPU 사용률 낮음 (다 자고 있어서).
- 마우스/키보드 입력은 받는데 게임 화면이 안 움직임.

### 진단

1. **Visual Studio 디버거 일시정지(Break All).**
2. **스레드 창**에서 모든 스레드의 콜스택 확인.
3. `WaitForSingleObject`, `RtlEnterCriticalSection`, `pthread_mutex_lock` 등에서 멈춰있는 스레드들을 찾는다.
4. 그 스레드들이 **서로의 락을 기다리고 있으면** 데드락 확정.

### 도구

- **WinDbg**의 `!locks` 명령
- 언리얼의 `stat thread`, `stat unitgraph`
- 정적 분석: clang의 ThreadSanitizer (TSan) — 락 순서 위반 자동 검출

---

## 6. 데드락의 친척들

### Livelock

스레드들이 서로 양보만 하느라 진척이 없는 상태. 데드락처럼 멈춰있진 않은데, **유의미한 일을 못 함.**

```cpp
// 둘 다 양보만 함
if (!Lock.TryLock())
{
    Sleep(1);
    return;   // 양보
}
```

### Starvation (기아)

특정 스레드가 우선순위에 밀려 영원히 자원 못 받는 상황. 우선순위 스케줄링의 부작용.

해결: **Aging** — 오래 기다린 스레드의 우선순위를 올려줌.

### Priority Inversion

6강에서 다룸. 낮은 우선순위가 락 잡고 있어서 높은 우선순위가 못 도는 현상. 해결책은 Priority Inheritance.

---

## 7. 코드로 — 데드락 만들기 + 고치기

### 데드락 발생 코드

```cpp
FCriticalSection LockA;
FCriticalSection LockB;

// 스레드 1
void T1Work()
{
    FScopeLock G1(&LockA);
    FPlatformProcess::Sleep(0.01f);  // 컨텍스트 스위칭 유도
    FScopeLock G2(&LockB);
    // ...
}

// 스레드 2
void T2Work()
{
    FScopeLock G1(&LockB);
    FPlatformProcess::Sleep(0.01f);
    FScopeLock G2(&LockA);
    // ...
}
```

두 스레드 같이 돌리면 거의 매번 멈춤.

### 수정: 락 순서 고정

```cpp
// 규칙: LockA → LockB 순서

void T1Work()
{
    FScopeLock G1(&LockA);
    FScopeLock G2(&LockB);
}

void T2Work()
{
    FScopeLock G1(&LockA);   // ← B 먼저 잡지 말고 A부터
    FScopeLock G2(&LockB);
}
```

이제 데드락 안 남.

---

## 8. 흔한 오해

### 오해 1: "락 적게 쓰면 데드락 안 생긴다"

락 하나만 써도 **재진입 데드락**(같은 스레드가 자기가 잡은 락을 다시 잡으려 함)이 가능. 또 락 외에 세마포어, 이벤트, I/O 대기도 데드락의 원인이 될 수 있음.

### 오해 2: "데드락은 락 두 개부터 가능"

자원 종류만 다양하면 락 하나로도 데드락. 또 락이 아닌 다른 동기화 객체끼리도 가능.
예: A는 락 잡고 이벤트 기다림, B는 이벤트 트리거 하기 전에 그 락 잡으려 함.

### 오해 3: "Try-lock 쓰면 데드락 끝"

데드락은 안 남. 대신 **livelock** 또는 **starvation** 위험. 그리고 잠깐 못 잡으면 안 되는 작업이면 의미 없음.

### 오해 4: "Lock-Free면 데드락 없다"

맞다. Atomic이나 lock-free 자료구조는 락이 아예 없으니 데드락 자체가 없음. 하지만 **livelock**과 retry 폭주는 가능.

### 오해 5: "GameThread는 단일이라 데드락 안 남"

GameThread 안에서만 도는 코드는 안 나지만, GameThread가 **워커가 잡은 락을 기다리고**, 워커가 **GameThread의 응답(예: ENQUEUE_RENDER_COMMAND 완료)을 기다리면** 데드락.
실제로 언리얼 초보 흔한 실수.

---

## 9. 면접에서 나오면

**Q1. 데드락이 뭔가요?**
→ "두 개 이상의 스레드가 서로 상대가 가진 자원을 기다리면서 영원히 진척하지 못하는 상태입니다. 예를 들어 스레드 A가 락1을 잡고 락2를 기다리고, 스레드 B가 락2를 잡고 락1을 기다리면 둘 다 영원히 멈춥니다."

**Q2. 데드락 발생의 4가지 조건은?**
→ "Mutual Exclusion(자원은 한 번에 한 스레드만), Hold and Wait(자원을 들고 다른 자원을 기다림), No Preemption(자원을 강제로 뺏을 수 없음), Circular Wait(자원 요청에 사이클이 존재함). 네 가지가 다 만족돼야 데드락이 가능합니다."

**Q3. 데드락 해결 방법은?**
→ "크게 4가지가 있습니다. 예방(4조건 중 하나를 못 만들게 함), 회피(요청 시 안전성 검사 - Banker's Algorithm), 탐지 후 복구(주기적 검사, DB 트랜잭션에서 많이 씀), 무시(타조 알고리즘). 실무에선 예방, 특히 락 순서를 전역으로 고정해서 순환 대기를 깨는 방법을 가장 많이 씁니다."

**Q4. 실무에서 데드락을 어떻게 피하시겠어요?**
→ "락에 전역 순서를 부여하고 모든 코드가 그 순서대로 잡게 합니다. 락 순서를 문서화하고, 락이 추가되면 어디 위치에 끼울지 결정합니다. 락 잡은 채로 무거운 일이나 다른 스레드의 응답 대기를 하지 않습니다. 락 안에서 콜백을 호출하지 않습니다 — 콜백 안에서 어떤 락을 잡을지 모르니까요."

**Q5. 데드락이 의심될 때 어떻게 디버깅하나요?**
→ "디버거를 일시정지하고 모든 스레드의 콜스택을 확인합니다. WaitForSingleObject나 mutex lock에서 멈춘 스레드들이 서로의 락을 기다리고 있는지 추적합니다. 평소엔 ThreadSanitizer나 정적 분석으로 락 순서 위반을 사전에 잡습니다."

**Q6. Deadlock, Livelock, Starvation의 차이는?**
→ "Deadlock은 서로 자원을 기다리며 영원히 멈춤. Livelock은 양보만 반복하며 멈추진 않지만 진척이 없음. Starvation은 우선순위에 밀려 특정 스레드만 자원을 못 받는 상태입니다. 셋 다 결과는 비슷하지만 원인과 해결책이 다릅니다 — Starvation은 Aging으로 해결합니다."

**Q7. 락 두 개 잡을 때 어떤 순서로 잡아야 하나요?**
→ "프로젝트 전역에 락 순서 규약을 정하고, 모든 코드가 동일한 순서로 잡습니다. 예를 들어 InventoryLock을 EquipLock보다 항상 먼저 잡는 규칙을 정하면, 한 쪽에서 역순으로 잡는 일이 없어 순환 대기가 생기지 않습니다."

---

## 10. 셀프 체크

- [ ] 데드락의 4가지 조건을 외워서 말할 수 있다.
- [ ] 해결 전략 4가지(예방, 회피, 탐지, 무시)를 안다.
- [ ] 실무에서 가장 많이 쓰는 해결책이 "락 순서 고정"이라는 걸 안다.
- [ ] 데드락, Livelock, Starvation을 구분할 수 있다.
- [ ] 데드락 디버깅 방법(디버거로 콜스택 확인)을 안다.
- [ ] 락 두 개 데드락 시나리오를 의사 코드로 그릴 수 있다.

---

## Part 3 마무리

7~10강을 통해 동시성의 핵심을 다 봤다.

| 강의 | 주제 | 한 줄 요약 |
|------|------|-----------|
| 7강 | Race Condition | 공유 + 쓰기 = 위험. `Counter++` 도 3개 명령. |
| 8강 | Mutex | 임계영역에 자물쇠. `FScopeLock` 무조건 써라. |
| 9강 | Atomic / Lock-Free | 한 변수면 atomic이 mutex보다 빠르다. 만능 아님. |
| 10강 | Deadlock | 락 순서를 전역으로 고정해라. 끝. |

면접에서 "멀티스레드 동기화 경험 있어요?" 들으면 이 4강의 내용을 묶어서 답하면 된다.

---

다음 강의: **11강 — 가상 메모리와 페이징** (Part 4 메모리 시작)
"각 프로세스가 똑같이 0x00000000부터 주소를 쓰는 마술의 비밀."
