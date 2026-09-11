# 10강 — State Pattern

> **목표:** 캐릭터 상태 머신이 if-else/enum 분기로 뒤엉키는 이유를 이해하고, State Pattern으로 상태별 로직을 분리해서 확장 가능한 구조로 짤 수 있게 된다.
> **선수 지식:** 8강(Observer Pattern), 9강(Singleton과 Factory).

---

## 1. 캐릭터 상태 분기가 200줄짜리 if-else가 된 사건

캐릭터가 대기(Idle), 이동(Move), 공격(Attack), 사망(Dead) 상태를 가진다. 처음엔 enum과 if-else로 짰다.

```cpp
enum class ECharacterState { Idle, Move, Attack, Dead };

void ACharacter::Tick(float DeltaTime)
{
    if (CurrentState == ECharacterState::Idle)
    {
        if (InputVelocity.Size() > 0)
        {
            CurrentState = ECharacterState::Move;
        }
        else if (bAttackPressed)
        {
            CurrentState = ECharacterState::Attack;
        }
        PlayAnimation(IdleAnim);
    }
    else if (CurrentState == ECharacterState::Move)
    {
        if (InputVelocity.Size() == 0)
        {
            CurrentState = ECharacterState::Idle;
        }
        else if (bAttackPressed)
        {
            CurrentState = ECharacterState::Attack;   // "이동 중 공격 가능?" 로직이 여기 또 필요
        }
        MoveCharacter(InputVelocity, DeltaTime);
        PlayAnimation(MoveAnim);
    }
    else if (CurrentState == ECharacterState::Attack)
    {
        // 공격 애니메이션 진행 체크, 콤보 입력 체크, 공격 중 이동 불가 처리...
        // 상태가 늘어날수록 이 블록도, 위 블록들의 분기 조건도 계속 늘어남
    }
    else if (CurrentState == ECharacterState::Dead)
    {
        // ...
    }
}
```

상태가 4개일 때도 벌써 "다른 상태로 전환 가능한지"를 체크하는 조건이 상태마다 흩어져있다. 여기에 "구르기(Dodge)", "스킬 시전(Casting)", "기절(Stun)"이 추가되면 **각 상태 블록 안에 다른 모든 상태로의 전환 조건이 다 들어가야 해서** `if-else`가 기하급수적으로 꼬인다. 게다가 "공격 중엔 이동 입력을 무시해야 한다"처럼 상태 간 규칙이 여러 블록에 중복돼서, 하나 고치면 다른 곳도 같이 고쳐야 하는 상황이 반복된다.

---

## 2. State Pattern — "상태마다 자기가 할 일과 전환 규칙을 스스로 갖는다"

발상의 전환은 이거다. **"캐릭터가 상태를 확인하며 분기하는" 게 아니라, "각 상태 자체가 자기 할 일과 다음 상태로의 전환 조건을 알고 있게"** 만든다.

```
[if-else 방식]                          [State Pattern]

Character                               Character ──→ 현재 CharacterState* 하나만 들고 있음
  │ if CurrentState == Idle: ...              │
  │ else if == Move: ...                      ▼
  │ else if == Attack: ...              CharacterState (인터페이스)
  │ else if == Dead: ...                  ├── IdleState::Tick()   → 자기 전환 조건도 자기가 앎
  └─ 모든 상태 로직이 한 함수에 뒤섞임      ├── MoveState::Tick()
                                          ├── AttackState::Tick()
                                          └── DeadState::Tick()
```

```cpp
class ACharacter;   // 전방 선언

class FCharacterState
{
public:
    virtual ~FCharacterState() = default;
    virtual void Enter(ACharacter* Character) {}                    // 이 상태로 들어올 때 한 번
    virtual void Tick(ACharacter* Character, float DeltaTime) = 0;  // 이 상태인 동안 매 프레임
    virtual void Exit(ACharacter* Character) {}                     // 이 상태를 떠날 때 한 번
};
```

