# 7강 — Object Pooling

> **목표:** 매 프레임 `new`/`delete`를 반복하면 왜 프레임이 끊기는지 메모리 단편화 관점까지 설명하고, Object Pooling으로 이 문제를 어떻게 해결하는지 실제 코드로 구현할 수 있게 된다.
> **선수 지식:** 3강(RAII), 4강(배열과 캐시 지역성).

---

## 1. 총알 하나 쏠 때마다 게임이 살짝 끊긴다

기관총을 쏘는 무기를 만들었다. 총알을 쏠 때마다 스폰하고, 맞거나 수명이 다하면 파괴한다.

```cpp
void AWeapon::Fire()
{
    ABullet* NewBullet = GetWorld()->SpawnActor<ABullet>(BulletClass, GetMuzzleLocation(), GetMuzzleRotation());
    NewBullet->Init(Damage, Speed);
}

void ABullet::OnLifetimeExpired()
{
    Destroy();   // 매번 액터를 파괴 — 내부적으로 메모리 해제 발생
}
```

초당 10발씩 쏘는 무기 몇 개가 화면에 있으면 프레임이 미세하게 끊기기 시작한다. 프로파일러를 열어보면 `SpawnActor`와 `Destroy` 근처에서 시간이 많이 든다. **총알 하나 만들고 없애는 게 왜 이렇게 비쌀까?**

---

## 2. new/delete가 매 프레임 비싼 이유 — 그냥 "느린 함수 호출"이 아니다

`new`는 단순히 "메모리 한 칸 떼주는" 게 아니다. 힙 메모리 할당자(allocator)에게 **"이만한 크기의 빈 공간 어디 없나"를 찾아달라고 요청**하는 과정이다.

```
힙 메모리 상태 (총알을 만들고 부수기를 반복한 뒤)

┌────┬──────┬────┬────────┬──┬───────┬────┐
│사용중│  빈공간 │사용중│  빈공간   │사용│  빈공간 │사용중│
└────┴──────┴────┴────────┴──┴───────┴────┘
  ↑ 이렇게 사용중/빈공간이 조각조각 흩어진 상태 = 메모리 단편화(Fragmentation)
```

`new`/`delete`를 반복하면 힙 안에 **사용 중인 블록과 빈 블록이 조각조각 흩어지는 메모리 단편화**가 생긴다. 빈 공간을 다 합치면 충분한데, 조각나 있어서 "연속된 큰 블록"을 못 찾는 상황까지 갈 수 있다. 할당자는 이 조각난 공간들 중 딱 맞는 자리를 찾느라 매번 탐색 비용을 쓴다. 게다가 **`delete`는 소멸자 호출 + 메모리 반납**을 하고, 언리얼의 `AActor::Destroy()`는 여기에 더해 월드에서 액터를 등록 해제하는 부가 작업까지 얹혀서 단순 `delete`보다 훨씬 무겁다.

여기에 4강에서 본 캐시 지역성 문제까지 겹친다. 매번 새로 할당된 총알은 힙의 **예측 불가능한 위치**에 생기니, 총알 100개를 순회하며 업데이트할 때도 캐시 미스가 반복된다. "총알 하나 쏘는 것"의 비용이 예상보다 훨씬 큰 이유가 여기 다 모여있다.

---

## 3. Object Pooling — "새로 만들지 말고 미리 만들어둔 걸 재사용해라"

해법은 단순하다. **총알을 게임 시작할 때 미리 넉넉하게 만들어두고, 안 쓸 땐 숨겨서 재워두고, 필요할 때 깨워서 재사용**한다. `new`/`delete`를 매 발사마다 하는 대신 딱 한 번씩만(풀을 만들 때, 게임이 끝날 때) 한다.

```
게임 시작 시                 발사할 때                    맞거나 수명 끝났을 때
┌─────────────┐            ┌─────────────┐             ┌─────────────┐
│ 비활성 풀    │  꺼내서    │ 활성 상태   │  다시 넣고   │ 비활성 풀     │
│ [B1][B2]... │ ────────→  │ 화면에 표시  │ ────────→   │ [B1][B3]...  │
│ [B3][B4]... │  활성화    │ 로직 작동    │  비활성화    │ (재사용 대기) │
└─────────────┘            └─────────────┘             └─────────────┘
     ↑ delete 없이 계속 순환한다 — 실제 파괴는 게임/레벨 종료 시 한 번만
```

