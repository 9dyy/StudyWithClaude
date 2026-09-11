# 9강 — Singleton과 Factory

> **목표:** 전통적인 Singleton 구현 방식과 그 문제점을 알고, 언리얼의 GameInstance/Subsystem이 왜 그 문제를 해결한 대안인지 설명할 수 있게 된다. 팩토리로 액터를 생성하는 이유도 판단할 수 있게 된다.
> **선수 지식:** 1강(static — Meyer's Singleton), 8강(Observer Pattern).

---

## 1. 전역 사운드 매니저를 여기저기서 new 하다가 생긴 혼란

배경음악과 효과음을 관리하는 `SoundManager`가 필요했다. 여러 시스템(UI, 전투, 대화)에서 각자 필요할 때 사운드를 재생해야 해서, 처음엔 각 시스템이 자기 `SoundManager` 인스턴스를 만들어 썼다.

```cpp
class UMenuWidget : public UUserWidget
{
    USoundManager* SoundManager = NewObject<USoundManager>();   // 여기서도 하나
};

class ACombatSystem : public AActor
{
    USoundManager* SoundManager = NewObject<USoundManager>();   // 여기서도 또 하나
};
```

인스턴스가 여러 개 생기니 "지금 재생 중인 사운드 목록"이 인스턴스마다 따로 관리돼서, 전투 시스템이 재생한 사운드를 메뉴 시스템이 끌 수가 없다. **딱 하나만 존재해야 하는 게 명확한 대상**인데 여러 개가 생겨버린 거다. 이럴 때 쓰는 게 **Singleton 패턴** — "이 클래스의 인스턴스는 프로그램 전체에서 딱 하나만 존재하게 강제한다."

---

## 2. 전통적인 Singleton — Meyer's Singleton (1강 복습)

1강에서 이미 본 내용이다. 지역 static 변수를 이용하면 "최초 호출 때 딱 한 번만 생성"되는 걸 활용해 Singleton을 만들 수 있다.

```cpp
class FSoundManager
{
public:
    static FSoundManager& Get()
    {
        static FSoundManager Instance;   // 최초 호출 때 딱 한 번 생성. C++11부터 초기화 자체는 스레드 안전
        return Instance;
    }

    void PlaySound(USoundBase* Sound) { /* ... */ }

private:
    FSoundManager() = default;                              // 생성자를 private으로 막아 외부에서 new 못하게
    FSoundManager(const FSoundManager&) = delete;            // 복사도 막아야 진짜 "하나"가 보장됨
    FSoundManager& operator=(const FSoundManager&) = delete;
};

// 사용하는 쪽 — 어디서든 항상 같은 인스턴스에 접근
FSoundManager::Get().PlaySound(ExplosionSound);
```

**생성자를 `private`으로 막고, 복사 생성자/대입 연산자까지 `delete`로 막는 게 핵심이다.** 이걸 빼먹으면 `FSoundManager Copy = FSoundManager::Get();`처럼 실수로 복사본을 만들어서 "딱 하나"라는 보장이 깨질 수 있다.

---

## 3. 이 방식이 언리얼에서 골치 아픈 이유

전통적인 Singleton은 순수 C++에서는 잘 동작하지만, 언리얼 프레임워크 안에서는 몇 가지 문제가 생긴다.

| 문제 | 설명 |
|------|------|
| 레벨/월드 전환과 생명주기가 안 맞음 | Singleton은 프로그램이 끝날 때까지 살아있는데, 사운드 매니저는 "이 플레이 세션 동안만" 살아야 할 수도 있음 |
| 언리얼 객체 시스템(GC, 리플렉션)과 안 맞음 | `UObject`가 아니라서 블루프린트 노출, 가비지 컬렉션 추적이 안 됨 |
| 테스트/PIE(에디터 내 플레이)에서 상태가 안 초기화됨 | 에디터에서 플레이를 여러 번 실행해도 static 인스턴스는 그대로 남아있어서 이전 세션 상태가 남을 수 있음 |
| 유닛 테스트가 어려움 | 전역 상태라서 테스트마다 깨끗한 상태로 리셋하기 까다로움 |

**요약하면, 전통적인 Singleton은 "프로그램 생명주기"에 묶이는데, 게임은 "월드/세션 생명주기"가 그것과 다르다.** 언리얼은 이 문제를 자기 프레임워크 안에 있는 계층 구조로 해결한다.

---

## 4. GameInstance/Subsystem — "생명주기가 명확한 Singleton"

언리얼은 이미 **생명주기가 명확하게 정의된 전역 객체 계층**을 갖고 있다. 이 계층에 얹으면 "사실상 Singleton"이면서 위 문제들이 다 해결된다.

```
GameInstance          — 게임 실행 시작부터 종료까지 (레벨을 넘나들어도 유지) — 딱 하나만 존재
  └─ GameInstanceSubsystem  — GameInstance와 생명주기를 같이하는 서브시스템

World                 — 레벨 하나가 로드되어 있는 동안만 (레벨 전환 시 새로 생성)
  └─ WorldSubsystem        — World와 생명주기를 같이하는 서브시스템
```

```cpp
UCLASS()
class USoundManagerSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override
    {
        Super::Initialize(Collection);
        UE_LOG(LogTemp, Log, TEXT("SoundManager 초기화 — GameInstance가 생성될 때 자동 호출"));
    }

    virtual void Deinitialize() override
    {
        UE_LOG(LogTemp, Log, TEXT("SoundManager 정리 — GameInstance가 파괴될 때 자동 호출"));
        Super::Deinitialize();
    }

    void PlaySound(USoundBase* Sound) { /* ... */ }
};

// 사용하는 쪽 — 어디서든 GameInstance를 거쳐 딱 하나뿐인 인스턴스에 접근
USoundManagerSubsystem* SoundManager = GetGameInstance()->GetSubsystem<USoundManagerSubsystem>();
SoundManager->PlaySound(ExplosionSound);
```

**"딱 하나만 존재한다"는 Singleton의 본질은 그대로 유지된다** — `GetSubsystem<T>()`는 항상 같은 인스턴스를 리턴한다. 다만 그 "하나"의 생명주기가 프로그램 전체가 아니라 **GameInstance(플레이 세션)나 World(레벨)에 명확히 묶여서**, PIE에서 플레이를 껐다 켜면 자동으로 새로 초기화되고, `UObject`라서 GC 추적과 블루프린트 노출도 공짜로 딸려온다. 8강에서 본 Observer Pattern과 조합하면, Subsystem이 전역 이벤트 허브 역할(예: `OnHPChanged`를 GameInstanceSubsystem에 모아서 여러 레벨에 걸쳐 구독)을 하기도 한다.

---

## 5. Factory Pattern — "생성 로직을 한 곳에 몰아넣기"

Singleton이 "인스턴스 개수"를 통제하는 패턴이라면, Factory는 **"생성 방법과 절차"를 한 곳에 몰아넣는** 패턴이다. 적 스폰 로직을 보자.

```cpp
// Factory 없이 — 스폰하는 곳마다 생성 로직이 흩어짐
void ASpawnPoint::SpawnEnemy(EEnemyType Type)
{
    AEnemy* NewEnemy = nullptr;
    if (Type == EEnemyType::Goblin)
    {
        NewEnemy = GetWorld()->SpawnActor<AGoblin>(SpawnLocation, SpawnRotation);
        NewEnemy->SetHealth(50.f);
        NewEnemy->EquipWeapon(GoblinClub);
    }
    else if (Type == EEnemyType::Orc)
    {
        NewEnemy = GetWorld()->SpawnActor<AOrc>(SpawnLocation, SpawnRotation);
        NewEnemy->SetHealth(120.f);
        NewEnemy->EquipWeapon(OrcAxe);
    }
    // 적 타입이 늘 때마다 이 if-else가 여기저기서 계속 반복됨
}
```

스폰 지점이 여러 곳(웨이브 스포너, 보스룸, 랜덤 인카운터)이면 이 `if-else` 블록이 **여러 파일에 복사돼서 흩어진다.** 오크의 초기 체력을 바꿔야 하면 그 복사본을 전부 찾아 고쳐야 한다.

```cpp
// Factory로 생성 로직을 한 곳에 모으기
class FEnemyFactory
{
public:
    static AEnemy* CreateEnemy(UWorld* World, EEnemyType Type, const FTransform& SpawnTransform)
    {
        switch (Type)
        {
        case EEnemyType::Goblin:
        {
            AGoblin* Goblin = World->SpawnActor<AGoblin>(SpawnTransform.GetLocation(), SpawnTransform.Rotator());
            Goblin->SetHealth(50.f);
            Goblin->EquipWeapon(GoblinClub);
            return Goblin;
        }
        case EEnemyType::Orc:
        {
            AOrc* Orc = World->SpawnActor<AOrc>(SpawnTransform.GetLocation(), SpawnTransform.Rotator());
            Orc->SetHealth(120.f);
            Orc->EquipWeapon(OrcAxe);
            return Orc;
        }
        default:
            return nullptr;
        }
    }
};

// 스폰하는 곳은 이제 "뭘 만들지"만 알면 되고, "어떻게 만들지"는 몰라도 됨
void ASpawnPoint::SpawnEnemy(EEnemyType Type)
{
    AEnemy* NewEnemy = FEnemyFactory::CreateEnemy(GetWorld(), Type, GetActorTransform());
}
```

오크 초기 체력을 바꾸는 요구사항이 오면 `FEnemyFactory::CreateEnemy` **딱 한 곳**만 고치면 된다. 스폰 지점이 몇 개든 상관없다. static 멤버 함수(1강)로 만든 것도 눈여겨볼 부분이다 — Factory는 보통 상태를 안 가지니, 인스턴스 없이 호출 가능한 static 함수로 만드는 게 자연스럽다.

---

## 6. Singleton + Factory를 같이 쓰는 경우

실전에서는 둘을 같이 쓰기도 한다. "적 생성 로직 자체는 Factory로 몰아넣고, 그 Factory를 관리하는 창구는 Subsystem(사실상 Singleton)으로 둔다."

```cpp
UCLASS()
class UEnemySpawnSubsystem : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    AEnemy* SpawnEnemy(EEnemyType Type, const FTransform& SpawnTransform)
    {
        AEnemy* NewEnemy = FEnemyFactory::CreateEnemy(GetWorld(), Type, SpawnTransform);
        OnEnemySpawned.Broadcast(NewEnemy);   // 8강 — 스폰 이벤트를 구독자들에게 알림
        return NewEnemy;
    }

    FOnEnemySpawned OnEnemySpawned;
};
```

Subsystem이 "이 월드에서 적 스폰은 딱 한 창구로만 이뤄진다"는 보장을 주고, Factory가 "실제로 어떻게 만들지"를 담당하고, Delegate가 "만들어졌다는 걸 누구에게 알릴지"를 담당한다 — 세 강의(Singleton/GameInstance, Factory, Observer)가 한 시스템 안에서 자연스럽게 겹치는 예다.

---

## 7. 흔한 오해

### 오해 1: "Singleton은 그냥 static 변수 하나 두는 것과 같다"

비슷해 보이지만 다르다. Singleton은 **"생성자를 막아서 외부에서 직접 인스턴스를 못 만들게 강제"**하는 게 핵심이다. 그냥 `static USoundManager* GlobalManager;`처럼 전역 포인터만 두면, 누군가 실수로 또 다른 인스턴스를 `new`로 만들 수 있어서 "딱 하나"가 보장되지 않는다.

### 오해 2: "언리얼에서는 Singleton을 아예 안 쓴다"

정확히는 "전통적인 방식의 Singleton 클래스"를 잘 안 쓴다는 거고, **GameInstance/Subsystem 자체가 사실상 Singleton 역할**을 한다. "딱 하나만 존재"라는 목적은 똑같이 달성하되, 언리얼 프레임워크의 생명주기 관리에 올라탄 형태로 구현하는 것뿐이다.

### 오해 3: "Factory는 무조건 클래스로 만들어야 한다"

간단한 경우엔 함수 하나로도 충분하다. 위 예시처럼 조건 분기가 몇 가지뿐이면 static 함수 하나로 시작하고, 생성 로직이 복잡해지고 여러 종류의 Factory가 필요해지면 그때 클래스 계층으로 확장해도 늦지 않다.

### 오해 4: "Singleton은 언제나 전역 상태라서 나쁘다"

무조건 나쁜 건 아니다. "이 개념이 게임 안에 진짜로 딱 하나만 존재해야 하는가"(사운드 매니저, 세이브 시스템, 입력 매니저)가 명확하면 적절한 선택이다. 문제는 "일단 편하니까" 아무 클래스나 Singleton으로 만드는 습관이지, 패턴 자체가 나쁜 게 아니다.

### 오해 5: "GetSubsystem을 호출할 때마다 새로 초기화 비용이 든다"

아니다. Subsystem은 GameInstance/World가 생성될 때 딱 한 번 `Initialize`가 호출되고, 이후 `GetSubsystem<T>()`는 이미 만들어진 인스턴스에 대한 포인터를 바로 리턴할 뿐이다.

---

## 8. 면접에서 나오면

**Q1. Singleton 패턴을 C++로 어떻게 구현하나요?**
→ "생성자를 private으로 막고, 복사 생성자와 대입 연산자도 삭제해서 외부에서 직접 인스턴스를 못 만들게 합니다. 정적 멤버 함수 안에 지역 static 객체를 두고 그 참조를 리턴하는 Meyer's Singleton 방식이 흔한데, C++11부터 이 지역 static의 초기화 자체가 스레드 안전하게 보장됩니다."

**Q2. 언리얼에서는 전통적인 Singleton 클래스를 잘 안 쓴다고 하는데, 왜 그런가요?**
→ "전통적인 Singleton은 프로그램 전체 생명주기에 묶이는데, 게임은 레벨/플레이 세션 단위로 생명주기가 다릅니다. UObject가 아니라서 가비지 컬렉션이나 블루프린트 노출도 안 되고, 에디터에서 플레이를 여러 번 돌리면 이전 상태가 남는 문제도 생깁니다."

**Q3. GameInstance나 Subsystem이 Singleton과 어떤 관계가 있나요?**
→ "GetSubsystem으로 가져오는 인스턴스는 항상 같은 하나이기 때문에 사실상 Singleton 역할을 합니다. 다만 생명주기가 GameInstance나 World에 명확히 묶여서 자동으로 초기화/정리되고, UObject라 GC 추적과 블루프린트 노출이 자연스럽게 되는 게 전통적인 Singleton과의 차이입니다."

**Q4. Factory Pattern이 왜 필요한가요?**
→ "객체를 생성하는 로직(어떤 초기값을 주고, 어떤 컴포넌트를 붙이는지)을 여러 곳에 흩어놓지 않고 한 곳에 모아두기 위해서입니다. 생성 방식이 바뀌어야 할 때 그 한 곳만 고치면 되고, 사용하는 쪽은 구체적인 생성 절차를 몰라도 됩니다."

**Q5. 적 종류가 늘어날수록 스폰 로직에 if-else가 계속 늘어난다면 어떻게 개선하시겠어요?**
→ "생성 로직을 Factory 클래스나 함수로 몰아서, 스폰 지점 코드는 어떤 타입을 만들지만 알고 실제 생성 절차는 Factory에 위임하겠습니다. 이러면 새 적 타입이 추가돼도 스폰 지점 코드를 건드릴 필요 없이 Factory 한 곳만 확장하면 됩니다."

**Q6. Singleton을 쓰면 안 좋은 경우는 언제인가요?**
→ "실제로는 여러 인스턴스가 필요할 수도 있는 대상을 성급하게 Singleton으로 만드는 경우입니다. 예를 들어 나중에 멀티플레이어에서 플레이어별로 따로 관리해야 하는 상태를 Singleton으로 박아두면, 구조를 다시 뜯어고쳐야 하는 상황이 생깁니다. '진짜로 하나만 존재해야 하는가'를 먼저 판단해야 합니다."

**Q7. Singleton과 Factory를 같이 쓰는 경우도 있나요?**
→ "네, 생성 로직 자체는 Factory에 맡기고, 그 Factory를 어디서나 같은 창구로 접근하게 하기 위해 Subsystem 같은 Singleton 형태로 감싸는 경우가 있습니다. 스폰 시스템을 Subsystem으로 두고 내부에서 Factory로 실제 생성을 처리하는 구조가 대표적입니다."

면접관이 그 다음에 던지기 좋은 질문: *"만약 캐릭터가 여러 상태(대기, 이동, 공격, 사망)를 오가는 로직에 if-else가 계속 쌓인다면 어떻게 정리하시겠어요?"* → 10강(State Pattern)으로 이어지는 질문이다.

---

## 9. 셀프 체크

- [ ] Meyer's Singleton을 C++로 구현하고, 복사 생성자를 막아야 하는 이유를 설명할 수 있다.
- [ ] 전통적인 Singleton이 언리얼에서 문제가 되는 이유(생명주기 불일치, GC/블루프린트 미지원)를 안다.
- [ ] GameInstance/Subsystem이 왜 "사실상 Singleton"인지, 전통적인 방식과 뭐가 다른지 설명할 수 있다.
- [ ] Factory Pattern이 생성 로직을 어떻게 한 곳에 몰아주는지 코드로 설명할 수 있다.
- [ ] Singleton과 Factory를 함께 쓰는 실전 구조를 예로 들 수 있다.

---

다음 강의: **10강 — State Pattern**
"캐릭터 상태 머신을 if-else 지옥 없이 어떻게 패턴으로 정리하는지"를 다룬다.