```cpp
class FIdleState : public FCharacterState
{
public:
    virtual void Enter(ACharacter* Character) override
    {
        Character->PlayAnimation(IdleAnim);
    }

    virtual void Tick(ACharacter* Character, float DeltaTime) override
    {
        if (Character->GetInputVelocity().Size() > 0)
        {
            Character->ChangeState(MakeUnique<FMoveState>());   // 3강 스마트 포인터로 상태 소유
        }
        else if (Character->IsAttackPressed())
        {
            Character->ChangeState(MakeUnique<FAttackState>());
        }
    }
};

class FAttackState : public FCharacterState
{
public:
    virtual void Enter(ACharacter* Character) override
    {
        Character->PlayAnimation(AttackAnim);
        ElapsedTime = 0.0f;
    }

    virtual void Tick(ACharacter* Character, float DeltaTime) override
    {
        // 공격 중엔 이동 입력을 아예 검사조차 안 함 — 이 상태 밖에서 신경 쓸 필요가 없어짐
        ElapsedTime += DeltaTime;
        if (ElapsedTime >= AttackDuration)
        {
            Character->ChangeState(MakeUnique<FIdleState>());
        }
    }

private:
    float ElapsedTime = 0.0f;
    static constexpr float AttackDuration = 0.6f;
};
```

```cpp
class ACharacter : public AActor
{
public:
    void Tick(float DeltaTime) override
    {
        CurrentState->Tick(this, DeltaTime);   // 캐릭터는 "지금 상태에게 위임"만 함
    }

    void ChangeState(TUniquePtr<FCharacterState> NewState)
    {
        CurrentState->Exit(this);
        CurrentState = MoveTemp(NewState);
        CurrentState->Enter(this);
    }

private:
    TUniquePtr<FCharacterState> CurrentState = MakeUnique<FIdleState>();
};
```

`ACharacter::Tick`은 이제 **"지금 상태 객체에게 물어봐"** 한 줄뿐이다. `FAttackState` 안에는 "이동 입력을 무시해야 한다"는 규칙이 **아예 검사할 필요조차 없이** 자연스럽게 지켜진다 — `FAttackState::Tick`이 이동 입력을 확인하는 코드 자체를 안 갖고 있으니까. 새 상태(구르기, 기절)를 추가해도 **새 클래스 하나 만들고, 그 상태로 들어가는 조건만 관련 상태에 추가**하면 된다. 기존 `FIdleState`, `FMoveState`의 다른 부분은 안 건드려도 된다.

---

## 3. 이게 실제로 얼마나 확장에 유리한가

"구르기(Dodge)" 상태를 추가한다고 해보자.

```cpp
// if-else 방식이었다면: 모든 상태 블록 안에 "구르기 입력 체크"를 하나씩 추가해야 함
// State Pattern이라면:

class FDodgeState : public FCharacterState
{
public:
    virtual void Enter(ACharacter* Character) override
    {
        Character->PlayAnimation(DodgeAnim);
        Character->SetInvincible(true);   // 구르기 중 무적 — 이 상태만의 고유 로직
    }

    virtual void Tick(ACharacter* Character, float DeltaTime) override
    {
        // ...
    }

    virtual void Exit(ACharacter* Character) override
    {
        Character->SetInvincible(false);
    }
};

// 기존 상태들 중 "구르기가 가능해야 하는" 상태에만 전환 조건 한 줄씩 추가
// (예: FIdleState::Tick, FMoveState::Tick에 각각)
if (Character->IsDodgePressed())
{
    Character->ChangeState(MakeUnique<FDodgeState>());
}
```

`FAttackState`, `FDeadState`처럼 "구르기가 애초에 불가능해야 하는" 상태는 **손댈 필요가 전혀 없다.** if-else 방식이었다면 "구르기 체크"를 모든 상태 블록에 넣고 나서 "어 근데 공격 중엔 구르면 안 되지" 하고 다시 예외 조건을 추가하는 식으로 갔을 거다. State Pattern은 애초에 **"이 상태에 없는 코드는 절대 실행되지 않는다"**는 게 구조로 보장된다.

---

## 4. 8강 Observer Pattern과의 연결 — 상태 전환을 알림으로

상태가 바뀔 때 UI(예: 상태 아이콘)나 사운드 시스템에 알려야 한다면, 8강에서 배운 Observer Pattern을 `ChangeState` 안에 자연스럽게 얹을 수 있다.

```cpp
DECLARE_MULTICAST_DELEGATE_TwoParams(FOnStateChanged, FName /*OldState*/, FName /*NewState*/);

class ACharacter : public AActor
{
public:
    FOnStateChanged OnStateChanged;

    void ChangeState(TUniquePtr<FCharacterState> NewState)
    {
        FName OldName = CurrentState->GetStateName();
        CurrentState->Exit(this);
        CurrentState = MoveTemp(NewState);
        CurrentState->Enter(this);
        OnStateChanged.Broadcast(OldName, CurrentState->GetStateName());   // UI 등이 이걸 구독
    }

private:
    TUniquePtr<FCharacterState> CurrentState = MakeUnique<FIdleState>();
};
```