```cpp
class ABulletPool
{
public:
    void Initialize(UWorld* World, TSubclassOf<ABullet> BulletClass, int32 PoolSize)
    {
        for (int32 i = 0; i < PoolSize; ++i)
        {
            ABullet* Bullet = World->SpawnActor<ABullet>(BulletClass);
            Bullet->SetActorHiddenInGame(true);
            Bullet->SetActorTickEnabled(false);
            InactiveBullets.Push(Bullet);   // 5강의 Stack — 꺼내고 넣는 순서가 중요치 않아 O(1)이면 충분
        }
    }

    ABullet* AcquireBullet()
    {
        if (InactiveBullets.Num() == 0)
        {
            UE_LOG(LogTemp, Warning, TEXT("풀이 바닥남 — 크기를 늘리는 걸 고려할 것"));
            return nullptr;   // 혹은 아래 5번 섹션의 확장 전략 적용
        }

        ABullet* Bullet = InactiveBullets.Pop();
        Bullet->SetActorHiddenInGame(false);
        Bullet->SetActorTickEnabled(true);
        return Bullet;
    }

    void ReleaseBullet(ABullet* Bullet)
    {
        Bullet->SetActorHiddenInGame(true);
        Bullet->SetActorTickEnabled(false);
        Bullet->ResetState();   // 데미지, 속도 등 이전 사용 흔적 초기화
        InactiveBullets.Push(Bullet);
    }

private:
    TArray<ABullet*> InactiveBullets;   // 4강 — 배열이라 캐시 지역성도 챙김
};
```

```cpp
// 무기 코드는 이제 new/delete를 전혀 신경 쓰지 않는다
void AWeapon::Fire()
{
    ABullet* Bullet = BulletPool->AcquireBullet();
    if (Bullet)
    {
        Bullet->Init(Damage, Speed);
        Bullet->SetActorLocationAndRotation(GetMuzzleLocation(), GetMuzzleRotation());
    }
}

void ABullet::OnLifetimeExpired()
{
    OwningPool->ReleaseBullet(this);   // Destroy() 대신 풀에 반납
}
```

핵심은 **"객체의 생성 비용을 게임플레이 도중이 아니라 로딩 시점으로 옮긴다"**는 것이다. 로딩 화면에서 몇 초 더 걸리는 건 플레이어가 못 느끼지만, 전투 중 프레임 드랍은 바로 느낀다.

---

## 4. 3강 RAII와의 관계 — 자동 반납은 어떻게 시킬까

3강에서 본 RAII를 여기 적용하면 "반납을 깜빡하는" 실수를 줄일 수 있다. 총알이 스코프를 벗어날 때(혹은 특정 이벤트가 끝날 때) 자동으로 풀에 반납되도록 감싸는 것도 가능하다.

```cpp
class FScopedPooledBullet
{
public:
    FScopedPooledBullet(ABulletPool* InPool) : Pool(InPool)
    {
        Bullet = Pool->AcquireBullet();
    }
    ~FScopedPooledBullet()
    {
        if (Bullet)
        {
            Pool->ReleaseBullet(Bullet);   // 소멸자에서 자동 반납 — ReleaseBullet 깜빡할 일 없음
        }
    }
    ABullet* Get() const { return Bullet; }

private:
    ABulletPool* Pool;
    ABullet* Bullet;
};
```

실제 총알처럼 "발사 후 한동안 살아있다가 알아서 끝나는" 경우엔 위 방식보다 앞서 본 `OnLifetimeExpired`에서 명시적으로 반납하는 쪽이 자연스럽지만, "이번 함수 안에서만 잠깐 쓰고 바로 반납할" 임시 오브젝트(예: 파티클 하나 잠깐 재생)에는 RAII 래퍼가 유용하다.

---

## 5. 풀이 바닥나면 — 확장 전략

풀 크기를 고정하면 "동시에 총알이 그 이상 필요한 순간"에 문제가 생긴다. 실전에서는 보통 두 방향으로 대응한다.

```cpp
ABullet* AcquireBullet()
{
    if (InactiveBullets.Num() == 0)
    {
        // 전략 1: 필요할 때 추가로 만든다 (동적 확장) — 순간적인 스파이크는 처리되지만
        //         그 순간엔 결국 new 비용을 다시 치른다.
        ABullet* NewBullet = GetWorld()->SpawnActor<ABullet>(BulletClass);
        return NewBullet;

        // 전략 2: 가장 오래전에 활성화된 것을 강제로 회수해서 재사용한다.
        //         (화면 밖으로 나간 지 오래된 총알을 우선 회수하는 방식 등)
    }

    return InactiveBullets.Pop();
}
```

**어느 쪽이든 "풀 크기를 처음부터 넉넉하게, 실측 기반으로 잡는 것"이 먼저다.** 확장 전략은 예외 상황을 위한 안전망이지, 풀 크기 설계를 대충 해도 되는 핑계가 아니다.

---

## 6. 여러 스레드가 같은 풀을 공유한다면 — Race Condition 주의

멀티스레드 렌더링이나 비동기 스폰 로직에서 여러 스레드가 **같은 풀 인스턴스**에 동시에 `AcquireBullet`/`ReleaseBullet`을 호출할 수 있다. 이 경우 `InactiveBullets`(내부적으로 `TArray`)를 두 스레드가 동시에 `Pop`/`Push` 하면, OS 시리즈 7강에서 다룬 **Race Condition**이 그대로 재현된다.