캐릭터는 여전히 "누가 이 이벤트를 듣는지" 전혀 모른다. State Pattern이 "무슨 상태인가"를 깔끔하게 정리해주고, Observer Pattern이 "그 변화를 누구에게 알릴까"를 깔끔하게 정리해주는 식으로 두 패턴이 자연스럽게 겹친다.

---

## 5. State Pattern이 항상 정답은 아니다

상태가 2~3개뿐이고 전환 규칙이 단순하면, 클래스를 여러 개 만드는 State Pattern이 오히려 **과한 설계**일 수 있다.

```cpp
// 상태가 딱 둘(문이 열림/닫힘)뿐이라면 이 정도로 충분하다 — 클래스 4개씩 만들 필요 없음
enum class EDoorState { Open, Closed };

void ADoor::Interact()
{
    CurrentState = (CurrentState == EDoorState::Closed) ? EDoorState::Open : EDoorState::Closed;
    PlayAnimation(CurrentState == EDoorState::Open ? OpenAnim : CloseAnim);
}
```

**판단 기준은 "상태 개수 × 상태별 고유 로직의 양"이다.** 상태가 늘어나거나(4개 이상), 상태마다 진입/유지/이탈 시 해야 할 일이 복잡해지는 순간부터 if-else의 유지보수 비용이 State Pattern으로 클래스를 나누는 비용을 넘어선다.

---

## 6. 흔한 오해

### 오해 1: "State Pattern은 상태가 몇 개든 항상 쓰는 게 좋다"

아니다. 상태가 2~3개고 로직이 단순하면 enum + 간단한 분기가 더 읽기 쉽다. State Pattern은 **상태가 많고 상태별 로직/전환 규칙이 복잡할 때** 이득이 커지는 패턴이다.

### 오해 2: "상태 전환 조건은 캐릭터(Context) 클래스가 판단해야 한다"

State Pattern의 핵심은 정반대다. **각 상태 객체가 자기 다음 전환 조건을 스스로 판단**하고, 캐릭터는 그 결정을 받아서 상태만 바꿔준다. 캐릭터 클래스에 다시 "지금 무슨 상태니까 다음은 뭐가 돼야 해"를 판단하는 로직을 넣으면 State Pattern을 쓰는 의미가 없어진다.

### 오해 3: "State Pattern을 쓰면 상태 전환 버그가 아예 안 생긴다"

패턴 자체가 버그를 없애주진 않는다. "이 상태에서 저 상태로 못 가게 막는 걸 깜빡하는" 실수는 여전히 가능하다. 다만 그 실수를 찾고 고치는 범위가 **해당 상태 클래스 하나로 좁혀진다는 게** 이득이다.

### 오해 4: "각 상태 클래스는 서로 참조하며 강하게 결합돼도 상관없다"

이상적으로는 상태끼리 서로를 직접 몰라도 되게 짜는 게 좋다. `FIdleState`가 `FMoveState`로 전환할 땐 "전환할 상태를 만들어서 넘겨준다"는 최소한의 의존만 있으면 되고, `FMoveState` 내부 구현까지 알 필요는 없다.

### 오해 5: "State Pattern과 Observer Pattern은 서로 관계없는 별개 패턴이다"

실전에서는 자주 같이 쓰인다. 상태가 바뀔 때 그 사실을 다른 시스템(UI, 사운드, 애니메이션)에 알려야 하는 경우가 많아서, `ChangeState` 안에서 Delegate를 `Broadcast`하는 조합이 흔하다.

---

## 7. 면접에서 나오면

**Q1. State Pattern이 뭔가요?**
→ "객체의 상태에 따라 달라지는 동작을, if-else로 한 곳에 몰아넣지 않고 상태마다 별도의 클래스로 분리하는 패턴입니다. 각 상태 클래스가 자기 할 일과 다음 상태로의 전환 조건을 스스로 가지고 있어서, Context 객체는 현재 상태 객체에게 위임만 하면 됩니다."

**Q2. if-else나 switch로 상태를 관리하면 왜 문제가 되나요?**
→ "상태가 늘어날수록 각 상태 블록 안에 다른 모든 상태로의 전환 조건이 중복돼서 들어가야 하고, 하나의 함수가 계속 커집니다. 상태 간 규칙(예: 공격 중엔 이동 불가)이 여러 블록에 흩어져서 하나 고치면 다른 곳도 같이 확인해야 하는 유지보수 문제가 생깁니다."

**Q3. State Pattern으로 바꾸면 뭐가 좋아지나요?**
→ "새로운 상태를 추가할 때 새 클래스 하나만 만들고, 그 상태로 들어가는 조건이 있는 곳에만 코드를 추가하면 됩니다. 기존 상태 클래스들 중 그 새 상태와 무관한 것들은 전혀 건드릴 필요가 없어서, 변경의 영향 범위가 좁아집니다."

**Q4. 캐릭터 상태 머신을 구현한다면 상태 전환 로직을 어디에 두시겠어요?**
→ "각 상태 클래스 안에 두겠습니다. 캐릭터(Context) 클래스가 다음 상태를 판단하게 하면 결국 캐릭터 쪽에 if-else가 다시 쌓이게 되어서, State Pattern을 쓰는 의미가 없어집니다. 상태 자신이 자기 전환 조건을 아는 게 이 패턴의 핵심입니다."

**Q5. 상태가 두세 개뿐인 문(Door) 같은 오브젝트에도 State Pattern을 쓰시겠어요?**
→ "아니요, 그런 경우엔 enum과 간단한 분기로 충분합니다. State Pattern은 상태 개수가 많고 상태별 로직이 복잡할 때 이득이 커지는 패턴이라, 단순한 경우에 클래스를 여러 개 만드는 건 과한 설계입니다."

**Q6. State Pattern과 Observer Pattern을 같이 쓰는 경우를 설명해주세요.**
→ "상태가 바뀔 때 그 변화를 UI나 사운드 시스템에 알려야 하는 경우, 상태 전환 함수 안에서 델리게이트를 Broadcast하는 방식으로 두 패턴을 조합합니다. 캐릭터는 상태 전환만 처리하고, 그 결과를 누가 구독해서 어떻게 반응할지는 신경 쓰지 않아도 됩니다."

**Q7. 상태 객체를 스마트 포인터로 관리하는 이유는 뭔가요?**
→ "상태가 전환될 때마다 이전 상태 객체는 더 이상 필요 없어지는데, unique_ptr을 쓰면 소유권이 명확하고 새 상태로 교체되는 순간 이전 상태 객체가 자동으로 소멸됩니다. delete를 직접 호출할 필요 없이 RAII로 안전하게 정리되는 겁니다."

면접관이 그 다음에 던지기 좋은 질문: *"지금까지 배운 패턴들(Pooling, Observer, Singleton/Factory, State) 중 두 개 이상을 같이 써야 하는 실제 시스템을 하나 설계해볼 수 있나요?"* → 보너스 강의에서 이런 종합형 질문에 답하는 연습을 다룬다.

---

## 8. 셀프 체크

- [ ] if-else 기반 상태 관리가 왜 상태 개수에 따라 유지보수가 어려워지는지 설명할 수 있다.
- [ ] State Pattern에서 "상태 전환 조건을 상태 자신이 갖는다"는 원칙을 코드로 설명할 수 있다.
- [ ] 새 상태를 추가할 때 State Pattern이 왜 기존 코드에 영향을 최소화하는지 예시로 들 수 있다.
- [ ] State Pattern이 오히려 과한 설계가 되는 경우(상태 2~3개, 단순 로직)를 판단할 수 있다.
- [ ] State Pattern과 Observer Pattern을 함께 쓰는 구조를 설명할 수 있다.

---

**Part 3 정리 — 게임에서 자주 쓰는 디자인 패턴**

| 강의 | 핵심 문제 | 해결 방식 |
|------|------|------|
| 7강 Object Pooling | 매 프레임 new/delete 비용 | 미리 만들어두고 재사용 |
| 8강 Observer Pattern | 상태 변화마다 관련 코드를 손으로 다 챙겨야 함 | 발행-구독으로 느슨한 결합 |
| 9강 Singleton과 Factory | 전역 인스턴스 난립, 생성 로직 중복 | 생명주기가 명확한 하나의 창구 + 생성 로직 집중 |
| 10강 State Pattern | 상태별 분기가 뒤엉킨 if-else | 상태마다 클래스로 로직/전환 조건 분리 |

---

다음 강의: **11강 — 스마트 포인터: unique_ptr, shared_ptr, weak_ptr**
"3강 RAII가 스마트 포인터로 어떻게 이어지는지, 그리고 shared_ptr을 잘못 쓰면 왜 메모리가 안 줄어드는지"를 다룬다.