```cpp
// 스레드 A와 스레드 B가 동시에 이 함수를 호출하면
ABullet* AcquireBullet()
{
    if (InactiveBullets.Num() == 0) return nullptr;
    return InactiveBullets.Pop();   // 두 스레드가 동시에 같은 총알을 꺼내갈 수 있음(!)
}
```

두 스레드가 동시에 `Num() == 0`을 확인하고 통과한 뒤 거의 동시에 `Pop()`을 호출하면, 배열 내부 인덱스 조작이 겹쳐서 **같은 총알을 두 스레드가 동시에 소유하게 되거나, 배열 자체가 깨지는** 문제로 이어질 수 있다. 대응 방법은 OS 시리즈에서 이미 다룬 것과 동일하다.

```cpp
ABullet* AcquireBullet()
{
    FScopeLock Lock(&PoolCriticalSection);   // 3강 RAII 패턴 그대로 — 락도 스코프 벗어나면 자동 해제
    if (InactiveBullets.Num() == 0) return nullptr;
    return InactiveBullets.Pop();
}
```

**대부분의 게임 로직은 메인 스레드(게임 스레드)에서만 도니까 이 문제를 신경 쓸 필요가 없는 경우가 많다.** 하지만 물리 콜백, 비동기 로딩 완료 콜백처럼 다른 스레드에서 풀에 접근할 가능성이 있다면, "누가 이 함수를 어느 스레드에서 부를 수 있는가"를 먼저 확인하고 필요할 때만 락을 넣는 게 맞다 — 항상 락을 걸면 단일 스레드 환경에서도 불필요한 비용이 생긴다.

---

## 7. 흔한 오해

### 오해 1: "Object Pooling은 총알처럼 자주 생성되는 것에만 쓴다"

가장 흔한 예가 총알/이펙트일 뿐, **짧은 생명주기로 자주 생성/파괴되는 모든 것**(파티클, 사운드 큐, UI 위젯, 적 AI의 임시 투사체)이 후보다. 판단 기준은 "생성 빈도가 높고 생명주기가 짧은가"다.

### 오해 2: "풀링하면 메모리를 절약할 수 있다"

오히려 반대다. 풀은 **"최대로 필요할 만한 개수를 미리 다 잡아두는" 것**이라 평소엔 안 쓰는 메모리도 계속 점유한다. 목적은 메모리 절약이 아니라 **런타임 중 할당/해제 비용과 단편화를 없애는 것**이다.

### 오해 3: "풀에서 꺼낸 오브젝트는 상태 초기화를 신경 안 써도 된다"

아니다. 이전에 쓰던 흔적(데미지 값, 속도, 타이머)이 그대로 남아있으면 재사용 시 버그가 난다. `AcquireBullet`/`ReleaseBullet` 어딘가에서 반드시 상태를 리셋해야 한다 — 위 코드의 `ResetState()`가 이 역할이다.

### 오해 4: "가비지 컬렉션이 있는 언어는 Object Pooling이 필요 없다"

C++/언리얼도 `UObject`는 가비지 컬렉터가 관리하지만, GC가 있다고 "할당/해제 비용"이 사라지는 건 아니다. GC 언어에서도 짧은 생명주기 객체를 반복 생성하면 GC 부하 자체가 프레임 드랍의 원인이 될 수 있어서, Object Pooling은 GC 유무와 무관하게 유효한 최적화다.

### 오해 5: "풀 크기는 일단 크게 잡아두면 안전하다"

너무 크게 잡으면 안 쓰는 메모리를 낭비하고, 초기화 시간(로딩 시점에 다 미리 스폰)도 늘어난다. 실측(최대 동시 활성 개수)을 기반으로 여유를 살짝 두는 정도가 적절하다.

### 오해 6: "풀은 게임 스레드에서만 쓰니까 동시 접근 걱정은 항상 없다"

대부분의 게임플레이 코드는 맞지만, 비동기 콜백(물리, 로딩 완료 등)에서 같은 풀에 접근할 수 있는 구조라면 얘기가 다르다. "이 풀에 접근하는 코드가 정말 한 스레드에서만 실행되는가"를 확인 없이 가정하면 Race Condition을 놓치기 쉽다.

---

## 8. 면접에서 나오면

**Q1. Object Pooling에 대해 설명해주세요.**
→ "자주 생성되고 짧게 살다 파괴되는 오브젝트를 매번 new/delete 하지 않고, 미리 만들어둔 풀에서 꺼내 쓰고 다시 반납하는 패턴입니다. 게임 시작 시 필요한 만큼 미리 생성해두고, 필요할 때 활성화, 다 쓰면 비활성화해서 재사용합니다."

**Q2. 매 프레임 new/delete를 반복하면 왜 성능 문제가 생기나요?**
→ "new는 힙 할당자가 적절한 빈 공간을 찾는 탐색 비용이 있고, 반복하면 사용 중인 블록과 빈 블록이 조각나는 메모리 단편화가 생겨서 할당이 점점 비싸집니다. 게다가 매번 새로 할당된 메모리는 위치가 제각각이라 캐시 지역성도 나빠집니다."

**Q3. Object Pooling이 메모리를 절약해주나요?**
→ "아닙니다. 오히려 최대로 필요할 만한 개수를 미리 확보해두기 때문에 평소엔 쓰지 않는 메모리도 계속 점유합니다. 목적은 메모리 절약이 아니라 런타임 중 할당/해제 비용과 단편화를 없애는 것입니다."

**Q4. 풀에서 꺼낸 오브젝트를 재사용할 때 놓치기 쉬운 부분은 뭔가요?**
→ "이전 사용 흔적을 초기화하는 걸 빠뜨리는 겁니다. 데미지 값이나 타이머 같은 상태가 남아있으면 재사용했을 때 예상과 다른 동작을 하는 버그로 이어져서, 반납하거나 꺼낼 때 반드시 상태를 리셋해야 합니다."

**Q5. 풀이 바닥나면 어떻게 처리하시겠어요?**
→ "동시에 필요한 최대 개수를 실측해서 풀 크기를 처음부터 넉넉하게 잡는 게 먼저입니다. 그래도 순간적으로 부족한 경우엔 그 순간만 추가로 생성하거나, 가장 오래된 활성 오브젝트를 강제로 회수하는 전략을 쓸 수 있습니다."

**Q6. 매 프레임 생성되는 게 아니라 게임 시작할 때 딱 한 번만 만드는 오브젝트에도 Object Pooling이 필요할까요?**
→ "필요 없습니다. Object Pooling은 생성 빈도가 높고 생명주기가 짧은 오브젝트에 의미가 있는 최적화라, 한 번만 만들고 계속 쓰는 오브젝트는 애초에 할당/해제 비용이 반복되지 않아서 풀링할 이유가 없습니다."

**Q7. 언리얼은 UObject를 가비지 컬렉터로 관리하는데, 그래도 Object Pooling이 필요한가요?**
→ "필요합니다. GC가 메모리 해제를 자동으로 해주긴 하지만, 짧은 생명주기 객체를 반복 생성하는 자체의 할당 비용과 GC가 처리해야 할 대상이 늘어나는 부하는 그대로입니다. GC 유무와 무관하게 반복 생성/파괴 패턴 자체를 줄이는 게 Object Pooling의 목적입니다."

**Q8. 여러 스레드가 같은 풀을 동시에 사용할 수 있다면 어떤 문제가 생기고, 어떻게 막나요?**
→ "InactiveBullets 같은 내부 컨테이너를 여러 스레드가 동시에 Pop/Push하면 Race Condition이 발생해서 같은 오브젝트를 두 스레드가 동시에 소유하거나 컨테이너 자체가 깨질 수 있습니다. FScopeLock 같은 RAII 락으로 AcquireBullet/ReleaseBullet 구간을 보호해서 막을 수 있는데, 실제로 여러 스레드에서 호출될 가능성이 있는 경우에만 락을 넣는 게 맞습니다."

면접관이 그 다음에 던지기 좋은 질문: *"풀링된 오브젝트가 비활성 상태에서 활성 상태로 바뀔 때 다른 시스템(UI, 사운드)에 이 변화를 알려야 한다면 어떻게 설계하시겠어요?"* → 8강(Observer Pattern)으로 이어지는 질문이다.

---

## 9. 셀프 체크

- [ ] new/delete 반복이 왜 비싼지 메모리 단편화와 캐시 지역성 관점에서 설명할 수 있다.
- [ ] Object Pooling의 동작 흐름(미리 생성 → 활성화/비활성화 → 재사용)을 코드로 설명할 수 있다.
- [ ] Object Pooling이 메모리 절약이 아니라 할당/해제 비용 제거가 목적이라는 걸 안다.
- [ ] 풀에서 꺼낸 오브젝트의 상태 초기화를 왜 반드시 챙겨야 하는지 안다.
- [ ] 풀이 바닥났을 때의 대응 전략(동적 확장, 강제 회수)을 설명할 수 있다.
- [ ] 여러 스레드가 같은 풀을 공유할 때 Race Condition이 왜, 어떻게 생기는지 설명할 수 있다.

---

다음 강의: **8강 — Observer Pattern**
"HP가 바뀔 때마다 UI가 어떻게 자동으로 갱신되는지, 언리얼 Delegate/Event로 느슨한 결합을 어떻게 만드는지"를 다룬다.
